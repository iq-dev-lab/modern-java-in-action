# Lazy Evaluation 구현 — Sink와 Push 모델

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Terminal 연산 호출 시 "역방향" Sink 체인을 구성하는 정확한 알고리즘은 무엇인가?
- Source에서 "한 원소씩 push"하면 모든 중간 연산이 "한 번의 패스"로 실행되는 메커니즘은?
- `forEach` 루프와 `Stream.forEach()` 간 본질적 차이가 없는 이유는?
- `peek()`이 디버깅 외에 왜 부작용을 만드는 위험한 연산인가?
- `Sink.cancellationRequested()`는 언제 참이 되고, 이것이 short-circuit을 어떻게 구현하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Lazy evaluation이 단순한 "성능 최적화"라고 생각하면 안 된다. 이것은 무한 Stream을 다룰 수 있게 하는 핵심 메커니즘이고, 각 원소가 모든 중간 연산을 "한 번에" 통과하게 하는 설계 철학이다. 이를 이해하지 못하면, 왜 `peek()`가 부작용을 일으키는지, 왜 병렬 Stream에서 예상과 다른 동작을 하는지 혼란스럽다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Sink를 단순한 "콜백"으로 생각
  map().forEach()를 보면 "map 함수가 먼저 실행되고, 
  그 결과에 forEach가 적용된다"고 착각
  
  실제: forEach가 Sink 체인을 구성하고,
       source에서 원소가 push될 때
       모든 중간 연산(Sink들)이 역순으로 호출됨

실수 2: peek()를 "부작용이 없는 안전한 함수"로 착각
  Stream.of(1, 2, 3)
      .peek(System.out::println)  // 프로덕션 코드에 사용?
      .filter(x > 1)
      .collect(toList());
  
  → peek()는 디버깅 목적
  → 부작용(로깅, 상태 변경)을 일으킬 수 있음
  → 프로덕션 코드에서 성능 저하 및 부작용 초래

실수 3: 중간 연산의 실행 순서를 잘못 이해
  Stream.of(1, 2, 3)
      .map(x -> { System.out.println("A: " + x); return x * 2; })
      .map(x -> { System.out.println("B: " + x); return x + 1; })
      .forEach(x -> System.out.println("C: " + x));
  
  예상: A: 1, A: 2, A: 3, B: 2, B: 4, B: 6, C: 3, C: 5, C: 7
  실제: A: 1, B: 2, C: 3, A: 2, B: 4, C: 5, A: 3, B: 6, C: 7
  
  → 각 원소가 모든 중간 연산을 한 번에 통과!

실수 4: forEach 루프와 Stream.forEach()가 다르다고 착각
  for (int x : list) { process(x); }
  vs
  list.stream().forEach(this::process);
  
  → 실제로는 거의 동일한 동작
  → 다만 Stream은 병렬화, 최적화 가능성이 있음
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// Lazy Evaluation과 Sink의 올바른 이해

// 패턴 1: 각 원소가 모든 중간 연산을 한 번에 통과
Stream.of(1, 2, 3)
    .filter(x -> {
        System.out.println("filter: " + x);
        return x > 1;
    })
    .map(x -> {
        System.out.println("map: " + x);
        return x * 2;
    })
    .forEach(x -> System.out.println("result: " + x));

// 출력 (각 원소별로 모든 연산 실행):
// filter: 1
// (skip, 필터링됨)
// filter: 2
// map: 2
// result: 4
// filter: 3
// map: 3
// result: 6

// 패턴 2: peek()는 디버깅만 (프로덕션 금지)
Stream.of(1, 2, 3, 4, 5)
    .filter(x -> x > 2)
    .peek(x -> System.out.println("DEBUG: " + x))  // 디버깅용 (임시)
    .map(x -> x * 2)
    .forEach(System.out::println);

// 패턴 3: cancellationRequested를 통한 short-circuit
Stream.iterate(1, x -> x + 1)
    .peek(x -> System.out.println("Checking: " + x))
    .filter(x -> x > 100)
    .limit(5)  // ← short-circuit
    .forEach(System.out::println);

// limit(5)의 Sink는 5번만 downstream.accept() 호출
// → 6번째는 Sink.cancellationRequested() = true 반환
// → source는 pull 중단

