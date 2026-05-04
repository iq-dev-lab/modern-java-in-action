# Common Pool 의존성 — commonPool()이 위험한 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `parallelStream()`과 `CompletableFuture`의 기본 Executor가 모두 `ForkJoinPool.commonPool()`을 공유하는 구조는?
- 한 곳에서 pool을 점유하면 전체 애플리케이션이 영향받는 시나리오는?
- `commonPool()`의 병렬도는 어떻게 결정되고, 이를 조정할 수 있는가?
- 격리된 ForkJoinPool 또는 전용 Executor를 분리하는 패턴은?
- IO-bound 작업을 commonPool에서 처리하면 왜 위험한가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Spring Boot 애플리케이션에서 `Stream.parallelStream()`과 `CompletableFuture.supplyAsync()`를 동시에 사용하면, 둘 다 같은 `ForkJoinPool.commonPool()`을 공유한다. 한 곳에서 pool의 worker를 모두 점유하면, 다른 곳의 병렬 작업이 멈춘다. 특히 IO-bound 작업(DB 쿼리, HTTP 호출)을 commonPool에서 처리하면, CPU-bound 병렬 처리가 무한 대기하는 교착 상태가 발생할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: parallelStream()과 CompletableFuture를 자유롭게 사용
  // 서로 다른 Executor를 사용한다고 착각
  List<Data> data = ...;
  
  CompletableFuture.supplyAsync(() -> {  // commonPool 사용
      return fetchFromDB();  // IO 대기 → worker 블로킹
  }).thenApply(result -> {
      return data.parallelStream()  // commonPool 사용
          .map(this::heavyCompute)
          .collect(toList());
  });
  
  // 결과: DB 조회로 commonPool이 마모, 
  //      parallelStream은 worker 부족으로 대기 → 교착

실수 2: commonPool()의 병렬도를 무시
  "기본 설정이면 충분하겠지"
  → 병렬도 = max(2, CPU 코어 - 1)
  → 4코어 머신: 병렬도 3개
  → 요청 수가 병렬도를 초과하면 즉시 queue 대기
  → latency 증가

실수 3: CompletableFuture의 기본 executor 몰라주기
  CompletableFuture.supplyAsync(fn);  // 어디서 실행?
  → commonPool (명시하지 않으면 기본값)
  → 별도 executor를 명시하지 않으면 공유 pool에 의존

실수 4: nested parallelStream
  data.parallelStream()
      .map(item -> item.getChildren()
          .parallelStream()  // 또 다른 parallelStream!
          .map(...)
          .collect(toList()))
      .collect(toList());
  
  // 외부 parallelStream의 task 중에 
  // 내부 parallelStream이 fork()
  // commonPool의 worker 모두 바쁜 상태에서
  // 내부 task가 join() 대기 → deadlock
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// Common Pool 의존성 해결 패턴

import java.util.concurrent.*;

// 패턴 1: IO-bound 작업용 별도 Executor
ExecutorService ioExecutor = Executors.newFixedThreadPool(
    Math.max(2, Runtime.getRuntime().availableProcessors() * 2)
);

CompletableFuture.supplyAsync(
    () -> {
        // DB 조회, HTTP 호출 등 IO 작업
        return fetchFromDatabase();
    },
    ioExecutor  // 명시적으로 ioExecutor 지정
).thenApplyAsync(result -> {
    // CPU-bound 작업은 commonPool 사용 가능
    return data.parallelStream()
        .map(this::heavyCompute)
        .collect(toList());
})
.exceptionally(ex -> {
    logger.error("작업 실패", ex);
    return null;
})
.whenComplete((result, ex) -> {
    // ioExecutor는 명시적으로 종료 필요
});

// 패턴 2: CPU-bound 작업용 격리된 ForkJoinPool
ForkJoinPool cpuPool = new ForkJoinPool(
    Runtime.getRuntime().availableProcessors()
);

// 다른 스레드에서 실행하지 않음 (deadlock 위험)
Integer result = cpuPool.invoke(
    new MergeSortTask(array, 0, array.length)
);

