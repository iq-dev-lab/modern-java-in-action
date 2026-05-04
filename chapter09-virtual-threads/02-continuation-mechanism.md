# Continuation 메커니즘 — Stack을 어떻게 yield/resume하는가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Continuation` 객체가 JVM 힙에 스택 프레임을 저장하는 구조는 어떻게 되는가?
- `yield()`로 현재 실행 상태를 저장하고 `run()`으로 재개하는 원리는?
- ForkJoinPool 캐리어 스레드가 Virtual Thread를 스케줄링하는 작업 큐 메커니즘은?
- Mount/Unmount 전환이 일어나는 정확한 시점은?
- StackChunk와 스택 프레임의 관계는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Virtual Thread의 가장 핵심이 바로 Continuation이다. 이 메커니즘을 이해하면, Virtual Thread가 왜 블로킹 I/O 중에 캐리어 스레드를 해제할 수 있는지, 왜 메모리 효율이 좋은지를 알 수 있다. 또한 Pinning 문제 진단과 성능 최적화의 기초가 된다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: "Virtual Thread도 결국 스레드니까 스택이 OS 메모리에 있다"
  // Continuation이 힙에 저장되는 것을 모름
  → VT 100만 개 × 1MB = 1TB 필요? (오류)
  → 실제로는 ~2KB만 필요

실수 2: "yield()/run() 호출이 일반 함수 호출과 같다"
  // Continuation은 partial execution을 지원
  VirtualThread.yield();
  // 제어 흐름: 스택 전체를 힙에 저장하고 반환
  → 나중에 yield()를 호출한 지점 바로 다음부터 재개
  → 일반 함수처럼 전체 함수를 다시 호출하지 않음

실수 3: "캐리어 스레드가 VT를 동시에 여러 개 실행할 수 있다"
  // 캐리어는 한 번에 하나의 VT만 마운트
  Carrier[0]:
    - VT[0] 마운트 → 실행 (50ms)
    - VT[0] 언마운트 (블로킹 I/O)
    - VT[1] 마운트 → 실행 (동시가 아니라 순차)
    - VT[1] 언마운트
    ...
  → 동시가 아니라 시분할 멀티플렉싱

실수 4: Mount/Unmount가 사용자 코드에서 명시적으로 호출되어야 한다고 생각
  // JVM이 자동으로 감지하고 수행
  VirtualThread.run(() -> {
      socket.read();  // 블로킹 감지 → 자동 언마운트
  });
  → 사용자는 평문 블로킹 코드를 쓸 수 있음 (async/await 필요 없음)
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 1. Continuation 직접 사용 (저수준, 일반적이지 않음)
Continuation cont = new Continuation(() -> {
    System.out.println("스택 프레임 1");
    Continuation.yield(SCOPE);  // 현재 스택을 힙에 저장
    System.out.println("재개됨");  // yield()를 호출한 위치 다음에서 재개
});

// 첫 번째 실행: main() → "스택 프레임 1" 출력 후 yield
cont.run(SCOPE);

// 두 번째 실행: yield() 다음부터 재개 → "재개됨" 출력
if (!cont.isDone()) {
    cont.run(SCOPE);
}

// 2. Virtual Thread 사용 (고수준, 권장)
Thread vt = Thread.ofVirtual().start(() -> {
    // JVM이 자동으로 Continuation을 관리
    // 블로킹 I/O 감지 → 자동 yield
    String data = readSocket();  // 블로킹
    // 자동으로 yield되고, 캐리어는 다른 VT로 전환
    processData(data);
});

// 3. ForkJoinPool 캐리어 스레드 직접 확인
ForkJoinPool forkJoin = ForkJoinPool.commonPool();
System.out.println("Parallelism: " + forkJoin.getParallelism());

// 4. Mount/Unmount 상태 모니터링 (JDK 21+ 예정)
VirtualThreadScheduler scheduler = ...;
// scheduler.monitorMountTransitions();
```

---

## 🔬 내부 동작 원리

### 1. Continuation 구조

```
Continuation 객체 (JVM 힙):

