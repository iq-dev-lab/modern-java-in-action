# 예외 처리 3가지 — exceptionally, handle, whenComplete

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `exceptionally(Throwable → T)`, `handle(BiFunction<T, Throwable, U>)`, `whenComplete(BiConsumer<T, Throwable>)`의 정확한 차이는?
- 예외 발생 시에만 콜백을 실행하려면 (exceptionally)?
- 성공과 실패 모두 처리하되 값을 변환하려면 (handle)?
- 성공과 실패 모두 부작용을 처리하려면 (whenComplete)?
- 예외 체인 전파와 복구 메커니즘은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

CompletableFuture의 비동기 체인에서 예외는 원인 지점에서 발생하지만, 처리 시점은 콜백 체인 어딘가다. 올바르지 않은 예외 처리는 silent failure를 야기하거나, 부분적 롤백만 수행되거나, 리소스가 정리되지 않는다. 세 가지 메서드의 정확한 의미 차이를 알아야 프로덕션 안정성을 확보할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: exceptionally에서 예외를 던짐
  cf.exceptionally(ex → {
      log.error("Error", ex);
      throw ex;  // ← 다시 던짐
  })
  
  → 결과 CF는 여전히 예외 상태
  → 다음 콜백도 실행되지 않음
  → 원치 않은 예외 전파

실수 2: whenComplete에서 값을 변환하려고 시도
  cf.whenComplete((value, ex) → {
      if (ex == null) {
          return value * 2;  // ← 반환 불가능
      }
  })
  
  → whenComplete는 void 반환 (BiConsumer)
  → 값 변환 불가능
  → handle() 사용 필요

실수 3: handle에서 예외를 무시
  cf.handle((value, ex) → {
      if (ex != null) {
          return null;  // ← 예외 무시, null 반환
      }
      return value;
  })
  
  → 예외 처리 로그 없음
  → 모니터링 불가능
  → Silent failure 발생

실수 4: whenComplete와 get()의 조합
  cf.whenComplete((v, ex) → {
      System.out.println("Completed");
  })
  .get();  // ← 예외 발생 시 블로킹됨
  
  → whenComplete는 예외를 복구하지 않음
  → get()은 여전히 예외를 던짐
  → 콜백의 로깅이 의미 없음
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 1. exceptionally: 예외만 처리, 복구
CompletableFuture<String> result = fetchUser()
    .thenApply(user → user.getName())
    .exceptionally(ex → {
        log.warn("User fetch failed, using default", ex);
        return "Anonymous";  // 복구 값 반환
    });

// 결과: "Anonymous" (예외에서 복구)

// 2. handle: 성공과 실패 모두 처리, 변환 가능
CompletableFuture<String> result2 = fetchUser()
    .thenApply(user → user.getName())
    .handle((value, ex) → {
        if (ex != null) {
            log.error("Failed", ex);
            return "Error: " + ex.getMessage();  // 예외 처리
        } else {
            return "Success: " + value;  // 성공 처리
        }
    });

// 3. whenComplete: 성공과 실패 모두, 부작용만
CompletableFuture<String> result3 = fetchUser()
    .thenApply(user → user.getName())
    .whenComplete((value, ex) → {
        if (ex != null) {
            log.error("Operation failed", ex);  // 로깅
            metrics.recordFailure();  // 메트릭
        } else {
            log.info("Operation succeeded with: " + value);
            metrics.recordSuccess();
        }
    });

// 결과: 부작용만 (값 변환 없음, 원본 값 유지 또는 예외 유지)

// 4. 실무: 예외 처리 파이프라인
CompletableFuture<String> userResult = fetchUser()
    .exceptionally(ex → {
        log.warn("Primary user fetch failed, trying fallback", ex);
        return fallbackUser();  // 복구
    })
    .thenApply(user → user.getName())
    .handle((name, ex) → {
        if (ex != null) {
            log.error("Name processing failed", ex);
            return "Unknown";
        }
        return name;
    })
    .whenComplete((name, ex) → {
        // 로그만, 값은 그대로
        log.info("Final result: " + name);
    });
