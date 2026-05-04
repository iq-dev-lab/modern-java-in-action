# Structured Concurrency (Java 21 Preview)

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `StructuredTaskScope`로 부모-자식 task 관계를 구조화하면 무엇이 달라지는가?
- `ShutdownOnFailure`와 `ShutdownOnSuccess` 정책은 언제 어떻게 쓰는가?
- 부모 스코프가 종료되면 자식 task는 자동으로 취소되는 메커니즘은?
- 기존 `ExecutorService` + `CompletableFuture` 패턴의 문제점은 무엇인가?
- ThreadLocal 메모리 누수를 어떻게 방지하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Virtual Thread와 Structured Concurrency는 함께 나온 개념이다. VT로 수백만 개 동시 task를 만들 수 있지만, 그것들을 관리하는 것이 문제다. Structured Concurrency는 task의 생명주기를 명확히 하여, 자동으로 리소스를 정리하고 누수를 방지한다. 마이크로서비스 아키텍처에서 여러 백엔드를 동시 호출할 때 필수 패턴이다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: CompletableFuture로 task를 마음껏 만들면 관리된다고 생각
  ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
  
  List<CompletableFuture<String>> futures = new ArrayList<>();
  for (int i = 0; i < 1_000_000; i++) {
      futures.add(CompletableFuture.supplyAsync(() -> {
          return callBackend(i);
      }, executor));
  }
  
  // 100만 개 future를 다 기다리면?
  CompletableFuture.allOf(futures.toArray(...)).join();
  
  → 문제 1: 하나라도 실패하면? 나머지는 계속 실행됨
  → 문제 2: timeout이 없으면 무한 대기 가능
  → 문제 3: 취소 메커니즘이 불분명 (cancel() 호출해야 하는데, 까먹으면 누수)

실수 2: Task를 완료했는데 관련 리소스(DB connection 등)가 남음
  // task가 exception으로 종료되면 리소스 정리 안 될 수 있음
  CompletableFuture<String> f = CompletableFuture.supplyAsync(() -> {
      Connection conn = getConnection();  // DB 연결
      // exception 발생 → conn 정리 안 됨 (누수)
      return query(conn);
  });

실제 버그:

public List<UserData> getUsersFromMultipleBackends() {
    List<CompletableFuture<UserData>> futures = new ArrayList<>();
    
    for (int userId : userIds) {
        futures.add(CompletableFuture.supplyAsync(() -> {
            return backend1.fetchUser(userId);  // 500ms
        }));
        futures.add(CompletableFuture.supplyAsync(() -> {
            return backend2.fetchUser(userId);  // 500ms (timeout 없음)
        }));
        // ...
    }
    
    return CompletableFuture.allOf(futures.toArray(...))
        .join();  // 모두 완료 대기 (무한 가능)
    
    // 결과: 하나의 backend가 응답 안 하면 전체 요청 hanging
}
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
import java.util.concurrent.StructuredTaskScope.*;
import java.util.*;

public class StructuredConcurrencyExample {
    
