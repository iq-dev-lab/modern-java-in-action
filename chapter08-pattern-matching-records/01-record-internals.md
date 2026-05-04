# Record 구조 (Java 16) — 자동 생성 메서드와 바이트코드

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Record 컴파일러는 canonical constructor, 접근자 메서드, equals/hashCode/toString을 어떻게 자동 생성하는가?
- `java.lang.Record` 추상 클래스 상속이 의미하는 바는?
- `invokedynamic` + `ObjectMethods.bootstrap`으로 메서드를 합성하는 메커니즘은?
- Record와 일반 클래스의 바이트코드 차이는?
- Compact constructor와 canonical constructor의 차이는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Record는 Java 16부터 데이터 캐리어 역할을 하는 표준화된 방식이다. DTO, Value Object, 함수형 프로그래밍에서 불변 데이터 구조가 핵심인데, Record의 내부 동작을 이해하지 못하면 성능 이슈(바이트코드 생성 비용), 직렬화 문제, sealed class와의 조합 시 ADT 패턴 적용을 제대로 활용할 수 없다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Record가 모든 getter를 자동 생성한다고 가정
  record Point(int x, int y) { }
  
  // getX(), getY() 없음!
  point.getX();  // 컴파일 에러
  point.x();     // 정확함 (accessor)
  
  → 관례는 JavaBean의 get/set이 아니라 필드명 메서드

실수 2: Record에 커스텀 constructor를 추가하며 필드 초기화를 깜빡함
  record Person(String name, int age) {
      public Person(String name) {
          // this.name과 this.age를 초기화하지 않음 → 컴파일 에러
          System.out.println("생성됨");
      }
  }
  // 해결: compact constructor 사용
  record Person(String name, int age) {
      public Person(String name) {
          this("Unknown", 0);  // 다른 constructor 호출
      }
      public Person(String name, int age) {
          // compact constructor: 필드를 암묵적으로 초기화
          if (name == null) throw new IllegalArgumentException();
      }
  }

실수 3: Record가 가변 필드를 가질 수 있다고 착각
  record Container(ArrayList<String> items) { }  // 컴파일됨
  
  Container c = new Container(new ArrayList<>());
  c.items().add("X");  // items 리스트가 변경됨
  // Record의 "불변성"은 필드 참조만 불변, 내용은 가변 객체면 변경 가능
  
  → 신뢰성을 위해 defensive copy 사용
  record Container(List<String> items) {
      public Container(List<String> items) {
          this.items = List.copyOf(items);  // 불변 복사
      }
  }
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// Record 올바른 사용 패턴

// 패턴 1: 간단한 DTO (자동 생성만 활용)
record Point(int x, int y) { }
Point p = new Point(10, 20);
System.out.println(p.x() + ", " + p.y());  // 10, 20
System.out.println(p);  // Point[x=10, y=20]
System.out.println(p.equals(new Point(10, 20)));  // true

// 패턴 2: 불변성 보장 (defensive copy)
record Contact(String email, List<String> phones) {
    public Contact(String email, List<String> phones) {
        this.email = email;
        this.phones = List.copyOf(phones);  // 불변 복사
    }
}
var c = new Contact("a@b.com", new ArrayList<>(List.of("123", "456")));
// c.phones()는 불변 List → 외부 수정 불가

// 패턴 3: Compact constructor로 검증
record Temperature(double celsius) {
    public Temperature {  // compact constructor (매개변수 없음)
        if (celsius < -273.15) throw new IllegalArgumentException("절대영도 미만");
        // 자동으로 this.celsius = celsius 생성됨
    }
}
new Temperature(25.0);  // OK
// new Temperature(-300);  // 예외 발생

// 패턴 4: 커스텀 메서드 추가
record Circle(double radius) {
    public double area() { return Math.PI * radius * radius; }
    public double circumference() { return 2 * Math.PI * radius; }
}
var c = new Circle(5.0);
System.out.println(c.area());  // 78.53981...

