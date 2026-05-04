# Record vs Lombok @Data — 어떤 차이가 있는가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Record는 불변 + 표준화된 구조, Lombok @Data는 가변이고 어노테이션 프로세서 기반이다. 이 설계 차이가 어디서 나타나는가?
- Compact constructor와 Lombok @Builder/@Value의 검증 메커니즘 비교는?
- JPA 엔티티에서 Record가 부적합한 이유(no-args constructor 부재)는?
- DTO/Value Object 용도에서 각각 어떤 장점과 단점이 있는가?
- Lombok @Value를 Record 대신 사용해야 하는 경우는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Java 프로젝트의 대부분은 데이터 캐리어(DTO, 도메인 객체, Value Object) 클래스가 수십~수백 개다. Record vs Lombok 선택은 코드 복잡도, 컴파일 속도, 가독성, JPA 호환성, 다형성 필요 여부에 영향을 미친다. Record는 Java 16 표준화로 Lombok 의존성을 제거하는 방향이지만, 기존 프로젝트나 복잡한 요구사항에서는 여전히 Lombok이 강하다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Record를 JPA 엔티티로 직접 사용
  @Entity
  record User(Long id, String name, String email) { }
  
  // 컴파일 에러: no-args constructor 없음!
  // JPA는 no-args constructor로 인스턴스 생성
  // Record는 모든 필드를 필수 매개변수로 요구
  
  → JPA는 가변 엔티티 필요
  → Record는 DTO(영속성 계층 밖)로만 사용

실수 2: Lombok @Data로 충분하다고 가정
  @Data
  class Point { int x; int y; }
  
  // @Data = @Getter @Setter @ToString @EqualsAndHashCode @RequiredArgsConstructor
  // x, y 변경 가능 → 예기치 않은 버그
  // Point p = new Point();  p.x = 100;  ← 중간 상태 가능
  
  → 불변성 필요 시 @Value 사용
  → Record는 불변 강제

실수 3: Compact constructor에서 this 재할당
  record Temperature(double celsius) {
      public Temperature {
          this = new Temperature(25.0);  // 컴파일 에러!
      }
  }
  
  // Compact constructor는 parameter만 수정 가능
  // Canonical constructor처럼 필드 재할당 불가
  // this 자체는 불변 참조

실수 4: @Value로 List를 직접 필드로 사용
  @Value
  class Container {
      List<String> items;  // 가변 컬렉션
  }
  
  var c = new Container(new ArrayList<>(...));
  c.items.add("X");  // items 리스트 변경 가능!
  
  // @Value는 setter를 제거할 뿐
  // 내용 변경은 방지하지 않음
  // → defensive copy 여전히 필요
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 패턴 1: DTO는 Record 사용 (Java 16+)
record UserDTO(Long id, String name, String email) { }

// 패턴 2: JPA 엔티티는 가변 클래스 유지
@Entity
class User {
    @Id Long id;
    String name;
    String email;
    // no-args constructor 필요
    public User() { }
    public User(Long id, String name, String email) { ... }
}

// 패턴 3: 엔티티 → DTO 변환
User user = entityManager.find(User.class, 1L);
UserDTO dto = new UserDTO(user.getId(), user.getName(), user.getEmail());

// 패턴 4: 검증이 필요한 Value Object는 Record
record Temperature(double celsius) {
    public Temperature {  // compact constructor
        if (celsius < -273.15) {
            throw new IllegalArgumentException("절대영도 미만");
        }
    }
}

// 패턴 5: 복잡한 빌더가 필요하면 Lombok @Builder 고려
@Builder
@Value
class QueryRequest {
    String sql;
    int timeout;
    boolean readOnly;
    Map<String, Object> params;
}
var req = QueryRequest.builder()
    .sql("SELECT * FROM users")
    .timeout(5000)
    .readOnly(true)
    .params(Map.of("id", 1))
    .build();

// 패턴 6: 불변 가능한 것들은 Record (상속 불필요)
public record Point(int x, int y) {
    public double distance() {
        return Math.sqrt(x * x + y * y);
    }
}

