# Async 변형 — thenApplyAsync와 Executor 지정 전략

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `thenApply`와 `thenApplyAsync`의 실행 스레드는 어떻게 다른가?
- `commonPool()` 공유 사용 시 발생하는 starvation 문제는?
- `thenApplyAsync(fn, executor)`로 전용 Executor를 지정할 때의 장점은?
- IO-bound vs CPU-bound 작업의 스레드 풀 설계는?
- Virtual Thread (Project Loom) 통합 시 Executor 전략은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

CompletableFuture의 async 변형이 기본값으로 `ForkJoinPool.commonPool()`을 사용하면, `parallelStream()`과 풀을 공유하게 된다. 만약 CompletableFuture 작업이 무거우면 parallelStream의 성능이 저하된다. 반대로 async 변형을 명시적 Executor 없이 사용하면 기본 스레드 수가 부족해 높은 latency가 발생한다. 올바른 Executor 설계는 비동기 성능의 핵심이다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: supplyAsync() 기본값을 신뢰
  // commonPool()은 CPU 코어 수 = 스레드 수
  // 4코어 머신 = 4개 스레드만
  CompletableFuture.supplyAsync(() → {
      // 네트워크 I/O (1초 대기)
      return fetchDataFromAPI();
  });
  // 4개 스레드로 1000개 비동기 작업 처리?
  // 평균 대기: 1000 / 4 = 250초 (!!!)

실수 2: parallelStream과 CompletableFuture 함께 사용
  List<Item> items = ...;
  
  // 둘 다 commonPool() 공유
  items.parallelStream()
      .map(item → 
          CompletableFuture.supplyAsync(() → expensive(item))
              .get()  // 블로킹
      )
      .collect(toList());
  
  // 결과: commonPool 스레드 고갈
  // ForkJoinTask 중첩으로 인한 starvation

실수 3: 동기 작업을 async 변형으로 감싸기
  // 불필요한 오버헤드
  cf.thenApplyAsync(value → value * 2)
    // 동기 계산인데 별도 스레드에 submit
    // executor 스레드 낭비
  
  → 동기면 thenApply 사용

실수 4: 모든 Executor를 무제한 스레드 풀로 설계
  ExecutorService executor = Executors.newCachedThreadPool();
  // 또는 newFixedThreadPool(1000);
  
  // 메모리: 스레드당 ~1MB
  // 1000개 스레드 = ~1GB 메모리
  // 컨텍스트 스위칭 오버헤드 증가
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 1. 동기 vs 비동기 선택
CompletableFuture<Integer> cf = CompletableFuture.completedFuture(10);

// 동기 (계산 작업, 빠름)
cf.thenApply(v → v * 2);

// 비동기 (I/O 작업, 느림)
cf.thenApplyAsync(v → fetchData(v));

// 2. Executor 명시적 지정
ExecutorService ioExecutor = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors() * 2
);

cf.thenApplyAsync(v → {
    // I/O 작업: DB 쿼리, API 호출 등
    return database.query(v);
}, ioExecutor);

// 3. CPU-bound vs IO-bound 분리
// CPU-bound: 계산 집약적
ExecutorService cpuExecutor = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors()
);

// IO-bound: I/O 대기
ExecutorService ioExecutor2 = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors() * (1 + W/C)
    // W/C = I/O 대기 시간 / 계산 시간
    // 네트워크: W/C ≈ 10~100
);

// 4. parallelStream과 CompletableFuture 분리
List<Item> items = ...;

// 방식 1: async 작업만 수행 (get() 제거)
List<CompletableFuture<Result>> futures = items.stream()
    .map(item → CompletableFuture.supplyAsync(() → expensive(item), ioExecutor))
    .collect(toList());

// 방식 2: 모든 future 완료 대기
CompletableFuture<List<Result>> allDone = CompletableFuture.allOf(
    futures.toArray(new CompletableFuture[0])
).thenApply(v → futures.stream().map(CF::join).collect(toList()));

// 5. Virtual Thread 활용 (Java 19+)
ExecutorService virtualThreadExecutor = Executors.newVirtualThreadPerTaskExecutor();

cf.thenApplyAsync(v → {
    // 수천 개의 가상 스레드로 I/O 작업 효율적 처리
    return slowIO(v);
}, virtualThreadExecutor);
```

---

## 🔬 내부 동작 원리

### 1. thenApply vs thenApplyAsync 실행 스레드

```java
// OpenJDK CompletableFuture.java

