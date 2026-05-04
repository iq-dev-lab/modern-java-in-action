# 다이아몬드 상속 해결 규칙 — Class > Interface, Sub > Super

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 두 인터페이스가 같은 default method를 제공할 때 왜 컴파일 에러가 발생하는가?
- "클래스 > 인터페이스" 규칙이 적용되는 정확한 경우는?
- "더 구체적인 인터페이스 우선"이라는 규칙은 무엇인가?
- 세 개 이상의 인터페이스 상속 시 우선도는 어떻게 계산되는가?
- Diamond Problem을 Java가 해결하지 않고 프로그래머에게 강제하는 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Java의 인터페이스는 여러 개를 동시에 구현할 수 있기 때문에, C++의 "Diamond Problem"과 유사한 상황이 자주 발생한다. Default Method를 도입하면서 이 문제가 더욱 심화되었다. 실제로 `Runnable`과 `Callable` 같은 인터페이스가 충돌할 때, 또는 복잡한 라이브러리 의존성에서 같은 default method를 여러 경로로 상속할 때, 명확한 우선 규칙을 모르면 예상 밖의 컴파일 에러나 잘못된 메서드 호출이 발생한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 같은 default method를 가진 두 인터페이스를 구현할 수 있다고 착각
  interface Runnable {
      default void execute() { System.out.println("Runnable"); }
  }
  interface Callable {
      default void execute() { System.out.println("Callable"); }
  }
  
  class Task implements Runnable, Callable {
      // 컴파일 에러!
      // 이 클래스는 execute() 메서드를 상속받지만 어느 것일지 모호함
  }
  
  // "두 인터페이스 모두 implements만 하면 되겠지?"
  // → 실제로는 명시적 오버라이드 필수

실수 2: 인터페이스 상속 체계를 무시
  interface Base { default void process() { print("Base"); } }
  interface Extended extends Base { default void process() { print("Extended"); } }
  
  class Impl implements Base, Extended {
      // Extended.process()가 우선 (더 구체적)
      // 하지만 프로그래머가 모르면 혼란 발생
  }

실수 3: 세 개 이상의 인터페이스에서 선택할 때
  interface I1 { default void m() { print("I1"); } }
  interface I2 extends I1 { default void m() { print("I2"); } }
  interface I3 extends I2 { default void m() { print("I3"); } }
  
  class C implements I1, I2, I3 {
      // 세 개 모두 같은 메서드를 가지는데?
      // 규칙: I3 (가장 구체적) > I2 > I1
      // 상속 관계로 결정됨
  }
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 올바른 패턴 1: 충돌 해결을 위한 명시적 오버라이드
interface Source1 {
    default String getData() { return "Source1"; }
}

interface Source2 {
    default String getData() { return "Source2"; }
}

class DataProvider implements Source1, Source2 {
    @Override
    public String getData() {
        // 명시적으로 선택: Source1 우선
        return Source1.super.getData();
    }
}

// 올바른 패턴 2: 인터페이스 상속으로 명확한 우선도 설정
interface Reader {
    default void read() { System.out.println("Generic read"); }
}

interface BufferedReader extends Reader {
    // 더 구체적인 구현 제공
    @Override
    default void read() { System.out.println("Buffered read"); }
}

class FileReader implements Reader, BufferedReader {
    // BufferedReader가 Reader를 확장하므로 BufferedReader.read() 자동 선택
    // 명시적 오버라이드 불필요
}

// 올바른 패턴 3: 클래스 구현이 모든 인터페이스보다 우선
interface Processor {
    default void process() { System.out.println("Interface default"); }
}

class RealProcessor implements Processor {
    @Override
    public void process() {
        System.out.println("Class implementation");
    }
}

// RealProcessor의 process()가 항상 호출됨
RealProcessor proc = new RealProcessor();
proc.process();  // "Class implementation"

Processor ref = proc;
ref.process();  // "Class implementation" (구현체 우선)
```

---

## 🔬 내부 동작 원리

### 1. Diamond Problem의 정의

```
        ┌─────────────┐
        │   Base      │
        │ void m(){}  │
        └──────┬──────┘
               │
        ┌──────┴──────┐
        │             │
    ┌───┴──┐      ┌───┴──┐
    │ Left │      │Right │
    │void  │      │void  │
    │ m(){}│      │ m(){}│
    └───┬──┘      └───┬──┘
        │             │
        └──────┬──────┘
               │
        ┌──────┴──────┐
        │  Diamond    │
        │  (충돌!)     │
        └─────────────┘

