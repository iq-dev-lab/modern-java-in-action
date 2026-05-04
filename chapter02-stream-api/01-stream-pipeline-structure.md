# Stream Pipeline 3단계 구조 — Source / Intermediate / Terminal

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Stream의 모든 파이프라인이 Source → Intermediate(0개 이상) → Terminal의 3단계 구조를 따르는가?
- 중간 연산이 즉시 실행되지 않고 "연결"만 되는 이유는 무엇인가?
- Lazy evaluation의 본질은 무엇이며, 이것이 무한 Stream을 가능하게 하는가?
- Stream 1회용 원칙이 존재하는 이유와 `IllegalStateException` 발생 메커니즘은?
- Terminal 연산이 평가를 시작하도록 설계한 이유는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Stream의 3단계 구조와 lazy evaluation을 이해하지 못하면, 왜 `map()` 후 즉시 값이 변하지 않는지, 왜 `peek()`이 부작용을 일으키는지, 왜 같은 Stream을 두 번 사용하면 예외가 발생하는지 혼란스럽다. 이 개념이 JDK 내부에서 어떻게 구현되었는지 이해하면, Stream을 올바르게 설계하고 성능 이슈를 디버깅할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 중간 연산 후 즉시 평가된다고 가정
  Stream.of(1, 2, 3)
      .map(x -> { System.out.println("map: " + x); return x * 2; })
      .map(x -> { System.out.println("map2: " + x); return x + 1; });
  
  예상: "map: 1", "map2: 2", "map: 2", "map2: 4", ...
  실제: 아무 출력 없음!
  → 이유: terminal 연산이 없어서 파이프라인이 구성만 됨

실수 2: Stream을 여러 번 소비 시도
  Stream<Integer> stream = Stream.of(1, 2, 3);
  stream.forEach(System.out::println);  // 1, 2, 3
  stream.forEach(System.out::println);  // 예외: IllegalStateException
  → 이유: Stream 1회용 원칙 위반

실수 3: peek()를 성능 최적화 목적으로 사용
  Stream.of(1, 2, 3)
      .peek(System.out::println)  // 부작용 발생
      .filter(x -> x > 1)
      .collect(toList());
  → peek()는 디버깅 목적일 뿐, 프로덕션 코드에서 부작용 초래

실수 4: 중간 연산이 체이닝 가능하다는 착각
  Stream<Integer> s = Stream.of(1, 2, 3);
  Stream<Integer> mapped = s.map(x -> x * 2);  // 새로운 Stream?
  mapped.forEach(System.out::println);  // 이전 stream s는 여전히 미평가
  → 혼란: mapped는 새 Stream이 아니라 파이프라인 노드
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 올바른 Stream 파이프라인 구성

// 패턴 1: 명확한 3단계 구조
Stream<Integer> result = Stream.of(1, 2, 3, 4, 5)  // Source
    .filter(x -> x > 2)                             // Intermediate 1
    .map(x -> x * 2)                                // Intermediate 2
    .collect(Collectors.toList());                  // Terminal
// Source, 중간 연산(2개), Terminal 명확함

// 패턴 2: Lazy evaluation 활용 (무한 Stream)
Stream.iterate(1, x -> x + 1)  // 무한 Source
    .filter(x -> x % 2 == 0)    // 중간 연산 (아직 미평가)
    .limit(5)                   // short-circuit (평가 신호)
    .forEach(System.out::println);  // Terminal (처음 평가 시작)
// limit(5) → short-circuit 신호, forEach → 처음 5개 짝수만 평가

// 패턴 3: peek()는 디버깅 용도로만
Stream.of(1, 2, 3)
    .filter(x -> x > 1)
    .peek(System.out::println)  // "2", "3" 출력 (디버깅)
    .map(x -> x * 2)
    .forEach(System.out::println);  // "4", "6" 출력