public <U> CompletableFuture<U> thenApply(Function<? super T, ? extends U> fn) {
    return uniApplyStage(null, fn);  // null = 기본 executor 없음
}

public <U> CompletableFuture<U> thenApplyAsync(Function<? super T, ? extends U> fn) {
    return uniApplyStage(asyncExecutor, fn);  // asyncExecutor 사용
}

public <U> CompletableFuture<U> thenApplyAsync(
    Function<? super T, ? extends U> fn, Executor executor) {
    return uniApplyStage(executor, fn);  // 명시적 executor
}

// uniApplyStage 구현
private <V> CompletableFuture<V> uniApplyStage(
    Executor e, Function<? super T, ? extends V> f) {
    
    if (f == null) throw new NullPointerException();
    CompletableFuture<V> d = new CompletableFuture<>();
    
    // UniApply 콜백 생성
    Object r;
    if ((r = result) == null) {
        // 아직 미완료: 스택에 추가
        unipush(new UniApply<T, V>(e, d, this, f));
    } else {
        // 이미 완료: 콜백 바로 실행
        // e가 null이면 현재 스레드, 아니면 e.execute()로 submit
        uniApplyNow(r, d, f, e);
    }
    return d;
}

// uniApplyNow: 콜백 실행 방식 결정
private static <T,V> void uniApplyNow(
    Object r, CompletableFuture<V> d,
    Function<? super T, ? extends V> f, Executor e) {
    
    T t = (T) r;
    V v;
    try {
        if (e != null) {
            // 비동기: executor에 submit
            e.execute(() → {
                try {
                    v = f.apply(t);
                    d.completeValue(v);
                } catch (Throwable ex) {
                    d.completeExceptionally(ex);
                }
            });
        } else {
            // 동기: 현재 스레드에서 즉시 실행
            v = f.apply(t);
            d.completeValue(v);
        }
    } catch (Throwable ex) {
        d.completeExceptionally(ex);
    }
}

실행 스레드:

thenApply(fn):
  ┌─ complete() 호출 스레드
  ├─ fn 실행
  └─ complete() 스레드에서 콜백 실행
  
  → 같은 스레드 (블로킹 없음)

thenApplyAsync(fn):
  ┌─ complete() 호출 스레드
  └─ e.execute(fn)
     └─ executor의 스레드 풀에서 fn 실행
  
  → 별도 스레드 (블로킹 가능)

thenApplyAsync(fn, myExecutor):
  ┌─ complete() 호출 스레드
  └─ myExecutor.execute(fn)
     └─ myExecutor의 스레드에서 fn 실행
  
  → 사용자 지정 스레드 (풀 선택)
```

### 2. ForkJoinPool.commonPool() 구조

```
┌─────────────────────────────────────────────────────────────┐
│ ForkJoinPool.commonPool()                                   │
├─────────────────────────────────────────────────────────────┤
│ 특징:                                                         │
│  1. 공유 풀 (전역 싱글톤)                                     │
│  2. 기본 스레드 수:                                            │
│     parallelism = Runtime.getRuntime().availableProcessors()│
│     (최소 1, 일반적으로 CPU 코어 수)                          │
│  3. Work-Stealing 알고리즘                                   │
│     → 유휴 스레드가 다른 스레드의 작업 획득 (FIFO 해서)     │
│                                                             │
│ 사용처:                                                       │
│  - parallelStream()                                         │
│  - CompletableFuture.supplyAsync() (executor 미지정 시)     │
│  - RecursiveTask / RecursiveAction (ForkJoin)              │
│                                                             │
│ 문제:                                                         │
│  - CPU 코어 수만큼만 스레드 → I/O 대기 시 비효율적         │
│  - 여러 작업이 공유 → 한 작업이 다른 작업 블로킹           │
└─────────────────────────────────────────────────────────────┘

스레드 수 계산:

CPU-bound (계산 집약적):
  최적 스레드 수 = CPU 코어 수
  예: 4코어 = 4개 스레드
  
