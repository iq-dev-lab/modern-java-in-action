# ForkJoinPool.commonPool() 의존성 문제와 해결

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `CompletableFuture.supplyAsync()` 기본 Executor가 `commonPool`인 이유와 문제는?
- `parallelStream()`과 `commonPool` 공유 시 발생하는 starvation은?
- IO-bound 작업의 최적 스레드 수 = N × (1 + W/C) 공식은?
- Virtual Thread를 Executor로 사용할 때의 장점은?
- 프로덕션에서 commonPool 의존성을 제거하려면?

---

## 🔍 왜 이 개념이 실무에서 중요한가

대부분의 비동기 프레임워크 (Spring Boot, Project Reactor, RxJava)가 CompletableFuture 기반이거나 비슷한 문제를 겪는다. commonPool 의존성을 모르고 프로덕션에 배포하면, 높은 부하 상황에서 갑자기 latency가 폭증하거나, parallelStream과의 경합으로 인해 예측 불가능한 성능 저하가 발생한다. 정확한 진단과 해결책을 알아야 안정적인 시스템을 구축할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: commonPool 기본값을 신뢰하고 비동기 작업 대량 처리
  // 스레드 풀 기본값 = CPU 코어 수 (4코어 머신 = 4 스레드)
  
  List<Future<Data>> futures = new ArrayList<>();
  for (int i = 0; i < 1000; i++) {
      futures.add(CompletableFuture.supplyAsync(() → {
          Thread.sleep(100);  // I/O 대기
          return fetchData(i);
      }));  // 암시적 commonPool 사용
  }
  
  // 4개 스레드로 1000개 작업 처리:
  // 평균 완료 시간: 1000 * 100ms / 4 = 25초 (!!!)

실수 2: parallelStream과 supplyAsync 함께 사용
  List<Item> items = ...;  // 1000개
  
  items.parallelStream()
      .map(item → 
          CompletableFuture.supplyAsync(() → {
              // 비동기 I/O
              return processWithAPI(item);
          })
          .get()  // 블로킹!
      )
      .collect(toList());
  
  → parallelStream과 supplyAsync가 commonPool 공유
  → 4개 스레드가 병렬 스트림 처리와 비동기 작업 경합
  → ForkJoinTask.invokeAll()이 스레드 대기 → Starvation
  → 전체 성능 저하

실수 3: I/O-bound 작업에 commonPool 스레드 수 적용
  // 네트워크 API (평균 500ms 대기)
  스레드 수 = 4 (CPU 코어)
  
  → I/O 대기 중 스레드 점유 (블로킹)
  → 효율성: 4 스레드 × (1000ms 처리 / 500ms I/O) ≈ 8개 작업/초
  → 실제 필요: 40~100개 스레드 가능

실수 4: 모든 CompletableFuture를 commonPool에서 처리
  // parallelStream 존재 시
  IntStream.range(0, 1000)
      .parallel()
      .map(i → {
          CompletableFuture.supplyAsync(() → expensive(i))
              .join();  // 블로킹
          return i;
      })
      .toArray();
  
  → ForkJoinPool.invoke() 내부에서 nested task 발생
  → commonPool 스레드 모두 자식 task 대기
  → 부모 task 블로킹 → Starvation
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 1. I/O-bound 전용 Executor 설계
int corePoolSize = 10;
int maxPoolSize = Math.max(32, 
    Runtime.getRuntime().availableProcessors() * (1 + 10));  // W/C ≈ 10
ExecutorService ioExecutor = new ThreadPoolExecutor(
    corePoolSize, maxPoolSize,
    60, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(1000),
    new ThreadFactory() {
        private int counter = 0;
        @Override
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r, "IoWorker-" + (counter++));
            t.setDaemon(false);
            return t;
        }
    }
);

// 2. 명시적 Executor 사용
CompletableFuture<Data> result = CompletableFuture
    .supplyAsync(() → {
        // I/O 작업: ioExecutor의 스레드에서 실행
        return fetchFromDatabase();
    }, ioExecutor);

// 3. parallelStream과 분리
List<CompletableFuture<Result>> futures = items.stream()  // 동기
    .map(item → CompletableFuture.supplyAsync(() → {
        // I/O: ioExecutor 사용
        return process(item);
    }, ioExecutor))
    .collect(toList());

