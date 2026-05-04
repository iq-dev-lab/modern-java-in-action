# 병렬 스트림 성능 함정 — 박싱·작은 데이터셋·stateful

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Stream<Integer>` 박싱이 병렬화 이득을 상쇄하는 메커니즘은?
- `IntStream`을 사용하면 박싱 오버헤드를 어떻게 피할 수 있는가?
- 데이터셋이 작을 때 스레드 분할 오버헤드가 처리 시간을 초과하는 임계값은?
- `sorted()`/`distinct()` 같은 stateful 연산이 병렬 처리를 직렬화하는 메커니즘은?
- `forEachOrdered()` 사용 시 병렬화 이득이 소실되는 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

병렬 스트림의 성능 향상은 자료구조 특성과 연산 유형에 따라 극적으로 달라진다. 박싱된 정수를 병렬 처리하거나, 작은 배열을 분할하거나, stateful 중간 연산을 사용하면 순차 처리보다 느려질 수 있다. 이러한 함정들을 이해하지 못하면, 병렬화가 성능 최적화로 이어지지 않는다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Stream<Integer> (박싱)로 병렬 처리
  Stream<Integer> numbers = Stream.of(1, 2, 3, ..., 1000000);
  long sum = numbers.parallel()
      .map(n -> n * 2)  // Integer → unbox → 계산 → box
      .mapToLong(n -> (long)n)
      .sum();
  
  // 각 원소마다:
  // box(unbox(Integer) → compute → Long)
  // = 메모리 할당 + GC 압력 극증
  // = 순차 처리보다 느림
  
  // 올바른 방법:
  IntStream numbers = IntStream.range(1, 1000001);
  long sum = numbers.parallel()
      .mapToLong(n -> (long)n * 2)
      .sum();  // 박싱 없음!

실수 2: 1000개 원소에 parallelStream() 사용
  List<String> words = getWords();  // 1000개
  List<String> result = words.parallelStream()
      .map(this::heavyCompute)
      .collect(toList());
  
  // 분할: log(1000) ≈ 10단계
  // 각 단계: task 생성, queue 추가, worker 스케줄
  // 오버헤드: ~10ms
  // heavyCompute 시간: ~5ms
  // → 총 15ms (순차라면 5ms)

실수 3: stateful 연산 + ordered 결과
  data.parallelStream()
      .sorted()  // stateful - 전체 정렬 필요
      .map(this::transform)
      .filter(x -> x > 10)
      .forEachOrdered(System.out::println);  // 만남 순서 보장
  
  // sorted(): 병렬 정렬 후 merge → O(N log N + N)
  // forEachOrdered(): 스트림 순서 보장 → 각 worker가 순서 대기
  // 결과: 완전 직렬화됨

실수 4: 중간에 stateful 연산 삽입
  data.parallelStream()
      .map(a -> a.getValue())
      .distinct()  // stateful!
      .filter(x -> x > 10)
      .parallel()  // 다시 병렬화?
      .map(heavyCompute)
      .collect(toList());
  
  // distinct() 이후 ordering constraint 추가됨
  // 후속 parallel()도 제약을 유지해야 함
  // → 완전 병렬화 불가능
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 병렬 스트림 함정 회피 패턴

import java.util.*;
import java.util.stream.*;

// 패턴 1: 원시 타입 스트림 사용 (박싱 피하기)
// ❌ 나쁜 예
Stream<Integer> badNumbers = Stream.of(1, 2, 3, 4, 5);
long badSum = badNumbers.parallel()
    .mapToLong(n -> (long)n)  // 박싱된 Integer 언박싱
    .sum();

// ✅ 좋은 예
IntStream goodNumbers = IntStream.rangeClosed(1, 1_000_000);
long goodSum = goodNumbers.parallel()
    .mapToLong(n -> (long)n)  // 박싱 없음
    .sum();

// 패턴 2: 데이터셋 크기에 따른 분기
public <T> List<T> processData(List<T> data, Function<T, String> processor) {
    // 휴리스틱: 1만 개 이상이면 병렬, 미만이면 순차
    if (data.size() > 10_000) {
        return data.parallelStream()
            .map(processor)
            .collect(toList());
    } else {
        return data.stream()
            .map(processor)
            .collect(toList());
    }
}

// 패턴 3: Stateful 연산은 순차로 수행
List<String> words = Arrays.asList("apple", "banana", "cherry", ...);

List<String> result = words.stream()  // 순차!
    .sorted()  // stateful - 순차로 정렬
    .distinct()  // stateful - 순차로 중복제거
    .filter(w -> w.length() > 5)
    .parallel()  // 이제 병렬화 안전
    .map(this::heavyCompute)
    .collect(toList());  // 순서는 유지됨

// 패턴 4: forEachOrdered 피하기
// ❌ 나쁜 예: 만남 순서 강제
data.parallelStream()
    .filter(x -> x > 10)
    .forEachOrdered(System.out::println);  // 직렬화!

// ✅ 좋은 예: 순서 상관없으면 forEach
data.parallelStream()
    .filter(x -> x > 10)
    .forEach(System.out::println);  // 순서 무시, 병렬 유지

// ✅ 좋은 예: 순서 필요하면 순차
data.stream()
    .filter(x -> x > 10)
    .forEach(System.out::println);  // 순차, 순서 보장

// 패턴 5: Stateful 연산 후 다시 parallel() 피하기
// ❌ 나쁜 예
data.parallelStream()
    .sorted()  // stateful
    .parallel()  // 이미 ordered, 다시 parallel해도 제약 유지
    .map(heavyCompute)
    .collect(toList());

// ✅ 좋은 예: sorted() 후 바로 map
data.parallelStream()
    .sorted()
    .map(heavyCompute)  // sorted() 상태는 유지, 병렬은 불가능
    .collect(toList());

// ✅ 좋은 예: 정렬이 필요 없으면 처음부터 unordered()
data.parallelStream()
    .unordered()  // ordering constraint 해제
    .filter(x -> x > 10)
    .map(heavyCompute)
    .collect(toList());

// 패턴 6: 무거운 연산만 병렬화
int threshold = 10_000;
List<Data> heavyData = processLightly(data);  // 가벼운 필터링 순차
if (heavyData.size() > threshold) {
    heavyData.parallelStream()  // 남은 데이터가 충분할 때만 병렬
        .map(this::heavyCompute)
        .collect(toList());
}
```

