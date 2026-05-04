# CompletableFuture 구조 — Future와의 차이와 Callback Hell 해결

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Future.get()`은 왜 블로킹이 문제이고, `CompletableFuture`는 어떻게 해결하는가?
- `CompletableFuture` 내부의 `volatile Object result`와 `Completion` 스택 구조는?
- Treiber 스택이 무엇이고, 왜 CompletableFuture가 이 구조를 선택했는가?
- `CompletionStage` 인터페이스가 콜백 등록 모델로 제공하는 장점은?
- `Future` → `CompletableFuture` 진화의 동기와 설계 철학은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`CompletableFuture`는 Spring WebFlux, Netty, Project Reactor, RxJava 등 모든 비동기 프레임워크의 기반이다. 내부 구조를 이해하면 왜 어떤 연산은 병렬로 실행되는지, 왜 exception이 전파되는지, 왜 특정 상황에서 스레드가 스타베이션되는지 예측할 수 있다. Callback Hell과 Pyramid of Doom을 벗어나기 위한 메커니즘을 알아야 실전 비동기 설계가 가능하다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Future.get()에 의존한 동기 대기 설계
  // 스레드 블로킹 → 스레드 풀 낭비
  Future<String> future = executorService.submit(() -> expensiveCall());
  String result = future.get(5, TimeUnit.SECONDS);  // 블로킹 대기
  process(result);
  
  → 스레드 낭비 (I/O 대기 중에도 스레드 홀딩)
  → 컨텍스트 스위칭 오버헤드 증가
  → timeout 처리가 복잡 (InterruptedException)
  → 결과로 여러 작업을 연결할 수 없음

실수 2: Callback Hell 패턴
  // 중첩된 콜백으로 인한 코드 복잡도
  future1.thenAccept(result1 -> {
      future2.thenAccept(result2 -> {
          future3.thenAccept(result3 -> {
              // 3중 중첩...
              process(result1, result2, result3);
          });
      });
  });
  
  → 가독성 저하
  → 에러 처리 누락 위험
  → 디버깅 어려움

실수 3: CompletableFuture를 단순한 Future 래퍼로 인식
  CompletableFuture<String> cf = new CompletableFuture<>();
  cf.complete(result);  // 한 번만 호출 가능
  → complete() 후 새로운 콜백 등록 시 즉시 실행됨을 모름
  → 상태 변경을 명시적으로 하지 않으면 영구 대기
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 1. 블로킹 없는 비동기 체인 구성
CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> fetchUser());
CompletableFuture<String> cf2 = cf1.thenCompose(user -> 
    CompletableFuture.supplyAsync(() -> fetchOrders(user))
);
cf2.thenAccept(orders -> process(orders));  // 블로킹 없음

// 2. 선언적 콜백 체인 (Flat한 구조)
CompletableFuture.supplyAsync(() -> fetchData())
    .thenApply(data -> transform(data))
    .thenApply(transformed -> enrich(transformed))
    .thenAccept(final -> save(final))
    .exceptionally(ex -> {
        log.error("Pipeline failed", ex);
        return null;
    });

// 3. 명시적 상태 관리
CompletableFuture<String> cf = new CompletableFuture<>();
// 1단계: 비동기 작업 시작
executor.submit(() -> {
    try {
        String result = expensiveCall();
        cf.complete(result);  // 성공 시
    } catch (Exception e) {
        cf.completeExceptionally(e);  // 실패 시
    }
});

// 2단계: 콜백 등록 (이미 완료되었으면 즉시 실행)
cf.thenAccept(result -> System.out.println("Result: " + result));

// 4. 콜백 등록 후 완료된 CompletableFuture는 즉시 콜백 실행
CompletableFuture<String> completed = CompletableFuture.completedFuture("ready");
completed.thenAccept(System.out::println);  // 즉시 출력 (블로킹 없음)
```

---

## 🔬 내부 동작 원리

### 1. Future vs CompletableFuture 구조 비교

