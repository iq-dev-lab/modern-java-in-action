# Platform Thread vs Virtual Thread — 1:1 vs M:N 모델

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- OS 스레드 1:1 매핑의 메모리 비용(스택 크기, 컨텍스트 스위칭)은 얼마나 되는가?
- Virtual Thread는 몇 개까지 만들 수 있고, Platform Thread와 왜 다른가?
- "Thread per Request" 모델이 처리량 상한선을 갖는 이유는?
- ForkJoinPool의 캐리어 스레드가 Virtual Thread를 스케줄링하는 메커니즘은?
- Virtual Thread가 100만 개도 만들 가능한데 무엇이 다른가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Spring Boot 서버가 CPU 코어 수보다 훨씬 많은 동시 요청을 처리해야 한다. Platform Thread(OS 스레드) 기반 모델은 메모리와 컨텍스트 스위칭 비용 때문에 수천 개 스레드를 만들 수 없다. Virtual Thread는 이 문제를 해결하여, 큰 코드 변경 없이 수백만 개의 가벼운 태스크를 동시 실행할 수 있게 한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: "Virtual Thread를 스레드 풀에 넣으면 효율적이다"
  ExecutorService pool = Executors.newFixedThreadPool(1000);
  // Virtual Thread의 생성 비용이 거의 0이므로 풀링이 오버헤드
  → Virtual Thread를 풀에 모아서 관리할 이유가 없음
  → newVirtualThreadPerTaskExecutor() 권장

실수 2: ThreadLocal이 Virtual Thread에도 안전하다고 가정
  // Virtual Thread 100만 개 × ThreadLocal 변수 = 메모리 누수 가능
  ThreadLocal<Connection> connLocal = ThreadLocal.withInitial(...);
  // 각 VT마다 별도의 값이 저장됨
  → 평문 필드나 structured concurrency 사용

실수 3: Platform Thread와 Virtual Thread를 무분별하게 혼용
  // CPU 집약적 작업은 Platform Thread를 써야 함
  ExecutorService compute = Executors.newVirtualThreadPerTaskExecutor();
  compute.submit(() -> {
      // 100% CPU 사용 연산 (컨텍스트 스위칭 오버헤드 증가)
      for (int i = 0; i < 1_000_000_000; i++) { ... }
  });
  → I/O 바운드 작업에만 Virtual Thread 사용
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 1. Virtual Thread per Task — 생성 비용이 거의 0이므로 매번 생성
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
for (int i = 0; i < 1_000_000; i++) {
    executor.submit(() -> blockingIO());  // 각 요청마다 VT 1개씩 생성
}

// 2. I/O 바운드 작업만 Virtual Thread 사용
executor.submit(() -> {
    String response = httpClient.get(url);  // 블로킹 I/O (적절)
    processResponse(response);
});

// 3. CPU 집약적 작업은 Platform Thread ForkJoinPool 사용
ForkJoinPool forkJoin = ForkJoinPool.commonPool();  // CPU 코어 수 스레드
forkJoin.invoke(recursiveTask);

// 4. ThreadLocal 대신 structured concurrency 사용
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    // 부모-자식 task 관계가 구조화됨
    // 각 VT가 스코프를 벗어나면 자동으로 정리됨
    Future<Data> task = scope.fork(() -> blockingIO());
    scope.join();
    return task.resultNow();
}

// 5. Spring Boot에서 Virtual Thread 활성화
// application.properties:
// spring.threads.virtual.enabled=true
// → @Async, WebClient 등이 자동으로 Virtual Thread 사용
```

---

## 🔬 내부 동작 원리

### 1. Platform Thread — 1:1 OS 스레드 매핑

```
Java Platform Thread:
  ┌──────────────────────────────────────┐
  │ 1 Java Thread                        │
  │  (java.lang.Thread)                  │
  │  ├─ 1MB 스택 메모리                    │
  │  ├─ OS 스레드 핸들 (Native)            │
  │  ├─ ThreadLocal Map                  │
  │  └─ 동기화 객체들                       │
  └──────────────────────────────────────┘
          ↓ 1:1 매핑
  ┌──────────────────────────────────────┐
  │ OS 스레드 (Linux pthread)            │
  │  ├─ 커널 스택 (8MB)                    │
  │  ├─ 레지스터, PC, 상태                  │
  │  └─ 스케줄링 큐에 관리됨                 │
  └──────────────────────────────────────┘

