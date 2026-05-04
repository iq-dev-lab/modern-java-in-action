# Stateless vs Stateful 연산 — map vs sorted/distinct

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `map()`/`filter()`같은 stateless 연산이 한 원소씩 처리 가능한 근본적 이유는?
- `sorted()`/`distinct()`/`limit()` 같은 stateful 연산이 전체 데이터를 모아야 하는 이유는?
- Stateful 연산이 병렬 스트림 성능을 떨어뜨리는 메커니즘은 무엇인가?
- 파이프라인 재정렬 시 stateful 연산의 위치가 성능에 미치는 영향은?
- 같은 결과를 내는 stateless와 stateful 구현의 성능 차이는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

병렬 스트림 성능을 최대화하려면 stateful 연산의 위치가 중요하다. Filter → sorted()는 데이터를 대폭 축소한 후 정렬하지만, sorted() → filter()는 모든 데이터를 정렬한 후 버린다. 대용량 데이터에서는 이 차이가 초 단위 성능 차이를 만든다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Stateful 연산 남용
  Stream.of(1..100M)
      .sorted()       // ← 100M 개 정렬 (O(100M log 100M))
      .filter(x > threshold)  // ← 필터링
      .limit(100)
      .collect(toList());
  
  → 효과: 불필요하게 모든 데이터 정렬
  → 시간: O(N log N) 낭비

실수 2: 여러 stateful 연산 체이닝
  Stream.of(...)
      .distinct()    // O(N) Set 생성
      .sorted()      // O(N log N)
      .distinct()    // O(N) 다시 Set?
      .collect(toList());
  
  → 중복 distinct() (이미 정렬되었으므로 Set 불필요)

실수 3: 병렬 스트림에서 stateful 연산의 비용 무시
  parallelStream()
      .sorted()      // ← 병렬 정렬은 단순하지 않음
                      // 각 청크 정렬 + 병합 필요 (오버헤드 큼)
      .forEach(...)
  
  → 오히려 순차보다 느릴 수 있음

실수 4: Stateful 연산의 메모리 오버헤드 간과
  List<String> urls = ... // 100M 요소
  urls.stream()
      .distinct()    // ← HashSet에 100M URL 저장?
      .collect(toList());
  
  → 메모리: 수 GB (String 객체 참조 저장)
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// Stateless vs Stateful 올바른 이해

// 패턴 1: Stateless 우선 (필터링으로 데이터 축소)
Stream.of(1..100M)
    .filter(x -> x % 2 == 0)        // Stateless, O(N)
    .filter(x -> x > 1000)          // Stateless, O(축소된 크기)
    .sorted()                       // Stateful, O(축소된 * log)
    .limit(100)                     // Short-circuit
    .collect(toList());
// 정렬: 축소된 데이터만 정렬 (효율적)

// 패턴 2: Distinct의 최적화
// Case A: SORTED 플래그 없음
Stream.of(3, 1, 3, 2, 1)
    .distinct()    // HashSet 생성, O(N)
    .forEach(System.out::println);

// Case B: SORTED 플래그 있음 (정렬 후)
Stream.of(3, 1, 3, 2, 1)
    .sorted()      // O(N log N), SORTED 플래그 설정
    .distinct()    // Set 불필요, 인접 비교 O(N)
    .forEach(System.out::println);
// 정렬되었으므로 distinct()는 Set 불필요

// 패턴 3: Limit으로 Stateful 비용 절감
Stream.iterate(1, x -> x + 1)
    .filter(x -> isPrime(x))
    .limit(100)      // ← stateless 먼저!
    .sorted()        // 100개만 정렬 O(100 log 100)
    .collect(toList());

// 반대 (재앙):
// sorted()         // 무한 소수 정렬? 메모리 폭발
// .limit(100)

