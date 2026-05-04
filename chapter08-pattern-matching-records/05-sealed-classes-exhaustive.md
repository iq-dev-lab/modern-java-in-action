# Sealed Classes (Java 17) — Exhaustive Pattern Matching

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `sealed class Shape permits Circle, Square, Triangle`이 switch expression의 exhaustive 검사를 가능하게 하는 메커니즘은?
- 모든 자식 케이스를 다루지 않으면 컴파일 에러가 발생하는 보장은 어떻게 구현되는가?
- ADT(Algebraic Data Type) 표현으로 `default` 없이 안전한 분기가 가능한 이유는?
- Non-sealed 자식 vs sealed 자식의 차이는?
- Sealed 계층 구조의 바이트코드 정보 저장 방식은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

상태 머신, 도메인 모델링, AST(추상 구문 트리) 표현에서 **닫힌 집합의 타입**을 정의할 필요가 빈번하다. Sealed class는 컴파일러가 타입 안전성을 강제하므로, 새로운 자식 타입이 추가될 때 모든 switch 문이 자동으로 컴파일 에러를 보낸다. 이것은 리팩토링 안전성을 높이고 실제로 ADT 패턴을 Java에 구현할 수 있게 해준다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: sealed를 선언했는데도 default가 필요하다고 생각
  sealed class Shape permits Circle, Square { }
  
  String area = switch (shape) {
      case Circle c -> "π r²";
      case Square s -> "s²";
      default -> "unknown";  // ❌ 불필요 (Circle, Square만 가능)
  };

실수 2: Sealed 계층이 깊어도 한 레벨만 커버하면 된다고 착각
  sealed class Shape permits Circle, Rectangle { }
  sealed class Circle extends Shape permits FilledCircle, EmptyCircle { }
  
  String desc = switch (shape) {
      case FilledCircle fc -> "filled";
      case EmptyCircle ec -> "empty";
      case Rectangle r -> "rect";
      // ❌ Circle을 구현한 다른 자식이 있으면 실패
  };

실수 3: Sealed의 자식을 더 추가할 수 있다고 가정
  sealed class Animal permits Cat, Dog { }
  
  public class Bird extends Animal { }  // ❌ 컴파일 에러
  // Bird는 permits 목록에 없음 → 상속 불가

실수 4: Non-sealed 자식의 의미를 모름
  sealed class Payment permits CreditCard, Cash, NonSealedPayment { }
  non-sealed class NonSealedPayment extends Payment { }
  
  public class Bitcoin extends NonSealedPayment { }  // ⚠️ 가능
  // NonSealedPayment가 non-sealed이므로 상속 가능
  // 하지만 switch에서 exhaustiveness 검사가 깨짐

실수 5: Record와 sealed를 혼동
  sealed record Point(int x, int y) permits Point3D { }  // ❌ record는 final
  // record는 이미 상속 불가 (final) → sealed 의미 없음
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 패턴 1: 기본 sealed 계층
sealed class Shape permits Circle, Rectangle, Triangle { }

final class Circle extends Shape {
    double radius;
}

final class Rectangle extends Shape {
    double width, height;
}

final class Triangle extends Shape {
    double a, b, c;
}

// switch expression에서 default 없음 (exhaustive)
String describe(Shape shape) {
    return switch (shape) {
        case Circle c -> "Circle with radius " + c.radius;
        case Rectangle r -> "Rectangle " + r.width + "x" + r.height;
        case Triangle t -> "Triangle";
        // default 불필요 (모든 자식 타입 다룸)
    };
}

// 패턴 2: 인터페이스도 sealed 가능
sealed interface Expression permits Number, BinaryOp, Variable { }

record Number(double value) implements Expression { }
record Variable(String name) implements Expression { }
record BinaryOp(Expression left, String op, Expression right) implements Expression { }

