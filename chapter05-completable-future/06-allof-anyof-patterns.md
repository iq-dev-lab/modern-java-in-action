# allOf · anyOf — 구현 분석과 실전 패턴

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `CompletableFuture.allOf()`는 모든 future 완료를 어떻게 감지하는가?
- `anyOf()`는 첫 번째 완료된 future만 어떻게 반환하는가?
- `allOf().thenApply(v → futures.stream().map(CF::join))`은 왜 필요한가?
- allOf에서 하나라도 예외 발생 시 동작은?
- Timeout과 결합한 partial failure 처리는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`allOf`는 여러 비동기 작업의 완료를 기다리는 가장 흔한 패턴이다. 마이크로서비스에서 10개의 API 호출을 병렬로 수행하고 모두 완료될 때까지 기다린 후 결과를 집계해야 할 때 사용된다. `anyOf`는 race condition (첫 응답)이 필요한 경우다. 하지만 구현 세부사항을 모르면 예외 처리, timeout, partial failure를 제대로 처리할 수 없다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: allOf의 반환 타입을 잘못 이해
  CompletableFuture<Void> allDone = CompletableFuture.allOf(cf1, cf2, cf3);
  
  // 틀린 코드: Void를 받는다고 생각
  allDone.thenAccept(v → {
      // ??? v는 null, 결과는 어디에?
      System.out.println(v);  // null
  });
  
  → allOf는 결과값 합치기를 지원하지 않음
  → 직접 futures 리스트로 결과 수집 필요

실수 2: anyOf의 첫 완료가 성공이 아닐 수도 있음을 모름
  CompletableFuture<Object> first = CompletableFuture.anyOf(cf1, cf2);
  
  // cf1이 예외, cf2가 100ms 후 성공인데,
  // cf1이 먼저 완료되면?
  first.thenAccept(v → {
      System.out.println("First: " + v);  // 예외 발생
  });
  
  → anyOf는 first-completed, 상태 무관
  → 예외 가능성 높음

실제 동작:
  CompletableFuture.<Object>failedFuture(new RuntimeException())
      이 다른 future보다 먼저 실패하면,
  anyOf의 결과도 그 예외가 된다.

실수 3: allOf에서 하나 실패 시 나머지는?
  CompletableFuture.allOf(cf1, cf2, cf3).get();
  
  // cf1 실패, cf2/cf3는 계속 실행 중
  // cf2/cf3의 결과는 버려짐 (리소스 누수 가능)

실수 4: timeout 처리 누락
  CompletableFuture.allOf(futures...).get();
  
  // timeout 없이 영구 블로킹 가능
  // 하나의 future가 완료되지 않으면 allOf도 영구 대기
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 1. allOf: 모든 future 완료 대기 후 결과 수집
List<CompletableFuture<String>> futures = Arrays.asList(
    fetchUser(), fetchOrders(), fetchProfile()
);

CompletableFuture<List<String>> allResults = CompletableFuture
    .allOf(futures.toArray(new CompletableFuture[0]))
    .thenApply(v → futures.stream()
        .map(CompletableFuture::join)  // 이미 완료됨, 블로킹 없음
        .collect(toList())
    );

List<String> results = allResults.get();  // ["user", "orders", "profile"]

// 2. allOf with exception handling
CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
    .exceptionally(ex → {
        log.error("One or more futures failed", ex);
        // 예외 로깅
        return null;  // 복구
    })
    .thenApply(v → {
        if (v == null) return new ArrayList<>();  // 실패했으면 빈 리스트
        
        return futures.stream()
            .map(cf → {
                try {
                    return cf.getNow(null);  // 예외 발생 시 null
                } catch (Exception e) {
                    return null;
                }
            })
            .filter(Objects::nonNull)
            .collect(toList());
    });

// 3. anyOf: 첫 완료된 결과 (성공/실패 무관)
CompletableFuture<Object> first = CompletableFuture.anyOf(
    fetchUserFast(), fetchUserSlow()
);

first.thenAccept(result → {
    System.out.println("First completed: " + result);
});

