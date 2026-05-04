# @FunctionalInterface와 SAM 변환 메커니즘

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Single Abstract Method(SAM)의 정의는 정확히 무엇인가?
- `default` 메서드와 `static` 메서드가 SAM 카운팅에서 제외되는 이유는?
- `@FunctionalInterface` 어노테이션이 컴파일 타임에 어떤 검증을 수행하는가?
- 왜 `Comparable`은 SAM인데도 함수형 인터페이스로 쓸 수 없는가?
- SAM 변환 시 컴파일러가 어떤 합성 클래스/메서드를 만드는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

람다 표현식은 함수형 인터페이스로만 변환 가능하다. 이 제약이 왜 존재하는지, 어떤 인터페이스가 함수형이고 어떤 것이 아닌지 정확히 이해하면, 올바른 설계의 인터페이스를 만들 수 있다. 또한 JDK의 `Function`, `Consumer`, `Predicate` 같은 표준 함수형 인터페이스들이 왜 이런 이름과 시그니처를 가졌는지도 이해할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Comparable을 람다로 변환할 수 있다고 생각
  // 이건 SAM인데 왜 안 되지?
  interface Comparable<T> {
      int compareTo(T o);  // 추상 메서드 1개
  }
  
  // 람다로 변환 시도 → 컴파일 에러
  Comparable<String> comp = (s) -> s.length();
  // Target type Comparable<String> is not a functional interface
  
  → 이유: Object의 public 메서드(equals, hashCode, toString 등)도 포함되어야 함

실수 2: default 메서드가 SAM을 깨진다고 생각
  interface MyInterface {
      void process(String s);  // 추상 메서드
      default void log() { System.out.println("log"); }  // default
  }
  
  // SAM이 깨진 줄 알았는데...
  MyInterface impl = s -> System.out.println(s);  // 문제없음!
  → default 메서드는 SAM 카운팅에 포함 안 됨

실수 3: @FunctionalInterface 어노테이션만으로 함수형이 된다고 생각
  @FunctionalInterface
  interface BadInterface {
      void method1();
      void method2();  // 추상 메서드 2개!
  }
  → 컴파일 에러: @FunctionalInterface 검증 실패

실수 4: Object 메서드를 override해도 여전히 SAM
  interface MyInterface {
      void process(String s);
      @Override
      String toString();  // Object 메서드 override
  }
  // 이것도 여전히 SAM (Object 메서드는 카운팅 안 됨)
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 함수형 인터페이스를 올바르게 설계

// 패턴 1: 표준 함수형 인터페이스 사용 (권장)
Function<String, Integer> getLength = String::length;
Consumer<String> print = System.out::println;
Predicate<Integer> isEven = n -> n % 2 == 0;

// 패턴 2: 커스텀 함수형 인터페이스는 @FunctionalInterface 명시
@FunctionalInterface
interface LongProcessor {
    long process(List<Integer> nums);  // SAM
    
    // default 메서드는 자유
    default void validate() {
        System.out.println("Validating...");
    }
    
    // static 메서드도 자유
    static LongProcessor identity() {
        return nums -> nums.stream().mapToLong(Integer::longValue).sum();
    }
}

// 사용
LongProcessor sum = nums -> nums.stream().mapToLong(Integer::longValue).sum();
LongProcessor product = nums -> nums.stream().mapToLong(Integer::longValue).reduce(1, (a, b) -> a * b);

// 패턴 3: Object 메서드 override 포함해도 SAM
@FunctionalInterface
interface DocumentProcessor {
    void process(String doc);
    
    @Override
    String toString();  // Object 메서드 override → SAM 유지
    
    @Override
    boolean equals(Object obj);  // 여전히 SAM
}

// 패턴 4: Multiple inheritance of abstract methods는 여전히 SAM
@FunctionalInterface
interface A {
    void method1();
}

@FunctionalInterface
interface B extends A {
    // A의 method1() 상속 → SAM
}

// 패턴 5: 커스텀 함수형 인터페이스로 도메인 표현력 증가
@FunctionalInterface
interface DataValidator {
    boolean validate(String data);
    
