# thenRun vs thenAccept vs thenApply — 결과 사용 패턴

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `thenRun(Runnable)`, `thenAccept(Consumer<T>)`, `thenApply(Function<T,U>)`의 차이는?
- 반환 타입이 `CompletableFuture<Void>`와 `CompletableFuture<U>`로 다른 이유는?
- 함수형 인터페이스 `Consumer`와 `Function`의 카테고리 차이는?
- 비동기 체인에서 세 메서드 중 어떤 것을 선택해야 하는가?
- "부작용 only"와 "값 반환"의 설계 의도는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

로그 기록, 메트릭 수집, 데이터베이스 저장, 캐시 갱신 등 대부분의 실무 작업은 "결과를 반환하지 않는 부작용"이다. 이들은 `thenAccept()`로 충분하다. 반면 데이터 변환, 필터링, 다음 단계 계산은 `thenApply()`로 처리해야 한다. 올바른 선택은 코드 의도를 명확하게 한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: thenApply를 부작용 작업에 사용
  // 안티패턴: null 반환으로 CompletableFuture<Void>를 명시적으로 만듦
  cf.thenApply(value → {
      log.info("Processing: " + value);
      save(value);
      return null;  // ← 이상함
  });
  
  → 의도 불명확
  → null 체크 필요
  → 값 반환 가능성을 암시
  → 읽는 사람이 혼동

실수 2: thenAccept에서 다음 값을 계산하려고 시도
  // 컴파일 안 됨 - Consumer는 void 반환
  cf.thenAccept(value → {
      int result = value * 2;  // 계산
      // 반환 불가능 (void 함수형 인터페이스)
  });
  
  // 해결: thenApply 사용
  cf.thenApply(value → value * 2);

실수 3: thenRun에서 값에 접근하려고 시도
  // 컴파일 안 됨 - Runnable은 입력도 없음
  cf.thenRun(() → {
      System.out.println(???);  // 값에 접근 불가능
  });
  
  → thenRun은 이전 값을 사용하지 않음
  → 부작용 only (로그, 플래그 설정 등)
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 1. thenRun: 결과 무시, 부작용만 (Runnable)
CompletableFuture<Integer> cf = CompletableFuture.completedFuture(42);

cf.thenRun(() → {
    System.out.println("Completed!");  // 값 접근 불가능
    incrementMetric();  // 부작용만
})
  .get();  // CompletableFuture<Void> 반환

// 2. thenAccept: 결과 소비, 변환 없음 (Consumer<T>)
cf.thenAccept(value → {
    System.out.println("Value: " + value);  // 값 접근
    saveToDb(value);  // 부작용
    // 반환 불가능 (void 메서드)
})
  .get();  // CompletableFuture<Void> 반환

// 3. thenApply: 결과 변환, 다음 단계로 전달 (Function<T,U>)
cf.thenApply(value → value * 2)
  .thenApply(doubled → "Result: " + doubled)
  .thenAccept(System.out::println)
  .get();  // "Result: 84"

// 실무 예제: 사용자 로드 → 주문 조회 → 로그 기록
CompletableFuture<User> userCF = fetchUser("alice");

userCF
    .thenApply(user → user.getId())  // User → ID (값 변환)
    .thenCompose(userId → fetchOrders(userId))  // ID → CF<Orders> (비동기)
    .thenAccept(orders → {  // Orders → void (부작용)
        log.info("Loaded {} orders", orders.size());
        cache.put("orders", orders);
    });
```

---

## 🔬 내부 동작 원리

### 1. 함수형 인터페이스 분류

```
함수형 인터페이스 (Functional Interface) = 추상 메서드 1개만 가짐

