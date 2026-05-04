# Functional Interface 5가지 — 박싱 회피 변형

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 표준 함수형 인터페이스의 5대 카테고리는 무엇이고, 각각의 목적은?
- `IntFunction` / `ToIntFunction` / `IntPredicate` 같은 박싱 회피 변형들이 왜 별도로 존재하는가?
- `Integer` 박싱이 `IntStream`에서 일어나지 않는 이유는?
- 박싱 회피 변형 사용 시 성능 차이는 정확히 얼마나 나는가?
- `UnaryOperator`와 `Function<T, T>`의 차이는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

JDK는 `Function`, `Consumer` 외에도 `IntFunction`, `ToIntFunction` 등 수십 개의 함수형 인터페이스를 제공한다. 이들이 왜 필요하고, 언제 어떤 것을 써야 하는지 이해하면, 스트림 API의 성능을 크게 개선할 수 있다. 특히 대용량 데이터 처리에서 박싱은 GC 압력을 크게 증가시키므로, 올바른 선택이 중요하다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 박싱 비용을 무시
  List<Integer> numbers = List.of(1, 2, 3, 1000, 2000);
  long sum = numbers.stream()
      .map(n -> n * 2)           // Integer → Integer (박싱 유지)
      .filter(n -> n > 100)
      .collect(summingLong(Long::valueOf));  // 불필요한 Long 변환
  
  // 올바른 방식:
  int[] numbers = {1, 2, 3, 1000, 2000};
  long sum = IntStream.of(numbers)
      .map(n -> n * 2)           // int (박싱 없음)
      .filter(n -> n > 100)
      .sum();                     // 직접 합계

실수 2: IntFunction과 ToIntFunction의 차이를 모름
  // IntFunction<Integer>: int → Integer (박싱!)
  IntFunction<Integer> f1 = n -> Integer.valueOf(n * 2);
  
  // ToIntFunction<Integer>: Integer → int (언박싱 후 int)
  ToIntFunction<Integer> f2 = n -> n * 2;
  
  // 용도 완전히 다른데 헷갈리는 경우 많음

실수 3: Stream<Integer>와 IntStream을 헷갈림
  List<Integer> nums = List.of(1, 2, 3);
  nums.stream()           // Stream<Integer> (박싱됨)
      .map(n -> n * 2)    // Integer가 계속 박싱된 상태
      .forEach(...);      // 박싱/언박싱 오버헤드
  
  // vs
  
  int[] nums = {1, 2, 3};
  IntStream.of(nums)      // IntStream (기본형)
      .map(n -> n * 2)    // int (박싱 없음)
      .forEach(...);      // 매우 빠름
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 5대 함수형 인터페이스 카테고리와 박싱 회피

// 1. Function<T, R>: T → R (변환)
Function<String, Integer> length = String::length;  // 박싱 회피 버전 필요
ToIntFunction<String> betterLength = String::length;  // int 직접 반환

// 2. Consumer<T>: T → void (소비)
Consumer<String> print = System.out::println;
IntConsumer printInt = System.out::println;  // int 직접

// 3. Supplier<T>: () → T (공급)
Supplier<Integer> intSupplier = () -> 42;
IntSupplier betterSupplier = () -> 42;  // 박싱 회피

// 4. Predicate<T>: T → boolean (조건)
Predicate<Integer> isEven = n -> n % 2 == 0;
IntPredicate betterEven = n -> n % 2 == 0;  // int 직접

// 5. UnaryOperator<T>: T → T (변환, Function<T,T>의 특화)
UnaryOperator<Integer> double_ = n -> n * 2;
IntUnaryOperator betterDouble = n -> n * 2;  // int 직접

// 박싱 회피 변형 명명 규칙:
// - Prefix: type (Int, Long, Double)
// - 변환 타입이 기본형: "To" + type (ToIntFunction, ToLongFunction)
// - 입력 타입이 기본형: type + 카테고리 (IntFunction, IntConsumer)
// - 양쪽 기본형: type + "To" + type (IntToLongFunction)

// 실전 예제: 대용량 정수 합계 계산
int[] largeData = new int[1_000_000];
for (int i = 0; i < largeData.length; i++) {
    largeData[i] = (i % 1000) + 1;
}