    default boolean validateAndLog(String data) {
        boolean valid = validate(data);
        System.out.println("Validation: " + (valid ? "OK" : "FAIL"));
        return valid;
    }
    
    static DataValidator notEmpty() {
        return data -> !data.trim().isEmpty();
    }
    
    static DataValidator minLength(int len) {
        return data -> data.length() >= len;
    }
}

// 사용 - 도메인 언어처럼 표현 가능
DataValidator emailValidator = DataValidator.notEmpty()
    .and(data -> data.contains("@"));
```

---

## 🔬 내부 동작 원리

### 1. SAM(Single Abstract Method) 정의와 카운팅 규칙

```
SAM 검증 알고리즘 (JLS 9.8):

함수형 인터페이스는:
  1) 인터페이스여야 함 (클래스 아님)
  2) 정확히 1개의 추상 메서드를 가짐
  3) Object의 public 메서드는 제외
  4) default 메서드는 제외
  5) static 메서드는 제외
  6) private 메서드는 제외

추상 메서드 카운팅 예시:

┌─────────────────────────────────────────────────┐
│ interface Comparator<T> {                       │
│   int compare(T o1, T o2);      // 추상 ✓       │
│   boolean equals(Object obj);   // Object ✗     │
│   default reverse() { ... }     // default ✗    │
│   static naturalOrder() { ... } // static ✗     │
│ }                                               │
│ → SAM: compare() 1개 ✓                         │
│ → 함수형 인터페이스 ✓                           │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│ interface Comparable<T> {                       │
│   int compareTo(T o);           // 추상 ✓       │
│   equals(Object obj);           // Object ✗     │
│ }                                               │
│ → SAM: compareTo() 1개 ✓                       │
│ → 함수형 인터페이스인가? ...                    │
│                                                 │
│ 그런데 왜 람다로 변환 불가?                    │
│ → Comparable은 값 의미론(identity)을 함께 제공
│   값 비교만 하는 람다로는 의미상 불충분        │
│ → JLS는 기술적으로 SAM 조건을 만족하지만,     │
│   실제 함수형 인터페이스로 취급하지 않음       │
│   (메서드 오버로드 해석 모호성 방지)           │
└─────────────────────────────────────────────────┘

추상 메서드가 2개 이상이면 NOT SAM:

┌─────────────────────────────────────────────────┐
│ interface BiConsumer<T, U> {                    │
│   void accept(T t, U u);        // 추상 ✓       │
│ }                                               │
│ → SAM ✓                                        │
│                                                 │
│ interface Closeable {                           │
│   void close();                 // 추상 ✓       │
│ }                                               │
│ → SAM ✓                                        │
│                                                 │
│ interface AutoCloseable {                       │
│   void close();                 // 추상 ✓       │
│ }                                               │
│ → SAM ✓                                        │
│                                                 │
│ interface WithDefault {                         │
│   void process();               // 추상 ✓       │
│   default void cleanup() { ... }  // default    │
│ }                                               │
│ → SAM ✓                                        │
│                                                 │
│ interface Multiple {                            │
│   void process();               // 추상 ✓       │
│   void finalize();              // 추상 ✓       │
│ }                                               │
│ → NOT SAM ✗ (2개 추상 메서드)                 │
└─────────────────────────────────────────────────┘
```

### 2. @FunctionalInterface 어노테이션의 검증

```java
// @FunctionalInterface 정의 (JDK 소스)
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface FunctionalInterface {
    // 표시용 어노테이션 (런타임 정보 제공)
}

// 컴파일러의 검증 동작:

// 케이스 1: 올바른 함수형 인터페이스
@FunctionalInterface
interface GoodInterface {
    void process(String s);  // SAM ✓
    
    // 컴파일러 검증:
    // ✓ 인터페이스인가? YES
    // ✓ 추상 메서드 1개인가? YES
    // ✓ Object 메서드 제외하면 1개인가? YES
    // → 컴파일 성공
}

// 케이스 2: 추상 메서드 2개
@FunctionalInterface
interface BadInterface1 {
    void process(String s);
    void finalize();
    
    // 컴파일 에러:
    // Multiple non-overriding abstract methods
    // found in interface BadInterface1
}

