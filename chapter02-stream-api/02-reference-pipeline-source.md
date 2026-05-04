# ReferencePipeline 소스코드 추적 — Head / StatelessOp / StatefulOp

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `AbstractPipeline`의 `previousStage`/`sourceStage`/`sourceOrOpFlags` 3개 필드가 어떻게 파이프라인을 구성하는가?
- `Head` 노드(소스)와 `StatelessOp`(map, filter), `StatefulOp`(sorted, distinct)의 계층 구조는?
- 파이프라인 노드들이 "연결"되는 메커니즘은 무엇인가?
- OpenJDK 소스에서 중간 연산 호출 시 새 노드가 생성되는 방식은?
- `StreamOpFlag`가 최적화에 미치는 영향은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Stream을 "검은 상자"로 사용해도 되지만, 복잡한 파이프라인에서 성능이 저하되면 원인을 파악해야 한다. OpenJDK의 파이프라인 구조를 이해하면, stateful 연산의 위치를 최적화하거나, 불필요한 중간 복사를 피할 수 있다. 또한 커스텀 Sink를 작성할 때도 파이프라인 구조 이해가 필수다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: map() 호출 시마다 새로운 배열이 생성된다고 착각
  Stream.of(1, 2, 3)
      .map(x -> x * 2)      // 임시 배열 [2, 4, 6] 생성?
      .map(x -> x + 1)      // 임시 배열 [3, 5, 7] 생성?
      .collect(toList());
  → 혼란: map()은 배열을 만들지 않음, 파이프라인 노드만 추가!

실수 2: 파이프라인 노드의 계층 구조를 모름
  sorted()를 사용하면 자동으로 성능 최적화?
  → 아니다! sorted()는 StatefulOp로 모든 데이터를 메모리에 로드
  
  sorted() 후 map() 순서:
    Stream.of(5, 1, 3, 2, 4)
        .sorted()           // 전체 정렬 (O(N log N))
        .map(x -> x * 2)    // 5개만 처리
  
  vs
  
  map() 후 sorted() 순서:
    Stream.of(5, 1, 3, 2, 4)
        .map(x -> x * 2)    // 5개 처리
        .sorted()           // 5개만 정렬

실수 3: StreamOpFlag를 모르고 임의로 파이프라인 구성
  "모든 파이프라인이 같은 성능"이라고 가정
  → 실제로는 SORTED, DISTINCT, SIZED 플래그가 다음 연산 최적화
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 올바른 Stream 파이프라인 설계

// 패턴 1: 파이프라인 노드 계층 이해
Stream<Integer> stream = Stream.of(1, 2, 3, 4, 5)  // Head 노드
    .filter(x -> x > 2)                            // StatelessOp 노드
    .map(x -> x * 2)                               // StatelessOp 노드
    .distinct()                                    // StatefulOp 노드
    .collect(Collectors.toList());                 // Terminal

// 내부 구조:
//   Terminal Sink
//      ↑
//   StatefulOp(distinct) Sink
//      ↑
//   StatelessOp(map) Sink
//      ↑
//   StatelessOp(filter) Sink
//      ↑
//   Head(source) Sink
//      ↑
//   Source (원소 push)

// 패턴 2: 필터링 → 매핑 순서 (최적화)
long count = Stream.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    .filter(x -> x % 2 == 0)      // 짝수만 (5개로 축소)
    .map(x -> {                   // 축소된 5개에만 적용
        System.out.println("Processing: " + x);
        return x * 2;
    })
    .count();
// "Processing"은 5번만 출력 (10번이 아님)

// 패턴 3: Stateful 연산은 마지막에
Stream.of(1, 2, 3, 4, 5)
    .filter(x -> x > 1)           // 필터링 먼저 (축소)
    .distinct()                   // 축소된 데이터만 distinct
    .sorted()                     // 축소된 데이터만 정렬
    .collect(Collectors.toList());