// 패턴 4: Parallel stream에서 Stateful
// 좋은 예: 필터 후 정렬
parallelStream()
    .filter(x > threshold)       // 병렬 필터 (데이터 축소)
    .collect(Collectors.toList()) // 병렬 수집
    new ArrayList<>(temp)
    .sort(...)                    // 필터된 데이터만 정렬 (sequential)
    // 또는 간단히:
    parallelStream()
        .filter(x > threshold)
        .collect(toCollection(ArrayList::new))
        .sort(...);

// 나쁜 예: 병렬 정렬
parallelStream()
    .sorted()        // 각 청크 정렬 + 병합 (오버헤드)
    .forEach(...)
// 순차 정렬이 더 빠를 수 있음

// 패턴 5: Stateful 연산 최소화
// 원래:
List<Integer> result = stream
    .distinct()
    .sorted()
    .map(x -> x * 2)
    .filter(x > threshold)
    .collect(toList());

// 최적화: filter 먼저, distinct 제거 (필터로 충분)
List<Integer> result = stream
    .filter(x > threshold)      // Stateless
    .map(x -> x * 2)            // Stateless
    .sorted()                   // Stateful (축소된 크기)
    .collect(toList());
```

---

## 🔬 내부 동작 원리

### 1. Stateless vs Stateful 메커니즘

```java
// OpenJDK: java.util.stream 구현 분석

// === Stateless Operation (map) ===
class StatelessOp<E_IN, E_OUT> extends ReferencePipeline<E_IN, E_OUT> {
    
    // 각 원소를 즉시 처리 → 다음 Sink로 전달
    @Override
    Sink<E_IN> opWrapSink(int flags, Sink<E_OUT> sink) {
        return new Sink.ChainedReference<E_IN, E_OUT>(sink) {
            @Override
            public void accept(E_IN u) {
                E_OUT v = transform(u);  // ← 한 원소만 처리
                downstream.accept(v);    // ← 즉시 다음 Sink에 전달
            }
        };
    }
    
    // 특징:
    // - 메모리 사용: O(1) (원소 1개만 기억)
    // - 시간 복잡도: O(N)
    // - 병렬화: 즉시 가능 (특정 청크 처리만 해도 됨)
    // - 순서 보장: 선택적 (ORDERED 플래그에 따라)
}

// === Stateful Operation (sorted) ===
class StatefulOp<E_IN, E_OUT> extends ReferencePipeline<E_IN, E_OUT> {
    
    // 모든 원소를 수집 → 처리 → 결과 전달
    @Override
    Sink<E_IN> opWrapSink(int flags, Sink<E_OUT> sink) {
        return new Sink.ChainedReference<E_IN, E_OUT>(sink) {
            List<E_OUT> buffer = new ArrayList<>();  // ← 상태 저장!
            
            @Override
            public void accept(E_IN u) {
                buffer.add(u);  // ← 모든 원소 수집
            }
            
            @Override
            public void end() {
                buffer.sort(...);  // ← 모든 원소 도착 후 처리
                buffer.forEach(downstream::accept);
            }
        };
    }
    
    // 특징:
    // - 메모리 사용: O(N) (모든 원소 저장)
    // - 시간 복잡도: O(N log N) 정렬인 경우
    // - 병렬화: 복잡함 (분할 정렬 + 병합)
    // - 순서 보장: 필수 (ORDERED)
}

구조 비교 다이어그램:

Stateless (map):
  item1 → [transform] → downstream.accept() ← 즉시
  item2 → [transform] → downstream.accept() ← 즉시
  item3 → [transform] → downstream.accept() ← 즉시
  메모리: O(1) (1개씩)

Stateful (sorted):
  item1 → [buffer.add()] ← 수집
  item2 → [buffer.add()] ← 수집
  item3 → [buffer.add()] ← 수집
  [메모리: O(N)]
  (모두 도착)
  [buffer.sort()] → [forEach downstream]
  메모리: O(N) (모두 보관)
```

### 2. 병렬 Stream에서의 Stateful 연산

```java
// 병렬 정렬의 복잡성

parallelStream()
    .sorted()

// 내부:
// 1. ForkJoinPool이 Stream을 청크로 분할
// 2. 각 청크를 별도 스레드에서 정렬
// 3. 정렬된 청크들을 병합 (merge)
// 4. 최종 순서 보장

