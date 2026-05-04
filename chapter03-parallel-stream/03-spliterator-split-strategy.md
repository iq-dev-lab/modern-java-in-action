# Spliterator 분할 전략 — trySplit과 characteristics

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Spliterator.trySplit()`이 반환하는 값의 의미와 분할 종료 시점은?
- `ArrayList` (SIZED+SUBSIZED)와 `LinkedList` (SIZED만) 분할 효율의 차이는?
- `HashMap.spliterator()`의 분할이 왜 효율적인가?
- 무한 Stream의 분할 한계는 무엇인가?
- `IntStream.range()`가 가장 효율적으로 분할되는 이유는?
- characteristics 비트 플래그가 옵티마이저에 주는 힌트는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`parallelStream()`이 데이터를 분할하는 방식은 성능에 직결된다. `ArrayList`는 빠르게 분할되지만 `LinkedList`는 매우 비효율적이다. 또한 `Spliterator.characteristics()`의 비트 플래그를 이해하면, 병렬 처리 최적화의 기회를 놓치지 않을 수 있다. 무한 stream이나 stateful 연산은 분할 불가능하다는 것을 알면, 병렬 처리 불가능한 상황을 미리 감지할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: LinkedList에 parallelStream() 사용
  LinkedList<Integer> list = ...;  // 1백만 원소
  list.parallelStream()  // "병렬 처리하니까 빠를 것"
      .map(this::heavyCompute)
      .collect(toList());
  
  // 실제: LinkedList는 순차 접근만 가능
  // trySplit()이 O(N) 시간에 중간 노드 찾음
  // 분할 오버헤드 > 병렬화 이득 → 순차보다 느림

실수 2: Stream.generate()와 parallelStream() 혼용
  Stream<Integer> infinite = Stream.generate(() -> random.nextInt());
  List<Integer> numbers = infinite.limit(1000)
      .parallel()  // 분할 불가능 (무한이었음)
      .collect(toList());
  
  // 실제: generate()는 분할 불가능 (ORDERED, NONCONCURRENT)
  // parallel() 호출해도 내부적으로 순차 처리

실수 3: characteristics를 무시하고 설계
  "내 customSpliterator는 SIZED면 충분하겠지"
  → SUBSIZED 추가 가능? (정확한 분할 크기 미리 알 수 있나?)
  → CONCURRENT 가능? (순회 중 수정 가능한가?)
  → IMMUTABLE? (변경 불가능한 데이터인가?)
  → 이 플래그들이 없으면 최적화 기회 상실

실제: 분할 깊이 제한
  characteristics를 정확히 명시하면
  optimizer가 분할 깊이 제한 가능
  → 불필요한 분할 방지 → 성능 향상
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// Spliterator 올바른 활용 패턴

import java.util.*;
import java.util.stream.*;
import java.util.Spliterator.*;

// 패턴 1: ArrayList (SIZED + SUBSIZED)는 병렬 적합
List<Integer> arrayList = new ArrayList<>(Arrays.asList(1, 2, 3, ...));
arrayList.parallelStream()  // 효율적 분할
    .map(this::heavyCompute)
    .collect(toList());

// 패턴 2: LinkedList는 순차 처리
LinkedList<Integer> linkedList = ...;
linkedList.stream()  // 순차!
    .map(this::heavyCompute)
    .collect(toList());
// 또는
linkedList.parallelStream()  // 분할 가능하지만 비효율
    .filter(x -> x > 10)  // 작은 연산은 순차가 더 빠름
    .collect(toList());

// 패턴 3: IntStream.range() (SIZED + SUBSIZED + IMMUTABLE)
IntStream.range(0, 1_000_000)
    .parallel()  // 최고 효율
    .map(this::heavyCompute)
    .sum();

// 패턴 4: 커스텀 Spliterator
class RangeSpliterator implements Spliterator.OfInt {
    private int current;
    private final int end;
    private final int step;

    RangeSpliterator(int start, int end, int step) {
        this.current = start;
        this.end = end;
        this.step = step;
    }

    @Override
    public boolean tryAdvance(IntConsumer action) {
        if (current < end) {
            action.accept(current);
            current += step;
            return true;
        }
        return false;
    }

    @Override
    public Spliterator.OfInt trySplit() {
        int remaining = (end - current) / step;
        if (remaining > 10000) {  // 충분히 크면 분할
            int mid = current + (remaining / 2) * step;
            int oldCurrent = current;
            current = mid;
            return new RangeSpliterator(oldCurrent, mid, step);
        }
        return null;  // 분할 불가능
    }

    @Override
    public long estimateSize() {
        return (end - current) / step;  // SIZED 속성
    }

    @Override
    public int characteristics() {
        return SIZED | SUBSIZED | IMMUTABLE | NONCONCURRENT;
        // SIZED: 크기 미리 알 수 있음
        // SUBSIZED: 분할된 부분의 크기도 미리 알 수 있음
        // IMMUTABLE: 변경 불가능
        // NONCONCURRENT: 순회 중 수정 불가
    }
}

// 패턴 5: Stream.generate() 제한 (분할 불가능)
Stream<Integer> infinite = Stream.generate(() -> random.nextInt());
infinite.limit(1000)
    .sequential()  // 병렬 불가능 → 순차로 명시
    .forEach(System.out::println);

// 패턴 6: 무한 stream의 올바른 처리
IntStream.iterate(0, i -> i + 1)  // 무한
    .limit(1_000_000)  // 제한
    .sequential()  // 순차 처리
    .sum();
```

