# Default Method 동작 원리 — invokespecial vs invokevirtual

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Default Method 호출 시 정확히 어떤 바이트코드 명령어가 생성되는가?
- 왜 `invokeinterface`를 사용하고, 구현체 메서드가 있으면 자동으로 우선되는가?
- `Interface.super.method()` 문법이 `invokespecial`로 컴파일되는 이유는?
- 메서드 해석(method resolution) 순서가 왜 중요한가?
- Default Method 도입이 ABI 호환성 문제를 어떻게 해결하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Java 8에서 Default Method를 도입한 이유는 기존 인터페이스에 새 메서드를 추가할 때 모든 구현체를 수정할 필요를 없애기 위함이다. 예를 들어, `Collection` 인터페이스에 `stream()`을 추가할 때, 수백 개의 구현체(`ArrayList`, `TreeSet`, 등)를 모두 수정할 수 없었다. Default Method는 이 "부서지기 쉬운 라이브러리 문제"(fragile base class problem)를 해결하지만, 호출 순서와 `super` 참조 방식을 잘못 이해하면 의외의 동작을 초래할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Default Method는 항상 인터페이스에서만 호출된다고 착각
  interface MyInterface {
      default void hello() { System.out.println("Interface"); }
  }
  class MyClass implements MyInterface {
      public void hello() { System.out.println("Class"); }
  }
  
  MyInterface ref = new MyClass();
  ref.hello();  // 결과: "Class" (구현체 메서드 우선!)
  
  // 오해: "default는 인터페이스에서 호출될 것"
  // 실제: 클래스 메서드가 있으면 항상 그것을 호출

실수 2: super 참조 시 부모 클래스의 super처럼 사용
  interface I1 { default void test() { System.out.println("I1"); } }
  interface I2 extends I1 { default void test() { System.out.println("I2"); } }
  class C implements I2 {
      public void test() {
          super.test();  // 부모 클래스의 test() 호출 (없음)
          // → 컴파일 에러: "super"는 클래스 문맥에서만 의미 있음
      }
  }
  
  // 올바른 방식: I2.super.test() 또는 I1.super.test()

실수 3: Default Method 검색 순서를 잘못 이해
  interface I1 { default void m() { System.out.println("I1"); } }
  interface I2 { default void m() { System.out.println("I2"); } }
  
  // 컴파일 에러: 두 인터페이스가 같은 default method를 가짐
  // 반드시 명시적으로 오버라이드해야 함
  class C implements I1, I2 {
      public void m() {
          I1.super.m();  // I1의 default method 호출
      }
  }
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 올바른 패턴 1: 구현체 메서드가 우선됨을 이해
interface Comparable<T> {
    default int compareTo(T o) { 
        return 0;  // 기본 구현
    }
}

class Person implements Comparable<Person> {
    private String name;
    
    // 이 메서드가 호출됨 (interface의 default 메서드는 건너뜀)
    @Override
    public int compareTo(Person o) {
        return this.name.compareTo(o.name);
    }
}

// 올바른 패턴 2: 다이아몬드 상속에서 명시적 super 호출
interface Base { default void log() { System.out.println("Base"); } }
interface Left extends Base { default void log() { System.out.println("Left"); } }
interface Right extends Base { default void log() { System.out.println("Right"); } }

class Diamond implements Left, Right {
    @Override
    public void log() {
        // 명시적으로 어느 인터페이스의 default method를 호출할지 지정
        Left.super.log();  // "Left" 출력
        Right.super.log(); // "Right" 출력
    }
}

// 올바른 패턴 3: Default Method로 기존 코드 호환성 유지
interface Collection<E> {
    // Java 8 이전부터 있는 메서드
    boolean add(E e);
    
    // Java 8에서 추가된 default method
    default Stream<E> stream() {
        return StreamSupport.stream(this.spliterator(), false);
    }
    
    default void forEach(Consumer<? super E> action) {
        for (E e : this) {
            action.accept(e);
        }
    }
}
```

---

## 🔬 내부 동작 원리

### 1. invokeinterface 바이트코드

```
Interface 참조를 통한 호출은 invokeinterface 명령어 사용:

자바 코드:
  interface MyInterface {
      default void greet() {
          System.out.println("Hello from interface");
      }
  }
  
  MyInterface obj = new MyImpl();
  obj.greet();

바이트코드:
  aload_1                          // obj 로드
  invokeinterface MyInterface.greet ()V, 2
                                   // 명령어, 메서드 개수(2)
                                   // obj(1) + 메서드(1)
  return