    // 패턴 1: 모두 성공해야 (ShutdownOnFailure)
    public UserProfile fetchUserProfile(int userId) 
            throws ExecutionException, InterruptedException {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            Future<User> userFuture = scope.fork(() -> 
                backendA.getUser(userId));
            Future<Address> addressFuture = scope.fork(() -> 
                backendB.getAddress(userId));
            Future<Preferences> prefsFuture = scope.fork(() -> 
                backendC.getPreferences(userId));
            
            scope.join();  // 모두 완료 또는 하나 실패
            scope.throwIfFailed();  // 실패했으면 exception
            
            return new UserProfile(
                userFuture.resultNow(),
                addressFuture.resultNow(),
                prefsFuture.resultNow()
            );
        }  // 스코프 종료: 완료되지 않은 task 모두 취소, 리소스 정리
    }
    
    // 패턴 2: 하나 성공하면 나머지 취소 (ShutdownOnSuccess)
    public String getImageFromFastestServer(String imageId) 
            throws ExecutionException, InterruptedException {
        try (var scope = new StructuredTaskScope.ShutdownOnSuccess<>()) {
            scope.fork(() -> serverA.getImage(imageId));  // 500ms
            scope.fork(() -> serverB.getImage(imageId));  // 200ms (승리)
            scope.fork(() -> serverC.getImage(imageId));  // 400ms
            
            scope.join();  // 하나 성공하면 즉시 반환
            
            return scope.result();  // 가장 빠른 결과 반환
        }  // 나머지 task 자동 취소
    }
    
    // 패턴 3: 커스텀 정책 (StructuredTaskScope.Handler)
    public List<Result> fetchWithCustomPolicy(List<Integer> ids) 
            throws ExecutionException, InterruptedException {
        try (var scope = new StructuredTaskScope<Result>()) {
            List<Future<Result>> futures = new ArrayList<>();
            
            for (int id : ids) {
                futures.add(scope.fork(() -> fetchData(id)));
            }
            
            scope.join();  // 모두 완료 대기
            
            // 부분 실패 허용
            return futures.stream()
                .map(f -> {
                    try {
                        return f.resultNow();
                    } catch (Exception e) {
                        return Result.EMPTY;  // 실패 시 기본값
                    }
                })
                .collect(Collectors.toList());
        }  // 스코프 종료: 모든 리소스 정리
    }
    
    // 패턴 4: 중첩된 스코프 (부모-자식 관계 구조화)
    public AggregatedData fetchAggregated(List<String> regions) 
            throws ExecutionException, InterruptedException {
        try (var parentScope = new StructuredTaskScope.ShutdownOnFailure()) {
            List<Future<RegionData>> results = new ArrayList<>();
            
            for (String region : regions) {
                results.add(parentScope.fork(() -> {
                    // 자식 스코프: 같은 region의 여러 백엔드 호출
                    try (var childScope = new StructuredTaskScope.ShutdownOnFailure()) {
                        var user = childScope.fork(() -> 
                            userService.getUser(region));
                        var product = childScope.fork(() -> 
                            productService.getProduct(region));
                        
                        childScope.join();
                        childScope.throwIfFailed();
                        
                        return new RegionData(
                            user.resultNow(),
                            product.resultNow()
                        );
                    }
                }));
            }
            
            parentScope.join();
            parentScope.throwIfFailed();
            
            return new AggregatedData(
                results.stream()
                    .map(Future::resultNow)
                    .collect(Collectors.toList())
            );
        }  // 모든 자식 스코프도 자동 정리
    }
}
```

---

## 🔬 내부 동작 원리

### 1. StructuredTaskScope 구조

```
StructuredTaskScope 생명주기:

┌────────────────────────────────────────────────┐
│ try (var scope = new StructuredTaskScope()) {  │
│   ┌──────────────────────────────────────────┐ │
│   │ OPEN: 새로운 task 수용                    │ │
│   │                                          │ │
│   │ scope.fork(() -> task1)  → Future[1]     │ │
│   │ scope.fork(() -> task2)  → Future[2]     │ │
│   │ scope.fork(() -> task3)  → Future[3]     │ │
│   │                                          │ │
│   │ 내부 상태:                               │ │
│   │ futures = [VT1, VT2, VT3]                │ │
│   │ completed = 0, failed = null             │ │
│   └──────────────────────────────────────────┘ │
│           ↓                                     │
│   scope.join()  // 모든 task 완료 대기         │
│           ↓                                     │
│   ┌──────────────────────────────────────────┐ │
│   │ CLOSED: 새로운 task 거부                 │ │
│   │                                          │ │
│   │ (VT1 완료, VT2 완료, VT3 완료)           │ │
│   │ completed = 3, failed = null (성공)      │ │
│   │                                          │ │
│   │ scope.result() 또는                       │ │
│   │ scope.throwIfFailed() 사용               │ │
│   └──────────────────────────────────────────┘ │
└─ finally ─────────────────────────────────────┘
  scope.close()
  → 미완료 task 모두 cancel()
  → 리소스 정리 (ThreadLocal 등)
```

### 2. ShutdownOnFailure vs ShutdownOnSuccess

```
ShutdownOnFailure (AND 의미론):

fork() → Future[1]  ──→ 완료
fork() → Future[2]  ──→ 완료
fork() → Future[3]  ──→ EXCEPTION ✗

scope.join()
  ↓
(join 진행 중) Future[3] exception 감지
  ↓
Cancellation 신호 → Future[1], [2]로 전파
  ↓
scope.throwIfFailed() → exception 재발생

사용 시기: 모든 데이터가 필요한 경우
예: 구매 주문 → 결제, 배송, 재고 모두 필수


ShutdownOnSuccess (OR 의미론):

fork() → Future[1]  ──→ 100ms 블로킹
fork() → Future[2]  ──→ 30ms 완료 ✓
fork() → Future[3]  ──→ 500ms 블로킹

scope.join()
  ↓
(join 진행 중) Future[2] 완료 감지
  ↓
Cancellation 신호 → Future[1], [3]로 전파
  ↓
scope.result() → Future[2]의 결과만 반환

사용 시기: 하나의 빠른 응답이면 충분한 경우
예: 가장 빠른 CDN 서버에서 이미지 다운로드


일반 StructuredTaskScope (커스텀 정책):

모든 task가 완료될 때까지 대기
(exception 발생 시에도 join 계속)

scope.join();
foreach future:
  if (future.exception) → 부분 실패 허용
  else → result 수집

사용 시기: 부분 실패를 용인하는 경우
예: 분석 리포트 생성 (일부 데이터 부족해도 생성)
```

### 3. 취소 메커니즘

```
부모 스코프 종료 시 자식 task 취소:

┌─────────────────────────────────────────┐
│ Parent Scope {                          │
│   ┌──────────────────────────────────┐  │
│   │ VT1 = fork(task1) ─→ running... │  │
│   │ VT2 = fork(task2) ─→ socket I/O │  │
│   │ VT3 = fork(task3) ─→ db query   │  │
│   └──────────────────────────────────┘  │
│           ↓                              │
│   (exception 또는 close() 호출)           │
│           ↓                              │
│   Cancellation 신호 전파:                │
│   VT1.cancel() → interrupt flag 설정    │
│   VT2.cancel() → socket.close()         │
│   VT3.cancel() → query 취소              │
│           ↓                              │
│   각 VT에서 InterruptedException 발생   │
│   catch 블록에서 cleanup 코드 실행       │
│           ↓                              │
│   VT1, VT2, VT3 모두 TERMINATED         │
│   리소스 정리 (connection 종료, 메모리) │
└─────────────────────────────────────────┘
```

### 4. ThreadLocal 안전성

```
문제: ExecutorService + CompletableFuture

List<CompletableFuture<Data>> futures = new ArrayList<>();
ThreadLocal<Context> contextLocal = ThreadLocal.withInitial(...);

for (int i = 0; i < 100; i++) {
    futures.add(CompletableFuture.supplyAsync(() -> {
        Context ctx = contextLocal.get();  // 다른 task의 value?
        return process(ctx);
    }, executor));
}

// 위험: VT가 threadlocal 값을 이전 작업에서 남은 값으로 가져올 수 있음
// (또는 다른 VT가 같은 threadlocal에 접근)


해결: StructuredTaskScope

try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Future<Data> f1 = scope.fork(() -> {
        Context ctx = new Context(...);  // 로컬 변수로 관리
        return process(ctx);
    });
    
    Future<Data> f2 = scope.fork(() -> {
        Context ctx = new Context(...);  // 각 fork마다 독립적
        return process(ctx);
    });
    
    scope.join();
}

→ scope 종료 시 자동으로 모든 task 정리
→ ThreadLocal 값이 남지 않음 (각 VT가 종료되면서 정리)
→ 메모리 누수 방지
```

---

## 💻 실전 실험

### 실험 1: ShutdownOnFailure 동작

```java
import java.util.concurrent.*;

public class StructuredConcurrencyTest {
    public static void main(String[] args) {
        try {
            failureExample();
        } catch (Exception e) {
            System.out.println("Exception caught: " + e.getMessage());
        }
    }
    
    static void failureExample() throws ExecutionException, InterruptedException {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            Future<String> f1 = scope.fork(() -> {
                System.out.println("[Task1] 시작");
                Thread.sleep(500);
                System.out.println("[Task1] 완료");
                return "result1";
            });
            
            Future<String> f2 = scope.fork(() -> {
                System.out.println("[Task2] 시작");
                Thread.sleep(100);
                System.out.println("[Task2] 실패 발생!");
                throw new RuntimeException("Task2 실패");
            });
            
            Future<String> f3 = scope.fork(() -> {
                System.out.println("[Task3] 시작");
                Thread.sleep(500);
                System.out.println("[Task3] 완료");
                return "result3";
            });
            
            System.out.println("join() 호출");
            scope.join();
            scope.throwIfFailed();
            
        } catch (ExecutionException e) {
            System.out.println("Task2의 exception으로 인해 join() 중단");
            System.out.println("Task1, Task3는 cancel 신호 수신");
        }
    }
}