// 패턴 4: StreamOpFlag 활용 (고급)
// sorted() 후 stream은 SORTED 플래그 가짐
// → distinct()가 더 효율적으로 동작 (중복 제거 시 인접 비교만)
// → 다음 sorted() 호출 시 이미 정렬됨 → 스킵 가능

// 패턴 5: 큰 컬렉션에서 limit() 활용
Stream.iterate(1, x -> x + 1)
    .filter(x -> x % 2 == 0)
    .limit(100)                   // short-circuit (100개만)
    .collect(Collectors.toList());
// limit() = short-circuit Stateful 연산
// → 필요한 데이터만 평가
```

---

## 🔬 내부 동작 원리

### 1. AbstractPipeline과 파이프라인 노드 구조

```java
// OpenJDK: java.util.stream.AbstractPipeline

abstract class AbstractPipeline<E_IN, E_OUT, S extends BaseStream<E_OUT, S>>
    extends PipelineHelper<E_OUT> implements BaseStream<E_OUT, S> {
    
    // 파이프라인 링크드 리스트 구조를 위한 필드들:
    private final AbstractPipeline sourceStage;  // 첫 번째 Head 노드 (모든 노드가 참조)
    private final AbstractPipeline previousStage;  // 이전 노드 (역방향 추적)
    private int sourceOrOpFlags;  // StreamOpFlag (SORTED, DISTINCT 등)
    private int combinedFlags;    // 누적 플래그
    private boolean lefthand;     // 병렬 처리 방향 플래그
    
    protected AbstractPipeline(AbstractPipeline previousStage, int opFlags) {
        this.previousStage = previousStage;
        this.sourceStage = previousStage.sourceStage;  // 첫 노드는 그대로 전파
        this.sourceOrOpFlags = opFlags;
        this.combinedFlags = previousStage.combinedFlags | opFlags;
    }
    
    // 중간 연산: 새 노드를 반환 (대사 패턴)
    public final <R> Stream<R> map(Function<? super E_OUT, ? extends R> mapper) {
        return new StatelessOp<E_OUT, R>(this, StreamShape.REFERENCE,
                                         StreamOpFlag.NOT_SORTED |
                                         StreamOpFlag.NOT_DISTINCT,
                                         mapper) {
            // 구체적인 Sink 구현
            @Override
            Sink<E_OUT> opWrapSink(int flags, Sink<R> downstream) {
                return new Sink.ChainedReference<E_OUT, R>(downstream) {
                    @Override
                    public void accept(E_OUT u) {
                        downstream.accept(mapper.apply(u));  // 매핑 후 전달
                    }
                };
            }
        };
    }
}

파이프라인 링크드 리스트 구조:

sourceStage (Head)
    ↓ previousStage 포인터로 연결 (역방향)
    ↓
StatelessOp(filter) ← sourceStage (같은 Head)
    ↓
StatelessOp(map) ← sourceStage (같은 Head)
    ↓
StatefulOp(sorted) ← sourceStage (같은 Head)
    ↓
Terminal Sink

모든 노드가 sourceStage (처음 Head)를 참조
→ terminal에서 sourceStage.wrapSink() 호출하면
→ 역방향 포인터로 모든 노드를 추적
→ 역방향 Sink 체인 생성
```

### 2. Head / StatelessOp / StatefulOp 계층

```java
// OpenJDK 구조 (단순화)

// === Head: 소스 노드 ===
static final class Head<E_IN, E_OUT> extends ReferencePipeline<E_IN, E_OUT> {
    private final Supplier<? extends Spliterator<E_OUT>> spliteratorSupplier;
    
    Head(Supplier<? extends Spliterator<E_OUT>> spliteratorSupplier, ...) {
        super(null, StreamOpFlag.HEAD, ...) ;
    }
    
    @Override
    Sink<E_OUT> opWrapSink(int flags, Sink<E_OUT> sink) {
        return sink;  // 변환 없음, 그대로 전달
    }
}