// allOf로 모두 완료 대기 (parallelStream 없음)
CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
    .thenApply(v → futures.stream()
        .map(CompletableFuture::join)
        .collect(toList())
    )
    .get();

// 4. Virtual Thread 활용 (Java 19+)
ExecutorService virtualExecutor = 
    Executors.newVirtualThreadPerTaskExecutor();

CompletableFuture<Data> vtResult = CompletableFuture
    .supplyAsync(() → slowIO(), virtualExecutor);

// Virtual Thread는 스레드당 ~1KB, 수천 개 생성 가능
// I/O 대기 시 자동 yield → OS 레벨 스케줄링 오버헤드 없음

// 5. commonPool 의존성 제거 완전 예제
class AsyncDataService {
    private final ExecutorService ioExecutor;
    private final int maxConcurrency;
    
    public AsyncDataService() {
        int processors = Runtime.getRuntime().availableProcessors();
        this.maxConcurrency = processors * (1 + 10);  // W/C
        this.ioExecutor = Executors.newFixedThreadPool(maxConcurrency);
    }
    
    public CompletableFuture<List<Data>> fetchBatch(List<Id> ids) {
        List<CompletableFuture<Data>> futures = ids.stream()
            .map(id → fetchOne(id))
            .collect(toList());
        
        return CompletableFuture
            .allOf(futures.toArray(new CompletableFuture[0]))
            .orTimeout(30, TimeUnit.SECONDS)
            .thenApply(v → futures.stream()
                .map(CompletableFuture::join)
                .collect(toList())
            );
    }
    
    private CompletableFuture<Data> fetchOne(Id id) {
        // ioExecutor 명시적 사용 → commonPool 회피
        return CompletableFuture.supplyAsync(() → 
            database.query(id), ioExecutor
        );
    }
}
```

---

## 🔬 내부 동작 원리

### 1. commonPool 의존성 구조

```
┌─────────────────────────────────────────────────────────┐
│ ForkJoinPool.commonPool()                               │
├─────────────────────────────────────────────────────────┤
│ 특징:                                                     │
│  - 전역 싱글톤 (모든 ForkJoin 작업 공유)                 │
│  - parallelism = Runtime.getRuntime()                  │
│              .availableProcessors()                    │
│  - 스레드 팩토리: InnocuousForkJoinWorkerThreadFactory   │
│  - Work-Stealing 큐 (FIFO 기반)                         │
│                                                         │
│ 스레드 계산:                                              │
│  - 4코어 머신 → 4개 스레드 (병렬도 4)                   │
│  - 16코어 머신 → 16개 스레드 (병렬도 16)                │
│                                                         │
│ 문제:                                                     │
│  - I/O 작업에 inadequate (I/O 대기 중 스레드 낭비)      │
│  - 여러 작업 소스의 경합 (parallelStream + CF)          │
│  - Nested parallelism → Starvation                     │
└─────────────────────────────────────────────────────────┘

CommonPool 사용 경로:

CompletableFuture.supplyAsync(fn)
  ↓
asyncExecutor 기본값 = null
  ↓
uniApplyStage(null, fn)
  ↓
execute() 호출 시:
  if (executor == null) {
      executor = ForkJoinPool.commonPool();
  }
  executor.execute(fn);
```

### 2. Starvation 메커니즘

```
Starvation: 스레드 풀의 모든 스레드가 다른 작업을 기다리는 현상

시나리오: parallelStream + CompletableFuture.supplyAsync()

┌─────────────────────────────────────────────────────────┐
│ CommonPool (4 스레드)                                    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ [Thread 1]  parallelStream 실행 중                      │
│             ├─ supplyAsync(future1)  → join() 블로킹  ◀ 문제
│             ├─ supplyAsync(future2)  → join() 블로킹
│             │                                         │
│ [Thread 2]  parallelStream 실행 중                      │
│             ├─ supplyAsync(future3)  → join() 블로킹  │
│             └─ (더 이상 스레드 없음)                    │
│                                                         │
│ [Thread 3]  future1 실행 대기 (부모 스레드가 블로킹)   │
│             ├─ 하지만 부모 스레드도 commonPool        │
│             └─ (deadlock 위험)                        │
│                                                         │
│ [Thread 4]  future2/3 실행 대기                        │
│             └─ (또 다른 작업 대기)                     │
│                                                         │
└─────────────────────────────────────────────────────────┘