메모리 계산:
  Java 스택: 1MB (XX:ThreadStackSize=1024)
  OS 커널 스택: 8MB
  MetaSpace (스레드 객체): ~1-2KB
  ThreadLocal: 변수마다 추가
  
  10,000 Platform Thread 생성 시:
    10,000 × 1MB = 10GB 힙
    10,000 × 8MB = 80GB 커널 메모리 (실제로 가용 메모리 초과)
  
  결론: Platform Thread는 수천 개가 한계

컨텍스트 스위칭 비용:
  CPU 코어 8개, 10,000 스레드 생성 시:
    각 스레드마다 ~1ms 타이슬라이스 (최악의 경우)
    → 1초에 10,000번 컨텍스트 스위칭
    → 캐시 미스, TLB(Translation Lookaside Buffer) 리셋 오버헤드
    → 유효 처리량 급격히 감소
```

### 2. Virtual Thread — M:N 캐리어 스레드 모델

```
Java Virtual Thread (Project Loom):
  ┌──────────────────────────────────────┐
  │ 1,000,000 Virtual Thread             │
  │  각각: ~1-2KB 메모리 (Continuation)   │
  │  ├─ 스택 프레임 = JVM 힙에 저장         │
  │  ├─ 마운트 상태 (carrier thread 점유) │
  │  │   또는                            │
  │  └─ 언마운트 상태 (대기, 캐리어 비점유) │
  └──────────────────────────────────────┘
        ↓ M:N 매핑 (CPU 코어 수개)
  ┌──────────────────────────────────────┐
  │ ForkJoinPool 캐리어 스레드            │
  │ (=CPU 코어 수, 기본 8개)               │
  │  각각:                                │
  │  ├─ 1MB 스택 메모리                    │
  │  ├─ OS 스레드 핸들                     │
  │  ├─ 할당된 VT들 스케줄링                │
  │  └─ VT가 블로킹 시 다른 VT로 전환       │
  └──────────────────────────────────────┘

M:N 스케줄링:
  가정: 100개 VT, 8개 캐리어 스레드
  
  시간: t=0
    VT[0..7] 마운트됨 (각 캐리어에 1:1)
    VT[8..99] 대기 중

  시간: t=100ms
    VT[0] socket.read() 호출 → 블로킹 I/O
      → VT[0] 언마운트 (스택을 힙의 Continuation에 저장)
      → Carrier[0]은 자유로워짐
      → VT[8] 마운트 (Carrier[0]에 할당)

  시간: t=200ms
    socket 데이터 도착 → VT[0] 깨어남
    → VT[0] 다시 마운트할 캐리어 찾음
    → Carrier[2] 할당 가능 → VT[0] 마운트
    → 계속 실행

결과:
  - 1,000,000 VT × ~1-2KB = ~2-4GB (1MB 스택 × 8개 캐리어 포함)
  - 컨텍스트 스위칭: 캐리어 수(8개)만큼만 발생
  - 블로킹 I/O 중에도 캐리어는 다른 VT 실행 (멀티플렉싱)
```

### 3. Thread per Request 모델의 상한선

```
Traditional Servlet Container (Platform Thread):

Request 도착
  ↓
Thread Pool에서 스레드 할당
  ├─ DB 쿼리 (50ms)
  ├─ API 호출 (100ms)
  └─ 응답 작성 (10ms)
  ├─ Total: 160ms 블로킹
  └─ 그 동안 스레드는 블로킹됨
  ↓
응답 반환 → 스레드 반환

처리량 계산:
  스레드 풀 크기: N (예: 200)
  요청당 응답 시간: 160ms
  
  초당 처리량 = N / 0.160s = 200 / 0.160 = 1,250 req/s
  
  문제: 메모리 제약
    N = 200 × 1MB = 200MB 스택
    → 서버 1GB 메모리면 최대 ~5,000개 스레드가 현실적 한계
    → 결국 처리량은 1,250 ~ 6,250 req/s 범위

Virtual Thread per Request:

Request 도착
  ↓
Virtual Thread 할당 (비용 거의 0)
  ├─ DB 쿼리 (50ms) → 언마운트 (캐리어 해제)
  ├─ API 호출 (100ms) → 언마운트 (캐리어 해제)
  └─ 응답 작성 (10ms) → 마운트
  ↓
응답 반환 → VT 정리

처리량 계산:
  캐리어 스레드: 8개 (CPU 코어 수)
  VT 수: 1,000,000 (이론적 무제한)
  동시 블로킹 I/O: 100개 VT
  
  초당 처리량 = (1,000,000 VT × 160ms 블로킹 시간 처리) / 8 캐리어
            ≈ CPU 활용도 기반 계산
            ≈ 수만 ~ 수십만 req/s (네트워크, DB 성능 한계까지)