┌─────────────────────────────────────────────────────────┐
│ 함수형 인터페이스 분류                                    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ 1. Supplier<T>      없음  →  T       (팩토리)             │
│    public T get()                                      │
│                                                         │
│ 2. Consumer<T>      T    →  void    (소비)              │
│    public void accept(T t)                             │
│                                                         │
│ 3. Function<T,R>    T    →  R       (변환)              │
│    public R apply(T t)                                 │
│                                                         │
│ 4. Predicate<T>     T    →  boolean (검사)             │
│    public boolean test(T t)                            │
│                                                         │
│ 5. Runnable         없음  →  void   (실행)              │
│    public void run()                                   │
│                                                         │
│ 6. BiConsumer<T,U>  (T,U) →  void  (이진 소비)         │
│    public void accept(T t, U u)                        │
│                                                         │
│ 7. BiFunction<T,U,R> (T,U) →  R    (이진 변환)        │
│    public R apply(T t, U u)                            │
│                                                         │
└─────────────────────────────────────────────────────────┘

CompletableFuture 체인에서 자주 사용:

  Supplier<T>      → supplyAsync(Supplier<T>)
  Consumer<T>      → thenAccept(Consumer<T>)
  Function<T,R>    → thenApply(Function<T,R>)
  Runnable         → thenRun(Runnable)
  BiFunction<T,U,R> → thenCombine(cf, BiFunction)
```

### 2. thenRun, thenAccept, thenApply 구현 비교

```java
// OpenJDK CompletableFuture.java

// 1. thenRun(Runnable) - 값 무시
public CompletableFuture<Void> thenRun(Runnable action) {
    return uniRunStage(null, action);
}

abstract static class UniCompletion<T,V> extends Completion {
    class UniRun extends UniCompletion<T,Void> {
        Runnable fn;
        
        public CompletableFuture<Void> tryFire(int mode) {
            CompletableFuture<T> source = src;
            CompletableFuture<Void> dest = dep;
            
            if (source.isDone() && dest != null) {
                try {
                    source.getRawResult();  // T 값을 읽지만 사용하지 않음
                    fn.run();  // 입력 없이 실행
                    dest.completeValue(null);  // Void 반환
                } catch (Throwable ex) {
                    dest.completeExceptionally(ex);
                }
            }
            return null;
        }
    }
}

// 2. thenAccept(Consumer<T>) - 값 소비, 반환 없음
public CompletableFuture<Void> thenAccept(Consumer<? super T> action) {
    return uniAcceptStage(null, action);
}

class UniAccept extends UniCompletion<T,Void> {
    Consumer<? super T> fn;
    
    public CompletableFuture<Void> tryFire(int mode) {
        CompletableFuture<T> source = src;
        CompletableFuture<Void> dest = dep;
        
        if (source.isDone() && dest != null) {
            try {
                T value = source.getRawResult();  // 값 추출
                fn.accept(value);  // Consumer에 전달
                dest.completeValue(null);  // Void 반환
            } catch (Throwable ex) {
                dest.completeExceptionally(ex);
            }
        }
        return null;
    }
}

// 3. thenApply(Function<T,U>) - 값 변환, 다음 단계로
public <U> CompletableFuture<U> thenApply(Function<? super T, ? extends U> fn) {
    return uniApplyStage(null, fn);
}

class UniApply extends UniCompletion<T,U> {
    Function<? super T, ? extends U> fn;
    
    public CompletableFuture<U> tryFire(int mode) {
        CompletableFuture<T> source = src;
        CompletableFuture<U> dest = dep;
        
        if (source.isDone() && dest != null) {
            try {
                T value = source.getRawResult();  // 값 추출
                U result = fn.apply(value);  // Function에 전달 및 변환
                dest.completeValue(result);  // U 값으로 완료
            } catch (Throwable ex) {
                dest.completeExceptionally(ex);
            }
        }
        return null;
    }
}
```

### 3. 반환 타입의 의미

```
┌──────────────────────────────────────────────────────────────────┐
│ 반환 타입 = 다음 체인 가능 여부                                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ thenRun(Runnable)                                                │
│   public CompletableFuture<Void> thenRun(Runnable action)        │
│   
│   ├─ 반환: CompletableFuture<Void>                              │
│   ├─ 의미: 다음 콜백은 더 이상 값 없음                           │
│   └─ 용도: 최종 부작용 (로그, 플래그 설정)                       │
│   
│   CompletableFuture<Integer> cf = cf1.thenApply(x → x * 2);
│   cf.thenRun(() → {})  // Integer는 버려짐
│      .thenRun(() → {})  // 계속 체인 가능 (값 없음)
│      .thenAccept(???)  // 컴파일 에러: Void 값 전달