Fork-Join 정렬 구조:

입력: [3, 1, 4, 1, 5, 9, 2, 6]
     ├─ [3, 1, 4, 1] (Thread 1)
     └─ [5, 9, 2, 6] (Thread 2)
           ↓
        정렬 (각 스레드)
           ↓
     ├─ [1, 1, 3, 4]
     └─ [2, 5, 6, 9]
           ↓
        병합 (main thread)
           ↓
     [1, 1, 2, 3, 4, 5, 6, 9]

오버헤드:
  병렬 분할: O(log N)
  각 청크 정렬: O((N/P) log(N/P)), P = 스레드 수
  병합: O(N) ← 직렬 작업!
  총: O(N log N) 같지만 오버헤드 증가

순차 정렬 vs 병렬 정렬:
  작은 데이터 (< 10K): 순차가 더 빠름
  중간 데이터 (10K ~ 1M): 비슷
  큰 데이터 (> 1M): 병렬이 빠름 (저수준 최적화 + 멀티코어)
```

### 3. Stateful 연산의 종류와 특성

```java
// 다양한 Stateful 연산들:

// 1. sorted(Comparator)
// 메모리: O(N)
// 시간: O(N log N)
// 병렬화: 복잡함

// 2. distinct()
// 메모리: O(unique 크기)
// 시간: O(N) (HashSet 기반)
// 최적화: SORTED 있으면 Set 불필요, O(N)으로 감소

// 3. limit(n)
// 메모리: O(min(N, n))
// 시간: O(min(N, n))
// 병렬화: 쉬움 (short-circuit)
// 특수: Stateful이지만 early-exit 가능

// 4. skip(n)
// 메모리: O(1)
// 시간: O(N)
// 병렬화: 어려움 (정확한 건너뛰기)

// 5. takeWhile(), dropWhile() (Java 9+)
// 메모리: O(1) ~ O(k)
// 시간: O(N)
// 병렬화: 쉬움 (조건으로 경계 결정)

Stateful 연산 플래그:

enum StreamOpFlag {
    IS_STATEFUL = 0x80000000;  // ← Stateful 표시
}

Stateful 연산은 병렬화 시:
1. 전체 파이프라인 재분석
2. 청크 분할 중단
3. 단일 스레드로 강제 실행 (또는 복잡한 병합 필요)
```

### 4. 성능 분석 예

```
파이프라인 재정렬의 성능 영향:

초기 파이프라인:
  Stream.of(1..100M)
      .sorted()       // 100M 정렬 O(100M log 100M) ~ 1초
      .filter(x > 90M) // 1000개만 필터링
      .map(x -> x * 2)
      .collect(toList());

최적화 후:
  Stream.of(1..100M)
      .filter(x > 90M) // 1000개로 축소 O(100M)
      .sorted()        // 1000개 정렬 O(1000 log 1000) ~ 1ms
      .map(x -> x * 2)
      .collect(toList());

성능 개선: 1000배!

메모리 비교:

초기 (sorted 먼저):
  버퍼 크기: 100M * 8 bytes = 800MB (int)
  
최적화 (filter 먼저):
  버퍼 크기: 1000 * 8 bytes = 8KB
  개선: 100,000배!
```

---

## 💻 실전 실험

### 실험 1: Stateless vs Stateful 메모리 사용

```java
import java.util.*;
import java.util.stream.*;

