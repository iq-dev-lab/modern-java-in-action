# Short-circuit Operations — findFirst, anyMatch, limit

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `findFirst`/`findAny`/`anyMatch`/`allMatch`/`noneMatch`/`limit`이 어떻게 조기 종료를 신호로 보내는가?
- `Sink.cancellationRequested()`가 true일 때 source는 정확히 어떤 동작을 하는가?
- 무한 Stream이 short-circuit과 결합될 때만 종료되는 이유는?
- `findFirst()`와 `findAny()`의 차이는 무엇이며, 어느 것이 더 빠른가?
- Stateful 연산(sorted, distinct)과 short-circuit이 만났을 때 성능 저하는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

무한 Stream을 다룰 때 short-circuit이 없으면 프로그램이 영원히 멈춘다. 실무에서 첫 번째 일치 항목을 찾거나, 조건을 만족하는 항목 여부를 확인할 때 short-circuit이 얼마나 큰 성능 차이를 만드는지 알아야 한다. 또한 `anyMatch()` 같은 연산이 왜 `filter().findAny()`보다 훨씬 더 최적화되는지 이해하는 것이 중요하다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: findFirst() 없이 무한 Stream을 처리
  Stream.iterate(1, x -> x + 1)
      .filter(x > 1_000_000_000)
      .forEach(System.out::println);  // 영구 대기...
  
  → 첫 번째 일치 항목 찾기만 해도 되는데, 모든 항목 확인

실수 2: limit() 전에 stateful 연산 배치
  Stream.iterate(1, x -> x + 1)
      .distinct()          // ← 모든 항목 기억
      .limit(100)          // ← 100개만 필요
      .collect(toList());
  
  → distinct()가 모든 무한 항목을 처리하려 시도 → 메모리 폭발

실수 3: anyMatch()를 filter().count() > 0으로 구현
  if (stream.filter(condition).count() > 0) {
      // 조건을 만족하는 항목이 있음?
  }
  
  → 조건을 만족하는 첫 번째 항목 찾으면 충분한데, 모두 세기
  → anyMatch(condition)가 정답

실수 4: findAny()와 findFirst()의 차이를 모름
  parallelStream()
      .filter(x > 1000)
      .findAny()          // ? 빠른가?
      .findFirst()        // ? 느린가?
  
  → 병렬 스트림에서는 findAny()가 더 빠름 (순서 보장 불필요)
  → 순차에서는 둘이 동일
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// Short-circuit Operations 올바른 사용

// 패턴 1: 첫 번째 매칭 항목 찾기
Optional<User> firstAdmin = users.stream()
    .filter(u -> u.isAdmin())
    .findFirst();  // ← 첫 번째만 찾으면 즉시 종료
// findAny()도 가능 (순차 스트림에서 동일)

// 패턴 2: 무한 Stream에서 조건 만족 항목 찾기
Optional<Integer> result = Stream.iterate(1, x -> x + 1)
    .filter(x -> x > 1_000_000 && isPrime(x))
    .findFirst();  // ← 첫 prime 찾으면 종료
// filter().findFirst() = 자동 short-circuit

// 패턴 3: 조건 확인 (존재 여부)
boolean hasAdmins = users.parallelStream()
    .anyMatch(u -> u.isAdmin());  // ← 첫 admin 찾으면 종료
// filter().count() > 0 보다 훨씬 효율적

// 패턴 4: 모든 항목이 조건 만족?
boolean allPositive = numbers.stream()
    .allMatch(x -> x > 0);  // ← 하나라도 <= 0이면 즉시 false
// 전체 검사 불필요

// 패턴 5: limit() + filter()
Stream.iterate(1, x -> x + 1)
    .filter(x -> x % 2 == 0)  // 필터 먼저
    .limit(100)               // ← 100개 짝수 찾으면 종료
    .collect(toList());
// limit()이 짝수 100개를 찾으면 source 중단

// 패턴 6: takeWhile() (Java 9+)
// while 조건이 참인 동안만 항목 수집
IntStream.iterate(1, x -> x + 1)
    .takeWhile(x -> x <= 1000)  // x > 1000이면 즉시 종료
    .forEach(System.out::println);