│ thenAccept(Consumer<T>)                                          │
│   public CompletableFuture<Void> thenAccept(Consumer<T> action)  │
│   
│   ├─ 반환: CompletableFuture<Void>                              │
│   ├─ 의미: 다음 콜백은 값 없음                                  │
│   └─ 용도: 값 소비 후 종료 또는 추가 부작용                     │
│   
│   CompletableFuture<Integer> cf = cf1.thenApply(x → x * 2);
│   cf.thenAccept(value → log.info("{}", value))
│      .thenRun(() → {})  // 값 없으므로 가능


│ thenApply(Function<T,U>)                                         │
│   public <U> CompletableFuture<U> thenApply(Function<T,U> fn)   │
│   
│   ├─ 반환: CompletableFuture<U>  (새로운 타입)                  │
│   ├─ 의미: 다음 콜백은 U 타입의 값 전달                         │
│   └─ 용도: 값 변환, 다음 단계로 전파                             │
│   
│   CompletableFuture<Integer> cf = cf1.thenApply(x → x * 2);
│   cf.thenApply(x → "Value: " + x)  // Integer → String
│      .thenApply(String::toUpperCase)  // String 체인
│      .thenAccept(System.out::println)  // 최종 값 사용
```

### 4. Consumer vs Function 시맨틱

```
Consumer<T>: T → void (부작용 함수)
  - 반환값이 없음 = 값 소비만
  - 사이드 이펙트 (I/O, DB, 캐시 등)
  - 체인 종료 또는 로깅/모니터링 용도
  
  예: value → log.info("{}", value)
      value → db.save(value)
      value → cache.put("key", value)

Function<T,R>: T → R (순수 함수)
  - 반환값이 있음 = 값 변환
  - 입력을 받아 새로운 값 생성
  - 체인 유지, 다음 단계로 전파
  
  예: value → value * 2
      value → "Value: " + value
      user → user.getId()

비동기 체인에서의 영향:

consumer:
  cf1.thenAccept(v → saveDb(v))  // Void 반환
    .thenRun(() → {})  // v 사용 불가능
    .thenAccept(null?)  // 타입 에러

function:
  cf1.thenApply(v → transform(v))  // U 반환
    .thenApply(u → enriched(u))  // u 사용 가능
    .thenAccept(System.out::println)  // 최종 소비
```

---

## 💻 실전 실험

### 실험 1: 반환 타입 차이 확인

```java
@Test
public void testReturnTypeDifference() throws Exception {
    CompletableFuture<Integer> cf = CompletableFuture.completedFuture(10);
    
    // thenRun: CompletableFuture<Void>
    CompletableFuture<Void> voidCF1 = cf.thenRun(() → 
        System.out.println("Done")
    );
    System.out.println("thenRun return: " + voidCF1.getClass());
    
    // thenAccept: CompletableFuture<Void>
    CompletableFuture<Void> voidCF2 = cf.thenAccept(v → 
        System.out.println("Value: " + v)
    );
    System.out.println("thenAccept return: " + voidCF2.getClass());
    
    // thenApply: CompletableFuture<String>
    CompletableFuture<String> stringCF = cf.thenApply(v → 
        "Value: " + v
    );
    System.out.println("thenApply return: " + stringCF.getClass());
    System.out.println("thenApply result: " + stringCF.get());
    
    // 체인 가능성
    voidCF1.thenRun(() → System.out.println("Chain 1"));
    // voidCF1.thenAccept(x → ...);  // 컴파일 에러: Void 값
    stringCF.thenApply(s → s.toUpperCase()).thenAccept(System.out::println);
}

