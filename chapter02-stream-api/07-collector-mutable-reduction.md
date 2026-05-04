# Collector 내부 — Mutable Reduction과 Combiner

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Collector`의 `supplier`/`accumulator`/`combiner`/`finisher`/`characteristics` 5대 컴포넌트는 각각 무엇인가?
- `reduce`(불변)와 `collect`(가변)의 본질적 차이는?
- Characteristics(CONCURRENT/UNORDERED/IDENTITY_FINISH)가 병렬 처리에 미치는 영향은?
- `Collectors.toList()` 내부 구현은 어떻게 되어 있는가?
- Custom Collector를 작성할 때 combiner를 정확히 어떻게 구현해야 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Stream의 최종 형태인 `collect()`는 `reduce()`보다 훨씬 효율적이다. 불변 reduction의 한계를 알아야 `collect()`를 올바르게 사용하고, 병렬 Stream에서 combiner의 역할을 이해해야 올바른 Custom Collector를 작성할 수 있다. 또한 `CONCURRENT` 특성의 의미를 모르면 병렬 처리 때 데이터 손상을 초래할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: reduce() 남용 (불변 연산)
  List<String> list = ...;
  String result = list.stream()
      .reduce("", (acc, str) -> acc + str);  // O(N²) ← 재앙!
  
  → 각 단계마다 새로운 String 객체 생성
  → 최악: "a" → "ab" → "abc" → "abcd" → ...
  → 시간: O(N²), 메모리: O(N²)

실수 2: Combiner 없이 병렬 Stream 사용
  class CustomCollector<T, A, R> implements Collector {
      @Override
      public BiConsumer<A, A> combiner() {
          return null;  // 또는 throw new UnsupportedOperationException()
      }
  }
  
  customCollector
      .parallelStream()
      .collect(customCollector);  // 데이터 손상!

실수 3: supplier() 부작용
  Collector<T, List<T>, List<T>> collector = Collector.of(
      () -> new ArrayList<>(),  // supplier
      // 만약 단순 변수 반환:
      () -> singleton,          // ← 공유 상태! 위험
      ...
  );
  
  병렬 처리: 모든 스레드가 같은 List 수정
  결과: 동시성 문제 발생!

실수 4: Characteristics를 무시
  class CustomCollector implements Collector {
      @Override
      public Set<Characteristics> characteristics() {
          return Collections.emptySet();  // ← 모든 플래그 없음
      }
  }
  
  → 병렬화 불가능하거나 성능 저하
  → 불필요한 동기화
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// Collector 올바른 이해 및 사용

// 패턴 1: 기본 collect (가변 reduction)
List<String> result = stream.collect(
    Collectors.toList()  // ArrayList 생성, add로 축적
);

// 패턴 2: 가변 vs 불변 comparison
// ❌ 불변 (reduce): O(N²)
String concat = stream.reduce("", (a, b) -> a + b);

// ✓ 가변 (collect): O(N)
String concat = stream.collect(
    StringBuilder::new,    // supplier
    (sb, s) -> sb.append(s),  // accumulator
    (sb1, sb2) -> sb1.append(sb2.toString())  // combiner
).toString();

// 패턴 3: Custom Collector 올바른 구현
Collector<Integer, ?, Long> sumCollector = Collector.of(
    () -> new long[1],        // supplier: 누적자 생성
    (arr, val) -> arr[0] += val,  // accumulator: 값 누적
    (arr1, arr2) -> {         // combiner: 병렬 결과 병합
        arr1[0] += arr2[0];
        return arr1;
    },
    arr -> arr[0],            // finisher: 최종 결과
    Collector.Characteristics.UNORDERED  // characteristics
);

long sum = stream.collect(sumCollector);

// 패턴 4: Characteristics 정확한 사용
Set<Characteristics> characteristics = EnumSet.of(
    Characteristics.CONCURRENT,    // 스레드 안전
    Characteristics.UNORDERED,     // 순서 무관
    Characteristics.IDENTITY_FINISH  // finisher = identity
);

// 패턴 5: Collectors 유틸리티 활용
// toList
List<String> list = stream.collect(Collectors.toList());

// toMap
Map<String, Integer> map = stream.collect(
    Collectors.toMap(Person::getName, Person::getAge)
);

// groupingBy (stateful)
Map<String, List<Person>> groups = stream.collect(
    Collectors.groupingBy(Person::getDepartment)
);

// joining (문자열)
String joined = stream.collect(
    Collectors.joining(", ")
);

// customMap
Map<String, Long> counts = stream.collect(
    Collectors.groupingBy(
        Person::getDepartment,
        Collectors.counting()
    )
);
```