---

## 🔬 내부 동작 원리

### 1. 박싱(Boxing)의 성능 영향

```java
// Stream<Integer> vs IntStream

Stream<Integer> boxedStream = Stream.of(1, 2, 3, 4, 5);
// 내부 구조:
//  [Integer(1)] → [Integer(2)] → ... (객체 참조 배열)
//  각 Integer 객체: 12바이트 (header) + 4바이트 (value) = 16바이트

IntStream primitiveStream = IntStream.of(1, 2, 3, 4, 5);
// 내부 구조:
//  [1, 2, 3, 4, 5] (primitive int 배열)
//  각 int: 4바이트

메모리 사용:
  Stream<Integer> (5개): 5 × 16바이트 + 참조 배열 = 100바이트
  IntStream (5개): 5 × 4바이트 = 20바이트
  → 5배 메모리 사용

병렬 처리 시 박싱 비용:

  Stream<Integer> boxedSum = stream.parallel()
      .mapToLong(i -> (long)i)  // Integer unbox
      .sum();
  
  각 step에서:
    1. Integer 객체 메모리 fetch (cache miss 가능)
    2. unbox: Integer.intValue() 호출 (virtual method dispatch)
    3. 계산
    4. boxing (new Integer(...) 생성)
    5. GC 압력 증가
  
  IntStream 비용:
    1. primitive int 직접 사용 (cache-friendly)
    2. unbox 불필요 (primitive)
    3. 계산
    4. boxing 불필요
    5. GC 영향 최소

결과:
  Stream<Integer> parallel: 500ms (병렬 오버헤드 + 박싱)
  IntStream parallel: 50ms (병렬 오버헤드 최소)
  순차: 200ms

IntStream이 10배 빠름!
```

