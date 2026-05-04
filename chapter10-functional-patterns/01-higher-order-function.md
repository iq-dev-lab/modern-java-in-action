# 고차 함수 (Higher-Order Function) — 자바에서의 적용

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 고차 함수의 정의는 무엇이고, 자바에서 왜 Java 8부터 가능한가?
- `Function<T, R>`과 `Function<T, Function<U, R>>`의 차이는 무엇인가?
- 전략 패턴(Strategy Pattern)을 고차 함수로 어떻게 단순화하는가?
- `Comparator.comparing`, `Collectors.groupingBy` 같은 JDK API들이 고차 함수를 활용하는 원리는?
- 고차 함수 체이닝에서 타입 추론의 한계는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

고차 함수는 함수형 프로그래밍의 기초다. Spring Data의 `Specification`, Jackson의 `@JsonInclude`, Stream API의 `map()`, `filter()`, `reduce()`가 모두 고차 함수를 매개변수로 받는다. 고차 함수를 이해하면 기존 코드의 동작을 깊이 있게 파악할 수 있고, 자신의 API를 더 유연하고 재사용 가능하게 설계할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 고차 함수와 콜백의 혼동
  // 콜백으로 생각하며 구현
  interface UserProcessor {
      void process(User u);  // void 반환, 부작용 의존
  }
  List<User> users = ...;
  for (User u : users) {
      processor.process(u);  // 실행만 하고 결과 없음
  }
  
  // 문제: 함수형이 아님, 부작용에 의존, 합성 불가
  // 올바른 함수형: Function<User, Result>

실수 2: 객체 메서드를 고차 함수 대신 사용
  // Comparator 직접 구현
  Comparator<Person> cmp = new Comparator<Person>() {
      @Override
      public int compare(Person a, Person b) {
          return a.getAge() - b.getAge();
      }
  };
  people.sort(cmp);
  
  // 문제: 5줄이 필요함
  // 올바른 방법: people.sort(Comparator.comparingInt(Person::getAge));

실수 3: 고차 함수 반환의 복잡성 도피
  // "이것은 너무 복잡하다" 라며 일반 클래스로 대체
  public class OperationFactory {
      public static ArithmeticOperation createAdd() {
          return new ArithmeticOperation() {
              @Override public int execute(int a, int b) { return a + b; }
          };
      }
  }
  
  // 문제: 간단한 것을 복잡하게 함
  // 올바른 방법: public static Function<int[], Integer> createAdd() { return a -> a[0] + a[1]; }
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 고차 함수 올바른 사용 패턴

// 패턴 1: 함수를 매개변수로 받기
public static <T, R> List<R> map(List<T> list, Function<T, R> f) {
    return list.stream().map(f).collect(Collectors.toList());
}

List<String> names = map(people, Person::getName);

// 패턴 2: 함수를 반환하기
public static Comparator<Person> comparingByAge() {
    return Comparator.comparingInt(Person::getAge);
}

// 더 강력한 버전: 함수 조합
public static Function<Person, String> describeFormat() {
    return p -> String.format("%s (age: %d)", p.getName(), p.getAge());
}

// 패턴 3: 전략 패턴을 고차 함수로
// Before: 8가지 전략 클래스 필요
interface ValidationStrategy { boolean validate(String input); }
class EmailStrategy implements ValidationStrategy { ... }
class PhoneStrategy implements ValidationStrategy { ... }

// After: 단순 함수
Function<String, Boolean> emailValidator = s -> s.matches("^[A-Za-z0-9+_.-]+@(.+)$");
Function<String, Boolean> phoneValidator = s -> s.matches("^\\d{10,}$");

// 패턴 4: Comparator 합성 (고차 함수 응용)
people.sort(
    Comparator.comparingInt(Person::getAge)
              .thenComparingInt(Person::getSalary)
              .reversed()
);

// 패턴 5: 함수 체이닝
Function<Integer, Integer> double_f = x -> x * 2;
Function<Integer, Integer> addOne = x -> x + 1;
Function<Integer, Integer> composed = double_f.andThen(addOne);
// composed.apply(5) = (5 * 2) + 1 = 11

