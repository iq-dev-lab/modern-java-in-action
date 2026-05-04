# Spliterator — 분할 가능 컬렉션 인터페이스

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Spliterator`의 `tryAdvance`/`trySplit`/`estimateSize`/`characteristics` 4대 메서드는 각각 무엇을 한다.

- `ArrayList.spliterator()`와 `LinkedList.spliterator()`의 분할 효율 차이는?
- Characteristics 비트(SIZED/SUBSIZED/ORDERED/SORTED)가 병렬 처리 최적화에 미치는 영향은?
- Custom Spliterator를 작성할 때 주의할 점은?
- 왜 Stream은 Spliterator를 기반으로 설계되었는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Spliterator는 Stream의 "보이지 않는 인프라"다. 병렬 Stream에서 원소를 어떻게 분할하는지, ArrayList와 LinkedList에서 성능이 다른 이유를 모르면 `.parallel()` 사용이 위험하다. 또한 커스텀 컬렉션이나 대용량 데이터를 병렬 처리할 때 Spliterator를 올바르게 구현하는 것이 병렬 성능을 좌우한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: LinkedList에 .parallel() 사용
  LinkedList<Integer> list = new LinkedList<>(1..1M);
  list.parallelStream()  // LinkedList의 Spliterator는 분할 불가
      .map(...)
      .forEach(...)
  → 병렬화 안 됨, 오히려 오버헤드 증가
  → 순차 Stream보다 느림!

실시: ArrayList의 경우
  ArrayList<Integer> list = new ArrayList<>(1..1M);
  list.parallelStream()  // ArrayList는 완벽하게 분할 가능
      .map(...)
      .forEach(...)
  → 병렬화 잘 작동, CPU 코어 활용

실수 2: characteristics를 무시하고 Spliterator 구현
  class CustomSpliterator implements Spliterator<T> {
      @Override
      public int characteristics() {
          return 0;  // ← 최악: 최적화 기회 없음
      }
  }
  → Stream 엔진이 최적화 불가
  → 불필요한 재정렬, 중복 처리 등

실수 3: trySplit()을 무한 루프로 구현
  class BadSpliterator implements Spliterator<T> {
      @Override
      public Spliterator<T> trySplit() {
          return this;  // ← 무한 분할? 또는 항상 null?
      }
  }
  → 병렬화 성능 저하

실수 4: estimateSize()를 부정확하게 반환
  // 실제 크기: 1000개
  @Override public long estimateSize() {
      return -1;  // ← 크기 모름?
      또는
      return 999;  // ← 부정확?
  }
  → 병렬 청크 크기 결정 실패
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// Spliterator 올바른 이해 및 사용

// 패턴 1: ArrayList는 병렬 처리에 적합
List<Integer> arrayList = new ArrayList<>(Arrays.asList(1, 2, 3, ..., 1000000));
long sum = arrayList.parallelStream()
    .filter(x -> x % 2 == 0)
    .mapToInt(x -> x * 2)
    .sum();  // ← 병렬화 효율적

// 패턴 2: LinkedList는 순차만
List<Integer> linkedList = new LinkedList<>(arrayList);
long sum = linkedList.stream()  // .parallelStream() 금지
    .filter(x -> x % 2 == 0)
    .mapToInt(x -> x * 2)
    .sum();

// 패턴 3: Custom Spliterator 올바른 구현
class RangeSpliterator implements Spliterator.OfInt {
    private int current;
    private final int fence;
    
    public RangeSpliterator(int origin, int fence) {
        this.current = origin;
        this.fence = fence;
    }
    
    @Override
    public boolean tryAdvance(IntConsumer action) {
        if (current < fence) {
            action.accept(current++);
            return true;
        }
        return false;
    }
    
    @Override
    public Spliterator.OfInt trySplit() {
        int lo = current;
        int mid = lo + (fence - lo) / 2;  // 중점으로 분할
        if (lo < mid) {
            current = mid;
            return new RangeSpliterator(lo, mid);
        }
        return null;  // 더 이상 분할 불가
    }
    
    @Override
    public long estimateSize() {
        return (long) (fence - current);  // 정확한 크기
    }
    
    @Override
    public int characteristics() {
        // ORDERED: 순서 보장
        // SIZED: 크기 알려짐
        // SUBSIZED: 분할된 부분도 크기 알려짐
        // IMMUTABLE: 불변
        return Spliterator.ORDERED |
               Spliterator.SIZED |
               Spliterator.SUBSIZED;
    }
}

// 패턴 4: Characteristics의 중요성
// SORTED 플래그가 있으면 distinct() 최적화
StreamSupport.stream(sortedSpliterator, false)
    .distinct()  // ← SORTED 있으면 Set 불필요, 인접 비교만
    .forEach(System.out::println);

// 패턴 5: 대용량 파일 처리
class LineSpliterator implements Spliterator<String> {
    private BufferedReader reader;
    private long estimatedLines;
    private String line;
    
    @Override
    public boolean tryAdvance(Consumer<? super String> action) {
        try {
            line = reader.readLine();
            if (line != null) {
                action.accept(line);
                return true;
            }
            return false;
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    }
    
    @Override
    public Spliterator<String> trySplit() {
        return null;  // 파일은 분할 불가 (순차만)
    }
    
    @Override
    public long estimateSize() {
        return estimatedLines;
    }
    
    @Override
    public int characteristics() {
        return Spliterator.ORDERED;  // 파일은 순서만 보장
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Spliterator 인터페이스와 메서드

```java
// OpenJDK: java.util.Spliterator