---

## 🔬 내부 동작 원리

### 1. Collector 인터페이스와 5대 컴포넌트

```java
// OpenJDK: java.util.stream.Collector

public interface Collector<T, A, R> {
    
    // 1. Supplier: 누적 용기 생성
    Supplier<A> supplier();
    // 반환: 빈 누적자 (예: new ArrayList())
    // 호출: 각 병렬 청크마다 1회
    // 목적: 초기 상태 제공

    // 2. Accumulator: 원소를 누적자에 추가
    BiConsumer<A, T> accumulator();
    // 파라미터: (누적자, 새로운 원소)
    // 동작: 누적자를 수정하고 원소 추가
    // 호출: 각 원소마다 1회
    // 특성: 가변 (누적자 상태 변경)

    // 3. Combiner: 병렬 결과 병합
    BinaryOperator<A> combiner();
    // 파라미터: (누적자1, 누적자2)
    // 반환: 병합된 누적자
    // 호출: 병렬 처리 시 청크 병합
    // 특성: 한 누적자를 수정, 다른 하나를 병합

    // 4. Finisher: 누적자를 최종 결과로 변환
    Function<A, R> finisher();
    // 파라미터: 누적자
    // 반환: 최종 결과
    // 예: ArrayList → unmodifiable List
    // 특수: IDENTITY_FINISH면 누적자 그대로

    // 5. Characteristics: 최적화 힌트
    Set<Characteristics> characteristics();
    // 반환: 비트 플래그 조합
    // 옵션: CONCURRENT, UNORDERED, IDENTITY_FINISH

    enum Characteristics {
        // CONCURRENT: 여러 스레드가 동시에 accumulator 호출 가능
        // (combiner 없이 동시 수정 가능, 예: ConcurrentHashMap)
        
        // UNORDERED: 순서 무관 (병렬화 최적화)
        // (예: Set, 순서 보장 불필요)
        
        // IDENTITY_FINISH: finisher = Function.identity()
        // (누적자가 바로 최종 결과, 변환 불필요)
    }
}

구체적인 예: Collectors.toList()

public static <T> Collector<T, ?, List<T>> toList() {
    return Collector.of(
        // supplier: 새 ArrayList 생성
        ArrayList::new,
        
        // accumulator: List.add() 호출
        List::add,
        
        // combiner: 두 리스트 병합
        (list1, list2) -> {
            list1.addAll(list2);
            return list1;
        },
        
        // finisher: List 그대로 반환 (변환 불필요)
        list -> Collections.unmodifiableList(list),
        
        // characteristics: 특수 없음
        // (순서 보장, 단일 스레드 accumulator)
    );
}
```

### 2. Reduce vs Collect (불변 vs 가변)