// 패턴 5: 정적 팩토리 메서드
record Interval(int start, int end) {
    public static Interval from(int start, int length) {
        return new Interval(start, start + length);
    }
}
var iv = Interval.from(0, 10);  // [0, 10)
```

---

## 🔬 내부 동작 원리

### 1. Record 컴파일러의 자동 생성

```
Java 소스:
  record Point(int x, int y) { }

컴파일러가 생성하는 코드 (의사코드):

  public final class Point extends java.lang.Record {
      private final int x;
      private final int y;

      // 1. Canonical Constructor
      public Point(int x, int y) {
          this.x = x;
          this.y = y;
      }

      // 2. Accessor 메서드 (get*이 아님!)
      public int x() { return this.x; }
      public int y() { return this.y; }

      // 3. equals (ObjectMethods.bootstrap로 생성)
      @Override
      public boolean equals(Object o) {
          if (!(o instanceof Point)) return false;
          Point other = (Point) o;
          return this.x == other.x && this.y == other.y;
      }

      // 4. hashCode (ObjectMethods.bootstrap로 생성)
      @Override
      public int hashCode() {
          return Objects.hash(x, y);
      }

      // 5. toString (ObjectMethods.bootstrap로 생성)
      @Override
      public String toString() {
          return "Point[x=" + x + ", y=" + y + "]";
      }
  }

특성:
  - final 클래스 (상속 불가)
  - java.lang.Record 상속 (모든 Record의 기반)
  - 모든 필드는 private final (불변)
  - 생성자는 canonical만 자동 생성
  - accessor는 필드명과 동일한 이름
```

### 2. ObjectMethods.bootstrap과 invokedynamic

```
Java 16 이후, equals/hashCode/toString 생성이 invokedynamic으로 최적화됨:

컴파일된 바이트코드 (javap -c -v):

  public boolean equals(java.lang.Object);
    Code:
       0: aload_0
       1: aload_1
       2: invokedynamic #12, 0  // InvokeDynamic #0:equals:(LPoint;LObject;)Z
       7: ireturn
    BootstrapMethods:
       0: #22 REF_invokeStatic java/lang/runtime/ObjectMethods.bootstrap
         (Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;...
         Method arguments:
           #23 java/lang/invoke/TypeDescriptor (LPoint;)
           #24 java/lang/String "x;y"  // 필드 이름들

  public int hashCode();
    Code:
       0: aload_0
       1: invokedynamic #13, 0  // InvokeDynamic #1:hashCode:(LPoint;)I
       6: ireturn

  public java.lang.String toString();
    Code:
       0: aload_0
       1: invokedynamic #14, 0  // InvokeDynamic #2:toString:(LPoint;)Ljava/lang/String;
       6: areturn

이점:
  - invokedynamic은 런타임에 메서드 핸들을 캐시
  - ObjectMethods.bootstrap이 리플렉션으로 필드를 읽고 최적화된 방법 조회
  - 첫 호출: 부트스트랩 오버헤드, 이후: 캐시된 메서드 핸들 사용
  - JVM이 JIT 컴파일할 때 자동 인라인 가능
```

### 3. Compact Constructor vs Canonical Constructor

```
record Temperature(double celsius) {
    // Canonical Constructor (명시적 형식)
    public Temperature(double celsius) {
        if (celsius < -273.15) throw new IllegalArgumentException();
        this.celsius = celsius;  // 명시적 초기화
    }
}

record Temperature(double celsius) {
    // Compact Constructor (암묵적 형식) ← 권장
    public Temperature {  // 매개변수 목록 없음!
        if (celsius < -273.15) throw new IllegalArgumentException();
        // this.celsius = celsius는 컴파일러가 자동 추가
    }
}

Compact constructor 장점:
  - 필드 초기화를 자동으로 수행 (실수 방지)
  - 검증 로직만 작성 → 가독성 향상
  - 매개변수와 필드 이름이 동일하므로 명확함
```

### 4. Java.lang.Record 추상 클래스

```
public abstract class Record {
    protected Record() { }  // 모든 Record의 기반

    public abstract boolean equals(Object obj);
    public abstract int hashCode();
    public abstract String toString();

    // 고급: 구성요소 검색 (Java 17+)
    protected final Object[] getExternalizableFields() { ... }
}

모든 Record는:
  - Record를 직접 상속 (명시하지 않아도)
  - equals/hashCode/toString을 구현해야 함 (추상 메서드)
  - 추가 상속 불가 (Record를 확장할 수 없음)
  → Record는 최종 클래스들의 기반 제공
```

### 5. java.lang.reflect.RecordComponent

```
java.lang.reflect.RecordComponent:
  Record의 필드 정보를 리플렉션으로 조회

public class RecordComponent {
    public Class<?> getType();                  // 필드 타입
    public String getName();                    // 필드 이름
    public Method getAccessor();                // accessor 메서드
    public AnnotatedType getAnnotatedType();    // 어노테이션 정보
    public <T extends Annotation> T getAnnotation(Class<T> annotationClass);
}

사용 예:
  record Point(int x, int y) { }
  
  RecordComponent[] components = Point.class.getRecordComponents();
  for (RecordComponent rc : components) {
      System.out.println(rc.getName());      // x, y
      System.out.println(rc.getType());      // int, int
      System.out.println(rc.getAccessor());  // x(), y()
  }

이를 활용한 동적 직렬화, 복사, DTO 변환 구현 가능
```

---

## 💻 실전 실험

### 실험 1: 바이트코드 분석

```bash
# 컴파일
javac Point.java

# Record 바이트코드 자세히 분석
javap -c -v -private Point.class

# 출력 예시:
public final class Point extends java.lang.Record
  // equals 메서드 보기
  public boolean equals(java.lang.Object);
    descriptor: (Ljava/lang/Object;)Z
    flags: PUBLIC
    Code:
      stack=3, locals=2, args_size=2
         0: aload_0
         1: aload_1
         2: if_acmpeq     19
         5: aload_1
         6: instanceof    #2  // class java/lang/Object
         9: ifeq          27
        12: aload_0
        13: aload_1
        14: checkcast     #2
        15: astore_1
        16: goto          19
        19: ireturn
```

### 실험 2: equals/hashCode/toString 성능 비교

```java
import java.lang.reflect.RecordComponent;

public class RecordBytecodeTest {
    record Point(int x, int y) { }

    public static void main(String[] args) throws Exception {
        Point p1 = new Point(10, 20);
        Point p2 = new Point(10, 20);

        // 1. Accessor 메서드 확인
        System.out.println("Accessor: " + p1.x() + ", " + p1.y());

        // 2. equals 테스트
        System.out.println("p1.equals(p2): " + p1.equals(p2));  // true
        System.out.println("p1 == p2: " + (p1 == p2));          // false

        // 3. hashCode 테스트
        System.out.println("p1.hashCode(): " + p1.hashCode());
        System.out.println("p2.hashCode(): " + p2.hashCode());  // 동일

        // 4. toString 테스트
        System.out.println("toString: " + p1);  // Point[x=10, y=20]

        // 5. RecordComponent 리플렉션
        RecordComponent[] components = Point.class.getRecordComponents();
        System.out.println("Record Components:");
        for (RecordComponent rc : components) {
            System.out.printf("  %s (%s) -> %s()%n",
                rc.getName(), rc.getType().getSimpleName(),
                rc.getAccessor().getName());
        }
    }
}

// 출력:
// Accessor: 10, 20
// p1.equals(p2): true
// p1 == p2: false
// p1.hashCode(): 993  (Objects.hash(10, 20))
// p2.hashCode(): 993
// toString: Point[x=10, y=20]
// Record Components:
//   x (int) -> x()
//   y (int) -> y()
```

### 실험 3: Compact Constructor 검증

```java
record Temperature(double celsius) {
    public Temperature {
        if (celsius < -273.15) {
            throw new IllegalArgumentException(
                "절대영도 미만: " + celsius
            );
        }
    }
}

public class CompactConstructorTest {
    public static void main(String[] args) {
        // 유효한 온도
        Temperature t1 = new Temperature(25.0);
        System.out.println(t1);  // Temperature[celsius=25.0]

        // 무효한 온도
        try {
            Temperature invalid = new Temperature(-300.0);
        } catch (IllegalArgumentException e) {
            System.out.println("예외: " + e.getMessage());
        }
    }
}
```

### 실험 4: 불변성 확인 (참조 vs 내용)

```java
import java.util.ArrayList;
import java.util.List;

record Container(ArrayList<String> items) { }

public class RecordImmutabilityTest {
    public static void main(String[] args) {
        ArrayList<String> list = new ArrayList<>(List.of("A", "B"));
        Container c = new Container(list);

        System.out.println("생성 후: " + c.items());  // [A, B]

        // 외부 리스트 수정
        list.add("C");
        System.out.println("외부 수정 후: " + c.items());  // [A, B, C]!

        // Record는 참조만 불변, 내용은 변경 가능
        // → 불변성 보장을 위해 defensive copy 필요

        // 올바른 방식
        record ImmutableContainer(List<String> items) {
            public ImmutableContainer(List<String> items) {
                this.items = List.copyOf(items);
            }
        }
        
        var list2 = new ArrayList<>(List.of("X", "Y"));
        var ic = new ImmutableContainer(list2);
        list2.add("Z");
        System.out.println("불변 컨테이너: " + ic.items());  // [X, Y]
    }
}
```

---

## 📊 성능/비교

```
Record vs 일반 클래스 (equals/hashCode/toString 생성) 비교:

특성                   | Record (Java 16+)    | 일반 클래스 + Lombok
─────────────────────┼──────────────────────┼──────────────────────────
컴파일 시간            | 빠름 (컴파일러 내장)   | 느림 (어노테이션 프로세서)
런타임 성능            | 동일 (invokedynamic)  | 동일 (생성된 메서드)
바이트코드 크기        | 더 작음               | Lombok이 더 큼
메모리 오버헤드        | 최소 (final 클래스)   | Lombok 런타임 가능
접근자 이름           | 필드명 (get* 아님)     | get* (JavaBean 관례)
불변성                | 강제됨 (final)        | 선택적 (@Value)
상속                  | 불가 (final)         | 가능
null 안전성           | 자동 (생성된 equals)   | 옵션 (@NonNull)

equals/hashCode 생성 속도 (첫 호출):
  Record invokedynamic bootstrap: ~2μs (캐시 후)
  Lombok 일반 메서드: ~1μs (이미 컴파일됨)
  
  → 첫 호출 후 성능 차이 무시할 수준
  → Record가 더 안전하고 표준화됨
```

---

## ⚖️ 트레이드오프

```
Record 설계 트레이드오프:

final 클래스만 가능:
  장점: 불변성 강제, 상속 관련 버그 방지
  단점: 다형성 활용 불가 (sealed class와 함께 사용)

필드는 public final:
  장점: 불변성 보장, 직렬화 용이
  단점: encapsulation 원칙 위반 (하지만 불변이므로 안전)

자동 생성 메서드만 지원:
  장점: 표준화, 실수 방지
  단점: 복잡한 equals/hashCode 로직 필요 시 직접 구현

invokedynamic 기반:
  장점: JVM이 최적화 가능, 유연함
  단점: 첫 호출 시 bootstrap 오버헤드 (무시할 수준)

Accessor 메서드 이름 (get* 아님):
  장점: 간결, 함수형 스타일
  단점: JavaBean 도구와 호환성 문제 (리플렉션으로 workaround)
```

---

## 📌 핵심 정리

```
Record 핵심 개념:

1. 자동 생성 코드:
   - Canonical constructor
   - Accessor 메서드 (필드명 메서드)
   - equals/hashCode/toString

2. 바이트코드 레벨:
   - invokedynamic + ObjectMethods.bootstrap
   - 첫 호출에 bootstrap, 이후 메서드 핸들 캐시

3. 불변성:
   - 모든 필드는 private final
   - 참조 불변, 내용은 가변 객체면 변경 가능
   - defensive copy로 완전 불변성 보장

4. Compact Constructor:
   - 검증/초기화 로직만 작성
   - this.field = field는 자동 생성

5. RecordComponent 리플렉션:
   - 필드 이름, 타입, accessor 메서드 동적 조회
   - 직렬화, 복사, DTO 변환 자동화 가능

6. java.lang.Record 상속:
   - 모든 Record의 기반
   - equals/hashCode/toString은 추상 메서드
   - 추가 상속 불가 (최종 클래스)
```

---

## 🤔 생각해볼 문제

**Q1.** Record의 accessor 메서드가 `getX()`가 아니라 `x()`인 이유는?

<details>
<summary>해설 보기</summary>

JavaBean 관례(get* prefix)는 GUI 프레임워크가 리플렉션으로 property를 찾기 위해 만들어졌다. Record는 Java 16(2021년)에 도입되었으며, 함수형 프로그래밍과 데이터-지향 설계를 고려했다.

`x()` 형태는:
- Kotlin, Scala, Rust 등 현대 언어의 accessor 관례와 일치
- 필드명과 메서드명이 동일하므로 코드 읽기 간결
- Optional, Stream 등 Java 함수형 API와 일관성

JavaBean과의 호환성 필요 시, 리플렉션 기반 BeanInfo나 MethodHandle로 workaround 가능.

</details>

---

**Q2.** `invokedynamic + ObjectMethods.bootstrap` 방식이 일반 메서드를 컴파일러가 생성하는 것보다 나은 점은?

<details>
<summary>해설 보기</summary>

일반 메서드 생성 (Lombok 방식):
  - 컴파일 시 전체 메서드 바이트코드 생성
  - 바이트코드 크기 증가
  - 필드 추가 시 메서드 재생성 필요

invokedynamic 방식 (Record):
  - 컴파일 시 부트스트랩 메타데이터만 기록 (작음)
  - 런타임에 ObjectMethods.bootstrap이 필드 리플렉션으로 최적화된 메서드 생성
  - 첫 호출: bootstrap 오버헤드 (~2μs)
  - 이후: 캐시된 MethodHandle 사용 (일반 메서드와 동일)
  - JVM이 JIT 컴파일할 때 메서드 핸들을 인라인 가능

결과: 바이트코드 크기는 작고, 런타임 성능은 동일 이상, JVM 최적화 여지 큼.

</details>

---

**Q3.** Record가 모든 필드를 `private final`로 강제하는데, 이것이 캡슐화(encapsulation) 원칙을 위반하지 않는 이유는?

<details>
<summary>해설 보기</summary>

캡슐화의 목표는 "내부 구현 변경으로부터 클라이언트 보호"다. 두 가지 메커니즘이 있다:

1. **접근 제어** (Access Control): private로 필드 노출 방지
2. **불변성** (Immutability): 노출된 객체가 변경 불가능

Record는 방법 2를 택한다:
  - 필드가 public이면 직접 접근 가능 (캡슐화 제거처럼 보임)
  - 하지만 final이므로 값 변경 불가능 → 내부 변경과 무관
  - 클라이언트는 필드값에 의존할 수 있음 (변경 불가능이므로)

예: record Point(int x, int y)
  - 내부적으로 x, y를 long으로 바꾸어도 accessor는 int로 변환 반환 가능 (논리적으로)
  - 하지만 public int x는 바꿀 수 없음 → 계약 위반

따라서 Record의 public final은 불변성 기반 캡슐화라고 볼 수 있다. 값 객체(Value Object)의 특성상 불변성이 캡슐화보다 중요하다.

</details>

---

<div align="center">

**[⬅️ 이전 챕터: Date/Calendar 마이그레이션](../chapter07-datetime-api/04-date-calendar-migration.md)** | **[홈으로 🏠](../README.md)** | **[다음: Record vs Lombok @Data ➡️](./02-record-vs-lombok.md)**

</div>