// 출력:
// join() 호출
// [Task1] 시작
// [Task2] 시작
// [Task3] 시작
// [Task2] 실패 발생!
// Task2의 exception으로 인해 join() 중단
// Task1, Task3는 cancel 신호 수신
```

### 실험 2: ShutdownOnSuccess 경쟁

```java
import java.util.concurrent.*;

public class RaceExample {
    public static void main(String[] args) throws Exception {
        String fastest = fastestServer();
        System.out.println("가장 빠른 응답: " + fastest);
    }
    
    static String fastestServer() throws ExecutionException, InterruptedException {
        try (var scope = new StructuredTaskScope.ShutdownOnSuccess<String>()) {
            scope.fork(() -> {
                Thread.sleep(500);
                System.out.println("Server A 응답");
                return "A";
            });
            
            scope.fork(() -> {
                Thread.sleep(100);
                System.out.println("Server B 응답");
                return "B";
            });
            
            scope.fork(() -> {
                Thread.sleep(300);
                System.out.println("Server C 응답");
                return "C";
            });
            
            scope.join();
            return scope.result();
        }
    }
}

// 출력:
// Server B 응답
// 가장 빠른 응답: B
// (Server A, C의 task는 cancel됨)
```

### 실험 3: 중첩된 스코프

```java
import java.util.concurrent.*;

public class NestedScopeExample {
    public static void main(String[] args) throws Exception {
        var result = fetchWithNesting();
        System.out.println("최종 결과: " + result);
    }
    
    static String fetchWithNesting() throws ExecutionException, InterruptedException {
        try (var parentScope = new StructuredTaskScope.ShutdownOnFailure()) {
            var f1 = parentScope.fork(() -> {
                // 자식 스코프 1
                try (var childScope = new StructuredTaskScope.ShutdownOnFailure()) {
                    var sub1 = childScope.fork(() -> {
                        Thread.sleep(100);
                        return "child1-sub1";
                    });
                    var sub2 = childScope.fork(() -> {
                        Thread.sleep(200);
                        return "child1-sub2";
                    });
                    childScope.join();
                    return sub1.resultNow() + " + " + sub2.resultNow();
                }
            });
            
            var f2 = parentScope.fork(() -> {
                // 자식 스코프 2
                try (var childScope = new StructuredTaskScope.ShutdownOnFailure()) {
                    var sub3 = childScope.fork(() -> {
                        Thread.sleep(50);
                        return "child2-sub3";
                    });
                    childScope.join();
                    return sub3.resultNow();
                }
            });
            
            parentScope.join();
            return f1.resultNow() + " | " + f2.resultNow();
        }
    }
}