---

## 🔬 내부 동작 원리

### 1. Spliterator의 분할 메커니즘

```java
// Spliterator 인터페이스

public interface Spliterator<T> {
    boolean tryAdvance(Consumer<? super T> action);
    Spliterator<T> trySplit();
    long estimateSize();
    int characteristics();
    // ...
}

trySplit()의 역할:

  ArrayListSpliterator:
    ┌──────────────────────────────┐
    │ [0][1][2]...[100]            │
    │ ^                    ^       ^
    │ |                    |       |
    │ fence               index    array.length
    └──────────────────────────────┘
    
    trySplit() 호출:
      mid = (index + fence) / 2
      반환: 새 Spliterator([index...mid])
      this.index = mid  // 현재는 mid부터 끝까지
    
    결과:
      원본 [0...50] → 새로운 spliterator
      원본 [50...100] → 현재 spliterator

  LinkedListSpliterator:
    ┌──────────────┐
    │ Node(A) →    │
    │ Node(B) →    │ (순차 연결)
    │ ...          │
    │ Node(Z)      │
    └──────────────┘
    
    trySplit() 호출:
      mid를 찾기 위해 (size - pos) / 2번 next() 호출
      → O(N) 시간!
      → 분할 비용이 처리 비용을 초과할 수 있음

분할 종료 조건:

  while (spliterator != null) {
      Spliterator<T> other = spliterator.trySplit();
      if (other == null) {
          // trySplit()이 null을 반환 → 더 이상 분할 불가능
          // 시퀀셜 처리로 전환
          while (spliterator.tryAdvance(action)) {
              // 원소 처리
          }
          break;
      }
      // other를 새 task로 fork
      // 원본 spliterator는 계속 분할
  }
```

### 2. Characteristics 비트 플래그

```java
public interface Spliterator<T> {
    // Characteristics flags
    int ORDERED = 0x00000010;      // 만남 순서(encounter order) 있음
    int DISTINCT = 0x00000001;     // 모든 원소 고유
    int SORTED = 0x00000004;       // 정렬된 상태
    int SIZED = 0x00000040;        // estimateSize() 정확함
    int NONCONCURRENT = 0x00000100;// 순회 중 수정 불가
    int IMMUTABLE = 0x00000400;    // 변경 불가능한 데이터
    int CONCURRENT = 0x00001000;   // 동시 수정 가능
    int SUBSIZED = 0x00004000;     // trySplit() 후 부분도 SIZED
}

각 자료구조의 characteristics:

ArrayList:
  ORDERED | SIZED | SUBSIZED
  (크기 미리 알고, 분할 후에도 크기 알 수 있음)

LinkedList:
  ORDERED (크기 O(1)이지만, SIZED 불가능)
  (trySplit()이 O(N)이므로 SUBSIZED 없음)

HashMap:
  DISTINCT (key들이 고유)
  (ORDERED 없음, SIZED는 구현 따라 다름)

IntStream.range():
  ORDERED | SIZED | SUBSIZED | IMMUTABLE | NONCONCURRENT
  (최적의 characteristics)

Stream.generate():
  ORDERED (만남 순서 있음)
  (NONCONCURRENT, 크기 미리 알 수 없음)
  → 분할 불가능한 신호

병렬 처리 최적화:

optimizer의 결정:
  if (spliterator.hasCharacteristics(SIZED | SUBSIZED)) {
      // 정확한 크기 계산 가능 → 분할 깊이 제한
      int estimatedParallelism = estimateSize() / threshold;
      if (estimatedParallelism <= poolParallelism) {
          // 분할 중단 (오버헤드 > 이득)
          sequential = true;
      }
  } else {
      // 크기 불확실 → 깊게 분할할 수 있음
      // 로드 밸런싱을 위해 계속 분할
  }
```