// 패턴 4: 중간 연산 체이닝의 이해
Stream<Integer> source = Stream.of(1, 2, 3);  // Source (1회용)
// source = source.map(...);  // 불가능! Stream 1회용
// 대신 새로운 변수에 전체 파이프라인 저장
Stream<Integer> processed = Stream.of(1, 2, 3)
    .map(x -> x * 2)
    .filter(x -> x > 2);  // 중간 연산만 → 아직 미평가

// processed는 terminal 없으면 평가 안 됨
List<Integer> list = processed.collect(Collectors.toList());  // 이제 평가!

// 패턴 5: 불변 설정과 여러 terminal 원하는 경우
Stream<Integer> src = Stream.of(1, 2, 3);
// src를 여러 번 사용하려면 List로 먼저 구체화
List<Integer> data = List.of(1, 2, 3);
long count = data.stream().count();  // Terminal 1
int sum = data.stream().mapToInt(x -> x).sum();  // Terminal 2 (새로운 Stream)
```

---

## 🔬 내부 동작 원리

### 1. Stream의 3단계 구조

```
모든 Stream 파이프라인의 구조:

┌─────────────────────────────────────────────────────┐
│  Source (소스 단계)                                   │
│  - Stream.of(1, 2, 3)                               │
│  - list.stream()                                    │
│  - Stream.generate(), Stream.iterate()              │
│  - stream = new ReferencePipeline.Head(...)         │
└─────────────────────────┬───────────────────────────┘
                          │ (파이프라인 연결)
┌─────────────────────────▼───────────────────────────┐
│  Intermediate Operations (중간 연산, 0개 이상)        │
│  - map(x -> x * 2)          → StatelessOp 노드       │
│  - filter(x -> x > 1)       → StatelessOp 노드       │
│  - sorted()                 → StatefulOp 노드        │
│  - distinct()               → StatefulOp 노드        │
│  각 중간 연산 = 파이프라인에 노드 추가 (미평가!)       │
└─────────────────────────┬───────────────────────────┘
                          │ (파이프라인 연결)
┌─────────────────────────▼───────────────────────────┐
│  Terminal Operation (터미널 연산, 정확히 1개)         │
│  - forEach(action)          → 모든 원소 순회         │
│  - collect(collector)       → 수집               │
│  - reduce(accumulator)      → 집계               │
│  - findFirst(), findAny()   → 선택적 종료           │
│  - count(), sum()           → 최종값 반환            │
│  Terminal 호출 = 파이프라인 평가 시작!               │
└─────────────────────────────────────────────────────┘

핵심: Source → Intermediate → Terminal
     (구성)   (연결만 함)    (평가 시작)
```

### 2. Lazy Evaluation 메커니즘

```java
// OpenJDK 소스: java.util.stream.ReferencePipeline

abstract class AbstractPipeline<E_IN, E_OUT, S extends BaseStream<E_OUT, S>>
    extends PipelineHelper<E_OUT> {
    
    // 이전 단계의 파이프라인
    private final AbstractPipeline sourceStage;
    private final AbstractPipeline previousStage;
    
    // 중간 연산은 새로운 파이프라인 노드를 반환 (평가 안 함!)
    public final <R> Stream<R> map(Function<? super P_OUT, ? extends R> mapper) {
        return new StatelessOp<P_OUT, R>(this, StreamShape.REFERENCE,
                                         StreamOpFlag.NOT_SORTED | ...,
                                         mapper) {
            @Override
            Sink<P_OUT> opWrapSink(int flags, Sink<R> sink) {
                return new Sink.ChainedReference<P_OUT, R>(sink) {
                    @Override
                    public void accept(P_OUT u) {
                        downstream.accept(mapper.apply(u));
                    }
                };
            }
        };
        // return 시점: map 함수 실행 안 됨! (노드만 추가)
    }
    
    // Terminal 연산은 파이프라인 평가를 시작
    public void forEach(Consumer<? super E_OUT> action) {
        evaluate(TerminalOp.makeForEach(action));  // ← 평가 시작!
    }
    
    private <R> R evaluate(TerminalOp<E_OUT, R> terminalOp) {
        // 1. sourceStage에서부터 역방향으로 Sink 체인 생성
        // 2. source에서 원소를 하나씩 push
        // 3. 모든 중간 연산이 한 번의 패스로 실행
    }
}