```
┌─────────────────────────────────────────────────────────────────┐
│ Future<T> (Java 5)                                              │
├─────────────────────────────────────────────────────────────────┤
│ - get(): T          (블로킹 대기)                                  │
│ - get(timeout, unit): T                                         │
│ - isDone(): boolean (완료 여부)                                   │
│ - isCancelled(): boolean                                        │
│ - cancel(mayInterrupt): boolean                                 │
│                                                                 │
│ 한계: 콜백 등록 불가, 비동기 체인 불가능                              │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ CompletableFuture<T> extends CompletionStage<T> (Java 8)        │
├─────────────────────────────────────────────────────────────────┤
│ 상태:                                                             │
│   volatile Object result;                                       │
│     → null: 미완료                                                │
│     → T instance: 성공 결과                                        │
│     → AltResult(throwable): 예외 결과                              │
│   volatile Completion stack;  (Treiber 스택)                      │
│     → 등록된 콜백들의 연결 리스트                                     │
│                                                                 │
│ 완료 메서드:                                                       │
│   complete(T value): boolean  (한 번만 성공)                      │
│   completeExceptionally(Throwable ex): boolean                  │
│   completeAsync(Supplier<T> supplier, Executor e)              │
│                                                                 │
│ 콜백 등록 메서드 (CompletionStage 구현):                           │
│   thenApply(Function<T,U>): CompletionStage<U>                 │
│   thenAccept(Consumer<T>): CompletionStage<Void>               │
│   thenRun(Runnable): CompletionStage<Void>                     │
│   thenCompose(Function<T, CompletionStage<U>>): ...            │
│   ...async 변형들 (별도 Executor 지정)                             │
└─────────────────────────────────────────────────────────────────┘
```

### 2. volatile Object result 상태 관리

```java
// OpenJDK CompletableFuture.java 일부
public class CompletableFuture<T> implements Future<T>, CompletionStage<T> {
    volatile Object result;       // null | T | AltResult
    volatile Completion stack;    // Treiber 스택의 top node
    
    // AltResult: 예외를 감싸는 래퍼 (Object 타입 동일성을 위함)
    static final class AltResult {
        final Throwable ex;
        AltResult(Throwable x) { this.ex = x; }
    }
}

상태 전이:
  null (초기) → T (완료 성공)
  null (초기) → AltResult(ex) (완료 실패)
  
  한 번의 CAS 연산으로 상태 변경 (원자성 보장)
  
예시:
  CompletableFuture<Integer> cf = new CompletableFuture<>();
  
  cf.complete(42);
    → result = Integer(42)
    → stack의 모든 Completion 노드를 순회하며 콜백 실행
  
  새로운 콜백 등록:
    cf.thenAccept(v -> System.out.println(v))
    → result != null이므로 즉시 콜백 실행 (스택 추가 불필요)
```

### 3. Treiber 스택: 콜백 체인 관리

```
Treiber 스택은 락 없는 (lock-free) LIFO 자료구조:
  - CAS (Compare-And-Swap) 연산으로만 동작
  - 여러 스레드가 동시에 push/pop 가능
  - 선입선출이 아닌 후입선출 (LIFO)

CompletableFuture의 콜백 등록 구조:

초기 상태:
  cf.result = null
  cf.stack = null

  ┌─────────────┐
  │ result: null│
  │ stack: null │
  └─────────────┘

첫 번째 콜백 등록: cf.thenApply(fn1)
  → new UniApply<T,U>(executor, this, fn1) 생성
  → CAS(stack, null, uniApply)로 스택에 추가

  ┌─────────────────────────┐
  │ result: null            │
  │ stack: UniApply(fn1)─┐  │
  │        ↑             │  │
  │        └─ next: null │  │
  └─────────────────────────┘

두 번째 콜백 등록: cf.thenApply(fn2)
  → new UniApply<T,U>(executor, this, fn2) 생성
  → CAS(stack, prev, new)로 스택에 추가 (prev를 new.next로 설정)

  ┌──────────────────────────────────┐
  │ result: null                     │
  │ stack: UniApply(fn2)───┐         │
  │        ↑                │         │
  │        ├─ next: UniApply(fn1)──┐ │
  │        │                  ↑     │ │
  │        │                  └─ next: null
  └──────────────────────────────────┘

complete() 호출: cf.complete(42)
  → result = Integer(42) (CAS로 원자적으로)
  → stack을 순회하며 각 콜백 실행:
    1. UniApply(fn2) 실행: fn2(42) → UniApply(result)
    2. 그 다음 UniApply(fn1) 실행: fn1(prev_result)

이점:
  - 락 없음 → 높은 동시성
  - O(1) 스택 추가 → 콜백 체인 생성 빠름
  - 여러 스레드가 동시에 콜백 등록 가능
```