// === StatelessOp: 상태 없는 중간 연산 (map, filter) ===
abstract static class StatelessOp<E_IN, E_OUT>
    extends ReferencePipeline<E_IN, E_OUT> {
    
    StatelessOp(AbstractPipeline upstream, StreamShape inputShape, int opFlags) {
        super(upstream, opFlags);
        // previousStage = upstream
        // sourceStage = upstream.sourceStage
    }
    
    // 구체화:
    // - StatelessOp에서 map 구현
    // - StatelessOp에서 filter 구현
    // - StatelessOp에서 flatMap 구현 등
}

// === StatefulOp: 상태를 유지하는 중간 연산 (sorted, distinct) ===
abstract static class StatefulOp<E_IN, E_OUT>
    extends ReferencePipeline<E_IN, E_OUT> {
    
    StatefulOp(AbstractPipeline upstream, StreamShape inputShape,
               int opFlags) {
        super(upstream, opFlags | StreamOpFlag.IS_STATEFUL);
    }
    
    // 구체화:
    // - SortedOp: 전체 데이터를 배열로 수집 → 정렬
    // - DistinctOp: Set으로 중복 추적 → 새로운 원소만 전달
}

차이점:

               | Stateless (map, filter) | Stateful (sorted, distinct)
───────────────┼────────────────────────┼─────────────────────────────
메모리 요구     | O(1)                   | O(N)
시간 복잡도     | 원소 1개 처리 O(1)      | 전체 N개 필요, O(N ~ N log N)
병렬화          | 즉시 가능              | 스트림 분할 후 병합 필요
다음 연산 영향  | SORTED 제거             | SORTED/DISTINCT 플래그 추가
```

### 3. StreamOpFlag와 최적화

```java
// OpenJDK: java.util.stream.StreamOpFlag

public class StreamOpFlag {
    // 플래그 상수들
    static final int SORTED = 0x4;           // stream이 정렬됨
    static final int DISTINCT = 0x1;         // stream의 원소들이 distinct
    static final int SIZED = 0x2;            // stream 크기가 알려짐
    static final int SHORT_CIRCUIT = 0x8;    // short-circuit 가능
    
    // 플래그 조작
    static int orFlags(int flags, int opFlags) { return flags | opFlags; }
    static int andFlags(int flags, int opFlags) { return flags & opFlags; }
    
    // 최적화 결정
    static int getOpFlags(Stream stream) {
        // stream의 현재 플래그 상태 반환
        // → 다음 연산이 이를 보고 최적화 여부 결정
    }
}

플래그의 실제 활용:

1. SORTED 플래그:
   stream.sorted()  // SORTED 플래그 설정
   stream.distinct()  // SORTED 플래그 확인
   // → 중복 제거 시 인접한 원소만 비교 (정렬되어 있으므로)
   // → 실제로는 LinkedHashSet 같은 구조 필요 없음

2. DISTINCT 플래그:
   stream.distinct()  // DISTINCT 플래그 설정
   stream.distinct()  // DISTINCT 이미 설정됨?
   // → 중복 제거 중복 연산 가능 (최적화 기회)

3. SIZED 플래그:
   Arrays.stream(arr)  // SIZED 플래그 설정 (배열 크기 알려짐)
   stream.filter(...)  // SIZED 유지? 제거?
   stream.count()     // SIZED 있으면 크기 직접 알 수 있음
```

### 4. Sink 체인 구성 (Terminal 호출 시)

```java
// Terminal 호출 → wrapSink() 역방향 호출 → Sink 체인 구성

public class ReferencePipeline {
    
    // Terminal 예: forEach()
    public void forEach(Consumer<? super E_OUT> action) {
        evaluate(TerminalOp.makeForEach(action));
    }
    
    private <R> R evaluate(TerminalOp<E_OUT, R> terminalOp) {
        // 1. sourceStage부터 역방향 Sink 체인 생성
        Sink<E_OUT> sink = terminalOp.createSink();
        sink = sourceStage.wrapSink(sink);  // ← 역방향 호출 시작
        
        // 2. 파이프라인의 모든 노드를 역방향으로 wrapSink 호출
        for (AbstractPipeline stage = this; stage != sourceStage; stage = stage.previousStage) {
            sink = stage.opWrapSink(flags, sink);  // 각 노드가 Sink 감싸기
        }
        
        // 3. Source에서 원소를 push
        copyInto(sink, spliterator);  // source.forEach(sink::accept)
    }
}