public class StatelessVsStatefulMemoryTest {
    public static void main(String[] args) {
        int size = 1_000_000;
        List<Integer> list = new ArrayList<>();
        for (int i = 0; i < size; i++) {
            list.add(i);
        }
        
        System.out.println("=== Stateless (map) ===");
        long start = System.nanoTime();
        long sum = list.stream()
            .map(x -> x * 2)  // O(1) 메모리
            .filter(x -> x > threshold)
            .mapToLong(x -> (long) x)
            .sum();
        long elapsed = System.nanoTime() - start;
        System.out.printf("Sum: %d, Time: %.2f ms%n", sum, elapsed / 1_000_000.0);
        
        System.out.println("\n=== Stateful (sorted) ===");
        start = System.nanoTime();
        List<Integer> sorted = list.stream()
            .sorted()  // O(N) 메모리 (배열 복사)
            .filter(x -> x > threshold)
            .limit(100)
            .collect(Collectors.toList());
        elapsed = System.nanoTime() - start;
        System.out.printf("Size: %d, Time: %.2f ms%n", sorted.size(), elapsed / 1_000_000.0);
        
        // 결과: map은 빠르고 메모리 효율적
        //      sorted는 느리고 메모리 사용
    }
}
```

### 실험 2: 파이프라인 순서 최적화

```java
import java.util.*;
import java.util.stream.*;

public class PipelineOrderOptimizationTest {
    public static void main(String[] args) {
        List<Integer> list = new ArrayList<>();
        for (int i = 0; i < 10_000_000; i++) {
            list.add(i);
        }
        
        System.out.println("=== sorted() 먼저 (비효율) ===");
        long start = System.nanoTime();
        List<Integer> result1 = list.stream()
            .sorted()           // 10M 정렬
            .filter(x -> x > 9_999_000)  // 1000개만 선택
            .collect(Collectors.toList());
        long elapsed1 = System.nanoTime() - start;
        
        System.out.println("\n=== filter() 먼저 (효율) ===");
        start = System.nanoTime();
        List<Integer> result2 = list.stream()
            .filter(x -> x > 9_999_000)  // 1000개로 축소
            .sorted()           // 1000개만 정렬
            .collect(Collectors.toList());
        long elapsed2 = System.nanoTime() - start;
        
        System.out.printf("Sorted first: %.2f ms%n", elapsed1 / 1_000_000.0);
        System.out.printf("Filter first: %.2f ms%n", elapsed2 / 1_000_000.0);
        System.out.printf("Speedup: %.1fx%n", (double) elapsed1 / elapsed2);
        
        // 결과: filter 먼저가 10배 이상 빠름
    }
}
```

### 실험 3: Distinct 최적화 (SORTED 플래그)

```java
import java.util.*;
import java.util.stream.*;

public class DistinctOptimizationTest {
    public static void main(String[] args) {
        List<Integer> list = new ArrayList<>();
        Random rand = new Random(42);
        for (int i = 0; i < 1_000_000; i++) {
            list.add(rand.nextInt(100_000));  // 많은 중복
        }
        
        System.out.println("=== distinct() 먼저 ===");
        long start = System.nanoTime();
        long count1 = list.stream()
            .distinct()    // HashSet 생성
            .count();
        long elapsed1 = System.nanoTime() - start;
        
        System.out.println("\n=== sorted() → distinct() ===");
        start = System.nanoTime();
        long count2 = list.stream()
            .sorted()      // SORTED 플래그 설정
            .distinct()    // Set 불필요, 인접 비교 (최적화됨)
            .count();
        long elapsed2 = System.nanoTime() - start;
        
        System.out.printf("distinct first: %.2f ms (count=%d)%n", elapsed1 / 1_000_000.0, count1);
        System.out.printf("sorted first: %.2f ms (count=%d)%n", elapsed2 / 1_000_000.0, count2);
        
        // 결과: SORTED 있으면 distinct() 최적화로 더 빠를 수 있음
    }
}
```

---

## 📊 성능/비교

```
Stateless vs Stateful 성능 특성:

연산      | 메모리 | 시간    | 병렬화 | 특성
──────────┼────────┼────────┼────────┼──────────
map       | O(1)   | O(N)   | 최적   | stateless
filter    | O(1)   | O(N)   | 최적   | stateless
flatMap   | O(1)   | O(N)   | 최적   | stateless
sorted    | O(N)   | O(NlogN)| 복잡  | stateful
distinct  | O(u)   | O(N)   | 제한   | stateful
limit     | O(k)   | O(k)   | 우수   | stateful (특수)
skip      | O(1)   | O(N)   | 제한   | stateful