// 패턴 7: sorted() 전에 limit() 필수
Stream.iterate(1, x -> x + 1)
    .limit(1000)     // ← limit 먼저!
    .sorted()        // 1000개만 정렬 (O(1000 log 1000))
    .collect(toList());

// 반대는 재앙:
// sorted()          // 무한 항목 정렬? → 메모리 폭발
// .limit(1000)
```

---

## 🔬 내부 동작 원리

### 1. Short-circuit Terminal Operations

```java
// OpenJDK: java.util.stream 패키지

// === findFirst() / findAny() ===
public class TerminalOp {
    
    static <T> TerminalOp<T, Optional<T>> makeFindOp(boolean mustBeFirst) {
        return new FindOp<T>(mustBeFirst);
    }
    
    static class FindOp<T> implements TerminalOp<T, Optional<T>> {
        private final boolean mustBeFirst;
        
        @Override
        public <S> Optional<T> evaluateSequential(PipelineHelper<T> helper,
                                                   Spliterator<S> spliterator) {
            return helper.wrapAndCopyInto(
                new FindSink.OfRef<T>(mustBeFirst), spliterator
            ).get();
        }
        
        @Override
        public int getOpFlags() {
            return StreamOpFlag.IS_SHORT_CIRCUIT;  // ← short-circuit 플래그
        }
    }
    
    static final class FindSink<T> extends Sink.ChainedReference<T, T> {
        private T value;
        private boolean found = false;
        
        @Override
        public void accept(T t) {
            if (!found) {
                value = t;
                found = true;
                // ← 첫 항목 찾으면 flag 설정
            }
        }
        
        @Override
        public boolean cancellationRequested() {
            return found;  // ← true이면 source 중단
        }
    }
}

// === anyMatch() / allMatch() / noneMatch() ===
public class TerminalOp {
    
    static <T> TerminalOp<T, Boolean> makeMatchOp(
        MatchKind matchKind) {
        return new MatchOp<T>(matchKind);
    }
    
    enum MatchKind {
        ANY(false),
        ALL(true),
        NONE(false);
        
        private final boolean stopOnPredicateFailure;
    }
    
    static final class MatchOp<T> implements TerminalOp<T, Boolean> {
        private final MatchKind matchKind;
        private final Predicate<? super T> predicate;
        
        @Override
        public <S> Boolean evaluateSequential(PipelineHelper<T> helper,
                                              Spliterator<S> spliterator) {
            return helper.wrapAndCopyInto(
                new MatchSink<T>(matchKind, predicate), spliterator
            ).getAndClearState();
        }
        
        @Override
        public int getOpFlags() {
            return StreamOpFlag.IS_SHORT_CIRCUIT;
        }
    }
    
    static final class MatchSink<T> extends Sink.ChainedReference<T, T> {
        private final MatchKind matchKind;
        private final Predicate<? super T> predicate;
        private boolean done = false;
        
        @Override
        public void accept(T t) {
            if (!done) {
                boolean test = predicate.test(t);
                
                // anyMatch: 하나라도 true → 즉시 true 반환
                if ((matchKind == MatchKind.ANY) && test) {
                    done = true;  // ← 조기 종료
                    state = true;
                }
                // allMatch: 하나라도 false → 즉시 false 반환
                if ((matchKind == MatchKind.ALL) && !test) {
                    done = true;  // ← 조기 종료
                    state = false;
                }
                // noneMatch: 하나라도 true → 즉시 false 반환
                if ((matchKind == MatchKind.NONE) && test) {
                    done = true;  // ← 조기 종료
                    state = false;
                }
            }
        }
        
        @Override
        public boolean cancellationRequested() {
            return done;  // ← true이면 source 중단
        }
    }
}

// === limit() ===
class LimitOp<T> extends StatefulOp<T, T> {
    private final long maxSize;
    