```java
// === reduce: 불변 reduction (함수형) ===
Integer sum1 = stream.reduce(
    0,  // identity
    (a, b) -> a + b,  // accumulator: 새 값 생성
    (a, b) -> a + b   // combiner
);

// 동작:
// 0 + 1 = 1 (새 Integer 객체)
// 1 + 2 = 3 (새 Integer 객체)
// 3 + 3 = 6 (새 Integer 객체)
// ...
// 시간: O(N), 메모리: O(N) (각 중간 값)

// === collect: 가변 reduction (명령형) ===
Integer sum2 = stream.collect(
    () -> new int[1],  // supplier: 누적자
    (arr, val) -> arr[0] += val,  // accumulator: 상태 수정
    (arr1, arr2) -> arr1[0] += arr2[0]  // combiner
).getValue();

// 동작:
// arr[0] = 0
// arr[0] += 1 → arr[0] = 1 (누적자 수정)
// arr[0] += 2 → arr[0] = 3 (누적자 수정)
// arr[0] += 3 → arr[0] = 6 (누적자 수정)
// ...
// 시간: O(N), 메모리: O(1) (누적자만)

성능 비교:

불변 reduction (String 연결):
  reduce("", (a, b) -> a + b)
  "" + "a" = "a"
  "a" + "b" = "ab"
  "ab" + "c" = "abc"
  ...
  시간: O(N²) ← String 복사!

가변 reduction (StringBuilder):
  collect(StringBuilder::new, (sb, s) -> sb.append(s), ...)
  sb.append("a")
  sb.append("b")
  sb.append("c")
  ...
  시간: O(N) ← 추가만!
```

### 3. 병렬 Stream에서 Combiner의 역할

```
병렬 Collector 실행 흐름:

Stream: [1, 2, 3, 4, 5, 6, 7, 8]

Step 1: 분할
  [1, 2] | [3, 4] | [5, 6] | [7, 8]

Step 2: 각 청크에서 supplier + accumulator
  Thread 0:
    acc0 = supplier()          // []
    accumulate(acc0, 1)        // [1]
    accumulate(acc0, 2)        // [1, 2]
  
  Thread 1:
    acc1 = supplier()          // []
    accumulate(acc1, 3)        // [3]
    accumulate(acc1, 4)        // [3, 4]
  
  Thread 2:
    acc2 = supplier()          // []
    accumulate(acc2, 5)        // [5]
    accumulate(acc2, 6)        // [5, 6]
  
  Thread 3:
    acc3 = supplier()          // []
    accumulate(acc3, 7)        // [7]
    accumulate(acc3, 8)        // [7, 8]

Step 3: 병합 (combiner 호출)
  combine(acc0, acc1)
    acc0.addAll(acc1)          // [1, 2, 3, 4]
  
  combine(acc2, acc3)
    acc2.addAll(acc3)          // [5, 6, 7, 8]
  
  combine([1,2,3,4], [5,6,7,8])
    result = [1, 2, 3, 4, 5, 6, 7, 8]

Step 4: Finisher (선택적)
  finisher(result) → 최종 형태

Combiner의 중요성:

CONCURRENT 없음 (일반적):
  accumulator: 단일 스레드만 호출
  combiner: 여러 스레드의 결과 병합
  
  병렬: [1,2] → acc0 (단일), [3,4] → acc1 (단일) → combine

CONCURRENT 있음:
  accumulator: 여러 스레드가 동시 호출
  combiner: 호출 안 될 수도 있음
  
  예: ConcurrentHashMap은 내부적으로 thread-safe하므로
      모든 스레드가 하나의 누적자에 직접 추가 가능
```

### 4. Characteristics 세부 해석

```java
enum Characteristics {
    // 1. CONCURRENT: 동시 accumulation 안전
    // 의미: accumulator를 여러 스레드가 동시 호출 가능
    // 조건: 누적자가 thread-safe해야 함
    // 예: ConcurrentHashMap (내부 동기화)
    //     AtomicReference.accumulateAndGet()
    // 효과: 병렬화에서 combiner 호출 최소화
    
    // 2. UNORDERED: 원소 순서 무관
    // 의미: 결과가 입력 순서와 무관
    // 예: Set (순서 없음)
    //     groupingBy().values() (그룹핑만 중요)
    // 효과: 병렬 청크 순서 자유
    
    // 3. IDENTITY_FINISH: finisher = identity 함수
    // 의미: accumulator = final result
    // 예: collect(toList()) (ArrayList가 결과)
    // 효과: finisher 호출 생략 (성능 향상)
}

최적화 테이블:

Characteristics         | 효과                    | 예시
───────────────────────┼──────────────────────┼──────────
없음                   | 순차만 지원             | toList()
UNORDERED              | 병렬화 추가             | toSet()
CONCURRENT             | 동시 accumulation      | toMap(병렬)
IDENTITY_FINISH        | finisher 생략           | collecting()
CONCURRENT|IDENTITY    | 최고 최적화             | custom Map
```