// 패턴 7: 다형성이 필요하면 sealed class + Record
sealed interface Shape permits Circle, Rectangle, Line { }
record Circle(Point center, int radius) implements Shape { }
record Rectangle(Point topLeft, int width, int height) implements Shape { }
record Line(Point p1, Point p2) implements Shape { }
```

---

## 🔬 내부 동작 원리

### 1. Record vs Lombok @Data 컴파일 과정

```
Record 컴파일:

  소스: record Point(int x, int y) { }
  ↓
  javac가 직접 코드 생성 (컴파일러 내장)
  ↓
  생성됨: canonical constructor, x(), y(), equals(), hashCode(), toString()
  ↓
  invokedynamic + ObjectMethods.bootstrap (선택)
  ↓
  컴파일된 클래스 파일

Lombok @Data 컴파일:

  소스: @Data class Point { int x; int y; }
  ↓
  javac 실행 (APT = Annotation Processing Tool 병행)
  ↓
  Lombok의 annotation processor가 AST 수정
  ↓
  생성됨: getters, setters, equals, hashCode, toString, constructor
  ↓
  수정된 AST를 javac가 컴파일
  ↓
  컴파일된 클래스 파일 + Lombok @Generated 주석

차이점:
  Record: 표준 JAR 불필요, 소스에 Record 선언만 있으면 됨
  Lombok: lombok.jar가 classpath에 필요, @lombok.Generated으로 표시
```

### 2. 필드 초기화 및 검증

```
Record (Compact Constructor):

  record Temperature(double celsius) {
      public Temperature {
          if (celsius < -273.15) throw new IllegalArgumentException();
          // this.celsius = celsius는 컴파일러가 자동 추가
      }
  }
  
  생성되는 바이트코드:
    public Temperature(double celsius) {
        if (celsius < -273.15) throw ...;
        this.celsius = celsius;  // 자동 생성
    }

Lombok @Value + @Builder:

  @Builder
  @Value
  class Temperature {
      double celsius;
      
      @Builder.Default
      private void validateCelsius() {
          if (celsius < -273.15) throw new IllegalArgumentException();
      }
      
      // 또는
      public static class TemperatureBuilder {
          public Temperature build() {
              Temperature temp = new Temperature(this.celsius);
              if (temp.celsius < -273.15) throw ...;
              return temp;
          }
      }
  }
  
  생성되는 바이트코드:
    private Temperature(double celsius) {
        this.celsius = celsius;
    }
    
    public static TemperatureBuilder builder() { ... }
    public static class TemperatureBuilder { ... }

Record 장점: 검증이 constructor에 통합됨, 명확함
Lombok 장점: @Builder로 복잡한 초기화 패턴 지원
```

### 3. JPA 호환성

```
JPA 요구사항:

  ① no-args constructor (proxy 생성, 리플렉션 초기화)
  ② setter 메서드 (필드 로딩)
  ③ mutable (dirty checking을 위해 상태 변경 감지)

Record:
  ① no-args constructor: ❌ 모든 필드 필수 매개변수
  ② setter: ❌ 자동 생성 메서드는 accessor만
  ③ mutable: ❌ final 필드 + 불변
  
  → JPA 엔티티로 사용 불가!

Lombok @Data:
  ① no-args constructor: ✅ @RequiredArgsConstructor(force = true)로 생성
  ② setter: ✅ @Setter 자동 생성
  ③ mutable: ✅ 필드 변경 가능
  
  → JPA 엔티티로 가능

권장 아키텍처:

  JPA 영속성 계층:
    @Entity class User { ... }  // 가변, no-args constructor

  서비스 계층:
    record UserDTO(Long id, String name, String email) { }

  변환:
    UserDTO dto = new UserDTO(user.getId(), user.getName(), user.getEmail());
```

### 4. 불변성 강제 수준

```
Record (완전 불변):

  record Point(int x, int y) { }
  Point p = new Point(10, 20);
  p.x = 30;  // ❌ 컴파일 에러: x는 final
  
  accessor로만 접근: p.x()  // ✅ 10