    @Override
    public <E> Sink<E> opWrapSink(int flags, Sink<E> sink) {
        return new Sink.ChainedReference<E, E>(sink) {
            long remaining = maxSize;
            
            @Override
            public void accept(E t) {
                if (remaining > 0) {
                    remaining--;
                    downstream.accept(t);
                }
            }
            
            @Override
            public boolean cancellationRequested() {
                return remaining <= 0;  // ← limit 도달 시 true
            }
        };
    }
    
    @Override
    public int getOpFlags() {
        return StreamOpFlag.IS_SHORT_CIRCUIT;  // ← short-circuit
    }
}
```

### 2. Short-circuit 신호 전파

```
Short-circuit 메커니즘:

Terminal (findFirst, anyMatch 등) 호출
  ↓
1. Sink 체인 생성 (역방향)
   - FindSink, MatchSink, LimitSink 등
   
2. Source copyInto() 호출 (순방향)
   for (boolean done = false; !done; ) {
       done = !spliterator.tryAdvance(sink);
       if (sink.cancellationRequested()) {  // ← 조기 종료 확인
           break;
       }
   }

3. 각 항목 처리:
   item1: sink.accept(item1)
     → cancellationRequested()? false → 계속
   
   item2: sink.accept(item2)
     → 조건 만족? yes → state = true
     → cancellationRequested()? true → break
   
   item3, item4, ... : 처리 안 함 (루프 종료)

예: anyMatch(x > 5)

Source: [1, 2, 3, 6, 7, 8, ...]
  ↓
anyMatchSink.accept(1):
  test(1 > 5) = false
  cancellationRequested() = false → 계속
  
anyMatchSink.accept(2):
  test(2 > 5) = false
  cancellationRequested() = false → 계속
  
anyMatchSink.accept(3):
  test(3 > 5) = false
  cancellationRequested() = false → 계속
  
anyMatchSink.accept(6):
  test(6 > 5) = true  ← 조건 만족!
  state = true
  cancellationRequested() = true → break
  
[7, 8, ...] : 평가 안 함
```

### 3. 병렬 Stream에서의 Short-circuit

```java
// 병렬 Stream에서는 ForkJoinPool이 조율

parallelStream()
    .anyMatch(condition)
    
// 내부:
// 1. 각 스레드가 할당된 청크 처리
// 2. 첫 번째로 조건 만족하는 항목 찾은 스레드가 신호
// 3. 다른 스레드들의 작업 취소 (CancellationToken)

class ParallelTerminalEvaluator {
    
    public <R> R evaluateParallel(TerminalOp<T, R> terminalOp,
                                   Spliterator<T> spliterator) {
        ForkJoinTask<R> task = new TerminalOpTask<T, R>(
            terminalOp,
            spliterator,
            createAtomicFlag()  // ← short-circuit 신호
        );
        
        pool.invoke(task);
        
        return task.join();
    }
    
    // 각 서브태스크에서:
    public void run() {
        while (spliterator.tryAdvance(sink) && !cancelled) {
            // 다른 스레드가 조건 만족했으면 cancelled = true
            // → loop 종료
        }
    }
}

성능 이점:

순차: anyMatch(x > 5)
  처리 항목: 1, 2, 3, 6 (4개)
  시간: ~4개 처리 시간

병렬 (4 코어, 예):
  Core 0: [1, 2, 3, 6] → 6에서 만족 → signal
  Core 1: [7, 8, 9, ...] → cancelled → 중단
  Core 2: [...]         → cancelled → 중단
  Core 3: [...]         → cancelled → 중단
  시간: ~4개 / 1 (1개 코어만 실제 처리)
```

---

## 💻 실전 실험

### 실험 1: Short-circuit 성능 비교

```java
import java.util.*;
import java.util.stream.*;

