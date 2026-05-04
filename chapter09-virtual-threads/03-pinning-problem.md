# Pinning 문제 — synchronized와 native call 진단·해결

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `synchronized` 블록 안에서 블로킹 I/O를 호출하면 왜 "Pinning"이 발생하는가?
- `-Djdk.tracePinnedThreads=full` 플래그가 Pinning을 어떻게 진단하는가?
- JFR `jdk.VirtualThreadPinned` 이벤트로 무엇을 알 수 있는가?
- `ReentrantLock`으로 교체하면 Pinning이 해결되는 이유는?
- Native call(JNI)에서의 Pinning은 완전히 회피 불가능한가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Virtual Thread의 가장 흔한 성능 문제가 Pinning이다. 기존 Platform Thread 코드에서는 `synchronized` 사용이 당연했는데, Virtual Thread로 전환하면 갑자기 처리량이 떨어질 수 있다. Pinning을 진단하고 `ReentrantLock`으로 마이그레이션하는 능력이 Virtual Thread 도입의 성패를 좌우한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: "synchronized를 Virtual Thread에서도 그냥 써도 된다"
  synchronized(cache) {
      String data = httpClient.get(url);  // 블로킹 I/O
  }
  → Pinning 발생 (캐리어 스레드 블로킹)
  → VT 100만 개 × 160ms I/O = 캐리어가 완전히 바쁜 상태
  → 처리량 저하 (VT의 이점 상실)

실수 2: "Pinning이 발생하면 서버가 튕긴다"
  // Pinning은 메모리 누수나 크래시 유발 안 함
  // 단순히 처리량 저하 (Platform Thread처럼 떨어짐)
  → 디버깅하지 않으면 "Virtual Thread 써봤는데 별로 빨라지지 않음" 발생

실수 3: "JNI 호출은 항상 Pinning이 발생한다고 포기"
  // 일부 JNI는 회피 불가능하지만, 성능 최적화 가능
  → 알림처 없이 JNI를 피하려고 하면 실용성 저하

실제 예제 버그:

public class CacheService {
    private Cache cache = new Cache();
    
    public String getData(String key) {
        synchronized(cache) {  // ← Pinning 원인
            if (cache.contains(key)) {
                return cache.get(key);
            }
            String value = remoteFetch(key);  // 블로킹 I/O
            cache.put(key, value);
            return value;
        }
    }
}

// Virtual Thread에서:
// 1. VT[0] 캐시 검사 (빠름)
// 2. remoteFetch() 블로킹 I/O 호출
// 3. "synchronized는 yield 불가능" → monitor lock 유지
// 4. Carrier 스레드 완전히 블로킹됨
// 5. Carrier의 다른 VT들 모두 대기 (처리량 급락)
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.ConcurrentHashMap;

public class CacheService {
    private ConcurrentHashMap<String, String> cache = new ConcurrentHashMap<>();
    private ReentrantLock lock = new ReentrantLock();  // ← synchronized 대신 사용
    
    // 패턴 1: ConcurrentHashMap + computeIfAbsent (권장)
    public String getData(String key) {
        return cache.computeIfAbsent(key, k -> {
            // 원자적 연산: 같은 키에 대해 한 스레드만 실행
            // Virtual Thread가 yield 가능
            return remoteFetch(k);  // 블로킹 I/O
        });
    }
    
    // 패턴 2: ReentrantLock 사용
    public String getDataWithLock(String key) {
        // 검사 로직은 lock 내 (짧게 유지)
        lock.lock();
        try {
            if (cache.containsKey(key)) {
                return cache.get(key);
            }
        } finally {
            lock.unlock();
        }
        
        // 블로킹 I/O는 lock 밖에서 수행
        String value = remoteFetch(key);
        
        // 저장은 다시 lock
        lock.lock();
        try {
            cache.putIfAbsent(key, value);
            return cache.get(key);
        } finally {
            lock.unlock();
        }
    }
    
