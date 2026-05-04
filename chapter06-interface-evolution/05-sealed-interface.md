# Sealed Interface (Java 17) — 상속 봉인과 Pattern Matching 연계

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Sealed interface를 도입한 근본적인 이유는?
- `sealed interface ... permits` 문법으로 구현체를 명시적으로 제한하는 목적은?
- `final`, `sealed`, `non-sealed` 수정자의 역할 차이는?
- Algebraic Data Type (ADT) 표현에서 sealed interface가 왜 필수적인가?
- Switch expression의 exhaustive matching이 sealed interface로 어떻게 보장되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Java 17에서 Sealed Class/Interface를 도입했을 때, 함수형 프로그래밍의 Algebraic Data Type을 Java에서도 표현할 수 있게 되었다. 이전에는 계층 구조를 제한할 방법이 없었기 때문에, `instanceof` 체인이나 리플렉션에 의존했고, 새로운 구현체가 추가될 때마다 switch 문을 수동으로 업데이트해야 했다. Sealed interface와 pattern matching을 조합하면 컴파일러가 모든 경우를 처리했는지 검증할 수 있다. 이를 모르면 런타임에 버그가 발생하거나, 구조가 복잡해진다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 새로운 구현체가 추가되면 모든 switch를 수동으로 수정
  interface Result { }
  class Success implements Result { Object value; }
  class Failure implements Result { String error; }
  
  // Case 1: switch
  switch (result) {
      case Success s -> print(s.value);
      case Failure f -> print(f.error);
  }
  
  // Case 2: 6개월 후 새로운 구현체 추가
  class Pending implements Result { }
  
  // Case 1의 switch는 자동 에러 없음 (컴파일러가 모름!)
  // 런타임에 Pending이 처리되지 않음
  // → NPE 또는 의도하지 않은 동작

실수 2: 누군가 라이브러리의 sealed interface를 위반하려 시도
  sealed interface ServiceProvider { }
  
  // 라이브러리 사용자가 자신만의 구현체를 만들려 함
  class MyService implements ServiceProvider { }  // 컴파일 에러 강제
  // → 혼란 ("왜 안 되지?")

실수 3: sealed vs final의 차이를 모름
  sealed interface I permits A, B {
      // A, B만 구현 가능
  }
  
  class A implements I { }  // 확장 가능
  final class B implements I { }  // 확장 불가
  
  class ChildA extends A { }  // 가능한가?
  // (규칙을 모르면 혼란)

실시 4: Pattern matching의 완전성을 신뢰하지 못함
  sealed interface Payment permits Card, Transfer {
      record Card(String number) implements Payment { }
      record Transfer(String account) implements Payment { }
  }
  
  String process(Payment p) {
      return switch (p) {
          case Card c -> "Card: " + c.number();
          case Transfer t -> "Transfer: " + t.account();
          // 모든 경우를 처리했으므로 default 불필요
      };
  }
  
  // 하지만 switch에 default를 추가하면?
  // (컴파일러가 경고하지 않음 - Java 17 초기 버전)
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 올바른 패턴 1: Sealed interface로 ADT 표현
sealed interface Result<T> permits SuccessResult, ErrorResult {
    <R> R match(Function<T, R> onSuccess, Function<String, R> onError);
}

record SuccessResult<T>(T value) implements Result<T> {
    @Override
    public <R> R match(Function<T, R> onSuccess, Function<String, R> onError) {
        return onSuccess.apply(value);
    }
}

record ErrorResult<T>(String error) implements Result<T> {
    @Override
    public <R> R match(Function<T, R> onSuccess, Function<String, R> onError) {
        return onError.apply(error);
    }
}

// 올바른 패턴 2: Switch expression으로 모든 경우 처리 (exhaustive)
interface Shape {
    double area();
}

sealed interface DrawableShape extends Shape permits Circle, Square, Triangle {
    void draw();
}

record Circle(double radius) implements DrawableShape {
    @Override public double area() { return Math.PI * radius * radius; }
    @Override public void draw() { System.out.println("⭕"); }
}

record Square(double side) implements DrawableShape {
    @Override public double area() { return side * side; }
    @Override public void draw() { System.out.println("□"); }
}