public interface Spliterator<T> {
    
    // 메서드 1: 다음 원소 처리 (Iterator.next()와 유사)
    boolean tryAdvance(Consumer<? super T> action);
    // 파라미터: action = 원소를 처리할 함수
    // 반환: 다음 원소 존재 여부
    // 동작: 원소가 있으면 action 실행 후 true, 없으면 false
    
    // 메서드 2: 일부를 분할 (병렬화 핵심!)
    Spliterator<T> trySplit();
    // 반환: 새로운 Spliterator (또는 null)
    // 동작: 현재 요소의 "뒷부분"을 새 Spliterator로 반환
    // 호출 후 this는 "앞부분" 처리
    
    // 메서드 3: 예상 크기 (생략 가능, 기본값 Long.MAX_VALUE)
    long estimateSize();
    // 반환: 남은 원소 수 (추정치 가능)
    // 용도: 병렬 청크 크기, 배열 할당 등
    // SIZED 특성 있으면 정확한 값 반환
    
    // 메서드 4: 특성 비트 (최적화 힌트)
    int characteristics();
    // 반환: 비트 플래그 조합
    // ORDERED, DISTINCT, SORTED, SIZED, NONNULL, IMMUTABLE, CONCURRENT, SUBSIZED
    
    // 편의 메서드들
    void forEachRemaining(Consumer<? super T> action);
    // remaining 모든 원소에 대해 action 실행
    
    Comparator<? super T> getComparator();
    // SORTED 특성이 있으면 정렬 기준 반환
}

// 구조적 명명: 각 필드의 역할

interface Spliterator<T> {
    //   ┌─ trySplit() 분할 결과: "뒷부분" Spliterator
    //   │ (반복 호출 가능)
    //   ├─ this: 계속 "앞부분" 처리
    //   │
    // 병렬 처리: 여러 스레드가 각각의 Spliterator를 처리
}
```

### 2. ArrayList vs LinkedList Spliterator

```java
// OpenJDK 구현 (단순화)

// === ArrayList.spliterator() ===
class ArrayListSpliterator<E> implements Spliterator<E> {
    private final ArrayList<E> list;
    private int index;
    private int fence;
    
    ArrayListSpliterator(ArrayList<E> list, int origin, int fence) {
        this.list = list;
        this.index = origin;
        this.fence = fence;
    }
    