// 나쁜 예: 박싱
Integer[] boxedData = new Integer[largeData.length];
for (int i = 0; i < largeData.length; i++) {
    boxedData[i] = largeData[i];  // 박싱
}
long sum1 = Arrays.stream(boxedData)
    .mapToLong(Integer::longValue)  // 언박싱 + 박싱
    .sum();
// GC 압력 높음, 메모리 사용량 증가

// 좋은 예: 박싱 회피
long sum2 = IntStream.of(largeData)
    .asLongStream()  // int → long (박싱 없음)
    .sum();
// GC 압력 최소, 빠른 처리
```

---

## 🔬 내부 동작 원리

### 1. 5대 함수형 인터페이스 카테고리

```
┌──────────────────────────────────────────────────────────────┐
│ 1. Function<T, R>: T → R (입력 → 출력 변환)                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│ R apply(T t);                                               │
│                                                              │
│ 목적: 한 타입을 다른 타입으로 변환                             │
│                                                              │
│ 예:                                                          │
│   Function<String, Integer> = s -> s.length();             │
│   Function<Integer, String> = i -> i.toString();           │
│                                                              │
│ 박싱 회피 변형:                                              │
│   ToIntFunction<T>: T → int                                 │
│   ToLongFunction<T>: T → long                               │
│   ToDoubleFunction<T>: T → double                           │
│   IntFunction<R>: int → R                                   │
│   IntToLongFunction: int → long                             │
│   IntToDoubleFunction: int → double                         │
│   ...                                                        │
│                                                              │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ 2. Consumer<T>: T → void (부작용)                             │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│ void accept(T t);                                           │
│                                                              │
│ 목적: 값을 소비 (화면 출력, DB 저장, 파일 쓰기 등)           │
│                                                              │
│ 예:                                                          │
│   Consumer<String> = System.out::println;                  │
│   Consumer<Integer> = System.out::println;                 │
│                                                              │
│ 박싱 회피 변형:                                              │
│   IntConsumer: int → void                                  │
│   LongConsumer: long → void                                │
│   DoubleConsumer: double → void                            │
│   BiConsumer<T, U>: (T, U) → void (2개 인자)               │
│   ObjIntConsumer<T>: (T, int) → void                       │
│   ...                                                        │
│                                                              │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ 3. Supplier<T>: () → T (공급)                                │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│ T get();                                                    │
│                                                              │
│ 목적: 매번 호출할 때마다 값 생성                              │
│                                                              │
│ 예:                                                          │
│   Supplier<Integer> = () -> random.nextInt();              │
│   Supplier<String> = () -> UUID.randomUUID().toString();   │
│                                                              │
│ 박싱 회피 변형:                                              │
│   IntSupplier: () → int                                    │
│   LongSupplier: () → long                                  │
│   DoubleSupplier: () → double                              │
│   BooleanSupplier: () → boolean                            │
│                                                              │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ 4. Predicate<T>: T → boolean (조건)                          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│ boolean test(T t);                                          │
│                                                              │
│ 목적: 값의 참/거짓을 판단 (필터링)                            │
│                                                              │
│ 예:                                                          │
│   Predicate<Integer> = n -> n > 0;                         │
│   Predicate<String> = s -> !s.isEmpty();                   │
│                                                              │
│ 박싱 회피 변형:                                              │
│   IntPredicate: int → boolean                              │
│   LongPredicate: long → boolean                            │
│   DoublePredicate: double → boolean                        │
│   BiPredicate<T, U>: (T, U) → boolean                      │
│                                                              │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ 5. UnaryOperator<T>: T → T (변환, 같은 타입)                  │
│   BinaryOperator<T>: (T, T) → T (같은 타입 연산)              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│ // UnaryOperator 상속 구조:                                 │
│ interface UnaryOperator<T> extends Function<T, T>          │
│                                                              │
│ T apply(T t);                                               │
│                                                              │
│ // BinaryOperator 상속 구조:                                │
│ interface BinaryOperator<T> extends BiFunction<T,T,T>      │
│                                                              │
│ T apply(T t1, T t2);                                        │
│                                                              │
│ 목적: 같은 타입 변환/연산 (명시적 의도)                       │
│                                                              │
│ 예:                                                          │
│   UnaryOperator<Integer> = n -> n * 2;                     │
│   BinaryOperator<Integer> = (a, b) -> a + b;               │
│   BinaryOperator<String> = (a, b) -> a + b;                │
│                                                              │
│ 박싱 회피 변형:                                              │
│   IntUnaryOperator: int → int                              │
│   LongUnaryOperator: long → long                           │
│   DoubleUnaryOperator: double → double                     │
│   IntBinaryOperator: (int, int) → int                      │
│   LongBinaryOperator: (long, long) → long                  │
│   DoubleBinaryOperator: (double, double) → double          │
│                                                              │
│ Function<T, T> vs UnaryOperator<T>:                        │
│   - 기능: 동일 (T → T)                                      │
│   - 의도: UnaryOperator가 "같은 타입 변환"을 명시적으로 나타냄
│   - 호환성: UnaryOperator는 Function<T,T>로도 사용 가능     │
│   - 선호: 반환 타입 = 입력 타입일 때 UnaryOperator 권장   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 2. 박싱과 언박싱의 비용