double evaluate(Expression expr) {
    return switch (expr) {
        case Number(double v) -> v;
        case Variable(String name) -> getVariable(name);
        case BinaryOp(Expression left, String op, Expression right) -> {
            double l = evaluate(left);
            double r = evaluate(right);
            yield switch (op) {
                case "+" -> l + r;
                case "-" -> l - r;
                case "*" -> l * r;
                default -> throw new IllegalArgumentException(op);
            };
        }
    };
}

// 패턴 3: Non-sealed 자식 (확장 가능한 포인트)
sealed class Response permits SuccessResponse, ErrorResponse, CustomResponse { }

final class SuccessResponse extends Response {
    Object data;
}

final class ErrorResponse extends Response {
    String message;
}

// CustomResponse는 non-sealed → 사용자가 확장 가능
non-sealed class CustomResponse extends Response {
    // 서드파티가 이것을 상속 가능
}

String handleResponse(Response response) {
    return switch (response) {
        case SuccessResponse s -> "Success: " + s.data;
        case ErrorResponse e -> "Error: " + e.message;
        case CustomResponse c -> "Custom";  // non-sealed이므로 더 있을 수 있음
        default -> "Unknown";  // 필수 (non-sealed 때문)
    };
}

// 패턴 4: ADT 표현 (함수형 스타일)
sealed interface Maybe<T> permits Just, Nothing { }

record Just<T>(T value) implements Maybe<T> { }
class Nothing implements Maybe<Object> { }

<T> String unwrap(Maybe<T> maybe) {
    return switch (maybe) {
        case Just<T> j -> "Just " + j.value;
        case Nothing _ -> "Nothing";
        // default 불필요 (sealed로 exhaustive)
    };
}

// 패턴 5: 깊은 sealed 계층
sealed class BinaryTree<T> permits Node, Leaf { }

final class Leaf<T> extends BinaryTree<T> {
    T value;
}

final class Node<T> extends BinaryTree<T> {
    BinaryTree<T> left, right;
}

<T> int size(BinaryTree<T> tree) {
    return switch (tree) {
        case Leaf<T> l -> 1;
        case Node<T> n -> size(n.left) + size(n.right);
    };
}

// 패턴 6: Enum vs Sealed (비교)
enum Color { RED, GREEN, BLUE }

sealed class DynamicColor permits RedColor, GreenColor, BlueColor { }
final class RedColor extends DynamicColor { }
final class GreenColor extends DynamicColor { }
final class BlueColor extends DynamicColor { }

// enum: 고정 값 (런타임 추가 불가)
// sealed: 컴파일 타임 정의, 런타임 사용자 확장 가능
```

---

## 🔬 내부 동작 원리

### 1. Sealed 제어 메커니즘

```
Sealed 선언:

  sealed class Shape permits Circle, Square { }

컴파일러 검증:
  ① permits 목록의 자식들이 실제로 Shape를 상속하는가?
  ② 자식들이 모두 final 또는 sealed인가?
  ③ Shape와 자식이 같은 모듈/패키지에 있는가?

바이트코드에 저장:

  sealed attribute: Shape의 자식 목록 저장
  
  java.lang.constant.ClassDesc record의 메타데이터:
    Shape.class {
        PermittedSubclasses attribute: [
            Circle,
            Square
        ]
    }

런타임 검증:

  Class.isSealed() → true
  Class.getPermittedSubclasses() → [Circle.class, Square.class]
  
  Class<?> subclass = Class.forName("CircleExtension");
  if (!isPermittedSubclass(subclass)) {
      throw IllegalAccessException("Not permitted");
  }
```

### 2. Exhaustiveness 검사 메커니즘

```
컴파일러의 exhaustiveness 검사:

switch (shape) {
    case Circle c -> ...;
    case Square s -> ...;
    case Triangle t -> ...;
}