// 결과:
// thenRun return: class java.util.concurrent.CompletableFuture
// thenAccept return: class java.util.concurrent.CompletableFuture
// thenApply return: class java.util.concurrent.CompletableFuture
// thenApply result: Value: 10
```

### 실험 2: 함수형 인터페이스 시맨틱

```java
@Test
public void testFunctionalInterfaceSemantics() throws Exception {
    CompletableFuture<Integer> cf = CompletableFuture.completedFuture(5);
    
    List<String> log = Collections.synchronizedList(new ArrayList<>());
    
    // Runnable: 입력 없음
    CompletableFuture<Void> r1 = cf.thenRun(() → {
        log.add("Runnable executed");
        // 값 접근 불가능
    });
    
    // Consumer: 입력 있음, 반환 없음
    CompletableFuture<Void> c1 = cf.thenAccept(value → {
        log.add("Consumer received: " + value);
        // 반환 불가능
    });
    
    // Function: 입력 있음, 반환 있음
    CompletableFuture<String> f1 = cf.thenApply(value → {
        log.add("Function received: " + value);
        return "Transformed: " + value;
    });
    
    r1.get();
    c1.get();
    String result = f1.get();
    
    log.forEach(System.out::println);
    System.out.println("Final result: " + result);
}

// 결과:
// Runnable executed
// Consumer received: 5
// Function received: 5
// Final result: Transformed: 5
```

### 실험 3: 체인에서의 타입 전파

```java
@Test
public void testTypeChaining() throws Exception {
    CompletableFuture<Integer> cf = CompletableFuture.completedFuture(10);
    
    CompletableFuture<String> chained = cf
        .thenApply(x → x * 2)  // Integer → Integer (20)
        .thenApply(x → "Result: " + x)  // Integer → String
        .thenApply(String::toUpperCase);  // String → String
    
    System.out.println("Final: " + chained.get());  // RESULT: 20
    
    // thenAccept 호출 시 값 타입 확인
    chained.thenAccept(s → {
        System.out.println("Accept string: " + s);
        System.out.println("String length: " + s.length());
    }).get();
    
    // 잘못된 체인 시도
    // CompletableFuture<Void> voidCF = cf.thenAccept(System.out::println);
    // voidCF.thenAccept(x → ...);  // 컴파일 에러
}

// 결과:
// Final: RESULT: 20
// Accept string: RESULT: 20
// String length: 10
```

---

## 📊 성능/비교

```
╔════════════════════════════════════════════════════════════════════╗
║ thenRun vs thenAccept vs thenApply 선택 가이드                      ║
╠════════════════════════════════════════════════════════════════════╣
║ 메서드     │ 입력  │ 반환    │ 반환 타입          │ 용도              ║
╠════════════════════════════════════════════════════════════════════╣
║ thenRun    │ X    │ void   │ CompletableFuture<Void> │ 최종 부작용    ║
║ thenAccept │ O    │ void   │ CompletableFuture<Void> │ 값 소비        ║
║ thenApply  │ O    │ O      │ CompletableFuture<U>    │ 값 변환        ║
╚════════════════════════════════════════════════════════════════════╝

메모리 오버헤드 (콜백 1개 기준):
  thenRun:    ~180B (Runnable 래퍼)
  thenAccept: ~200B (Consumer 래퍼)
  thenApply:  ~220B (Function 래퍼 + 반환값)

실행 시간:
  모두 동일 (O(1) 콜백 등록, O(1) 콜백 실행)

스택 크기 (콜백 1000개 체인):
  thenRun:    O(1) - Void 값 전파 최소
  thenAccept: O(1) - Void 값 전파 최소
  thenApply:  O(1) - 각 U 값 저장 (변환 결과)