### 4. Completion 추상 클래스와 구현

```java
// OpenJDK CompletableFuture.java
abstract static class Completion extends ForkJoinTask<Void> {
    volatile Completion next;  // 스택의 다음 노드
    
    abstract CompletableFuture<?> tryFire(int mode);
    
    // mode:
    // SYNC (0): 동기 실행 (현재 스레드)
    // ASYNC (1): 비동기 실행 (executor 사용)
    // NESTED (2): 중첩 실행 (이미 executor 스레드 내부)
}

주요 구현체:

1. UniApply (단일 입력, 변환)
   class UniApply<T,V> extends UniCompletion<T,V> {
       Function<? super T,? extends V> fn;
       CompletableFuture<V> tryFire(int mode) {
           if (claim()) {  // CAS로 소유권 획득
               try {
                   CompletableFuture<V> d = dep;
                   CompletableFuture<T> a = src;
                   Function<? super T,? extends V> f = fn;
                   if (f != null && d != null && a != null) {
                       T t = a.getRawResult();
                       V v = f.apply(t);
                       d.completeValue(v);
                   }
               } catch (Throwable ex) {
                   d.completeThrowable(ex);
               }
           }
           return null;
       }
   }

2. BiApply (두 입력, 변환)
   class BiApply<T,U,V> extends BiCompletion<T,U,V> {
       BiFunction<? super T,? super U,? extends V> fn;
       // 두 CompletableFuture의 완료를 대기하고 fn 적용
   }

3. CoCompletion (동시성 제어)
   class CoCompletion extends BiCompletion<?, ?, ?> {
       // 두 future 중 하나가 완료되면 즉시 콜백 실행
   }
```

### 5. CompletionStage 인터페이스

```java
public interface CompletionStage<T> {
    // 변환 (Function)
    <U> CompletionStage<U> thenApply(Function<? super T, ? extends U> fn);
    <U> CompletionStage<U> thenApplyAsync(Function<? super T, ? extends U> fn);
    <U> CompletionStage<U> thenApplyAsync(Function<? super T, ? extends U> fn,
                                           Executor executor);
    
    // 소비 (Consumer)
    CompletionStage<Void> thenAccept(Consumer<? super T> action);
    CompletionStage<Void> thenAcceptAsync(Consumer<? super T> action);
    CompletionStage<Void> thenAcceptAsync(Consumer<? super T> action,
                                           Executor executor);
    
    // 실행 (Runnable)
    CompletionStage<Void> thenRun(Runnable action);
    CompletionStage<Void> thenRunAsync(Runnable action);
    CompletionStage<Void> thenRunAsync(Runnable action, Executor executor);
    
    // 선택지 (일부만 실행)
    <U> CompletionStage<U> applyToEither(CompletionStage<? extends T> other,
                                          Function<? super T, U> fn);
    
    // 예외 처리
    CompletionStage<T> exceptionally(Function<Throwable, ? extends T> fn);
    CompletionStage<T> handle(BiFunction<? super T, Throwable, ? extends T> fn);
    CompletionStage<Void> whenComplete(BiConsumer<? super T, ? super Throwable> action);
    
    // 두 개 조합
    <U, V> CompletionStage<V> thenCombine(
        CompletionStage<? extends U> other,
        BiFunction<? super T, ? super U, ? extends V> fn);
}
```

---

## 💻 실전 실험

### 실험 1: Treiber 스택 콜백 순서 확인

```java
@Test
public void testCallbackOrderInStack() {
    CompletableFuture<Integer> cf = new CompletableFuture<>();
    
    List<String> order = Collections.synchronizedList(new ArrayList<>());
    
    cf.thenAccept(v -> order.add("first"));
    cf.thenAccept(v -> order.add("second"));
    cf.thenAccept(v -> order.add("third"));
    
    cf.complete(42);
    
    // Treiber 스택은 LIFO이므로 역순 실행
    // 실제 출력: [third, second, first]
    System.out.println(order);
    
    // 실행 결과:
    // [third, second, first]
    // → 마지막에 등록된 콜백이 먼저 실행됨 (스택 특성)
}

// 해석:
// complete() 호출 시 스택을 head부터 순회:
// stack: [third(top)] -> [second] -> [first] -> null
// 따라서 third, second, first 순서로 콜백 실행
```