ForkJoinPool.invoke() 내부 로직:

public <T> T invoke(ForkJoinTask<T> task) {
    Thread t = Thread.currentThread();
    if (t instanceof ForkJoinWorkerThread) {
        ForkJoinWorkerThread w = (ForkJoinWorkerThread)t;
        if (w.pool == this) {
            return task.invoke();  // 현재 풀 스레드라면 실행
        }
    }
    // 외부 스레드라면 externalPush
    externalPush(task);
    return task.join();  // 블로킹 대기!
}

결과:
  - parallelStream의 join()이 외부 스레드라면
  - externalPush로 commonPool에 task 등록
  - 그 task는 부모 task의 블로킹을 기다림
  - 부모 task 블로킹 중 자식 task 실행 불가능
  → Deadlock 또는 Starvation
```

### 3. I/O-bound 최적 스레드 수 계산

```
일반적인 스레드 풀 공식:

  최적 스레드 수 = N × (1 + W/C)
  
  N = CPU 코어 수
  W = I/O 대기 시간 (wall-clock time)
  C = 계산 시간 (computation time)

예 1: 데이터베이스 쿼리
  - 평균 쿼리 시간: 100ms
  - 평균 처리 시간: 10ms (결과 처리)
  - W/C = 100 / 10 = 10
  - 최적 스레드 수 = 4 × (1 + 10) = 44개

예 2: REST API 호출
  - 평균 HTTP 왕복: 500ms
  - 평균 처리 시간: 50ms (JSON 파싱)
  - W/C = 500 / 50 = 10
  - 최적 스레드 수 = 4 × (1 + 10) = 44개

예 3: 캐시 조회 (빠름)
  - 평균 I/O 시간: 1ms
  - 평균 처리 시간: 1ms
  - W/C = 1 / 1 = 1
  - 최적 스레드 수 = 4 × (1 + 1) = 8개

구현:

ExecutorService createIoExecutor(int ioWaitMs, int computeMs) {
    int cores = Runtime.getRuntime().availableProcessors();
    double wc = (double) ioWaitMs / computeMs;
    int threadCount = (int) Math.ceil(cores * (1 + wc));
    
    return Executors.newFixedThreadPool(threadCount);
}

// 사용
ExecutorService dbExecutor = createIoExecutor(100, 10);  // 44개
ExecutorService apiExecutor = createIoExecutor(500, 50);  // 44개
ExecutorService cacheExecutor = createIoExecutor(1, 1);   // 8개
```

### 4. Virtual Thread 기반 Executor

```
Virtual Thread (Project Loom, Java 19+):

  가상 스레드 = OS 스레드가 아닌 JVM 관리 스레드
  특징:
    - 생성 비용: ~1KB (플랫폼 스레드 ~1MB)
    - I/O 블로킹 자동 yield (carrier thread 해제)
    - 수천 개 스레드 생성 가능

사용:

ExecutorService virtualExecutor = 
    Executors.newVirtualThreadPerTaskExecutor();

// 각 작업마다 새로운 가상 스레드 생성
CompletableFuture.supplyAsync(() → slowIO(), virtualExecutor);

동작 원리:

┌─────────────────────────────────────────────────────┐
│ 플랫폼 스레드 (Carrier Thread)                       │
│  ├─ 가상 스레드 1 (I/O 블로킹) ← yield             │
│  ├─ 가상 스레드 2 (계산 중)  ← 실행                │
│  ├─ 가상 스레드 3 (I/O 블로킹) ← yield             │
│  └─ ... (수백 개 가상 스레드)                       │
│                                                     │
│ I/O 블로킹 발생 시:                                  │
│  가상 스레드 1의 I/O 대기 → carrier thread yield   │
│  → 다른 가상 스레드 2 스케줄링                     │
│  → OS 스케줄링 오버헤드 없음                       │
└─────────────────────────────────────────────────────┘

성능 비교 (10000개 작업, 100ms I/O 각):

플랫폼 스레드 (44개):
  - 메모리: 44 × 1MB = 44MB
  - 컨텍스트 스위칭: 많음 (~1ms × 44)
  - 총 시간: 100ms × (10000 / 44) ≈ 23초

가상 스레드 (unbounded):
  - 메모리: 10000 × 1KB = 10MB
  - 컨텍스트 스위칭: 없음 (OS 레벨)
  - 총 시간: 100ms (I/O 병렬)