컴파일러 로직:

  1. shape의 타입: Shape (sealed)
  2. Shape의 permits 자식: [Circle, Square, Triangle]
  3. 모든 case를 매핑:
     - case Circle: ✅ 커버
     - case Square: ✅ 커버
     - case Triangle: ✅ 커버
  4. permits 목록을 모두 다뤘는가? 네 → exhaustive ✅
  5. default 필요 없음

불완전한 switch:

switch (shape) {
    case Circle c -> ...;
    case Square s -> ...;
    // Triangle 누락
}

컴파일러 로직:

  1. shape 타입: Shape (sealed)
  2. permits 자식: [Circle, Square, Triangle]
  3. 커버된 case: Circle, Square
  4. 누락된 case: Triangle ❌
  5. 컴파일 에러: "switch expression does not cover all possible input values"

또는 default 추가로 해결:

switch (shape) {
    case Circle c -> ...;
    case Square s -> ...;
    default -> ...;  // 모든 나머지 경우 (Triangle 포함)
}
```

### 3. Non-sealed 자식의 의미

```
Sealed 계층:

  sealed class Shape permits Circle, NonSealed { }
  
  final class Circle extends Shape { }  // sealed 계층 종료
  
  non-sealed class NonSealed extends Shape { }  // 계층 계속 열음

Non-sealed의 의미:

  public class Bitcoin extends NonSealed { }  // ✅ 가능
  public class Ethereum extends NonSealed { }  // ✅ 가능
  
  → NonSealed를 상속한 자식이 몇 개인지 알 수 없음
  → Shape를 구현한 타입도 불확정

Exhaustiveness 검사:

switch (shape) {
    case Circle c -> ...;
    case NonSealed ns -> ...;  // ns에 몇 개 자식이 있는지 모름
}

컴파일러:
  → NonSealed 자식이 미지의 다른 타입일 수 있음
  → exhaustive라고 보장 불가
  → default 필수

switch (shape) {
    case Circle c -> ...;
    case NonSealed ns -> ...;
    default -> ...;  // 필수
}
```

### 4. Sealed Record (가장 일반적)

```
Record는 암묵적으로 final:

  record Circle(double radius) { }
  // final class Circle extends Record { ... }

Sealed + Record:

  sealed interface Shape permits CircleRec, SquareRec { }
  
  record CircleRec(double radius) implements Shape { }
  record SquareRec(double side) implements Shape { }

이점:
  ① record의 불변성 + sealed의 exhaustiveness
  ② ADT의 대수적 구조 표현
  ③ 패턴 매칭 + switch expression 결합

바이트코드:

  CircleRec.class:
    - implements Shape
    - record 생성 코드 (canonical constructor, accessors, equals/hashCode/toString)
    - sealed 메타데이터 (Shape의 PermittedSubclasses에 포함)

```

### 5. ADT(대수적 데이터 타입) 구현

```
ADT 개념 (함수형 언어):

  Scala:
    sealed trait Tree[+A]
    case class Leaf[A](value: A) extends Tree[A]
    case class Node[A](left: Tree[A], right: Tree[A]) extends Tree[A]
    
    tree match {
        case Leaf(v) => ...
        case Node(l, r) => ...
    }

Java (sealed + record):

  sealed interface Tree<T> permits Leaf, Node { }
  
  record Leaf<T>(T value) implements Tree<T> { }
  record Node<T>(Tree<T> left, Tree<T> right) implements Tree<T> { }
  
  static <T> int size(Tree<T> tree) {
      return switch (tree) {
          case Leaf<T> l -> 1;
          case Node<T> n -> size(n.left()) + size(n.right());
      };
  }

구현 비교:

  Scala: pattern match (exhaustive 강제)
  Java: switch expression (exhaustive 선택사항, sealed로 가능)
  
  Java sealed = ADT의 합 타입(sum type) 표현
  Java record = ADT의 곱 타입(product type) 표현
```

---

## 💻 실전 실험

### 실험 1: 기본 sealed 계층

```java
sealed interface HttpResponse permits SuccessResponse, ErrorResponse, RedirectResponse { }

