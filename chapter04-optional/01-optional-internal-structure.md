# Optional 내부 구조 — 왜 final class인가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Optional이 단 하나의 필드 `private final T value`만 갖는 이유는?
- `EMPTY` 상수가 싱글톤으로 재사용되는 설계 의도는?
- `final class`로 상속을 금지한 근본적 이유는?
- `@ValueBased` 어노테이션과 `==` 비교 금지의 관계는?
- Optional이 `Serializable`을 구현하지 않은 이유와 영향은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Optional은 Java 8의 널 처리 패러다임을 바꿨으나, 그 설계는 매우 신중하고 제한적이다. Optional의 내부 구조를 이해하면, 왜 필드나 컬렉션 매개변수에 쓸 수 없는지, 왜 직렬화가 불가능한지를 깨닫게 되고, 따라서 메서드 반환 타입으로만 사용해야 한다는 설계 철학을 체득할 수 있다. 이는 API 설계 원칙의 핵심이기도 하다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Optional을 클래스 필드로 사용
  class User {
      Optional<String> email;  // ❌ 위험!
      Optional<Integer> age;
  }
  → 메모리 오버헤드: User 객체 당 Optional 래퍼 추가
  → 직렬화 불가: ORM/JSON 직렬화 시 예외 또는 null 처리 불명확
  → 해결: private String email; + public Optional<String> getEmail()

실수 2: Optional == 비교 사용
  Optional<String> opt1 = Optional.of("value");
  Optional<String> opt2 = Optional.of("value");
  if (opt1 == opt2) { ... }  // ❌ false (참조 비교)
  → @ValueBased 클래스에서 ==는 정의되지 않음
  → 해결: opt1.equals(opt2)

실수 3: Optional 서브클래스 작성
  class CustomOptional<T> extends Optional<T> { ... }  // ❌ 컴파일 에러
  → Optional은 final class
  → 설계자의 의도적 제한

실수 4: Optional을 컬렉션에 저장
  List<Optional<String>> list = new ArrayList<>();
  list.add(Optional.of("value"));
  → Optional 래핑의 레이어 증가, 읽기 성능 저하
  → 해결: null을 허용하는 List 또는 빈 리스트 사용
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// Optional 올바른 설계 패턴

// 패턴 1: 메서드 반환 타입으로만 사용
public Optional<User> findUserById(long id) {
    return userRepository.findById(id);
}

// 호출 시:
Optional<User> user = findUserById(1L);
user.ifPresent(u -> System.out.println(u.getName()));

// 패턴 2: 필드는 일반 타입 + Optional 반환 메서드
class User {
    private String email;  // null 가능하지만 필드는 일반 타입
    
    public Optional<String> getEmail() {
        return Optional.ofNullable(email);
    }
}

// 패턴 3: 필터링과 값 추출
Optional<String> result = findUserById(1L)
    .filter(u -> u.isActive())
    .map(User::getEmail)
    .filter(email -> email.contains("@"))
    .or(() -> Optional.of("default@example.com"));

System.out.println(result.orElse("unknown"));

// 패턴 4: null이 가능한 컬렉션
List<String> emails = users.stream()
    .flatMap(u -> u.getEmail().stream())  // Optional → Stream
    .collect(Collectors.toList());

// flatMap(Optional::stream)으로 Optional을 Stream으로 전환
// 빈 Optional → 빈 Stream → 결과에서 제외
```

---

## 🔬 내부 동작 원리

### 1. Optional의 단순한 구조

```java
// OpenJDK Optional.java (단순화)
public final class Optional<T> {
    
    private static final Optional<?> EMPTY = new Optional<>(null);
    
    private final T value;  // ← 단 하나의 필드!
    
    private Optional(T value) {
        this.value = value;
    }
    
    // 정적 팩토리 메서드
    @SuppressWarnings("unchecked")
    public static <T> Optional<T> empty() {
        return (Optional<T>) EMPTY;  // 싱글톤 상수 재사용!
    }
    
    public static <T> Optional<T> of(T value) {
        return new Optional<>(Objects.requireNonNull(value));
    }
    
    public static <T> Optional<T> ofNullable(T value) {
        return value == null ? empty() : of(value);
    }
    
    // 값 추출
    public T get() {
        if (value == null) {
            throw new NoSuchElementException("No value present");
        }
        return value;
    }
    
    public T orElse(T other) {
        return value != null ? value : other;
    }
}