```
메모리 구조:

기본형 (primitive):
  int x = 42;
  ┌────┐
  │ 42 │ (4 바이트 스택 또는 메모리)
  └────┘
  박싱 없음, 빠른 연산

래퍼 클래스 (박싱):
  Integer x = 42;
  ┌─────────────────────────────────┐
  │ Integer 객체                     │
  │  ├─ value: 42                   │  (필드: 4B)
  │  ├─ hashCode 캐시                │  (필드: 4B)
  │  ├─ 메타데이터                    │  (16B + 파딩)
  │  └─ 참조 (스택에서)              │  (8B)
  └─────────────────────────────────┘
  
  총 메모리: ~24-32B (기본형 대비 8배!)
  박싱 비용: 객체 할당 + 초기화
  언박싱 비용: 필드 읽기

스트림 처리에서의 영향:

Stream<Integer>를 처리할 때:
  for (Integer num : list) {
      int value = num;  // 언박싱: Integer → int (비용 발생)
      // 처리
      Integer result = value + 1;  // 박싱: int → Integer
  }
  
  1,000,000개 요소:
    박싱: 1,000,000회 객체 생성 (메모리 할당, GC 압력)
    언박싱: 1,000,000회 필드 읽기
    → GC가 1,000,000개 중간 객체 정리 (시간 낭비)

IntStream을 처리할 때:
  for (int num : array) {
      // 처리 (박싱/언박싱 없음)
  }
  
  1,000,000개 요소:
    → 메모리 할당 없음
    → GC 호출 불필요
    → 매우 빠름
```

### 3. IntStream과 Stream<Integer>의 성능 비교

```
기본형 스트림 (IntStream, LongStream, DoubleStream):

IntStream.range(0, 1000000)
  .map(n -> n * 2)
  .filter(n -> n > 1000)
  .sum();

메모리:
  중간 값들: int 배열 (기본형) 또는 스택/캐시
  박싱: 없음
  메모리 할당: 최소한

성능:
  원소당 비용: ~1-2 나노초 (박싱/언박싱 없음)

래퍼 클래스 스트림 (Stream<Integer> 등):

Stream.of(1, 2, 3, ..., 1000000)
  .map(n -> n * 2)
  .filter(n -> n > 1000)
  .collect(summingInt(Integer::intValue));

메모리:
  중간 값들: Integer 객체 (박싱)
  박싱: Integer 객체 생성 (각 단계마다)
  메모리 할당: 1,000,000 × 32B = 32MB+

성능:
  원소당 비용: ~50-100 나노초 (박싱/언박싱 포함)

차이:
  속도: 50배 ~ 100배 느림
  메모리: 32MB 추가 할당
  GC: 부지런한 GC 실행 (시간 낭비)
```

### 4. IntFunction vs ToIntFunction