Lazy Evaluation 흐름:

Step 1: 파이프라인 구성 (평가 X)
  Stream.of(1, 2, 3)
      .map(x -> x * 2)      ← 노드 추가, 함수 미호출
      .filter(x -> x > 2)   ← 노드 추가, 함수 미호출

Step 2: Terminal 호출 (평가 시작)
  .forEach(System.out::println)  ← evaluate() 시작

Step 3: 역방향 Sink 체인 생성
  forEach(Sink) 
    → filter(Sink)
      → map(Sink)
        → source(Sink)

Step 4: Source에서 원소를 하나씩 push (순방향 흐름)
  source[0]: 1 → map(1*2=2) → filter(2>2? no) → skip
  source[1]: 2 → map(2*2=4) → filter(4>2? yes) → println(4)
  source[2]: 3 → map(3*2=6) → filter(6>2? yes) → println(6)
```

### 3. Stream 1회용 원칙

```java
// OpenJDK 소스: java.util.stream.BaseStream

public interface BaseStream<T, S extends BaseStream<T, S>>
    extends AutoCloseable {
    
    S filter(Predicate<? super T> predicate);
    // filter()는 새로운 Stream을 반환하지만...
    // 내부적으로는 파이프라인 노드를 추가한 것
    
    void forEach(Consumer<? super T> action);
    // forEach() 호출 = BaseStream 상태 변경!
}

// 실제 구현 (단순화):
class Stream<T> implements BaseStream<T, Stream<T>> {
    private boolean consumed = false;  // 핵심: 1회용 플래그
    
    public void forEach(Consumer<? super T> action) {
        if (consumed) {
            throw new IllegalStateException("Stream has already been operated upon or closed");
        }
        consumed = true;  // ← 평가 후 "사용됨"으로 표시
        evaluate(...);
    }
    
    public void forEachOrdered(Consumer<? super T> action) {
        if (consumed) {
            throw new IllegalStateException(...);
        }
        consumed = true;
        evaluate(...);
    }
}

// Terminal 연산들도 모두 consumed 체크:
boolean anyMatch(Predicate<? super T> predicate);
Optional<T> findFirst();
T reduce(T identity, BinaryOperator<T> accumulator);
// 모두 호출 시 consumed = true 설정

Stream<Integer> stream = Stream.of(1, 2, 3);
stream.forEach(System.out::println);      // consumed = true 설정
stream.forEach(System.out::println);      // ← 예외: "already operated upon"
```

### 4. Terminal 연산의 역할

```
Terminal 연산이 평가를 시작하는 이유:

1. 무한 Stream을 다룰 수 있음
  Stream.iterate(1, x -> x + 1)  // 무한 소스 (평가 안 할 때는 안 전개)
      .limit(5)                   // short-circuit (5개만)
      .forEach(System.out::println);

2. 필요한 만큼만 평가 (효율성)
  Stream.of(1, 2, 3, 4, 5)
      .filter(x -> x > 2)
      .findFirst();               // 첫 번째만 필요 → 나머지 평가 안 함

3. 선택적 최적화 가능
  - Stateless 연산만 → 병렬 처리 가능
  - Stateful 연산 포함 → 순차 처리 필요
  - Terminal이 호출될 때 전체 파이프라인 분석 후 최적화

Terminal 호출 없으면:
  Stream.of(1, 2, 3).map(x -> x * 2);  // 의미 없음!
  // map 함수가 아예 실행 안 됨
  // 부작용도 없고, 값도 나오지 않음
```

---

## 💻 실전 실험

### 실험 1: Lazy Evaluation 확인

```java
import java.util.stream.*;