메모리 구조:
  Optional<String> opt1 = Optional.of("hello");
  
  힙:
  ┌─ Optional 객체 ────────────┐
  │ value → "hello" (String 참조) │
  └────────────────────────────┘
  
  Optional<String> opt2 = Optional.empty();
  
  힙 (공유):
  ┌─ Optional 싱글톤 EMPTY ─┐
  │ value → null           │
  └───────────────────────┘
  opt2 → EMPTY (같은 참조)
  
메모리 절감:
  빈 Optional이 매번 새 인스턴스를 만들지 않고 EMPTY를 재사용
  → 빈 값이 많은 데이터에서 메모리 절약
```

### 2. @ValueBased 어노테이션

```java
@jdk.internal.ValueBased
public final class Optional<T> {
    // ...
}

@ValueBased의 의미:
  - Optional은 "값 기반 클래스" (value-based class)
  - 인스턴스 필드 상태만으로 동등성 결정 (참조 ID 무관)
  - == 비교는 정의되지 않음 (문법적으로는 가능하지만 의도와 맞지 않음)
  - 향후 JVM 최적화 대상 (valhalla 프로젝트)

Optional.equals() 구현:
  @Override
  public boolean equals(Object obj) {
      if (!(obj instanceof Optional)) {
          return false;
      }
      Optional<?> other = (Optional<?>) obj;
      return Objects.equals(value, other.value);  // 값 비교
  }

Examples:
  Optional<String> opt1 = Optional.of("A");
  Optional<String> opt2 = Optional.of("A");
  
  opt1 == opt2       // false (참조 비교)
  opt1.equals(opt2)  // true  (값 비교)
  
  // 싱글톤 EMPTY의 경우만 == true
  Optional<String> empty1 = Optional.empty();
  Optional<String> empty2 = Optional.empty();
  empty1 == empty2   // true (같은 EMPTY 상수)
```

### 3. final class의 설계 의도

```
왜 Optional이 final인가?

가능한 서브클래싱의 문제:
  class OptionalExt<T> extends Optional<T> {
      private String description;  // 추가 필드
      
      public OptionalExt(T value, String desc) {
          super(value);  // 문제: super 생성자는 package-private
          this.description = desc;
      }
  }

문제점:
  1. 직렬화 불가: Optional이 Serializable 미구현 → 서브클래스도 불가
  2. 값 기반 의미론 위반: 서브클래스가 추가 필드 → equals/hashCode 모호
  3. JVM 최적화 방해: Valhalla의 값 타입 최적화는 final 클래스만 가능
  4. API 안정성: 서브클래싱 가능 → Optional의 내부 최적화 불가능
     (예: EMPTY 싱글톤이 깨짐)

설계자의 철학:
  Optional은 "값을 담는 컨테이너"일 뿐, 상속해서 기능 확장할 대상이 아님
  필요하면 Optional을 조합하거나 래핑하라
  
  // 좋은 예: Optional 조합
  Optional<Optional<String>> nested = findUser().map(User::getEmail);
  Optional<String> email = nested.flatMap(Function.identity());
  
  // 나쁜 예: Optional 서브클래싱 (불가능)
  // class MyOptional<T> extends Optional<T> { ... }  ❌
```

### 4. Serializable 미구현의 영향

```java
// Optional은 Serializable을 구현하지 않음
public final class Optional<T> {  // Serializable 없음
    private final T value;
}

직렬화 시도 시:
  Optional<String> opt = Optional.of("test");
  ObjectOutputStream oos = new ObjectOutputStream(outputStream);
  oos.writeObject(opt);  // NotSerializableException!

왜 Serializable을 구현하지 않았는가?

이유 1: 필드에 쓸 수 없도록 강제
  class User {
      Optional<String> email;  // 직렬화 불가 → 컴파일 에러 위험
  }
  → Optional을 필드로 쓰는 안티패턴 차단

이유 2: 값 기반 의미론
  Optional은 "값"이지 "객체"가 아님
  → 참조 ID 기반 직렬화는 의미 없음
  → 값만 직렬화하면 되므로 Optional 래핑 자체는 불필요

해결책: Jackson의 jackson-datatype-jdk8
  <dependency>
      <groupId>com.fasterxml.jackson.datatype</groupId>
      <artifactId>jackson-datatype-jdk8</artifactId>
  </dependency>
  
  ObjectMapper mapper = new ObjectMapper()
      .registerModule(new JavaTimeModule());  // Optional 지원
  
  String json = mapper.writeValueAsString(user);
  // Optional<String> email는 "email" 필드로 직렬화 (null 또는 값)