┌─────────────────────────────────────────────────┐
│ Continuation {                                  │
│   private Object scope;        // Scope 참조    │
│   private Runnable task;       // 실행할 코드   │
│   private StackChunk stackChunk; // 스택 저장소 │
│   private boolean done;        // 완료 여부    │
│   private boolean mounted;     // 마운트 상태   │
│ }                                               │
└─────────────────────────────────────────────────┘

StackChunk 객체:

┌─────────────────────────────────────────────────┐
│ StackChunk {                                    │
│   private byte[] sp;           // 스택 포인터   │
│   private byte[] pc;           // 프로그램 카운터│
│   private byte[] regs[];       // 레지스터 값들 │
│   private StackChunk parent;   // 다음 청크     │
│   private Object[] locals;     // 지역 변수들   │
│   private Object[] stack;      // 피연산자 스택 │
│ }                                               │
└─────────────────────────────────────────────────┘

스택 프레임 체인:

Continuation {
  stackChunk → StackChunk[0] {  // 가장 위쪽 (현재 frame)
    frame info: processRequest()
    locals: [request, response]
    stack: [tempValue1, tempValue2]
    parent → StackChunk[1] {  // 호출 스택
      frame info: handleRequest()
      locals: [connection]
      stack: []
      parent → StackChunk[2] {
        frame info: main()
        locals: [args]
        parent → null
      }
    }
  }
}
```

### 2. Mount/Unmount 전환

```
Virtual Thread 생명주기:

┌─────────────────────────────────────────────────┐
│ VirtualThread 생성                              │
│ ┌──────────────────────────────────────────┐   │
│ │ state = NEW                              │   │
│ │ continuation = new Continuation(task)    │   │
│ └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
             ↓ start() 호출
┌─────────────────────────────────────────────────┐
│ VirtualThread 준비                              │
│ ┌──────────────────────────────────────────┐   │
│ │ state = RUNNABLE                         │   │
│ │ scheduler queue에 추가                    │   │
│ └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
             ↓ 캐리어 스레드 선택
┌─────────────────────────────────────────────────┐
│ Mount: VT를 캐리어 스레드에 할당                │
│ ┌──────────────────────────────────────────┐   │
│ │ mounted = true                           │   │
│ │ Carrier[0].mount(vt)                     │   │
│ │ → continuation.run(scope)  // 스택 복원   │   │
│ │ → vt.task.run()           // 사용자 코드 │   │
│ └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
             ↓ 블로킹 I/O 감지
┌─────────────────────────────────────────────────┐
│ Yield: 스택을 Continuation에 저장               │
│ ┌──────────────────────────────────────────┐   │
│ │ Carrier 스택:                             │   │
│ │  [frame: continuation.run()]             │   │
│ │   ↓ yield() 호출 (JVM 내부)              │   │
│ │ [전체 스택 → StackChunk[] 변환]           │   │
│ │ ↓ yield 반환 (Carrier 스택 정리)         │   │
│ │ Carrier 스택: [frame: scheduler loop]    │   │
│ └──────────────────────────────────────────┘   │
│ ┌──────────────────────────────────────────┐   │
│ │ VT 상태:                                  │   │
│ │  mounted = false                         │   │
│ │  state = BLOCKED or WAITING              │   │
│ │  stackChunk = [복원된 전체 스택]          │   │
│ └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
             ↓ I/O 완료 후 다시 준비
┌─────────────────────────────────────────────────┐
│ Resume: 다시 마운트할 캐리어 찾기               │
│ ┌──────────────────────────────────────────┐   │
│ │ state = RUNNABLE                         │   │
│ │ scheduler queue에 다시 추가                │   │
│ │ (이전과 다른 캐리어에 할당될 수 있음)     │   │
│ └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
             ↓ 캐리어 스레드 선택 (다른 Carrier 가능)