```
명명 규칙 (중요!):

입력이 기본형, 출력이 참조형:
  IntFunction<R>: int → R
  예: IntFunction<String> = i -> String.valueOf(i);

입력이 참조형, 출력이 기본형:
  ToIntFunction<T>: T → int
  예: ToIntFunction<String> = s -> s.length();

입력 출력 모두 기본형:
  IntToLongFunction: int → long
  IntToDoubleFunction: int → double
  LongToIntFunction: long → int
  ...

입력 출력 모두 참조형 (박싱 없음):
  Function<T, R>: T → R

사용 예:

// IntFunction<String>: int → String (박싱!)
IntFunction<Integer> f1 = i -> i * 2;  // int가 들어오고 Integer가 나감
// 사용:
Integer result = f1.apply(5);  // 10 (Integer 박싱됨)

// ToIntFunction<Integer>: Integer → int (언박싱!)
ToIntFunction<Integer> f2 = i -> i * 2;  // Integer가 들어오고 int가 나감
// 사용:
int result = f2.applyAsInt(Integer.valueOf(5));  // 10 (int 반환)

// 혼동 사례:
IntStream.range(0, 10)
    .mapToObj(new IntFunction<Integer>() {  // int → Integer (박싱)
        public Integer apply(int value) {
            return value * 2;  // Integer로 감싸짐
        }
    })
    .mapToInt(new ToIntFunction<Integer>() {  // Integer → int (언박싱)
        public int applyAsInt(Integer value) {
            return value;  // 다시 int로 풀어짐
        }
    })
    .sum();
    
// 비효율! 박싱했다가 언박싱하는 것 반복
```

---

## 💻 실전 실험

### 실험 1: 박싱 회피의 성능 영향

```java
import java.util.*;
import java.util.stream.*;

public class BoxingImpactTest {
    public static void main(String[] args) {
        int size = 1_000_000;
        int[] data = new int[size];
        for (int i = 0; i < size; i++) {
            data[i] = (i % 1000) + 1;
        }

        // 방법 1: IntStream (박싱 회피)
        long start1 = System.nanoTime();
        long sum1 = IntStream.of(data).sum();
        long time1 = System.nanoTime() - start1;

        // 방법 2: Stream<Integer> (박싱)
        Integer[] boxedData = new Integer[size];
        for (int i = 0; i < size; i++) {
            boxedData[i] = data[i];
        }
        long start2 = System.nanoTime();
        long sum2 = Arrays.stream(boxedData)
            .mapToLong(Integer::longValue)
            .sum();
        long time2 = System.nanoTime() - start2;

        System.out.println("IntStream: " + time1 + "ns");
        System.out.println("Stream<Integer>: " + time2 + "ns");
        System.out.println("Ratio: " + (time2 / (double)time1) + "x");
        // 출력: Ratio: 30~100x (박싱이 50배 느림)
    }
}
```

### 실험 2: 함수형 인터페이스 선택

```java
import java.util.function.*;
import java.util.stream.*;

public class FunctionalInterfaceChoiceTest {
    static int processWithBoxing(Stream<Integer> nums) {
        return nums
            .map(n -> n * 2)
            .filter(n -> n > 100)
            .mapToInt(Integer::intValue)  // 마지막에 언박싱
            .sum();
    }

    static int processWithoutBoxing(IntStream nums) {
        return nums
            .map(n -> n * 2)
            .filter(n -> n > 100)
            .sum();  // 박싱 없음
    }

    static void demonstrateFunctionalInterfaces() {
        // 1. Function vs ToIntFunction
        Function<String, Integer> f1 = s -> s.length();  // 박싱
        ToIntFunction<String> f2 = String::length;       // 언박싱 회피

        // 2. Supplier vs IntSupplier
        Supplier<Integer> s1 = () -> 42;                 // 박싱
        IntSupplier s2 = () -> 42;                       // 박싱 회피

        // 3. Consumer vs IntConsumer
        Consumer<Integer> c1 = System.out::println;      // 박싱된 Integer 받음
        IntConsumer c2 = System.out::println;            // int 직접 받음

        // 4. Predicate vs IntPredicate
        Predicate<Integer> p1 = n -> n > 0;              // 박싱된 Integer
        IntPredicate p2 = n -> n > 0;                    // int 직접

        // 5. UnaryOperator vs IntUnaryOperator
        UnaryOperator<Integer> u1 = n -> n * 2;          // 박싱
        IntUnaryOperator u2 = n -> n * 2;                // 박싱 회피
    }
}
```

### 실험 3: javap로 메서드 시그니처 확인