JPA/Hibernate의 경우:
  직렬화 불가 → Optional을 엔티티 필드로 쓸 수 없음
  @Entity
  class User {
      private String email;  // ← 직렬화 가능
      
      @Transient
      public Optional<String> getEmail() {
          return Optional.ofNullable(email);
      }
  }
```

---

## 💻 실전 실험

### 실험 1: EMPTY 싱글톤 확인

```java
import java.lang.reflect.*;

public class OptionalEmptySingletonTest {
    public static void main(String[] args) throws Exception {
        Optional<String> empty1 = Optional.empty();
        Optional<String> empty2 = Optional.empty();
        Optional<Integer> empty3 = Optional.empty();
        
        // 싱글톤 확인
        System.out.println("empty1 == empty2: " + (empty1 == empty2));      // true
        System.out.println("empty1 == empty3: " + (empty1 == empty3));      // true!
        
        // EMPTY 상수에 접근
        Field emptyField = Optional.class.getDeclaredField("EMPTY");
        emptyField.setAccessible(true);
        Object empty = emptyField.get(null);
        
        System.out.println("empty1 객체 주소: " + System.identityHashCode(empty1));
        System.out.println("EMPTY 객체 주소: " + System.identityHashCode(empty));
        System.out.println("같은 객체: " + (empty1 == empty));  // true
        
        // 메모리: 모든 빈 Optional이 하나의 EMPTY 싱글톤을 공유
    }
}
```

### 실험 2: @ValueBased와 == vs equals

```java
public class ValueBasedTest {
    public static void main(String[] args) {
        Optional<String> opt1 = Optional.of("test");
        Optional<String> opt2 = Optional.of("test");
        Optional<String> opt3 = Optional.of("test");
        
        // == 비교 (참조 비교)
        System.out.println("opt1 == opt2: " + (opt1 == opt2));  // false
        System.out.println("opt1 == opt3: " + (opt1 == opt3));  // false
        
        // equals 비교 (값 비교)
        System.out.println("opt1.equals(opt2): " + opt1.equals(opt2));  // true
        System.out.println("opt1.equals(opt3): " + opt1.equals(opt3));  // true
        
        // Set의 동등성 확인 (equals 사용)
        Set<Optional<String>> set = new HashSet<>();
        set.add(opt1);
        set.add(opt2);
        set.add(opt3);
        System.out.println("Set 크기: " + set.size());  // 1 (equals로 중복 제거)
        
        // @ValueBased의 의미
        System.out.println("\n@ValueBased: 값 기반 동등성");
        Optional<String> empty1 = Optional.empty();
        Optional<String> empty2 = Optional.empty();
        System.out.println("empty1 == empty2: " + (empty1 == empty2));    // true (싱글톤)
        System.out.println("empty1.equals(empty2): " + empty1.equals(empty2)); // true
    }
}
```

### 실험 3: 직렬화 불가 확인

```java
import java.io.*;

public class OptionalSerializationTest {
    public static void main(String[] args) throws Exception {
        Optional<String> opt = Optional.of("serializable?");
        
        // 직렬화 시도
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(baos);
        
        try {
            oos.writeObject(opt);
            System.out.println("직렬화 성공!");
        } catch (NotSerializableException e) {
            System.out.println("직렬화 실패: " + e.getClass().getSimpleName());
            System.out.println("메시지: " + e.getMessage());
            // java.util.Optional
        }
        oos.close();
    }
}

// 출력:
// 직렬화 실패: NotSerializableException
// 메시지: java.util.Optional
```

### 실험 4: 필드와 메서드 반환의 차이

```java
import java.io.*;

public class OptionalFieldVsReturn {
    
    // ❌ 안티패턴: Optional을 필드로 사용
    static class BadDesign {
        Optional<String> email;  // 문제 있는 설계
        
        public BadDesign(String email) {
            this.email = Optional.ofNullable(email);
        }
    }
    
    // ✅ 올바른 패턴: Optional은 반환 타입만
    static class GoodDesign {
        private String email;  // 필드는 일반 타입
        
        public GoodDesign(String email) {
            this.email = email;
        }
        
        public Optional<String> getEmail() {
            return Optional.ofNullable(email);
        }
    }
    