public class LazyEvaluationTest {
    public static void main(String[] args) {
        System.out.println("=== 중간 연산만 (평가 X) ===");
        Stream.of(1, 2, 3)
            .peek(x -> System.out.println("peek: " + x))
            .map(x -> {
                System.out.println("map: " + x);
                return x * 2;
            })
            .filter(x -> {
                System.out.println("filter: " + x);
                return x > 2;
            });
        // 출력: 아무 것도 없음! (평가 안 됨)

        System.out.println("\n=== Terminal 추가 (평가 O) ===");
        Stream.of(1, 2, 3)
            .peek(x -> System.out.println("peek: " + x))
            .map(x -> {
                System.out.println("map: " + x);
                return x * 2;
            })
            .filter(x -> {
                System.out.println("filter: " + x);
                return x > 2;
            })
            .forEach(x -> System.out.println("result: " + x));
        // 출력:
        // peek: 1
        // map: 1
        // filter: 2
        // (skip because 2 > 2 is false)
        // peek: 2
        // map: 2
        // filter: 4
        // result: 4
        // peek: 3
        // map: 3
        // filter: 6
        // result: 6
    }
}
```

### 실험 2: Stream 1회용 원칙

```java
import java.util.stream.*;

public class StreamOneTimeTest {
    public static void main(String[] args) {
        Stream<Integer> stream = Stream.of(1, 2, 3);
        
        // 첫 번째 terminal 호출
        System.out.println("First forEach:");
        stream.forEach(System.out::println);  // 1, 2, 3
        
        // 두 번째 terminal 호출
        System.out.println("Second forEach:");
        try {
            stream.forEach(System.out::println);  // IllegalStateException!
        } catch (IllegalStateException e) {
            System.out.println("Exception: " + e.getMessage());
            // "Stream has already been operated upon or closed"
        }
        
        // 해결책: 새 Stream 생성
        System.out.println("\nWith new Stream:");
        List<Integer> list = List.of(1, 2, 3);
        list.stream().forEach(System.out::println);  // OK
        list.stream().forEach(System.out::println);  // OK (새 Stream)
    }
}
```

### 실험 3: 무한 Stream과 short-circuit

```java
import java.util.stream.*;

public class InfiniteStreamTest {
    public static void main(String[] args) {
        System.out.println("=== 무한 Stream (terminal이 short-circuit) ===");
        
        // 무한 Stream이지만 limit()과 findFirst()로 종료
        Stream.iterate(1, x -> x + 1)
            .peek(x -> System.out.println("Generated: " + x))
            .filter(x -> x % 2 == 0)
            .limit(3)  // short-circuit: 3개만
            .forEach(x -> System.out.println("Result: " + x));
        
        System.out.println("\n=== findFirst()로 조기 종료 ===");
        var result = Stream.iterate(1, x -> x + 1)
            .filter(x -> {
                System.out.println("Checking: " + x);
                return x > 10;
            })
            .findFirst();  // 첫 번째 찾으면 즉시 종료
        System.out.println("Found: " + result);
    }
}
```

### 실험 4: javap로 ByteCode 확인

```bash
# LazyEvaluationTest.java 컴파일
javac LazyEvaluationTest.java

# ByteCode 확인
javap -c LazyEvaluationTest | grep -A 20 "public static void main"

# 출력에서 확인할 사항:
# 1. map() 호출 시 함수가 즉시 로드되지 않음
# 2. forEach() 호출이 evaluate() 메서드 호출로 변환
# 3. 중간 연산은 StatelessOp/StatefulOp 객체 생성만 함
```

---

## 📊 성능/비교

```
Lazy Evaluation의 성능 영향:

상황                          | 즉시 평가    | Lazy (Stream)
──────────────────────────────┼──────────────┼──────────────
무한 컬렉션 처리               | 메모리 폭발   | OK (유한 terminal)
필요한 것만 처리              | 모두 처리     | ✓ short-circuit
여러 중간 연산 체이닝          | 중간 배열 생성| ✓ 한 번의 패스
메모리 오버헤드                | 높음         | 낮음 (1회용)

예제: 1,000,000개 정수에서 처음 10개 짝수 찾기