### 실험 2: 이미 완료된 Future에 콜백 등록

```java
@Test
public void testCallbackOnCompletedFuture() throws InterruptedException {
    CompletableFuture<Integer> cf = CompletableFuture.completedFuture(42);
    
    CountDownLatch latch = new CountDownLatch(1);
    
    cf.thenAccept(v -> {
        System.out.println("Callback executed with " + v);
        latch.countDown();
    });
    
    // 블로킹 없음 → 즉시 반환
    System.out.println("After registering callback");
    
    latch.await(100, TimeUnit.MILLISECONDS);
    
    // 출력 순서:
    // After registering callback
    // Callback executed with 42
}

// 해석:
// result != null이므로 콜백 등록 즉시 실행됨
// 스택에 추가되지 않음 → 블로킹 없음
```

### 실험 3: result 상태 확인

```java
@Test
public void testResultStateTransition() throws Exception {
    CompletableFuture<String> cf = new CompletableFuture<>();
    
    // 초기 상태: result = null, stack = null
    System.out.println("isDone: " + cf.isDone());  // false
    
    // complete() 호출: result = "done"
    cf.complete("done");
    
    System.out.println("isDone: " + cf.isDone());  // true
    System.out.println("Result: " + cf.get());     // "done"
    
    // 재시도는 실패
    boolean second = cf.complete("another");
    System.out.println("Second complete: " + second);  // false
    System.out.println("Result still: " + cf.get());   // "done"
}

// 해석:
// CAS 연산이 상태를 null → "done"으로 한 번만 변경 가능
// 이후 complete() 호출은 CAS 실패로 반환값 false
```

---

## 📊 성능/비교

```
╔════════════════════════════════════════════════════════════════════╗
║ Future vs CompletableFuture 성능 비교                                ║
╠════════════════════════════════════════════════════════════════════╣
║ 작업          │ Future.get()    │ CF.thenApply()  │ 특성              ║
╠════════════════════════════════════════════════════════════════════╣
║ 스레드 대기    │ 스레드 블로킹    │ 콜백 기반 (non-blocking) │            ║
║ 메모리 (콜백)  │ 콜백 미지원     │ 콜백당 ~200B  │            ║
║ 콜백 체인     │ 불가능         │ 가능 (평탄)    │            ║
║ 예외 처리     │ try-catch      │ exceptionally  │ 선언적      ║
╚════════════════════════════════════════════════════════════════════╝

동시성 비교 (1000개 콜백 등록, 1000번 완료):

Future.get() 패턴:
  public void processWithFuture(ExecutorService es) {
      List<Future<Integer>> futures = new ArrayList<>();
      for (int i = 0; i < 1000; i++) {
          futures.add(es.submit(() -> heavyWork()));
      }
      // 1000개 스레드 블로킹 대기
      futures.forEach(f -> {
          try {
              process(f.get());  // 블로킹
          } catch (InterruptedException | ExecutionException e) {
              // 예외 처리
          }
      });
  }
  → 스레드 풀 크기 >= 1000 필요
  → 컨텍스트 스위칭 오버헤드

CompletableFuture 패턴:
  public void processWithCF(ExecutorService es) {
      for (int i = 0; i < 1000; i++) {
          CompletableFuture.supplyAsync(() -> heavyWork(), es)
              .thenAccept(this::process)
              .exceptionally(ex -> {
                  log.error("Error", ex);
                  return null;
              });
      }
      // 등록만 수행, 블로킹 없음
      // 실제 실행은 비동기
  }
  → 스레드 풀 크기는 작아도 가능
  → 메모리: O(완료되지 않은 CF 수)

Treiber 스택 콜백 등록 성능:
  스택 push (콜백 등록): O(1) CAS
  스택 순회 (콜백 실행): O(n) where n = 콜백 수
```

---

## ⚖️ 트레이드오프