// 패턴 4: 복잡한 파이프라인에서 각 원소의 흐름 추적
Stream.of(1, 2, 3, 4, 5)
    .filter(x -> x % 2 == 0)      // Sink 1
    .map(x -> x * 10)              // Sink 2
    .filter(x -> x > 15)            // Sink 3
    .map(x -> {                    // Sink 4
        System.out.printf("[transform] %d → %d%n", x / 10, x + 100);
        return x + 100;
    })
    .forEach(x -> System.out.printf("[result] %d%n", x));

// 각 원소별 흐름:
// 1: filter1 skip
// 2: filter1 pass → map2(20) → filter3(20>15 pass) → map4(120) → forEach(120)
// 3: filter1 skip
// 4: filter1 pass → map2(40) → filter3(40>15 pass) → map4(140) → forEach(140)
// 5: filter1 skip
```

---

## 🔬 내부 동작 원리

### 1. Sink 인터페이스와 구현

```java
// OpenJDK: java.util.stream.Sink

public interface Sink<T> extends Consumer<T> {
    
    // 원소 수용 (Consumer 상속)
    void accept(T t);
    
    // 처리 완료 신호 (reduce/collect 등에서 호출)
    default void end() {}
    
    // 조기 종료 여부 (short-circuit)
    default boolean cancellationRequested() {
        return false;
    }
    
    // 데이터 특성 (병렬화 최적화)
    default void begin(long size) {}
}

// Sink 구현 예: Chained Sink (다음 Sink를 감싸기)
abstract static class ChainedReference<T_IN, T_OUT>
    implements Sink<T_IN> {
    
    protected final Sink<? super T_OUT> downstream;
    
    public ChainedReference(Sink<? super T_OUT> downstream) {
        this.downstream = downstream;
    }
    
    // 대부분의 구현은 downstream 위임
    @Override
    public void begin(long size) {
        downstream.begin(size);
    }
    
    @Override
    public void end() {
        downstream.end();
    }
    
    @Override
    public boolean cancellationRequested() {
        return downstream.cancellationRequested();
    }
}

// map() Sink 구현 예
class MapSink<T_IN, T_OUT> extends ChainedReference<T_IN, T_OUT> {
    private final Function<? super T_IN, ? extends T_OUT> mapper;
    
    @Override
    public void accept(T_IN u) {
        downstream.accept(mapper.apply(u));  // 매핑 후 전달
    }
}

// filter() Sink 구현 예
class FilterSink<T> extends ChainedReference<T, T> {
    private final Predicate<? super T> predicate;
    
    @Override
    public void accept(T u) {
        if (predicate.test(u)) {
            downstream.accept(u);  // 필터링을 통과한 것만 전달
        }
    }
}

// limit() Sink 구현 예
class LimitSink<T> extends ChainedReference<T, T> {
    private long remaining;
    
    @Override
    public void accept(T u) {
        if (remaining > 0) {
            remaining--;
            downstream.accept(u);
            if (remaining == 0) {
                // ← cancellationRequested 신호 (source 중단)
            }
        }
    }
    
    @Override
    public boolean cancellationRequested() {
        return remaining == 0;  // 제한에 도달
    }
}
```

### 2. Terminal 호출 → Sink 체인 구성 → 평가 흐름

```java
// OpenJDK: java.util.stream.ReferencePipeline

public class ReferencePipeline {
    
    public void forEach(Consumer<? super E_OUT> action) {
        evaluate(TerminalOp.makeForEach(action));
    }
    
    final <R> R evaluate(TerminalOp<E_OUT, R> terminalOp) {
        // Step 1: Terminal Sink 생성
        Sink<E_OUT> terminalSink = terminalOp.createSink(terminalOp.getOpFlags());
        // forEach의 경우: ForEachSink(action)
        
        // Step 2: Sink 체인 구성 (역방향)
        // map과 filter 같은 중간 연산 노드들이 Sink를 감싸기
        Sink<E_OUT> sink = wrapSink(terminalSink);
        
        // Step 3: Source에서 원소 push (순방향)
        // Spliterator를 사용해 원소를 하나씩 sink에 전달
        copyInto(sink, spliterator());
    }
    
    // Sink 체인 구성 핵심: 역방향으로 wrapSink 호출
    @SuppressWarnings("unchecked")
    private Sink<E_IN> wrapSink(Sink<E_OUT> sink) {
        for (@SuppressWarnings("rawtypes") AbstractPipeline stage = this;
             stage != sourceStage;
             stage = stage.previousStage) {
            sink = stage.opWrapSink(sink.getStreamShape(), sink);
            // 각 노드가 자신의 Sink를 생성하여 이전 Sink를 감싸기
        }
        return (Sink<E_IN>) sink;
    }
    