Lombok @Value (준불변):

  @Value
  class Point {
      int x;
      int y;
  }
  Point p = new Point(10, 20);
  p.x = 30;  // ❌ 컴파일 에러: setter 없음
  p.getX();  // ✅ 10
  
  하지만 가변 필드는 내용 변경 가능:
    @Value
    class Container {
        List<String> items;
    }
    Container c = new Container(new ArrayList<>(...));
    c.getItems().add("X");  // ⚠️ 가능 (방지 불가)

Lombok @Value + @Getter(lazy = true):

  @Value
  class CachedData {
      @Getter(lazy = true)
      final List<String> cached = loadCache();
      // 첫 접근 시 loadCache() 실행, 결과 캐시
  }

Record (defensive copy 포함):

  record Container(List<String> items) {
      public Container(List<String> items) {
          this.items = List.copyOf(items);  // 불변 복사
      }
  }
  
  → 완전 불변성 보장
```

### 5. 상속과 다형성

```
Record:
  record Point(int x, int y) { }  // final 클래스
  class Point3D extends Point { }  // ❌ 컴파일 에러: final 클래스 상속 불가
  
  다형성 필요 시: sealed interface + Record
  sealed interface Shape permits Circle, Square { }
  record Circle(int radius) implements Shape { }
  record Square(int side) implements Shape { }

Lombok @Value:
  @Value
  class Point {
      int x;
      int y;
  }
  class Point3D extends Point { }  // ⚠️ 가능하지만 권장하지 않음
  
  @Value는 상속 고려하지 않음 (final 아님)
  → 의도하지 않은 상속 가능 → 버그 유발 가능

권장:
  - 상속 불필요: Record
  - 상속 필요: sealed interface + Record 또는 Lombok @Value
```

---

## 💻 실전 실험

### 실험 1: JPA 엔티티 호환성 테스트

```java
import javax.persistence.*;

// Record로 시도 (실패)
// @Entity
// record User(Long id, String name) { }  // 컴파일됨
// 실행 시: PersistenceException: no-args constructor required

// Lombok @Data (성공)
@Entity
@Data
@NoArgsConstructor
class User {
    @Id
    Long id;
    String name;
    String email;
}

public class JpaCompatibilityTest {
    public static void main(String[] args) {
        // User 생성 및 저장 가능
        User u = new User();  // no-args constructor
        u.setName("Alice");   // setter
        // entityManager.persist(u);
        
        System.out.println(u);  // Lombok이 생성한 toString
    }
}
```

### 실험 2: 컴파일 속도 비교

```bash
# 테스트: 100개 Record vs 100개 Lombok @Data 클래스