IO-bound (I/O 대기):
  최적 스레드 수 = CPU 코어 수 × (1 + W/C)
  W/C = 평균 I/O 대기 시간 / 평균 계산 시간
  
  예 1: 네트워크 API (대기 1초, 처리 0.1초)
    = 4 × (1 + 1/0.1) = 4 × 11 = 44개 스레드
  
  예 2: 메모리 캐시 조회 (대기 0.001초, 처리 0.01초)
    = 4 × (1 + 0.001/0.01) = 4 × 1.1 = 4~5개 스레드

commonPool vs 전용 Executor:

  commonPool (parallelism = 4):
    ┌─ parallelStream
    ├─ CF.supplyAsync (1)
    ├─ CF.supplyAsync (2)
    └─ CF.supplyAsync (3)
    
    [Thread 1] [Thread 2] [Thread 3] [Thread 4]
    │          │          │          │
    ├─ PS      ├─ CF(1)    ├─ CF(2)   ├─ CF(3)
    │          │          │          │
    → 4개 작업 경쟁, I/O 대기 중 스레드 낭비

  전용 Executor (parallelism = 20):
    ┌─ parallelStream (Thread 1~4)
    ├─ ioExecutor (Thread 5~20)
    
    [1] [2] [3] [4] [5] [6]...[20]
    ├─ PS              ├─ CF(1) CF(2) ... CF(16)
    │                  │
    → parallelStream과 async 작업 독립 실행
```

### 3. Executor 선택 전략

```
┌─────────────────────────────────────────────────────────┐
│ I/O-bound Executor 설계 (권장)                            │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ ExecutorService ioExecutor = Executors.newFixedThreadPool(
│     Math.max(4, Runtime.getRuntime().availableProcessors() * 2)
│ );
│                                                         │
│ 또는:                                                     │
│                                                         │
│ ThreadPoolExecutor ioExecutor = new ThreadPoolExecutor(
│     10,  // corePoolSize (항상 유지)                    │
│     50,  // maxPoolSize (필요시 확장)                  │
│     60, TimeUnit.SECONDS,  // keepAliveTime            │
│     new LinkedBlockingQueue<>(1000),  // 큐 크기        │
│     new ThreadFactory() { ... }  // 스레드 팩토리       │
│ );
│                                                         │
│ 특징:                                                     │
│  - corePoolSize: 항상 유지되는 스레드                   │
│  - maxPoolSize: 큐 가득 시 추가 생성                   │
│  - keepAliveTime: 추가 스레드의 유휴 시간               │
│  - 큐: 작업 임시 저장                                    │
│                                                         │
└─────────────────────────────────────────────────────────┘

Virtual Thread 활용 (Java 19+):

  ExecutorService virtualExecutor = 
      Executors.newVirtualThreadPerTaskExecutor();
  
  cf.thenApplyAsync(v → slowIO(v), virtualExecutor);
  
  장점:
    - 스레드당 메모리 ~1KB (플랫폼 스레드 ~1MB)
    - 수천 개의 가상 스레드 생성 가능
    - I/O 대기 시 자동 yield (효율적)
    - OS 스케줄링 오버헤드 없음
  
  단점:
    - Java 19+ 필요
    - 일부 라이브러리 미지원
```

---

## 💻 실전 실험

### 실험 1: thenApply vs thenApplyAsync 스레드 비교

```java
@Test
public void testThreadExecutionDifference() throws Exception {
    CompletableFuture<Integer> cf = CompletableFuture.completedFuture(42);
    
    String syncThread = new CompletableFuture<String>() {{
        cf.thenApply(v → {
            completeValue(Thread.currentThread().getName());
            return null;
        });
    }}.get();
    
    String asyncThread = new CompletableFuture<String>() {{
        cf.thenApplyAsync(v → {
            completeValue(Thread.currentThread().getName());
            return null;
        });
    }}.get();
    
    System.out.println("thenApply thread: " + syncThread);
    // main (콜백 등록 스레드)
    
    System.out.println("thenApplyAsync thread: " + asyncThread);
    // ForkJoinPool-1-worker-1 (commonPool 스레드)
}