// 패턴 3: 서로 다른 작업 분리
// - parallelStream: CPU-bound (정렬, 수학 계산)
// - CompletableFuture(ioExecutor): IO-bound (DB, 네트워크)

ForkJoinPool.getCommonPoolParallelism();  // 병렬도 확인

// 패턴 4: Nested parallelStream 피하기 (또는 주의)
data.parallelStream()
    .map(item -> {
        // parallelStream 대신 순차 처리
        return item.getChildren().stream()  // 순차!
            .map(this::transform)
            .collect(toList());
    })
    .collect(toList());

// 또는 flatten할 때 flatMap 사용
data.parallelStream()
    .flatMap(item -> item.getChildren().stream())  // 순차
    .map(this::transform)
    .collect(toList());

// 패턴 5: commonPool 병렬도 조정 (JVM 시작 시)
// java -Djava.util.concurrent.ForkJoinPool.common.parallelism=8
```

---

## 🔬 내부 동작 원리

### 1. commonPool() 초기화

```java
// ForkJoinPool.java 내부 (단순화)

public static ForkJoinPool commonPool() {
    ForkJoinPool p = common;
    if (p == null) {
        // 지연 초기화 (첫 사용 시)
        common = p = new ForkJoinPool(
            defaultCommonMax,  // 병렬도 결정
            defaultFactory,
            null, false
        );
    }
    return p;
}

병렬도 계산:
  defaultCommonMax = max(2, 
    Runtime.getRuntime().availableProcessors() - 1
  );
  
  예시:
    1코어: max(2, 0) = 2
    4코어: max(2, 3) = 3
    8코어: max(2, 7) = 7
    16코어: max(2, 15) = 15

병렬도 조정 (JVM 시작 시):
  java -Djava.util.concurrent.ForkJoinPool.common.parallelism=16
  
  실행 중 조정은 불가능 (초기화 후 변경 불가)
```

### 2. parallelStream과 CompletableFuture의 pool 공유

```
parallelStream() 내부:

Stream.parallel()
  → ForkJoinTask의 subclass 생성
  → ForkJoinPool.commonPool() 사용
  → collect(toList()) 시 invoke() 호출

CompletableFuture의 기본 executor:

CompletableFuture.supplyAsync(fn)
  → asyncExecution(fn, defaultExecutor)
  → defaultExecutor = ForkJoinPool.commonPool()

둘 다 동일 pool 공유:

┌────────────────────────────────────┐
│  ForkJoinPool.commonPool()         │
│  parallelism = 7 (8코어 기준)      │
│  ┌──────────────────────────────┐  │
│  │ WorkQueue[1], [3], [5], ...  │  │
│  │ (7개 worker)                  │  │
│  └──────────────────────────────┘  │
└────────────────────────────────────┘
        ↑                 ↑
   parallelStream    CompletableFuture
   (동일 pool)       (동일 pool)

한 쪽이 모든 worker 점유 → 다른 쪽 대기
```

### 3. Deadlock 시나리오

```
Scenario: parallelStream + CompletableFuture 교착

Time | parallelStream Worker | CompletableFuture Task
─────┼─────────────────────┼──────────────────────
t0   | task1.fork()        |
     | task2.fork()        |
     | task3.fork()        |
     | task4.fork()        |
     | (모든 worker 바쁨)    |
─────┼─────────────────────┼──────────────────────
t1   |                     | supplyAsync(fetchDB)
     |                     | → worker 없음 → 대기
─────┼─────────────────────┼──────────────────────
t2   | join()              |
     | fetchDB 결과 필요    |
     | → 교착 상태!        |
─────┼─────────────────────┼──────────────────────

왜 이런가?
  parallelStream: compute() 중에 join()
  CompletableFuture: 다른 스레드에서 실행 필요
  → 둘 다 같은 pool → worker 부족 → deadlock
```

### 4. IO-bound 작업의 위험성

```
문제 시나리오: IO-bound를 commonPool에서

CompletableFuture.supplyAsync(() -> {
    return httpClient.get("http://api.example.com/data");  // 1초 대기
}, ForkJoinPool.commonPool());  // 잘못된 사용