문제: Diamond가 Base로부터 m()을 두 경로로 상속받음
해결 방안:
  C++: 가상 상속 (virtual inheritance)
  Java: 명시적 오버라이드 강제 또는 상속 관계로 결정
```

### 2. 메서드 해석 규칙 (Method Resolution)

```
Java의 default method 해석 순서:

1단계: 클래스 구현 확인
  class C implements I1, I2, I3 {
      public void m() { ... }  ← 이것이 호출됨
  }
  규칙 1: 클래스의 메서드 > 모든 인터페이스의 default method

2단계: 단일 인터페이스 (충돌 없음)
  interface I {
      default void m() { ... }
  }
  class C implements I { }
  // I.m() 자동 상속

3단계: 다중 인터페이스 (충돌 발생)
  interface I1 { default void m() { print("I1"); } }
  interface I2 { default void m() { print("I2"); } }
  class C implements I1, I2 { }
  
  컴파일 에러: ambiguous method
  해결: 반드시 명시적 오버라이드
  class C implements I1, I2 {
      public void m() {
          I1.super.m();  // 또는 I2.super.m()
      }
  }

4단계: 상속 관계 (우선도 결정)
  interface I1 { default void m() { print("I1"); } }
  interface I2 extends I1 { default void m() { print("I2"); } }
  class C implements I1, I2 { }
  
  I2가 I1을 확장하므로 I2.m() 자동 선택
  규칙 2: 더 구체적인 인터페이스 > 부모 인터페이스
  명시적 오버라이드 불필요
```

### 3. "더 구체적인" 정의

```
인터페이스 상속 관계로 "더 구체적" 판단:

상황 1: 직접 상속
  interface I1 { ... }
  interface I2 extends I1 { ... }
  → I2가 I1보다 구체적
  
상황 2: 전이적 상속
  interface I1 { default void m() { } }
  interface I2 extends I1 { default void m() { } }
  interface I3 extends I2 { default void m() { } }
  
  class C implements I1, I2, I3 {
      // I3 > I2 > I1 우선순위
      // I3.m() 자동 선택
  }

상황 3: 평행 상속 (무관한 인터페이스)
  interface I1 { default void m() { } }
  interface I2 { default void m() { } }
  
  class C implements I1, I2 {
      // I1과 I2 사이에 상속 관계 없음
      // → 컴파일 에러, 모호함
  }

상황 4: 클래스 상속이 있을 때
  class Parent { void m() { } }
  interface I { default void m() { } }
  
  class C extends Parent implements I {
      // Parent.m() 자동 선택
      // 규칙: 클래스 > 인터페이스 항상
  }
```

### 4. 복잡한 다이아몬드 해결

```
실제 시나리오:

    Base (void m() {print("Base")})
      │
  ┌───┴───┐
  │       │
 Sub1    Sub2 (오버라이드: void m() {print("Sub2")})
  │       │
  └───┬───┘
      │
    Child

규칙 적용:
  1. Base와 Sub1, Sub2가 모두 m()을 가짐
  2. Sub2가 m()을 오버라이드함 (Sub2 > Sub1)
  3. Sub1과 Sub2 간에는 상속 관계 없음
  4. → Sub2.m()을 선택

코드:
  interface Base { default void m() { print("Base"); } }
  interface Sub1 extends Base { }
  interface Sub2 extends Base { default void m() { print("Sub2"); } }
  
  class Child implements Sub1, Sub2 {
      // Sub2.m()이 자동으로 선택됨
      // Sub2가 m()을 명시적으로 오버라이드했으므로 더 구체적
  }
```

### 5. 명시적 super 호출의 메커니즘

```java
interface I1 { default void m() { print("I1"); } }
interface I2 { default void m() { print("I2"); } }

class C implements I1, I2 {
    @Override
    public void m() {
        // invokespecial로 컴파일됨
        I1.super.m();
        I2.super.m();
    }
}

바이트코드:
  0: aload_0
  1: invokespecial I1.m()V
  4: aload_0
  5: invokespecial I2.m()V
  8: return

invokespecial의 의미:
  - 정적 바인딩 (런타임 다형성 없음)
  - I1.super.m() → 컴파일 시점에 I1의 default method로 고정
  - 런타임에 실제 타입을 무시하고 I1의 구현 호출
```

---

## 💻 실전 실험

### 실험 1: 충돌 감지

```java
interface I1 { default void test() { System.out.println("I1"); } }
interface I2 { default void test() { System.out.println("I2"); } }

// 컴파일 에러: ambiguous method 'test' inherited...
class Conflict implements I1, I2 { }

// 해결
class Fixed implements I1, I2 {
    @Override
    public void test() {
        I1.super.test();
    }
}