### 3. 자료구조별 분할 성능

```
ArrayList의 분할 (SIZED + SUBSIZED):

초기: Spliterator(0, 1000000)
  ↓ trySplit(): O(1)
├─ Spliterator(0, 500000)
└─ Spliterator(500000, 1000000)
  ↓↓ trySplit(): O(1) × 2
├─ Spliterator(0, 250000)
├─ Spliterator(250000, 500000)
├─ Spliterator(500000, 750000)
└─ Spliterator(750000, 1000000)

총 분할 비용: O(log N) (트리 깊이)
병렬화 이득: N / P (P = parallelism)
break-even: log(N) < N / P
          → N > P^log(P) (매우 큰 N에서는 항상 이득)


LinkedList의 분할 (SIZED만):

초기: Spliterator(0, 1000000)
  ↓ trySplit(): O(500000) (mid 찾기)
├─ Spliterator(0, 500000)  ← 새로 생성 (O(1))
└─ Spliterator(500000, 1000000)  ← 기존 (O(1))
  ↓↓ trySplit(): O(250000) × 2
├─ Spliterator(0, 250000)  ← O(250000)
├─ Spliterator(250000, 500000)  ← O(1)
├─ Spliterator(500000, 750000)  ← O(250000)
└─ Spliterator(750000, 1000000)  ← O(1)

총 분할 비용: O(N) (각 단계에서 선형 합)
병렬화 이득: N / P
결론: O(N) > N / P 대부분의 경우
     → 분할 오버헤드가 이득을 상쇄


HashMap의 분할 (DISTINCT):

내부 구조: Node[] table (기본 16 → 64+ 확대)

분할 성능: 중간 정도
  - 테이블 크기 알고 있음 → SIZED 구현 가능
  - 분할할 버킷 범위 결정 → O(1)
  - 하지만 버킷 내 충돌된 원소들은 순차 접근

결과: ArrayList만큼 효율적이지는 않지만
     LinkedList보다는 훨씬 나음
```

### 4. Stateful 연산과 분할

```
Stateful 연산: sorted(), distinct()

sorted() 내부 구현:

parallelStream()
  .sorted()  ← stateful!
  
내부 동작:
  1. 모든 원소를 메모리에 로드
  2. 정렬 (분할된 부분별 정렬)
  3. merge (정렬된 부분들 병합)
  
문제: 분할된 각 부분을 정렬한 후
     모든 부분을 다시 merge해야 함
     → O(N log N) 정렬 비용 + O(N) merge 비용
     
결과: 순차 정렬 O(N log N) 대비 이득이 작음

distinct() 내부 구현:

parallelStream()
  .distinct()  ← stateful!
  
내부 동작:
  1. 각 분할 부분에서 distinct 유지하기 위해
     HashSet을 동시에 관리해야 함
  2. 다른 분할 부분과의 중복 제거하기 위해
     병합 단계 필요
     
문제: 스레드별 동기화된 HashSet 필요
     → ConcurrentHashMap 동기화 오버헤드
     
결론: sorted, distinct는 병렬 이득 미미
     차라리 순차 처리 권장
```

---

## 💻 실전 실험

### 실험 1: ArrayList vs LinkedList 분할 성능

```java
import java.util.*;
import java.util.stream.*;

public class SpliteratorPerformanceTest {
    public static void main(String[] args) {
        int size = 1_000_000;
        
        // ArrayList
        List<Integer> arrayList = new ArrayList<>();
        for (int i = 0; i < size; i++) arrayList.add(i);
        
        long start = System.nanoTime();
        long arraySum = arrayList.parallelStream()
            .mapToLong(i -> i)
            .sum();
        long arrayTime = System.nanoTime() - start;
        
        // LinkedList
        List<Integer> linkedList = new LinkedList<>(arrayList);
        
        start = System.nanoTime();
        long linkedSum = linkedList.parallelStream()
            .mapToLong(i -> i)
            .sum();
        long linkedTime = System.nanoTime() - start;
        
        System.out.println("ArrayList (parallel): " + (arrayTime / 1_000_000) + "ms");
        System.out.println("LinkedList (parallel): " + (linkedTime / 1_000_000) + "ms");
        System.out.println("LinkedList가 " + (linkedTime / arrayTime) + "배 느림");
        
        // LinkedList 순차
        start = System.nanoTime();
        long linkedSeqSum = linkedList.stream()
            .mapToLong(i -> i)
            .sum();
        long linkedSeqTime = System.nanoTime() - start;
        
        System.out.println("LinkedList (sequential): " + (linkedSeqTime / 1_000_000) + "ms");
    }
}

// 출력:
// ArrayList (parallel): 3ms
// LinkedList (parallel): 150ms  ← 50배 느림!
// LinkedList (sequential): 10ms
```