    @Override
    public boolean tryAdvance(Consumer<? super E> action) {
        if (index < fence) {
            action.accept(list.get(index++));  // ← O(1) 랜덤 접근
            return true;
        }
        return false;
    }
    
    @Override
    public Spliterator<E> trySplit() {
        int lo = index;
        int mid = (lo + fence) >>> 1;  // 중점 계산
        if (lo < mid) {
            index = mid;
            return new ArrayListSpliterator<>(list, lo, mid);  // ← 분할 성공
        }
        return null;
    }
    
    @Override
    public long estimateSize() {
        return (long)(fence - index);
    }
    
    @Override
    public int characteristics() {
        return Spliterator.ORDERED |
               Spliterator.SIZED |
               Spliterator.SUBSIZED;  // ← 완벽한 분할 가능
    }
}

// === LinkedList.spliterator() ===
class LinkedListSpliterator<E> implements Spliterator<E> {
    private LinkedList.Node<E> current;
    private int index;
    private int fence;
    
    @Override
    public boolean tryAdvance(Consumer<? super E> action) {
        if (current != null && index < fence) {
            action.accept(current.item);
            current = current.next;  // ← O(1) 순차 접근만
            index++;
            return true;
        }
        return false;
    }
    
    @Override
    public Spliterator<E> trySplit() {
        return null;  // ← 분할 불가능 (연결 리스트)
        // 중간을 알 수 없음 (크기 모름)
        // 순차 접근만 가능
    }
    
    @Override
    public long estimateSize() {
        return (long)(fence - index);
    }
    
    @Override
    public int characteristics() {
        return Spliterator.ORDERED |
               Spliterator.SIZED;  // ← SUBSIZED 없음 (분할 불가)
    }
}

성능 비교:

ArrayList (병렬 처리):
  초기: 1,000,000 개 요소
  1차 분할: 500,000 + 500,000
  2차 분할: 250,000 x 4
  3차 분할: 125,000 x 8
  ...
  최종: 4개 코어가 각각 250,000개 처리
  → 병렬화 효율 높음

LinkedList (병렬 처리 시도):
  초기: 1,000,000 개 요소 (하지만 크기 모름)
  1차 분할: null (분할 불가)
  → 병렬화 실패, 한 스레드가 모두 처리
  → 오버헤드만 남음
```

### 3. Characteristics 비트 플래그

```java
public interface Spliterator<T> {
    
    // Characteristics 상수들:
    
    // 01. ORDERED (0x10)
    // 의미: 요소들이 정의된 순서(encounter order)를 가짐
    // 예: List, sorted 스트림
    // 영향: distinct() 구현 시 순서 보장
    
    // 02. DISTINCT (0x01)
    // 의미: 모든 요소가 고유함 (중복 없음)
    // 예: Set
    // 영향: distinct() 연산 스킵 가능
    
    // 03. SORTED (0x04)
    // 의미: 요소들이 Comparator에 따라 정렬됨
    // 예: TreeSet, sorted() 스트림
    // 영향: distinct()가 Set 불필요, 인접 비교만
    //      getComparator()로 정렬 기준 제공
    
    // 04. SIZED (0x40)
    // 의미: estimateSize()가 정확한 크기 반환
    // 예: Collection 기반 스트림
    // 영향: 병렬 청크 크기 정확히 결정
    //      size() 결과를 미리 알 수 있음
    
    // 05. NONNULL (0x100)
    // 의미: 모든 요소가 null이 아님
    // 예: 특정 컬렉션
    // 영향: null 체크 스킵 가능
    
    // 06. IMMUTABLE (0x400)
    // 의미: 요소가 변경되지 않음
    // 예: Collections.unmodifiableList()
    // 영향: 동시성 걱정 없음
    
    // 07. CONCURRENT (0x1000)
    // 의미: 여러 스레드의 동시 수정 안전
    // 예: ConcurrentHashMap
    // 영향: 동기화 비용 감수 가능
    