    // Source 원소 push
    static <P_IN, P_OUT> void copyInto(Sink<P_OUT> wrappedSink,
                                       Spliterator<P_IN> spliterator) {
        if (!StreamOpFlag.SHORT_CIRCUIT.isKnown(wrappedSink.getOpFlags())) {
            spliterator.forEachRemaining(wrappedSink);
        } else {
            // short-circuit 가능하면 cancellationRequested 확인
            for (boolean done = false; !done; ) {
                done = !spliterator.tryAdvance(wrappedSink);
                if (wrappedSink.cancellationRequested()) break;
            }
        }
    }
}

구체적인 Sink 체인 구성 예:

파이프라인 (구성):
  Stream.of(1, 2, 3)
      .filter(x > 1)        // 노드 A
      .map(x * 2)           // 노드 B
      .forEach(println)

Terminal forEach() 호출:
  ↓
Sink 체인 구성 (역방향):
  1. terminalSink = ForEachSink(println)
  2. nodeB.opWrapSink(terminalSink) → MapSink(terminalSink)
  3. nodeA.opWrapSink(mapSink) → FilterSink(mapSink)
  
Sink 체인 (최종):
  FilterSink
    ↓ downstream = MapSink
       ↓ downstream = ForEachSink
          ↓ downstream = null

Source 원소 push (순방향):
  spliterator.forEachRemaining(filterSink)
  
  filterSink.accept(1):
    if (1 > 1) false → skip
  
  filterSink.accept(2):
    if (2 > 1) true → mapSink.accept(2)
      mapSink: 2 * 2 = 4 → forEachSink.accept(4)
        forEachSink: println(4)
  
  filterSink.accept(3):
    if (3 > 1) true → mapSink.accept(3)
      mapSink: 3 * 2 = 6 → forEachSink.accept(6)
        forEachSink: println(6)
```

### 3. Short-circuit 메커니즘

```java
// limit() 같은 short-circuit 연산의 동작

class LimitOp<T> extends StatefulOp<T, T> {
    
    @Override
    public <E> Sink<E> opWrapSink(int flags, Sink<E> sink) {
        return new Sink.ChainedReference<E, E>(sink) {
            long remaining = limit;
            
            @Override
            public void begin(long size) {
                downstream.begin(Math.min(size, limit));
            }
            
            @Override
            public void accept(E t) {
                if (remaining > 0) {
                    remaining--;
                    downstream.accept(t);
                }
            }
            
            @Override
            public void end() {
                downstream.end();
            }
            
            @Override
            public boolean cancellationRequested() {
                // ← 핵심: limit 도달 시 true
                return remaining == 0 || downstream.cancellationRequested();
            }
        };
    }
}

short-circuit 동작:

Stream.iterate(1, x -> x + 1)
    .limit(3)
    .forEach(println)

copyInto() 구현 (short-circuit 버전):
  for (boolean done = false; !done; ) {
    done = !spliterator.tryAdvance(limitSink);
    if (limitSink.cancellationRequested()) {  // ← 조기 종료 신호
      break;
    }
  }

실행:
  limitSink.accept(1) → remaining=2, println(1)
  limitSink.accept(2) → remaining=1, println(2)
  limitSink.accept(3) → remaining=0, println(3)
  (다음) limitSink.cancellationRequested() = true → break
  
무한 Stream 종료 가능!
```

### 4. forEach vs Stream.forEach

```java
// 두 방식의 본질적 동작 비교

// 방식 1: for 루프
for (int x : list) {
    process(x);
}

// 방식 2: Stream.forEach()
list.stream().forEach(this::process);

// 내부 동작:

// for 루프:
Iterator<Integer> iter = list.iterator();
while (iter.hasNext()) {
    int x = iter.next();
    process(x);
}

// Stream.forEach:
Sink<Integer> sink = new Sink<Integer>() {
    public void accept(Integer x) {
        process(x);  // action 실행
    }
};
for (int x : list) {
    sink.accept(x);  // 같은 방식
}

결론: 기본적으로 동일한 메커니즘!

Stream이 제공하는 추가 가치:
  1. 병렬화 가능 (parallel())
  2. 중간 연산 체이닝 (filter, map)
  3. short-circuit 최적화 (limit, findFirst)
  4. 함수형 스타일 가독성