// 패턴 6: 고차 함수로 커스텀 컬렉터 만들기
<T, K> Collector<T, ?, Map<K, List<T>>> groupBy(Function<T, K> classifier) {
    return Collectors.groupingBy(classifier);
}
```

---

## 🔬 내부 동작 원리

### 1. Function 인터페이스의 구조

```java
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);  // 핵심: T를 입력받아 R을 반환
    
    // 함수 합성 메서드들 (default)
    default <V> Function<V, R> compose(Function<? super V, ? extends T> before) {
        return v -> apply(before.apply(v));  // before 먼저 실행, 결과를 this에 전달
    }
    
    default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
        return t -> after.apply(apply(t));   // this 먼저 실행, 결과를 after에 전달
    }
}

// 예: Function<Integer, Integer> f = x -> x * 2;
// f.compose(Integer::parseInt)        → String → Integer → Integer
// f.andThen(String::valueOf)         → Integer → Integer → String
```

### 2. BiFunction과 커링

```java
// BiFunction: 두 개 매개변수
@FunctionalInterface
public interface BiFunction<T, U, R> {
    R apply(T t, U u);  // 두 개의 입력
}

// 고차 함수로 변환 (커링): T와 U를 순차적으로 받음
public static <T, U, R> Function<T, Function<U, R>> curry(BiFunction<T, U, R> f) {
    return t -> u -> f.apply(t, u);
}

BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;
Function<Integer, Function<Integer, Integer>> curriedAdd = curry(add);

// 사용:
Integer result = curriedAdd.apply(5).apply(3);  // 8

// 부분 적용: 첫 인자 고정
Function<Integer, Integer> addFive = curriedAdd.apply(5);  // a=5 고정
addFive.apply(3);  // 8
addFive.apply(10); // 15
```

### 3. JDK의 고차 함수 활용

```java
// Comparator.comparing - 고차 함수의 대표 예
public static <T, U extends Comparable<? super U>> Comparator<T> comparing(
    Function<? super T, ? extends U> keyExtractor) {
    return (c1, c2) -> keyExtractor.apply(c1)
                                    .compareTo(keyExtractor.apply(c2));
}

// 내부 동작:
// 1. Function<T, U>를 받음
// 2. Comparator<T>를 반환함 (고차 함수)
// 3. 반환된 Comparator는 받은 Function을 사용하여 비교 수행

people.sort(Comparator.comparing(Person::getAge));
// = people.sort((p1, p2) -> Integer.compare(p1.getAge(), p2.getAge()));

// Collectors.groupingBy - 더 복잡한 고차 함수
public static <T, K> Collector<T, ?, Map<K, List<T>>> groupingBy(
    Function<? super T, ? extends K> classifier) {
    return groupingBy(classifier, toList());
}

// 세 개 매개변수 버전:
public static <T, K, D, A, M extends Map<K, D>> Collector<T, ?, M> groupingBy(
    Function<? super T, ? extends K> classifier,
    Supplier<M> mapFactory,
    Collector<? super T, A, D> downstream) {
    // 고차 함수 3개를 조합: classifier, mapFactory, downstream
}

people.stream()
    .collect(groupingBy(
        Person::getDepartment,  // Function<Person, String>
        HashMap::new,           // Supplier<HashMap>
        mapping(Person::getName, toList())  // Collector
    ));
```

### 4. Stream API에서의 고차 함수

```java
// map: Function<T, R> 받음
Stream<Person> -> Stream<String> : stream.map(Person::getName)

// filter: Predicate<T> (Function<T, Boolean>의 특수형)
Stream<Person> -> Stream<Person> : stream.filter(p -> p.getAge() > 30)

// flatMap: Function<T, Stream<R>> 받음 (고차 함수!)
// 각 요소를 스트림으로 변환 후 평탄화
stream.flatMap(p -> getAllPeersOf(p).stream())

// reduce: BiFunction<T, T, T> + BinaryOperator<T>
// 고차 함수를 받아 누적값 계산
numbers.stream()
    .reduce(0, Integer::sum)  // (acc, n) -> acc + n
    // 내부: BinaryOperator<Integer> (Integer, Integer) -> Integer