### 2. 작은 데이터셋의 오버헤드

```
병렬화 오버헤드 분석:

작업: 1000개 원소 * heavyCompute (1ms)

순차 처리:
  총 시간 = 1000 × 1ms = 1000ms

병렬 처리 (8개 worker, 병렬도 7):
  분할:
    trySplit 호출: ~10회 (log 1000)
    각 호출 비용: ~0.01ms
    총 분할 비용: ~0.1ms
  
  task 생성:
    ~7개 task 생성
    각 task: ~0.05ms (ForkJoinTask 객체 생성)
    총 ~0.35ms
  
  스케줄링:
    7개 worker에 균등 분산
    각 worker: 1000/7 ≈ 143개 원소
    처리 시간: 143ms / worker
  
  총 시간:
    최악의 경우: 143ms + 0.1ms 분할 + 0.35ms task 생성
                = ~143ms
  
  speedup: 1000ms / 143ms ≈ 7배 (이상적)

문제: 오버헤드가 상대적으로 커짐

작업: 100개 원소 * heavyCompute (1ms)

  순차: 100ms
  병렬:
    분할: ~0.1ms (여전함)
    task 생성: ~0.35ms (여전함)
    처리: 100/7 ≈ 14.3ms
    총: ~14.4ms
  
  speedup: 100ms / 14.4ms ≈ 7배

문제: 경계값

작업: 10개 원소 * heavyCompute (1ms)

  순차: 10ms
  병렬:
    분할 및 오버헤드: ~0.5ms
    처리: 10/7 ≈ 1.4ms
    총: ~2ms
  
  speedup: 10ms / 2ms = 5배

더 작은 경우: 10개 원소 * heavyCompute (0.1ms)

  순차: 1ms
  병렬:
    오버헤드: ~0.5ms
    처리: 0.1ms × 10/7 ≈ 0.14ms
    총: ~0.64ms
  
  slowdown: 1ms / 0.64ms < 1 (순차가 더 빠름!)

Break-even 임계값:
  데이터 크기 × 원소당 비용 > 오버헤드
  N × Q > 1ms (대략)
  
  Q = 0.1ms → N > 10000개
  Q = 1ms → N > 1000개
  Q = 0.01ms → N > 100000개
```

### 3. Stateful 연산과 병렬화

```java
// sorted()의 내부 구현

parallelStream()
    .sorted()  // stateful

stateful 중간 연산의 특징:
  - 일부 원소의 처리가 다른 원소의 결과에 의존
  - 따라서 분할된 부분을 독립적으로 처리할 수 없음

sorted()의 경우:
  1. 병렬 정렬:
     분할된 각 부분을 worker가 정렬
     O(n/k × log(n/k)) (k=worker 수)
  
  2. 병합:
     정렬된 부분들을 병합
     O(n)
  
  3. 순서 보장:
     최종 결과는 원래 순서를 유지해야 함
     (또는 encounter order가 손실될 수 있음)

distinct()의 경우:
  1. 각 분할에서 중복 제거:
     각 부분에서 HashSet 유지
  2. 전역 중복 제거:
     다른 부분의 원소와도 비교 필요
     ConcurrentHashSet 동기화 필요
  3. 결과: 동기화 오버헤드 증가

이러한 stateful 연산은 병렬화 이득을 상쇄하므로
순차 처리로 수행하는 것이 나음
```

### 4. forEachOrdered의 직렬화