```

### 4. Continuation 스택 저장 메커니즘

```
Virtual Thread의 스택은 JVM 힙에 Continuation 객체로 저장됨.

언마운트 시점 (VT가 I/O 대기):
  
  ┌─────────────────────────────────────────┐
  │ Carrier Thread 스택                     │
  │ ┌───────────────────────────────────┐   │
  │ │ native socket.read()              │   │
  │ │ ← Blocking I/O (VirtualThread에서) │   │
  │ └───────────────────────────────────┘   │
  │ ↓ yield() 호출 (암묵적)                   │
  │ ┌───────────────────────────────────┐   │
  │ │ VirtualThread.yield()             │   │
  │ │ → Continuation.yield()            │   │
  │ └───────────────────────────────────┘   │
  └─────────────────────────────────────────┘
              ↓
  Continuation 객체로 변환:
  ┌─────────────────────────────────────────┐
  │ JVM Heap (Continuation 저장)            │
  │ ┌───────────────────────────────────┐   │
  │ │ Continuation {                    │   │
  │ │   StackChunk frames;  // 스택 프레임 │   │
  │ │   int sp;             // 스택 포인터 │   │
  │ │   int pc;             // 프로그램 카운터│
  │ │   VirtualThread vt;   // 소유 VT  │   │
  │ │ }                                 │   │
  │ └───────────────────────────────────┘   │
  │ │                                    │   │
  │ │ [함수 스택 프레임들]                  │   │
  │ │  main() frame                     │   │
  │ │  processRequest() frame           │   │
  │ │  httpCall() frame → socket.read() │   │
  │ │                                    │   │
  │ └───────────────────────────────────┘   │
  └─────────────────────────────────────────┘
              ↓
  Carrier 스레드 해제 → 다른 VT 실행
```

---

## 💻 실전 실험

### 실험 1: Platform Thread vs Virtual Thread 메모리 비교

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

public class ThreadMemoryComparison {
    static AtomicInteger threadCount = new AtomicInteger(0);
    
    public static void main(String[] args) throws Exception {
        // Platform Thread 생성
        System.out.println("=== Platform Thread ===");
        ExecutorService platformPool = Executors.newCachedThreadPool();
        long platformStart = Runtime.getRuntime().totalMemory();
        
        for (int i = 0; i < 10000; i++) {
            platformPool.submit(() -> {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {}
            });
        }
        Thread.sleep(100);
        long platformEnd = Runtime.getRuntime().totalMemory();
        System.out.println("Platform 메모리 증가: " + 
            ((platformEnd - platformStart) / 1024 / 1024) + " MB");
        
        platformPool.shutdown();
        Thread.sleep(2000);
        
        // Virtual Thread 생성
        System.out.println("\n=== Virtual Thread ===");
        ExecutorService virtualPool = Executors.newVirtualThreadPerTaskExecutor();
        long virtualStart = Runtime.getRuntime().totalMemory();
        
        for (int i = 0; i < 1_000_000; i++) {
            virtualPool.submit(() -> {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {}
            });
        }
        Thread.sleep(100);
        long virtualEnd = Runtime.getRuntime().totalMemory();
        System.out.println("Virtual 메모리 증가: " + 
            ((virtualEnd - virtualStart) / 1024 / 1024) + " MB");
        
        virtualPool.shutdown();
    }
}
```

### 실험 2: Thread per Request 처리량 비교

```bash
# Platform Thread 기반 (스레드 풀 200개):
# 응답 시간 160ms 기준
#   초당 처리량 = 200 / 0.160 = 1,250 req/s

# Virtual Thread 기반:
# 동일한 160ms 블로킹 I/O
#   초당 처리량 = 50,000+ req/s

# 성능 향상: ~40배
```

### 실험 3: 캐리어 스레드 확인

```bash
# Virtual Thread 1000개 생성하고 캐리어 스레드 수 확인:

java -Xmx4g -Dcom.sun.management.jmxremote \
  VirtualThreadTest.java

# jcmd로 스레드 목록 확인:
jcmd <pid> Thread.print

# 출력 예:
# ForkJoinPool-1-worker-1 (Carrier)
# ForkJoinPool-1-worker-2 (Carrier)
# ...
# ForkJoinPool-1-worker-8 (Carrier)
# → CPU 코어 수(8)만큼만 보임
```

---

## 📊 성능/비교