```

---

## 💻 실전 실험

### 실험 1: 각 원소의 실행 순서 추적

```java
public class SinkExecutionOrderTest {
    public static void main(String[] args) {
        System.out.println("=== 각 원소별 모든 연산 통과 ===");
        
        Stream.of(1, 2, 3)
            .peek(x -> System.out.printf("  [1] source: %d%n", x))
            .filter(x -> {
                boolean pass = x > 1;
                System.out.printf("  [2] filter(%d): %s%n", x, pass ? "pass" : "skip");
                return pass;
            })
            .peek(x -> System.out.printf("  [3] after filter: %d%n", x))
            .map(x -> {
                int result = x * 2;
                System.out.printf("  [4] map(%d): %d%n", x, result);
                return result;
            })
            .peek(x -> System.out.printf("  [5] after map: %d%n", x))
            .forEach(x -> System.out.printf("  [6] forEach: %d%n\n", x));
        
        // 출력:
        // [1] source: 1
        // [2] filter(1): skip
        // [1] source: 2
        // [2] filter(2): pass
        // [3] after filter: 2
        // [4] map(2): 4
        // [5] after map: 4
        // [6] forEach: 4
        // [1] source: 3
        // [2] filter(3): pass
        // [3] after filter: 3
        // [4] map(3): 6
        // [5] after map: 6
        // [6] forEach: 6
    }
}
```

### 실험 2: peek() 부작용 위험성

```java
import java.util.*;

public class PeekSideEffectTest {
    public static void main(String[] args) {
        System.out.println("=== peek()로 인한 부작용 ===");
        
        // 안티패턴: peek()로 상태 변경
        Set<Integer> processedInPeek = new HashSet<>();
        
        List<Integer> result = Stream.of(1, 2, 3, 4, 5)
            .filter(x -> x > 2)
            .peek(x -> {
                // 부작용: 외부 상태 변경
                processedInPeek.add(x);
                System.out.println("Peek side effect: " + x);
            })
            .map(x -> x * 2)
            .collect(Collectors.toList());
        
        System.out.println("Result: " + result);
        System.out.println("Peek side effects recorded: " + processedInPeek);
        
        // 문제점:
        // 1. peek() 부작용은 terminal이 있어야만 발생
        // 2. terminal 없으면 peek()도 실행 안 됨
        // 3. 병렬 Stream에서 예측 불가능한 순서
        
        System.out.println("\n=== 올바른 방식 ===");
        Set<Integer> processedCorrect = new HashSet<>();
        List<Integer> result2 = Stream.of(1, 2, 3, 4, 5)
            .filter(x -> x > 2)
            .map(x -> {
                processedCorrect.add(x);  // collect 전에 처리
                return x * 2;
            })
            .collect(Collectors.toList());
        
        System.out.println("Result: " + result2);
        System.out.println("Processed: " + processedCorrect);
    }
}
```

### 실험 3: Short-circuit 동작 확인

```java
public class ShortCircuitTest {
    public static void main(String[] args) {
        System.out.println("=== short-circuit: limit ===");
        
        Stream.iterate(1, x -> x + 1)
            .peek(x -> System.out.println("Generated: " + x))
            .limit(3)
            .forEach(x -> System.out.println("Processed: " + x));
        
        // 출력:
        // Generated: 1
        // Processed: 1
        // Generated: 2
        // Processed: 2
        // Generated: 3
        // Processed: 3
        // (4번째 이상 생성 안 함!)
        
        System.out.println("\n=== short-circuit: findFirst ===");
        
        var result = Stream.iterate(1, x -> x + 1)
            .peek(x -> System.out.println("Checking: " + x))
            .filter(x -> x > 10)
            .findFirst();
        
        System.out.println("Found: " + result);
        // 1~10은 filter skip, 11에서 findFirst 만족 후 즉시 종료
    }
}
```

### 실험 4: forEach vs Stream.forEach 성능

```java
import java.util.*;

public class ForEachPerformanceTest {
    public static void main(String[] args) {
        List<Integer> list = new ArrayList<>();
        for (int i = 0; i < 1_000_000; i++) {
            list.add(i);
        }
        
        // 방식 1: for 루프
        long start = System.nanoTime();
        long sum1 = 0;
        for (int x : list) {
            sum1 += x;
        }
        long elapsed1 = System.nanoTime() - start;
        
        // 방식 2: Stream.forEach (순차)
        start = System.nanoTime();
        long[] sum2 = {0};
        list.stream().forEach(x -> sum2[0] += x);
        long elapsed2 = System.nanoTime() - start;
        
        // 방식 3: Stream.reduce (함수형)
        start = System.nanoTime();
        long sum3 = list.stream().reduce(0L, (a, b) -> a + b);
        long elapsed3 = System.nanoTime() - start;
        
        System.out.printf("For loop: %.2f ms%n", elapsed1 / 1_000_000.0);
        System.out.printf("Stream forEach: %.2f ms%n", elapsed2 / 1_000_000.0);
        System.out.printf("Stream reduce: %.2f ms%n", elapsed3 / 1_000_000.0);
        
        // 결과: 비슷한 성능 (Stream 오버헤드 미미)
    }
}
```

---

## 📊 성능/비교

```
Sink 체인과 Lazy Evaluation의 성능 특성:

상황                    | For 루프    | Stream  | 성능 차이
──────────────────────┼────────────┼─────────┼──────────
단순 순회              | 기준        | ~1.0x   | 동등
필터 + 매핑 (복잡)    | 기준        | ~1.1x   | 미미
무한 Stream + limit   | 불가능      | ✓       | 명확한 이득
병렬화                | 수동 구성   | 자동    | 이득
메모리 (중간 배열 없음)| 기준        | ✓       | 메모리 이득

Lazy Evaluation의 메모리 절감:

eager (즉시 평가):
  Stream.of(1..1M)
      .map(x -> x * 2)      // ← 100만 개 요소의 배열 생성
      .filter(x > threshold)
      .collect(toList())    // ← 필터링된 배열 생성
  → 메모리: O(1M + 필터링된 크기)

lazy (Stream):
  Stream.of(1..1M)
      .map(x -> x * 2)      // ← 아무 배열 생성 안 함
      .filter(x > threshold)
      .collect(toList())    // ← 최종 결과만 생성
  → 메모리: O(필터링된 크기) ✓ 절감

Short-circuit 성능:

무한 Stream에서 첫 10개 짝수:
  Stream.iterate(1, x -> x + 1)
      .filter(x % 2 == 0)
      .limit(10)
      .collect(toList())
  
  처리된 원소: ~20개 (10개 짝수를 찾기 위함)
  중지된 원소: 무한대 - 20 ✓
  → short-circuit 없으면 메모리 폭발
```

---

## ⚖️ 트레이드오프

```
Lazy Evaluation의 트레이드오프:

장점:
  ✓ 무한 Stream 처리 가능
  ✓ 메모리 효율 (중간 배열 없음)
  ✓ short-circuit 최적화 (필요한 것만 처리)
  ✓ 병렬화 기회 (terminal에서 결정)
  ✓ 우아한 함수형 체이닝

단점:
  ✗ 각 원소가 모든 중간 연산을 통과 → CPU 캐시 친화성 낮음
  ✗ Sink 객체 생성 오버헤드 (for 루프보다 느릴 수 있음)
  ✗ peek() 같은 부작용 함수의 유혹
  ✗ 디버깅 어려움 (실행 순서 파악 필수)
  ✗ Terminal 없으면 아무 일도 일어나지 않음

For 루프의 이점:
  - 각 중간 단계의 배열 크기 최적화 가능
  - CPU 캐시 친화적 (배열 순차 접근)
  - 디버깅 쉬움 (각 단계 명시적)

Stream의 이점:
  - 병렬화 (parallel())
  - 함수형 스타일
  - 복합 연산 가독성
  - 무한 Stream 지원

선택 기준:
  For 루프: 단순 순회, 성능 극대화, 디버깅 필요
  Stream: 복합 연산, 병렬화 필요, 무한 데이터
```

---

## 📌 핵심 정리

```
Lazy Evaluation 메커니즘:

1. Terminal 호출 시작
   forEach(), collect(), reduce() 등

2. Sink 체인 생성 (역방향)
   각 중간 연산 노드가 Sink를 생성/감싸기
   마지막 Sink = Terminal Sink

3. 최종 Sink 체인 (역순)
   Terminal Sink
     ← 마지막 중간 연산 Sink
       ← 이전 중간 연산 Sink
         ← ...
           ← Head Sink

4. Source 원소 Push (순방향)
   Spliterator.forEachRemaining(sink)
   각 원소가 모든 중간 연산을 한 번에 통과

5. Short-circuit 구현
   Sink.cancellationRequested()
   true 반환 시 source 중단
   limit(), findFirst() 등 구현

핵심 이해:
  - 각 원소가 "수직으로" 모든 연산 통과
  - "수평으로" 배열을 생성하지 않음
  - Terminal 없으면 아무 일도 일어나지 않음
  - peek()는 디버깅 전용
  - forEach 루프와 본질 동일