// 케이스 3: 어노테이션 없이도 함수형
interface ImplicitFunctional {
    void process(String s);  // SAM
}
Runnable r = () -> { };  // @FunctionalInterface 없어도 동작
// 어노테이션은 검증용 (명시적 의도 표현)

// 케이스 4: 런타임 정보 활용
@FunctionalInterface
interface Validator {
    boolean isValid(String input);
}

// 런타임에 확인 가능
Validator v = input -> !input.isEmpty();
Class<?> cls = v.getClass();
if (cls.isAnnotationPresent(FunctionalInterface.class)) {
    System.out.println("This is a functional interface");
}
```

### 3. SAM 변환의 바이트코드 생성 과정

```
Java 소스:
  @FunctionalInterface
  interface Processor {
      int process(String s);
  }
  
  Processor p = s -> s.length();

컴파일러의 처리:

Step 1: 함수형 인터페이스 검증
  Processor:
    - 인터페이스? YES
    - 추상 메서드? process(String) 1개 ✓
    - SAM? YES ✓

Step 2: 람다식을 SAM에 매핑
  Lambda: (s) -> s.length()
  SAM: process(String s) : int
  매핑:
    - 매개변수: s (String)
    - 반환값: s.length() (int)
    - 타입 일치 ✓

Step 3: 람다 본체 메서드 생성
  private static int lambda$0(String s) {
      return s.length();
  }

Step 4: invokedynamic 명령어 생성
  invokedynamic #25,  0  // apply:()Ljava/util/function/Processor;
  
  BootstrapMethods:
    LambdaMetafactory.metafactory(
      lookup,
      "process",
      MethodType(Processor),          // 대상 함수형 인터페이스
      MethodType(String → int),       // SAM 시그니처
      MethodHandle(lambda$0),         // 구현체
      MethodType(String → int)        // 인스턴트화 타입
    )

Step 5: 런타임 합성
  invokedynamic 실행 시:
    → LambdaMetafactory가 Processor 구현체 동적 생성
    → lambda$0을 invoke하도록 구현
    → CallSite에 캐싱
```

### 4. 람다 vs Comparable 비교

```
Comparable<T>:
  abstract int compareTo(T o);
  (Object의 public 메서드 제외)
  → 기술적으로 SAM ✓

그런데 왜 함수형이 아닌가?

이유 1: 의미론적 혼동
  Comparable comp = (a) -> a.hashCode();  // ???
  comp.compareTo(other);  // 반환값이 hashCode?
  → 의미상 말이 안 됨

이유 2: 메서드 오버로드 해석
  sort(Comparator<T> c);        // 함수형 인터페이스
  sort(Comparable<T> target);   // 함수형이 아님
  
  // 만약 Comparable도 함수형이면:
  list.sort((a) -> a.hashCode());  // Comparable로 해석?
  list.sort(String::compareTo);     // Comparator로 해석?
  → 컴파일러가 어떻게 구분할지 불명확

이유 3: 설계 관례
  JLS는 명시적으로:
  "SAM을 만족하더라도, 
   특정 인터페이스(Comparable, Serializable 등)는
   함수형이 아니다"
  
  이는 고의적인 설계 선택이다.

따라서:
  Comparable<T> {
    int compareTo(T o);  // SAM인데
  }
  Comparable comp = (x) -> 0;  // 컴파일 에러
  
  대신:
  Comparator<T> {
    int compare(T o1, T o2);  // SAM
  }
  Comparator comp = (a, b) -> a.compareTo(b);  // OK
```

---

## 💻 실전 실험

### 실험 1: @FunctionalInterface 검증

```java
// ValidFunctionalInterface.java

@FunctionalInterface
interface CountProcessor {
    long countMatches(String[] items, String pattern);
    
    default void report() {
        System.out.println("Processing...");
    }
    
    static CountProcessor create() {
        return (items, pattern) -> 
            java.util.Arrays.stream(items)
                .filter(item -> item.contains(pattern))
                .count();
    }
}