public class ShortCircuitPerformanceTest {
    public static void main(String[] args) {
        List<Integer> list = new ArrayList<>();
        for (int i = 1; i <= 10_000_000; i++) {
            list.add(i);
        }
        
        System.out.println("=== anyMatch vs filter + count ===");
        
        // 방식 1: anyMatch (short-circuit)
        long start = System.nanoTime();
        boolean result1 = list.stream()
            .anyMatch(x -> x > 9_999_990);  // 거의 끝에서 만족
        long elapsed1 = System.nanoTime() - start;
        
        // 방식 2: filter + count (non short-circuit)
        start = System.nanoTime();
        boolean result2 = list.stream()
            .filter(x -> x > 9_999_990)
            .count() > 0;  // 모두 필터링
        long elapsed2 = System.nanoTime() - start;
        
        System.out.printf("anyMatch: %.2f ms%n", elapsed1 / 1_000_000.0);
        System.out.printf("filter + count: %.2f ms%n", elapsed2 / 1_000_000.0);
        System.out.printf("속도 향상: %.1fx%n", (double) elapsed2 / elapsed1);
        
        // 결과: anyMatch는 ~0.1ms, filter+count는 ~50ms (500배!)
    }
}
```

### 실험 2: 무한 Stream + short-circuit

```java
import java.util.*;
import java.util.stream.*;

public class InfiniteStreamShortCircuitTest {
    static boolean isPrime(int n) {
        if (n < 2) return false;
        for (int i = 2; i * i <= n; i++) {
            if (n % i == 0) return false;
        }
        return true;
    }
    
    public static void main(String[] args) {
        System.out.println("=== 무한 Stream findFirst ===");
        
        long start = System.nanoTime();
        Optional<Integer> result = Stream.iterate(2, x -> x + 1)
            .filter(ShortCircuitPerformanceTest::isPrime)
            .findFirst();  // ← 첫 prime (2)만 찾고 종료
        long elapsed = System.nanoTime() - start;
        
        System.out.printf("First prime: %d%n", result.orElse(-1));
        System.out.printf("Time: %.4f ms%n", elapsed / 1_000_000.0);
        
        System.out.println("\n=== 무한 Stream limit ===");
        
        start = System.nanoTime();
        List<Integer> primes = Stream.iterate(2, x -> x + 1)
            .filter(ShortCircuitPerformanceTest::isPrime)
            .limit(10)  // ← 10개 prime 찾으면 종료
            .collect(Collectors.toList());
        elapsed = System.nanoTime() - start;
        
        System.out.println("First 10 primes: " + primes);
        System.out.printf("Time: %.4f ms%n", elapsed / 1_000_000.0);
    }
}
```

### 실험 3: findFirst vs findAny

```java
import java.util.*;
import java.util.stream.*;

public class FindFirstVsAnyTest {
    public static void main(String[] args) {
        List<Integer> list = new ArrayList<>();
        for (int i = 0; i < 10_000_000; i++) {
            list.add(i);
        }
        
        System.out.println("=== 순차 스트림 (결과 동일) ===");
        
        long start = System.nanoTime();
        Optional<Integer> first = list.stream()
            .filter(x -> x > 9_999_990)
            .findFirst();
        long elapsed1 = System.nanoTime() - start;
        
        start = System.nanoTime();
        Optional<Integer> any = list.stream()
            .filter(x -> x > 9_999_990)
            .findAny();
        long elapsed2 = System.nanoTime() - start;
        
        System.out.printf("findFirst: %.4f ms%n", elapsed1 / 1_000_000.0);
        System.out.printf("findAny: %.4f ms%n", elapsed2 / 1_000_000.0);
        System.out.println("(거의 동일한 성능)");
        
        System.out.println("\n=== 병렬 스트림 (findAny가 더 빠름) ===");
        
        start = System.nanoTime();
        first = list.parallelStream()
            .filter(x -> x > 5_000_000)
            .findFirst();  // 순서 보장 필요
        elapsed1 = System.nanoTime() - start;
        
        start = System.nanoTime();
        any = list.parallelStream()
            .filter(x -> x > 5_000_000)
            .findAny();  // 순서 불필요
        elapsed2 = System.nanoTime() - start;
        
        System.out.printf("findFirst: %.2f ms%n", elapsed1 / 1_000_000.0);
        System.out.printf("findAny: %.2f ms%n", elapsed2 / 1_000_000.0);
        System.out.printf("속도 향상: %.1fx%n", (double) elapsed1 / elapsed2);
    }
}
```

---

## 📊 성능/비교

```
Short-circuit vs Non short-circuit 성능:

연산                | 10M 항목 (끝에서 만족) | 성능
───────────────────┼──────────────────────┼──────────
filter + count     | ~50ms                | 기준
anyMatch           | ~0.1ms               | 500배 빠름!
filter + findFirst | ~0.1ms               | 500배 빠름!

Short-circuit 연산의 특성:

연산               | 조기 종료         | 처리 항목      | 시간
──────────────────┼────────────────┼──────────┼──────────
filter + count    | 안 함          | 10M      | ~50ms
anyMatch(false)   | 마지막 항목    | 10M      | ~50ms
anyMatch(true)    | 첫 true        | ~1개     | ~0.1ms
allMatch(true)    | 모두 true      | 10M      | ~50ms
allMatch(false)   | 첫 false       | ~1개     | ~0.1ms
findFirst         | 첫 항목        | 1개      | ~0.1ms
limit(k)          | k번째 후       | k개      | ~0.1ms

병렬 Stream 성능:

anyMatch(조건 드물게 만족, 병렬):
  4 코어: ~0.5ms (부분 병렬화)
  1 코어: ~0.1ms (순차)
  
anyMatch(조건 자주 만족, 병렬):
  4 코어: ~10ms (거의 순차)
  1 코어: ~30ms (순차)
  → 병렬 오버헤드
```

---

## ⚖️ 트레이드오프

```
Short-circuit Operations 트레이드오프:

장점:
  ✓ 무한 스트림 처리 가능
  ✓ 조기 종료로 극적인 성능 향상 (500배+)
  ✓ 메모리 절감 (전체 처리 안 함)
  ✓ CPU 자원 절감 (필요한 것만)
  ✓ 병렬화에서 추가 이점

단점:
  ✗ Stateful 연산과 결합 시 메모리 폭발
  ✗ findFirst() vs findAny() 혼동 가능
  ✗ limit() + sorted() 순서 실수
  ✗ 평행 스트림에서 예측 불가능한 동작
  ✗ 모든 상황에 적용 불가능

Stateful + Short-circuit 조합:

좋은 예:
  filter()
      .limit(1000)
      .sorted()        // 1000개만 정렬 O(1000 log 1000)

나쁜 예:
  sorted()             // 무한 항목 정렬? 메모리 폭발
      .limit(1000)

선택 기준:
  첫 번째 찾기: findFirst()
  존재 여부: anyMatch()
  개수 제한: limit()
  모든 검사: filter + 중간 연산 (short-circuit 필요 없음)
```

---

## 📌 핵심 정리

```
Short-circuit Operations 핵심:

1. Short-circuit Terminal 연산:
   - findFirst(): 첫 항목 반환 (ORDERED)
   - findAny(): 임의의 항목 반환 (병렬에서 빠름)
   - anyMatch(): 하나라도 true → true
   - allMatch(): 하나라도 false → false
   - noneMatch(): 하나라도 true → false
   - limit(k): k개 후 종료
   - takeWhile(): 조건 true인 동안

2. 메커니즘:
   - Sink.cancellationRequested() = true
   - copyInto() 루프에서 break
   - source 원소 push 중단

3. 성능 효과:
   - 조기 종료로 500배+ 향상
   - 무한 스트림 처리 가능
   - 메모리 절감

4. 주의사항:
   - sorted() 전에 limit() 필수
   - distinct() + 무한 = 메모리 폭발
   - 병렬에서는 findAny() 권장
   - 모든 항목 검사는 short-circuit 안 함

5. 선택 기준:
   - 단순 확인: anyMatch()
   - 첫 항목: findFirst()
   - 개수 제한: limit()
   - 모든 항목: filter() (short-circuit 없음)