// 4. timeout 결합
CompletableFuture<List<String>> withTimeout = CompletableFuture
    .allOf(futures.toArray(new CompletableFuture[0]))
    .orTimeout(5, TimeUnit.SECONDS)  // Java 9+
    .thenApply(v → futures.stream()
        .map(CompletableFuture::join)
        .collect(toList())
    )
    .exceptionally(ex → {
        if (ex instanceof TimeoutException) {
            log.warn("Timeout waiting for all futures");
        }
        return Collections.emptyList();
    });

// 5. Partial success: 일부 실패 허용
CompletableFuture<List<Result>> partialResults = CompletableFuture
    .allOf(futures.toArray(new CompletableFuture[0]))
    .thenApply(v → futures.stream()
        .map(cf → {
            try {
                return new Result(cf.join(), null);
            } catch (CompletionException e) {
                return new Result(null, e.getCause());
            }
        })
        .collect(toList())
    );
```

---

## 🔬 내부 동작 원리

### 1. CompletableFuture.allOf() 구현

```java
// OpenJDK CompletableFuture.java

public static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs) {
    return andTree(cfs, 0, cfs.length - 1);
}

// Tree 구조로 병렬 결합
static CompletableFuture<Void> andTree(
    CompletableFuture<?>[] cfs, int lo, int hi) {
    
    CompletableFuture<Void> d = new CompletableFuture<Void>();
    
    if (lo > hi) {
        d.result = null;
        return d;
    }
    
    int mid = (lo + hi) >>> 1;
    CompletableFuture<?> fst = null, snd = null;
    CompletableFuture<Void> r = null, s = null;
    
    // 트리 구축: [0,1] + [2,3] = 병렬 결합
    if (hi == lo) {
        fst = cfs[lo];
    } else {
        fst = andTree(cfs, lo, mid);
        snd = andTree(cfs, mid + 1, hi);
    }
    
    // BiRelay: 두 개 future의 완료를 대기
    CompletableFuture<Void> frontier = new CompletableFuture<>();
    if (!fst.isDone()) {
        fst.whenComplete((r1, ex1) → {
            if (snd != null && !snd.isDone()) {
                snd.whenComplete((r2, ex2) → {
                    if (ex1 != null) frontier.completeExceptionally(ex1);
                    else if (ex2 != null) frontier.completeExceptionally(ex2);
                    else frontier.complete(null);
                });
            } else {
                if (ex1 != null) frontier.completeExceptionally(ex1);
                else frontier.complete(null);
            }
        });
    }
    
    return frontier;
}

동작:

allOf(CF1, CF2, CF3, CF4)
  │
  └─ andTree([CF1, CF2, CF3, CF4], 0, 3)
     │
     ├─ andTree([CF1, CF2], 0, 1)          ─ BiRelay(CF1, CF2)
     │  ├─ andTree([CF1], 0, 0) → CF1
     │  └─ andTree([CF2], 1, 1) → CF2
     │
     └─ andTree([CF3, CF4], 2, 3)          ─ BiRelay(CF3, CF4)
        ├─ andTree([CF3], 2, 2) → CF3
        └─ andTree([CF4], 3, 3) → CF4

완료 조건:
  - 모든 CF 완료
  - 하나라도 예외 발생 시 즉시 그 예외 전파
  - 모두 정상 시 null로 완료
```

### 2. CompletableFuture.anyOf() 구현

```java
public static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs) {
    return orTree(cfs, 0, cfs.length - 1);
}