// 결과:
// thenApply thread: main
// thenApplyAsync thread: ForkJoinPool-1-worker-1
```

### 실험 2: commonPool starvation 시연

```java
@Test
public void testCommonPoolStarvation() throws Exception {
    long start = System.currentTimeMillis();
    
    List<CompletableFuture<Integer>> futures = new ArrayList<>();
    
    // 20개의 느린 비동기 작업 (각 1초)
    for (int i = 0; i < 20; i++) {
        futures.add(CompletableFuture.supplyAsync(() → {
            try { Thread.sleep(1000); } catch (InterruptedException e) {}
            return 1;
        }));
    }
    
    // commonPool이 4개 스레드뿐이면:
    // 병렬도 = 4
    // 20개 작업 / 4 스레드 = 5 라운드
    // 총 시간: 5 × 1000ms = 5000ms
    
    CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).get();
    
    long duration = System.currentTimeMillis() - start;
    System.out.println("Duration: " + duration + "ms");
    // ~5000ms (4코어 기준)
}

// 해결: 전용 Executor 사용
ExecutorService executor = Executors.newFixedThreadPool(20);

List<CompletableFuture<Integer>> futures2 = new ArrayList<>();
for (int i = 0; i < 20; i++) {
    futures2.add(CompletableFuture.supplyAsync(() → {
        try { Thread.sleep(1000); } catch (InterruptedException e) {}
        return 1;
    }, executor));  // ← executor 지정
}

// 병렬도 = 20 스레드
// 20개 작업 / 20 스레드 = 1 라운드
// 총 시간: 1000ms
```

### 실험 3: Executor 스레드 풀 크기의 영향

```java
@Test
public void testExecutorPoolSizeImpact() throws Exception {
    int taskCount = 50;
    long ioTime = 100;  // 각 작업 100ms I/O 대기
    
    // 풀 크기별 성능 비교
    for (int poolSize : Arrays.asList(4, 10, 50)) {
        ExecutorService executor = Executors.newFixedThreadPool(poolSize);
        
        long start = System.currentTimeMillis();
        
        List<CompletableFuture<Integer>> futures = new ArrayList<>();
        for (int i = 0; i < taskCount; i++) {
            futures.add(CompletableFuture.supplyAsync(() → {
                try { Thread.sleep(ioTime); } catch (InterruptedException e) {}
                return 1;
            }, executor));
        }
        
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).get();
        
        long duration = System.currentTimeMillis() - start;
        double throughput = (taskCount * 1000.0) / duration;
        
        System.out.printf("Pool size %d: %dms, %.1f tasks/sec%n",
            poolSize, duration, throughput);
        
        executor.shutdown();
    }
}

// 결과 (예상):
// Pool size 4: 1250ms, 40.0 tasks/sec
// Pool size 10: 500ms, 100.0 tasks/sec
// Pool size 50: 100ms, 500.0 tasks/sec
```

---

## 📊 성능/비교

```
╔════════════════════════════════════════════════════════════════════╗
║ Executor 선택별 성능 (50개 작업, 100ms I/O 각)                     ║
╠════════════════════════════════════════════════════════════════════╣
║ Executor         │ 스레드 수 │ 총 시간     │ 처리율        │      ║
╠════════════════════════════════════════════════════════════════════╣
║ commonPool (4)   │ 4        │ 1250ms    │ 40 tasks/sec │      ║
║ Fixed(10)       │ 10       │ 500ms     │ 100 tasks/sec│      ║
║ Fixed(50)       │ 50       │ 100ms     │ 500 tasks/sec│      ║
║ Virtual(unbounded)│ ∞       │ 100ms     │ 500 tasks/sec│      ║
╚════════════════════════════════════════════════════════════════════╝

메모리 사용:

평탄폼 스레드:
  스레드당 ~1MB (스택)
  10개 스레드 = ~10MB
  50개 스레드 = ~50MB
  1000개 스레드 = ~1GB

가상 스레드:
  스레드당 ~1KB (힙 메모리)
  10000개 스레드 = ~10MB
  1000000개 스레드 = ~1GB (가능, 하지만 비권장)

컨텍스트 스위칭:

commonPool (4스레드, 50개 작업):
  활성 스레드 = 4
  스위칭 횟수 = 적음

Fixed(50, 50개 작업):
  활성 스레드 = 50
  스위칭 횟수 = 중간

Virtual(50개 작업):
  활성 스레드 = 1 (OS 관점)
  스위칭 횟수 = 0 (OS 레벨)