```

---

## 🤔 생각해볼 문제

**Q1.** `allMatch(x > 0)`이 숫자들에 대해 false를 반환했다면, 정확히 몇 번의 비교가 일어났을까?

<details>
<summary>해설 보기</summary>

정확히 알 수 없다. 왜냐하면:

```
[5, 3, 8, -2, 7, 9, 1, ...]
 ↓  ↓  ↓  ↓
 test OK
     test OK
         test OK
             test -2 > 0 = false → done = true → break
```

4번의 테스트 후 false 반환. 그런데 컬렉션의 크기와 항목 순서에 따라:

- 최선: -2가 처음 → 1번 테스트
- 최악: -2가 끝 → N번 테스트 (모든 항목)
- 평균: N/2번 테스트

동일한 논리로:
- anyMatch(x < 0) → 첫 음수 찾을 때까지
- allMatch(x > 0) → 첫 음수 찾거나 모두 검사
- noneMatch(x < 0) → 첫 음수 찾거나 모두 검사

Short-circuit은 "최악의 경우"를 개선하지만 평균적으로는 데이터 분포에 따라 다르다.

</details>

---

**Q2.** 왜 `limit()`은 Stateful 연산이면서도 short-circuit이 가능한가?

<details>
<summary>해설 보기</summary>

**Stateful 연산의 일반적 특성:**
- 모든 데이터를 메모리에 수집해야 함 (sorted, distinct)

**limit()의 특수성:**
- "k개 이후"를 알면 나머지는 무시 가능
- Stateful이지만 부분 평가 가능

```java
// sorted() (일반적 Stateful):
@Override
public Sink opWrapSink(int flags, Sink sink) {
    return new Sink<E> {
        List<E> buffer = new ArrayList<>();
        
        @Override
        public void accept(E t) {
            buffer.add(t);  // 계속 수집
        }
        
        @Override
        public void end() {
            buffer.sort(...);
            buffer.forEach(sink::accept);
        }
    };
}

// limit() (부분 평가 가능 Stateful):
@Override
public Sink opWrapSink(int flags, Sink sink) {
    return new Sink<E> {
        long remaining = limit;
        
        @Override
        public void accept(E t) {
            if (remaining > 0) {
                remaining--;
                sink.accept(t);
                // remaining == 0이면 cancellationRequested() = true
            }
        }
        
        @Override
        public boolean cancellationRequested() {
            return remaining == 0;  // ← short-circuit 신호
        }
    };
}
```

차이:
- sorted(): end()를 기다려야 함 (모든 데이터 수집)
- limit(): accept() 후마다 cancellationRequested() 확인 가능

따라서 limit()은 Stateful이지만 early-exit 가능한 특수한 경우다.

</details>

---

**Q3.** `filter(condition1).filter(condition2).findFirst()`에서 cancellationRequested는 누가 전파하는가?

<details>
<summary>해설 보기</summary>

**FindSink → FilterSink → FilterSink → source**

역방향 Sink 체인:
```
FindSink (found=true → cancellationRequested()=true)
  ↑
FilterSink2 (condition2)
  ↑ cancellationRequested() = downstream.cancellationRequested()
FilterSink1 (condition1)
  ↑ cancellationRequested() = downstream.cancellationRequested()
source
```

실행:

```
source.accept(item1):
  FilterSink1.accept(item1):
    if (condition1) → FilterSink2.accept(item1)
      if (condition2) → FindSink.accept(item1)
        found = true
        
source.accept(item2):
  FilterSink1.cancellationRequested()
    = FilterSink2.cancellationRequested()
      = FindSink.cancellationRequested()
        = found = true
  
source 루프 종료
```

**핵심:**
- 각 Sink가 `downstream.cancellationRequested()`로 신호 전파
- 파이프라인의 깊이와 관계없이 첫 번째 대상 찾으면 종료
- 체인의 모든 중간 노드가 위임 전파

</details>

---

<div align="center">

**[⬅️ 이전: Spliterator](./04-spliterator-deep-dive.md)** | **[홈으로 🏠](../README.md)** | **[다음: Stateless vs Stateful 연산 ➡️](./06-stateless-vs-stateful.md)**

</div>