결과:
  1개 worker가 1초 동안 IO 대기
  parallelism = 7이면, 7개 worker 중 1개는 블로킹
  7개 요청이 동시에 오면, 모든 worker 블로킹 됨
  
  → CPU-bound parallelStream은 worker 없어서 대기
  → latency 급증

최악의 시나리오: nested blocking
  CompletableFuture(IO1) ← supplyAsync(DB 조회)
    .thenApply(result →
        data.parallelStream()  ← CPU-bound 작업
            .map(...)
            .collect(...)
    );
  
  DB 조회가 commonPool 점유
  → parallelStream은 worker 부족
  → 진행 불가
```

---

## 💻 실전 실험

### 실험 1: Common Pool 고갈 데모

```java
import java.util.concurrent.*;
import java.util.stream.*;

public class CommonPoolExhaustionDemo {
    public static void main(String[] args) throws Exception {
        int parallelism = ForkJoinPool.getCommonPoolParallelism();
        System.out.println("Common Pool Parallelism: " + parallelism);

        // parallelStream으로 commonPool의 모든 worker 점유
        IntStream.range(0, parallelism).parallel()
            .forEach(i -> {
                try {
                    System.out.println("Task " + i + " 시작");
                    Thread.sleep(5000);  // 5초 대기
                    System.out.println("Task " + i + " 완료");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });

        System.out.println("parallelStream 완료");

        // 이 시점에 CompletableFuture 실행
        long start = System.currentTimeMillis();
        CompletableFuture<Void> cf = CompletableFuture.supplyAsync(() -> {
            System.out.println("CF 시작");
            return "완료";
        }).thenAccept(result -> {
            System.out.println("CF 결과: " + result);
        });

        cf.join();  // 매우 오래 대기 (parallelStream이 완료될 때까지)
        long elapsed = System.currentTimeMillis() - start;
        System.out.println("CF 소요 시간: " + elapsed + "ms");
    }
}

// 출력:
// Common Pool Parallelism: 7
// Task 0 시작
// Task 1 시작
// ...
// Task 6 시작
// (5초 대기)
// Task 0 완료
// ...
// parallelStream 완료
// CF 시작  ← 5초 후에야 시작
// CF 결과: 완료
// CF 소요 시간: ~5000ms (또는 그 이상)
```

### 실험 2: 격리된 Pool vs Common Pool 성능 비교

```java
import java.util.concurrent.*;

public class IsolatedPoolBenchmark {
    static class IntensiveTask extends RecursiveTask<Long> {
        final int[] array;
        final int start, end;
        static final int THRESHOLD = 10000;

        IntensiveTask(int[] array, int start, int end) {
            this.array = array;
            this.start = start;
            this.end = end;
        }

        @Override
        protected Long compute() {
            if (end - start <= THRESHOLD) {
                long sum = 0;
                for (int i = start; i < end; i++) sum += array[i];
                return sum;
            }
            int mid = (start + end) / 2;
            IntensiveTask left = new IntensiveTask(array, start, mid);
            IntensiveTask right = new IntensiveTask(array, mid, end);
            left.fork();
            return right.compute() + left.join();
        }
    }

    public static void main(String[] args) throws Exception {
        int[] largeArray = new int[10_000_000];
        for (int i = 0; i < largeArray.length; i++) largeArray[i] = 1;

        // Common Pool
        long start = System.nanoTime();
        long resultCommon = ForkJoinPool.commonPool()
            .invoke(new IntensiveTask(largeArray, 0, largeArray.length));
        long timeCommon = System.nanoTime() - start;

        // Isolated Pool
        ForkJoinPool isolatedPool = new ForkJoinPool(
            Runtime.getRuntime().availableProcessors()
        );
        start = System.nanoTime();
        long resultIsolated = isolatedPool
            .invoke(new IntensiveTask(largeArray, 0, largeArray.length));
        long timeIsolated = System.nanoTime() - start;
        isolatedPool.shutdown();

        System.out.println("Common Pool: " + (timeCommon / 1_000_000) + "ms");
        System.out.println("Isolated Pool: " + (timeIsolated / 1_000_000) + "ms");
        System.out.println("차이: " + ((timeCommon - timeIsolated) / 1_000_000) + "ms");
    }
}
```

### 실험 3: IO-bound 작업 분리 효과

```java
import java.util.concurrent.*;
import java.util.*;

public class IOBoundSeparationDemo {
    static ExecutorService ioExecutor = Executors.newFixedThreadPool(
        Math.max(2, Runtime.getRuntime().availableProcessors() * 2)
    );