파이프라인 재정렬 영향 (100M 요소, 99% 필터링):

순서 1: sorted() → filter()
  시간: O(100M log 100M) + O(100M) ~ 2초
  메모리: O(100M) ~ 800MB

순서 2: filter() → sorted()
  시간: O(100M) + O(1M log 1M) ~ 50ms
  메모리: O(1M) ~ 8MB
  
개선: 40배 빠름, 100배 메모리 절감!

병렬 Stream 성능 (4코어):

100M 요소 sorted (병렬):
  순차: ~2초
  병렬 (직접): ~1.5초 (오버헤드 > 이득)
  
1M 요소 sorted (병렬):
  순차: ~20ms
  병렬: ~15ms (약간의 이득)
```

---

## ⚖️ 트레이드오프

```
Stateless vs Stateful 트레이드오프:

Stateless의 장점:
  ✓ O(1) 메모리
  ✓ 병렬화 최적 (각 청크 독립 처리)
  ✓ 빠른 실행 (직선적 흐름)
  ✓ 순서 보장 선택적

Stateless의 단점:
  ✗ 복잡한 연산은 표현 어려움
  ✗ 상태 유지 불가능
  ✗ 순서 정렬 불가능

Stateful의 장점:
  ✓ 정렬, 중복 제거 등 복잡한 연산 가능
  ✓ 최종 결과 보장
  ✓ 순서 정렬 가능

Stateful의 단점:
  ✗ O(N) 메모리 사용
  ✗ 병렬화 복잡함
  ✗ 느린 실행
  ✗ 무한 Stream 처리 불가능 (메모리)

설계 원칙:
1. 필터링으로 데이터 축소
2. Stateful 연산 미루기
3. Short-circuit으로 조기 종료
4. 병렬화 시 Stateless 우선
5. 메모리 제한 환경에서는 Stateless 만
```

---

## 📌 핵심 정리

```
Stateless vs Stateful 핵심:

1. Stateless (map, filter, flatMap):
   - 각 원소를 독립적으로 처리
   - O(1) 메모리, O(N) 시간
   - 병렬화 최적
   - 순서 선택적

2. Stateful (sorted, distinct, limit):
   - 모든 원소를 수집 후 처리
   - O(N) 메모리, O(N log N) 이상 시간
   - 병렬화 복잡함
   - 순서 필수

3. 성능 최적화:
   - Stateless 먼저 (데이터 축소)
   - Stateful 뒤로 (축소된 데이터 처리)
   - Short-circuit으로 조기 종료

4. 메모리 효율:
   - Stateless: 배열 생성 안 함
   - Stateful: 중간 배열/Set 필요

5. 병렬화:
   - Stateless: 각 청크 독립 처리
   - Stateful: 스레드 동기화 + 병합 필요

최적 파이프라인:
  filter (축소)
    → map (변환)
      → filter (재축소)
        → sorted (축소된 것만)
          → distinct (필요시)
            → limit (최종 제한)
```

---

## 🤔 생각해볼 문제

**Q1.** `filter() → sorted() → distinct()`와 `sorted() → distinct()`의 성능은 어떻게 다른가?

<details>
<summary>해설 보기</summary>

**Case 1: filter() → sorted() → distinct()**
```
10M 요소 (50% 필터)
  filter: O(10M) → 5M
  sorted: O(5M log 5M) ~ 중간 시간
  distinct: O(5M) 인접 비교 (SORTED 있으므로)
```

**Case 2: sorted() → distinct()**
```
10M 요소
  sorted: O(10M log 10M) ~ 긴 시간
  distinct: O(10M) 인접 비교 (SORTED 있으므로)
```

Case 1이 훨씬 빠름!
- filter가 50% 축소
- 정렬 대상이 5M으로 줄어듦
- distinct()는 둘 다 O(N)이지만 input이 다름

실제 성능:
```
Case 2: sorted(10M) = ~2초
Case 1: filter(10M) + sorted(5M) = ~0.5초 + ~1초 = ~1.5초
(filter 오버헤드는 무시할 수 있음)