record Triangle(double base, double height) implements DrawableShape {
    @Override public double area() { return base * height / 2; }
    @Override public void draw() { System.out.println("△"); }
}

// 컴파일러가 모든 경우를 확인 (exhaustiveness check)
String describeShape(DrawableShape shape) {
    return switch (shape) {
        case Circle c -> "Circle with radius " + c.radius;
        case Square s -> "Square with side " + s.side;
        case Triangle t -> "Triangle with base " + t.base;
        // default 필요 없음 (sealed이므로 모든 경우 처리)
    };
}

// 올바른 패턴 3: 계층적 sealed interface
sealed interface Number permits Integer, Decimal {
    int intValue();
}

sealed interface Integer extends Number permits SmallInt, LargeInt {
    // 추가 메서드
}

non-sealed class SmallInt implements Integer {
    private int value;
    SmallInt(int value) { this.value = value; }
    @Override public int intValue() { return value; }
}

final class LargeInt implements Integer {
    private long value;
    LargeInt(long value) { this.value = (int)value; }
    @Override public int intValue() { return (int)value; }
}

sealed class Decimal extends Number {
    // 추가적인 제약 가능
}

// 올바른 패턴 4: Pattern matching with sealed interface
sealed interface HttpResponse permits SuccessResponse, ErrorResponse, RedirectResponse {}

record SuccessResponse(int code, String body) implements HttpResponse {}
record ErrorResponse(int code, String message) implements HttpResponse {}
record RedirectResponse(int code, String location) implements HttpResponse {}