static CompletableFuture<Object> orTree(
    CompletableFuture<?>[] cfs, int lo, int hi) {
    
    CompletableFuture<Object> d = new CompletableFuture<Object>();
    
    if (lo > hi) {
        d.result = null;
        return d;
    }
    
    int mid = (lo + hi) >>> 1;
    CompletableFuture<?> fst = null, snd = null;
    
    if (hi == lo) {
        fst = cfs[lo];
    } else {
        fst = orTree(cfs, lo, mid);
        snd = orTree(cfs, mid + 1, hi);
    }
    
    // BiRelay OR: 둘 중 하나만 완료되면 즉시 반환
    CompletableFuture<Object> frontier = new CompletableFuture<>();
    
    fst.whenComplete((r1, ex1) → {
        if (!frontier.isDone()) {
            if (ex1 == null) {
                frontier.complete(r1);  // 성공하면 즉시
            } else if (snd != null) {
                snd.whenComplete((r2, ex2) → {
                    if (ex2 == null) {
                        frontier.complete(r2);
                    } else {
                        frontier.completeExceptionally(ex1);  // fst 예외
                    }
                });
            } else {
                frontier.completeExceptionally(ex1);
            }
        }
    });
    
    return frontier;
}

동작:

anyOf(CF1, CF2, CF3, CF4)는
  CF1, CF2, CF3, CF4 중 **첫 번째 완료된 것의 결과를 반환**
  (성공이든 예외든 상관없음)

예: CF1(실패 100ms) vs CF2(성공 50ms)
    → CF2가 먼저 완료 → CF2의 결과값 반환

예: CF1(실패 50ms) vs CF2(성공 100ms)
    → CF1이 먼저 완료 → CF1의 예외 전파
    (CF2는 여전히 실행 중)
```

### 3. allOf vs anyOf 동작 비교

```
allOf (AND 로직):

  ┌─ CF1 (100ms) ─┐
  ├─ CF2 (200ms) ─┤─→ allOf (모두 완료) ─→ 200ms 후 Void
  ├─ CF3 (50ms)  ─┤
  └─ CF4 (150ms) ┘

완료 조건:
  - max(CF1, CF2, CF3, CF4) = 200ms
  - 하나라도 예외 발생 시 즉시 그 예외


anyOf (OR 로직):

  ┌─ CF1 (100ms) ─┐
  ├─ CF2 (200ms) ─┤─→ anyOf (첫 완료) ─→ 50ms 후 CF3 결과
  ├─ CF3 (50ms)  ─┤
  └─ CF4 (150ms) ┘

완료 조건:
  - min(CF1, CF2, CF3, CF4) = 50ms (CF3)
  - CF3의 결과 (성공/실패) 그대로 반환
```

### 4. 결과 수집 메커니즘

```
allOf는 결과를 직접 반환하지 않고, Void만 반환

CompletableFuture<Void> all = CompletableFuture.allOf(cf1, cf2, cf3);

all은:
  - 모두 완료 ✓ = null로 완료
  - 하나 실패 ✗ = 예외로 완료

결과 수집:

1단계: allOf로 완료 신호 받기
  CompletableFuture<Void> allDone = allOf(futures..);

2단계: 각 future에서 결과 수집
  List<Result> results = futures.stream()
      .map(cf → cf.join())  // 이미 완료됨, 블로킹 없음
      .collect(toList());

join() vs get():
  - join(): 블로킹, unchecked exception (CompletionException)
  - get(): 블로킹, checked exception (ExecutionException)
  
  allOf 후에는 모두 완료되어 있으므로 블로킹 없음
  따라서 join() 선호 (체크드 예외 처리 불필요)

예외 처리:

  CompletableFuture.allOf(futures.toArray(...))
      .thenApply(v → {
          return futures.stream()
              .map(cf → {
                  try {
                      return cf.getNow(null);  // 또는 join()
                  } catch (CompletionException e) {
                      return null;  // 예외 무시
                  }
              })
              .collect(toList());
      })

getNow(null): 이미 완료되어 있으면 값 반환, 미완료면 null 반환
```

---

## 💻 실전 실험

### 실험 1: allOf 동시 실행 확인

```java
@Test
public void testAllOfConcurrency() throws Exception {
    long start = System.currentTimeMillis();
    
    List<CompletableFuture<Integer>> futures = Arrays.asList(
        asyncTask("task1", 100),
        asyncTask("task2", 200),
        asyncTask("task3", 150)
    );
    
    CompletableFuture<Void> allDone = CompletableFuture.allOf(
        futures.toArray(new CompletableFuture[0])
    );
    
    allDone.get();  // 모두 완료 대기
    
    long duration = System.currentTimeMillis() - start;
    System.out.println("Duration: " + duration + "ms");
    // ~200ms (병렬 실행, max(100,200,150))
}