```

### 2. 메서드 검색 순서 (Method Resolution Order)

```
default method 호출 시 Java는 다음 순서로 메서드 탐색:

1단계: 객체의 실제 클래스에서 메서드 검색
  class C implements I { public void m() { ... } }
  C obj = new C();
  obj.m();  // C.m() 호출 (인터페이스 default 무시)

2단계: 클래스가 구현한 인터페이스를 선언 순서로 탐색
  class C implements I1, I2, I3 { }
  // I1.m(), I2.m(), I3.m() 순서로 탐색
  // I1에서 찾으면 중단

3단계: 인터페이스 상속 체계를 따라 탐색
  interface I2 extends I1 { default void m() {...} }
  // I2.m()을 먼저 검사, 없으면 I1.m() 검사

예시:
  interface I1 { default void m() { print("I1"); } }
  interface I2 extends I1 { default void m() { print("I2"); } }
  interface I3 { default void m() { print("I3"); } }
  
  class C implements I2, I3 { }
  C obj = new C();
  obj.m();  // I2.m() 호출 (I3는 탐색되지 않음)

충돌 발생:
  interface I1 { default void m() { print("I1"); } }
  interface I2 { default void m() { print("I2"); } }
  
  class C implements I1, I2 { }
  // 컴파일 에러!
  // 명시적으로 오버라이드 필요:
  class C implements I1, I2 {
      public void m() {
          I1.super.m();  // 또는 I2.super.m()
      }
  }
```

### 3. invokespecial로 컴파일되는 super 호출

```
자바 코드:
  class C implements I1, I2 {
      public void test() {
          I1.super.test();  // 명시적 super 호출
      }
  }

바이트코드:
  aload_0                          // this 로드
  invokespecial I1.test ()V
                                   // invokespecial: 정적 바인딩!
                                   // I1의 default method 호출 보장
  return

invokeinterface vs invokespecial:
  invokeinterface:
    - 동적 디스패치 (런타임에 실제 메서드 결정)
    - 메서드 탐색 순서 적용
    - 접수자(receiver)의 실제 타입 기반
  
  invokespecial:
    - 정적 바인딩 (컴파일 시점에 메서드 결정)
    - I1.super.test() → I1의 default method로 고정
    - 다이아몬드 상속에서 명확하게 어느 인터페이스의 메서드인지 지정
```

### 4. 상속 체계에서의 우선 규칙

```
복잡한 다중 상속 시 우선 규칙:

클래스 계층:
  class Parent { void m() { print("Parent"); } }
  class C extends Parent implements I { }
  
  C obj = new C();
  obj.m();  // Parent.m() 호출 (인터페이스 default 무시)
  
  규칙 1: 클래스는 항상 인터페이스보다 우선
  → 추상 클래스든 구체 클래스든 관계없음

인터페이스 선택 규칙:
  interface I1 { default void m() { } }
  interface I2 extends I1 { default void m() { } }
  
  class C implements I1, I2 { }
  // I2가 I1을 확장하므로 I2.m()이 우선 (더 구체적)
  
  class D implements I2, I1 { }
  // 선언 순서 무관, 상속 관계로 결정: I2.m()
```

### 5. 구현체에서 default method 오버라이드

```java
// 자바 코드
interface Collection<E> {
    default Stream<E> stream() {
        return StreamSupport.stream(this.spliterator(), false);
    }
}

class ArrayList<E> implements Collection<E> {
    // stream()을 오버라이드 (최적화)
    @Override
    public Stream<E> stream() {
        return super.stream();  // Object.super로 기본 구현 호출 가능
        // 또는 자체 최적화 구현 제공
    }
}

// 바이트코드에서 ArrayList.stream()이 호출됨
// invokeinterface Collection.stream() → 실제 수신자(ArrayList) 결정
// → ArrayList.stream() 메서드 테이블 항목 확인 → 존재 → 호출
```

---

## 💻 실전 실험

### 실험 1: 메서드 검색 순서 확인

```java
interface I1 { default void show() { System.out.println("I1"); } }
interface I2 extends I1 { default void show() { System.out.println("I2"); } }
interface I3 { default void show() { System.out.println("I3"); } }

class Impl implements I2, I3 {
    // 명시적 오버라이드 필요 (I2와 I3가 동일한 default method)
    @Override
    public void show() {
        I2.super.show();  // I2를 선택
    }
}