```

---

## 💻 실전 실험

### 실험 1: commonPool 스레드 부족 확인

```java
@Test
public void testCommonPoolThreadShortage() throws Exception {
    long start = System.currentTimeMillis();
    
    List<CompletableFuture<Integer>> futures = new ArrayList<>();
    
    // 100개 작업, 각 100ms I/O
    for (int i = 0; i < 100; i++) {
        futures.add(CompletableFuture.supplyAsync(() → {
            try { Thread.sleep(100); } catch (InterruptedException e) {}
            return 1;
        }));  // 기본값: commonPool
    }
    
    CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).get();
    
    long duration = System.currentTimeMillis() - start;
    System.out.println("Duration: " + duration + "ms");
    
    // 4코어 = 4 스레드
    // 예상: 100 * 100ms / 4 ≈ 2500ms
    // 실제: ~2500ms
}

// 결과:
// Duration: ~2500ms
```

### 실험 2: 명시적 Executor로 개선

```java
@Test
public void testExplicitExecutorImprovement() throws Exception {
    ExecutorService ioExecutor = Executors.newFixedThreadPool(50);
    
    long start = System.currentTimeMillis();
    
    List<CompletableFuture<Integer>> futures = new ArrayList<>();
    
    for (int i = 0; i < 100; i++) {
        futures.add(CompletableFuture.supplyAsync(() → {
            try { Thread.sleep(100); } catch (InterruptedException e) {}
            return 1;
        }, ioExecutor));  // 명시적 executor
    }
    
    CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).get();
    
    long duration = System.currentTimeMillis() - start;
    System.out.println("Duration: " + duration + "ms");
    
    ioExecutor.shutdown();
    
    // 50개 스레드
    // 예상: 100 * 100ms / 50 ≈ 200ms
    // 실제: ~200ms
}

// 결과:
// Duration: ~200ms
```

### 실험 3: Virtual Thread 성능

```java
@Test
public void testVirtualThreadPerformance() throws Exception {
    ExecutorService virtualExecutor = 
        Executors.newVirtualThreadPerTaskExecutor();
    
    long start = System.currentTimeMillis();
    
    List<CompletableFuture<Integer>> futures = new ArrayList<>();
    
    for (int i = 0; i < 100; i++) {
        futures.add(CompletableFuture.supplyAsync(() → {
            try { Thread.sleep(100); } catch (InterruptedException e) {}
            return 1;
        }, virtualExecutor));  // 가상 스레드
    }
    
    CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).get();
    
    long duration = System.currentTimeMillis() - start;
    System.out.println("Duration: " + duration + "ms");
    
    // 스레드 수 제한 없음
    // 예상: 100ms (모두 병렬)
    // 실제: ~100ms
}

// 결과:
// Duration: ~100ms (Java 19+ 필요)
```

---

## 📊 성능/비교

```
╔═════════════════════════════════════════════════════════════════════╗
║ Executor 전략별 성능 (100개 작업, 100ms I/O 각)                    ║
╠═════════════════════════════════════════════════════════════════════╣
║ 전략            │ 스레드 수 │ 메모리  │ 총 시간    │ 처리율       ║
╠═════════════════════════════════════════════════════════════════════╣
║ commonPool      │ 4        │ 4MB    │ 2500ms   │ 40 t/sec   ║
║ Fixed(50)      │ 50       │ 50MB   │ 200ms    │ 500 t/sec  ║
║ Virtual(unbnd) │ 100      │ 100KB  │ 100ms    │ 1000 t/sec ║
╚═════════════════════════════════════════════════════════════════════╝

메모리 사용:

플랫폼 스레드:
  스택 크기: 기본 ~1MB/thread
  50개 스레드 = ~50MB

가상 스레드:
  힙 메모리: ~1KB/thread
  1000개 스레드 = ~1MB
  10000개 스레드 = ~10MB

컨텍스트 스위칭:

commonPool (4스레드):
  - 100개 작업 동시 경합
  - 스위칭 횟수: 매우 빈번
  - 오버헤드: 큼

Fixed(50):
  - 50개 스레드 중 일부 유휴
  - 스위칭 횟수: 중간 정도
  - 오버헤드: 중간

Virtual:
  - 가상 스레드 = OS 스레드 아님
  - 스위칭: I/O 레벨에서만 (OS 관여 안 함)
  - 오버헤드: 거의 없음