    public static void main(String[] args) {
        // BadDesign의 문제점
        System.out.println("BadDesign 메모리 구조:");
        BadDesign bad = new BadDesign("test@example.com");
        System.out.println("  - String: ~56 바이트 (헤더 12 + 44)");
        System.out.println("  - Optional: 16 바이트 (헤더 12 + 4 참조)");
        System.out.println("  - 총 추가 메모리: ~16 바이트 (Optional 래핑)");
        
        // GoodDesign의 이점
        System.out.println("\nGoodDesign 메모리 구조:");
        GoodDesign good = new GoodDesign("test@example.com");
        System.out.println("  - String: ~56 바이트만");
        System.out.println("  - Optional은 메서드 반환 시만 생성 (임시)");
        System.out.println("  - 저장 메모리: Optional 오버헤드 없음");
        
        // 사용 관점의 명확성
        System.out.println("\nAPI 명확성:");
        System.out.println("BadDesign: email 필드가 null 가능한지 불명확");
        System.out.println("GoodDesign: getEmail()의 Optional 반환 → null 가능을 명시");
    }
}
```

---

## 📊 성능/비교

```
Optional 메모리 오버헤드:

필드로 Optional 사용:
  class User {
      Optional<String> email;  // 추가 메모리
  }
  
  User 객체 메모리:
    필드 offset 0 (email 참조): 8 바이트
    Optional 객체 (별도): 16 바이트
      - 객체 헤더: 12 바이트
      - value 필드: 4 바이트 (참조, 실제는 8 on 64-bit)
    
  결과: User 당 Optional 래핑으로 8~24 바이트 추가 메모리

메서드 반환으로 Optional 사용:
  public Optional<String> getEmail() {
      return Optional.ofNullable(email);
  }
  
  메모리: 호출 시에만 Optional 임시 생성 → GC 대상
  1,000,000명의 User 리스트 + getEmail() 호출:
    - 필드: 1M × 16 바이트 = 16 MB (메모리에 계속 존재)
    - 반환: ~임시 객체 → GC 수거 → 메모리 절감

성능 비교 (수백만 건 읽기):
  
  필드 직접 접근: user.email
    메모리: 필드 + Optional 객체 모두 존재
    캐시: null check, Optional 필드 접근 (extra indirection)
  
  메서드 반환: user.getEmail()
    메모리: 필드만 + Optional 호출 시 생성 (JIT 최적화 가능)
    캐시: getEmail() JIT 인라인 → Optional 생성 제거 가능

EMPTY 싱글톤의 이점:
  Optional<String> empty1 = Optional.empty();  // EMPTY 반환
  Optional<String> empty2 = Optional.empty();  // 같은 EMPTY 반환
  
  메모리: 1개 EMPTY 객체 재사용 (new 불필요)
  대규모 스트림 필터링에서 빈 Optional이 많으면 메모리 절약
```

---

## ⚖️ 트레이드오프

```
Optional의 설계 제약:

final class:
  장점: JVM 최적화 가능, 설계 의도 강화, 싱글톤 EMPTY 보호
  단점: 커스텀 Optional 확장 불가

Serializable 미구현:
  장점: 필드 사용 안티패턴 차단, 직렬화 복잡성 회피
  단점: JPA 엔티티에서 사용 불가, 직렬화 필요 시 래핑 필요

단일 value 필드:
  장점: 메모리 경량, 단순한 의미론
  단점: 부가 정보 저장 불가 (예: 설명 또는 실패 원인)

@ValueBased 제약:
  장점: 값 기반 의미론 명시, 향후 최적화 여지
  단점: == 비교 사용 불가 (사용 가능하지만 의도와 맞지 않음)
```

---

## 📌 핵심 정리

```
Optional 내부 구조 핵심:

구조:
  - 단 하나의 필드: private final T value
  - EMPTY 싱글톤: 모든 빈 Optional이 재사용
  - final class: 상속 금지, JVM 최적화 보호

값 기반 클래스:
  - @ValueBased: 값 기반 동등성
  - equals(): 값 비교 (O == o로 비교하지 말 것)
  - hashCode(): 값 기반

직렬화:
  - Serializable 미구현 (의도적)
  - 필드에 쓸 수 없도록 강제
  - Jackson/JPA 처리 필요

설계 철학:
  - 메서드 반환 타입으로만 사용
  - 필드는 일반 타입 + Optional 반환 메서드
  - 값 컨테이너일 뿐, 확장 대상이 아님