```

---

## ⚖️ 트레이드오프

| 전략 | 장점 | 단점 | 사용 시기 |
|------|------|------|---------|
| **thenApply (동기)** | 빠름, 메모리 절약 | 블로킹 가능 | 계산 작업, 빠른 변환 |
| **commonPool** | 간단, 추가 설정 없음 | 스레드 부족, 공유 경합 | 프로토타입, 테스트 |
| **고정 스레드 풀** | 예측 가능, 제어 가능 | 스레드 관리 필요 | 프로덕션 I/O 작업 |
| **가상 스레드** | 확장성, 메모리 효율 | Java 19+ 필요, 라이브러리 지원 | 대량 동시 I/O (미래) |

---

## 📌 핵심 정리

1. **thenApply**: 동기 실행 (현재 스레드), 빠름, 블로킹 가능

2. **thenApplyAsync**: 기본값 commonPool, CPU 코어 수만큼만 스레드

3. **commonPool starvation**: I/O 대기 중 스레드 낭비, parallelStream과 경합

4. **전용 Executor**: IO-bound용 풀 크기 = CPU 코어 × (1 + W/C)

5. **Executor 지정**: `thenApplyAsync(fn, executor)`로 명시적 제어

6. **Virtual Thread**: Java 19+ 이상에서 스레드 수 무제한에 가까운 확장성

7. **실무 설계**: CPU 작업 = commonPool, I/O 작업 = 전용 고정 풀

---

## 🤔 생각해볼 문제

<details>
<summary>Q1: 왜 CompletableFuture는 기본값으로 commonPool을 사용하는가?</summary>

**해설**: 대부분의 서버 애플리케이션에서 직접 CPU 계산 작업을 CompletableFuture로 처리하지 않기 때문이다. 주로 I/O 결과를 콜백으로 처리하는 경우가 많고, 그 콜백들은 보통 빠르다 (밀리초 단위). 따라서 공유 풀도 충분했다.

하지만 현대 마이크로서비스에서는 CompletableFuture로 다양한 I/O 작업을 비동기로 처리하므로, commonPool 기본값은 좋은 선택이 아니다. **따라서 프로덕션에서는 항상 명시적 Executor를 지정해야 한다.**

</details>

<details>
<summary>Q2: ThreadPoolExecutor의 corePoolSize vs maxPoolSize는?</summary>

**해설**: 

- **corePoolSize**: 항상 유지되는 스레드. 작업이 없어도 살아있음. 예: 10
- **maxPoolSize**: 최대 스레드 개수. 큐가 가득 차면 추가 스레드 생성. 예: 50
- **큐 크기**: corePoolSize < 현재 스레드 < maxPoolSize 사이에 작업 대기

동작 순서:
1. 현재 스레드 < corePoolSize → 즉시 새 스레드 생성
2. 현재 스레드 >= corePoolSize → 큐에 추가
3. 큐 가득 → maxPoolSize까지 스레드 생성
4. 현재 스레드 > corePoolSize → keepAliveTime 후 제거

```java
// 예: 최대 50개 작업 처리
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    10,  // 항상 10개 스레드 유지
    50,  // 최대 50개 스레드
    60, TimeUnit.SECONDS,  // 추가 스레드는 60초 유휴 후 종료
    new LinkedBlockingQueue<>(1000)  // 1000개 작업까지 대기 가능
);
```

</details>

<details>
<summary>Q3: 가상 스레드는 모든 상황에서 완벽한가?</summary>

**해설**: 아니다. 몇 가지 제약이 있다:

1. **네이티브 코드 호출**: Virtual Thread는 중단 불가능한 네이티브 코드 실행 중 JVM 스레드 바운딩 (성능 저하)
2. **Synchronized 블록**: Virtual Thread에서 synchronized 사용 시 carrier thread 바운딩
3. **라이브러리 미지원**: 일부 오래된 라이브러리는 가상 스레드 미지원
4. **디버깅**: 수천 개의 가상 스레드 스택 추적 어려움

따라서 Virtual Thread는 "I/O-heavy, sync-free" 워크로드에서만 최적이다.

</details>

<div align="center">

**[⬅️ 이전: thenRun vs thenAccept vs thenApply](./03-thenrun-thenaccept-thenapply.md)** | **[홈으로 🏠](../README.md)** | **[다음: 예외 처리 3가지 ➡️](./05-exception-handling-three-ways.md)**

</div>