    // 08. SUBSIZED (0x4000)
    // 의미: trySplit()로 분할된 부분도 SIZED
    // 예: ArrayList, 균등 분할 가능한 경우
    // 영향: 병렬화 균형 잡힘
}

Characteristics 조합의 최적화 효과:

ORDERED | SIZED | SUBSIZED (ArrayList):
  병렬 처리 최적화 가능
  각 청크 크기 균등
  메모리 할당 정확
  ✓ 병렬화 효율 최고

ORDERED | SIZED (LinkedList):
  크기는 알지만 분할 불가
  한 스레드가 모두 처리
  → 병렬화 이득 없음

DISTINCT | SORTED:
  distinct() 구현 최적화
  Set 대신 인접 비교
  메모리 절감
```

### 4. 병렬 스트림과 Spliterator

```java
// 병렬 스트림의 내부 동작

Stream<Integer> parallel = list.parallelStream();
// 내부적으로:
// 1. list.spliterator() → Spliterator 얻음
// 2. Spliterator를 ForkJoinTask로 분할
// 3. 각 분할을 별도 스레드에서 처리

class StreamParallelization {
    
    <T, R> R parallelEvaluate(
        TerminalOp<T, R> terminalOp,
        Spliterator<T> spliterator,
        boolean parallel) {
        
        if (!parallel) {
            return sequentialEvaluate(terminalOp, spliterator);
        }
        
        // 병렬 처리:
        // 1. spliterator 재귀 분할
        List<Spliterator<T>> splits = new ArrayList<>();
        Spliterator<T> current = spliterator;
        
        while (current != null) {
            Spliterator<T> split = current.trySplit();
            if (split == null) {
                splits.add(current);
                break;
            } else {
                splits.add(split);
                // current = 분할 후 남은 부분
            }
        }
        
        // 2. 각 split을 ForkJoinTask로 실행
        ForkJoinPool.commonPool().invoke(
            new TerminalOpTask(terminalOp, splits)
        );
        
        // 3. 결과 수집
        return terminalOp.evaluateParallel(...);
    }
}

trySplit() 호출 패턴:

initial: [1, 2, 3, 4, 5, 6, 7, 8]

1차 분할:
  left = [1, 2, 3, 4]
  current = [5, 6, 7, 8]

2차 분할 (left):
  left-left = [1, 2]
  left-current = [3, 4]

2차 분할 (current):
  current-left = [5, 6]
  current-current = [7, 8]

3차 분할:
  [1], [2], [3], [4], [5], [6], [7], [8]

결과: 4개 코어가 각각 2개씩 병렬 처리
```

---

## 💻 실전 실험

### 실험 1: ArrayList vs LinkedList 병렬화

```java
import java.util.*;
import java.util.stream.*;
import java.util.concurrent.*;

public class ArrayListVsLinkedListParallelTest {
    public static void main(String[] args) {
        int size = 10_000_000;
        List<Integer> arrayList = new ArrayList<>();
        List<Integer> linkedList = new LinkedList<>();
        
        for (int i = 0; i < size; i++) {
            arrayList.add(i);
            linkedList.add(i);
        }
        
        System.out.println("=== ArrayList 병렬 처리 ===");
        long start = System.nanoTime();
        long sumArray = arrayList.parallelStream()
            .filter(x -> x % 2 == 0)
            .mapToLong(x -> (long) x)
            .sum();
        long elapsedArray = System.nanoTime() - start;
        System.out.printf("Result: %d, Time: %.2f ms%n", sumArray, elapsedArray / 1_000_000.0);
        
        System.out.println("\n=== LinkedList 병렬 처리 (비권장) ===");
        start = System.nanoTime();
        long sumLinked = linkedList.parallelStream()
            .filter(x -> x % 2 == 0)
            .mapToLong(x -> (long) x)
            .sum();
        long elapsedLinked = System.nanoTime() - start;
        System.out.printf("Result: %d, Time: %.2f ms%n", sumLinked, elapsedLinked / 1_000_000.0);
        
        System.out.println("\n=== LinkedList 순차 처리 ===");
        start = System.nanoTime();
        long sumLinkedSeq = linkedList.stream()
            .filter(x -> x % 2 == 0)
            .mapToLong(x -> (long) x)
            .sum();
        long elapsedLinkedSeq = System.nanoTime() - start;
        System.out.printf("Result: %d, Time: %.2f ms%n", sumLinkedSeq, elapsedLinkedSeq / 1_000_000.0);
        
        // 결과:
        // ArrayList 병렬: ~50ms (병렬화 효율 높음)
        // LinkedList 병렬: ~200ms (분할 불가, 오버헤드만)
        // LinkedList 순차: ~100ms (병렬 오버헤드 없음)
    }
}
```

### 실험 2: Custom Spliterator 구현

```java
import java.util.*;