```

---

## 🤔 생각해볼 문제

**Q1.** `Stream.of(1, 2, 3).map(x -> x * 2).filter(x > 2).forEach(println)` 파이프라인에서 "2"가 출력되지 않는 이유는?

<details>
<summary>해설 보기</summary>

각 원소별 흐름:

```
원소 1:
  map(1) → 1 * 2 = 2
  filter(2 > 2) → false (필터링됨)
  forEach 도달 안 함

원소 2:
  map(2) → 2 * 2 = 4
  filter(4 > 2) → true (통과)
  forEach(4) → 출력

원소 3:
  map(3) → 3 * 2 = 6
  filter(6 > 2) → true (통과)
  forEach(6) → 출력
```

즉, 필터 조건 `x > 2`는 **매핑된 값 4, 6**에 대해 적용되므로 매핑된 2는 필터링되어 출력되지 않는다. 

만약 원래 값에 대해 필터링하려면:
```java
Stream.of(1, 2, 3)
    .filter(x -> x > 2)    // 원래 값 기준 필터
    .map(x -> x * 2)       // 그 다음 매핑
    .forEach(System.out::println);  // 3*2=6만 출력
```

</details>

---

**Q2.** Terminal 없이 다음 코드를 실행하면 어떻게 되는가?

```java
Stream<Integer> stream = Stream.of(1, 2, 3)
    .peek(x -> System.out.println("peek: " + x))
    .map(x -> x * 2)
    .filter(x -> x > 2);
// 여기서 끝남
```

<details>
<summary>해설 보기</summary>

**아무 출력이 없다.**

이유:
1. `Stream.of()` → Head 노드 생성
2. `peek()` → Peek Sink 노드 추가 (아직 평가 안 함)
3. `map()` → Map Sink 노드 추가 (아직 평가 안 함)
4. `filter()` → Filter Sink 노드 추가 (아직 평가 안 함)
5. 코드 끝 → Terminal 호출 안 함

따라서:
- 파이프라인 구조만 메모리에 존재
- peek() 함수 실행 안 함
- 원소 처리 안 함
- "peek: " 문자열 출력 안 함

**핵심: Terminal이 평가의 신호!**

만약 마지막에 terminal을 추가하면:
```java
stream.collect(Collectors.toList());  // 이제 평가 시작!
// "peek: 1", "peek: 2", "peek: 3" 출력됨
```

이것이 lazy evaluation이다.

</details>

---

**Q3.** 다음 두 코드의 성능 차이는?

```java
// 코드 A: for 루프로 필터링 후 매핑
List<Integer> result = new ArrayList<>();
for (int x : list) {
    if (x > 100) {
        result.add(x * 2);
    }
}

// 코드 B: Stream으로 필터 후 매핑
List<Integer> result = list.stream()
    .filter(x -> x > 100)
    .map(x -> x * 2)
    .collect(Collectors.toList());
```

<details>
<summary>해설 보기</summary>

**이론상 코드 A가 약간 더 빠르지만, 실무에서는 무시할 수준.**

이유:

코드 A (for 루프):
- Iterator 생성
- 각 원소 1회 검사 (if)
- 통과한 것만 list.add()
- O(N) 순회

코드 B (Stream):
- Stream 객체 생성
- filter Sink, map Sink 객체 생성 (오버헤드)
- Spliterator 생성
- 각 원소가 2개 Sink를 통과 (약간의 간접 호출)
- collect(toList()) → ArrayList 생성
- O(N) 순회

성능 차이:
- 작은 리스트 (< 10K): for 루프가 5~10% 빠를 수 있음
- 중간 리스트 (10K ~ 1M): 거의 동등 (1% 이내)
- 병렬화: Stream이 쉬움 (`.parallel()` 추가)

**선택 기준:**
- 성능 극대화: for 루프
- 가독성 + 유지보수: Stream
- 병렬화 가능성: Stream
- 무한 데이터: Stream 필수

실무에서 성능 테스트:
```
For 루프: ~100ms
Stream: ~102ms
차이: 2% (무시할 수 있는 수준)
```

따라서 "Stream이 느리다"는 것은 대부분 오해이며, 실제로는 가독성을 얻으며 성능 대가는 미미하다.

</details>

---

<div align="center">

**[⬅️ 이전: ReferencePipeline 소스코드 추적](./02-reference-pipeline-source.md)** | **[홈으로 🏠](../README.md)** | **[다음: Spliterator ➡️](./04-spliterator-deep-dive.md)**

</div>