```

---

## 🔬 내부 동작 원리

### 1. 세 메서드의 시그니처와 의미

```java
// 1. exceptionally: 예외만 처리
public CompletableFuture<T> exceptionally(
    Function<Throwable, ? extends T> fn) {
  
  입력: Throwable (예외만)
  반환: T (복구 값)
  시그니처: Throwable → T
  
  동작:
    - CF 상태가 예외? → fn(exception) 실행
    - CF 상태가 정상? → fn 실행 안 함, 원본 값 그대로 반환

// 2. handle: 성공/실패 모두 처리
public <U> CompletableFuture<U> handle(
    BiFunction<? super T, Throwable, ? extends U> fn) {
  
  입력: (T or null, Throwable or null)
       → 성공: (value, null)
       → 실패: (null, exception)
  반환: U (변환된 값)
  시그니처: (T, Throwable) → U
  
  동작:
    - 항상 fn 실행 (성공/실패 관계없음)
    - fn의 반환값으로 새 CF 완료

// 3. whenComplete: 성공/실패 모두, 부작용만
public CompletableFuture<T> whenComplete(
    BiConsumer<? super T, ? super Throwable> action) {
  
  입력: (T or null, Throwable or null)
  반환: void (부작용만)
  시그니처: (T, Throwable) → void
  
  동작:
    - 항상 action 실행 (성공/실패 관계없음)
    - 원본 값 또는 예외 그대로 유지 (변환 불가)
```

### 2. 예외 처리 메커니즘

```
CompletableFuture의 예외 저장:

  public class CompletableFuture<T> {
      volatile Object result;  // null | T | AltResult(Throwable)
  }
  
  static final class AltResult {
      final Throwable ex;
      AltResult(Throwable x) { this.ex = x; }
  }

  예외 발생 시:
    result = AltResult(new RuntimeException("error"))

예외 처리 콜백 (UniExceptionally):

  class UniExceptionally extends UniCompletion<T,T> {
      Function<Throwable, ? extends T> fn;
      
      public CompletableFuture<T> tryFire(int mode) {
          CompletableFuture<T> src = source;
          CompletableFuture<T> dst = dep;
          
          Object r;
          if ((r = src.result) != null && dst != null) {
              if (r instanceof AltResult) {
                  // 예외 상태
                  Throwable ex = ((AltResult)r).ex;
                  try {
                      T value = fn.apply(ex);  // 복구 함수 실행
                      dst.completeValue(value);
                  } catch (Throwable ex2) {
                      dst.completeExceptionally(ex2);  // 복구 중 실패
                  }
              } else {
                  // 정상 상태 - fn 실행 안 함
                  dst.completeValue((T)r);
              }
          }
          return null;
      }
  }

handle 콜백 (BiHandle):

  class BiHandle extends BiCompletion<T,T,U> {
      BiFunction<? super T, Throwable, ? extends U> fn;
      
      public CompletableFuture<U> tryFire(int mode) {
          CompletableFuture<T> src = source;
          CompletableFuture<U> dst = dep;
          
          Object r;
          if ((r = src.result) != null && dst != null) {
              T t;
              Throwable ex;
              
              if (r instanceof AltResult) {
                  // 예외 상태
                  t = null;
                  ex = ((AltResult)r).ex;
              } else {
                  // 정상 상태
                  t = (T)r;
                  ex = null;
              }
              
              try {
                  U u = fn.apply(t, ex);  // 항상 실행
                  dst.completeValue(u);
              } catch (Throwable ex2) {
                  dst.completeExceptionally(ex2);
              }
          }
          return null;
      }
  }

whenComplete 콜백 (BiWhenComplete):

  class BiWhenComplete extends BiCompletion<T,T,T> {
      BiConsumer<? super T, ? super Throwable> action;
      
      public CompletableFuture<T> tryFire(int mode) {
          CompletableFuture<T> src = source;
          CompletableFuture<T> dst = dep;
          
          Object r;
          if ((r = src.result) != null && dst != null) {
              T t;
              Throwable ex;
              
              if (r instanceof AltResult) {
                  t = null;
                  ex = ((AltResult)r).ex;
              } else {
                  t = (T)r;
                  ex = null;
              }
              
              try {
                  action.accept(t, ex);  // 부작용만
              } catch (Throwable ex2) {
                  dst.completeExceptionally(ex2);  // action 중 예외
              }
              
              // 원본 상태 그대로 전파
              if (r instanceof AltResult) {
                  dst.completeExceptionally(ex);
              } else {
                  dst.completeValue(t);
              }
          }
          return null;
      }
  }
```

### 3. 예외 전파 경로

```
┌─────────────────────────────────────────────────────┐
│ 예외 전파 다이어그램                                  │
└─────────────────────────────────────────────────────┘

1. exceptionally: 예외 → 복구 → 정상

   CF1 (정상) ─→ [변환] ─→ CF2 (정상)
                    │
                   실패 (예외)
                    │
                    ↓
   CF1 (예외) ─→ [exceptionally] ─→ CF2 (복구값, 정상)
                         or
   CF1 (예외) ─→ [exceptionally] ─→ CF2 (새 예외)


2. handle: 예외 → 값 변환

   CF1 (정상) ─→ [fn(value, null)] ─→ CF2 (변환값)
   CF1 (예외) ─→ [fn(null, ex)] ─→ CF2 (변환값)
   
   예: handle((v, ex) → ex != null ? "Error" : v)
       CF1 (예외) → CF2 (정상: "Error")


3. whenComplete: 부작용, 상태 유지

   CF1 (정상) ─→ [action(value, null)] ─→ CF2 (정상, 원본값)
                 부작용만 (로깅, 메트릭)
   
   CF1 (예외) ─→ [action(null, ex)] ─→ CF2 (예외, 원본예외)
                 부작용만 (로깅, 메트릭)


예외 체인:

  fetchUser()  // CF<User> (예외: UserNotFound)
    .thenApply(user → user.getName())
    // ↓ 예외 전파 (변환 실행 안 함)
    .exceptionally(ex → {  // "예외만 처리"
        if (ex instanceof UserNotFound) {
            return "Guest";  // 복구
        }
        throw ex;  // 다른 예외는 다시 던짐
    })
    // ↓ 정상값 "Guest"
    .handle((name, ex) → {  // "성공과 실패 모두"
        // name = "Guest" (복구된 값)
        // ex = null
        return "Name: " + name;
    })
    // ↓ "Name: Guest"
    .whenComplete((result, ex) → {  // "부작용만"
        log.info("Final: {}", result);
    })
    // ↓ "Name: Guest" (값 그대로)
```

---

## 💻 실전 실험

### 실험 1: exceptionally 동작 확인

```java
@Test
public void testExceptionally() throws Exception {
    // 케이스 1: 정상 완료
    CompletableFuture<Integer> cf1 = CompletableFuture.completedFuture(10)
        .exceptionally(ex → {
            System.out.println("exceptionally called");  // 실행 안 됨
            return -1;
        });
    
    System.out.println("Case 1 result: " + cf1.get());  // 10
    
    // 케이스 2: 예외 발생
    CompletableFuture<Integer> cf2 = CompletableFuture.<Integer>failedFuture(
        new RuntimeException("Error")
    ).exceptionally(ex → {
        System.out.println("exceptionally called");  // 실행 됨
        return -1;
    });
    
    System.out.println("Case 2 result: " + cf2.get());  // -1
}

// 결과:
// Case 1 result: 10
// exceptionally called
// Case 2 result: -1
```

### 실험 2: handle의 성공/실패 모두 처리

```java
@Test
public void testHandle() throws Exception {
    // 성공 경로
    CompletableFuture<String> cf1 = CompletableFuture.completedFuture(100)
        .handle((value, ex) → {
            if (ex != null) {
                return "Error: " + ex.getMessage();
            }
            return "Value: " + value;
        });
    
    System.out.println("Success case: " + cf1.get());  // Value: 100
    
    // 실패 경로
    CompletableFuture<String> cf2 = CompletableFuture.<Integer>failedFuture(
        new RuntimeException("Oops")
    ).handle((value, ex) → {
        if (ex != null) {
            return "Error: " + ex.getMessage();
        }
        return "Value: " + value;
    });
    
    System.out.println("Failure case: " + cf2.get());  // Error: Oops
}

// 결과:
// Success case: Value: 100
// Failure case: Error: Oops
```

### 실험 3: whenComplete는 값을 변환하지 않음

```java
@Test
public void testWhenCompletePreservesValue() throws Exception {
    // 정상 경로
    CompletableFuture<Integer> cf1 = CompletableFuture.completedFuture(42)
        .whenComplete((value, ex) → {
            System.out.println("Processing value: " + value);
            // 반환값이 영향 없음
        })
        .thenApply(v → v * 2);  // 원본값 42가 전파됨
    
    System.out.println("Result: " + cf1.get());  // 84
    
    // 예외 경로
    CompletableFuture<Integer> cf2 = CompletableFuture.<Integer>failedFuture(
        new RuntimeException("Error")
    ).whenComplete((value, ex) → {
        System.out.println("Exception occurred: " + ex.getMessage());
        // 여기서 예외를 처리해도 다음 콜백도 예외 받음
    });
    
    try {
        cf2.get();
    } catch (ExecutionException e) {
        System.out.println("Caught: " + e.getCause().getMessage());
        // Error
    }
}

// 결과:
// Processing value: 42
// Result: 84
// Exception occurred: Error
// Caught: Error
```

### 실험 4: 예외 복구 체인

```java
@Test
public void testExceptionRecoveryChain() throws Exception {
    CompletableFuture<String> result = fetchUser()
        .exceptionally(ex → {
            log.info("Primary fetch failed, retrying...");
            return fetchUserFromBackup();
        })
        .thenApply(user → user.toUpperCase())
        .exceptionally(ex → {
            log.info("Transform failed, using default");
            return "DEFAULT";
        })
        .whenComplete((value, ex) → {
            log.info("Final result: " + value);
        });
    
    // 각 단계에서 예외 처리 가능, whenComplete는 최종 로깅
}

private CompletableFuture<String> fetchUser() {
    return CompletableFuture.failedFuture(new RuntimeException("API down"));
}

private String fetchUserFromBackup() {
    return "Alice";
}
```

---

## 📊 성능/비교

```
╔════════════════════════════════════════════════════════════════════╗
║ 예외 처리 메서드 선택 가이드                                        ║
╠════════════════════════════════════════════════════════════════════╣
║ 메서드      │ 입력           │ 반환    │ 경우       │ 용도         ║
╠════════════════════════════════════════════════════════════════════╣
║ exceptionally│ Throwable   │ T      │ 예외만    │ 예외 복구     ║
║ handle      │ (T, Throwable)│ U      │ 모두      │ 성공/실패처리║
║ whenComplete│ (T, Throwable)│ void   │ 모두      │ 부작용(로깅) ║
╚════════════════════════════════════════════════════════════════════╝

메모리 오버헤드:
  exceptionally:  ~200B (Function 래퍼)
  handle:         ~220B (BiFunction 래퍼)
  whenComplete:   ~220B (BiConsumer 래퍼)

실행 시간:
  모두 동일 (O(1) 콜백 등록, O(1) 실행)

예외 처리 체인의 뎁스:

  cf.exceptionally(...).exceptionally(...).exceptionally(...)
  
  각 exceptionally는 새 CF를 생성하므로,
  깊은 체인 시 메모리 증가
  
  대신 하나의 handle이나 직접 try-catch가 더 효율적
```

---

## ⚖️ 트레이드오프

| 메서드 | 장점 | 단점 | 선택 기준 |
|--------|------|------|---------|
| **exceptionally** | 간단, 예외만 처리 | 성공 경로 처리 불가 | 예외 복구만 필요 |
| **handle** | 성공/실패 모두, 값 변환 | 복잡한 로직 필요 | 조건부 처리 필요 |
| **whenComplete** | 최종 부작용(로깅) | 값 변환 불가 | 모니터링/정리 작업 |

---

## 📌 핵심 정리

1. **exceptionally**: `Throwable → T` — 예외만 처리, 복구 값 반환

2. **handle**: `(T, Throwable) → U` — 성공/실패 모두, 값 변환 가능

3. **whenComplete**: `(T, Throwable) → void` — 성공/실패 모두, 부작용만

4. **예외 전파**: exceptionally로 복구 → 다음 콜백 정상 실행

5. **값 유지**: whenComplete는 원본 값 또는 예외 그대로 전파

6. **복합 처리**: 복구 → 변환 → 부작용 순서로 체인 구성

7. **실무 패턴**: 예외 복구(exceptionally) + 조건 처리(handle) + 모니터링(whenComplete)

---

## 🤔 생각해볼 문제

<details>
<summary>Q1: exceptionally에서 새로운 예외를 던지면?</summary>

**해설**: 새로운 예외로 상태가 변경되고, 다음 콜백으로 전파된다.

```java
cf.exceptionally(ex → {
    throw new RuntimeException("Recovery failed", ex);  // 새 예외
})
.exceptionally(ex2 → {
    // ex2는 RuntimeException (원본 예외 X)
    return "Fallback";
});

// exceptionally는 예외를 다시 던질 수 있으며,
// 그 예외가 다음 콜백으로 전파된다.
```

</details>

<details>
<summary>Q2: whenComplete에서 예외를 던지면?</summary>

**해설**: whenComplete 자체의 예외가 원본 예외를 덮어써(suppress) 복합 예외가 발생한다.

```java
cf.whenComplete((value, ex) → {
    throw new RuntimeException("Logging failed");
})
.exceptionally(ex → {
    // ex의 원인(cause)은 원본 예외
    // 하지만 직접 예외는 "Logging failed"
    return "Recovered";
});
```

이러한 이유로 whenComplete 내부에서는 예외를 던지지 않아야 한다.

</details>

<details>
<summary>Q3: handle에서 null을 반환하면?</summary>

**해설**: null이 정상 값으로 처리되고, 다음 콜백으로 전파된다.

```java
cf.handle((value, ex) → {
    if (ex != null) {
        return null;  // 예외를 null로 변환
    }
    return value;
})
.thenApply(v → {
    if (v == null) {
        // handle에서 반환한 null
    }
    return v;
});

// 이는 예외를 무시한 것처럼 보이므로 권장되지 않음
// 대신 명시적 값(예: "DEFAULT", -1 등) 반환 권장
```

</details>

<div align="center">

**[⬅️ 이전: Async 변형과 Executor 지정](./04-async-executor-strategy.md)** | **[홈으로 🏠](../README.md)** | **[다음: allOf · anyOf ➡️](./06-allof-anyof-patterns.md)**

</div>