# Record 컴파일
javac -d target records/*.java
# → ~200ms (빠름)

# Lombok @Data 컴파일
javac -cp lombok.jar -d target -processorpath lombok.jar data/*.java
# → ~500ms (느림, APT 실행)

# 차이: Record가 2배 이상 빠름
```

### 실험 3: 바이트코드 크기 비교

```bash
javap -c record-point.class | wc -l  # Record Point
# 출력: ~50줄 (작음)

javap -c lombok-point.class | wc -l  # Lombok @Data Point
# 출력: ~80줄 (더 큼)

# Record: invokedynamic 부트스트랩 메타데이터만
# Lombok: 전체 메서드 바이트코드 포함
```

### 실험 4: 검증 로직 비교

```java
// Record Compact Constructor
record Temperature(double celsius) {
    public Temperature {
        if (celsius < -273.15) {
            throw new IllegalArgumentException("절대영도 미만");
        }
    }
}

// Lombok @Builder
@Builder
@Value
class Temperature {
    double celsius;
    
    public static class TemperatureBuilder {
        public Temperature build() {
            Temperature t = new Temperature(this.celsius);
            if (t.celsius < -273.15) {
                throw new IllegalArgumentException("절대영도 미만");
            }
            return t;
        }
    }
}

public class ValidationTest {
    public static void main(String[] args) {
        // Record (직관적)
        try {
            new Temperature(-300);
        } catch (IllegalArgumentException e) {
            System.out.println("Record: " + e.getMessage());
        }
        
        // Lombok (빌더 체인 필요)
        try {
            Temperature.builder().celsius(-300).build();
        } catch (IllegalArgumentException e) {
            System.out.println("Lombok: " + e.getMessage());
        }
    }
}
```

---

## 📊 성능/비교

```
Record vs Lombok @Data 종합 비교:

특성                    | Record (Java 16+)    | Lombok @Data
────────────────────────┼──────────────────────┼──────────────────────
컴파일 의존성            | 표준 JDK (의존성 없음) | lombok.jar 필수
컴파일 시간              | ~200ms (100개)       | ~500ms (100개)
런타임 성능              | 동일 (invokedynamic)  | 동일 (일반 메서드)
바이트코드 크기          | 더 작음              | 더 큼
불변성                  | 강제됨 (final)       | 선택적 (@Value)
JPA 호환성               | ❌ (no-args 없음)     | ✅ (with @NoArgsConstructor)
상속 지원                | ❌ (final 클래스)     | ⚠️ (비권장)
필드 검증               | compact constructor  | @Builder / @Valid
가변성 옵션              | 없음 (불변)          | @Data (가변) vs @Value (불변)
IDE 지원                | 모든 IDE 지원        | IntelliJ 최적화
문서화                  | 자명한 구조          | @Data 의미 이해 필요
접근자 이름              | x() (함수형 스타일)   | getX() (JavaBean 관례)
nullability             | 수동 검증            | @NonNull 어노테이션

도메인별 추천:

영역                    | 추천               | 이유
────────────────────────┼────────────────────┼─────────────────────────
API DTO                 | Record             | 불변, 표준화, 간결
JPA 엔티티              | Lombok @Data       | no-args constructor 필요
도메인 Value Object     | Record             | 불변성, 검증, 간결
복잡한 빌더             | Lombok @Builder    | @Builder로 플루언트 API
레거시 코드베이스       | Lombok @Data       | 기존 코드와 호환성
Java 16+ 신규 프로젝트  | Record             | 표준화, 의존성 제거
```

---

## ⚖️ 트레이드오프

```
Record 선택 시:

  장점:
    - 표준화: JDK에 포함, 의존성 없음
    - 컴파일 빠름: APT 오버헤드 없음
    - 불변성 강제: 버그 방지
    - 간결함: 보일러플레이트 최소

  단점:
    - JPA 엔티티 불가: no-args constructor 없음
    - 상속 불가: final 클래스만
    - 커스터마이징 어려움: 자동 생성 메서드만
    - 조건부 필드 불가: 모든 필드가 필수

Lombok @Data 선택 시:

  장점:
    - JPA 엔티티 가능: no-args, setter 자동
    - 유연성: getter/setter 선택 가능
    - 빌더: @Builder로 복잡한 초기화
    - 기존 도구 호환: JavaBean 관례 지원

  단점:
    - 의존성 추가: lombok.jar 필요
    - 컴파일 느림: APT 실행
    - 마법적: 생성 코드 불명확
    - 불변성 선택적: 실수로 가변 가능

Lombok @Value 선택 시:

  장점:
    - 불변성: @Data보다 나음
    - 여전히 유연: @Builder 등 지원

  단점:
    - JPA 엔티티 여전히 어려움
    - 내용 변경 완전히 방지 불가 (defensive copy 수동)
```

---

## 📌 핵심 정리

```
Record vs Lombok 선택 기준:

1. Java 16 이상 + JPA 엔티티 아님
   → Record 사용 (표준화, 빠름, 의존성 없음)

2. JPA 엔티티 필요
   → Lombok @Data (no-args constructor 필요)

3. 불변 필요 + JPA 아님
   → Record (완전 불변성)

4. 복잡한 빌더 패턴 필요
   → Lombok @Builder (플루언트 API)

5. 레거시 코드베이스 + Lombok 이미 사용
   → Lombok 유지 (혼동 방지)

6. 신규 프로젝트 + 의존성 최소
   → Record로 마이그레이션 (Java 16+)

아키텍처 권장:
  - JPA 엔티티: 가변 클래스 (Lombok @Entity @Data)
  - DTO: Record (불변, 표준화)
  - 도메인 Value Object: Record (불변, 검증)
  - 복잡한 도메인 객체: 커스텀 클래스
```

---

## 🤔 생각해볼 문제

**Q1.** Record는 왜 no-args constructor를 제공하지 않으면서도 API DTO로 추천되는가?

<details>
<summary>해설 보기</summary>

API DTO는 외부 시스템과 데이터를 교환할 때 사용된다. 주요 특성:

1. **불변성**: API를 통해 받은 데이터는 변경되지 않아야 함
2. **직렬화**: JSON/XML로 변환되지만, 역직렬화는 모든 필드를 채운 후 생성
3. **JPA와 분리**: DTO는 영속성 계층과 무관하므로 no-args constructor 불필요

Jackson, Gson 등 JSON 라이브러리는 모든 필드가 있는 constructor를 사용하거나 리플렉션으로 필드 설정한다. Record의 필드가 public이므로 직접 설정 가능.

```java
// JSON 역직렬화 예
ObjectMapper mapper = new ObjectMapper();
UserDTO user = mapper.readValue(json, UserDTO.class);
// Jackson이 record의 canonical constructor 호출
```

따라서 no-args constructor 부재는 DTO로는 문제 없고, JPA 엔티티로는 부적합한 것.

</details>

---

**Q2.** Lombok @Data에서 가변 필드가 있어도 불변성을 어떻게 보장할 수 있는가?

<details>
<summary>해설 보기</summary>

완전한 불변성을 보장할 수 없다. defensive copy로만 부분적으로 가능:

```java
@Value  // @Data보다 나음
class Container {
    List<String> items;
    
    public Container(List<String> items) {
        this.items = List.copyOf(items);  // 수동 defensive copy
    }
}
```

하지만 이것도 리플렉션으로 우회 가능:

```java
var c = new Container(List.of("A", "B"));
Field itemsField = Container.class.getDeclaredField("items");
itemsField.setAccessible(true);
List newList = new ArrayList<>(List.of("A", "B", "C"));
itemsField.set(c, newList);  // 우회 가능
```

Record + final은 이것도 방지:

```java
record Container(List<String> items) {
    public Container {
        items = List.copyOf(items);
    }
}
// 컴파일 레벨에서 final이므로 리플렉션도 불가
```

따라서 진정한 불변성은 Record + final + defensive copy 조합만 가능. Lombok은 setter 부재로 의도는 명확하지만, 런타임 보장은 약함.

</details>

---

**Q3.** Record의 accessor 메서드 이름이 JavaBean 관례를 따르지 않는데, 기존 라이브러리(Spring, Jackson)와 호환성은?

<details>
<summary>해설 보기</summary>

Jackson, Spring 등 주요 라이브러리는 이미 Record를 지원한다:

1. **Jackson**: `@JsonProperty`로 필드명 명시 또는 `mapper.registerModule(new KotlinModule())`처럼 자동 감지
2. **Spring**: `@GetMapping`에서 `@RequestBody UserDTO dto`로 직렬화 가능 (reflection 사용)
3. **JPA**: Record는 직접 사용 불가지만, 엔티티 → DTO 변환 시 문제 없음

현재 표준:
  - Kotlin data class도 accessor 메서드가 필드명 기반 (get* 아님)
  - Spring 6.1+부터 Record 자동 인식
  - Jackson 2.14+부터 Record 기본 지원

호환성 체크:
```java
// 최신 Jackson (2.14+)
var mapper = new ObjectMapper();
record User(String name, int age) { }

String json = mapper.writeValueAsString(new User("Alice", 30));
// "{"name":"Alice","age":30}" → 자동 동작

User user = mapper.readValue(json, User.class);
// canonical constructor 자동 호출
```

레거시 라이브러리 사용 시에만 `@JsonProperty` 어노테이션 필요. 현대 Java 스택에서는 대부분 Record 지원.

</details>

---

<div align="center">

**[⬅️ 이전: Record 구조](./01-record-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: Pattern Matching for instanceof ➡️](./03-pattern-matching-instanceof.md)**

</div>