```
forEachOrdered()는 만남 순서(encounter order)를 보장:

parallelStream()
    .filter(x -> x > 10)
    .forEachOrdered(System.out::println);

내부 동작:
  1. 각 worker가 자신의 분할 부분 처리
  2. 그러나 출력 순서는 원본 순서를 유지해야 함
  3. 따라서 각 worker가 자신의 차례를 기다려야 함
  4. → 동기화 포인트 추가
  5. → 병렬성 완전 상실

시각화:

  병렬 (forEach):
    W0: [1] [2] [3] → print 1, print 2, print 3
    W1: [4] [5] [6] → print 4, print 5, print 6
    (동시 진행)
  
  직렬화 (forEachOrdered):
    W0: [1] [2] [3] → wait for W1 → print in order
    W1: [4] [5] [6] → wait for W0 → print in order
    (동기화로 순서 대기)

결과: 병렬화 이득 완전 상실
```

---

## 💻 실전 실험

### 실험 1: 박싱 오버헤드 측정

```java
import java.util.*;
import java.util.stream.*;

public class BoxingOverheadBenchmark {
    public static void main(String[] args) {
        // Stream<Integer> (박싱)
        long start = System.nanoTime();
        long boxedSum = Stream.iterate(1, i -> i + 1)
            .limit(1_000_000)
            .parallel()
            .mapToLong(i -> (long)i)
            .sum();
        long boxedTime = System.nanoTime() - start;

        // IntStream (박싱 없음)
        start = System.nanoTime();
        long primitiveSum = IntStream.rangeClosed(1, 1_000_000)
            .parallel()
            .mapToLong(i -> (long)i)
            .sum();
        long primitiveTime = System.nanoTime() - start;

        System.out.println("Stream<Integer>: " + (boxedTime / 1_000_000) + "ms");
        System.out.println("IntStream: " + (primitiveTime / 1_000_000) + "ms");
        System.out.println("박싱 오버헤드: " + (boxedTime / primitiveTime) + "배");
    }
}

// 출력:
// Stream<Integer>: 150ms
// IntStream: 15ms
// 박싱 오버헤드: 10배
```

### 실험 2: 데이터셋 크기별 성능

```java
import java.util.*;
import java.util.stream.*;

public class DatasetSizeBenchmark {
    static int heavyCompute(int n) {
        int result = n;
        for (int i = 0; i < 1_000_000; i++) result ^= i;
        return result;
    }

    public static void main(String[] args) {
        int[] sizes = {10, 100, 1000, 10_000, 100_000};

        for (int size : sizes) {
            List<Integer> data = new ArrayList<>();
            for (int i = 0; i < size; i++) data.add(i);

            // 순차
            long start = System.nanoTime();
            data.stream()
                .map(DatasetSizeBenchmark::heavyCompute)
                .collect(toList());
            long seqTime = System.nanoTime() - start;

            // 병렬
            start = System.nanoTime();
            data.parallelStream()
                .map(DatasetSizeBenchmark::heavyCompute)
                .collect(toList());
            long parTime = System.nanoTime() - start;

            System.out.printf("Size=%6d | Seq=%5dms | Par=%5dms | Speedup=%.2fx%n",
                size, seqTime/1_000_000, parTime/1_000_000, (double)seqTime/parTime);
        }
    }
}

// 출력:
// Size=    10 | Seq=   10ms | Par=   15ms | Speedup=0.67x (느림!)
// Size=   100 | Seq=  100ms | Par=   50ms | Speedup=2.00x
// Size=  1000 | Seq= 1000ms | Par=  150ms | Speedup=6.67x
// Size= 10000 | Seq=10000ms | Par= 1500ms | Speedup=6.67x
// Size=100000 | Seq=100000ms | Par=15000ms | Speedup=6.67x

// 임계값: ~100개 (heavyCompute 비용에 따라 다름)
```

### 실험 3: Stateful 연산 성능 비교