---

## 💻 실전 실험

### 실험 1: reduce vs collect 성능

```java
import java.util.*;
import java.util.stream.*;

public class ReduceVsCollectTest {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();
        for (int i = 0; i < 100_000; i++) {
            list.add("item" + i + " ");
        }
        
        System.out.println("=== reduce (불변) ===");
        long start = System.nanoTime();
        String result1 = list.stream()
            .reduce("", (a, b) -> a + b);  // O(N²)
        long elapsed1 = System.nanoTime() - start;
        
        System.out.println("\n=== collect (가변) ===");
        start = System.nanoTime();
        String result2 = list.stream()
            .collect(
                StringBuilder::new,
                (sb, s) -> sb.append(s),
                (sb1, sb2) -> sb1.append(sb2)
            ).toString();
        long elapsed2 = System.nanoTime() - start;
        
        System.out.printf("reduce: %.2f ms%n", elapsed1 / 1_000_000.0);
        System.out.printf("collect: %.2f ms%n", elapsed2 / 1_000_000.0);
        System.out.printf("속도 향상: %.1fx%n", (double) elapsed1 / elapsed2);
        
        // 결과: collect가 100배 이상 빠름!
    }
}
```

### 실험 2: Custom Collector 구현

```java
import java.util.*;
import java.util.stream.*;

public class CustomCollectorTest {
    static class Statistics {
        long sum = 0;
        long count = 0;
        double avg;
        
        void accept(int val) {
            sum += val;
            count++;
        }
        
        void combine(Statistics other) {
            sum += other.sum;
            count += other.count;
        }
        
        void finish() {
            avg = count == 0 ? 0 : (double) sum / count;
        }
    }
    
    public static void main(String[] args) {
        List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);
        
        Statistics stats = list.stream().collect(
            Collector.of(
                Statistics::new,            // supplier
                Statistics::accept,         // accumulator
                Statistics::combine,        // combiner
                s -> { s.finish(); return s; },  // finisher
                Collector.Characteristics.UNORDERED
            )
        );
        
        System.out.printf("Sum: %d, Count: %d, Avg: %.2f%n", 
            stats.sum, stats.count, stats.avg);
    }
}
```

### 실험 3: Characteristics의 영향

```java
import java.util.*;
import java.util.stream.*;
import java.util.concurrent.atomic.*;

public class CharacteristicsTest {
    public static void main(String[] args) {
        List<Integer> list = new ArrayList<>();
        for (int i = 0; i < 1_000_000; i++) {
            list.add(i);
        }
        
        System.out.println("=== collect(toList) 순차 ===");
        long start = System.nanoTime();
        List<Integer> result1 = list.stream()
            .collect(Collectors.toList());
        long elapsed1 = System.nanoTime() - start;
        
        System.out.println("\n=== collect(toList) 병렬 ===");
        start = System.nanoTime();
        List<Integer> result2 = list.parallelStream()
            .collect(Collectors.toList());
        long elapsed2 = System.nanoTime() - start;
        
        System.out.printf("Sequential: %.2f ms%n", elapsed1 / 1_000_000.0);
        System.out.printf("Parallel: %.2f ms%n", elapsed2 / 1_000_000.0);
        
        // IDENTITY_FINISH 없으면 finisher 오버헤드
        // 병렬에서 combiner 비용 + finisher 비용
    }
}
```

---

## 📊 성능/비교