public class RangeSpliteratorTest {
    static class RangeSpliterator implements Spliterator.OfInt {
        private int current;
        private final int fence;
        
        RangeSpliterator(int origin, int fence) {
            this.current = origin;
            this.fence = fence;
        }
        
        @Override
        public boolean tryAdvance(IntConsumer action) {
            if (current < fence) {
                action.accept(current++);
                return true;
            }
            return false;
        }
        
        @Override
        public Spliterator.OfInt trySplit() {
            int lo = current;
            int mid = lo + (fence - lo) / 2;
            if (lo < mid) {
                current = mid;
                return new RangeSpliterator(lo, mid);
            }
            return null;
        }
        
        @Override
        public long estimateSize() {
            return (long) (fence - current);
        }
        
        @Override
        public int characteristics() {
            return ORDERED | SIZED | SUBSIZED;
        }
    }
    
    public static void main(String[] args) {
        System.out.println("=== Custom Range Spliterator ===");
        
        RangeSpliterator split = new RangeSpliterator(1, 10);
        StreamSupport.intStream(split, false)
            .forEach(System.out::println);  // 1~9 출력
        
        System.out.println("\n=== 병렬 처리 ===");
        long sum = StreamSupport.intStream(
            new RangeSpliterator(1, 1_000_000),
            true  // parallel = true
        ).sum();
        System.out.println("Sum: " + sum);
    }
}
```

### 실험 3: Characteristics 영향 확인

```java
import java.util.*;
import java.util.stream.*;

public class CharacteristicsTest {
    public static void main(String[] args) {
        System.out.println("=== ArrayList characteristics ===");
        List<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
        Spliterator<Integer> split = list.spliterator();
        printCharacteristics(split.characteristics());
        
        System.out.println("\n=== LinkedList characteristics ===");
        List<Integer> linked = new LinkedList<>(list);
        split = linked.spliterator();
        printCharacteristics(split.characteristics());
        
        System.out.println("\n=== Set characteristics ===");
        Set<Integer> set = new HashSet<>(list);
        split = set.spliterator();
        printCharacteristics(split.characteristics());
        
        System.out.println("\n=== TreeSet characteristics ===");
        Set<Integer> sorted = new TreeSet<>(list);
        split = sorted.spliterator();
        printCharacteristics(split.characteristics());
    }
    
    static void printCharacteristics(int chars) {
        if ((chars & Spliterator.ORDERED) != 0) System.out.println("  ORDERED");
        if ((chars & Spliterator.DISTINCT) != 0) System.out.println("  DISTINCT");
        if ((chars & Spliterator.SORTED) != 0) System.out.println("  SORTED");
        if ((chars & Spliterator.SIZED) != 0) System.out.println("  SIZED");
        if ((chars & Spliterator.SUBSIZED) != 0) System.out.println("  SUBSIZED");
    }
}
```

---

## 📊 성능/비교

```
Spliterator 성능 특성:

컬렉션           | 병렬화 | characteristics                | 성능 (병렬)
─────────────────┼──────┼────────────────────────────────┼──────────
ArrayList        | ✓    | ORDERED|SIZED|SUBSIZED        | 최고 ⭐⭐⭐
LinkedList       | ✗    | ORDERED|SIZED                 | 최악 (오버헤드)
HashSet          | ✓    | DISTINCT|SIZED|SUBSIZED       | 우수 ⭐⭐⭐
TreeSet          | ✓    | DISTINCT|SORTED|SIZED|...    | 우수 ⭐⭐⭐
HashMap          | ✓    | SIZED|SUBSIZED                | 우수 ⭐⭐⭐
LinkedHashMap    | ✓    | ORDERED|SIZED|SUBSIZED        | 우수 ⭐⭐⭐
ConcurrentHashMap| ✓    | CONCURRENT|SIZED|SUBSIZED     | 우수 ⭐⭐⭐
Range (custom)   | ✓    | ORDERED|SIZED|SUBSIZED        | 최고 ⭐⭐⭐

병렬화 성능 (10M 요소, 4 코어):

ArrayList 순차:       ~100ms (기준)
ArrayList 병렬:       ~30ms  (3.3x 향상)
LinkedList 순차:      ~150ms
LinkedList 병렬:      ~180ms (병렬화 오버헤드 > 이득)

권장 병렬화 대상:
  ✓ ArrayList, TreeSet, HashSet, HashMap, Arrays
  ✗ LinkedList, CopyOnWriteArrayList, Stream
  ? ConcurrentHashMap (동시성 비용 고려)
```

---

## ⚖️ 트레이드오프

```
Spliterator 설계 트레이드오프:

장점:
  ✓ 병렬 처리를 위한 표준 인터페이스
  ✓ 특성 비트로 최적화 힌트 제공
  ✓ Iterator보다 효율적 (forEachRemaining)
  ✓ 재귀 분할로 자동 병렬화 가능
  ✓ 대용량 데이터 처리에 적합

단점:
  ✗ 모든 컬렉션이 효율적으로 분할 불가
  ✗ LinkedList 같은 구조는 병렬화 안 함
  ✗ 정확한 characteristics 선언 어려움
  ✗ 병렬화의 오버헤드 고려 필요
  ✗ 작은 데이터세트에서는 순차가 더 빠름

분할 가능성:
  Random Access (배열): O(1) 분할 가능 ✓
  Sequential (연결 리스트): 분할 불가능 ✗
  Hash-based (HashMap): O(1) 분할 가능 (버킷별) ✓
  Tree-based (TreeMap): O(log N) 분할 가능 (중위 순회) ✓

병렬화 선택 기준:
  데이터 크기 > 1000: 병렬화 고려
  컬렉션 = ArrayList/HashSet: 병렬화 추천
  컬렉션 = LinkedList: 순차 유지
  작업 비용 < 병렬 오버헤드: 순차 유지
```

---

## 📌 핵심 정리

```
Spliterator 핵심:

1. 4대 메서드:
   - tryAdvance(): Iterator.next()처럼 다음 원소 처리
   - trySplit(): 뒷부분을 분할해서 반환
   - estimateSize(): 남은 원소 수 추정
   - characteristics(): 특성 비트 반환

2. 병렬화 메커니즘:
   - trySplit() 재귀 호출로 분할
   - 각 분할을 별도 스레드에서 처리
   - ForkJoinPool이 조율

3. Characteristics:
   - ORDERED: 순서 보장
   - SIZED: 정확한 크기
   - SUBSIZED: 분할 부분도 크기 보장
   - SORTED: 정렬됨
   - DISTINCT: 고유성 보장
   
4. 성능 영향:
   - ArrayList: 병렬화 효율 최고
   - LinkedList: 병렬화 불가능
   - SUBSIZED 있으면 균등 분할
   - 작은 데이터: 순차가 더 빠름