개선: ~33% 빠름
```

원칙: **데이터 크기를 먼저 줄이고 stateful 적용**

</details>

---

**Q2.** Stateful 연산이 "상태"를 유지해야 하는데, 병렬 Stream에서 각 스레드의 상태는 어떻게 병합되는가?

<details>
<summary>해설 보기</summary>

**병렬 정렬의 병합 과정:**

```
초기 Stream: [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5]
분할: [3, 1, 4, 1] | [5, 9, 2, 6] | [5, 3, 5]

Thread 1:
  accept(3), accept(1), accept(4), accept(1)
  buffer = [3, 1, 4, 1]
  
Thread 2:
  accept(5), accept(9), accept(2), accept(6)
  buffer = [5, 9, 2, 6]
  
Thread 3:
  accept(5), accept(3), accept(5)
  buffer = [5, 3, 5]

Reduce Phase (병합):
  1. 각 buffer 로컬 정렬
     buffer1 = [1, 1, 3, 4]
     buffer2 = [2, 5, 6, 9]
     buffer3 = [3, 5, 5]
  
  2. 정렬된 버퍼들을 병합
     result = merge(buffer1, merge(buffer2, buffer3))
     = merge([1,1,3,4], merge([2,5,6,9], [3,5,5]))
     = merge([1,1,3,4], [2,3,5,5,5,6,9])
     = [1,1,2,3,3,4,5,5,5,6,9]
```

**Distinct의 병합:**
```
Thread 1: distinct 상태 = {1, 3, 4}
Thread 2: distinct 상태 = {2, 5, 6, 9}
Thread 3: distinct 상태 = {3, 5}

병합: Set.addAll() → {1, 2, 3, 4, 5, 6, 9}
```

**핵심:**
- 각 스레드가 로컬 상태(buffer, set) 유지
- 최후에 병합(merge) 단계
- 병합 자체가 추가 연산 (O(N) 이상)
- 병렬 오버헤드 > 이득 경우 많음

따라서 작은 데이터는 순차 정렬이 더 빠르다.

</details>

---

**Q3.** SORTED 플래그가 있으면 distinct()가 왜 Set 대신 인접 비교를 사용할까?

<details>
<summary>해설 보기</summary>

**정렬된 배열의 중복 제거:**

```
입력 (정렬됨): [1, 1, 2, 2, 2, 3, 5, 5]

Set 방식 (일반적):
  seen = new HashSet<>();
  stream.filter(x -> seen.add(x))
  
  seen.add(1) → true  → accept(1)
  seen.add(1) → false → skip
  seen.add(2) → true  → accept(2)
  seen.add(2) → false → skip
  seen.add(2) → false → skip
  ...
  
  메모리: O(unique 크기) = Set 유지

인접 비교 방식 (SORTED 있을 때):
  prev = null
  stream.filter(x -> {
      if (prev == null || prev != x) {
          prev = x;
          return true;
      }
      return false;
  })
  
  prev != 1 → true → accept(1), prev=1
  prev == 1 → false → skip
  prev != 2 → true → accept(2), prev=2
  prev == 2 → false → skip
  prev == 2 → false → skip
  ...
  
  메모리: O(1) ← 이전 값만 기억!
```

**효과:**
- 메모리: O(unique) → O(1)
- 시간: O(N) → O(N)
- 시간 복잡도는 같지만 메모리 대폭 절감

이것이 SORTED 플래그의 최적화 효과!

```java
// OpenJDK 코드:
if (streamFlags & SORTED) {
    // SORTED 있으면 이 구현 선택
    useAdjacentComparison();
} else {
    // 일반적인 Set 기반
    useHashSet();
}
```

</details>

---

<div align="center">

**[⬅️ 이전: Short-circuit Operations](./05-short-circuit-operations.md)** | **[홈으로 🏠](../README.md)** | **[다음: Collector 내부 ➡️](./07-collector-mutable-reduction.md)**

</div>