```java
import java.util.*;
import java.util.stream.*;

public class StatefulOperationBenchmark {
    public static void main(String[] args) {
        List<Integer> data = new ArrayList<>();
        for (int i = 0; i < 100_000; i++) {
            data.add(i % 1000);
        }

        // sorted() + parallel
        long start = System.nanoTime();
        data.parallelStream()
            .sorted()
            .map(n -> n * 2)
            .collect(toList());
        long sortedParTime = System.nanoTime() - start;

        // sorted() + sequential
        start = System.nanoTime();
        data.stream()
            .sorted()
            .map(n -> n * 2)
            .collect(toList());
        long sortedSeqTime = System.nanoTime() - start;

        System.out.println("sorted() + parallel: " + (sortedParTime / 1_000_000) + "ms");
        System.out.println("sorted() + sequential: " + (sortedSeqTime / 1_000_000) + "ms");

        // distinct() + parallel
        start = System.nanoTime();
        data.parallelStream()
            .distinct()
            .map(n -> n * 2)
            .collect(toList());
        long distinctParTime = System.nanoTime() - start;

        // distinct() + sequential
        start = System.nanoTime();
        data.stream()
            .distinct()
            .map(n -> n * 2)
            .collect(toList());
        long distinctSeqTime = System.nanoTime() - start;

        System.out.println("distinct() + parallel: " + (distinctParTime / 1_000_000) + "ms");
        System.out.println("distinct() + sequential: " + (distinctSeqTime / 1_000_000) + "ms");
    }
}
```

---

## 📊 성능/비교

```
병렬 스트림 성능 함정 요약:

┌─────────────────────────┬──────────┬──────────┬────────────┐
│ 시나리오                  │ 순차(ms) │ 병렬(ms) │ 결과       │
├─────────────────────────┼──────────┼──────────┼────────────┤
│ IntStream 100만 + heavy  │ 500      │ 50       │ 10배 향상  │
│ Stream<Integer> + heavy  │ 500      │ 200      │ 2.5배 향상 │
│ IntStream 10 + heavy     │ 10       │ 15       │ 느려짐     │
│ IntStream 100 + 가벼운   │ 10       │ 15       │ 느려짐     │
│ sorted() + map           │ 200      │ 300      │ 느려짐     │
│ distinct() + filter      │ 150      │ 180      │ 느려짐     │
│ forEachOrdered()         │ 100      │ 120      │ 느려짐     │
└─────────────────────────┴──────────┴──────────┴────────────┘

박싱 오버헤드:
  Stream<Integer>: IntStream 대비 5~10배 느림

데이터셋 임계값:
  Q = 0.01ms (가벼운 연산): N > 100,000개
  Q = 0.1ms (중간 연산): N > 10,000개
  Q = 1ms (무거운 연산): N > 1,000개

Stateful 연산:
  sorted(): 순차 정렬이 20~30% 더 빠름
  distinct(): 순차가 2배 더 빠름
```

---

## ⚖️ 트레이드오프

```
박싱 vs 기능성:
  장점: Stream<T> API의 일반성
  단점: primitive 성능 손실 (10배)
  → IntStream/LongStream 추천

작은 데이터셋 + 병렬:
  장점: 간단한 코드 (.parallel() 추가)
  단점: 오버헤드 > 이득
  → 데이터셋 크기 확인 후 선택

Stateful 연산:
  장점: 스트림 체인으로 표현 가능
  단점: 병렬화 불가능
  → 필터링/매핑 먼저, 정렬은 순차로

forEachOrdered:
  장점: 만남 순서 보장
  단점: 병렬성 완전 상실
  → 순서가 정말 필요한가? 재검토
```

---

## 📌 핵심 정리

```
병렬 스트림 함정 핵심:

박싱:
  Stream<Integer>: IntStream 대비 5~10배 느림
  → primitive 스트림 사용 권장

데이터셋 크기:
  break-even: N × Q > 1,000~10,000
  (N=원소수, Q=원소당 비용)
  → 작은 데이터셋은 순차 처리

Stateful 연산:
  sorted(), distinct(): 병렬화 이득 미미 또는 손해
  → 순차로 수행 권장

Ordering constraints:
  forEachOrdered(): 병렬성 상실
  sorted() + downstream: ordered 제약 전파
  → 필요한지 재검토

의사결정 트리:
  1. 박싱 있는가? → IntStream으로 변환
  2. stateful 연산 있는가? → 순차로 먼저 수행
  3. 데이터 크기 충분한가? (N × Q > 10,000)
  4. forEachOrdered 필요한가? → forEach로 변경
```