    // 패턴 3: 쓰기는 lock, 읽기는 no-lock
    public String getDataOptimized(String key) {
        // 읽기 fast path (lock 없음)
        if (cache.containsKey(key)) {
            return cache.get(key);
        }
        
        // 쓰기가 필요할 때만 lock
        lock.lock();
        try {
            // Double-checked locking (읽기 후 쓰기 전 재확인)
            if (cache.containsKey(key)) {
                return cache.get(key);
            }
            String value = remoteFetch(key);
            cache.put(key, value);
            return value;
        } finally {
            lock.unlock();
        }
    }
    
    private String remoteFetch(String key) {
        try {
            return httpClient.get("http://api.example.com/data/" + key);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Pinning 메커니즘

```
정상적인 Virtual Thread (ReentrantLock):

Carrier[0]:
  VT[0] 마운트 → lock.lock() 획득
         ↓
       데이터 확인 (빠름, ~1μs)
         ↓
       lock.unlock() 해제
         ↓
       remoteFetch() → socket.read() 블로킹
         ↓
       Continuation.yield() 자동 호출 ✓
         ↓
       Carrier[0] 해제 → VT[1] 마운트
       
Carrier[1]:
  VT[1] 마운트 → ... (다른 작업)

Carrier[0]:
  socket 데이터 도착 → VT[0] 준비
  → VT[0] 마운트 (다시 실행)


Pinning이 발생하는 Virtual Thread (synchronized):

Carrier[0]:
  VT[0] 마운트 → synchronized(cache) 획득 (monitor lock)
         ↓
       데이터 확인
         ↓
       remoteFetch() → socket.read() 블로킹
         ↓
       JVM이 yield() 시도
       ┌────────────────────────────────┐
       │ 문제: monitor lock이 여전히 유지│
       │ (JVM 구현 제약)                │
       │ → yield() 불가능               │
       │ → Continuation.yield() 호출 X  │
       └────────────────────────────────┘
         ↓
       OS 커널의 park()로 블로킹 (I/O 완료 대기)
       Carrier[0] 완전히 점유됨 ✗
         ↓
  Carrier[0]의 다른 VT들:
       lock을 얻기 위해 Carrier[0]을 기다림
       → Carrier는 OS 블로킹 상태 (응답 없음)
       → 데드락 또는 처리량 급락
```

### 2. Monitor Lock의 제약

```
Java Monitor (synchronized의 구현):

┌──────────────────────────────────────┐
│ Object (모든 객체가 monitor 포함)    │
│ ┌────────────────────────────────┐   │
│ │ markword:                      │   │
│ │  - 해시 코드                    │   │
│ │  - GC 마크                      │   │
│ │  - Lock 상태 & owner thread    │   │
│ │    (스택 프레임 주소 또는 heap) │   │
│ └────────────────────────────────┘   │
└──────────────────────────────────────┘

synchronized의 세 가지 상태:

1. Biased Lock (경합 없음):
   markword에 thread ID 저장
   → 빠름

2. Thin Lock / Stack Lock (경합 적음):
   monitor object를 스택 프레임에 생성
   → 상대적으로 빠름

3. Fat Lock / Inflated Lock (경합 심함):
   monitor object를 힙에 할당
   → 느림, OS wait queue 사용

Virtual Thread의 문제:
  - VT의 스택은 StackChunk (힙의 배열)
  - monitor lock이 "스택 프레임" 주소를 저장할 수 없음
  - Fat Lock으로 inflated 되어야 함
  - Fat Lock은 OS wait queue 사용 → OS park()
  → Carrier 스레드 블로킹 (Pinning)

따라서 synchronized는 Virtual Thread에서 항상 최악의 경우(Fat Lock)로 동작할 가능성 높음.

ReentrantLock의 해결:
  - explicit lock acquisition/release (synchronized와 달리)
  - JVM이 "yield 가능한 시점"을 명확히 파악
  - lock을 놓는 즉시 yield() 허용
```

### 3. Pinning 진단

```
-Djdk.tracePinnedThreads=full 플래그 동작:

구동:
java -Djdk.tracePinnedThreads=full MyApp.java

Virtual Thread가 yield 불가능할 때:
  JVM이 스택 트레이스 출력:
  
  "VirtualThread-0" #26 pinned
  at java.net.SocketInputStream.read0 (native) @bci=0 (Pinned by synchronized)
    at java.net.SocketInputStream.read (SocketInputStream.java:XXX)
    at MyApp.remoteFetch (MyApp.java:50)
    at MyApp.lambda$getData$0 (MyApp.java:30)
    at java.util.concurrent.ConcurrentHashMap.computeIfAbsent (...)
    ... pinned reason: synchronized block (holder: <object id>)

분석:
  - "Pinned by synchronized" → synchronized 블록이 원인
  - stack trace → synchronized가 어디에 있는지 정확히 파악
  - object id → 어느 객체의 monitor인지 식별

다른 Pinning 원인들:
  "VirtualThread-1" #27 pinned
  at sun.misc.Unsafe.park (native) @bci=0 (Pinned by native)
  → JNI call이 VirtualThread를 블로킹 중

  "VirtualThread-2" #28 pinned
  at java.util.concurrent.locks.LockSupport.parkUntil
  → Platform Thread도 같은 패턴 (정상)
```

### 4. JFR 이벤트

```
Flight Recorder (JFR) 설정 및 분석:

명령:
jcmd <pid> JFR.start \
  filename=/tmp/recording.jfr \
  duration=60s \
  settings=profile

jfr print --events jdk.VirtualThreadPinned /tmp/recording.jfr

출력:
Event Type: jdk.VirtualThreadPinned
Time: 2024-05-04T10:15:23.456Z
VirtualThread: VirtualThread-0
Duration: 100ms
Reason: synchronized
StackTrace:
  java.net.SocketInputStream.read (SocketInputStream.java:100)
  MyApp.remoteFetch (MyApp.java:50)
  ...

분석:
  - 100ms 동안 pinned (I/O 블로킹 시간)
  - synchronized가 정확한 원인
  - 어떤 코드 라인인지 추적 가능

Event 발생 시점:
  - VT가 pinned 상태로 yield되지 않으려고 시도할 때마다 이벤트 생성
  - 같은 VT가 반복적으로 pinned되면 여러 이벤트
```

---

## 💻 실전 실험

### 실험 1: Pinning 진단

```bash
# 1. Pinning 발생하는 코드 작성
cat > PinningTest.java << 'EOF'
import java.util.*;
import java.net.*;
import java.io.*;

public class PinningTest {
    static Object lock = new Object();
    
    public static void main(String[] args) throws Exception {
        // Virtual Thread 100개 생성
        for (int i = 0; i < 100; i++) {
            Thread.ofVirtual().start(() -> {
                try {
                    blockingWithPinning();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            });
        }
        
        Thread.sleep(5000);
    }
    
    static void blockingWithPinning() throws Exception {
        synchronized(lock) {
            // synchronized 블록 내에서 I/O 블로킹
            Socket socket = new Socket("example.com", 80);
            InputStream is = socket.getInputStream();
            byte[] buf = new byte[1024];
            is.read(buf);  // ← Pinning 발생
            socket.close();
        }
    }
}
EOF

# 2. Pinning 진단 플래그 적용
java -Djdk.tracePinnedThreads=full PinningTest.java 2>&1 | grep -A 20 "pinned"

# 출력 (일부):
# "VirtualThread-0" #26 pinned
# at java.net.SocketInputStream.read0 (native)
# ...
# pinned reason: synchronized block
```

### 실험 2: synchronized vs ReentrantLock 성능

```java
import java.util.concurrent.*;
import java.util.concurrent.locks.*;
import java.util.*;

public class SynchronizedVsLock {
    static class PinningVersion {
        Object lock = new Object();
        Map<String, String> cache = new HashMap<>();
        
        public String get(String key) throws Exception {
            synchronized(lock) {
                if (cache.containsKey(key)) {
                    return cache.get(key);
                }
                String value = simulateBlockingIO();
                cache.put(key, value);
                return value;
            }
        }
    }
    
    static class LockVersion {
        ReentrantLock lock = new ReentrantLock();
        Map<String, String> cache = new HashMap<>();
        
        public String get(String key) throws Exception {
            lock.lock();
            try {
                if (cache.containsKey(key)) {
                    return cache.get(key);
                }
            } finally {
                lock.unlock();
            }
            
            String value = simulateBlockingIO();
            
            lock.lock();
            try {
                cache.putIfAbsent(key, value);
                return cache.get(key);
            } finally {
                lock.unlock();
            }
        }
    }
    
    static String simulateBlockingIO() throws Exception {
        Thread.sleep(100);  // I/O 시뮬레이션
        return "data";
    }
    
    public static void main(String[] args) throws Exception {
        // synchronized 버전
        System.out.println("=== synchronized Version ===");
        PinningVersion pinning = new PinningVersion();
        long start = System.currentTimeMillis();
        
        CountDownLatch latch = new CountDownLatch(100);
        ExecutorService vpool = Executors.newVirtualThreadPerTaskExecutor();
        
        for (int i = 0; i < 100; i++) {
            final int idx = i;
            vpool.submit(() -> {
                try {
                    pinning.get("key" + (idx % 10));
                } catch (Exception e) {}
                latch.countDown();
            });
        }
        
        latch.await();
        long elapsed = System.currentTimeMillis() - start;
        System.out.println("처리 시간: " + elapsed + " ms");
        
        vpool.shutdown();
        
        // ReentrantLock 버전
        System.out.println("\n=== ReentrantLock Version ===");
        LockVersion lockVersion = new LockVersion();
        start = System.currentTimeMillis();
        
        latch = new CountDownLatch(100);
        vpool = Executors.newVirtualThreadPerTaskExecutor();
        
        for (int i = 0; i < 100; i++) {
            final int idx = i;
            vpool.submit(() -> {
                try {
                    lockVersion.get("key" + (idx % 10));
                } catch (Exception e) {}
                latch.countDown();
            });
        }
        
        latch.await();
        elapsed = System.currentTimeMillis() - start;
        System.out.println("처리 시간: " + elapsed + " ms");
        
        vpool.shutdown();
    }
}

// 예상 결과:
// synchronized Version: ~1000ms (캐리어 스레드 블로킹)
// ReentrantLock Version: ~100ms (yield 가능)
```

### 실험 3: JFR로 Pinning 분석

```bash
# 1. JFR 기록 시작
jcmd <pid> JFR.start \
  name=pinning_test \
  filename=/tmp/pinning.jfr \
  duration=30s \
  settings=profile

# 2. PinningTest 실행
java -Djdk.tracePinnedThreads=short PinningTest.java

# 3. JFR 파일 분석
jfr print --events jdk.VirtualThreadPinned /tmp/pinning.jfr

# 4. JFR 그래프 (jdk.codeviewer 등)
jfr --open /tmp/pinning.jfr
```

---

## 📊 성능/비교

| 항목 | synchronized | ReentrantLock | ConcurrentHashMap |
|------|-------------|---------------|-------------------|
| Pinning 여부 | 예 (VT에서) | 아니오 | 아니오 |
| Virtual Thread 처리량 | 낮음 (Platform처럼) | 높음 | 매우 높음 |
| 구현 복잡도 | 낮음 | 중간 | 중간 |
| double-checked locking | 필요 | 필요 | 불필요 |
| 마이그레이션 난이도 | 높음 (코드 재구성) | 중간 (lock/unlock) | 낮음 |

---

## ⚖️ 트레이드오프

### ReentrantLock의 장점
- **Pinning 회피**: yield() 명시적으로 가능
- **유연성**: tryLock(), lockInterruptibly() 등 고급 기능
- **명시적 제어**: lock/unlock 위치를 개발자가 결정

### ReentrantLock의 제약
- **코드 복잡**: try-finally 필수 (synchronized는 자동)
- **실수 가능성**: unlock 호출 누락 시 데드락
- **마이그레이션**: 모든 synchronized를 찾아 교체해야 함

### JNI의 Pinning (회피 불가능)
- Native 코드가 캐리어 스레드를 직접 사용하면 항상 Pinning
- 해결책: Native 코드를 Java로 재작성, 또는 별도 Platform Thread 풀 사용

---

## 📌 핵심 정리

1. **Pinning**: synchronized 또는 JNI 호출 중에 VT가 yield 불가능 → 캐리어 블로킹
2. **원인**: monitor lock이 스택 프레임 주소를 저장 (StackChunk와 호환 불가)
3. **진단**: `-Djdk.tracePinnedThreads=full`, JFR `jdk.VirtualThreadPinned` 이벤트
4. **해결**: `ReentrantLock` 사용 또는 `ConcurrentHashMap` 같은 동시성 컬렉션
5. **마이그레이션**: synchronized 블록을 찾아 lock/unlock으로 교체

---

## 🤔 생각해볼 문제

**Q1.** synchronized 블록이 짧으면 (1마이크로초) Pinning이 문제가 아닌가?

<details>
<summary>해설 보기</summary>

맞다. Pinning의 영향은 **블로킹 시간**에 정비례한다.

synchronized(cache) {
    if (cache.contains(key)) return cache.get(key);  // 1μs
}

이 경우:
- lock acquisition: 10ns
- contains/get: 100ns
- lock release: 10ns
- Total: 200ns (매우 짧음)
- Pinning 영향: 무시할 수 있는 수준

반면:
synchronized(cache) {
    remoteFetch(url);  // 100ms (I/O 블로킹)
}

이 경우:
- lock 시간: 100ms (대부분 I/O 대기)
- Pinning 영향: 심각 (캐리어 100ms 블로킹)

결론: synchronized 블록 내에서 **블로킹 I/O를 호출하면 안 된다**. 검사/업데이트만 짧게 유지하면 Pinning의 영향 최소화 가능 (하지만 여전히 비효율).

</details>

---

**Q2.** ReentrantLock으로 바꾸면 Pinning이 완전히 해결되는가?

<details>
<summary>해설 보기</summary>

**대부분 해결되지만 100% 보장은 아니다.**

ReentrantLock은 기본적으로 yield 가능하다:

lock.lock();
try {
    // 검사
} finally {
    lock.unlock();  // ← 이 시점에 yield 가능
}

remoteFetch();  // lock 없이 yield 가능

하지만 예외:

1. **같은 스레드가 반복적으로 lock**:
   lock.lock();
   lock.lock();  // 재진입
   // 스택에 lock 정보가 쌓임 (이것도 일종의 pinning)
   → 일반적으로는 문제 없음

2. **condition variable 대기**:
   Condition cond = lock.newCondition();
   lock.lock();
   cond.await();  // OS wait queue로 갈 수 있음
   → 이 경우도 yield 가능 (나중에 signal 받으면 깨어남)

따라서 **ReentrantLock이 항상 해결책**이 맞다. synchronized보다는 훨씬 낫지만, lock 구조 자체를 피하는 것 (ConcurrentHashMap.computeIfAbsent)이 최선.

</details>

---

**Q3.** Platform Thread에서는 Pinning이 문제가 아닌데, Virtual Thread에서만 문제인 이유는?

<details>
<summary>해설 보기</summary>

**그 이유는 캐리어 스레드의 개수 차이**다.

Platform Thread:
- 200개 스레드 풀
- 각 스레드가 독립적인 OS 스택 보유
- synchronized 블로킹 → 그 스레드만 블로킹
- 다른 199개 스레드는 계속 실행
- 처리량 저하 = 1/200 (무시할 수 있는 수준)

Virtual Thread:
- 100만 개 VT
- 8개 캐리어 스레드만 존재
- synchronized 블로킹 → 캐리어 블로킹
- 같은 캐리어의 다른 VT들 모두 대기
- 처리량 저하 = 1/8 (심각)

예시:
Platform Thread: 1,250 req/s
  하나의 synchronized 블로킹 → 1,250 × (199/200) ≈ 1,244 req/s (무시)

Virtual Thread: 50,000 req/s
  하나의 synchronized 블로킹 → 50,000 × (7/8) ≈ 43,750 req/s (심각한 저하)

따라서 Virtual Thread에서는 같은 코드도 성능 특성이 완전히 달라진다.

</details>

---

<div align="center">

**[⬅️ 이전: Continuation 메커니즘](./02-continuation-mechanism.md)** | **[홈으로 🏠](../README.md)** | **[다음: Structured Concurrency ➡️](./04-structured-concurrency.md)**

</div>