┌─────────────────────────────────────────────────┐
│ Remount: VT를 새로운 캐리어에 할당              │
│ ┌──────────────────────────────────────────┐   │
│ │ mounted = true                           │   │
│ │ Carrier[2].mount(vt)  // 이번엔 Carrier2│   │
│ │ → continuation.run(scope)  // 스택 복원   │   │
│ │ → yield() 호출지점 다음부터 계속 실행     │   │
│ └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
             ↓ 완료
┌─────────────────────────────────────────────────┐
│ Unmount + Done:                                 │
│ ┌──────────────────────────────────────────┐   │
│ │ continuation.done = true                 │   │
│ │ state = TERMINATED                       │   │
│ │ Carrier가 VT를 해제                      │   │
│ └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

### 3. ForkJoinPool 스케줄러

```
VirtualThread 스케줄링 (ForkJoinPool 기반):

┌──────────────────────────────────────────────────┐
│ ForkJoinPool 공유 풀                             │
│ (CPU 코어 수 = 8개 캐리어 스레드)                │
│                                                  │
│ ┌────────────────┐                              │
│ │ Carrier[0]     │  Work Queue: [VT5, VT2, ...] │
│ │ 현재 실행: VT0 │                              │
│ └────────────────┘                              │
│                                                  │
│ ┌────────────────┐                              │
│ │ Carrier[1]     │  Work Queue: [VT8, VT1, ...] │
│ │ 현재 실행: VT3 │                              │
│ └────────────────┘                              │
│                                                  │
│ ┌────────────────┐                              │
│ │ Carrier[2]     │  Work Queue: [VT9, VT4, ...] │
│ │ 현재 실행: idle │  (블로킹 대기 중)            │
│ └────────────────┘                              │
│                                                  │
│ ... (총 8개)                                     │
│                                                  │
│ Global Queue: [VT20, VT15, VT50, ...]           │
│ (ready 상태 대기)                               │
└──────────────────────────────────────────────────┘

스케줄링 단계:

1. VT[10] 생성:
   Global Queue에 추가: [VT20, VT15, VT50, VT10]

2. Carrier[2]가 idle:
   Global Queue에서 VT 획득 → VT10 할당
   mount(VT10) → continuation.run()

3. VT[10]이 socket.read() 호출:
   socket이 응답 없음 → JVM이 감지
   → Continuation.yield() 자동 호출
   → StackChunk에 스택 저장
   → Carrier[2]로 반환

4. Carrier[2]가 다시 work:
   Global Queue에서 VT 획득 → VT15 할당
   mount(VT15) → continuation.run()

5. VT[10]의 socket이 데이터 수신:
   selector가 이벤트 감지 → VT[10] 준비 상태
   Global Queue에 추가: [VT20, VT50, VT10, VT30]

6. 다음 캐리어가 이용 가능할 때:
   Global Queue에서 VT[10] 획득
   mount(VT10) → continuation.run() (스택 복원)
   → yield() 호출지점 다음부터 계속 실행
```

### 4. Pinning 발생 원인 (Continuation 관점)

```
일반적인 Virtual Thread (정상):

VT[0] → Carrier[0].run()
  └─ socket.read()
     └─ JVM 감지 → yield() 자동
        └─ Carrier[0] 해제, StackChunk 저장
           └─ Carrier[0] 다른 VT 실행

Synchronized 블록 포함 (Pinning 발생):

VT[0] → Carrier[0].run()
  └─ synchronized(obj) {
       ├─ socket.read()  // 블로킹
       │  └─ 문제: synchronized는 monitor lock 유지 필요
       │     (스택 상태가 필요 → yield 불가능)
       └─ yield() 시도 실패
          └─ Carrier[0] 완전히 블로킹됨
             └─ Carrier[0]의 다른 VT들 대기 (처리량 악화)

Pinning을 일으키는 두 가지 원인 (Java 21 기준):
  1. synchronized 블록/메서드 (위 다이어그램)
     → ReentrantLock으로 교체하면 해결 (LockSupport.park 기반)
  2. native method 안에서 블로킹
     → JNI 호출 중에는 yield 불가 (스택 상태가 OS native frame에 있음)
     → 가능한 한 native 블로킹을 NIO 비동기로 대체
```