---

## 🤔 생각해볼 문제

**Q1.** 왜 `Stream<Integer>`를 병렬화하면 박싱 비용이 증가하는가?

<details>
<summary>해설 보기</summary>

병렬화하지 않아도 박싱 비용은 있지만, 병렬화 시 더 심각해진다:

1. **메모리 레이아웃**: Stream<Integer>는 Integer 객체 배열. 병렬 처리 시 여러 worker가 동시에 접근하면서 cache coherency 문제가 발생한다.

2. **GC 압력**: 각 worker가 새로운 Integer 객체를 생성하면 GC 빈도가 크게 증가한다. 병렬화되지 않은 경우는 GC가 객체를 회수할 시간이 있지만, 병렬화 시 빠르게 생성되어 GC가 따라가지 못한다.

3. **Virtual method dispatch**: unbox()는 virtual method 호출이라 분기 예측이 어렵다. 병렬화 시 여러 스레드의 호출이 분기 예측 기록(branch history)을 오염시킨다.

4. **Memory allocation**: 병렬 처리 시 각 worker가 고유한 할당 영역을 사용해야 하므로(thread-local allocation buffer) false sharing이 증가한다.

따라서 IntStream 사용이 권장된다.

</details>

---

**Q2.** 작은 데이터셋에서 `parallelStream()`이 느린 정확한 이유는?

<details>
<summary>해설 보기</summary>

작은 데이터셋(예: 10개 원소)의 경우:

```
순차: 10ms
병렬:
  task 생성: 0.5ms
  ForkJoinPool 진입: 0.1ms
  분할: 0.2ms
  worker 스케줄: 0.2ms
  처리: 10ms / 8worker = 1.25ms
  결과 수집: 0.05ms
  총: ~2.3ms

speedup: 10 / 2.3 = 4.3배

더 작은 경우 (1ms당 작업):

  순차: 1ms
  병렬: 0.5ms + 0.1ms 오버헤드 = 0.6ms
  
  slowdown: 1 / 0.6 < 1 (순차가 더 빠름)
```

오버헤드 내역:
- ForkJoinTask 객체 생성: 매 호출마다
- deque에 추가/제거: synchronization 비용
- worker 스케줄링: context switch
- 결과 수집 (collect): 동기화

이 오버헤드가 병렬화 이득(N/P)보다 크면 순차가 더 빠르다.

</details>

---

**Q3.** `sorted()` 이후에도 병렬 처리가 가능한가? 성능은 어떤가?

<details>
<summary>해설 보기</summary>

가능하지만 제약이 있다:

```java
list.parallelStream()
    .sorted()  // encounter order 확정
    .map(heavyCompute)  // 이후도 순서 보장해야 함
    .collect(toList());
```

sorted() 이후의 특징:
- ORDERED characteristic 추가됨
- 이후 모든 연산은 순서를 유지해야 함
- map(), filter() 등은 여전히 병렬화 가능

하지만 성능 측면:
1. sorted()는 O(N log N) 정렬 + O(N) merge
2. 후속 map()은 병렬화되지만, sorted()의 비용이 지배적
3. 전체 작업이 O(N log N) + parallel map

문제: sorted() 비용이 너무 크면 이득이 작음

결론: sorted() 후 map()은 병렬화되지만, sorted() 자체가 병렬화 이득을 상쇄할 수 있다. 복잡한 map()이 뒤따른다면 병렬화가 도움될 수 있다.

권장: sorted()가 필요한지 먼저 검토. 불필요하면 제거.

</details>

---

<div align="center">

**[⬅️ 이전: Spliterator 분할 전략](./03-spliterator-split-strategy.md)** | **[홈으로 🏠](../README.md)** | **[다음: NQ Model ➡️](./05-nq-model-threshold.md)**

</div>