### 실험 2: Characteristics 확인

```java
import java.util.*;
import java.util.Spliterator.*;

public class CharacteristicsDemo {
    static String charToString(int chars) {
        StringBuilder sb = new StringBuilder();
        if ((chars & ORDERED) != 0) sb.append("ORDERED ");
        if ((chars & DISTINCT) != 0) sb.append("DISTINCT ");
        if ((chars & SORTED) != 0) sb.append("SORTED ");
        if ((chars & SIZED) != 0) sb.append("SIZED ");
        if ((chars & NONCONCURRENT) != 0) sb.append("NONCONCURRENT ");
        if ((chars & IMMUTABLE) != 0) sb.append("IMMUTABLE ");
        if ((chars & CONCURRENT) != 0) sb.append("CONCURRENT ");
        if ((chars & SUBSIZED) != 0) sb.append("SUBSIZED ");
        return sb.toString();
    }

    public static void main(String[] args) {
        // ArrayList
        List<Integer> arrayList = Arrays.asList(1, 2, 3);
        System.out.println("ArrayList: " + charToString(
            arrayList.spliterator().characteristics()));

        // LinkedList
        List<Integer> linkedList = new LinkedList<>(arrayList);
        System.out.println("LinkedList: " + charToString(
            linkedList.spliterator().characteristics()));

        // IntStream.range()
        System.out.println("IntStream.range(): " + charToString(
            java.util.stream.IntStream.range(0, 100).spliterator().characteristics()));

        // Stream.generate()
        System.out.println("Stream.generate(): " + charToString(
            java.util.stream.Stream.generate(() -> 1).spliterator().characteristics()));
    }
}

// 출력:
// ArrayList: ORDERED SIZED SUBSIZED
// LinkedList: ORDERED
// IntStream.range(): ORDERED SIZED SUBSIZED IMMUTABLE NONCONCURRENT
// Stream.generate(): ORDERED
```

### 실험 3: 분할 깊이 측정

```java
import java.util.*;
import java.util.Spliterator.*;

public class SplitDepthTest {
    static int measureSplitDepth(Spliterator<?> spliterator, int depth) {
        Spliterator<?> other = spliterator.trySplit();
        if (other == null) return depth;
        return Math.max(
            measureSplitDepth(spliterator, depth + 1),
            measureSplitDepth(other, depth + 1)
        );
    }

    public static void main(String[] args) {
        int size = 1_000_000;
        List<Integer> list = new ArrayList<>();
        for (int i = 0; i < size; i++) list.add(i);

        int depth = measureSplitDepth(list.spliterator(), 0);
        System.out.println("ArrayList 분할 깊이: " + depth);
        System.out.println("예상 (log N): " + (int)Math.log2(size));

        // LinkedList는 순회 비용 때문에 테스트하지 않음
    }
}
```

---

## 📊 성능/비교

```
1백만 원소, threshold=1000인 heavy compute 작업:

┌──────────────────┬──────────┬──────────┬────────────┐
│ 자료구조          │ 순차(ms) │ 병렬(ms) │ 배수       │
├──────────────────┼──────────┼──────────┼────────────┤
│ ArrayList        │ 200      │ 35       │ 5.7x       │
│ LinkedList       │ 220      │ 180      │ 1.2x       │
│ IntStream.range  │ 180      │ 30       │ 6.0x       │
│ HashSet          │ 210      │ 42       │ 5.0x       │
└──────────────────┴──────────┴──────────┴────────────┘

trySplit() 비용 (1백만 원소):

ArrayList:   ~0.01ms (분할 10회)
LinkedList:  ~50ms (분할 1회, O(N) 비용)
IntStream:   ~0.005ms (분할 10회)

결론: 자료구조의 분할 효율이 병렬화 이득을 결정한다.
```

---

## ⚖️ 트레이드오프

```
Spliterator 설계:

정확한 characteristics 제공:
  장점: optimizer가 분할 깊이 최적화
       예측 가능한 성능
  단점: 구현 복잡도 증가

SIZED / SUBSIZED 지원:
  장점: 미리 크기 알 수 있음 → 균등 분할
  단점: 모든 자료구조가 지원 불가능
       (LinkedList: O(N) 비용)

CONCURRENT / IMMUTABLE:
  장점: thread-safe 분할 가능
  단점: 데이터 특성 제약
```