```
Collector 성능 특성:

연산           | 메모리 | 시간  | 병렬화 | 특성
──────────────┼────────┼──────┼────────┼──────────
reduce         | O(N)   | O(N) | 가능   | 불변
collect        | O(result)| O(N)| 가능   | 가변
toList         | O(N)   | O(N) | 우수   | UNORDERED
toSet          | O(N)   | O(N) | 우수   | UNORDERED
toMap          | O(N)   | O(N) | 제한   | 기타
groupingBy     | O(N)   | O(N) | 우수   | stateful

String 연결 성능 비교:

reduce: 10,000개 문자열
  시간: ~5초 (O(N²))
  메모리: GC 부하 높음

StringBuilder.collect:
  시간: ~50ms (O(N))
  메모리: GC 부하 낮음

개선: 100배 빠름!

병렬 처리 성능 (4코어, 1M 요소):

toList 순차:       ~100ms
toList 병렬:       ~30ms (3.3배)

HashMap reduce:
  순차:            ~50ms
  병렬:            ~80ms (combiner 오버헤드 > 이득)
  
collect(toMap):
  순차:            ~50ms
  병렬:            ~40ms (약간의 이득)
```

---

## ⚖️ 트레이드오프

```
Collector 설계 트레이드오프:

가변 Reduction의 장점:
  ✓ 메모리 효율 (누적자만)
  ✓ 성능 우수 (상태 수정)
  ✓ 병렬화 용이 (combiner)
  ✓ 대용량 처리 적합

불변 Reduction의 장점:
  ✓ 함수형 스타일
  ✓ 부작용 없음
  ✓ 스레드 안전성 자동

CONCURRENT 특성:
  장점: 병렬화 오버헤드 감소 (combiner 최소화)
  단점: 누적자가 thread-safe해야 함
  
UNORDERED 특성:
  장점: 병렬 청크 순서 자유 (성능 향상)
  단점: 결과 순서 보장 안 함

IDENTITY_FINISH 특성:
  장점: finisher 생략 (성능 향상)
  단점: 누적자 = 최종 결과여야 함

선택 기준:
  단순 수집: toList(), toSet() (표준)
  복잡한 집계: Custom Collector
  병렬화 필요: CONCURRENT/UNORDERED 고려
  순서 중요: UNORDERED 제외
```

---

## 📌 핵심 정리

```
Collector 핵심:

1. 5대 컴포넌트:
   - supplier: 누적자 생성
   - accumulator: 원소 추가
   - combiner: 병렬 결과 병합
   - finisher: 최종 결과 변환
   - characteristics: 최적화 힌트

2. reduce vs collect:
   - reduce: 불변, O(N) 메모리 낭비
   - collect: 가변, O(1~N) 메모리 효율
   
3. 병렬 처리:
   - supplier: 청크마다 1회
   - accumulator: 각 원소마다
   - combiner: 청크 병합
   - characteristics: 최적화 결정

4. Characteristics:
   - CONCURRENT: 동시 accumulation 안전
   - UNORDERED: 순서 무관
   - IDENTITY_FINISH: finisher 생략

5. 성능:
   - 가변 > 불변 (String 100배)
   - 병렬 도움이 되려면 오버헤드 < 이득
   - 작은 데이터: 순차가 더 빠를 수 있음

최적 Collector 구현:
  1. 가변 누적자 사용
  2. combiner 명확하게
  3. 적절한 characteristics 선언
  4. finisher 최소화 (IDENTITY_FINISH)
```

---

## 🤔 생각해볼 문제

**Q1.** `collect(toList())`를 병렬 스트림에서 사용했을 때, 여러 스레드가 같은 ArrayList에 동시에 add()하면 동시성 문제가 발생하지 않을까?

<details>
<summary>해설 보기</summary>

발생하지 않는다. 왜냐하면:

```
병렬 toList() 동작:

Step 1: supplier
  Thread 0: list0 = new ArrayList<>()
  Thread 1: list1 = new ArrayList<>()
  Thread 2: list2 = new ArrayList<>()
  (각 스레드가 독립적인 ArrayList 생성!)

Step 2: accumulator
  Thread 0: list0.add(1), list0.add(2)  ← list0에만 추가
  Thread 1: list1.add(3), list1.add(4)  ← list1에만 추가
  Thread 2: list2.add(5), list2.add(6)  ← list2에만 추가
  (각 스레드가 자신의 list만 수정)

Step 3: combiner
  main: list0.addAll(list1)  ← 순차 진행
  main: result.addAll(list2)
  (main 스레드가 병합, 동시성 없음)
```