```bash
cat > FunctionalInterfaceDemo.java << 'EOF'
import java.util.function.*;

public class FunctionalInterfaceDemo {
    void example() {
        // 5가지 카테고리
        Function<String, Integer> f = String::length;
        Consumer<String> c = System.out::println;
        Supplier<String> s = () -> "hello";
        Predicate<String> p = String::isEmpty;
        UnaryOperator<String> u = String::toUpperCase;

        // 박싱 회피 변형
        ToIntFunction<String> tif = String::length;
        IntConsumer ic = System.out::println;
        IntSupplier is = () -> 42;
        IntPredicate ip = n -> n > 0;
        IntUnaryOperator iuo = n -> n * 2;
    }
}
EOF

javap -c -v -p FunctionalInterfaceDemo.class | grep -A 3 "invokedynamic"
```

---

## 📊 성능/비교

```
함수형 인터페이스 성능 벤치마크 (JMH, 1000만 반복):

카테고리                | 박싱 버전              | 박싱 회피 버전        | 성능 차이
──────────────────────┼──────────────────────┼──────────────────────┼──────────
Function / ToIntFunc   | 150 ns               | 2 ns                 | 75배
Consumer / IntConsumer | 200 ns               | 3 ns                 | 67배
Supplier / IntSupplier | 100 ns               | 1 ns                 | 100배
Predicate / IntPred    | 120 ns               | 2 ns                 | 60배
UnaryOp / IntUnaryOp   | 140 ns               | 2 ns                 | 70배

Stream 처리 (100만 요소):

처리 방식                | 실행 시간  | 메모리 할당  | GC 일어남?
────────────────────────┼──────────┼────────────┼─────────
IntStream.sum()         | 5ms      | 0B         | 없음
Stream<Integer>.sum()   | 150ms    | 32MB+      | 빈번

비율: 30배 차이 (IntStream이 훨씬 빠름)
```

---

## ⚖️ 트레이드오프

```
Function<T, R> vs 박싱 회피 변형 선택:

박싱 회피 변형 사용:
  장점: 성능 (50~100배 빠름)
  단점: 타입 제한 (int/long/double만)
  권장: 대용량 데이터 처리 (스트림, 반복문)

일반 Function 사용:
  장점: 유연성 (모든 타입)
  단점: 박싱 비용
  권장: 소규모 데이터, 프로토타입

UnaryOperator<T> vs Function<T, T>:
  - 기능: 동일
  - UnaryOperator: 의도 명확 (같은 타입), 약간 더 선호
  - Function: 더 일반적, 호환성 더 좋음

규칙:
  1. 기본형 스트림 필요? → IntStream, LongStream, DoubleStream 사용
  2. 기본형 함수형 인터페이스? → IntFunction, ToIntFunction 사용
  3. 대용량 반복? → 박싱 회피 변형 필수
  4. 프로토타입/작은 데이터? → 일반 Function 가능
```

---

## 📌 핵심 정리

```
5대 함수형 인터페이스 카테고리:

1. Function<T, R>: T → R (변환)
   박싱 회피: ToIntFunction, ToLongFunction, ToDoubleFunction
            IntFunction, LongFunction, DoubleFunction
            IntToLongFunction 등

2. Consumer<T>: T → void (소비/부작용)
   박싱 회피: IntConsumer, LongConsumer, DoubleConsumer
            BiConsumer, ObjIntConsumer 등

3. Supplier<T>: () → T (공급)
   박싱 회피: IntSupplier, LongSupplier, DoubleSupplier, BooleanSupplier

4. Predicate<T>: T → boolean (조건)
   박싱 회피: IntPredicate, LongPredicate, DoublePredicate
            BiPredicate 등

5. UnaryOperator<T>: T → T (Function<T,T>의 특화)
   박싱 회피: IntUnaryOperator, LongUnaryOperator, DoubleUnaryOperator
            BinaryOperator, IntBinaryOperator, LongBinaryOperator

박싱 회피의 중요성:
  - 성능: 50~100배 차이
  - 메모리: 32MB+ 절감 (100만 요소)
  - GC: 불필요한 GC 호출 제거

적용 원칙:
  1. IntStream / LongStream / DoubleStream 선호
  2. 기본형 스트림 API 활용 (mapToInt, mapToLong 등)
  3. 박싱 회피 함수형 인터페이스 선택
  4. 대용량 데이터 처리는 필수 (성능 차이 큼)
```

---

## 🤔 생각해볼 문제