public class DiamondTest {
    public static void main(String[] args) {
        Fixed f = new Fixed();
        f.test();  // I1 출력
    }
}
```

### 실험 2: 상속 우선도 확인

```java
interface Animal { default void sound() { System.out.println("Generic"); } }
interface Dog extends Animal { default void sound() { System.out.println("Woof"); } }
interface Cat extends Animal { default void sound() { System.out.println("Meow"); } }

class Hybrid implements Dog, Cat {
    // Dog와 Cat 모두 sound()를 가지지만 서로 무관
    // 컴파일 에러: 모호함
}

class HybridFixed implements Dog, Cat {
    @Override
    public void sound() {
        Dog.super.sound();  // Dog 선택
    }
}

// 비교: 상속 체계가 명확한 경우
interface Vehicle { default void move() { System.out.println("Vehicle"); } }
interface Car extends Vehicle { default void move() { System.out.println("Car"); } }
interface ElectricCar extends Car { }

class Tesla implements Vehicle, Car, ElectricCar {
    // ElectricCar → Car → Vehicle 상속 체계
    // Car.move() 자동 선택 (명시적 오버라이드 불필요)
}
```

### 실험 3: 클래스 우선도 확인

```java
interface Service { default void serve() { System.out.println("Interface"); } }
abstract class BaseService { public void serve() { System.out.println("Abstract"); } }

class Impl extends BaseService implements Service {
    // BaseService.serve()가 호출됨 (클래스 > 인터페이스)
}

public class PriorityTest {
    public static void main(String[] args) {
        Impl impl = new Impl();
        impl.serve();  // "Abstract" 출력
        
        Service ref = impl;
        ref.serve();  // "Abstract" (여전히 클래스 메서드)
    }
}
```

---

## 📊 성능/비교

```
메서드 해석 비용 분석:

상황 1: 클래스 메서드 (최빠름)
  invokevirtual 사용 → 메서드 테이블 직접 인덱싱
  성능: ~3ns (JIT 인라인 후)

상황 2: 단일 인터페이스 default
  invokeinterface 사용 → 동적 디스패치
  성능: ~3ns (JIT 인라인 후)

상황 3: 다중 인터페이스, 상속 관계 명확
  컴파일 시점에 우선도 결정 → invokeinterface
  성능: ~3ns (JIT 인라인 후)

상황 4: super.method() 호출
  invokespecial 사용 → 정적 바인딩
  성능: ~3ns (항상 인라인 가능)

컴파일 비용:
  메서드 검색: O(상속 깊이 × 인터페이스 수)
  일반적으로 O(1)~O(N) (N은 인터페이스 수, 최대 ~10개)

결론:
  해석 규칙이 복잡하지만 성능 영향은 미미
  컴파일 시점에 해석되므로 런타임 오버헤드 없음
```

---

## ⚖️ 트레이드오프

```
Diamond Problem 해결 방식 비교:

C++ 접근 (가상 상속):
  + 컴파일러가 자동 해결
  - 메모리 레이아웃 복잡 (vtable 증가)
  - 성능 오버헤드
  - 초기화 순서 혼란

Java 접근 (명시적 오버라이드):
  + 명확한 의도 표현
  + 성능 오버헤드 없음
  + 메모리 효율적
  - 컴파일 에러 강제 (귀찮음)
  - 프로그래머가 직접 처리

Java의 철학:
  "Fail-fast": 컴파일 시점에 모호함을 강제로 해결
  "명시적이 암시적보다 낫다": 어느 메서드가 호출될지 명확

트레이드오프 분석:
  안전성 vs 편의성:
    Java 선택: 안전성 (컴파일 에러는 안전)
    대가: 프로그래머가 명시적 해결 필요

  단순성 vs 유연성:
    Java 선택: 유연성 (명시적 super 호출로 선택 가능)
    대가: 규칙의 복잡성
```

---

## 📌 핵심 정리

```
Diamond Problem 해결 규칙:

규칙 1: 클래스 > 인터페이스
  class C extends Parent implements I1, I2 {
      // Parent의 메서드가 I1, I2의 default method보다 우선
  }

규칙 2: 더 구체적인 인터페이스 > 부모 인터페이스
  interface I1 { default void m() { } }
  interface I2 extends I1 { default void m() { } }
  class C implements I1, I2 {
      // I2.m() 자동 선택 (상속 관계로 I2가 I1보다 구체적)
  }

규칙 3: 무관한 인터페이스는 컴파일 에러
  interface I1 { default void m() { } }
  interface I2 { default void m() { } }
  class C implements I1, I2 {
      // 컴파일 에러! 명시적 해결 필수
      public void m() {
          I1.super.m();  // 선택
      }
  }