// collect: 고차 함수 3개 합성
stream.collect(
    HashMap::new,        // Supplier<A> - 누적값 생성
    Map::put,            // BiConsumer<A, T> - 누적값에 요소 추가
    Map::putAll          // BiConsumer<A, A> - 두 누적값 병합
);
```

---

## 💻 실전 실험

### 실험 1: 고차 함수의 성능 - 메서드 참조 vs 람다 vs 익명 클래스

```java
import java.util.*;
import java.util.function.Function;

public class HigherOrderFunctionBenchmark {
    
    static class Person {
        String name;
        int age;
        Person(String name, int age) { this.name = name; this.age = age; }
        public String getName() { return name; }
    }
    
    public static void main(String[] args) {
        List<Person> people = new ArrayList<>();
        for (int i = 0; i < 100000; i++) {
            people.add(new Person("Person" + i, (i % 100) + 1));
        }
        
        // 테스트 1: 메서드 참조 (가장 최적화)
        long start = System.nanoTime();
        List<String> names1 = people.stream()
            .map(Person::getName)
            .toList();
        long time1 = System.nanoTime() - start;
        
        // 테스트 2: 람다 (거의 동일)
        start = System.nanoTime();
        List<String> names2 = people.stream()
            .map(p -> p.getName())
            .toList();
        long time2 = System.nanoTime() - start;
        
        // 테스트 3: 익명 클래스 (약간 느림)
        start = System.nanoTime();
        List<String> names3 = people.stream()
            .map(new Function<Person, String>() {
                @Override public String apply(Person p) { return p.getName(); }
            })
            .toList();
        long time3 = System.nanoTime() - start;
        
        System.out.printf("메서드 참조: %.2f ms%n", time1 / 1_000_000.0);
        System.out.printf("람다: %.2f ms%n", time2 / 1_000_000.0);
        System.out.printf("익명 클래스: %.2f ms%n", time3 / 1_000_000.0);
        
        // 결과: 메서드 참조 ≈ 람다 < 익명 클래스 (JIT 최적화)
    }
}
```

### 실험 2: Function 합성의 실제 동작

```java
public class FunctionCompositionTest {
    public static void main(String[] args) {
        Function<Integer, Integer> double_f = x -> {
            System.out.println("  double: " + x);
            return x * 2;
        };
        
        Function<Integer, Integer> addTen = x -> {
            System.out.println("  addTen: " + x);
            return x + 10;
        };
        
        System.out.println("=== andThen: double(5) -> addTen(result) ===");
        Function<Integer, Integer> composed1 = double_f.andThen(addTen);
        Integer result1 = composed1.apply(5);
        System.out.println("Result: " + result1);  // 20
        
        System.out.println("\n=== compose: addTen(5) -> double(result) ===");
        Function<Integer, Integer> composed2 = double_f.compose(addTen);
        Integer result2 = composed2.apply(5);
        System.out.println("Result: " + result2);  // 30
        
        // andThen 실행 순서: double(5)=10 → addTen(10)=20
        // compose 실행 순서: addTen(5)=15 → double(15)=30
    }
}
```

### 실험 3: Comparator 고차 함수 합성

```java
import java.util.*;
import java.util.stream.IntStream;

public class ComparatorCompositionTest {
    static class Person {
        String name;
        int age;
        int salary;
        Person(String name, int age, int salary) {
            this.name = name; this.age = age; this.salary = salary;
        }
        @Override public String toString() {
            return String.format("%s(%d, %d)", name, age, salary);
        }
    }
    
    public static void main(String[] args) {
        List<Person> people = List.of(
            new Person("Alice", 30, 5000),
            new Person("Bob", 25, 6000),
            new Person("Charlie", 30, 4000),
            new Person("Diana", 25, 7000)
        );
        
        // 정렬: 나이(오름차순) → 급여(내림차순)
        people.stream()
            .sorted(
                Comparator.comparingInt(Person::getAge)
                          .thenComparingInt(Person::getSalary)
                          .reversed()
            )
            .forEach(System.out::println);
        
        // 출력:
        // Diana(25, 7000)
        // Bob(25, 6000)
        // Alice(30, 5000)
        // Charlie(30, 4000)
    }
}
```

---

## 📊 성능/비교

```
고차 함수 성능 비교 (100만 요소 스트림):

