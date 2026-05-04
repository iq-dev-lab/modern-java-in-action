<div align="center">

# 🌊 Modern Java in Action

**"Lambda를 쓰는 것과, `invokedynamic`이 `LambdaMetafactory`를 거쳐 런타임에 어떻게 `Function` 구현체를 합성하는지 아는 것은 다르다"**

<br/>

> *"Stream을 쓰면 코드가 짧아지겠지 — 와 — `ReferencePipeline`이 중간 연산을 어떻게 lazy하게 연결하고, terminal 연산에서 `Spliterator`로 분할되며, `ForkJoinPool.commonPool()`이 왜 위험한지 아는 것의 차이를 만드는 레포"*

자바 8 람다가 `invokedynamic`과 `LambdaMetafactory`로 변환되는 과정, Stream의 `ReferencePipeline`이 lazy evaluation을 구현하는 방식, `CompletableFuture`가 Treiber stack 위에서 lock-free 콜백 체인을 구성하는 원리, Virtual Thread의 Continuation이 힙에 스택 프레임을 저장하고 캐리어 스레드 위에서 mount/unmount되는 과정까지
**왜 이렇게 설계됐는가** 라는 질문으로 자바 8 ~ 21 함수형 진화의 내부를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![Java](https://img.shields.io/badge/Java-8_~_21-ED8B00?style=flat-square&logo=openjdk&logoColor=white)](https://openjdk.org/projects/jdk/21/)
[![invokedynamic](https://img.shields.io/badge/invokedynamic-LambdaMetafactory-5382a1?style=flat-square&logo=openjdk&logoColor=white)](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-6.html#jvms-6.5.invokedynamic)
[![JMH](https://img.shields.io/badge/JMH-Benchmark-orange?style=flat-square)](https://github.com/openjdk/jmh)
[![Docs](https://img.shields.io/badge/Docs-56개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

모던 자바에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"어떤 기능이 추가됐나"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "Lambda는 익명 클래스보다 간결합니다" | `invokedynamic` + `LambdaMetafactory.metafactory()`가 런타임에 `Function` 구현체를 합성하는 과정, `javap -c -v`로 BootstrapMethods 분해, Anonymous Inner Class와의 바이트코드/메모리/성능 차이 |
| "Stream은 함수형 스타일로 컬렉션을 처리합니다" | `ReferencePipeline`의 Head / StatelessOp / StatefulOp 계층 구조, lazy evaluation이 `sourceStage`와 연산 체인으로 어떻게 구현되는지, `Spliterator.trySplit()`이 병렬 분할의 핵심인 이유 |
| "`effectively final`이어야 람다에서 캡처 가능합니다" | 캡처된 변수가 스택에서 힙으로 어떻게 박싱되는지, 익명 클래스의 합성 필드(`this$0`, `val$x`)와 람다의 인자 전달 방식 차이, JVM 스펙 상 final 요구의 근본 이유 |
| "Virtual Thread를 쓰면 처리량이 늘어납니다" | `Continuation` 객체가 JVM 힙에 실행 상태를 저장하는 방식, 블로킹 I/O 시 캐리어 스레드에서 언마운트되는 과정, `synchronized` Pinning이 왜 발생하고 어떻게 진단하는가 |
| "`CompletableFuture`로 비동기 체인을 만드세요" | `volatile Object result` + `volatile Completion stack`(Treiber stack)로 lock-free 콜백을 구현하는 방식, `thenApply` vs `thenCompose` vs `thenCombine`의 정확한 차이, `commonPool()` 의존성이 위험한 이유 |
| "Record는 보일러플레이트를 줄입니다" | Record 컴파일러가 자동 생성하는 `accessor` / `equals` / `hashCode` / `toString`의 바이트코드, Lombok `@Data`와의 차이, `Record Pattern`(Java 21)이 구조 분해를 어떻게 표현하는가 |
| "Optional로 NPE를 막으세요" | Optional이 `final class`로 설계된 이유, `map` vs `flatMap`의 Functor/Monad 의미, 필드/매개변수/Collection을 Optional로 감싸면 안 되는 근본 이유 |
| 이론 나열 | 재현 가능한 JMH 벤치마크 + JVM 플래그(`-Djdk.tracePinnedThreads=full`, `-XX:+PrintAssembly`) + `javap -c -v -p`로 BootstrapMethods 분해 + Spring Boot 통합 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

> 💡 모든 챕터는 첫 문서부터 독립적으로 읽을 수 있도록 설계됐습니다. 깊이 있는 학습은 「전체 학습 지도」와 「추천 학습 경로」를 따라가세요.

[![Ch1](https://img.shields.io/badge/🔹_Ch1-Lambda_Internals-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)](./chapter01-lambda-internals/01-lambda-to-invokedynamic.md)
[![Ch2](https://img.shields.io/badge/🔹_Ch2-Stream_API_내부-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)](./chapter02-stream-api/01-stream-pipeline-structure.md)
[![Ch3](https://img.shields.io/badge/🔹_Ch3-Parallel_Stream-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)](./chapter03-parallel-stream/01-fork-join-pool-internals.md)
[![Ch4](https://img.shields.io/badge/🔹_Ch4-Optional_패턴-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)](./chapter04-optional/01-optional-internal-structure.md)
[![Ch5](https://img.shields.io/badge/🔹_Ch5-CompletableFuture-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)](./chapter05-completable-future/01-completable-future-structure.md)
[![Ch6](https://img.shields.io/badge/🔹_Ch6-인터페이스_진화-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)](./chapter06-interface-evolution/01-default-method-internals.md)
[![Ch7](https://img.shields.io/badge/🔹_Ch7-Date/Time_API-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)](./chapter07-datetime-api/01-localdate-zoneddatetime.md)
[![Ch8](https://img.shields.io/badge/🔹_Ch8-Pattern_Matching_&_Records-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)](./chapter08-pattern-matching-records/01-record-internals.md)
[![Ch9](https://img.shields.io/badge/🔹_Ch9-Virtual_Threads-5382a1?style=for-the-badge&logo=openjdk&logoColor=white)](./chapter09-virtual-threads/01-platform-vs-virtual-thread.md)
[![Ch10](https://img.shields.io/badge/🔹_Ch10-함수형_패턴-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)](./chapter10-functional-patterns/01-higher-order-function.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: Lambda Expression Internals — invokedynamic과 LambdaMetafactory

> **핵심 질문:** Lambda는 어떻게 바이트코드로 변환되는가? `invokedynamic`은 어떻게 `LambdaMetafactory`를 호출해 런타임에 함수형 인터페이스 구현체를 합성하는가? `effectively final` 제약은 왜 필요한가?

<details>
<summary><b>invokedynamic 분해부터 박싱 회피 함수형 인터페이스까지 (6개 문서) — Java 8</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Lambda → invokedynamic 변환 — 바이트코드 분해](./chapter01-lambda-internals/01-lambda-to-invokedynamic.md) | 람다가 컴파일 시점에 어떻게 `invokedynamic` 명령어와 합성 메서드(`lambda$0`)로 분리되는지, `javap -c -v`로 BootstrapMethods 테이블 분해, `LambdaMetafactory.metafactory()`가 호출되는 시점, 익명 클래스 변환 방식과의 결정적 차이 |
| [02. @FunctionalInterface와 SAM 변환 메커니즘](./chapter01-lambda-internals/02-functional-interface-sam.md) | Single Abstract Method 인터페이스 검증 규칙, `default` / `static` 메서드가 SAM 카운팅에서 제외되는 이유, `@FunctionalInterface` 어노테이션의 역할(컴파일 검증), Comparable이 SAM이 아닌 이유, SAM 변환 시 컴파일러가 만드는 합성 클래스 분석 |
| [03. Method Reference 4가지 — 각각의 바이트코드](./chapter01-lambda-internals/03-method-reference-4-types.md) | `Class::staticMethod`(REF_invokeStatic), `instance::method`(REF_invokeVirtual + 캡처), `Class::instanceMethod`(unbound, 첫 인자가 receiver), `Class::new`(REF_newInvokeSpecial) 4가지의 BootstrapMethods 인자 차이, 각각이 람다와 동등한 코드로 어떻게 디슈가링되는가 |
| [04. Closure와 Variable Capture — effectively final의 이유](./chapter01-lambda-internals/04-closure-variable-capture.md) | 캡처된 지역변수가 람다 합성 메서드의 인자로 전달되는 방식, 익명 클래스의 합성 필드(`val$x`) 방식과의 차이, `effectively final` 제약이 JVM 스펙이 아닌 컴파일러 결정인 이유, 가변 캡처(`AtomicReference`/배열 트릭)의 위험성과 스레드 안전성 |
| [05. Lambda vs Anonymous Inner Class — 메모리·성능 비교](./chapter01-lambda-internals/05-lambda-vs-anonymous-class.md) | 람다는 클래스 파일을 만들지 않고 런타임에 `LambdaMetafactory`로 합성, 익명 클래스는 `Outer$1.class` 파일 생성, JVM 시작 시간/메모리 풋프린트/CallSite 캐싱 차이, JMH로 두 방식의 호출 성능 측정, 캡처 없는 람다가 싱글톤으로 재사용되는 동작 |
| [06. Functional Interface 5가지 — 박싱 회피 변형](./chapter01-lambda-internals/06-functional-interface-categories.md) | `Function` / `Consumer` / `Supplier` / `Predicate` / `UnaryOperator`/`BinaryOperator` 5대 카테고리, `IntFunction` / `ToIntFunction` / `IntPredicate` 등 박싱 회피 변형의 존재 이유, `Integer` 박싱이 `IntStream`에서 일어나지 않는 이유와 JMH 측정 |

</details>

<br/>

### 🔹 Chapter 2: Stream API 내부 동작 — ReferencePipeline과 Lazy Evaluation

> **핵심 질문:** Stream은 어떻게 lazy하게 동작하는가? `ReferencePipeline`의 Head / StatelessOp / StatefulOp 계층은 어떻게 연결되고, terminal 연산은 어떻게 평가를 트리거하는가? Collector의 Mutable Reduction은 어떻게 구현되는가?

<details>
<summary><b>Stream Pipeline 3단계 구조부터 Custom Collector 작성까지 (8개 문서) — Java 8 / 9 / 16</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Stream Pipeline 3단계 구조 — Source / Intermediate / Terminal](./chapter02-stream-api/01-stream-pipeline-structure.md) | 모든 Stream의 3단계 구조(Source → Intermediate → Terminal), 중간 연산이 즉시 실행되지 않고 연결만 되는 이유, terminal 연산이 호출되어야 평가가 시작되는 lazy evaluation의 본질, Stream이 1회용인 이유와 두 번 소비 시 `IllegalStateException`이 발생하는 메커니즘 |
| [02. ReferencePipeline 소스코드 추적 — Head / StatelessOp / StatefulOp](./chapter02-stream-api/02-reference-pipeline-source.md) | `AbstractPipeline`의 `previousStage` / `sourceStage` / `sourceOrOpFlags` 필드 분석, `Head`(소스), `StatelessOp`(map, filter), `StatefulOp`(sorted, distinct) 계층의 차이, OpenJDK 소스코드 직접 추적으로 파이프라인 노드 연결 방식 분해 |
| [03. Lazy Evaluation 구현 — Sink와 Push 모델](./chapter02-stream-api/03-lazy-evaluation-sink.md) | terminal 연산 호출 시 역방향으로 `Sink` 체인을 구성하는 과정, source가 한 원소씩 push하면 모든 중간 연산이 한 번의 패스로 실행되는 동작, `forEach`와 `for` 루프의 본질적 차이가 없는 이유, `peek`이 디버깅 외엔 부작용을 만드는 위험 |
| [04. Spliterator — 분할 가능 컬렉션 인터페이스](./chapter02-stream-api/04-spliterator-deep-dive.md) | `Spliterator`의 `tryAdvance` / `trySplit` / `estimateSize` / `characteristics` 4대 메서드, `ArrayList.spliterator()` vs `LinkedList.spliterator()`의 분할 효율 차이, `Spliterator.SIZED`/`SUBSIZED`/`ORDERED`/`SORTED` characteristics가 병렬 처리 최적화에 미치는 영향, Custom Spliterator 작성 |
| [05. Short-circuit Operations — findFirst, anyMatch, limit](./chapter02-stream-api/05-short-circuit-operations.md) | `findFirst` / `findAny` / `anyMatch` / `allMatch` / `noneMatch` / `limit`이 어떻게 조기 종료를 신호로 보내는지, `Sink.cancellationRequested()`의 역할, 무한 Stream(`Stream.iterate`, `Stream.generate`)이 short-circuit과 결합될 때만 종료되는 이유 |
| [06. Stateless vs Stateful 연산 — map vs sorted/distinct](./chapter02-stream-api/06-stateless-vs-stateful.md) | `map` / `filter` 같은 stateless 연산이 한 원소씩 처리 가능한 이유, `sorted` / `distinct` / `limit` 같은 stateful 연산이 전체 데이터를 모아야 하는 이유, stateful 연산이 병렬 스트림 성능을 떨어뜨리는 메커니즘, 파이프라인 재정렬 시 stateful 연산 위치의 영향 |
| [07. Collector 내부 — Mutable Reduction과 Combiner](./chapter02-stream-api/07-collector-mutable-reduction.md) | `Collector`의 `supplier` / `accumulator` / `combiner` / `finisher` / `characteristics` 5대 컴포넌트, `reduce`(immutable)와 `collect`(mutable)의 본질적 차이, `Collector.Characteristics.CONCURRENT` / `UNORDERED` / `IDENTITY_FINISH`가 병렬 처리에 미치는 영향, `Collectors.toList()` 내부 구현 |
| [08. Custom Collector 작성 — Collector.of() 분석](./chapter02-stream-api/08-custom-collector.md) | `Collector.of()` 정적 팩토리 시그니처 분석, 누적자(accumulator)와 결합자(combiner) 작성 가이드, 병렬 처리 시 combiner가 호출되는 조건, 실전 예제(통계 수집, 그룹핑 변형, 다단계 분류) Custom Collector 구현 |

</details>

<br/>

### 🔹 Chapter 3: 병렬 스트림과 Fork/Join — Work-Stealing의 내부

> **핵심 질문:** ForkJoinPool은 어떻게 Work-Stealing을 구현하는가? `parallelStream()`이 사용하는 `commonPool()`이 왜 위험한가? 병렬 스트림이 오히려 느려지는 임계값은 어떻게 결정되는가?

<details>
<summary><b>ForkJoinPool 동작 원리부터 NQ 모델 임계값까지 (5개 문서) — Java 7 / 8</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. ForkJoinPool 동작 원리 — Work-Stealing 알고리즘](./chapter03-parallel-stream/01-fork-join-pool-internals.md) | 각 worker 스레드가 자신의 deque를 가지고 LIFO로 작업을 push/pop하는 구조, 다른 스레드의 deque에서 FIFO로 훔치는(steal) Work-Stealing 메커니즘, `ForkJoinTask`의 `fork()`와 `join()`이 deque에 미치는 영향, 일반 ThreadPoolExecutor와의 결정적 차이 |
| [02. Common Pool 의존성 — commonPool()이 위험한 이유](./chapter03-parallel-stream/02-common-pool-danger.md) | `parallelStream()`과 `CompletableFuture` 기본 Executor가 모두 `ForkJoinPool.commonPool()`을 공유하는 구조, 한 곳에서 풀을 점유하면 전체 애플리케이션이 영향받는 시나리오, 격리된 ForkJoinPool 또는 전용 Executor 분리 패턴 |
| [03. Spliterator 분할 전략 — trySplit과 characteristics](./chapter03-parallel-stream/03-spliterator-split-strategy.md) | `ArrayList`(SIZED+SUBSIZED, 균등 분할)와 `LinkedList`(SIZED만, 순차 분할 비효율) 분할 비교, `HashMap.spliterator()`의 분할 효율, 무한 Stream의 분할 한계, `IntStream.range()`가 가장 효율적으로 분할되는 이유, characteristics 비트 플래그가 옵티마이저에 주는 힌트 |
| [04. 병렬 스트림 성능 함정 — 박싱·작은 데이터셋·stateful](./chapter03-parallel-stream/04-parallel-stream-pitfalls.md) | `Stream<Integer>`의 박싱이 병렬화 이득을 상쇄하는 메커니즘(`IntStream`으로 회피), 데이터셋이 작을 때 스레드 분할 오버헤드가 처리 시간을 초과하는 임계값, `sorted` / `distinct` 같은 stateful 연산이 병렬 처리를 직렬화하는 케이스, `forEachOrdered` 사용 시 병렬화 이득 소실 |
| [05. NQ Model — 언제 병렬 스트림을 써야 하는가](./chapter03-parallel-stream/05-nq-model-threshold.md) | Brian Goetz의 NQ 모델(N=데이터 크기, Q=원소당 처리 비용), `N × Q > 10,000` 휴리스틱의 의미, JMH로 N과 Q를 변화시키며 직렬/병렬 교차점 측정, IO-bound vs CPU-bound 작업의 병렬화 적합성, `parallelStream()` 결정 트리 |

</details>

<br/>

### 🔹 Chapter 4: Optional 패턴 — Functor와 Monad의 자바 구현

> **핵심 질문:** Optional은 왜 `final class`로 설계됐는가? `map`과 `flatMap`의 차이는 단순한 시그니처 차이를 넘어 무엇을 의미하는가? Optional을 필드와 매개변수에 쓰면 안 되는 근본적 이유는?

<details>
<summary><b>Optional 내부 구조부터 직렬화 함정까지 (5개 문서) — Java 8 / 9 / 10</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Optional 내부 구조 — 왜 final class인가](./chapter04-optional/01-optional-internal-structure.md) | `private final T value` 단일 필드 구조, `EMPTY` 상수의 싱글톤 재사용, `final class`로 상속 금지한 설계 의도(value-based class), `@ValueBased` 어노테이션과 `==` 비교 금지, `Serializable` 미구현 결정의 영향 |
| [02. Optional.of vs ofNullable vs empty — 정확한 사용 시점](./chapter04-optional/02-of-vs-ofnullable-vs-empty.md) | `Optional.of(null)`이 NPE를 던지는 의도적 설계, `ofNullable`이 안전한 래핑을 보장하는 메커니즘, `Optional.empty()`가 `EMPTY` 상수를 반환하는 최적화, "이 값이 null이면 버그"와 "null일 수 있음"의 의도를 코드로 표현하는 방법 |
| [03. map vs flatMap — Functor와 Monad 패턴](./chapter04-optional/03-map-vs-flatmap-monad.md) | `map`이 `Optional<T> → Optional<U>` 변환(Functor)인 반면 `flatMap`이 `Optional<T> → Optional<U>` 평탄화(Monad)인 이유, `Optional<Optional<T>>`가 발생하는 케이스와 `flatMap`으로 회피, 함수형 카테고리 이론에서의 Functor/Monad 법칙이 Optional에 어떻게 적용되는가 |
| [04. Optional 안티패턴 — 필드·매개변수·Collection 금지](./chapter04-optional/04-optional-antipatterns.md) | Optional을 필드로 사용하면 안 되는 이유(직렬화 불가, 메모리 오버헤드), 매개변수 Optional이 호출자 부담을 늘리는 메커니즘, `Optional<List<T>>`가 빈 리스트와 의미 중복인 이유, JDK API에서 Optional이 메서드 반환 타입에만 사용된 설계 일관성 |
| [05. 직렬화·ORM·Optional — 실전 통합](./chapter04-optional/05-optional-serialization-orm.md) | `Optional`이 `Serializable`을 구현하지 않은 결과, JPA 엔티티 필드에 Optional을 쓰면 안 되는 이유와 Hibernate의 처리, Jackson의 Optional 직렬화 모듈(jackson-datatype-jdk8), GraphQL `null vs missing` 구분과 Optional의 한계 |

</details>

<br/>

### 🔹 Chapter 5: CompletableFuture 비동기 프로그래밍 — Lock-Free 콜백 체인

> **핵심 질문:** `CompletableFuture`는 어떻게 lock-free로 비동기 콜백 체인을 구현하는가? `thenApply` / `thenCompose` / `thenCombine`의 정확한 차이는? 예외 처리 3가지(`exceptionally` / `handle` / `whenComplete`)는 언제 어떻게 다른가?

<details>
<summary><b>Treiber Stack 구조부터 ForkJoinPool 분리 전략까지 (7개 문서) — Java 8 / 9 / 12</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. CompletableFuture 구조 — Future와의 차이와 Callback Hell 해결](./chapter05-completable-future/01-completable-future-structure.md) | `volatile Object result`(null/T/AltResult) + `volatile Completion stack`(Treiber stack) 구조, `Future`의 `get()` 블로킹 한계와 `CompletionStage` 인터페이스의 콜백 등록 모델, `Future` → `CompletableFuture` 진화의 동기 |
| [02. thenApply vs thenCompose vs thenCombine — Function vs Future 차이](./chapter05-completable-future/02-thenapply-thencompose-thencombine.md) | `thenApply(Function<T,U>)`(동기 변환), `thenCompose(Function<T,CompletionStage<U>>)`(비동기 체인 평탄화 = flatMap), `thenCombine(CompletionStage<U>, BiFunction<T,U,V>)`(두 비동기 결과 병합)의 시그니처 차이, `Optional.map` / `flatMap`과의 패턴 동질성 |
| [03. thenRun vs thenAccept vs thenApply — 결과 사용 패턴](./chapter05-completable-future/03-thenrun-thenaccept-thenapply.md) | `thenRun(Runnable)`(결과 무시), `thenAccept(Consumer<T>)`(결과 소비, 반환 없음), `thenApply(Function<T,U>)`(변환 후 반환) 3가지의 사용 시점, `Consumer` vs `Function`의 함수형 인터페이스 카테고리 차이가 비동기 체인에서 의미하는 것 |
| [04. Async 변형 — thenApplyAsync와 Executor 지정 전략](./chapter05-completable-future/04-async-executor-strategy.md) | `thenApply` vs `thenApplyAsync`의 실행 스레드 차이(현재 스레드 vs `commonPool()`), `thenApplyAsync(fn, executor)`로 전용 Executor 지정, IO-bound 작업에 `commonPool` 사용 시의 위험, Executor를 명시적으로 분리하는 패턴 |
| [05. 예외 처리 3가지 — exceptionally, handle, whenComplete](./chapter05-completable-future/05-exception-handling-three-ways.md) | `exceptionally(Throwable → T)`(예외 시에만 호출), `handle(BiFunction<T, Throwable, U>)`(성공/실패 모두 호출, 변환 가능), `whenComplete(BiConsumer<T, Throwable>)`(부작용만, 결과 변경 불가) 3가지의 정확한 차이와 사용 시점 |
| [06. allOf · anyOf — 구현 분석과 실전 패턴](./chapter05-completable-future/06-allof-anyof-patterns.md) | `CompletableFuture.allOf()`가 모든 future 완료 시 트리거되는 메커니즘, `anyOf()`의 first-completion 패턴, `allOf().thenApply(v -> futures.stream().map(CF::join))`로 결과 수집, 부분 실패 시 전체 동작과 Timeout 결합 패턴 |
| [07. ForkJoinPool.commonPool() 의존성 문제와 해결](./chapter05-completable-future/07-commonpool-dependency.md) | `CompletableFuture.supplyAsync()` 기본 Executor가 `commonPool`인 함정, `parallelStream()`과 풀 공유 시 발생하는 starvation, IO-bound 작업에 적합한 별도 Executor 설계(스레드 수 = N × (1 + W/C) 공식), Virtual Thread Executor 통합 |

</details>

<br/>

### 🔹 Chapter 6: 인터페이스 진화 — Default Method부터 Sealed Interface까지

> **핵심 질문:** Default Method는 다이아몬드 상속을 어떻게 해결하는가? Private Interface Method가 도입된 이유는? Sealed Interface는 Pattern Matching과 어떻게 연계되는가?

<details>
<summary><b>Default Method 동작부터 Sealed Interface까지 (5개 문서) — Java 8 / 9 / 17</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Default Method 동작 원리 — invokespecial vs invokevirtual](./chapter06-interface-evolution/01-default-method-internals.md) | Default Method가 호출될 때 사용되는 `invokeinterface` 명령어, 구현체 메서드 우선 규칙(class > interface), `super` 호출 시 `Interface.super.method()` 문법이 `invokespecial`로 컴파일되는 과정, ABI 호환성을 깨지 않으면서 인터페이스를 확장한 설계 의도 |
| [02. 다이아몬드 상속 해결 규칙 — Class > Interface, Sub > Super](./chapter06-interface-evolution/02-diamond-inheritance-rules.md) | 두 인터페이스가 같은 default method를 가질 때 컴파일 에러를 발생시키고 명시적 오버라이드를 강제하는 규칙, "더 구체적인 인터페이스 우선" 규칙(SubInterface가 SuperInterface 오버라이드), Class가 Interface보다 우선하는 규칙, 전이적 적용 케이스 분석 |
| [03. Private Interface Method (Java 9) — 코드 중복 제거](./chapter06-interface-evolution/03-private-interface-method.md) | Java 9에서 도입된 private interface method가 default method 간 공통 로직을 캡슐화하는 방식, `private` / `private static` 변형, 기존 `static` helper 클래스 패턴 대비 응집도 향상, JDK 표준 라이브러리에서의 활용 사례 |
| [04. Static Interface Method와 Helper Method 패턴](./chapter06-interface-evolution/04-static-interface-method.md) | Java 8 이전의 `Collections.unmodifiableList(List)` 같은 외부 helper 클래스 패턴, Java 8 `List.of()`, `Optional.empty()` 같은 static factory를 인터페이스에 통합한 진화, static interface method가 상속되지 않는 규칙과 그 근거 |
| [05. Sealed Interface (Java 17) — 상속 봉인과 Pattern Matching 연계](./chapter06-interface-evolution/05-sealed-interface.md) | `sealed interface ... permits A, B, C` 문법으로 구현체를 명시적으로 제한하는 메커니즘, `final` / `sealed` / `non-sealed` 자식 한정자, ADT(Algebraic Data Type) 표현 가능성, switch expression의 exhaustive matching이 컴파일러에 의해 보장되는 원리 |

</details>

<br/>

### 🔹 Chapter 7: 새로운 Date/Time API — 불변성과 명확한 의미론

> **핵심 질문:** `LocalDate` / `LocalDateTime` / `ZonedDateTime` / `Instant`는 각각 무엇을 표현하는가? 왜 모든 메서드가 `withYear()` 같은 "새 인스턴스 반환"인가? `Date`/`Calendar`에서 마이그레이션할 때의 함정은?

<details>
<summary><b>4가지 시간 타입부터 Date 마이그레이션까지 (4개 문서) — Java 8</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. LocalDate · LocalDateTime · ZonedDateTime · Instant — 의미와 사용 시점](./chapter07-datetime-api/01-localdate-zoneddatetime.md) | `LocalDate`(타임존 없는 날짜), `LocalDateTime`(타임존 없는 일시), `ZonedDateTime`(타임존 포함), `Instant`(UTC 타임라인의 한 점) 4가지 타입의 의미론적 차이, "회의 시각"과 "이벤트 시각"이 각각 어떤 타입에 매핑되는지, DB 저장 시 어떤 타입을 쓸지 결정하는 가이드 |
| [02. 불변성 설계 — 왜 withYear() 같은 메서드만 있는가](./chapter07-datetime-api/02-immutability-design.md) | 모든 Date/Time 객체가 `final class`로 설계된 이유, `setYear()` 대신 `withYear()`가 새 인스턴스를 반환하는 의도(value object), 스레드 안전성과 동시성 자료구조에 안전하게 사용 가능한 이유, `Date`의 가변성이 일으켰던 버그들 |
| [03. TemporalAdjuster · TemporalQuery — 활용 패턴](./chapter07-datetime-api/03-temporal-adjuster-query.md) | `TemporalAdjuster`(예: "다음 월요일", "이번 달 마지막 영업일")로 복잡한 날짜 계산을 함수형으로 표현, `TemporalAdjusters` 정적 팩토리 활용, `TemporalQuery`로 날짜에서 정보를 추출하는 패턴, Custom TemporalAdjuster 작성 |
| [04. Date/Calendar → New API 마이그레이션 전략](./chapter07-datetime-api/04-date-calendar-migration.md) | `Date.toInstant()`, `Calendar.toInstant()`, `Date.from(Instant)` 변환 API, JPA 엔티티에서 `@Temporal` 어노테이션 대신 `LocalDateTime` 직접 매핑, 외부 라이브러리/레거시 코드와의 경계에서 변환 패턴, 시간대 누락으로 발생하는 24시간 오차 함정 |

</details>

<br/>

### 🔹 Chapter 8: Pattern Matching & Records — 데이터 중심 설계

> **핵심 질문:** Record는 어떻게 보일러플레이트를 자동 생성하고 어떤 바이트코드가 되는가? Pattern Matching for `instanceof`, switch expression, record pattern이 결합되어 만드는 표현력은? Sealed Class와 합쳐 exhaustive matching을 컴파일러가 어떻게 보장하는가?

<details>
<summary><b>Record 바이트코드부터 Record Pattern까지 (6개 문서) — Java 14 / 16 / 17 / 21</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Record 구조 (Java 16) — 자동 생성 메서드와 바이트코드](./chapter08-pattern-matching-records/01-record-internals.md) | Record 컴파일러가 자동 생성하는 canonical constructor, accessor 메서드, `equals` / `hashCode` / `toString`의 바이트코드, `java.lang.Record` 추상 클래스 상속, `invokedynamic` + `ObjectMethods.bootstrap`으로 `equals`/`hashCode`/`toString`을 합성하는 방식 |
| [02. Record vs Lombok @Data — 어떤 차이가 있는가](./chapter08-pattern-matching-records/02-record-vs-lombok.md) | Record는 final class + 불변 + 표준화 vs Lombok @Data는 가변 + 어노테이션 프로세서 의존, Compact constructor와 Lombok 검증 로직 비교, JPA 엔티티에서 Record가 부적합한 이유(no-args constructor 부재), DTO/Value Object 용도 적합성 |
| [03. Pattern Matching for instanceof (Java 16) — 캐스팅 제거](./chapter08-pattern-matching-records/03-pattern-matching-instanceof.md) | `if (obj instanceof String s) { ... s.length() ... }` 문법이 어떻게 바이트코드 상 자동 캐스팅으로 변환되는지, scope 결정 규칙(then-branch에서만 가시), Negation 시 변수 가시성 변화, switch와 결합 시의 의미 |
| [04. Switch Expression (Java 14) — yield와 화살표 문법](./chapter08-pattern-matching-records/04-switch-expression.md) | `switch ... -> ...` 화살표 문법이 fall-through를 제거하는 의미, 여러 라벨 결합(`case A, B ->`), 블록 내 `yield` 키워드로 값 반환, `default` 누락 시 비exhaustive 에러, expression vs statement 두 형태 비교 |
| [05. Sealed Classes (Java 17) — Exhaustive Pattern Matching](./chapter08-pattern-matching-records/05-sealed-classes-exhaustive.md) | `sealed class Shape permits Circle, Square, Triangle`가 switch expression의 exhaustive 검사를 가능하게 하는 메커니즘, 모든 자식 케이스를 다루지 않으면 컴파일 에러가 발생하는 보장, ADT 표현으로 `default` 없이 안전한 분기 |
| [06. Record Pattern (Java 21) — 구조 분해 패턴](./chapter08-pattern-matching-records/06-record-pattern-destructuring.md) | `case Point(int x, int y) -> ...` 문법으로 record를 즉시 분해하는 메커니즘, 중첩 record 분해(`Line(Point(int x1, int y1), Point(int x2, int y2))`), `var` 추론 결합, switch와 결합한 ADT 풀 매칭 패턴, 함수형 언어(Scala/Kotlin/Rust)와의 비교 |

</details>

<br/>

### 🔹 Chapter 9: Virtual Threads & Project Loom — 패러다임의 전환

> **핵심 질문:** Virtual Thread는 Platform Thread와 무엇이 본질적으로 다른가? Continuation은 어떻게 stack을 yield/resume하는가? Pinning은 왜 발생하고 어떻게 진단·해결하는가? Structured Concurrency는 부모-자식 task 관계를 어떻게 구조화해 리소스 누수를 막는가?

<details>
<summary><b>1:1 vs M:N 모델부터 Structured Concurrency까지 (5개 문서) — Java 19 Preview / 21</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Platform Thread vs Virtual Thread — 1:1 vs M:N 모델](./chapter09-virtual-threads/01-platform-vs-virtual-thread.md) | OS 스레드 1:1 매핑 모델의 비용(1MB 스택, 컨텍스트 스위칭, OS 자원 한계)과 M:N 모델로 수백만 Virtual Thread를 만들 수 있는 메커니즘, "Thread per Request" 모델의 처리량 상한선, Virtual Thread가 이 문제를 해결하는 설계 |
| [02. Continuation 메커니즘 — Stack을 어떻게 yield/resume하는가](./chapter09-virtual-threads/02-continuation-mechanism.md) | `Continuation` 객체가 JVM 힙에 Virtual Thread의 스택 프레임을 저장하는 방식, `yield()`로 현재 실행 상태를 저장하고 `run()`으로 재개하는 원리, ForkJoinPool 캐리어 스레드(CPU 코어 수)가 Virtual Thread를 스케줄링하는 과정, mount/unmount 전환 |
| [03. Pinning 문제 — synchronized와 native call 진단·해결](./chapter09-virtual-threads/03-pinning-problem.md) | `synchronized` 블록 안에서 블로킹 I/O 호출 시 캐리어 스레드도 함께 블로킹되는 "Pinning" 발생 원인(JVM 모니터 구현 제약), `-Djdk.tracePinnedThreads=full` 플래그와 JFR `jdk.VirtualThreadPinned` 이벤트로 진단, `ReentrantLock`으로 교체해 Pinning을 해결하는 패턴 |
| [04. Structured Concurrency (Java 21 Preview)](./chapter09-virtual-threads/04-structured-concurrency.md) | `StructuredTaskScope`로 부모-자식 task 관계를 구조화하는 모델, `ShutdownOnFailure` / `ShutdownOnSuccess` 정책, 자식 task 누수 방지(부모 스코프 종료 시 모든 자식 자동 취소), `ExecutorService` + `CompletableFuture` 패턴 대비 안전성 향상 |
| [05. Thread Pool → Virtual Thread 전환 전략](./chapter09-virtual-threads/05-thread-pool-migration.md) | 기존 ThreadPoolExecutor 기반 코드를 `Executors.newVirtualThreadPerTaskExecutor()`로 전환하는 가이드, 풀링이 Virtual Thread에서 안티패턴인 이유(생성 비용이 거의 0), ThreadLocal 메모리 누수 회피, Spring Boot `spring.threads.virtual.enabled=true` 통합 |

</details>

<br/>

### 🔹 Chapter 10: 함수형 프로그래밍 패턴 — 자바에서의 함수형 사고

> **핵심 질문:** 자바에서 고차 함수 / 커링 / 메모이제이션을 어떻게 구현하는가? 영속 자료구조는 무엇이며, Either/Result 패턴은 예외 처리를 어떻게 대체하는가?

<details>
<summary><b>고차 함수부터 Either/Result 패턴까지 (5개 문서) — Java 8 / 14 / 21</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 고차 함수 (Higher-Order Function) — 자바에서의 적용](./chapter10-functional-patterns/01-higher-order-function.md) | 함수를 매개변수로 받거나 반환하는 고차 함수의 정의, `Function<T, R>` / `Function<T, Function<U, R>>`로 표현하는 방식, 전략 패턴(Strategy)을 함수형으로 단순화하는 패턴, JDK API에서 고차 함수가 적용된 사례(`Comparator.comparing`, `Collectors.groupingBy`) |
| [02. 커링 (Currying)과 부분 적용 (Partial Application)](./chapter10-functional-patterns/02-currying-partial-application.md) | `BiFunction<A, B, R>`을 `Function<A, Function<B, R>>`로 변환하는 커링, 부분 적용으로 함수의 일부 인자를 미리 고정하는 패턴, 자바 문법의 한계(다른 함수형 언어 대비 verbose)와 그럼에도 적용 가능한 실전 예제(설정 주입, DI) |
| [03. 메모이제이션 패턴 — ConcurrentHashMap.computeIfAbsent 활용](./chapter10-functional-patterns/03-memoization-computeifabsent.md) | 순수 함수의 결과를 캐싱하는 메모이제이션 패턴, `ConcurrentHashMap.computeIfAbsent`로 thread-safe 메모이제이션 구현, `Function<T, R>`을 메모이제이션 데코레이터로 감싸는 일반화, 캐시 크기 제한(`Caffeine`) 통합과 메모리 누수 회피 |
| [04. 영속 자료구조 (Persistent Data Structure)와 함수형 컬렉션](./chapter10-functional-patterns/04-persistent-data-structure.md) | 변경 시 새 버전을 만들고 이전 버전을 보존하는 영속 자료구조 개념, 구조 공유(structural sharing)로 O(N) 복사를 회피하는 메커니즘, `List.of()` / `Map.of()`(Java 9) immutable 컬렉션의 한계, Vavr의 `Vector` / `HashMap` 영속 컬렉션 활용 |
| [05. Either/Result 패턴 — Vavr vs 직접 구현](./chapter10-functional-patterns/05-either-result-pattern.md) | 예외를 값으로 표현하는 `Either<L, R>` / `Result<T, E>` 패턴, checked exception의 함수형 한계와 대안, Vavr `Try` / `Either` 활용, Java 21 Sealed Interface + Record로 직접 구현하는 패턴, Spring `@ExceptionHandler` 와의 결합 |

</details>

---

## 🔬 실험 환경

```yaml
# docker-compose.yml
services:
  java-lab:
    image: eclipse-temurin:21-jdk
    volumes:
      - ./src:/workspace/src
      - ./benchmarks:/workspace/benchmarks
    working_dir: /workspace
    command: /bin/bash
    stdin_open: true
    tty: true
    environment:
      - JAVA_OPTS=-XX:+UseZGC -Xmx2g --enable-preview

  jmh-runner:
    image: maven:3.9-eclipse-temurin-21
    volumes:
      - ./benchmarks:/workspace
    working_dir: /workspace
    command: mvn clean package && java -jar target/benchmarks.jar
```

```bash
# ── Lambda → invokedynamic 분해 ──────────────────────────────────────

# 바이트코드 + BootstrapMethods 테이블 출력
javap -c -v -p MyLambda.class

# 합성 메서드(lambda$0) 포함 모든 메서드 표시 (-p)
# BootstrapMethods 섹션에서 LambdaMetafactory 호출 인자 확인

# ── JIT 컴파일 + 어셈블리 ──────────────────────────────────────────

# 주의: -XX:+PrintAssembly는 hsdis 라이브러리가 필요합니다
#       설치: https://github.com/openjdk/jdk/tree/master/src/utils/hsdis
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+LogCompilation \
     -XX:+PrintAssembly \
     MyApp

# ── Virtual Thread 진단 ─────────────────────────────────────────────

# Pinning 발생 추적 (synchronized 안에서 블로킹 시 출력)
java -Djdk.tracePinnedThreads=full MyVirtualThreadApp

# ── Java Flight Recorder (성능 프로파일링) ──────────────────────────

# 60초 기록
jcmd <pid> JFR.start duration=60s filename=recording.jfr

# Virtual Thread Pinning 이벤트 추출
jfr print --events jdk.VirtualThreadPinned recording.jfr

# ── JMH 벤치마크 실행 ──────────────────────────────────────────────

# 3 fork / 5 warmup / 10 measurement / 4 threads
java -jar benchmarks.jar -f 3 -wi 5 -i 10 -t 4 ".*StreamBenchmark.*"
```

```java
// 기본 JMH 벤치마크 구조 (모든 챕터에서 활용)
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Fork(3)
@Warmup(iterations = 5)
@Measurement(iterations = 10)
public class ModernJavaBenchmark {

    @Benchmark
    public long sequentialStream() { /* ... */ }

    @Benchmark
    public long parallelStream() { /* ... */ }

    @Benchmark
    public long traditionalForLoop() { /* ... */ }
}
```

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 구조로 작성됩니다. 각 문서는 약 **400~1000줄** (코드 블록 포함), 평균 학습 시간 **20~40분**입니다.

| 섹션 | 설명 | 예상 분량 |
|------|------|----------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 | 5문항 |
| 🔍 **왜 이 기능이 도입되었는가** | Java 진화 맥락에서의 도입 배경 | 1~2 단락 |
| 😱 **흔한 오해 또는 잘못된 사용** | Before — 원리를 모를 때의 접근 | 코드 + 주석 |
| ✨ **올바른 이해와 사용** | After — 원리를 알고 난 후의 설계/구현 | 코드 패턴 |
| 🔬 **내부 동작 원리** | JDK 소스코드 + 바이트코드 분석 (`javap -c -v`) + ASCII 다이어그램 | 핵심 분량, 200~500줄 |
| 💻 **실험으로 확인하기** | 실행 가능한 코드 + JVM 플래그 + javap 분석 | 2~3개 실험 |
| 📊 **성능 비교** | JMH 벤치마크 또는 마이크로벤치마크 수치 | 표 + 측정값 |
| ⚖️ **트레이드오프** | 이 설계의 장단점, 다른 접근과의 비교 | 짧은 비교표 |
| 📌 **핵심 정리** | 한 화면 요약 | 코드 블록 1개 |
| 🤔 **생각해볼 문제** | 개념을 더 깊이 이해하기 위한 질문 + 해설 | 3문항 + 펼침 해설 |

---

## 🗺️ 추천 학습 경로

<details>
<summary><b>🟢 "Lambda와 Stream을 매일 쓰지만 내부가 궁금하다" — 입문 (4주)</b></summary>

<br/>

```
1주차  Ch1-01  Lambda → invokedynamic 변환 → 람다의 본질 이해
        Ch1-03  Method Reference 4가지 → 각 케이스 바이트코드 차이
        Ch1-04  Closure와 Variable Capture → effectively final의 이유
        Ch1-06  Functional Interface 5가지 → 박싱 회피 변형

2주차  Ch2-01  Stream Pipeline 3단계 구조 → lazy의 본질
        Ch2-02  ReferencePipeline 소스코드 → 내부 계층 추적
        Ch2-05  Short-circuit Operations → 조기 종료 메커니즘
        Ch2-07  Collector Mutable Reduction → reduce vs collect

3주차  Ch4-01  Optional 내부 구조 → final class 설계 의도
        Ch4-03  map vs flatMap → Functor와 Monad
        Ch4-04  Optional 안티패턴 → 필드/매개변수 금지 이유
        Ch7-01  LocalDate / ZonedDateTime → 4가지 시간 타입

4주차  Ch5-01  CompletableFuture 구조 → Future와의 차이
        Ch5-02  thenApply vs thenCompose → 비동기 체인 평탄화
        Ch5-05  예외 처리 3가지 → exceptionally / handle / whenComplete
```

</details>

<details>
<summary><b>🟡 "Virtual Thread를 도입하려는데 모던 자바 전반을 정리한다" — 중급 (8주)</b></summary>

<br/>

```
1~2주차  Chapter 1 + 2 — Lambda Internals + Stream API
          → invokedynamic 분해, ReferencePipeline 추적

3주차    Chapter 3 — 병렬 스트림과 Fork/Join
          → commonPool 위험성, NQ 모델 임계값

4주차    Chapter 4 + 7 — Optional + Date/Time
          → Functor/Monad 패턴, 불변 시간 타입

5~6주차  Chapter 5 — CompletableFuture
          → Treiber stack 구조, Executor 분리 전략

7주차    Chapter 8 — Pattern Matching & Records
          → Record 바이트코드, Sealed + exhaustive matching

8주차    Chapter 9 — Virtual Threads
          → Continuation 메커니즘, Pinning 진단/해결
```

</details>

<details>
<summary><b>🔴 "Java 8 ~ 21 함수형 진화를 바이트코드 수준까지 완전히 정복" — 전체 정복 (3개월)</b></summary>

<br/>

```
1개월차  Chapter 1 + 2 + 3 — 람다와 Stream 내부 완전 이해
          → javap -c -v로 BootstrapMethods 분해
          → ReferencePipeline / Spliterator 소스코드 추적
          → ForkJoinPool Work-Stealing 알고리즘 분석

2개월차  Chapter 4 + 5 + 6 + 7 — 함수형 API 전반
          → Optional Functor/Monad 법칙 검증
          → CompletableFuture Treiber stack 추적
          → Default Method 다이아몬드 해결 규칙
          → Date/Time 불변성과 마이그레이션 전략

3개월차  Chapter 8 + 9 + 10 — 최신 패러다임
          → Record / Sealed / Pattern Matching 통합
          → Virtual Thread Continuation 힙 저장 분석
          → ScopedValue, Structured Concurrency 도입
          → 함수형 패턴(고차 함수, 메모이제이션, Either) 실전
```

</details>

---

## 🔗 연관 레포지토리

| 레포 | 주요 내용 | 연관 챕터 |
|------|----------|-----------|
| [jvm-deep-dive](https://github.com/dev-book-lab/jvm-deep-dive) | JVM 구조, 클래스로딩, GC, JIT 컴파일 | Ch1(invokedynamic 명령어 디스패치), Ch9(Continuation의 JVM 힙 저장) |
| [java-concurrency-deep-dive](https://github.com/dev-book-lab/java-concurrency-deep-dive) | JMM, 락 내부, Virtual Thread, ConcurrentHashMap | Ch3(ForkJoinPool Work-Stealing), Ch5(CompletableFuture lock-free), Ch9(Virtual Thread 동시성 모델) |
| [effective-java](https://github.com/dev-book-lab/effective-java) | Effective Java 모범 사례 정리 | Ch1(람다와 메서드 참조 선호), Ch4(Optional 반환 가이드), Ch8(불변 클래스와 Record) |
| [object](https://github.com/dev-book-lab/object) | Object 책 — OOP 설계 원칙 | Ch1(함수형이 OOP를 보완하는 방식), Ch10(고차 함수와 전략 패턴) |
| [spring-boot-internals](https://github.com/dev-book-lab/spring-boot-internals) | Spring MVC 요청 처리, DispatcherServlet | Ch9(Tomcat 스레드 모델 → Virtual Thread 전환) |
| [linux-for-backend-deep-dive](https://github.com/dev-book-lab/linux-for-backend-deep-dive) | OS 프로세스·스레드, 컨텍스트 스위칭, epoll | Ch9(Virtual Thread가 epoll 위에서 동작하는 방식) |

> 💡 이 레포는 **자바 8 ~ 21 함수형 진화의 내부**에 집중합니다. `jvm-deep-dive`로 invokedynamic과 클래스로딩을 먼저 이해하면 Chapter 1(Lambda Internals)의 깊이가 배가됩니다. `java-concurrency-deep-dive`로 ForkJoinPool과 Virtual Thread를 익히면 Chapter 3, 5, 9가 자연스럽게 연결됩니다.

---

## 📚 Reference

- [Modern Java in Action — Raoul-Gabriel Urma, Mario Fusco, Alan Mycroft](https://www.manning.com/books/modern-java-in-action) — 모던 자바 입문서
- [Java Concurrency in Practice — Brian Goetz et al.](https://jcip.net/) — 동시성 바이블
- [Java Language Specification (Java SE 21)](https://docs.oracle.com/javase/specs/jls/se21/html/index.html)
- [JVM Specification — invokedynamic](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-6.html#jvms-6.5.invokedynamic)
- [JEP 261 (Module System)](https://openjdk.org/jeps/261)
- [JEP 286 (Local-Variable Type Inference, var)](https://openjdk.org/jeps/286)
- [JEP 361 (Switch Expression)](https://openjdk.org/jeps/361)
- [JEP 378 (Text Blocks — Standard, Java 15)](https://openjdk.org/jeps/378)
- [JEP 395 (Records)](https://openjdk.org/jeps/395)
- [JEP 406 (Pattern Matching for switch — Preview)](https://openjdk.org/jeps/406)
- [JEP 409 (Sealed Classes)](https://openjdk.org/jeps/409)
- [JEP 440 (Record Patterns)](https://openjdk.org/jeps/440)
- [JEP 441 (Pattern Matching for switch)](https://openjdk.org/jeps/441)
- [JEP 444 (Virtual Threads)](https://openjdk.org/jeps/444)
- [JEP 453 (Structured Concurrency — Preview)](https://openjdk.org/jeps/453)
- [Brian Goetz — Stream API Javadoc](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/Stream.html)
- [Aleksey Shipilëv's Blog](https://shipilev.net/) — JVM 성능과 JMM 분석의 최고 자료

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

*"Lambda를 쓰는 것과, `invokedynamic`이 `LambdaMetafactory`를 거쳐 런타임에 어떻게 `Function` 구현체를 합성하는지 아는 것은 다르다"*

</div>