public class FunctionalInterfaceTest {
    public static void main(String[] args) {
        // 컴파일 성공: default와 static은 SAM 카운팅 제외
        CountProcessor proc = (items, pattern) -> 
            java.util.Arrays.stream(items)
                .filter(item -> item.contains(pattern))
                .count();
        
        String[] words = {"apple", "apricot", "banana"};
        long count = proc.countMatches(words, "ap");
        System.out.println("Matches: " + count);  // 2
        
        proc.report();  // default 메서드 호출
    }
}
```

컴파일:
```bash
javac FunctionalInterfaceTest.java
# 성공 - SAM 조건 만족, default와 static은 제외됨
```

### 실험 2: 잘못된 함수형 인터페이스 감지

```java
// BadFunctionalInterface.java

// 에러 케이스 1: 추상 메서드 2개
@FunctionalInterface
interface Multiple {
    void process(String s);
    void finalize();
}
// 컴파일 에러:
// Multiple non-overriding abstract methods found
// in interface Multiple

// 에러 케이스 2: 추상 메서드 없음
@FunctionalInterface
interface Empty {
    default void log() { }
}
// 컴파일 에러:
// No abstract method found in interface Empty

// 올바른 케이스: Object 메서드는 제외
@FunctionalInterface
interface WithObjectMethod {
    void process();
    @Override
    String toString();  // Object 메서드 → 제외
    @Override
    boolean equals(Object obj);  // 제외
}
// 컴파일 성공 - process()만 SAM
```

### 실험 3: javap로 SAM 메서드 확인

```java
// MyFunctionalInterface.java
@FunctionalInterface
interface Calculator {
    int calculate(int a, int b);
    
    default void reset() { }
    
    static Calculator add() {
        return (a, b) -> a + b;
    }
}

public class SamAnalysis {
    public static void main(String[] args) {
        Calculator calc = (a, b) -> a * b;
    }
}
```

```bash
javap -c -v -p MyFunctionalInterface.class | grep -A 5 "methods:"
```

출력:
```
public abstract int calculate(int, int);
public void reset();
public static Calculator add();
```

결과:
- calculate: public abstract (SAM) ✓
- reset: public (default → 구현됨) ✓
- add: public static ✓

---

## 📊 성능/비교

```
함수형 인터페이스 분류별 특성:

인터페이스         | SAM       | 함수형  | 용도
──────────────────┼──────────┼────────┼──────────────
Comparable<T>     | YES      | NO     | 값 비교 + 정렬
Comparator<T>     | YES      | YES    | 유연한 정렬
Function<T, R>    | YES      | YES    | T → R 변환
Consumer<T>       | YES      | YES    | T → void (부작용)
Supplier<T>       | YES      | YES    | () → T
Predicate<T>      | YES      | YES    | T → boolean
Runnable          | YES      | YES    | () → void

@FunctionalInterface 어노테이션 검증 비용:
  - 컴파일 타임에만 발생
  - 런타임: 영향 없음
  - 문서화 + 검증용 (성능 무관)

SAM 변환 성능:
  - 람다: invokedynamic (CallSite 캐싱) ~1-2ns/호출
  - 익명 클래스: 메서드 호출 ~1-2ns/호출
  - 메서드 참조: 약간 더 최적화 가능 (~1ns/호출)
  → 실제로는 거의 차이 없음 (마이크로초 이하)
```

---

## ⚖️ 트레이드오프

```
@FunctionalInterface 사용 결정:

사용해야 하는 경우:
  ✓ 설계한 인터페이스가 함수형 인터페이스일 때
  ✓ 명시적으로 의도를 표현하고 싶을 때
  ✓ 컴파일 시간에 검증받고 싶을 때 (실수 방지)
  ✓ 팀 규칙으로 정했을 때

사용하지 않아도 되는 경우:
  ✓ 기존 코드(Object, Comparable 등)
  ✓ 런타임 동적 생성 인터페이스
  → 어노테이션 없어도 SAM이면 함수형 인터페이스

권장:
  항상 @FunctionalInterface를 명시적으로 붙이자
  → 의도 명확화, 실수 방지, 문서화
```

---

## 📌 핵심 정리

```
SAM (Single Abstract Method):
  - 정확히 1개의 추상 메서드
  - Object 메서드 제외
  - default / static / private 메서드 제외