| 항목 | Platform Thread | Virtual Thread |
|------|-----------------|----------------|
| 메모리 (스레드당) | ~1MB | ~1-2KB |
| 생성 시간 | ~1ms | ~10μs |
| 최대 개수 (8GB 메모리) | ~8,000 | ~5,000,000 |
| 컨텍스트 스위칭 빈도 | 매우 높음 | 캐리어 수(8)만큼 |
| I/O 대기 중 자원 점유 | 전체 스택(1MB) | 스택 프레임만 힙에 저장 |
| CPU 코어 활용 | CPU × 스레드 수 | 최적 (CPU 코어만큼) |
| Thread per Request 처리량 | 수천 req/s | 수만 req/s |
| Spring 통합 | 기본값 | spring.threads.virtual.enabled=true |

---

## ⚖️ 트레이드오프

### Virtual Thread의 장점
- **메모리 효율**: 1:1 스택 오버헤드 제거
- **처리량**: 블로킹 I/O 중에 다른 작업 멀티플렉싱
- **간단한 코드**: Platform Thread와 동일한 명령형 코드

### Virtual Thread의 제약
- **Pinning 문제**: synchronized 블록이나 JNI 호출 시 캐리어 스레드 블로킹
- **ThreadLocal 메모리**: 100만 VT × ThreadLocal 변수 = 메모리 누수 가능
- **디버깅**: 스택 트레이스가 복잡함 (Continuation 레이어)
- **CPU 집약적 작업**: 100% CPU 작업은 Platform Thread가 나음

---

## 📌 핵심 정리

1. **Platform Thread**는 1:1 OS 스레드 매핑 → 수천 개가 한계
2. **Virtual Thread**는 M:N 캐리어 모델 → 수백만 개 생성 가능
3. **캐리어 스레드**(ForkJoinPool)는 CPU 코어 수만큼만 존재
4. **블로킹 I/O 중** VT는 언마운트 → 캐리어는 다른 VT 실행
5. **처리량**: PT 1,250 req/s vs VT 50,000+ req/s (160ms I/O 기준)

---

## 🤔 생각해볼 문제

**Q1.** Virtual Thread가 100만 개도 만들 수 있는데, 왜 Platform Thread는 단 몇천 개만 만드는가?

<details>
<summary>해설 보기</summary>

근본적 차이는 스택 저장소의 위치다.

**Platform Thread**: 각 스레드당 1MB OS 커널 스택 필요 (OS 관리 커널 메모리)
**Virtual Thread**: 스택이 JVM 힙에 Continuation 객체로 저장 (동적 할당)

VT는 블로킹 I/O 중 언마운트되면 스택은 힙에 있고 캐리어는 해제되므로, 1MB vs ~2KB 차이로 메모리 문제 해결.

</details>

---

**Q2.** Virtual Thread가 블로킹 I/O 중에 언마운트되면, 그 VT를 다시 깨우는 주체는?

<details>
<summary>해설 보기</summary>

Socket, 파일 시스템, 네트워크 I/O 완료 이벤트가 VT를 깨운다.

메커니즘:
1. VT가 socket.read() 호출 → 블로킹
2. JVM의 I/O multiplexer (epoll/kqueue)가 감시
3. 데이터 도착 → selector 이벤트 발생
4. JVM이 대기 중인 VT 찾아서 준비 상태로 변경
5. 다음 캐리어 스레드가 마운트해서 계속 실행

I/O 이벤트 기반 multiplexing이 필수.

</details>

---

**Q3.** Spring Boot에서 `spring.threads.virtual.enabled=true`로 설정하면 어떤 부분이 Virtual Thread를 사용하는가?

<details>
<summary>해설 보기</summary>

Virtual Thread를 사용하는 부분:

1. **Servlet 처리 스레드**: 각 HTTP 요청마다 VT 할당
2. **비동기 작업**: @Async 메서드 → VT 기반 실행
3. **WebClient 등 HTTP 클라이언트**: 블로킹 호출이 VT 최적화
4. **JDBC, database driver**: 블로킹 DB 쿼리가 VT 효율적으로 활용

설정:
```properties
spring.threads.virtual.enabled=true
server.tomcat.threads.max=200
server.tomcat.threads.min-spare=10
```

enabled=false면 기존 Platform Thread 사용.

</details>

---

<div align="center">

**[⬅️ 이전 챕터: Record Pattern](../chapter08-pattern-matching-records/06-record-pattern-destructuring.md)** | **[홈으로 🏠](../README.md)** | **[다음: Continuation 메커니즘 ➡️](./02-continuation-mechanism.md)**

</div>