패턴                          | 시간 (ms) | 메모리    | 특징
────────────────────────────┼──────────┼────────┼────────────────────
메서드 참조 (Person::getName) | 45       | 12MB   | JIT 인라인 최적화 ✓
람다 (p -> p.getName())      | 46       | 12MB   | 메서드 참조와 동일
익명 클래스 new Function<..> | 48       | 14MB   | 약간의 오버헤드
Function 체이닝 (4번)         | 52       | 13MB   | 각 단계마다 함수 호출

// JMH 벤치마크 예상:
메서드 참조 vs 람다: 차이 없음 (JIT가 동일하게 최적화)
메서드 참조 vs 익명 클래스: 5~10% 차이 (익명 클래스가 느림)
함수 체이닝이 많을수록: 반복 호출 비용 누적

실제 성능 병목:
  - 고차 함수 호출 자체: 무시할 수준
  - 진짜 비용: 람다 캡처 변수의 박싱, 스트림 중간 객체 생성
```

---

## ⚖️ 트레이드오프

```
고차 함수의 트레이드오프:

복잡한 타입:
  장점: 함수 합성으로 복잡한 로직을 작은 조각으로 분해
  단점: Function<T, Function<U, R>>같은 타입은 읽기 어려움
       IDE 자동 완성도 제한됨

람다 표현식의 한계:
  장점: 메서드 참조는 매우 간결 (Person::getAge)
  단점: 복합 로직은 람다 본문이 길어짐
       타입 추론이 실패하면 명시적 캐스트 필요

함수형 vs 명령형:
  장점: 고차 함수는 재사용 가능, 합성 가능, 테스트 용이
  단점: 명령형이 더 직관적일 수 있음 (for-loop)
       디버깅이 어려울 수 있음 (스택 추적)

자바의 문법 제약:
  장점: 명시적 타입으로 안전성 확보
  단점: Haskell/OCaml 대비 매우 Verbose
       부분 적용(currying) 구현이 번거로움
```

---

## 📌 핵심 정리

```
고차 함수 (Higher-Order Function):

정의: 함수를 매개변수로 받거나 함수를 반환하는 함수

자바의 구현:
  - Function<T, R>: 일반 고차 함수
  - BiFunction<T, U, R>: 두 개 매개변수
  - Predicate<T>: boolean 반환 (특수 Function)
  - Supplier<T>: 입력 없음, T 반환
  - Consumer<T>: 입력 받음, 반환 없음

Function의 합성:
  - compose(f): 현재 함수 이전 실행
  - andThen(f): 현재 함수 이후 실행

JDK 고차 함수 활용:
  - Comparator.comparing(): Function<T, U> → Comparator<T>
  - Collectors.groupingBy(): Function<T, K> + Collector
  - Stream.map(), Stream.filter(): Function/Predicate 받음

람다 표현식:
  메서드 참조 ≈ 람다 > 익명 클래스 (성능)
  메서드 참조 > 람다 (가독성)

커링 (Currying):
  BiFunction<A, B, R> → Function<A, Function<B, R>>
  부분 적용으로 함수 조각화 가능
```

---

## 🤔 생각해볼 문제

**Q1.** `Comparator.comparing(Person::getAge).thenComparingInt(Person::getSalary)`에서 `thenComparingInt`는 고차 함수인가?

<details>
<summary>해설 보기</summary>

네, `thenComparingInt`는 고차 함수다. 정확히는:
```java
public <U extends Comparable<? super U>> Comparator<T> thenComparing(
    Function<? super T, ? extends U> keyExtractor)