즉시 평가 (for 루프):
  for (int i = 1; i <= 1000000; i++) {
      if (i % 2 == 0) {
          process(i);
          if (++count == 10) break;  // 조기 종료
      }
  }
  → ~20개 반복 (10개 짝수)

Stream (Lazy):
  Stream.iterate(1, x -> x + 1)
      .filter(x -> x % 2 == 0)
      .limit(10)
      .forEach(this::process);
  → ~20번 내부 호출 (lazy + short-circuit)

메모리 사용량 비교:
  Arrays.stream(largeArray)
      .map(x -> x * 2)           // 임시 배열 생성 X (lazy)
      .filter(x -> x > threshold)
      .collect(toList());        // 최종 컬렉션만 생성
  → 메모리: O(결과 크기)
  
  vs
  
  List<Integer> temp1 = new ArrayList<>();
  for (int x : largeArray) {
      temp1.add(x * 2);          // 임시 배열 (메모리 낭비)
  }
  List<Integer> result = new ArrayList<>();
  for (int x : temp1) {
      if (x > threshold) {
          result.add(x);
      }
  }
  → 메모리: O(array + 임시 배열)
```

---

## ⚖️ 트레이드오프

```
Lazy Evaluation의 트레이드오프:

장점:
  ✓ 무한 Stream 처리 가능
  ✓ 메모리 효율 (1회 패스)
  ✓ Short-circuit 최적화
  ✓ 함수형 인터페이스로 우아한 체이닝
  ✓ 병렬 처리 최적화 (terminal에서 결정)

단점:
  ✗ Terminal 없이는 평가 안 됨 (실수하기 쉬움)
  ✗ Stream 1회용 원칙 (새로운 학습 곡선)
  ✗ peek() 같은 부작용 가능성
  ✗ 디버깅 어려움 (중간 값을 콘솔에 찍기 힘듦)
  ✗ 예외 처리 복잡 (파이프라인 중간에서 예외 발생)

구성(Composability) vs 즉시 실행(Eagerness):
  Lazy: 각 단계가 느슨하게 결합 → 최적화 가능
  Eager: 각 단계가 즉시 완료 → 디버깅 쉬움

선택 기준:
  Stream 사용: 대용량, 무한, 복합 연산
  for 루프 사용: 간단한 순회, 디버깅 필요, 성능 극대화
```

---

## 📌 핵심 정리

```
Stream Pipeline 3단계 구조:

1. Source (소스 단계)
   - Stream.of(), list.stream(), Stream.generate()
   - 원소의 출발점

2. Intermediate Operations (중간 연산, 0개 이상)
   - map(), filter(), sorted(), distinct()
   - 호출 시 노드만 추가, 함수 미실행
   - Stateless: map(), filter() → 한 원소씩 처리
   - Stateful: sorted(), distinct() → 전체 필요

3. Terminal Operation (터미널 연산, 정확히 1개)
   - forEach(), collect(), reduce(), findFirst()
   - 호출 시 평가 시작 (역방향 Sink 체인 → 순방향 원소 push)

Lazy Evaluation:
  - 중간 연산은 "구조"만 구성, 실행 안 함
  - Terminal이 호출되어야 처음 평가 시작
  - 무한 Stream + short-circuit으로 유한하게 처리

Stream 1회용 원칙:
  - Terminal 호출 시 stream 상태 변경
  - 두 번 사용 시 IllegalStateException
  - 여러 terminal 필요 → 여러 Stream 생성

핵심 메커니즘:
  Terminal 호출 → 역방향 Sink 체인 구성 → Source push →
  모든 중간 연산이 한 번의 패스로 실행 → 최종 결과
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 "map: 2", "filter: 2"가 출력되지 않는 이유는 무엇인가?