함수형 인터페이스:
  - SAM을 만족하면서
  - JLS가 허용하는 인터페이스
  - Comparable 같은 예외 있음

@FunctionalInterface:
  - 컴파일 타임 검증
  - 런타임 메타데이터
  - 의도 명시화 (선택사항이지만 권장)

SAM 변환:
  - javac가 람다를 SAM과 매핑
  - lambda$N 메서드 생성
  - invokedynamic 명령어 발행
  - LambdaMetafactory로 런타임 합성

최적화:
  - 캐시된 invokedynamic는 매우 빠름
  - 메서드 참조가 약간 더 최적화 가능
  - 익명 클래스와 비슷한 성능
```

---

## 🤔 생각해볼 문제

**Q1.** SAM 인터페이스 A를 확장한 B가 A의 메서드를 오버라이드하면, B도 여전히 SAM인가?

<details>
<summary>해설 보기</summary>

네, 여전히 SAM이다.

```java
@FunctionalInterface
interface A {
    void process(String s);
}

@FunctionalInterface
interface B extends A {
    // A의 process 상속 → SAM
}

// 올바른 오버라이드 시그니처
@FunctionalInterface
interface C extends A {
    @Override
    void process(String s);  // 재선언하지만 새로운 추상 메서드 아님
    // 결과: 여전히 SAM (추상 메서드는 process 1개)
}
```

오버라이드는 새로운 추상 메서드를 추가하지 않으므로, SAM 조건을 유지한다.

</details>

---

**Q2.** 특정 인터페이스(예: Comparable)가 기술적으로 SAM인데 함수형이 아닌 이유는 설계 의도일 뿐인가? 아니면 다른 기술적 이유가 있는가?

<details>
<summary>해설 보기</summary>

주로 설계 의도이지만, 실제 기술적 문제도 있다:

1. **의미론적 차이**
   - Comparable: 객체 자신의 값을 비교 (자신이 주체)
   - Comparator: 두 객체를 외부에서 비교 (비교 규칙이 주체)
   - 람다로 쓰면 의미상 혼동 (Comparable의 의도 상실)

2. **메서드 오버로드 모호성**
   ```java
   sort(Comparable<T>);  // Comparable 인자
   sort(Comparator<T>);  // Comparator 인자
   
   // 만약 Comparable이 함수형이면:
   sort((a) -> a.something());  // 어느 쪽으로 해석?
   ```

3. **JLS 명시적 제외**
   - JLS 9.8: "Some interfaces are excluded from being functional interfaces"
   - 이는 고의적 설계, 근거는 위의 이유들

따라서 SAM 조건만으로는 함수형이 아니다. JLS가 명시적으로 인정해야 한다.

</details>

---

**Q3.** Object 메서드를 override할 때, 어떤 메서드들은 카운팅 제외인가?

<details>
<summary>해설 보기</summary>

Object의 **public 추상이 아닌** 메서드들만 제외된다:

```java
Object의 메서드들:
  public boolean equals(Object obj);     // 제외 (public)
  public int hashCode();                 // 제외 (public)
  public String toString();              // 제외 (public)
  protected Object clone();              // 포함 (protected)
  protected void finalize();             // 포함 (protected)
  // 등등

인터페이스에서 Override:
  @FunctionalInterface
  interface Custom {
      void process();               // SAM ✓
      @Override
      boolean equals(Object obj);   // Object public → 제외
      // → 여전히 SAM (process 1개만 카운팅)
  }
```

규칙:
- Object의 public 메서드: 자동 카운팅 제외 (모든 객체가 상속)
- Object의 protected 메서드: 포함 (override해야 카운팅에 포함 가능)

대부분의 경우 Object public 메서드(equals, hashCode, toString)만 override하므로, 이들은 모두 제외된다.

</details>

---

<div align="center">

**[⬅️ 이전: Lambda → invokedynamic 변환](./01-lambda-to-invokedynamic.md)** | **[홈으로 🏠](../README.md)** | **[다음: Method Reference 4가지 ➡️](./03-method-reference-4-types.md)**

</div>