```

---

## ⚖️ 트레이드오프

| 메서드 | 장점 | 단점 | 선택 기준 |
|--------|------|------|---------|
| **thenRun** | 간단, 명확한 의도 | 값 사용 불가 | 최종 후작업 (로그, 플래그) |
| **thenAccept** | 값 접근, 부작용 표현 | 값 변환 불가 | 값 소비 (DB, 캐시 저장) |
| **thenApply** | 값 변환, 체인 유지 | 타입 관리 필요 | 데이터 변환, 다음 단계 |

---

## 📌 핵심 정리

1. **thenRun**: `Runnable` — 입력 없음, 부작용만, 최종 작업

2. **thenAccept**: `Consumer<T>` — 입력 O, 반환 X, 값 소비

3. **thenApply**: `Function<T,U>` — 입력 O, 반환 O, 값 변환

4. **반환 타입의 의미**: `<Void>`는 다음 값 없음, `<U>`는 U 타입 전파

5. **함수형 인터페이스 선택**: 부작용만? Consumer 또는 Runnable / 변환 필요? Function

6. **체인 설계**: 변환(thenApply) → 소비(thenAccept) → 부작용(thenRun)

7. **타입 안정성**: 컴파일러가 반환 타입으로 의도를 강제

---

## 🤔 생각해볼 문제

<details>
<summary>Q1: 왜 thenRun과 thenAccept 모두 CompletableFuture<Void>를 반환하는가?</summary>

**해설**: 둘 다 부작용만 처리하고 값을 반환하지 않으므로, 다음 콜백에 전달할 값이 없다. 따라서 `Void`로 통일된다.

차이:
- `thenRun`: 이전 값을 사용할 필요 없을 때 (로그만 기록)
- `thenAccept`: 이전 값을 사용할 필요 있을 때 (값 기반 부작용)

둘 다 `CompletableFuture<Void>` 반환이므로, 이후 체인은 다시 thenRun만 가능하다.

```java
cf.thenAccept(v → log.info("{}", v))  // 값 사용
  .thenRun(() → resetFlag())  // 값 없음
  .thenRun(() → close());  // 계속 부작용
```

</details>

<details>
<summary>Q2: thenApply 체인에서 중간에 부작용을 삽입하려면?</summary>

**해설**: `thenApply` 내에서 부작용을 수행한 후 값을 반환하거나, `thenAccept`로 일시적으로 벗어났다가 다시 들어온다.

```java
// 방식 1: thenApply 내에서 부작용 처리
cf.thenApply(v → {
    log.info("Value: {}", v);  // 부작용
    return v * 2;  // 반환
})

// 방식 2: thenAcceptAsync + thenApply로 분리
cf.thenAccept(v → cache.put("key", v))  // Void 반환
  // ??? 여기서 값에 다시 접근 불가능

// 방식 3: peek 패턴 (Java Stream에서 영감)
cf.thenApply(v → {
    log.info("Value: {}", v);  // 부작용
    return v;  // 값 그대로 반환
})
```

실무에서는 로깅/캐시 갱신은 `thenAcceptAsync`로 분리하고, 변환은 `thenApply`로 진행한다.

</details>

<details>
<summary>Q3: 왜 Function의 반환 타입을 명시해야 하는가?</summary>

**해설**: 제네릭 타입 시스템을 활용해 다음 콜백에 정확한 타입을 전달하기 위해서다.

```java
CompletableFuture<Integer> cf = CompletableFuture.completedFuture(10);

// thenApply: 반환 타입이 컴파일 타임에 결정됨
CompletableFuture<String> cf1 = cf.thenApply(v → "Value: " + v);
// 컴파일러: String 타입 추론 → cf1 = CF<String>

// thenApply 체인: 타입 안정성 보장
cf.thenApply(v → v * 2)  // CF<Integer>
  .thenApply(v → "Result: " + v)  // CF<String>
  .thenApply(String::length)  // CF<Integer>
  .thenAccept(System.out::println);  // Integer 전달됨

// 타입 없이 구현했다면?
// Runtime에 ClassCastException 가능
// 또는 일일이 cast 필요
```

제네릭 타입이 컴파일 타임 타입 체크를 가능하게 한다.

</details>

<div align="center">

**[⬅️ 이전: thenApply vs thenCompose vs thenCombine](./02-thenapply-thencompose-thencombine.md)** | **[홈으로 🏠](../README.md)** | **[다음: Async 변형과 Executor 지정 ➡️](./04-async-executor-strategy.md)**

</div>