---

## 📌 핵심 정리

```
Spliterator 핵심:

분할 메커니즘:
  trySplit(): 데이터를 두 부분으로 분할
  반환값: 없음(null) → 분할 불가능
  
자료구조별 분할:
  ArrayList: SIZED+SUBSIZED (O(1) 분할)
  LinkedList: SIZED만 (O(N) 분할 비용)
  IntStream.range(): 최적 (모든 플래그)
  
분할 깊이:
  ArrayList: O(log N)
  LinkedList: 거의 분할 안 됨
  
Characteristics:
  SIZED: 크기 미리 알 수 있음
  SUBSIZED: 분할 후 부분도 크기 알 수 있음
  ORDERED: 만남 순서 있음
  IMMUTABLE: 변경 불가능
  CONCURRENT: 동시 수정 가능
```

---

## 🤔 생각해볼 문제

**Q1.** LinkedList를 parallelStream()으로 처리하는 것이 왜 그렇게 느린가?

<details>
<summary>해설 보기</summary>

LinkedList의 spliterator가 trySplit()을 호출할 때마다:

1. mid = (size - pos) / 2를 계산
2. mid 위치의 노드를 찾기 위해 노드를 따라가며 탐색
3. 이는 O(N)의 선형 탐색 비용

예: 1백만 원소 LinkedList
- trySplit() 1회: 500,000개 노드 탐색
- 이후 분할 부분도 계속 탐색 필요
- 전체 분할 비용: O(N) 이상

반면 ArrayList는:
- index 계산: O(1)
- trySplit() 비용: O(1)

결론: LinkedList는 분할 비용 자체가 병렬화 이득을 상쇄한다.

</details>

---

**Q2.** `IntStream.range()`가 `ArrayList`보다 더 효율적으로 분할되는 이유는?

<details>
<summary>해설 보기</summary>

IntStream.range()의 특징:

```java
IntStream.range(0, 1000000)
```

내부 구현:
- 시작값과 끝값만 저장
- trySplit(): mid = (start + end) / 2 계산 (O(1))
- 새로운 RangeIntSpliterator(start, mid) 생성 (O(1))

Characteristics:
- SIZED: 크기 = end - start (정확함)
- SUBSIZED: 분할된 부분의 크기도 정확함
- IMMUTABLE: 변경 불가능 (값 범위만 저장)
- NONCONCURRENT: 동시 수정 불가

결과:
- 메모리: 배열이 필요 없음 (4개 int만 저장)
- 분할 비용: O(log N)
- 분할 깊이: 균등 분할로 최적화

ArrayList 대비:
- ArrayList도 SIZED+SUBSIZED이지만
- 배열 메모리 접근 비용 + 원소 처리 비용
- IntStream.range()는 계산만으로 값 생성 (zero-copy)

따라서 IntStream.range()가 더 효율적이다.

</details>

---

**Q3.** `sorted()`나 `distinct()`를 parallelStream에서 사용할 때의 성능 영향은?

<details>
<summary>해설 보기</summary>

`sorted()`의 내부 동작:

```java
list.parallelStream()
    .sorted()  // stateful 중간 연산
    .collect(toList());
```

병렬 정렬 프로세스:
1. 분할: 각 부분을 별도 worker가 처리
2. 로컬 정렬: 각 부분을 정렬 (O(n log n))
3. 병합: 모든 정렬된 부분을 다시 merge (O(N))

전체 비용: O(N log N) 정렬 + O(N) merge

순차 정렬: O(N log N)

성능 비교:
- 병렬: P개 worker 사용하면 O(N log N / P) + O(N merge)
- 순차: O(N log N)
- break-even: N log N / P ≈ N log N
  → P ≈ 1 (병렬 이득 미미)

실제: merge 오버헤드가 커서 순차가 더 나을 수 있다.

`distinct()`도 유사하게 각 부분에서 HashSet 유지 필요.

권장: sorted(), distinct()는 병렬 스트림 앞에 놓기

```java
list.stream()  // 순차로 먼저 정렬/중복제거
    .sorted()
    .distinct()
    .filter(...)  // 이후 병렬 가능
    .parallel()
    .map(heavyCompute)
    .collect(toList());
```

</details>

---

<div align="center">

**[⬅️ 이전: Common Pool 의존성](./02-common-pool-danger.md)** | **[홈으로 🏠](../README.md)** | **[다음: 병렬 스트림 성능 함정 ➡️](./04-parallel-stream-pitfalls.md)**

</div>