public class DefaultMethodTest {
    public static void main(String[] args) {
        Impl impl = new Impl();
        impl.show();  // 출력: I2 (I2가 I1을 확장하므로 더 구체적)
        
        I2 ref2 = impl;
        ref2.show();  // 출력: I2 (실제 클래스 메서드 호출)
    }
}
```

### 실험 2: invokeinterface vs invokespecial 비교

```bash
# Impl.java 컴파일
javac -d . Impl.java

# 바이트코드 확인
javap -c Impl.class

# 결과:
#  0: aload_0
#  1: invokespecial I2.show()V
#  4: return

# I1.super.show() 호출:
#  0: aload_0
#  1: invokespecial I1.show()V
#  4: return
```

### 실험 3: 클래스 메서드 vs 인터페이스 default 메서드

```java
interface Greetable {
    default void greet() {
        System.out.println("Default: Hello from interface");
    }
}

class Person implements Greetable {
    @Override
    public void greet() {
        System.out.println("Overridden: Hello from class");
    }
}

class NoOverride implements Greetable {
    // default method 사용
}

public class Test {
    public static void main(String[] args) {
        Person p = new Person();
        p.greet();  // "Overridden: Hello from class"
        
        Greetable g1 = p;
        g1.greet();  // "Overridden: Hello from class"
        
        NoOverride no = new NoOverride();
        no.greet();  // "Default: Hello from interface"
        
        Greetable g2 = no;
        g2.greet();  // "Default: Hello from interface"
    }
}
```

---

## 📊 성능/비교

```
invokeinterface 성능 특성:

1. 런타임 메서드 검색 비용
   invokeinterface:
     - 인터페이스 메서드 테이블 인덱싱: O(1)
     - 실제 구현체 메서드 테이블에서 동일 슬롯 검색: O(1)
     - 일회성 비용 (JIT가 인라인 캐시 구성)

2. JIT 최적화 후 성능
   처음 호출: invokeinterface 동적 디스패치
   100회 이상: JIT가 receiver type 단일화 감지 → 인라인
   → 최종적으로 가상 호출과 동일 성능

3. 다형성이 높은 경우
   interface 참조가 다양한 구현체를 가리키면:
     invokeinterface: 인라인 캐시 invalidation 빈번
     결과: 성능 저하 (메서드 테이블 검색 반복)

비교:
  메서드 종류        | 첫 호출 비용 | 100회 후 | JIT 인라인 가능
  ────────────────┼────────────┼──────────┼────────────────
  invokevirtual   | ~50ns      | ~3ns     | 예
  invokeinterface | ~100ns     | ~3ns     | 예 (단형성)
  invokespecial   | ~30ns      | ~3ns     | 예 (항상)
  invokestatic    | ~20ns      | ~3ns     | 예
```

---

## ⚖️ 트레이드오프

```
Default Method 도입의 트레이드오프:

장점:
  ✓ 기존 인터페이스에 메서드 추가 가능 (호환성)
  ✓ 라이브러리 진화가 가능 (구현체 수정 불필요)
  ✓ 일반적인 메서드는 하나의 구현만 필요

단점:
  ✗ 메서드 검색 순서가 복잡해짐 (클래스 > 인터페이스)
  ✗ 다중 상속에서 충돌 → 명시적 오버라이드 강제
  ✗ super 참조 문법이 C++과 다름 (Interface.super)
  ✗ 성능: invokeinterface는 invokevirtual보다 초기 비용 높음

호환성 vs 복잡성:
  Default Method 없음:
    - 깔끔한 메서드 해석
    - 새 메서드 추가 시 모든 구현체 수정 필요
  
  Default Method:
    - 라이브러리 진화 가능
    - 메서드 검색 규칙이 더 복잡
    - 상속 관계에 따라 예상 밖의 결과 가능
```

---

## 📌 핵심 정리

```
Default Method의 핵심 메커니즘:

1. invokeinterface 명령어
   - 인터페이스 참조를 통한 호출
   - 동적 디스패치로 실제 메서드 결정

2. 메서드 검색 순서 (MRO)
   - 클래스 메서드 > 인터페이스 default method
   - 다중 인터페이스: 선언 순서, 확장 관계로 우선도 결정
   - 충돌 시: 명시적 오버라이드 필수

3. super 참조
   - Interface.super.method(): invokespecial로 컴파일
   - 정확히 어느 인터페이스의 메서드인지 명시
   - 다이아몬드 상속 해결 수단