구체적인 Sink 체인 예:

// 파이프라인:
stream.filter(x > 2).map(x * 2).forEach(println)

// Sink 체인 (역방향으로 생성):
forEach Sink
    ↓ (println 호출)
filter Sink (map Sink에서 받은 값만 전달)
    ↓ (x > 2 확인)
map Sink (filter Sink에서 받은 값만 처리)
    ↓ (x * 2 계산)
source Sink (원소 수용)

// 순방향 실행:
source: 1 → map(1*2=2) → filter(2>2? no) → skip println
source: 2 → map(2*2=4) → filter(4>2? yes) → println(4)
source: 3 → map(3*2=6) → filter(6>2? yes) → println(6)
```

---

## 💻 실전 실험

### 실험 1: 파이프라인 노드 구조 확인 (리플렉션)

```java
import java.lang.reflect.*;
import java.util.stream.*;

public class PipelineStructureTest {
    public static void main(String[] args) {
        Stream<Integer> stream = Stream.of(1, 2, 3)
            .filter(x -> x > 1)
            .map(x -> x * 2);
        
        // 리플렉션으로 파이프라인 노드 추적
        Object pipeline = stream;
        int depth = 0;
        
        while (pipeline != null && depth < 10) {
            System.out.printf("Node %d: %s%n", depth, pipeline.getClass().getSimpleName());
            
            try {
                // previousStage 필드 접근
                Field prev = pipeline.getClass().getDeclaredField("previousStage");
                prev.setAccessible(true);
                Object prevStage = prev.get(pipeline);
                
                // sourceOrOpFlags 확인
                Field flags = pipeline.getClass().getDeclaredField("sourceOrOpFlags");
                flags.setAccessible(true);
                int opFlags = flags.getInt(pipeline);
                System.out.printf("  OpFlags: 0x%x%n", opFlags);
                
                pipeline = prevStage;
                depth++;
            } catch (NoSuchFieldException e) {
                break;
            }
        }
        
        // 출력 예:
        // Node 0: StatelessOp (map)
        //   OpFlags: 0x???
        // Node 1: StatelessOp (filter)
        //   OpFlags: 0x???
        // Node 2: Head
        //   OpFlags: 0x1 (HEAD flag)
    }
}
```

### 실험 2: Stateless vs Stateful 성능

```java
import java.util.stream.*;

public class StatelessVsStatefulTest {
    public static void main(String[] args) {
        System.out.println("=== Stateless (filter + map) ===");
        long start = System.nanoTime();
        int result = Stream.iterate(1, x -> x + 1)
            .limit(10_000_000)
            .filter(x -> x % 2 == 0)  // Stateless
            .map(x -> x * 2)           // Stateless
            .mapToInt(x -> x)
            .sum();
        long elapsed1 = System.nanoTime() - start;
        System.out.printf("Result: %d, Time: %.2f ms%n", result, elapsed1 / 1_000_000.0);
        
        System.out.println("\n=== Stateful (sorted) ===");
        start = System.nanoTime();
        result = Stream.of(5, 3, 8, 1, 9, 2)
            .sorted()                // Stateful (모든 원소 필요)
            .mapToInt(x -> x)
            .sum();
        long elapsed2 = System.nanoTime() - start;
        System.out.printf("Result: %d, Time: %.4f ms%n", result, elapsed2 / 1_000_000.0);
        
        // Stateless가 훨씬 빠름 (O(N) vs O(N log N))
    }
}
```

### 실험 3: StreamOpFlag 확인

```java
import java.util.stream.*;
import java.lang.reflect.*;