record SuccessResponse(int code, String body) implements HttpResponse { }
record ErrorResponse(int code, String message) implements HttpResponse { }
record RedirectResponse(int code, String location) implements HttpResponse { }

public class SealedExhaustiveTest {
    public static void main(String[] args) {
        HttpResponse response = new SuccessResponse(200, "OK");
        
        // default 없이 exhaustive
        String result = switch (response) {
            case SuccessResponse s -> "Success: " + s.body;
            case ErrorResponse e -> "Error: " + e.message;
            case RedirectResponse r -> "Redirect to: " + r.location;
        };
        
        System.out.println(result);
    }
}
```

### 실험 2: Sealed 계층 리플렉션

```java
import java.util.Arrays;

sealed interface Shape permits Circle, Rectangle { }
record Circle(double radius) implements Shape { }
record Rectangle(double width, double height) implements Shape { }

public class SealedReflectionTest {
    public static void main(String[] args) {
        Class<?> shapeClass = Shape.class;
        
        System.out.println("Is sealed: " + shapeClass.isSealed());
        
        if (shapeClass.isSealed()) {
            Class<?>[] permitted = shapeClass.getPermittedSubclasses();
            System.out.println("Permitted subclasses:");
            for (Class<?> clazz : permitted) {
                System.out.println("  - " + clazz.getSimpleName());
            }
        }
    }
}

// 출력:
// Is sealed: true
// Permitted subclasses:
//   - Circle
//   - Rectangle
```

### 실험 3: Non-sealed 자식

```java
sealed interface Response permits Success, NonSealedError { }

record Success(Object data) implements Response { }

non-sealed class NonSealedError implements Response {
    String message;
    public NonSealedError(String message) { this.message = message; }
}

// 사용자가 NonSealedError를 상속 확장 가능
class CustomError extends NonSealedError {
    public CustomError(String message, String code) {
        super(message);
    }
}

public class NonSealedTest {
    public static void main(String[] args) {
        Response response = new Success("data");
        
        // NonSealedError가 non-sealed이므로 default 필수
        String result = switch (response) {
            case Success s -> "Success: " + s.data;
            case NonSealedError e -> "Error: " + e.message;
            default -> "Other";  // 필수 (non-sealed 자식 가능성)
        };
        
        System.out.println(result);
    }
}
```

### 실험 4: ADT 패턴 (식 계산기)

```java
sealed interface Expr permits NumExpr, AddExpr, MulExpr { }

record NumExpr(int value) implements Expr { }
record AddExpr(Expr left, Expr right) implements Expr { }
record MulExpr(Expr left, Expr right) implements Expr { }

public class ADTCalculatorTest {
    static int evaluate(Expr expr) {
        return switch (expr) {
            case NumExpr n -> n.value;
            case AddExpr a -> evaluate(a.left) + evaluate(a.right);
            case MulExpr m -> evaluate(m.left) * evaluate(m.right);
        };
    }
    
    static String toString(Expr expr) {
        return switch (expr) {
            case NumExpr n -> String.valueOf(n.value);
            case AddExpr a -> "(" + toString(a.left) + " + " + toString(a.right) + ")";
            case MulExpr m -> "(" + toString(m.left) + " * " + toString(m.right) + ")";
        };
    }
    
    public static void main(String[] args) {
        // (3 + 4) * 5 = 35
        Expr expr = new MulExpr(
            new AddExpr(new NumExpr(3), new NumExpr(4)),
            new NumExpr(5)
        );
        
        System.out.println("Expression: " + toString(expr));     // (3 + 4) * 5
        System.out.println("Result: " + evaluate(expr));         // 35
    }
}
```

---

## 📊 성능/비교

```
Sealed vs Open 계층:

특성                    | Open Hierarchy  | Sealed Class
───────────────────────┼────────────────┼─────────────────────
상속 제약                | 없음 (누구든)   | permits 목록만
Exhaustiveness 검사      | 필수 default   | 선택 (sealed 계층 내)
컴파일 검증              | 약함           | 강함
런타임 오버헤드          | 없음           | 거의 없음 (~바이트)
바이트코드 크기          | 작음           | 같음 (메타데이터 추가)
JVM 최적화               | 어려움         | 쉬움 (타입 범위 제한)

Sealed 메타데이터 크기:

  Attribute: PermittedSubclasses
  크기: 약 8 + (자식 수 × 2) 바이트
  
  10개 자식: ~28 바이트 (무시할 수준)

런타임 성능:

  Open: Object obj = new UnknownSubclass();
        switch (obj) { ... }  // 런타임 타입 디스패치, JVM 최적화 어려움
  
  Sealed: sealed interface X permits A, B, C { }
          switch (obj) { ... }  // JVM이 3가지만 가능, JIT 최적화 가능

  → sealed가 JVM 최적화 여지 더 큼 (단, 현실 성능은 미미)
```

---

## ⚖️ 트레이드오프

```
Sealed Class 도입:

장점:
  - 타입 안전성: permits로 확장 제한
  - Exhaustiveness: switch에서 default 선택 가능
  - 의도 명확: "이 계층은 닫혀있음" 문서화
  - 리팩토링 안전: 새 자식 추가 시 모든 switch 자동 에러
  - ADT 표현: 함수형 패턴 모델링

단점:
  - Java 17 필수: 레거시 코드 미지원
  - permits 목록 관리: 계층이 클수록 복잡
  - 유연성 감소: 확장 포인트 필요 시 non-sealed 혼용
  - 학습곡선: sealed/non-sealed 차이 이해 필요

Non-sealed 혼용 시:

  장점:
    - Controlled extensibility (sealed + non-sealed 조합)
    - 라이브러리 설계 (사용자가 일부만 확장)

  단점:
    - Exhaustiveness 검사 부분적만 가능
    - 코드 복잡도 증가
    - switch에서 default 여전히 필요

```

---

## 📌 핵심 정리

```
Sealed Classes 핵심:

1. 문법:
   sealed class Shape permits Circle, Square { }
   → 오직 Circle, Square만 상속 가능

2. 자식 타입:
   final: sealed 계층 종료
   sealed: 계층 계속 (재제한)
   non-sealed: 열린 확장 (exhaustiveness 검사 제한)

3. Exhaustiveness:
   sealed + 모든 자식 다룸 → default 선택
   non-sealed 있음 → default 필수

4. 바이트코드:
   PermittedSubclasses attribute로 메타데이터 저장
   Class.getPermittedSubclasses() 리플렉션 가능

5. ADT 표현:
   sealed interface + record = 대수적 데이터 타입
   Scala/Haskell 같은 패턴 매칭 표현 가능

6. 리팩토링 안전성:
   새 자식 추가 → 모든 switch 자동 컴파일 에러
   → 누락 방지 (더 안전한 코드)
```

---

## 🤔 생각해볼 문제

**Q1.** Sealed class와 final class의 차이는 무엇인가?

<details>
<summary>해설 보기</summary>

**final class:**
```java
final class Shape { }
public class Circle extends Shape { }  // ❌ 컴파일 에러
```

모든 상속을 금지. 이 클래스는 **그 자체**로 사용되어야 한다.

**sealed class:**
```java
sealed class Shape permits Circle, Square { }
public class Circle extends Shape { }      // ✅ 가능 (permits 목록)
public class Triangle extends Shape { }    // ❌ 컴파일 에러 (목록 없음)
```

**특정 자식**에게만 상속을 허용. 자식들이 정의되는 방식을 제어한다.

차이:
- **final**: "이것을 상속하지 마세요" → 상속 자체 불가
- **sealed**: "이 자식들만 상속 가능" → controlled hierarchy

**sealed의 이점:**
```java
sealed interface Result permits Success, Failure { }