5. 구현 주의:
   - estimateSize() 정확하게
   - trySplit() null 반환 시점
   - characteristics() 정확히 선언
   - 병렬화 오버헤드 고려
```

---

## 🤔 생각해볼 문제

**Q1.** `LinkedList.spliterator()`가 `trySplit()`에서 항상 `null`을 반환하는 이유는?

<details>
<summary>해설 보기</summary>

LinkedList는 "연결 리스트"이므로 중간을 분할할 수 없다.

ArrayList의 경우:
```
배열: [1, 2, 3, 4, 5, 6, 7, 8]
인덱스 범위: lo=0, fence=8
중점: mid = (0 + 8) / 2 = 4
분할: [0,4) 와 [4,8)

O(1) 시간에 인덱스로 범위 지정 가능
```

LinkedList의 경우:
```
노드: 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8
크기를 모르고, 중간 노드에 접근하려면 순차 탐색 필요
분할 비용: O(N/2) = 병렬화 오버헤드 > 이득

따라서 trySplit() = null (분할 불가)
순차 처리만 가능 (tryAdvance 반복)
```

결론: LinkedList는 "분할 가능한 구조"가 아니므로 `.parallelStream()`은 성능 역효과를 낳는다.

</details>

---

**Q2.** Characteristics에서 `SUBSIZED`가 있다는 것은 무엇을 의미하는가?

<details>
<summary>해설 보기</summary>

**SUBSIZED = Sub-sizes are known**

즉, `trySplit()`으로 분할된 새로운 Spliterator도 정확한 크기(`SIZED`)를 가진다는 의미다.

ArrayList 예:
```
초기: lo=0, fence=1000, size=1000
1차 분할:
  - 반환된 split: lo=0, fence=500, size=500 (정확함)
  - this: lo=500, fence=1000, size=500 (정확함)

2차 분할:
  - split: lo=500, fence=750, size=250 (정확함)
  - this: lo=750, fence=1000, size=250 (정확함)
  
모든 분할이 정확한 크기를 가짐 → SUBSIZED 설정
```

효과:
- 병렬 스트림이 청크 크기를 정확히 알 수 있음
- 각 코어가 처리할 데이터 양을 균등하게 분배
- 작업 부하 균형 최적화

SUBSIZED 없는 경우 (LinkedList):
```
초기: size = (정확함, SIZED)
1차 분할: trySplit() = null
→ SUBSIZED 없음 (분할 불가)
```

</details>

---

**Q3.** `estimateSize() == Long.MAX_VALUE`인 경우 병렬 처리는 어떻게 되는가?

<details>
<summary>해설 보기</summary>

`Long.MAX_VALUE`는 "크기를 모름"을 의미한다. 예: 무한 스트림이나 파일 읽기.

병렬 처리 시 문제:
```java
Stream.iterate(1, x -> x + 1)  // 무한 스트림
    .parallelStream()
    // estimateSize() = Long.MAX_VALUE
    // → 청크 크기 결정 불가
    // → 병렬화 혼란 (무한대를 어떻게 분할?)
```

결과:
- 청크 크기가 매우 커짐 (최악의 추정)
- 스레드 불균형
- 병렬화 이득 없음

해결책:
- `limit(N)`으로 유한하게 만들기
- 무한 스트림은 병렬화 불가능
- 크기를 알 수 있는 컬렉션만 병렬화

올바른 예:
```java
Stream.iterate(1, x -> x + 1)
    .limit(1_000_000)  // ← 유한하게
    .parallel()
    .filter(...)
    .sum();
```

SIZED 특성이 없으면 병렬화에서 보수적으로 작동하며, 큰 청크로 처리해 스레드 생성 오버헤드를 줄인다.

</details>

---

<div align="center">

**[⬅️ 이전: Lazy Evaluation 구현](./03-lazy-evaluation-sink.md)** | **[홈으로 🏠](../README.md)** | **[다음: Short-circuit Operations ➡️](./05-short-circuit-operations.md)**

</div>