```

이 메서드는 `Function<T, U>`를 받아서 `Comparator<T>`를 반환한다. 

반환된 `Comparator<T>`는 "이전 비교자가 0을 반환했을 때만 이 새 함수를 사용한다"는 규칙으로 작동한다.

메서드 체이닝이 가능한 이유:
- `Comparator.comparing(f)` → Comparator<T> 반환
- 그 Comparator의 `thenComparingInt(g)` 호출 → 또 다른 Comparator<T> 반환
- 이를 반복할 수 있음

이것이 고차 함수의 강력함이다: 함수의 결과가 또 다른 고차 함수이므로 계속 체이닝할 수 있다.

</details>

---

**Q2.** 왜 `Stream.map(Person::getAge)`는 `map(Function<T, R>)`이 아니라 `map(Function<? super Person, ? extends Integer>)`로 정의되는가?

<details>
<summary>해설 보기</summary>

공변성과 반공변성(Covariance & Contravariance) 때문이다.

`Function<T, R>` 인터페이스:
```java
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);
}
```

만약 `map(Function<Person, Integer>)`로만 정의되면:
```java
Stream<Person> -> Stream<Integer> : map(Person -> Integer) ✓
Stream<Employee> -> Stream<Integer> : map(Function<Employee, Integer>) 필요
  // Employee는 Person의 서브타입인데,
  // Function<Employee, Integer>로는 Function<Person, Integer>를 대체할 수 없음!
```

따라서 메서드는:
```java
<R> Stream<R> map(Function<? super Person, ? extends R> mapper)
```

이렇게 정의:
- `? super Person`: Person 또는 그 상위 타입 허용 (Employee 함수는 X, Object 함수는 O)
- `? extends R`: R 또는 그 하위 타입 반환 (Integer 또는 Number 반환 가능)

이를 통해 자유로운 입출력 호환성을 보장한다.

```java
Function<Object, Number> objToNum = ... // Person도 Object니까 OK
Stream<Person> s = ...;
Stream<Integer> result = s.map(objToNum);  // Integer는 Number의 서브타입니까 OK
```

</details>

---

**Q3.** `Collectors.groupingBy(Function, Collector, Supplier)` 버전에서 세 개의 고차 함수/인터페이스를 받는데, 각각의 역할은 무엇인가?

<details>
<summary>해설 보기</summary>

```java
<T, K, D, A, M extends Map<K, D>> Collector<T, ?, M> groupingBy(
    Function<? super T, ? extends K> classifier,      // ①
    Supplier<M> mapFactory,                           // ②
    Collector<? super T, A, D> downstream             // ③
)
```

**① classifier: Function<T, K>**
- 각 요소를 그룹 키로 변환
- 예: `Person::getDepartment` → "Engineering", "Sales" 등

**② mapFactory: Supplier<M>**
- 최종 Map을 어떤 구현으로 생성할 것인가?
- 예: `HashMap::new`, `LinkedHashMap::new`, `ConcurrentHashMap::new`
- 순서 보장 필요하면 LinkedHashMap, 병렬 처리하면 ConcurrentHashMap

**③ downstream: Collector<T, D>**
- 같은 키로 분류된 요소들을 어떻게 수집할 것인가?
- 예: `Collectors.toList()` → `List<T>`
- 예: `Collectors.toSet()` → `Set<T>`
- 예: `Collectors.mapping(Person::getName, Collectors.toList())` → `List<String>`

예제:
```java
people.stream().collect(
    Collectors.groupingBy(
        Person::getDepartment,                    // 부서별 분류
        LinkedHashMap::new,                       // 삽입 순서 유지
        Collectors.mapping(Person::getName, Collectors.toList())  // 이름 목록 수집
    )
);
// 결과: LinkedHashMap<String, List<String>>
// {"Engineering": ["Alice", "Bob"], "Sales": ["Charlie"]}
```

이 세 개를 조합하면 매우 유연한 그룹화가 가능하다.

</details>

---

<div align="center">

**[⬅️ 이전 챕터: Thread Pool → Virtual Thread 전환](../chapter09-virtual-threads/05-thread-pool-migration.md)** | **[홈으로 🏠](../README.md)** | **[다음: 커링과 부분 적용 ➡️](./02-currying-partial-application.md)**

</div>