// 컴파일러가 보증: 
// Result는 무조건 Success 또는 Failure
// → switch에서 exhaustive 검사 가능
```

따라서 sealed는 **계층 구조의 의도적 제어**이고, final은 **상속 금지**다.

</details>

---

**Q2.** Non-sealed 자식이 있으면 왜 exhaustiveness 검사가 깨지는가?

<details>
<summary>해설 보기</summary>

```java
sealed class Shape permits Circle, NonSealed { }
final class Circle extends Shape { }
non-sealed class NonSealed extends Shape { }

// 사용자가 정의할 수 있음
public class Triangle extends NonSealed { }
public class Pentagon extends NonSealed { }
```

컴파일러의 관점:

1. **Shape의 자식:**
   - Circle (final) ✅ 확정
   - NonSealed (non-sealed) ⚠️ 미지수

2. **NonSealed의 자식:**
   - Triangle (가능) ✅
   - Pentagon (가능) ✅
   - 사용자가 정의한 다른 자식? (미지) ⚠️

따라서 switch(shape)에서:
```java
switch (shape) {
    case Circle c -> ...;
    case NonSealed ns -> ...;
    // case Triangle은? 자동으로 NonSealed로 감싸짐
    // 다른 NonSealed 자식이 있을 수 있음
}
```

컴파일러:
- Circle → exhaustive
- NonSealed → exhaustive (그 자신과 모든 자식)
- 근데 NonSealed의 자식이 미지 → 보장 불가

따라서 **default 필수**:
```java
switch (shape) {
    case Circle c -> ...;
    case NonSealed ns -> ...;
    default -> ...;  // 미래의 NonSealed 자식 대비
}
```

**핵심:** Non-sealed = "확장 포인트 열려있음" → exhaustiveness 보장 불가 → default 필수

</details>

---

**Q3.** Sealed interface를 record로 구현할 때 왜 유용한가?

<details>
<summary>해설 보기</summary>

**불변성 + Exhaustiveness 조합:**

```java
// Sealed: 계층 구조 제어
sealed interface Payment permits CardPayment, CashPayment, CheckPayment { }

// Record: 불변 + 자동 equals/hashCode/toString
record CardPayment(String cardNumber, double amount) implements Payment { }
record CashPayment(double amount) implements Payment { }
record CheckPayment(String checkNumber, double amount) implements Payment { }
```

**이점:**

1. **ADT 표현 (함수형):**
   ```java
   double process(Payment p) {
       return switch (p) {
           case CardPayment cp -> chargeCard(cp.cardNumber, cp.amount);
           case CashPayment cp -> acceptCash(cp.amount);
           case CheckPayment ckp -> depositCheck(ckp.checkNumber, ckp.amount);
       };
   }
   ```
   - 모든 케이스 강제 (exhaustive)
   - 패턴 매칭으로 값 추출
   - 함수형 언어의 ADT와 유사

2. **불변성:**
   - Payment는 수정될 수 없음
   - 결과 예측 가능
   - 동시성 안전

3. **안전성:**
   - 새 Payment 종류 추가 시 모든 switch 자동 에러
   - 누락 불가능

**vs 레거시 패턴:**
```java
// Visitor pattern (Java 1.0 시대)
interface Payment {
    void accept(PaymentVisitor v);
}

interface PaymentVisitor {
    void visit(CardPayment p);
    void visit(CashPayment p);
}

// 복잡하고 보일러플레이트 많음
// sealed + record가 훨씬 간결
```

따라서 sealed interface + record = **현대 Java의 ADT 구현 표준**

</details>

---

<div align="center">

**[⬅️ 이전: Switch Expression](./04-switch-expression.md)** | **[홈으로 🏠](../README.md)** | **[다음: Record Pattern ➡️](./06-record-pattern-destructuring.md)**

</div>