    static List<Integer> fetchDataFromDB() throws InterruptedException {
        Thread.sleep(1000);  // DB 조회 시뮬레이션
        return Arrays.asList(1, 2, 3, 4, 5);
    }

    static Integer heavyCompute(int value) {
        int result = value;
        for (int i = 0; i < 100_000_000; i++) result ^= i;
        return result;
    }

    public static void main(String[] args) throws Exception {
        // 잘못된 방법: IO 작업을 commonPool에서
        long start = System.nanoTime();
        CompletableFuture<List<Integer>> badWay = CompletableFuture
            .supplyAsync(() -> {
                try { return fetchDataFromDB(); }
                catch (InterruptedException e) { throw new RuntimeException(e); }
            })  // commonPool 사용 (잘못!)
            .thenApply(data -> data.parallelStream()
                .map(IOBoundSeparationDemo::heavyCompute)
                .collect(toList())
            );
        badWay.join();
        long timeBad = System.nanoTime() - start;

        // 올바른 방법: IO 작업을 별도 Executor에서
        start = System.nanoTime();
        CompletableFuture<List<Integer>> goodWay = CompletableFuture
            .supplyAsync(() -> {
                try { return fetchDataFromDB(); }
                catch (InterruptedException e) { throw new RuntimeException(e); }
            }, ioExecutor)  // IO Executor 명시
            .thenApply(data -> data.parallelStream()
                .map(IOBoundSeparationDemo::heavyCompute)
                .collect(toList())
            );
        goodWay.join();
        long timeGood = System.nanoTime() - start;

        System.out.println("잘못된 방법: " + (timeBad / 1_000_000) + "ms");
        System.out.println("올바른 방법: " + (timeGood / 1_000_000) + "ms");
        ioExecutor.shutdown();
    }
}
```

---

## 📊 성능/비교

```
Common Pool 영향 시뮬레이션 (8코어, parallelism=7):

시나리오: parallelStream (5초) 후 CompletableFuture (IO 1초)

구분                      │ 시간(초) │ 설명
──────────────────────────┼─────────┼──────────────────────
1. 순차 처리               │ 6       │ 기준선
2. parallelStream (공유)   │ 5.5+1.5 │ parallelStream 5초 후
   + CF (commonPool)      │ = 7.0   │ CF는 worker 대기로 1.5초
3. parallelStream (공유)   │ 5 + 1   │ 최적 (worker 충분)
   + CF (격리, ioExecutor)│ = 5.1   │ CF는 병렬 진행
4. 두 작업 동시진행        │ 5       │ 최적 (IO와 CPU 병렬)
   (별도 executor)        │ = 5.0   │

CF의 IO 대기 시간에 parallelStream이 언제 끝나는지에 따라 달라짐:

Case A: CF가 먼저 시작 (IO 블로킹)
  CF (1초) + parallelStream (5초) = 6초

Case B: parallelStream이 먼저 진행
  parallelStream (5초) + CF 대기 (? 초) = 5+ ?초
  → CF가 worker 없어서 대기 → 최악 6~7초
```

---

## ⚖️ 트레이드오프

```
Common Pool 공유:
  장점: 단순성 (명시적 executor 관리 불필요)
  단점: 간섭 (다른 작업이 영향받음)
       예측 불가능한 지연
       IO-bound 혼재 시 deadlock 위험

격리된 Pool 사용:
  장점: 격리 (다른 작업 영향 없음)
       예측 가능한 성능
  단점: 메모리 오버헤드 (pool 복수 생성)
       명시적 shutdown 필요
       설정 복잡성

IO Executor 분리:
  장점: 스레드 블로킹 격리
       CPU-bound 작업 보호
  단점: 스레드 수 관리 필요
       context switching 비용
```

---

## 📌 핵심 정리

```
Common Pool 의존성 핵심:

문제:
  parallelStream과 CompletableFuture 모두
  ForkJoinPool.commonPool() 공유
  
병렬도:
  max(2, CPU_CORES - 1)
  → 4코어: 3개 worker
  → 8코어: 7개 worker
  → 요청이 병렬도를 초과하면 즉시 대기

위험:
  한 곳에서 pool 점유 → 다른 곳 지연
  IO-bound + CPU-bound 혼재 → deadlock 위험
  nested parallelStream → 교착 상태

해결책:
  IO 작업: ExecutorService 분리
  CPU-bound: 격리된 ForkJoinPool
  명시적 executor 지정
  병렬도 조정 (JVM 옵션)
```

---

## 🤔 생각해볼 문제

**Q1.** `ForkJoinPool.commonPool()`의 병렬도가 CPU 코어 수에서 1을 뺀 이유는?

<details>
<summary>해설 보기</summary>

`max(2, CPU_CORES - 1)`의 설계 의도:

1. **Main 스레드 예약**: 애플리케이션의 main 스레드가 IO 대기 등으로 CPU를 사용할 여지를 남겨둔다. 만약 모든 코어를 ForkJoinPool worker로 할당하면, main 스레드나 다른 스레드가 실행할 여지가 없다.

2. **System 스레드 고려**: GC, 타이머, 신호 처리 등 JVM의 시스템 스레드들도 CPU 시간이 필요하다.

3. **컨텍스트 스위칭 오버헤드**: 코어 수와 정확히 같은 스레드 수를 사용하면 컨텍스트 스위칭이 많아진다.

따라서 (코어 수 - 1)이 합리적인 휴리스틱이다. 단, 최소값 2는 이중 코어 시스템에서도 어느 정도 병렬성을 보장한다.

</details>

---

**Q2.** Common Pool을 직접 종료(shutdown)할 수 있는가?

<details>
<summary>해설 보기</summary>

아니다. `ForkJoinPool.commonPool()`은 **직접 종료할 수 없다**.

```java
ForkJoinPool.commonPool().shutdown();  // 효과 없음
```

이유:
- commonPool은 싱글톤으로 JVM 전체가 공유
- 한 부분에서 종료하면 다른 부분이 영향받음
- commonPool은 JVM 종료 시까지 유지됨

만약 대상 pool을 종료하고 싶다면:
```java
ForkJoinPool customPool = new ForkJoinPool(8);
try {
    // 작업
} finally {
    customPool.shutdown();  // 가능
}
```

격리된 pool만 명시적으로 종료 가능하다.

</details>

---

**Q3.** `parallelStream()`의 병렬도를 조정할 수 있는가?

<details>
<summary>해설 보기</summary>

직접적으로는 아니지만, 몇 가지 방법이 있다:

```java
// 방법 1: Custom Collector를 사용 (권장 X)
// → 복잡하고 비효율적

// 방법 2: 커스텀 ForkJoinPool 생성 후 invoke
ForkJoinPool customPool = new ForkJoinPool(16);
List<Integer> result = customPool.invoke(
    ForkJoinTask.adapt(() ->
        data.parallelStream()
            .map(...)
            .collect(toList())
    )
);
customPool.shutdown();

// 방법 3: JVM 시작 시 commonPool 병렬도 조정
// java -Djava.util.concurrent.ForkJoinPool.common.parallelism=16

// 방법 4: 순차 처리하고 필요한 부분만 parallelStream
data.stream()  // 순차
    .filter(predicate)
    .collect(toList())
    .parallelStream()  // 필터된 데이터만 병렬
    .map(...)
    .collect(toList());
```

실제로는 parallelStream을 너무 자주 사용하는 것보다, 필요한 곳에만 선별적으로 사용하는 것이 낫다.

</details>

---

<div align="center">

**[⬅️ 이전: ForkJoinPool 동작 원리](./01-fork-join-pool-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: Spliterator 분할 전략 ➡️](./03-spliterator-split-strategy.md)**

</div>