```java
Stream.of(1, 2, 3)
    .peek(x -> System.out.println("peek: " + x))
    .map(x -> { System.out.println("map: " + x); return x * 2; })
    .filter(x -> { System.out.println("filter: " + x); return x > 2; })
    .forEach(x -> System.out.println("result: " + x));

// 실제 출력:
// peek: 1
// map: 1
// filter: 2
// peek: 2
// map: 2
// filter: 4
// result: 4
// ...
```

<details>
<summary>해설 보기</summary>

각 원소는 모든 중간 연산을 한 번의 패스로 거친다. 원소 1의 경우:
- peek(1) → "peek: 1" 출력
- map(1) → "map: 1" 출력, 결과 2 반환
- filter(2 > 2? false) → "filter: 2" 출력 후 필터링됨
- forEach로 가지 않음 (필터링됨)

다음 원소 2로 이동:
- peek(2) → "peek: 2" 출력
- map(2) → "map: 2" 출력, 결과 4 반환
- filter(4 > 2? true) → "filter: 4" 출력, 통과
- forEach(4) → "result: 4" 출력

즉, 모든 중간 연산이 "한 원소씩 순차 처리"되며, 필터링된 원소는 다음 연산으로 전달되지 않는다. 이것이 lazy evaluation의 핵심: 불필요한 처리를 건너뜀.

</details>

---

**Q2.** `Stream.of(1, 2, 3).map(...)` 호출 후 반환된 것이 "새 Stream"인가, 아니면 "기존 Stream 수정"인가?

<details>
<summary>해설 보기</summary>

반환된 것은 기술적으로는 새로운 `Stream` 객체이지만, 내부적으로는 "파이프라인 노드를 추가한" 것이다.

```java
Stream<Integer> stream1 = Stream.of(1, 2, 3);  // Head 노드
Stream<Integer> stream2 = stream1.map(x -> x * 2);  // StatelessOp 노드
```

`stream2`는 새 객체이지만 `stream1`과 연결되어 있다. `stream2`에 terminal을 호출하면 전체 파이프라인이 평가된다.

중요한 점:
- `stream1`은 더 이상 사용할 수 없다 (consumed 상태가 아니지만, 연결됨)
- `stream1`을 다시 사용하려면 새 중간 연산을 해야 함 (불가능)
- 실무에서는 fluent 스타일로 연쇄 호출하므로 stream1 변수 자체는 사용 안 함

따라서 "새 Stream"이라고 생각하되, 내부적으로는 "파이프라인 노드 추가"라고 이해하는 것이 정확하다.

</details>

---

**Q3.** `peek()`가 디버깅 목적이 아니라면, 왜 Stream API에 포함되었는가?

<details>
<summary>해설 보기</summary>

`peek()`는 다음 두 목적으로 설계되었다:

1. **디버깅 (공식 목적)**
   ```java
   Stream.of(1, 2, 3)
       .map(x -> x * 2)
       .peek(x -> System.out.println("After map: " + x))
       .filter(x -> x > 2)
       .forEach(System.out::println);
   ```

2. **부작용 발생 (권장하지 않음)**
   ```java
   // 안티패턴: 로깅 등의 부작용
   stream.peek(x -> logger.info("Processing: " + x))
   ```

Java 8 설계자들은 Stream이 "순수 함수형"이기를 원했지만, 실무에서 디버깅의 필요성을 인정했다. 그 결과 `peek()`가 추가됨.

주의할 점:
- `peek()`는 "작업"을 수행하지 않는다 (terminal 아님)
- `peek()`만 있고 terminal이 없으면 부작용도 발생하지 않음
- 프로덕션 코드에서 부작용 있는 `peek()`는 안티패턴

더 나은 대안:
- 필터링/변환 후 `collect()`로 결과 수집
- 필요시 반복 후 별도 처리: `collect(Collectors.toList()).forEach(...)`

</details>

---

<div align="center">

**[⬅️ 이전 챕터: Functional Interface 5가지](../chapter01-lambda-internals/06-functional-interface-categories.md)** | **[홈으로 🏠](../README.md)** | **[다음: ReferencePipeline 소스코드 추적 ➡️](./02-reference-pipeline-source.md)**

</div>