void handleResponse(HttpResponse response) {
    switch (response) {
        case SuccessResponse(var code, var body) when code == 200 -> 
            System.out.println("Success: " + body);
        case SuccessResponse(var code, var body) -> 
            System.out.println("Success with code " + code);
        case ErrorResponse(var code, var msg) -> 
            System.err.println("Error " + code + ": " + msg);
        case RedirectResponse(var code, var loc) -> 
            System.out.println("Redirect to " + loc);
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Sealed Interface의 컴파일

```
자바 코드:
  sealed interface Payment permits Card, Transfer {
      record Card(String number) implements Payment {}
      record Transfer(String account) implements Payment {}
  }

바이트코드:
  public sealed interface Payment
    permits 'Card', 'Transfer'
    {
        // 메서드 없음 (sealed 정보만 기록)
    }
  
  // 각 구현체에 sealed 정보 기록:
  public final record Card (String) implements Payment {
      // Payment의 permitted subtypes에만 포함 가능
  }
  
  public final record Transfer (String) implements Payment {
      // Payment의 permitted subtypes에만 포함 가능
  }

클래스 파일 정성:
  Sealed attribute:
    - 클래스 파일에 "Sealed" 속성 추가
    - 허용된 구현체 목록 저장
    - 컴파일러가 이를 읽어 제한 검증

sealed vs non-sealed vs final:

sealed interface Payment permits A, B {
  - A, B만 구현 가능
  - A, B는 sealed / non-sealed / final 가능
}

class A implements Payment {
  - 확장 가능 (non-sealed 기본)
  class ChildA extends A { }  // 가능
}

final class B implements Payment {
  - 확장 불가
  class ChildB extends B { }  // 컴파일 에러
}

non-sealed class C implements Payment {
  - 암시적으로 확장 가능 (sealed interface의 제한을 풀음)
  class ChildC extends C { }  // 가능
}
```

### 2. Pattern Matching의 Exhaustiveness Check

```
Java 17+ Pattern Matching:

sealed interface Result<T> permits Success, Failure {}
record Success<T>(T value) implements Result<T> {}
record Failure<T>(String error) implements Result<T> {}

// 1. Exhaustive switch (모든 경우 처리됨)
String handle(Result<String> result) {
    return switch (result) {
        case Success(var value) -> "OK: " + value;
        case Failure(var error) -> "ERROR: " + error;
        // default 불필요 (sealed이므로 모든 경우 덮임)
    };
}

바이트코드:
  tableswitch / lookupswitch 최적화 가능
  (컴파일러가 모든 경우를 알고 있으므로)

// 2. Non-sealed로 확장하면 exhaustiveness 깨짐
non-sealed class PendingResult<T> implements Result<T> {
    // 새 구현체!
}

String handle(Result<String> result) {
    return switch (result) {
        case Success(var value) -> "OK: " + value;
        case Failure(var error) -> "ERROR: " + error;
        // 컴파일 에러: PendingResult 미처리
        // 해결: default 추가 또는 case PendingResult 추가
    };
}

// 3. Pattern matching with guard
sealed interface Operation permits Add, Subtract, Multiply {}
record Add(int a, int b) implements Operation {}
record Subtract(int a, int b) implements Operation {}
record Multiply(int a, int b) implements Operation {}

int evaluate(Operation op) {
    return switch (op) {
        case Add(var a, var b) when a > 0 && b > 0 -> a + b;
        case Add(var a, var b) -> -(a + b);
        case Subtract(var a, var b) -> a - b;
        case Multiply(var a, var b) -> a * b;
    };
}
```

### 3. Sealed Hierarchy와 Type System

```
Sealed interface 계층:

    ┌─────────────────┐
    │   Expression    │ (sealed)
    │   permits:      │
    │   - Number      │
    │   - BinOp       │
    │   - Var         │
    └────────┬────────┘
             │
    ┌────────┴──────────────────┬──────────────┐
    │                           │              │
┌───┴──┐              ┌────────┴─┐        ┌───┴──┐
│Number│              │ BinOp    │        │ Var  │
│      │              │          │        │      │
│sealed│              │sealed    │        │final │
│      │              │          │        │      │
└───┬──┘              └────┬─────┘        └──────┘
    │                      │
    │        ┌─────────────┼─────────────┐
    │        │             │             │
┌───┴──┐ ┌──┴──┐      ┌───┴──┐     ┌───┴──┐
│IntNum│ │Float│      │ Plus │     │Minus │
│final │ │final│      │final │     │final │
└──────┘ └─────┘      └──────┘     └──────┘

Type narrowing:
  Expression expr = ...;
  
  switch (expr) {
      case Number n -> n.value();  // Number로 타입 좁혀짐
      case BinOp b -> b.op();      // BinOp로 타입 좁혀짐
      case Var v -> v.name();      // Var로 타입 좁혀짐
  }
  
바이트코드:
  checkcast Number
  aload_0
  invokevirtual Number.value()
  
  (컴파일러가 타입 캐스트를 자동 삽입)
```

### 4. Record와 Sealed의 조합

```java
// Java 17: sealed interface + record (강력한 조합)
sealed interface Expr permits ConstExpr, BinExpr, NegExpr {
    int eval();
}

// 불변 상수 표현
record ConstExpr(int value) implements Expr {
    @Override public int eval() { return value; }
}

// 불변 이항 연산 표현
record BinExpr(Expr left, String op, Expr right) implements Expr {
    @Override public int eval() {
        return switch (op) {
            case "+" -> left.eval() + right.eval();
            case "-" -> left.eval() - right.eval();
            case "*" -> left.eval() * right.eval();
            case "/" -> left.eval() / right.eval();
            default -> throw new IllegalArgumentException();
        };
    }
}

// 불변 부정 표현
record NegExpr(Expr expr) implements Expr {
    @Override public int eval() { return -expr.eval(); }
}

// 패턴 매칭으로 계산
int compute(Expr e) {
    return switch (e) {
        case ConstExpr(int v) -> v;
        case BinExpr(var l, var op, var r) -> 
            evaluateBinOp(l.eval(), op, r.eval());
        case NegExpr(var ex) -> -ex.eval();
    };
}

특징:
  - 불변성 보장 (record)
  - 계약 강제 (sealed)
  - 패턴 매칭 (exhaustiveness check)
  - 함수형 표현 (ADT)
```

---

## 💻 실전 실험

### 실험 1: 기본 Sealed Interface

```java
sealed interface Status permits Active, Inactive, Suspended {}

record Active(long since) implements Status {}
record Inactive(long since) implements Status {}
record Suspended(String reason) implements Status {}

public class SealedTest {
    static String describeStatus(Status status) {
        return switch (status) {
            case Active(long since) -> "Active since " + since;
            case Inactive(long since) -> "Inactive since " + since;
            case Suspended(String reason) -> "Suspended: " + reason;
            // default 필요 없음!
        };
    }
    
    public static void main(String[] args) {
        Status s1 = new Active(System.currentTimeMillis());
        Status s2 = new Suspended("Maintenance");
        
        System.out.println(describeStatus(s1));
        System.out.println(describeStatus(s2));
    }
}
```

### 실험 2: Exhaustiveness Check

```java
sealed interface Event permits ClickEvent, SubmitEvent, ValidateEvent {}

record ClickEvent(int x, int y) implements Event {}
record SubmitEvent(String formId) implements Event {}
record ValidateEvent(String fieldId) implements Event {}

public class ExhaustiveTest {
    static void handleEvent(Event event) {
        // 모든 경우를 처리하지 않으면 컴파일 에러
        switch (event) {
            case ClickEvent(int x, int y) -> 
                System.out.println("Clicked at " + x + "," + y);
            case SubmitEvent(String id) -> 
                System.out.println("Submitted form " + id);
            // case ValidateEvent 빠짐!
            // 컴파일 에러: "the switch expression does not cover all possible input values"
        }
    }
}
```

### 실험 3: Pattern Matching with Guard

```java
sealed interface Command permits CreateCommand, DeleteCommand, UpdateCommand {}

record CreateCommand(String name, Object data) implements Command {}
record DeleteCommand(String id) implements Command {}
record UpdateCommand(String id, Object data) implements Command {}

public class GuardTest {
    static String processCommand(Command cmd) {
        return switch (cmd) {
            case CreateCommand(var name, var data) 
                when name != null && !name.isEmpty() ->
                "Creating " + name;
            
            case CreateCommand(_, _) ->
                "Invalid create command";
            
            case DeleteCommand(var id) 
                when id.length() > 0 ->
                "Deleting " + id;
            
            case DeleteCommand(_) ->
                "Invalid delete command";
            
            case UpdateCommand(var id, var data) ->
                "Updating " + id;
        };
    }
}
```

### 실험 4: Sealed Hierarchy

```java
sealed interface Result<T> permits Outcome, Failure {
    <R> R fold(Function<T, R> success, Function<String, R> failure);
}

sealed interface Outcome<T> extends Result<T> permits Success, Empty {
    @Override
    default <R> R fold(Function<T, R> success, Function<String, R> failure) {
        return fold_success(success, failure);
    }
    
    <R> R fold_success(Function<T, R> success, Function<String, R> failure);
}

record Success<T>(T value) implements Outcome<T> {
    @Override
    public <R> R fold_success(Function<T, R> success, Function<String, R> failure) {
        return success.apply(value);
    }
}

record Empty<T>() implements Outcome<T> {
    @Override
    public <R> R fold_success(Function<T, R> success, Function<String, R> failure) {
        return failure.apply("No value");
    }
}

record Failure<T>(String error) implements Result<T> {
    @Override
    public <R> R fold(Function<T, R> success, Function<String, R> failure) {
        return failure.apply(error);
    }
}

public class HierarchyTest {
    public static void main(String[] args) {
        Result<String> result = new Success<>("Hello");
        
        String output = result.fold(
            value -> "Got: " + value,
            error -> "Error: " + error
        );
        
        System.out.println(output);  // Got: Hello
    }
}
```

---

## 📊 성능/비교

```
Sealed Interface의 성능 특성:

컴파일 시점:
  - Sealed attribute 검증: O(N) (N = 구현체 수)
  - Exhaustiveness check: O(1) (sealed이므로 모든 경우 미리 알려짐)
  - Pattern matching 최적화: 가능 (tableswitch/lookupswitch)

런타임 성능:

Switch 최적화 비교:

Non-sealed interface:
  switch (obj) {
      case Type1 t1 -> ...
      case Type2 t2 -> ...
      default -> ...  // 필수 (다른 구현체 가능)
  }
  
  바이트코드: lookupswitch (일반적 케이스)
  instanceof 체인: O(N)

Sealed interface:
  switch (obj) {
      case Type1 t1 -> ...
      case Type2 t2 -> ...
      // default 불필요
  }
  
  바이트코드: tableswitch 가능 (더 빠름)
  instanceof 체인: O(1) (컴파일러가 순서 최적화)

실제 성능:
  메서드 호출: ~3ns (JIT 인라인)
  instanceof: 비슷 (sealed 여부 무관)
  switch: sealed가 미세하게 빠를 수 있음 (tableswitch)

메모리:
  Sealed attribute: ~50 bytes (클래스 파일)
  런타임: 무시할 수 있음

일반적으로:
  성능 차이: 무시할 수 있음
  코드 안전성: 크게 향상 (exhaustiveness check)
  컴파일러 최적화: sealed가 유리
```

---

## ⚖️ 트레이드오프

```
Sealed Interface 도입의 트레이드오프:

장점:
  ✓ 계약 강제 (unexpected subtype 불가)
  ✓ Exhaustiveness check (컴파일 타임 검증)
  ✓ ADT 표현 가능 (함수형 프로그래밍)
  ✓ 컴파일러 최적화 (switch 최적화)
  ✓ 의도 명확 (sealed = "이 인터페이스는 폐쇄적")
  ✓ 버그 방지 (runtime classcast 예외 감소)

단점:
  ✗ Java 17+ 필수 (하위 호환성 부담)
  ✗ 학습 곡선 (sealed, non-sealed, 패턴 매칭)
  ✗ API 변경 시 허용 목록 수정 필요
  ✗ 라이브러리 버전 관리 복잡성
  ✗ 구현체 추가 시 인터페이스 수정 필요

비교: sealed vs non-sealed design

Non-sealed design (Open-Closed):
  interface Plugin { }
  // 누구든 구현체 추가 가능
  
  장점:
    + 확장성 (누구든 플러그인 작성 가능)
    + 느슨한 결합
  
  단점:
    - 모든 경우를 처리할 수 없음 (default 필수)
    - 버전 관리 복잡성
    - 보안 위험 (악의적 구현체)

Sealed design (Design by Contract):
  sealed interface Result permits Success, Failure { }
  
  장점:
    + 엄격한 계약
    + 컴파일 타임 검증 가능
    + 성능 최적화
  
  단점:
    - 확장성 제한
    - API 변경 비용

선택 기준:
  Plugin API (확장성 중요) → non-sealed
  Domain Model (안정성 중요) → sealed
  공개 라이브러리 → 신중하게 선택 (버전 호환성)
```

---

## 📌 핵심 정리

```
Sealed Interface의 역할:

1. 정의
   sealed interface Name permits Type1, Type2, ... { }
   - Type1, Type2만 구현 가능
   - 다른 구현체 불가 (컴파일 에러)

2. 수정자 조합
   sealed ... permits A, B
     ↓
   A: class (암시적 non-sealed)
      final class (확장 금지)
      sealed class (제한적 확장)
   
   B: non-sealed class (제한 해제)
      final class (확장 금지)

3. 컴파일러 검증
   - 모든 구현체가 permitted 목록에 있는가?
   - switch의 exhaustiveness check 가능
   - instanceof 체인 최적화

4. Pattern Matching
   switch (sealed_obj) {
       case Type1 t1 -> ...
       case Type2 t2 -> ...
       // default 불필요 (모든 경우 덮임)
   }

5. ADT 표현
   sealed interface Result<T> permits Success, Failure
   - 함수형 언어의 대수적 타입
   - 합 타입 (sum type) 표현
   - 패턴 매칭과 조합

6. 성능
   - 컴파일 타임 최적화 (tableswitch)
   - 런타임 instanceof 빠름
   - 메모리 오버헤드 무시할 수 있음

7. Java 버전
   Java 15-16: Preview (sealed) — 실험적, --enable-preview 플래그 필요
   Java 17+ : Final (sealed, pattern matching) — 표준 사용 가능, 권장 버전
   Java 19+ : 패턴 매칭 향상 (guard, type narrowing)
   Java 21+ : Record Pattern 결합으로 Exhaustive Matching 완성
```

---

## 🤔 생각해볼 문제

**Q1.** Sealed interface를 사용하면 새로운 구현체를 추가할 때마다 인터페이스 코드를 수정해야 하는데, 이것이 번거롭지 않은가?

<details>
<summary>해설 보기</summary>

좋은 질문이다. 이것은 **설계 철학의 차이**다:

**Open-Closed Principle (OCP) vs Design by Contract**

Sealed 철학:
```java
sealed interface Payment permits Card, Transfer {
    // 이 세 가지만 지원
}

// 6개월 후 PayPal 지원 필요
sealed interface Payment permits Card, Transfer, PayPal {
    // ← 인터페이스 수정 (breaking change)
}
```

이것이 "문제"인 것처럼 보이지만, 실제로는:

1. **명시적 의도**
   - 인터페이스 수정 = 새로운 경우 추가 = 의도적 결정
   - 번거로움 = 신중한 의사결정 강제

2. **컴파일 에러로 인한 안전성**
   ```java
   String handlePayment(Payment p) {
       return switch (p) {
           case Card c -> "Card";
           case Transfer t -> "Transfer";
           // PayPal 추가 시 컴파일 에러 → 반드시 처리
       };
   }
   ```

3. **Non-sealed로의 확장은 가능**
   ```java
   sealed interface Payment permits KnownPayment, UnknownPayment {}
   sealed interface KnownPayment extends Payment 
       permits Card, Transfer {}
   non-sealed class UnknownPayment implements Payment {}
   ```

**비교: 번거로움의 가치**

Non-sealed (쉬움):
```java
interface Payment { }  // 누구든 구현 가능
class PayPal implements Payment { }  // 추가 자유로움
```
문제: switch에서 PayPal을 처리 안 하면?
```java
String pay(Payment p) {
    return switch (p) {
        case Card c -> "Card";
        case Transfer t -> "Transfer";
        // PayPal 미처리 → 런타임 에러!
    };
}
```

Sealed (번거로움):
```java
sealed interface Payment permits Card, Transfer, PayPal {}
```
장점: 컴파일러가 모든 구현체를 알고 있음 → 강제 검증

**결론**: "번거로움"은 사실 "안전성 장치"다. 라이브러리 설계자가 의도적으로 API를 폐쇄하는 것.

</details>

---

**Q2.** Pattern matching에서 guard clause를 사용하면 exhaustiveness check가 어떻게 작동하는가?

<details>
<summary>해설 보기</summary>

**Guard는 exhaustiveness check에 영향을 줄 수 있다:**

```java
sealed interface Number permits Int, Double {}
record Int(int value) implements Number {}
record Double(double value) implements Number {}

// 모든 경우 처리됨 (guard 없음)
String describe1(Number n) {
    return switch (n) {
        case Int i -> "Integer";
        case Double d -> "Double";
    };
}
// ✓ 컴파일 성공

// Guard를 추가하면?
String describe2(Number n) {
    return switch (n) {
        case Int i when i.value > 0 -> "Positive Integer";
        case Int i when i.value == 0 -> "Zero";
        // Int의 음수는? (i.value < 0)
        case Double d -> "Double";
    };
}
// ✓ 컴파일 성공 (Int를 두 경우로 나눠도 exhaustive)
// 이유: Int의 모든 가능성이 처리됨

// 문제: guard가 모든 경우를 커버하지 않으면
String describe3(Number n) {
    return switch (n) {
        case Int i when i.value > 0 -> "Positive";  // 양수만
        case Double d -> "Double";
        // Int의 0, 음수는? ← 미처리
    };
}
// ✗ 컴파일 에러: "not exhaustive"

// 해결 1: guard 없는 fallback 추가
String describe4(Number n) {
    return switch (n) {
        case Int i when i.value > 0 -> "Positive";
        case Int i -> "Non-positive Integer";  // fallback
        case Double d -> "Double";
    };
}
// ✓ 컴파일 성공

// 해결 2: default 추가
String describe5(Number n) {
    return switch (n) {
        case Int i when i.value > 0 -> "Positive";
        case Double d -> "Double";
        default -> "Other";  // fallback
    };
}
// ✓ 컴파일 성공
```

**Rules:**

1. Guard가 없는 경우: pattern만으로 exhaustiveness 결정
2. Guard가 있는 경우: 모든 pattern의 가능성을 합쳐서 결정
3. guard가 불완전하면: 또 다른 case나 default 필요

**컴파일러의 로직:**

```
sealed interface X permits A, B, C

switch (x) {
    case A a when a.prop > 0 -> ...
    case A a -> ...  // A의 모든 경우 처리됨
    case B b -> ...
    case C c -> ...
}
// ✓ exhaustive: A (guard + fallback), B, C 모두 처리

switch (x) {
    case A a when a.prop > 0 -> ...
    case B b -> ...  // A의 guard=false인 경우 미처리
    case C c -> ...
}
// ✗ not exhaustive: A의 일부 미처리
```

</details>

---

**Q3.** Sealed interface와 Record를 조합하면 무엇이 좋은가?

<details>
<summary>해설 보기</summary>

**불변성 + 폐쇄성 = 완벽한 ADT:**

```java
// Java 17: record + sealed
sealed interface Expr permits ConstExpr, BinExpr, NegExpr {
    int eval();
}

record ConstExpr(int value) implements Expr {
    @Override public int eval() { return value; }
}

record BinExpr(Expr left, String op, Expr right) implements Expr {
    @Override public int eval() {
        // right-associative
        return switch (op) {
            case "+" -> left.eval() + right.eval();
            case "-" -> left.eval() - right.eval();
            case "*" -> left.eval() * right.eval();
            case "/" -> left.eval() / right.eval();
            default -> throw new IllegalArgumentException(op);
        };
    }
}

record NegExpr(Expr expr) implements Expr {
    @Override public int eval() { return -expr.eval(); }
}

// 사용
Expr expr = new BinExpr(
    new ConstExpr(3),
    "+",
    new NegExpr(new ConstExpr(2))
);

int result = expr.eval();  // 1
```

**이점:**

1. **불변성** (record)
   ```java
   ConstExpr c = new ConstExpr(5);
   c.value = 10;  // 컴파일 에러! (불변)
   ```

2. **폐쇄성** (sealed)
   ```java
   sealed interface Expr permits ConstExpr, BinExpr, NegExpr
   class MaliciousExpr implements Expr {}  // 컴파일 에러!
   ```

3. **패턴 매칭** (record + sealed)
   ```java
   int simplify(Expr e) {
       return switch (e) {
           case ConstExpr(int v) -> v;
           case BinExpr(ConstExpr(int a), "+", ConstExpr(int b)) 
               -> a + b;  // 상수 폴딩 (중첩 패턴)
           case BinExpr(var l, var op, var r) -> 
               new BinExpr(l, op, r).eval();
           case NegExpr(NegExpr(var inner)) -> inner;  // 이중 부정 제거
           case NegExpr(var e) -> -e.eval();
       };
   }
   ```

4. **타입 안전성**
   - 모든 필드가 final (thread-safe)
   - 직렬화 자동 지원
   - equals/hashCode/toString 자동 생성

**함수형 언어의 ADT 구현:**

Haskell:
```haskell
data Expr = Const Int
          | BinOp Expr String Expr
          | Neg Expr

eval :: Expr -> Int
eval (Const n) = n
eval (BinOp l op r) = ...
eval (Neg e) = -eval e
```

Java 17 (거의 동일):
```java
sealed interface Expr permits ConstExpr, BinExpr, NegExpr {}
record ConstExpr(int n) implements Expr {}
record BinExpr(Expr l, String op, Expr r) implements Expr {}
record NegExpr(Expr e) implements Expr {}

int eval(Expr expr) {
    return switch (expr) {
        case ConstExpr(int n) -> n;
        case BinExpr(var l, var op, var r) -> ...
        case NegExpr(var e) -> -eval(e);
    };
}
```

**왜 조합이 강력한가:**

- Record: 데이터 불변성
- Sealed: 케이스 완성성 (exhaustiveness)
- Pattern matching: 우아한 케이스 분석
- 컴파일러: 모든 것을 검증

이 조합이 없으면:
```java
// 이전 방식 (Java 8-16)
interface Expr {}
class ConstExpr implements Expr {
    private final int value;
    // getter, equals, hashCode, toString 수동 작성
}

int eval(Expr e) {
    if (e instanceof ConstExpr) {
        ConstExpr c = (ConstExpr) e;
        return c.getValue();
    } else if (e instanceof BinExpr) {
        // ...
    } else {
        throw new UnsupportedOperationException();
    }
}
// 더 길고, 보일러플레이트 많음
```

</details>

---

<div align="center">

**[⬅️ 이전: Static Interface Method](./04-static-interface-method.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: LocalDate · ZonedDateTime ➡️](../chapter07-datetime-api/01-localdate-zoneddatetime.md)**

</div>