```

---

## ⚖️ 트레이드오프

| 전략 | 장점 | 단점 | 사용 시기 |
|------|------|------|---------|
| **commonPool** | 설정 불필요 | 스레드 부족, 경합 | 프로토타입 |
| **Fixed Pool** | 예측 가능, 제어 | 메모리 사용, 관리 필요 | 프로덕션 (Java <19) |
| **Virtual** | 확장성, 메모리 효율 | Java 19+ 필요 | 프로덕션 (Java 19+) |

---

## 📌 핵심 정리

1. **commonPool 기본값**: CPU 코어 수만큼 스레드 → I/O 작업에 부족

2. **starvation**: parallelStream + CompletableFuture.join() → nested parallelism → 데드락

3. **I/O-bound 공식**: 스레드 수 = N × (1 + W/C), N=코어수, W/C=I/O대기/계산

4. **명시적 Executor**: `thenApplyAsync(fn, executor)` 사용으로 commonPool 회피

5. **Virtual Thread**: Java 19+ 이상, `newVirtualThreadPerTaskExecutor()` 사용으로 수천 개 동시 I/O 처리

6. **프로덕션 설계**: commonPool 의존성 제거, I/O-bound용 별도 풀 구성

7. **모니터링**: 스레드 풀 크기, 큐 길이, rejection 수를 메트릭으로 수집

---

## 🤔 생각해볼 문제

<details>
<summary>Q1: commonPool parallelism을 JVM 옵션으로 증가시킬 수 있는가?</summary>

**해설**: 가능하지만 권장되지 않는다.

```java
// -Djava.util.concurrent.ForkJoinPool.common.parallelism=50

System.setProperty("java.util.concurrent.ForkJoinPool.common.parallelism", "50");

int parallelism = ForkJoinPool.getCommonPoolParallelism();
System.out.println(parallelism);  // 50

// 장점: commonPool 스레드 증가
// 단점: 
//   1. 전역 설정 (parallelStream에도 영향)
//   2. 메모리 증가 (50MB)
//   3. 컨텍스트 스위칭 오버헤드
//   4. 다른 애플리케이션과 공유 시 경합

// 결론: 명시적 Executor 사용이 낫다
```

</details>

<details>
<summary>Q2: Virtual Thread에서 synchronized를 사용하면?</summary>

**해설**: Carrier thread 바운딩 발생, 성능 저하.

```java
ExecutorService virtualExecutor = 
    Executors.newVirtualThreadPerTaskExecutor();

Object lock = new Object();

for (int i = 0; i < 1000; i++) {
    CompletableFuture.supplyAsync(() → {
        synchronized (lock) {  // ← 문제
            // carrier thread 바운딩
            return heavyComputation();
        }
    }, virtualExecutor);
}

// synchronized 블록에 들어가면,
// JVM은 가상 스레드를 carrier thread에 바운딩
// → 가상 스레드의 장점 상실

// 해결: ReentrantLock 또는 다른 동기화 방식 사용
ReentrantLock lock2 = new ReentrantLock();
lock2.lock();
try {
    return heavyComputation();
} finally {
    lock2.unlock();
}
```

</details>

<details>
<summary>Q3: W/C 공식이 정확하지 않을 수 있지 않은가?</summary>

**해설**: 맞다. W/C는 평균값이고, 실제 성능은 패턴에 따라 달라진다.

```
문제점:

1. W/C는 평균값
   - 네트워크 지연: 50ms ~ 5000ms (편차 큼)
   - 공식: N × (1 + 100) 계산
   - 실제: 일부는 빠르고 일부는 느림

2. CPU 캐시 효율
   - 스레드 수가 많으면 L1/L2 캐시 미스 증가
   - 컨텍스트 스위칭 비용

3. Amdahl의 법칙
   - 동기화/락으로 인한 직렬화 부분
   - parallelism 한계

권장사항:
  1. 초기값: N × (1 + W/C)
  2. 모니터링: 스레드 풀 통계 수집
  3. 조정: 부하 테스트로 최적값 찾기
  4. Virtual Thread 검토 (미래 가능성)
```

</details>

<div align="center">

**[⬅️ 이전: allOf · anyOf](./06-allof-anyof-patterns.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: Default Method 동작 원리 ➡️](../chapter06-interface-evolution/01-default-method-internals.md)**

</div>