---

## 💻 실전 실험

### 실험 1: Continuation yield/run 직접 사용

```java
import jdk.internal.vm.Continuation;
import jdk.internal.vm.ContinuationScope;

public class ContinuationTest {
    static ContinuationScope scope = new ContinuationScope("test");
    
    public static void main(String[] args) {
        Continuation cont = new Continuation(scope, () -> {
            System.out.println("[1] 첫 번째 실행");
            Continuation.yield(scope);
            System.out.println("[2] yield() 후 재개");
            Continuation.yield(scope);
            System.out.println("[3] 두 번째 yield() 후 재개");
        });
        
        System.out.println("=== 첫 번째 run() ===");
        cont.run();  // 스택 프레임 저장 후 yield()
        System.out.println("yield() 반환, done=" + cont.isDone());
        
        System.out.println("\n=== 두 번째 run() ===");
        cont.run();  // StackChunk에서 스택 복원
        System.out.println("yield() 반환, done=" + cont.isDone());
        
        System.out.println("\n=== 세 번째 run() ===");
        cont.run();  // 완료
        System.out.println("완료, done=" + cont.isDone());
    }
}

// 출력:
// === 첫 번째 run() ===
// [1] 첫 번째 실행
// yield() 반환, done=false
//
// === 두 번째 run() ===
// [2] yield() 후 재개
// yield() 반환, done=false
//
// === 세 번째 run() ===
// [3] 두 번째 yield() 후 재개
// 완료, done=true
```

### 실험 2: Virtual Thread 마운트/언마운트 추적

```bash
# JDK 21+ 에서 Virtual Thread 생성 및 모니터링:

java -Djdk.VirtualThreadScheduler.traceEvents=true \
     -Djdk.tracePinnedThreads=full \
     VirtualThreadApp.java

# 출력:
# [VirtualThread] Virtual Thread 0 mounted on Carrier 0
# [VirtualThread] Virtual Thread 0 yielded (park)
# [VirtualThread] Virtual Thread 1 mounted on Carrier 0
# [VirtualThread] Virtual Thread 0 resumed on Carrier 2
# ...
```

### 실험 3: StackChunk 크기 확인

```java
import java.lang.management.*;
import com.sun.management.ThreadMXBean;

public class ContinuationMemoryTest {
    public static void main(String[] args) throws Exception {
        ThreadMXBean bean = (ThreadMXBean) ManagementFactory.getThreadMXBean();
        
        // Virtual Thread 1000개 생성
        for (int i = 0; i < 1000; i++) {
            Thread.ofVirtual().start(() -> {
                try {
                    Thread.sleep(10000);  // 블로킹 I/O 시뮬레이션
                } catch (InterruptedException e) {}
            });
        }
        
        Thread.sleep(100);  // 모든 VT가 블로킹될 시간
        
        long heapUsed = bean.getHeapMemoryUsage().getUsed();
        System.out.println("Heap Used: " + (heapUsed / 1024 / 1024) + " MB");
        // 예상: ~200 MB (1000 VT × ~200KB)
        
        Thread.sleep(10000);
    }
}
```

---

## 📊 성능/비교

| 항목 | Platform Thread Stack | Virtual Thread StackChunk |
|------|----------------------|--------------------------|
| 크기 | 1MB (고정) | ~1-2KB (동적) |
| 저장소 | OS 커널 메모리 | JVM 힙 |
| yield/resume | 불가능 | 가능 (Continuation) |
| 메모리 절감 | - | 500배 (1MB vs 2KB) |
| 컨텍스트 스위칭 | OS 스케줄링 | JVM (ForkJoinPool) |
| 캐리어당 VT 수 | 1 (1:1) | 1000+ (M:N) |