public class StreamOpFlagTest {
    public static void main(String[] args) {
        // SORTED 플래그 확인
        Stream<Integer> sorted = Stream.of(1, 2, 3)
            .sorted();  // SORTED 플래그 설정
        
        printOpFlags(sorted, "After sorted()");
        
        sorted = sorted.filter(x -> x > 1);
        printOpFlags(sorted, "After sorted() + filter()");
    }
    
    static void printOpFlags(Stream<?> stream, String label) {
        try {
            Field flags = stream.getClass().getDeclaredField("sourceOrOpFlags");
            flags.setAccessible(true);
            int value = flags.getInt(stream);
            System.out.printf("%s: 0x%x%n", label, value);
            // SORTED = 0x4
            // DISTINCT = 0x1
            // SIZED = 0x2
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### 실험 4: Sink 체인 실시간 확인

```java
import java.util.stream.*;

public class SinkChainTest {
    public static void main(String[] args) {
        System.out.println("=== Sink 실행 순서 확인 ===");
        Stream.of(1, 2, 3)
            .peek(x -> System.out.println("  [1] peek(source): " + x))
            .filter(x -> {
                boolean pass = x > 1;
                System.out.println("  [2] filter(" + x + "): " + (pass ? "pass" : "skip"));
                return pass;
            })
            .map(x -> {
                int result = x * 2;
                System.out.println("  [3] map(" + x + "): " + result);
                return result;
            })
            .peek(x -> System.out.println("  [4] peek(after map): " + x))
            .forEach(x -> System.out.println("  [5] forEach: " + x));
        
        // 출력:
        // [1] peek(source): 1
        // [2] filter(1): skip
        // [1] peek(source): 2
        // [2] filter(2): pass
        // [3] map(2): 4
        // [4] peek(after map): 4
        // [5] forEach: 4
        // [1] peek(source): 3
        // [2] filter(3): pass
        // [3] map(3): 6
        // [4] peek(after map): 6
        // [5] forEach: 6
    }
}
```

---

## 📊 성능/비교

```
파이프라인 노드 계층별 성능 특성:

연산 타입              | 메모리  | 시간   | 병렬화  | 다음 연산 영향
─────────────────────┼────────┼────────┼────────┼──────────────
map(), filter()      | O(1)   | O(N)   | 즉시   | 플래그 제거
sorted()             | O(N)   | O(NlogN)| 병합   | SORTED 추가
distinct()           | O(N)   | O(N)   | 재정렬 | DISTINCT 추가
limit()              | O(k)   | O(k)   | 제한   | short-circuit
skip()               | O(1)   | O(N)   | 즉시   | 플래그 유지

최적화 예:

좋은 순서 (필터링 먼저):
  Stream.of(1..100M)
      .filter(x > 50M)      // O(100M) 필터 → 50M 통과
      .sorted()             // O(50M log 50M) 정렬
      .map(...)             // 50M 원소만 처리
  → 총 O(100M + 50M log 50M)

나쁜 순서 (정렬 먼저):
  Stream.of(1..100M)
      .sorted()             // O(100M log 100M) 정렬
      .filter(x > 50M)      // O(100M) 필터 → 50M 통과
      .map(...)             // 50M 원소만 처리
  → 총 O(100M log 100M + 100M) (불필요하게 전체 정렬)
```

---

## ⚖️ 트레이드오프

```
ReferencePipeline 설계의 트레이드오프:

장점:
  ✓ 통일된 파이프라인 구조 (Head/StatelessOp/StatefulOp)
  ✓ 역방향 Sink 체인으로 한 번의 패스 가능
  ✓ StreamOpFlag로 자동 최적화 기회
  ✓ 병렬 처리를 위한 명확한 경계 (Stateful)
  ✓ 무한 Stream 처리 (Stateless 중심)

단점:
  ✗ Stateful 연산은 모든 데이터를 메모리에 로드
  ✗ 파이프라인 노드 객체 생성 오버헤드
  ✗ 복잡한 파이프라인에서 디버깅 어려움
  ✗ 프리미티브 타입 변환 비용 (stream<Integer> vs IntStream)
  ✗ 짧은 스트림에서는 오버헤드가 상대적으로 큼

파이프라인 구성 vs 평가:
  - 구성: 중간 연산 호출 시 노드 객체 생성 (비용 O(중간 연산 수))
  - 평가: Terminal 호출 시 Sink 체인 생성 및 실행 (비용 O(N))
  - 일반적으로 생성 비용은 무시할 수 있음

선택 기준:
  작은 스트림 (< 1K): for 루프가 더 빠를 수 있음
  중간 스트림: Stream 오버헤드 상쇄
  대용량 스트림: Stream의 한 번의 패스 이점
  복합 연산: Stream 가독성 이점
```

---

## 📌 핵심 정리

```
ReferencePipeline 구조:

1. 파이프라인 노드 계층:
   - Head: 소스 (첫 노드, sourceStage 역할)
   - StatelessOp: map(), filter() (한 원소씩 처리)
   - StatefulOp: sorted(), distinct() (전체 데이터 필요)
   - Terminal: forEach(), collect() (평가 시작)

2. 필드 구조:
   - previousStage: 이전 노드 (역방향 추적)
   - sourceStage: 첫 Head 노드 (모든 노드가 공유)
   - sourceOrOpFlags: 이 노드의 StreamOpFlag
   - combinedFlags: 누적 플래그 (최적화 정보)

3. Sink 체인:
   - Terminal 호출 시 생성
   - 역방향: Terminal Sink → ... → Head Sink
   - 실행: 순방향 (Source push)
   - 한 원소씩 모든 중간 연산 통과

4. StreamOpFlag 최적화:
   - SORTED: 이미 정렬됨 → distinct() 최적화
   - DISTINCT: 이미 unique → 재처리 불필요
   - SIZED: 크기 알려짐 → count() 최적화
   - SHORT_CIRCUIT: 조기 종료 가능 (limit, findFirst)

5. 성능 최적화 원칙:
   - 필터링 먼저 (데이터 축소)
   - Stateful 연산 뒤로 (작은 데이터만 처리)
   - 중복 distinct() 제거 (플래그 확인)
```

---

## 🤔 생각해볼 문제

**Q1.** `Stream.of(1, 2, 3).filter(x > 1).map(x * 2).forEach(...)` 파이프라인에서 Sink 체인은 정확히 어떻게 생성되고 실행되는가?

<details>
<summary>해설 보기</summary>

**Sink 체인 생성 (역방향):**

1. `forEach()`는 `TerminalOp.makeForEach()`를 호출
2. `evaluate(terminalOp)` → `terminalOp.createSink()` → `ForEachSink` 생성
3. `map` StatelessOp 노드의 `opWrapSink(flags, ForEachSink)`
   → `x * 2`를 적용한 새 Sink 반환
4. `filter` StatelessOp 노드의 `opWrapSink(flags, mapSink)`
   → `x > 1` 필터를 적용한 새 Sink 반환
5. `Head` 노드의 `opWrapSink(flags, filterSink)`
   → 변환 없이 그대로 반환 (Source 역할)

**최종 Sink 체인 (구조):**
```
ForEachSink (println)
  ← mapSink (x * 2 적용)
    ← filterSink (x > 1 필터)
      ← sourceSink (원소 수용)
```

**실행 (순방향):**
```
source.forEach(sourceSink::accept)

원소 1:
  sourceSink.accept(1)
  → filterSink.accept(1)
     → (1 > 1? false) → skip

원소 2:
  sourceSink.accept(2)
  → filterSink.accept(2)
     → (2 > 1? true) → mapSink.accept(2)
        → mapSink.accept(4)  // 2 * 2 = 4
           → ForEachSink.accept(4)
              → println(4)

원소 3:
  sourceSink.accept(3)
  → filterSink.accept(3)
     → (3 > 1? true) → mapSink.accept(3)
        → mapSink.accept(6)
           → ForEachSink.accept(6)
              → println(6)
```

핵심: 각 Sink는 다음 Sink를 감싸고 있으며, `downstream.accept()`로 다음 Sink에 값을 전달한다.

</details>

---

**Q2.** `StatefulOp`가 모든 데이터를 메모리에 로드해야 하는 이유는 무엇인가? `sorted()` 예시로 설명하라.

<details>
<summary>해설 보기</summary>

**이유: 상태를 유지해야 함**

`sorted()`는 모든 원소를 비교해야 정렬 순서를 결정할 수 있다.

```java
Stream.of(5, 3, 8, 1, 2)
    .sorted()  // ← 첫 원소 5를 수용했다고 해서 이것이 첫 번째인지 알 수 없음
               // 모든 원소를 봐야 정렬 순서 결정

// Stateless (map)와의 차이:
Stream.of(5, 3, 8, 1, 2)
    .map(x -> x * 2)  // ← 각 원소를 즉시 처리, 다음 원소와 무관
```

**구현:**

```java
class SortedOp extends StatefulOp {
    @Override
    public Sink<E_OUT> opWrapSink(int flags, Sink<E_OUT> sink) {
        return new Sink.ChainedReference<E_OUT, E_OUT>(sink) {
            List<E_OUT> buffer = new ArrayList<>();  // ← 상태 저장
            
            @Override
            public void accept(E_OUT t) {
                buffer.add(t);  // 모든 원소 수집
            }
            
            @Override
            public void end() {
                buffer.sort(null);  // 정렬
                buffer.forEach(downstream::accept);  // 정렬된 순서로 전달
            }
        };
    }
}
```

**메모리 요구:**
- `filter()`: O(1) (원소 1개만 기억)
- `map()`: O(1) (원소 변환 후 즉시 전달)
- `sorted()`: O(N) (모든 원소를 List에 저장)
- `distinct()`: O(N) (Set으로 본 원소들 기억)

**최적화:**
- Stateful 연산 전 필터링으로 데이터 축소
- 여러 Stateful 연산 연쇄 시 마지막 Stateful 후 다시 축소

</details>

---

**Q3.** `StreamOpFlag.SORTED`가 설정된 후 `distinct()`를 호출하면 어떤 최적화가 일어나는가?

<details>
<summary>해설 보기</summary>

**최적화: 정렬된 데이터의 distinct**

일반적인 `distinct()` 구현:
```java
Set<E> seen = new HashSet<>();
stream.filter(x -> seen.add(x))  // 새로운 원소만 통과
```

이는 O(N) 시간, O(N) 공간이지만 모든 원소를 Set에 저장해야 한다.

**SORTED 플래그가 있으면:**
```java
// 정렬된 스트림의 distinct는 더 간단:
stream.filter(new Predicate<E>() {
    E prev = null;
    boolean first = true;
    
    @Override
    public boolean test(E curr) {
        if (first || !curr.equals(prev)) {
            prev = curr;
            first = false;
            return true;
        }
        return false;
    }
})
```

**장점:**
- 인접한 두 원소만 비교 → Set 불필요
- 공간 복잡도: O(N) → O(1)
- 시간: O(N) (비교만) vs O(N log N) (정렬) → 이미 정렬되어 있으므로 결과적으로 O(N)

**코드에서 확인:**
```java
Stream.of(1, 2, 2, 3, 3, 3)
    .sorted()                // SORTED 플래그 설정
    .distinct()              // 최적화된 distinct (Set 불필요)
    .forEach(System.out::println);  // 1, 2, 3
```

OpenJDK에서 실제로 `DistinctOps`는 SORTED 플래그를 확인하고 다른 구현을 선택한다.

</details>

---

<div align="center">

**[⬅️ 이전: Stream Pipeline 3단계 구조](./01-stream-pipeline-structure.md)** | **[홈으로 🏠](../README.md)** | **[다음: Lazy Evaluation 구현 ➡️](./03-lazy-evaluation-sink.md)**

</div>