// 출력:
// 최종 결과: child1-sub1 + child1-sub2 | child2-sub3
```

---

## 📊 성능/비교

| 항목 | ExecutorService + CompletableFuture | StructuredTaskScope |
|------|-------------------------------------|-------------------|
| 취소 메커니즘 | 명시적 (cancel() 호출) | 자동 (scope 종료) |
| 리소스 정리 | 수동 (finally 필수) | 자동 (try-with-resources) |
| 부모-자식 관계 | 불명확 | 구조화됨 |
| ThreadLocal 누수 | 위험 | 안전 |
| 부분 실패 처리 | 수동 (예외 처리) | 정책으로 선택 |
| 코드 복잡도 | 중간 | 낮음 |

---

## ⚖️ 트레이드오프

### StructuredTaskScope의 장점
- **자동 정리**: scope 종료 시 모든 리소스 자동 해제
- **명확한 관계**: 부모-자식 task 관계 구조화
- **예외 정책**: ShutdownOnFailure/Success로 명시적 선택
- **ThreadLocal 안전**: 스코프 내 독립적 관리

### StructuredTaskScope의 제약
- **제한된 유연성**: 스코프 밖으로 나가면 task 관리 불가
- **Preview API**: Java 21에서도 아직 preview (안정화 진행 중)
- **학습곡선**: 기존 CompletableFuture와 다른 패러다임

---

## 📌 핵심 정리

1. **StructuredTaskScope**: 부모-자식 task 관계를 구조화하여 생명주기 관리
2. **ShutdownOnFailure**: 모든 task가 성공해야 함 (AND)
3. **ShutdownOnSuccess**: 하나만 성공하면 나머지 취소 (OR)
4. **자동 정리**: scope 종료 시 완료되지 않은 task 모두 cancel
5. **ThreadLocal 안전**: VT마다 로컬 변수 사용으로 메모리 누수 방지

---

## 🤔 생각해볼 문제

**Q1.** ShutdownOnFailure에서 한 task가 실패하면 나머지 task는 취소되는데, 이미 진행 중인 작업은 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**취소되지만 즉시 중단되지 않는다. Cancellation 신호만 전파된다.**

메커니즘:

1. Task가 exception 발생
2. Scope이 이를 감지
3. 나머지 task에 cancel() 호출
4. cancel()은 interrupt flag 설정 (SIGINT 신호 같은 것)
5. 각 task가 interrupt flag를 확인 (보통 InterruptedException으로)
6. Task는 cleanup 로직을 실행하고 종료

만약 task가 interrupt flag를 무시하면:
```java
scope.fork(() -> {
    while (true) {
        // interrupt flag 무시
        doWork();
    }
    // cleanup이 실행되지 않음
});
```

이 경우:
- Task는 계속 실행 (cancel 무시)
- Scope은 대기 (join에서 blocking)
- 결국 timeout 또는 무한 대기

따라서 task는 반드시:
```java
scope.fork(() -> {
    try {
        while (!Thread.currentThread().isInterrupted()) {
            doWork();
        }
    } finally {
        cleanup();  // 항상 실행
    }
});
```

이렇게 작성해야 안전한 취소가 가능.

</details>

---

**Q2.** 중첩된 스코프에서 자식 스코프가 실패하면 부모 스코프도 실패하는가?

<details>
<summary>해설 보기</summary>

**그렇다. exception은 부모로 전파된다.**

```java
try (var parentScope = new StructuredTaskScope.ShutdownOnFailure()) {
    parentScope.fork(() -> {
        try (var childScope = new StructuredTaskScope.ShutdownOnFailure()) {
            childScope.fork(() -> { throw new RuntimeException("자식 실패"); });
            childScope.join();
            childScope.throwIfFailed();  // ← exception 발생
            return "never reached";
        }  // 자식 스코프 종료
    });
    
    parentScope.join();
    parentScope.throwIfFailed();  // ← 자식의 exception이 여기서 발생
}
```

결과: 부모 throwIfFailed()에서 RuntimeException 재발생

취소 흐름:
1. 자식 스코프의 exception
2. 자식 스코프가 close()
3. fork된 task에서 exception 재발생
4. 부모 스코프의 다른 task들 cancel
5. 부모 스코프도 실패

따라서 exception 처리는 부모에서만 하면 된다:
```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    scope.fork(() -> childOperation1());
    scope.fork(() -> childOperation2());
    scope.join();
} catch (ExecutionException e) {
    // 어느 자식에서든 발생한 exception 처리
}
```

</details>

---

**Q3.** StructuredTaskScope는 Virtual Thread에서만 사용해야 하는가?

<details>
<summary>해설 보기</summary>

**아니다. Platform Thread에서도 사용 가능하고 권장된다.**

사실 StructuredTaskScope의 이점은 task 관리와 리소스 정리 자체에 있으므로:

```java
// Virtual Thread에서
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    scope.fork(() -> task1());
    scope.fork(() -> task2());
    scope.join();
}

// Platform Thread에서 (ExecutorService 기반)
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    scope.fork(() -> task1());
    scope.fork(() -> task2());
    scope.join();
}
```

둘 다 동일하게 동작한다. 단, Virtual Thread에서는 처리량이 훨씬 크기 때문에:
- VT: 100만 개 task를 동시 관리 (가능)
- PT: 200개 task 정도만 관리 (스레드 풀 크기)

따라서 실질적으로는 VT와 함께 사용할 때 진가가 드러난다.

권장:
- VT + StructuredTaskScope: 메인 패턴 (대량 task 관리)
- PT + StructuredTaskScope: 부분적 사용 (작은 규모)
- PT + ExecutorService: legacy (새로운 코드는 권장 안 함)

</details>

---

<div align="center">

**[⬅️ 이전: Pinning 문제](./03-pinning-problem.md)** | **[홈으로 🏠](../README.md)** | **[다음: Thread Pool → Virtual Thread 전환 ➡️](./05-thread-pool-migration.md)**

</div>