---

## ⚖️ 트레이드오프

### Continuation의 장점
- **메모리**: 스택을 1-2KB로 압축
- **유연성**: yield/resume으로 자유로운 스케줄링
- **확장성**: 수백만 개 task 동시 실행

### Continuation의 제약
- **복잡성**: JVM 컴파일러와 GC가 복잡
- **디버깅**: 스택 트레이스가 여러 Continuation 계층
- **Pinning**: synchronized 블록이 yield 방해

---

## 📌 핵심 정리

1. **Continuation**: JVM 힙의 StackChunk 배열로 스택 저장
2. **yield()**: 현재 스택을 StackChunk에 저장하고 캐리어 반환
3. **run()**: StackChunk에서 스택 복원해서 yield() 이후 계속 실행
4. **ForkJoinPool**: CPU 코어 수의 캐리어로 수백만 VT 스케줄링
5. **Mount/Unmount**: I/O 이벤트 기반 자동 전환 (사용자 코드 변경 없음)

---

## 🤔 생각해볼 문제

**Q1.** Continuation은 일반 함수 호출과 달리 partial execution을 지원한다. 이것이 가능한 이유는?

<details>
<summary>해설 보기</summary>

일반 함수 호출은 전체 스택 프레임이 실행 중 스택에 머물러야 하기 때문에 중간에 멈출 수 없다.

Continuation은 **스택 전체를 힙의 StackChunk 배열로 저장**하기 때문에 가능하다:

1. yield() 호출 시:
   - 현재 스택 포인터(sp), 프로그램 카운터(pc), 레지스터들을 StackChunk 객체에 저장
   - Carrier 스택 완전 정리 (스택 메모리 해제)
   - yield() 함수에서 반환

2. run() 호출 시:
   - StackChunk에서 sp, pc, 레지스터 복원
   - yield() 호출지점 다음부터 계속 실행

즉, **스택을 메모리 객체로 변환**하는 것으로 partial execution 구현.

</details>

---

**Q2.** Virtual Thread가 블로킹 I/O 중에 자동으로 yield되는데, JVM이 어떻게 블로킹을 감지하는가?

<details>
<summary>해설 보기</summary>

**JVM이 아니라 OS 커널과 selector를 함께 사용**한다.

메커니즘:
1. VT가 socket.read() 호출 (JNI)
2. 데이터 없음 → EAGAIN 반환 (non-blocking)
3. JVM의 I/O multiplexer (epoll/kqueue)에 등록
4. JVM이 selector.select()로 이벤트 대기
5. 데이터 도착 → selector 이벤트
6. VT 깨워서 준비 상태로 변경

따라서 블로킹 I/O 라이브러리(JDBC, HttpClient)의 지원이 필수. Traditional blocking API를 사용하면서도 JVM 내부에서 non-blocking으로 변환.

</details>

---

**Q3.** 같은 Continuation 객체를 여러 스레드에서 동시에 run()할 수 있는가?

<details>
<summary>해설 보기</summary>

**아니다. Continuation은 스레드 안전하지 않다.**

한 번에 하나의 스레드(또는 캐리어)만 run()을 호출해야 한다.

Virtual Thread도 마찬가지:
- VT는 정확히 하나의 캐리어에서만 마운트됨
- mount 중에 다른 캐리어가 run()을 호출하면 data race 발생
- JVM 스케줄러가 이를 보장 (같은 VT를 동시에 여러 캐리어에 할당하지 않음)

따라서 VT가 블로킹되어 언마운트되기 전까지, 다른 캐리어가 그 VT를 건드릴 수 없다.

</details>

---

<div align="center">

**[⬅️ 이전: Platform Thread vs Virtual Thread](./01-platform-vs-virtual-thread.md)** | **[홈으로 🏠](../README.md)** | **[다음: Pinning 문제 ➡️](./03-pinning-problem.md)**

</div>