4. 호환성
   - 기존 구현체: default method 자동 상속
   - 새 메서드 추가: 컴파일 에러 없음
   - 의도적 오버라이드: 성능/의미 향상

5. 성능
   - invokeinterface: 초기 비용 높지만 JIT 인라인 가능
   - 단형성 확인 시 invokevirtual 수준으로 최적화
   - 다형성 높으면 인라인 캐시 invalidation 빈번
```

---

## 🤔 생각해볼 문제

**Q1.** 왜 Java는 `super.method()`가 아니라 `Interface.super.method()` 문법을 사용하는가?

<details>
<summary>해설 보기</summary>

단순히 `super.method()`라고 쓰면, 이것이 어느 인터페이스의 메서드인지 알 수 없다. 특히 다이아몬드 상속에서:

```java
interface I1 { default void m() { } }
interface I2 extends I1 { default void m() { } }
interface I3 { default void m() { } }
class C implements I2, I3 {
    public void m() {
        super.m();  // I2.m()? I3.m()? I1.m()?
    }
}
```

모호함을 제거하기 위해 Java는 명시적으로 인터페이스를 지정하도록 강제한다: `I2.super.m()`. 이렇게 하면 `invokespecial`이 정확히 어떤 인터페이스의 메서드를 호출할지 컴파일 시점에 결정할 수 있다.

또한 `super`는 클래스 문맥에서만 의미 있으므로, 인터페이스 내에서 default method가 `super`를 참조하는 것은 혼란을 초래한다. `Interface.super`는 이 문제를 명확히 한다.

</details>

---

**Q2.** `ArrayList` 같은 구현체가 `Collection`의 새로운 default method `stream()`을 자동으로 상속하는데, 성능상 문제는 없는가?

<details>
<summary>해설 보기</summary>

`Collection.stream()`의 기본 구현은 `spliterator()`를 사용한다:

```java
default Stream<E> stream() {
    return StreamSupport.stream(this.spliterator(), false);
}
```

이는 모든 컬렉션에 적용되는 범용 구현이며, 특정 컬렉션에 최적화되어 있지 않다. 따라서:

1. **기존 구현체 (`ArrayList`, `HashSet` 등)**: Java 8에서 자신의 `stream()` 구현을 제공하여 기본 구현을 오버라이드한다. 이를 통해 최적화된 성능을 얻는다.

2. **새로운 구현체**: 만약 Java 8 이후에 `Collection` 인터페이스를 구현하는 클래스를 만든다면, default `stream()` 구현을 상속받는다. 느릴 수 있지만, 최소한 컴파일 에러는 발생하지 않는다.

3. **JIT 최적화**: default method도 일반 메서드처럼 JIT 컴파일되므로, 호출 패턴이 명확하면 인라인화된다.

따라서 호환성과 성능 사이의 적절한 트레이드오프이다.

</details>

---

**Q3.** `Interface I1, I2` 간에 같은 default method가 있을 때 컴파일 에러가 발생하는데, 왜 동일한 메서드 시그니처를 가진 `I1`과 `I2`를 구현하는 것이 금지되는가?

<details>
<summary>해설 보기</summary>

만약 컴파일러가 이를 허용한다면:

```java
interface I1 { default void m() { print("I1"); } }
interface I2 { default void m() { print("I2"); } }
class C implements I1, I2 {
    // 명시적 오버라이드 없음
}
C obj = new C();
obj.m();  // I1.m()? I2.m()? 컴파일러도 선택 불가능
```

이는 **모호성** 때문에 런타임 에러를 초래할 수 있다. Java는 이를 컴파일 시점에 강제로 해결하도록 설계했다:

```java
class C implements I1, I2 {
    @Override
    public void m() {
        I1.super.m();  // 또는 I2.super.m()
    }
}
```

이렇게 명시적으로 해결하면:
1. 프로그래머의 의도가 명확
2. 런타임 모호성 없음
3. `invokespecial`로 정확한 메서드 호출 보장

이는 "fail-fast" 원칙: 컴파일 시점에 문제를 감지하는 것이 더 안전하다.

</details>

---

<div align="center">

**[⬅️ 이전 챕터: commonPool 의존성 문제](../chapter05-completable-future/07-commonpool-dependency.md)** | **[홈으로 🏠](../README.md)** | **[다음: 다이아몬드 상속 해결 규칙 ➡️](./02-diamond-inheritance-rules.md)**

</div>