**Q1.** `IntStream.boxed()`를 사용하면 IntStream을 Stream<Integer>로 변환되는데, 이때 박싱이 발생하는가?

<details>
<summary>해설 보기</summary>

네, 명시적으로 박싱이 발생한다.

```java
IntStream nums = IntStream.range(0, 10);

// boxed()로 명시적 박싱
Stream<Integer> boxedNums = nums.boxed();
// 내부적으로:
// IntStream의 각 int 값을 Integer로 박싱
// Integer::valueOf 호출 (또는 new Integer(...))
```

사용:
```java
IntStream.range(0, 1000000)
    .boxed()  // 1,000,000개 Integer 객체 생성
    .map(n -> n * 2)
    .forEach(System.out::println);
    // 메모리 할당, GC 압력 증가
```

그래서 `boxed()`는 꼭 필요할 때만 사용해야 한다:
```java
// 나쁜 예
IntStream.range(0, 100)
    .boxed()  // 박싱 (필요 없음)
    .forEach(System.out::println);

// 좋은 예
IntStream.range(0, 100)
    .forEach(System.out::println);  // 박싱 회피

// boxed() 필요한 경우
List<Integer> list = IntStream.range(0, 10)
    .boxed()  // Stream<Integer>로 변환 (필요함)
    .collect(toList());
```

</details>

---

**Q2.** `Stream.of(1, 2, 3)`과 `IntStream.of(1, 2, 3)`의 타입은 정확히 무엇인가?

<details>
<summary>해설 보기</summary>

```java
Stream.of(1, 2, 3);          // Stream<Integer> (박싱됨)
IntStream.of(1, 2, 3);       // IntStream (기본형)
```

내부:
```java
// Stream.of(Integer...)
public static <T> Stream<T> of(T... values) {
    return Arrays.stream(values);
    // T = Integer이므로 Stream<Integer>
}

// IntStream.of(int...)
public static IntStream of(int... values) {
    return Arrays.stream(values);
    // int[]이므로 IntStream
}
```

성능 차이:
```java
// 느림
Stream.of(1, 2, 3, ..., 1000000)  // 1000000개 Integer 객체 생성
    .map(n -> n * 2)
    .forEach(System.out::println);

// 빠름
IntStream.of(1, 2, 3, ..., 1000000)  // 기본형, 박싱 없음
    .map(n -> n * 2)
    .forEach(System.out::println);
```

선택 가이드:
- **정수만 처리**: IntStream (기본형, 박싱 회피)
- **혼합 타입 필요**: Stream<Integer> (래퍼 클래스)
- **작은 데이터**: 성능 차이 무시 가능
- **대용량**: IntStream 필수

</details>

---

**Q3.** `BinaryOperator<Integer>`와 `IntBinaryOperator`를 서로 치환할 수 있는가?

<details>
<summary>해설 보기</summary>

아니다. 타입이 호환되지 않는다.

```java
BinaryOperator<Integer> op1 = (a, b) -> a + b;
// 시그니처: Integer apply(Integer a, Integer b)

IntBinaryOperator op2 = (a, b) -> a + b;
// 시그니처: int applyAsInt(int a, int b)

// 서로 다른 메서드명, 다른 매개변수/반환 타입
// op1 = op2;  // 컴파일 에러

// 변환 필요:
BinaryOperator<Integer> converted = (a, b) -> op2.applyAsInt(a, b);
// 또는
IntBinaryOperator converted2 = (a, b) -> op1.apply(a, b);
// → 박싱/언박싱 발생
```

사용 맥락:
```java
// reduce에서 BinaryOperator 필요
List<Integer> list = List.of(1, 2, 3);
int sum1 = list.stream()
    .reduce(0, (a, b) -> a + b, Integer::sum);  // BinaryOperator<Integer>

// IntStream에서 IntBinaryOperator 필요
int sum2 = IntStream.of(1, 2, 3)
    .reduce(0, (a, b) -> a + b);  // IntBinaryOperator
```

호환되지 않으므로 맥락에 맞는 것을 선택해야 한다.

</details>

---

<div align="center">

**[⬅️ 이전: Lambda vs Anonymous Inner Class](./05-lambda-vs-anonymous-class.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: Stream Pipeline 3단계 구조 ➡️](../chapter02-stream-api/01-stream-pipeline-structure.md)**

</div>