핵심: Characteristics에 `CONCURRENT`가 없으면:
- accumulator: 단일 스레드만 호출
- combiner: 다른 스레드의 결과를 병렬로 받아 병합
- 동시성 문제 없음

만약 `CONCURRENT` 있다면:
- 여러 스레드가 하나의 누적자에 동시 추가
- ArrayList는 CONCURRENT가 없으므로 위험
- ConcurrentHashMap 같은 thread-safe 누적자 필요

</details>

---

**Q2.** `combiner`에서 첫 번째 파라미터 누적자를 수정하는 이유는?

<details>
<summary>해설 보기</summary>

**메모리 효율성:**

```java
// ❌ 나쁜 구현 (새 객체 생성)
@Override
public BinaryOperator<List<T>> combiner() {
    return (list1, list2) -> {
        List<T> combined = new ArrayList<>();  // ← 새 객체 생성
        combined.addAll(list1);
        combined.addAll(list2);
        return combined;
    };
}

병렬 병합: [A], [B], [C], [D]
  combine([A], [B])      → [AB] (새 객체)
  combine([C], [D])      → [CD] (새 객체)
  combine([AB], [CD])    → [ABCD] (새 객체)
  메모리: 원본 + 중간 3개 = 4배

// ✓ 좋은 구현 (첫 번째 수정)
@Override
public BinaryOperator<List<T>> combiner() {
    return (list1, list2) -> {
        list1.addAll(list2);  // ← list1 제자리 수정
        return list1;
    };
}

병렬 병합: [A], [B], [C], [D]
  combine([A], [B])      → [AB] (list1 수정)
  combine([C], [D])      → [CD] (list1 수정)
  combine([AB], [CD])    → [ABCD] (list1 수정)
  메모리: 원본 + 스택 = 2배
```

**규칙:**
```
Combiner = BinaryOperator<A> combiner()
  → 두 누적자를 받아 하나를 반환
  → 보통 list1.addAll(list2); return list1;
  → 두 번째 누적자는 이후 사용 안 됨 (버려도 됨)
```

이것이 "mutable reduction"의 핵심: 객체 재사용으로 메모리 절감.

</details>

---

**Q3.** `IDENTITY_FINISH` 특성이 있으면 finisher를 호출하지 않는다는데, 그럼 finisher는 왜 있는가?

<details>
<summary>해설 보기</summary>

**두 가지 경우:**

1. **IDENTITY_FINISH 있음**
```java
Collector.of(
    ArrayList::new,           // supplier
    List::add,                // accumulator
    (l1, l2) -> { l1.addAll(l2); return l1; },  // combiner
    Function.identity(),      // finisher = identity
    Collector.Characteristics.IDENTITY_FINISH
)

동작:
  누적자 = ArrayList
  finisher(ArrayList) = ArrayList (변환 없음)
  
최적화: Stream 엔진이 finisher 호출 생략
  result = accumulator;  // finisher 호출 안 함
```

2. **IDENTITY_FINISH 없음**
```java
Collector.of(
    ArrayList::new,
    List::add,
    (l1, l2) -> { l1.addAll(l2); return l1; },
    Collections::unmodifiableList  // finisher = 변환
    // IDENTITY_FINISH 없음
)

동작:
  누적자 = ArrayList
  finisher(ArrayList) = unmodifiableList (변환!)
  
최적화: 없음, finisher 호출 필수
  result = finisher(accumulator);
```

**즉, finisher는:**
- 누적자의 타입과 최종 결과의 타입이 다를 때 필요
- 예: ArrayList → unmodifiableList
- IDENTITY_FINISH면 변환 없이 누적자 = 최종 결과

성능 차이:
- IDENTITY_FINISH 있음: finisher 호출 생략 → 빠름
- 없음: finisher 호출 필수 → 느림 (특히 병렬에서)

</details>

---

<div align="center">

**[⬅️ 이전: Stateless vs Stateful 연산](./06-stateless-vs-stateful.md)** | **[홈으로 🏠](../README.md)** | **[다음: Custom Collector 작성 ➡️](./08-custom-collector.md)**

</div>