규칙 4: 선언 순서는 무관 (상속 관계가 중요)
  class C implements I2, I1 {
      // I1 I2 순서와 관계없이 상속 관계로 결정
  }

메커니즘:
  - 컴파일 시점에 모든 해석 (런타임 비용 없음)
  - invokespecial (명시적 super) / invokeinterface (일반 호출)
  - 모호함 발견 → 컴파일 에러 (fail-fast)
```

---

## 🤔 생각해볼 문제

**Q1.** 왜 Java는 "더 구체적인 인터페이스"의 메서드를 자동으로 선택하는가?

<details>
<summary>해설 보기</summary>

이것은 "분명한 의도"를 존중하는 설계다. 인터페이스 `I2`가 `I1`을 확장하고 동일한 메서드를 오버라이드한다는 것은:

1. 설계자가 명시적으로 `I2`의 버전을 사용하려고 했다는 뜻
2. `I2`가 `I1`의 기능을 "더 구체적으로" 구현했다는 뜻

만약 Java가 선언 순서(`implements I1, I2` 순서)를 따르거나, 선택을 강제했다면:
- 프로그래머가 인터페이스 상속 구조를 이해해야 함
- 재귀적 상속 관계에서 혼란 증가

반대로 "더 구체적인 것 자동 선택"하면:
- 상속 관계가 명확히 표현됨
- 프로그래머의 의도가 코드에 반영됨
- 대부분의 경우 명시적 오버라이드 불필요

</details>

---

**Q2.** 클래스 메서드가 인터페이스 default method보다 항상 우선인 이유는?

<details>
<summary>해설 보기</summary>

클래스는 인터페이스보다 "구체적"하기 때문이다:

1. **의도의 명확성**
   ```java
   class MyList extends AbstractList implements Collection {
       public void add(E e) { ... }  // 클래스 고유 구현
   }
   ```
   클래스에서 메서드를 직접 구현한 것은 "이 구현이 정확하다"는 뜻

2. **하위 호환성**
   Java 8 이전에 이미 존재하던 클래스들:
   ```java
   class ArrayList extends AbstractList {  // Collection은 인터페이스
       public void add(E e) { ... }  // 기존 구현
   }
   ```
   Java 8에서 `Collection`에 새로운 default method를 추가했을 때, `ArrayList`의 기존 구현이 무시되면 안 된다.

3. **상속 체계 존중**
   클래스 상속은 "강한" 계약(contract)이고, 인터페이스 구현은 "약한" 계약이다. 강한 계약이 우선되어야 한다.

```java
class Parent { void m() { print("Parent"); } }
class Child extends Parent implements Interface {  // Interface.m() default 있음
}
// Parent.m()이 호출됨 (클래스 상속이 구현보다 우선)
```

</details>

---

**Q3.** 만약 세 인터페이스 `I1 extends I2`, `I2 extends I3`, `I3 { default void m() }`이고, 클래스가 `implements I1, I2, I3`이면 어떤 메서드가 호출되는가?

<details>
<summary>해설 보기</summary>

**I1의 메서드**가 호출된다.

이유:
1. I1, I2, I3 모두 같은 `m()` 메서드를 가짐 (상속으로)
2. 하지만 I1이 "가장 구체적" (I1 → I2 → I3 순서)
3. 선언 순서 무관 (상속 관계로 결정)

코드:
```java
interface I3 { default void m() { print("I3"); } }
interface I2 extends I3 { }
interface I1 extends I2 { }

class C implements I1, I2, I3 {
    // I1.m() 호출 (I1이 가장 구체적)
}

C c = new C();
c.m();  // "I3" 출력 (실제로는 I3의 구현이지만 I1에서 상속)
```

우선도:
- I1 > I2 > I3 (상속 깊이가 깊을수록 구체적)

명시적 오버라이드 불필요:
```java
class C implements I1, I2, I3 {
    // I1이 자동 선택되므로 컴파일 에러 없음
}
```

만약 I1과 I3가 평행하다면 (무관):
```java
interface I1 { default void m() { print("I1"); } }
interface I3 { default void m() { print("I3"); } }
interface I2 extends I1, I3 { }  // I2가 둘 다 구현

class C implements I1, I2, I3 {
    // I2가 I1과 I3보다 구체적 → I2.m() 선택
    // I2 내에서도 충돌이 있다면 I2에서 명시적으로 해결했을 것
}
```

</details>

---

<div align="center">

**[⬅️ 이전: Default Method 동작 원리](./01-default-method-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: Private Interface Method ➡️](./03-private-interface-method.md)**

</div>