private CompletableFuture<Integer> asyncTask(String name, long delayMs) {
    return CompletableFuture.supplyAsync(() → {
        try { Thread.sleep(delayMs); } catch (InterruptedException e) {}
        System.out.println(name + " completed");
        return 1;
    });
}

// 결과:
// task1 completed
// task3 completed
// task2 completed
// Duration: ~200ms
```

### 실험 2: anyOf는 첫 완료

```java
@Test
public void testAnyOfFirstCompletion() throws Exception {
    long start = System.currentTimeMillis();
    
    List<CompletableFuture<Integer>> futures = Arrays.asList(
        asyncTask("slow", 500),
        asyncTask("fast", 50),
        asyncTask("medium", 200)
    );
    
    CompletableFuture<Object> first = CompletableFuture.anyOf(
        futures.toArray(new CompletableFuture[0])
    );
    
    Object result = first.get();  // 첫 완료 대기
    
    long duration = System.currentTimeMillis() - start;
    System.out.println("Duration: " + duration + "ms");
    // ~50ms (첫 완료)
    System.out.println("Result: " + result);  // 1 (fast의 결과)
}

// 결과:
// fast completed
// Duration: ~50ms
// Result: 1
```

### 실험 3: allOf에서 예외 처리

```java
@Test
public void testAllOfWithException() throws Exception {
    CompletableFuture<Integer> cf1 = CompletableFuture.completedFuture(1);
    CompletableFuture<Integer> cf2 = CompletableFuture.failedFuture(
        new RuntimeException("Error in cf2")
    );
    CompletableFuture<Integer> cf3 = CompletableFuture.completedFuture(3);
    
    try {
        CompletableFuture.allOf(cf1, cf2, cf3).get();
    } catch (ExecutionException e) {
        System.out.println("Caught: " + e.getCause().getMessage());
        // "Error in cf2"
    }
}

// 결과:
// Caught: Error in cf2
// → cf2의 예외가 allOf를 실패시킴
// → cf1, cf3의 결과는 무시됨
```

### 실험 4: 결과 수집

```java
@Test
public void testAllOfResultCollection() throws Exception {
    List<CompletableFuture<String>> futures = Arrays.asList(
        CompletableFuture.completedFuture("A"),
        CompletableFuture.completedFuture("B"),
        CompletableFuture.completedFuture("C")
    );
    
    List<String> results = CompletableFuture
        .allOf(futures.toArray(new CompletableFuture[0]))
        .thenApply(v → futures.stream()
            .map(CompletableFuture::join)
            .collect(toList())
        )
        .get();
    
    System.out.println("Results: " + results);  // [A, B, C]
}

// 결과:
// Results: [A, B, C]
```

---

## 📊 성능/비교

```
╔════════════════════════════════════════════════════════════════════╗
║ allOf vs anyOf 성능 비교 (3개 future, 100ms 각)                   ║
╠════════════════════════════════════════════════════════════════════╣
║ 연산     │ 완료 조건        │ 시간  │ 메모리 │ 사용 시기        ║
╠════════════════════════════════════════════════════════════════════╣
║ allOf    │ 모두 완료        │ 300ms│ O(n)  │ 배치 처리        ║
║ anyOf    │ 첫 완료          │ 100ms│ O(n)  │ Race condition   ║
╚════════════════════════════════════════════════════════════════════╝

Tree 구축 복잡도:

allOf(n개 future):
  - Tree depth: O(log n)
  - BiRelay 노드: O(n)
  - 메모리: O(n)

예: allOf(cf1, cf2, cf3, cf4)
  Tree:
    ┌─ BiRelay(CF1, CF2)
    │  ├─ CF1
    │  └─ CF2
    └─ BiRelay(CF3, CF4)
       ├─ CF3
       └─ CF4
    
    BiRelay 노드: 3개