```

---

## 🤔 생각해볼 문제

**Q1.** Optional.empty()가 호출될 때마다 새 인스턴스를 만들지 않고 EMPTY 싱글톤을 반환하는 방식의 장단점은?

<details>
<summary>해설 보기</summary>

장점:
1. 메모리 절약: 빈 Optional이 수백만 개 생성되어도 EMPTY 객체 하나만 존재
2. 캐시 효율: 반복되는 empty() 호출이 항상 같은 객체 반환 → CPU 캐시 히트율 향상
3. 객체 생성 오버헤드 제거: new Optional(null) 없음 → GC 부담 감소

단점:
1. 제네릭 타입 정보 손실: Optional<String>.empty()와 Optional<Integer>.empty()가 사실 같은 객체
   - 타입 안전성은 컴파일러 차원에서 보장
   - 런타임 타입 검사는 불가능
2. 스레드 안전성: EMPTY가 전역 상태이지만, 불변이므로 문제 없음
3. 캐시 일관성 문제 없음: EMPTY는 수정되지 않음 (value는 null, 변경 불가)

결론: 빈 Optional이 자주 생성되는 상황(필터링, 매핑)에서 매우 효율적. 다만 Optional<T>의 제네릭 타입은 컴파일 타임에만 의미 있음.

</details>

---

**Q2.** Optional을 클래스 필드로 사용하면 안 되는 이유를 설계 관점에서 설명하시오.

<details>
<summary>해설 보기</summary>

세 가지 근본적 이유:

1. 직렬화/역직렬화 불가
   - Optional은 Serializable을 구현하지 않음
   - JPA, JSON 직렬화, 분산 캐시 등에서 예외 발생
   - class User { Optional<String> email; }로 설계 → ORM 매핑 불가
   - Optional은 의도적으로 필드 사용을 차단

2. 메모리 오버헤드
   - Optional 래핑으로 매 객체마다 16~24 바이트 추가
   - User 1,000,000명 → 16~24 MB 낭비
   - 메서드 반환으로 사용 시 임시 객체 → GC 수거 가능

3. 의미 모호성 및 API 불명확성
   - User.email = Optional.of("test") vs User.email = null
   - null과 Optional.empty()의 의미 분리 실패
   - 호출자는 이 필드가 null 가능한지, Optional인지 파악 어려움
   - public Optional<String> getEmail()로 반환 → 의도 명시적

해결책:
  class User {
      private String email;  // 필드는 null 가능해도 Optional 없음
      
      public Optional<String> getEmail() {
          return Optional.ofNullable(email);
      }
  }
  
이 패턴이 Optional의 설계자(Brian Goetz) 의도임.

</details>

---

**Q3.** @ValueBased 클래스에서 `==` 대신 `equals()`를 사용해야 하는 이유는?

<details>
<summary>해설 보기</summary>

@ValueBased의 계약:
  Optional은 "값 기반" 클래스 → 인스턴스의 의미는 state(value 필드)에만 의존
  참조 ID(메모리 주소)는 무의미

== 비교의 문제:
  Optional<String> opt1 = Optional.of("A");
  Optional<String> opt2 = Optional.of("A");
  
  opt1 == opt2  // false (다른 메모리 주소)
  
  그런데 opt1과 opt2는 동일한 값 "A"를 담고 있음
  → == 비교는 의미적으로 틀림

equals() 비교의 정확성:
  opt1.equals(opt2)  // true (value 필드가 동일)
  
  Optional.equals() 구현:
    return value != null ? value.equals(other.value) : other.value == null;
  
  값 기반 의미론을 정확히 구현

향후 JVM 최적화:
  Valhalla 프로젝트(값 타입 지원)에서 @ValueBased는
  "이 클래스는 향후 원시 타입처럼 취급될 수 있다"는 신호
  → == 비교는 정의되지 않거나 예측 불가능할 수 있음
  
  따라서 현재 코드에서 == 사용 → 향후 호환성 문제 가능

결론: @ValueBased 클래스는 항상 equals() 사용. EMPTY 싱글톤 예외 제외.

</details>

---

<div align="center">

**[⬅️ 이전 챕터: NQ Model](../chapter03-parallel-stream/05-nq-model-threshold.md)** | **[홈으로 🏠](../README.md)** | **[다음: Optional.of vs ofNullable vs empty ➡️](./02-of-vs-ofnullable-vs-empty.md)**

</div>