| 측면 | Future | CompletableFuture |
|------|--------|------------------|
| **간단함** | 간단 (get() 명확) | 복잡 (많은 메서드) |
| **블로킹** | 필수 (get()) | 선택 (콜백 기반) |
| **메모리** | 낮음 | 콜백당 ~200B |
| **예외처리** | try-catch | exceptionally, handle |
| **코드 가독성** | get() 명확 | 콜백 체인 (점진적) |
| **동시성** | 제한적 | 높음 (스택 기반) |
| **Deadline 처리** | timeout 명시 | 복잡 (별도 로직) |

---

## 📌 핵심 정리

1. **Future의 한계**: `get()` 블로킹으로 스레드 낭비, 콜백 등록 불가능 → CompletableFuture로 진화

2. **CompletableFuture 상태**: `volatile Object result`로 완료 결과 저장 (null/T/AltResult), `volatile Completion stack`으로 콜백 체인 관리

3. **Treiber 스택**: 락 없는 LIFO 자료구조, CAS로 동시 접근 처리, 여러 스레드의 동시 콜백 등록 지원

4. **콜백 실행 타이밍**: 콜백 등록 시 result != null이면 즉시 실행, 스택 추가 없음

5. **CompletionStage 인터페이스**: 변환(thenApply), 소비(thenAccept), 실행(thenRun), 예외처리(exceptionally) 등의 선언적 메서드 제공

6. **성능**: 스레드 블로킹 없음, 콜백당 메모리 증가, 동시성 높음 (CAS 기반)

7. **설계 철학**: 비동기 작업의 콜백 체인을 선언적으로 구성, Callback Hell 해결, 리액티브 프로그래밍의 기초

---

## 🤔 생각해볼 문제

<details>
<summary>Q1: 왜 Treiber 스택을 LIFO로 설계했는가? FIFO가 나은 점은?</summary>

**해설**: LIFO (후입선출)는 CAS 연산의 복잡도를 O(1)로 유지할 수 있게 해준다. FIFO로 구현하려면 tail 포인터를 관리해야 하는데, 동시성 환경에서 head와 tail을 원자적으로 갱신해야 하므로 더 복잡하다.

실제로 콜백 순서는 사용자가 통제할 수 없으므로 (비동기이기 때문에), FIFO 보장은 불필요하다. 대신 O(1) 등록을 선택한 설계이다.

다만, complete() 호출 시 모든 콜백이 즉시 실행되는 상황에서는 역순 실행 순서가 결정론적이므로, 디버깅할 때 예측 가능한 순서를 제공한다.

</details>

<details>
<summary>Q2: CompletableFuture가 이미 완료되었을 때 콜백을 등록하면, 등록 즉시 실행되는가?</summary>

**해설**: **대부분의 경우 YES, 하지만 async 변형은 NO**.

```java
CompletableFuture<Integer> cf = CompletableFuture.completedFuture(42);

// 즉시 실행 (현재 스레드)
cf.thenAccept(System.out::println);  // "42" 즉시 출력

// 즉시 실행 안 함 (별도 스레드/executor)
cf.thenAcceptAsync(System.out::println);  // 나중에 출력
```

이유: `thenAccept()`는 동기 콜백으로, result != null을 확인 후 현재 스레드에서 실행. `thenAcceptAsync()`는 executor에 submit하므로 별도 스레드에서 나중에 실행된다.

</details>

<details>
<summary>Q3: 만약 complete() 중에 콜백이 새로운 콜백을 등록하면 어떻게 되는가?</summary>

**해설**: **새로운 콜백도 실행된다**, 하지만 complete() 호출을 시작한 스레드에서만 스택 순회가 진행된다.

```java
CompletableFuture<Integer> cf = new CompletableFuture<>();

cf.thenAccept(v -> {
    System.out.println("1st callback");
    // 이 시점에서 새로운 콜백 등록
    cf.thenAccept(v2 -> System.out.println("nested"));
});

cf.complete(42);

// 출력:
// 1st callback
// nested (같은 스레드에서 즉시 실행)
```

원인: `thenAccept()` 호출 시 result != null이므로 즉시 실행. 동시성을 보장하기 위해 lockState CAS 확인 후 중복 실행 방지.

</details>

<div align="center">

**[⬅️ 이전 챕터: 직렬화·ORM·Optional](../chapter04-optional/05-optional-serialization-orm.md)** | **[홈으로 🏠](../README.md)** | **[다음: thenApply vs thenCompose vs thenCombine ➡️](./02-thenapply-thencompose-thencombine.md)**

</div>