결과 집계:

Stream.map(CF::join):
  - O(n) 순회
  - 이미 완료되어 있으므로 실제 블로킹 없음
  - 메모리: O(n) (결과 리스트)
```

---

## ⚖️ 트레이드오프

| 메서드 | 완료 조건 | 장점 | 단점 | 사용 시기 |
|--------|---------|------|------|---------|
| **allOf** | 모두 완료 | 병렬성, 예측 가능 | 예외 시 전체 실패, 느림 | 배치 처리 |
| **anyOf** | 첫 완료 | 빠름, responsive | 나머지 작업 계속 실행, 예외 가능 | Cache/Fallback |

---

## 📌 핵심 정리

1. **allOf**: 모든 future 완료 대기, Tree 병렬 구축, Void 반환

2. **anyOf**: 첫 완료된 future 결과 반환, Race condition 처리

3. **결과 수집**: `allOf(...).thenApply(v → futures.stream().map(CF::join))`

4. **예외 처리**: allOf에서 하나 실패 시 즉시 그 예외 전파

5. **partial success**: 각 future를 개별적으로 예외 처리

6. **timeout**: `orTimeout(duration, unit)` 또는 `completeOnTimeout()`

7. **성능**: allOf O(n·log n), anyOf O(min(완료시간))

---

## 🤔 생각해볼 문제

<details>
<summary>Q1: allOf에서 하나가 완료되지 않으면?</summary>

**해설**: allOf는 영구 대기한다.

```java
CompletableFuture<Integer> cf1 = CompletableFuture.completedFuture(1);
CompletableFuture<Integer> cf2 = new CompletableFuture<>();  // 미완료

CompletableFuture.allOf(cf1, cf2)
    .get();  // 영구 블로킹

// 해결: timeout 사용
allOf(cf1, cf2)
    .get(5, TimeUnit.SECONDS);  // 5초 후 TimeoutException
```

따라서 allOf 사용 시 **반드시 timeout을 지정해야 한다.**

</details>

<details>
<summary>Q2: anyOf에서 첫 완료가 예외라면?</summary>

**해설**: anyOf의 결과도 예외가 되고, 다른 future는 계속 실행된다.

```java
CompletableFuture<Integer> cf1 = CompletableFuture.failedFuture(
    new RuntimeException("fast fail")
);
CompletableFuture<Integer> cf2 = CompletableFuture.supplyAsync(() → {
    try { Thread.sleep(1000); } catch (InterruptedException e) {}
    return 2;
});

try {
    CompletableFuture.anyOf(cf1, cf2).get();
} catch (ExecutionException e) {
    System.out.println("Got: " + e.getCause().getMessage());
    // "fast fail"
}

// cf2는 여전히 1초 기다린 후 완료 (결과 버려짐)
```

이는 race condition이 있는 API 호출에서 문제가 될 수 있다.

</details>

<details>
<summary>Q3: 1000개의 future를 allOf에 넘기면?</summary>

**해설**: Tree 깊이가 O(log 1000) ≈ 10 레벨이고, 메모리는 O(1000) BiRelay 노드가 생성된다.

```java
List<CompletableFuture<String>> futures = new ArrayList<>();
for (int i = 0; i < 1000; i++) {
    futures.add(fetchData(i));
}

CompletableFuture.allOf(futures.toArray(...)).get();
```

문제점:
- 메모리: ~1000 × 300B = ~300KB
- 컨텍스트: Tree 구축 자체가 오버헤드
- 개선: 배치 처리 (100개씩)

```java
for (int batch = 0; batch < 1000; batch += 100) {
    List<CF> batchFutures = futures.subList(batch, Math.min(batch+100, 1000));
    allOf(batchFutures.toArray(...)).get();
}
```

</details>

<div align="center">

**[⬅️ 이전: 예외 처리 3가지](./05-exception-handling-three-ways.md)** | **[홈으로 🏠](../README.md)** | **[다음: commonPool 의존성 문제 ➡️](./07-commonpool-dependency.md)**

</div>
