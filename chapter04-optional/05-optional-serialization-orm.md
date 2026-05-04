# 직렬화·ORM·Optional — 실전 통합

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Optional이 Serializable을 구현하지 않은 결과는 무엇인가?
- JPA 엔티티 필드에 Optional을 쓰면 안 되는 이유는?
- Hibernate는 Optional을 어떻게 처리하는가? (실패 케이스)
- Jackson의 Optional 직렬화 모듈(jackson-datatype-jdk8)은 무엇을 해결하는가?
- GraphQL의 `null vs missing` 구분과 Optional의 한계는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

실제 프로젝트에서 Optional은 ORM, JSON 직렬화, 원격 API 호출 등 여러 계층을 통과해야 한다. Optional이 Serializable을 구현하지 않은 설계는 이런 실무 환경에서 명확한 제약을 만든다. Jackson, Hibernate, GraphQL 같은 실제 도구들과의 상호작용을 이해하지 못하면, 엔티티 설계부터 API 응답까지 여러 단계에서 문제가 발생한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: JPA 엔티티에 Optional 필드 사용
  @Entity
  @Table(name = "users")
  class User {
      @Id
      long id;
      
      @Column(nullable = true)
      Optional<String> email;  // ❌ Serializable 아님!
  }
  
  문제:
    - Hibernate: NotSerializableException 또는 매핑 오류
    - JSON 응답: Optional이 어떻게 직렬화되나?
    - DB 캐시: Optional 객체를 DB에 어떻게 저장?

실수 2: JSON 응답에서 Optional이 그대로 나온다
  @RestController
  class UserController {
      @GetMapping("/{id}")
      User getUser(@PathVariable Long id) {
          Optional<User> user = userService.findById(id);
          return user.get();  // Optional을 까고 반환
          // 또는 user.orElseThrow();
      }
  }
  
  응답 JSON:
  {
    "id": 1,
    "email": "test@example.com"  // Optional 래핑 없음 (OK)
  }
  
  하지만:
  @GetMapping("/{id}/email")
  Optional<String> getEmail(@PathVariable Long id) {  // ❌ Optional 반환
      return userService.getEmail(id);
  }
  
  응답 JSON:
  {
    "value": "test@example.com",  // Optional이 직렬화됨 (가짜 구조!)
    "present": true
  }
  또는 "test@example.com" (Jackson jackson-datatype-jdk8 필요)

실수 3: Hibernate가 Optional 컬럼 타입을 모른다
  @Entity
  class Order {
      @Column(columnDefinition = "VARCHAR(50)")  // ❌ Optional은 타입이 아님
      Optional<String> trackingNumber;
  }
  
  Hibernate:
    - Optional은 자바 타입, SQL 타입이 아님
    - VARCHAR(50)를 어디에 매핑? Optional.value에?
    - 역직렬화 시 어떻게 Optional을 복원?

실수 4: GraphQL에서 null의 의미가 모호
  # GraphQL Schema
  type User {
    id: ID!
    email: String!  # non-null
    phone: String   # nullable (null 가능)
    bio: String     # nullable
  }
  
  Query에서:
  {
    user(id: "1") {
      id
      email
      phone  # 응답이 null → "전화 없음" 또는 "데이터 오류"?
      bio    # 응답이 없음 (필드 생략) vs null?
    }
  }
  
  Java에서 Optional로 이를 표현하기는 어려움
  → null / Optional.empty() / 필드 누락의 구분 불명확

실수 5: Spring Data Redis에 Optional을 캐시하기
  @Cacheable("user_cache")
  public Optional<User> findById(Long id) {
      return userRepository.findById(id);
  }
  
  문제:
    - Redis 직렬화: Optional.java 직렬화 실패
    - Jackson 캐시 모듈: Optional이 뭔지 몰라 오류
    - 캐시 히트 판정: Optional.empty() vs 캐시 미스?
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// Optional을 고려한 ORM/직렬화 설계 패턴

// 패턴 1: JPA 엔티티는 Optional 필드 없음
@Entity
@Table(name = "users")
class User {
    @Id
    long id;
    
    @Column(nullable = false)
    String name;
    
    @Column(nullable = true)  // nullable이지만 Optional 아님
    String email;
    
    @Column(nullable = true)
    String phone;
    
    // 생성자, getter/setter
    public User(long id, String name, String email, String phone) {
        this.id = id;
        this.name = Objects.requireNonNull(name);
        this.email = email;  // null 허용
        this.phone = phone;  // null 허용
    }
    
    // ✅ Optional은 메서드 반환에만
    public Optional<String> getEmail() {
        return Optional.ofNullable(email);
    }
    
    public Optional<String> getPhone() {
        return Optional.ofNullable(phone);
    }
}

// 패턴 2: REST 컨트롤러에서 Optional 처리
@RestController
@RequestMapping("/api/users")
class UserController {
    
    @GetMapping("/{id}")
    public UserResponse getUser(@PathVariable Long id) {
        Optional<User> user = userService.findById(id);
        return user
            .map(UserResponse::from)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
    
    @GetMapping("/{id}/email")
    public Optional<String> getEmail(@PathVariable Long id) {
        // ❌ Optional을 JSON으로 직렬화하지 않으려면
        // ResponseEntity나 DTO를 사용
        return userService.getEmail(id);
    }
    
    // ✅ 올바른 방법 1: DTO 사용
    @GetMapping("/{id}/email")
    public EmailResponse getEmailResponse(@PathVariable Long id) {
        Optional<String> email = userService.getEmail(id);
        return new EmailResponse(email.orElse(null));
    }
    
    // ✅ 올바른 방법 2: ResponseEntity 사용
    @GetMapping("/{id}/email")
    public ResponseEntity<String> getEmailAsResponse(@PathVariable Long id) {
        Optional<String> email = userService.getEmail(id);
        return email
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
}

// 패턴 3: Jackson 모듈로 Optional 직렬화
// pom.xml
// <dependency>
//     <groupId>com.fasterxml.jackson.datatype</groupId>
//     <artifactId>jackson-datatype-jdk8</artifactId>
// </dependency>

@Configuration
class JacksonConfig {
    @Bean
    ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.registerModule(new JavaTimeModule());
        mapper.registerModule(new Jdk8Module());  // Optional 지원
        mapper.setSerializationInclusion(JsonInclude.Include.NON_ABSENT);
        // NON_ABSENT: Optional.empty()는 필드 생략, null도 필드 생략
        return mapper;
    }
}

// 패턴 4: JPA with Converter (Optional을 처리하려면)
@Entity
class Order {
    @Id
    long id;
    
    // ❌ Optional 필드 사용 안 함
    // ✅ Converter로 처리
    @Convert(converter = StringOptionalConverter.class)
    String trackingNumber;  // nullable
    
    public Optional<String> getTrackingNumber() {
        return Optional.ofNullable(trackingNumber);
    }
}

class StringOptionalConverter
    implements AttributeConverter<Optional<String>, String> {
    
    @Override
    public String convertToDatabaseColumn(Optional<String> attribute) {
        return attribute.orElse(null);  // Optional → DB null
    }
    
    @Override
    public Optional<String> convertToEntityAttribute(String dbData) {
        return Optional.ofNullable(dbData);  // DB → Optional
    }
}

// 패턴 5: Redis 캐시와 Optional
@Configuration
@EnableCaching
class CacheConfig {
    
    @Bean
    RedisCacheManager cacheManager(RedisConnectionFactory cf) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()
                )
            );
        return RedisCacheManager.create(cf);
    }
}

@Service
class UserService {
    
    @Cacheable("users")
    public Optional<User> findById(Long id) {
        // 문제: Optional.empty()는 캐시하면 안 됨 (캐시 미스와 구분 불가)
        return repository.findById(id);  // Optional<User> 반환
    }
    
    // ✅ 좋은 방법: Optional을 벗기고 캐시
    @Cacheable("user_by_id")
    public User findByIdDirect(Long id) {
        return repository.findById(id)
            .orElse(null);  // 캐시: null 또는 User 객체
    }
    
    public Optional<User> findById(Long id) {
        // null 대신 Optional.empty()로 변환
        return Optional.ofNullable(findByIdDirect(id));
    }
}

// 패턴 6: GraphQL과 Optional
@Configuration
class GraphQLConfig {
    
    @Bean
    GraphQLSchema graphQLSchema() {
        return GraphQLSchema.newSchema()
            .query(QueryType.newQuery()
                .field(GraphQLField.newField()
                    .name("user")
                    .type(GraphQLNonNull.nonNull(UserType.USER))  // non-null User
                    .argument(GraphQLArgument.newArgument()
                        .name("id")
                        .type(GraphQLNonNull.nonNull(GraphQLLong.GRAPHQL_LONG))
                        .build())
                    .build())
                .build())
            .build();
    }
}

// GraphQL Resolver에서 Optional 처리
@Component
class UserResolver implements GraphQLQueryResolver {
    
    public User user(Long id) {
        // GraphQL: Optional을 사용하지 않고 null로 반환
        return userService.findById(id)
            .orElse(null);  // null → GraphQL error 또는 필드 null
    }
    
    @GraphQLField
    @GraphQLNonNull
    public User resolveUser(@GraphQLArgument Long id) {
        // non-null 필드: null 반환 불가 → exception
        return userService.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Optional과 Java 직렬화

```java
// Optional은 Serializable 미구현
public final class Optional<T> {
    // ... Serializable 없음
    private final T value;
}

직렬화 시도:
  Optional<String> opt = Optional.of("test");
  ObjectOutputStream oos = new ObjectOutputStream(file);
  oos.writeObject(opt);  // NotSerializableException!

왜 미구현했는가?

설계 의도:
  1. Optional은 "값 컨테이너"일 뿐, 저장 대상이 아님
  2. 필드에 쓰면 안 된다는 신호
  3. 값만 직렬화하면 되므로 Optional 자체는 불필요

결과:
  Optional<T>는 임시 오브젝트 (메서드 반환 시만 생성)
  영구 저장(DB, 캐시, 파일)이 필요하면 value만 추출

해결책:
  - Jackson jackson-datatype-jdk8: Optional → JSON 변환
  - JPA @Transient: Optional 필드는 매핑 제외
  - 커스텀 Converter: Optional<T> ↔ T 변환
```

### 2. Jackson과 Optional

```java
// 기본 Jackson (jackson-datatype-jdk8 없음)
ObjectMapper mapper = new ObjectMapper();

Optional<String> opt = Optional.of("value");
String json = mapper.writeValueAsString(opt);
// 출력: {"value":"value","present":true} (내부 필드 노출)

// jackson-datatype-jdk8 추가
mapper.registerModule(new Jdk8Module());

json = mapper.writeValueAsString(opt);
// 출력: "value" (값만 추출)

json = mapper.writeValueAsString(Optional.empty());
// 출력: null (empty()는 null로 표현)

설정: SerializationFeature.WRITE_ABSENT_AS_NULL
  - NON_ABSENT: Optional.empty()를 필드에서 제외
  - INCLUDE_ALWAYS: Optional.empty()를 null로 포함

역직렬화:
  String json = "\"hello\"";
  Optional<String> opt = mapper.readValue(json, new TypeReference<Optional<String>>(){});
  // 결과: Optional.of("hello")
  
  String nullJson = "null";
  Optional<String> opt2 = mapper.readValue(nullJson, new TypeReference<Optional<String>>(){});
  // 결과: Optional.empty()

내부 동작:
  Jdk8Module은 StdDeserializer<Optional<?>>와 StdSerializer<Optional<?>>를 등록
  
  직렬화:
    Optional.isPresent() ? 값 직렬화 : null 처리
  
  역직렬화:
    값 → Optional.of(값)
    null → Optional.empty()
```

### 3. Hibernate와 Optional

```java
// Hibernate는 Optional을 지원하지 않음
@Entity
class User {
    @Column
    Optional<String> email;  // ❌ 매핑 불가
}

Hibernate 동작:
  1. @Column 분석
  2. Optional의 타입 파라미터 <String> 인식 시도
  3. 하지만 Optional.value 필드가 private final → 접근 불가
  4. ClassLoadingException 또는 MappingException

Hibernate 6.2+의 개선:
  // 일부 버전에서 Optional 지원 추가 (제한적)
  @Column(columnDefinition = "VARCHAR(255)")
  Optional<String> email;
  
  하지만:
  - Optional을 자동 언래핑하는 로직 필요
  - JSON 직렬화와의 혼동 가능성
  - 권장되지 않음

올바른 패턴:
  @Entity
  class User {
      @Column(nullable = true)
      String email;  // 일반 필드
      
      @Transient  // JPA 매핑 제외
      public Optional<String> getEmail() {
          return Optional.ofNullable(email);
      }
  }
  
  동작:
  1. Hibernate가 email (String) 매핑
  2. getEmail() 호출 시에만 Optional 생성
  3. 직렬화: Optional 포함 안 됨 (Transient)
  4. JSON 응답: getEmail() 반환값 직렬화
     Jackson이 Optional을 처리 (jackson-datatype-jdk8)
```

### 4. GraphQL의 null vs missing

```
GraphQL의 세 가지 상태:

1. Non-null 필드 (String!)
   {
     name: "Alice"  // ✅ 값 있음
     name: null     // ❌ 불가능 (타입 위반)
   }

2. Nullable 필드 (String)
   {
     email: "alice@example.com"  // ✅ 값 있음
     email: null                 // ✅ 명시적 null
     email: (필드 생략)           // ⚠️ missing (다른 의미)
   }

3. Missing vs Null의 구분:
   사용자의 이메일이 없는 경우:
   - null: "이메일 필드가 있고, 그 값이 null"
   - missing: "이메일 필드를 요청했지만 응답에 없음"
   
   JSON으로 표현:
   null 응답:
     {"id": 1, "email": null}  // 필드 존재, 값 null
   
   missing 응답:
     {"id": 1}  // email 필드 자체가 없음

Java/Optional의 한계:
  Java Optional은 null과 empty의 구분만 가능
  GraphQL의 null과 missing을 구분할 방법 없음
  
  // Java 객체
  class User {
      String email;  // null 가능
  }
  
  // GraphQL 응답
  case 1: null 반환
    {"id": 1, "email": null}
  
  case 2: 필드 생략 (missing)
    Jackson @JsonInclude(NON_NULL) 설정으로 가능
    → 값이 null이면 필드 생략
    → 하지만 "null인 상황"과 "데이터 없는 상황"을 구분 못 함

Optional로 구분하려면:
  class User {
      String email;  // null이면 이메일 없음
  }
  
  public Optional<String> getEmail() {
      return Optional.ofNullable(email);
  }
  
  // GraphQL에서:
  // Optional.of(값) → "email": "값"
  // Optional.empty() → "email" 필드 생략
  // 하지만 여전히 명시적 null과의 구분 불가능

GraphQL 전문가의 해결책:
  명시적인 필드 추가:
  type User {
    id: ID!
    email: String
    emailProvided: Boolean!  // null vs missing 구분
  }
  
  또는 Union 타입:
  union EmailResult = EmailValue | NoEmail
  
  type EmailValue {
    value: String!
  }
  
  type NoEmail {
    reason: String
  }
```

---

## 💻 실전 실험

### 실험 1: Jackson Optional 직렬화

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.databind.JsonInclude;
import com.fasterxml.jackson.datatype.jdk8.Jdk8Module;
import java.util.Optional;

public class JacksonOptionalTest {
    
    static class User {
        long id;
        String name;
        Optional<String> email;
        Optional<String> phone;
        
        User(long id, String name, String email, String phone) {
            this.id = id;
            this.name = name;
            this.email = Optional.ofNullable(email);
            this.phone = Optional.ofNullable(phone);
        }
    }
    
    public static void main(String[] args) throws Exception {
        System.out.println("=== Jackson 기본 (Jdk8Module 없음) ===");
        ObjectMapper basicMapper = new ObjectMapper();
        
        User user = new User(1, "Alice", "alice@example.com", null);
        String basicJson = basicMapper.writeValueAsString(user);
        System.out.println(basicJson);
        // {"id":1,"name":"Alice","email":{"value":"alice@example.com","present":true},"phone":{"present":false}}
        
        System.out.println("\n=== Jackson + Jdk8Module ===");
        ObjectMapper advancedMapper = new ObjectMapper();
        advancedMapper.registerModule(new Jdk8Module());
        
        String advancedJson = advancedMapper.writeValueAsString(user);
        System.out.println(advancedJson);
        // {"id":1,"name":"Alice","email":"alice@example.com","phone":null}
        
        System.out.println("\n=== Jackson + Jdk8Module + NON_ABSENT ===");
        ObjectMapper smartMapper = new ObjectMapper();
        smartMapper.registerModule(new Jdk8Module());
        smartMapper.setSerializationInclusion(JsonInclude.Include.NON_ABSENT);
        
        String smartJson = smartMapper.writeValueAsString(user);
        System.out.println(smartJson);
        // {"id":1,"name":"Alice","email":"alice@example.com"}
        // phone 필드가 absent (null)이므로 생략됨
        
        System.out.println("\n=== 역직렬화 ===");
        String inputJson = "{\"id\":2,\"name\":\"Bob\",\"email\":null}";
        User reversedUser = smartMapper.readValue(inputJson, User.class);
        System.out.println("id: " + reversedUser.id);
        System.out.println("name: " + reversedUser.name);
        System.out.println("email: " + reversedUser.email);  // Optional.empty
        System.out.println("phone: " + reversedUser.phone);  // null (역직렬화 불가능)
    }
}

// 출력:
// === Jackson 기본 (Jdk8Module 없음) ===
// {"id":1,"name":"Alice","email":{"value":"alice@example.com","present":true},"phone":{"present":false}}
//
// === Jackson + Jdk8Module ===
// {"id":1,"name":"Alice","email":"alice@example.com","phone":null}
//
// === Jackson + Jdk8Module + NON_ABSENT ===
// {"id":1,"name":"Alice","email":"alice@example.com"}
//
// === 역직렬화 ===
// id: 2
// name: Bob
// email: Optional.empty
// phone: null (역직렬화 불가능)
```

### 실험 2: JPA Converter 사용

```java
import jakarta.persistence.*;
import java.util.Optional;

@Entity
@Table(name = "orders")
class Order {
    @Id
    long id;
    
    String orderId;
    
    // ❌ 직접 Optional 필드 사용 대신
    // ✅ Converter 사용
    @Convert(converter = StringOptionalConverter.class)
    @Column(name = "tracking_number", nullable = true)
    String trackingNumber;
    
    public Optional<String> getTrackingNumber() {
        return Optional.ofNullable(trackingNumber);
    }
}

@Converter(autoApply = true)
class StringOptionalConverter
    implements AttributeConverter<Optional<String>, String> {
    
    @Override
    public String convertToDatabaseColumn(Optional<String> attribute) {
        System.out.println("DB에 저장: " + attribute);
        return attribute.orElse(null);
    }
    
    @Override
    public Optional<String> convertToEntityAttribute(String dbData) {
        System.out.println("엔티티로 로드: " + Optional.ofNullable(dbData));
        return Optional.ofNullable(dbData);
    }
}

public class JpaConverterTest {
    public static void main(String[] args) {
        System.out.println("=== JPA Converter로 Optional 처리 ===");
        
        // 실제 Hibernate 사용 시:
        // Optional<String> tracking = order.getTrackingNumber();
        // → Optional.ofNullable(데이터베이스값)
        
        // setTrackingNumber는 없음 (필드가 String이므로)
        // order.trackingNumber = "TRK123";  // SQL: UPDATE orders SET tracking_number = 'TRK123'
        
        System.out.println("Converter는 String ↔ Optional 변환을 담당");
        System.out.println("필드는 String으로 유지 (SQL과 호환)");
    }
}
```

### 실험 3: GraphQL의 null vs missing

```
GraphQL 스키마:
  type User {
    id: ID!
    name: String!
    email: String
    phone: String
  }

Java 구현:
  class User {
      long id;
      String name;
      String email;  // null 가능
      String phone;  // null 가능
  }

쿼리:
  {
    user(id: "1") {
      id
      name
      email
      phone
    }
  }

GraphQL 응답 시나리오:

시나리오 1: email = "alice@example.com", phone = null
  {"id": "1", "name": "Alice", "email": "alice@example.com", "phone": null}

시나리오 2: email = null (데이터 없음), phone = null
  {"id": "1", "name": "Alice", "email": null, "phone": null}

시나리오 3: Jackson @JsonInclude(NON_NULL)
  email = "alice@example.com", phone = null
  {"id": "1", "name": "Alice", "email": "alice@example.com"}
  // phone 필드가 null이므로 생략됨

문제: 시나리오 2와 3의 차이를 GraphQL에서 구분 불가능
  - 시나리오 2: phone이 null (명시적으로 "없음")
  - 시나리오 3: phone이 missing (필드가 응답에 없음)
  
  클라이언트는 두 상태를 동일하게 처리하게 됨

해결책: Optional 사용 (완벽하진 않지만)
  Java:
    String email;  // null이면 "이메일 없음"
    String phone;  // null이면 "전화 없음"
    
    public Optional<String> getEmail() {
        return Optional.ofNullable(email);
    }
  
  Jackson + jackson-datatype-jdk8:
    Optional.of(값) → "값"
    Optional.empty() → null
  
  GraphQL 응답:
    email = Optional.of("alice@example.com") → "alice@example.com"
    phone = Optional.empty() → null
    
  여전히 JSON 수준에서는 구분 불가능하지만,
  Java 코드에서는 Optional의 의미가 명확함
```

---

## 📊 성능/비교

```
직렬화 방식 성능:

1. Optional 필드 직렬화 (기본 Jackson)
   {"id":1,"email":{"value":"test@example.com","present":true}}
   
   크기: 더 크고 복잡함 (value, present 필드)
   속도: 느림 (내부 필드 접근)
   호환성: 낮음 (non-standard JSON)

2. Jdk8Module 사용
   {"id":1,"email":"test@example.com"}
   
   크기: 작음 (값만 포함)
   속도: 빠름 (직접 값 직렬화)
   호환성: 높음 (표준 JSON)

3. NON_ABSENT + Jdk8Module
   {"id":1,"email":"test@example.com"}  // null은 생략
   
   크기: 가장 작음 (null 필드 제외)
   속도: 빠름 (null 필드 스킵)
   호환성: 중간 (일부 필드 누락)

메모리:
  Optional 필드 (필드에 사용):
    엔티티 객체당 16 바이트 × null 가능 필드 수
    1,000 객체 × 3개 Optional = 48 KB 낭비
  
  Optional 메서드 반환:
    호출 시에만 Optional 생성 (임시)
    메모리 부담 최소

캐시 성능:
  Optional 필드 직렬화: CPU 캐시 미스 증가
  값만 직렬화: CPU 캐시 효율 향상

데이터베이스:
  Optional 필드: 매핑 오버헤드 (Converter 필요)
  일반 필드: 직접 매핑 (빠름)
```

---

## ⚖️ 트레이드오프

```
Optional 필드 사용:
  장점: "없을 수 있음"을 명시적으로 표현
  단점: 직렬화 불가, JPA 복잡도, 메모리 오버헤드

Optional 메서드 반환 + 일반 필드:
  장점: 직렬화 가능, JPA 호환, 의도 명확
  단점: API 사용자가 Optional 처리해야 함

Jackson jackson-datatype-jdk8:
  장점: JSON과 Optional 매핑 간소화
  단점: 추가 의존성, null과 empty의 구분 모호

JPA Converter:
  장점: Optional 필드 매핑 가능
  단점: 복잡도 증가, 권장되지 않음

GraphQL과 Optional:
  장점: null vs missing 표현 시도 가능
  단점: 완벽한 구분 불가능, 추가 설계 필요
```

---

## 📌 핵심 정리

```
Optional 직렬화/ORM 통합:

직렬화:
  - Optional은 Serializable 미구현
  - Jackson jackson-datatype-jdk8 필수
  - NON_ABSENT 설정으로 null 필드 생략 가능

JPA:
  - 엔티티 필드에 Optional 사용 금지
  - @Transient + 메서드 반환 패턴 사용
  - Converter로 Optional 매핑 (비권장)

Hibernate:
  - Optional 자동 지원 제한적
  - 일반 필드 + @Transient 메서드가 표준

REST API:
  - Optional을 반환하지 말 것
  - DTO나 ResponseEntity 사용
  - jackson-datatype-jdk8로 Optional 처리

캐시 (Redis):
  - Optional을 직접 캐시하지 말 것
  - 값만 추출해 캐시
  - Optional로 래핑해서 반환

GraphQL:
  - Optional은 null/missing 구분 못 함
  - 명시적 필드 추가 또는 Union 타입 고려
  - 설계 단계에서 null 의미론 명확히

설계 원칙:
  "Optional은 메서드 반환에만, 필드와 직렬화는 값으로"
```

---

## 🤔 생각해볼 문제

**Q1.** Jackson의 `jackson-datatype-jdk8`을 사용할 때 `JsonInclude.Include.NON_ABSENT`의 의미는?

<details>
<summary>해설 보기</summary>

NON_ABSENT의 정의:
  "값이 "빠진" 상태를 필드에서 제외하라"
  
  Optional의 경우:
  - Optional.empty() = "absent" (빠진 상태)
  - Optional.of(값) = "present" (있는 상태)

동작:
  class User {
      Optional<String> email;
      Optional<String> phone;
  }
  
  user = User with email="test@example.com", phone=Optional.empty()
  
  json = mapper.writeValueAsString(user);
  // 결과: {"email": "test@example.com"}  // phone 필드 생략!

다른 설정과의 비교:
  
  설정                          | 결과 JSON
  ──────────────────────────────┼─────────────────────────────────
  INCLUDE_ALWAYS               | {"email":"test@example.com","phone":null}
  NON_NULL (null 제외)           | {"email":"test@example.com","phone":null} (Optional.empty=null)
  NON_ABSENT (absent 제외)      | {"email":"test@example.com"}
  NON_EMPTY (empty만 제외)       | {"email":"test@example.com"}

실무적 의미:
  1. API 응답을 간결하게 → NON_ABSENT 사용
  2. null을 명시적으로 표현 → NON_NULL 또는 INCLUDE_ALWAYS

주의:
  NON_ABSENT는 Optional에만 적용되지 않음:
  - String name = null → 포함/제외 여부 불명확
  - Collection<String> items = [] (빈 컬렉션) → ABSENT? NO (컬렉션은 값)
  - Map<String, String> map = {} (빈 맵) → ABSENT? NO

결론: NON_ABSENT는 Optional.empty()와 null 값을 모두 제외. 필드를 동적으로 생략하고 싶을 때 사용.

</details>

---

**Q2.** JPA 엔티티가 Optional 필드를 가질 수 없는 근본 이유는?

<details>
<summary>해설 보기</summary>

JPA(Java Persistence API)의 근본적 제약:

1. 데이터베이스 매핑 불가능
   Optional은 Java의 추상화 개념일 뿐, SQL 타입이 아님
   
   SQL 테이블:
     CREATE TABLE users (
       id BIGINT PRIMARY KEY,
       name VARCHAR(255),
       email VARCHAR(255)
     );
   
   email은 VARCHAR(255) 또는 null을 저장 가능
   하지만 Optional<String>은 SQL 레벨에서 표현할 수 없음
   
   Hibernate (JPA 구현체)는:
   - Optional의 value 필드가 private → 접근 불가
   - 타입 파라미터 Optional<T>의 T를 추출해야 함
   - 역직렬화 시 Optional을 어떻게 복원할지 불명확

2. 직렬화 계약 위반
   JPA는 엔티티가 Serializable을 구현하기를 기대
   Optional이 Serializable 미구현 → 계약 위반
   
   순차:
   - 엔티티 저장: 데이터베이스에만 저장 (Optional 필드는 Serializable 아니므로 문제 안 됨)
   - 엔티티 캐시: 메모리 캐시/디스트리뷰티드 캐시에 저장
     → NotSerializableException 발생

3. 지연 로딩(Lazy Loading) 구현 불가능
   @LazyCollection으로 프록시 객체 생성 필요
   Optional은 final class → 프록시 불가능
   
   Hibernate는 CGLIB/ByteBuddy로 서브클래싱해서 프록시 생성
   Optional은 final이므로 서브클래싱 불가

4. 영속성 관리 로직 복잡화
   JPA의 Dirty Checking (수정 감지):
   
   user.email = "new@example.com";  // String 필드 → 감지 용이
   
   user.getEmail().ifPresentOrElse(
       e -> user.setEmail("new@example.com"),
       () -> user.setEmail("new@example.com")
   );  // Optional 필드 → 감지 어려움
   
   Optional 필드의 상태 변화를 감지하려면
   Optional 자체가 Serializable이고 비교 가능해야 함

올바른 패턴:
  @Entity
  class User {
      String email;  // nullable String 필드 (SQL과 일치)
      
      @Transient  // JPA 매핑 제외
      public Optional<String> getEmail() {
          return Optional.ofNullable(email);  // Java 레벨에서만 Optional
      }
  }

이렇게 하면:
  - DB: email VARCHAR(255) NULL
  - 엔티티 필드: String email (nullable)
  - Java API: getEmail() → Optional<String>

결론: JPA는 영속화를 위한 도메인 모델. Optional은 Java 레벨 안전성 도구. 두 개념을 혼합하면 충돌 발생. 필드는 영속화 가능한 타입, API는 Optional로 제공하는 것이 설계 원칙.

</details>

---

**Q3.** GraphQL에서 `null`과 `missing` 필드를 Optional로 구분할 수 있는가?

<details>
<summary>해설 보기</summary>

완벽하게 구분할 수 없다. 이유:

JSON 제약:
  JSON은 필드의 값과 필드의 부재만 구분 가능
  
  {"email": null}       // 필드 있음, 값은 null
  {"email": undefined}  // JavaScript에서만 가능, JSON에서는 불가능
  {}                    // 필드 없음 (missing)

Java Optional의 한계:
  Optional은 두 가지 상태만 표현:
  - Optional.of(값)      // "있음"
  - Optional.empty()     // "없음" (→ null로 직렬화)
  
  JSON으로 표현하면:
  - Optional.of("email") → "email": "..."
  - Optional.empty()     → "email": null 또는 필드 생략
  
  "명시적 null"과 "값 부재"의 구분 불가능

불완전한 해결책들:

방법 1: Optional + @JsonInclude
  @JsonInclude(Include.NON_ABSENT)
  class User {
      Optional<String> email;
  }
  
  문제: 여전히 JSON 수준에서는 null과 missing의 구분 안 됨

방법 2: 명시적 플래그 추가 (추천)
  type User {
    id: ID!
    email: String
    hasEmail: Boolean!  // "email이 설정되었는가?"를 명시적으로
  }
  
  응답:
    {"id": "1", "email": "test@example.com", "hasEmail": true}
    {"id": "2", "email": null, "hasEmail": false}

방법 3: Union 타입 (GraphQL 권장)
  union EmailResult = EmailValue | NoEmail
  
  type EmailValue {
    value: String!
  }
  
  type NoEmail {
    reason: String
  }
  
  type User {
    id: ID!
    email: EmailResult!  // null 불가능
  }
  
  쿼리:
    {
      user(id: "1") {
        id
        email {
          ... on EmailValue { value }
          ... on NoEmail { reason }
        }
      }
    }
  
  응답:
    {"id": "1", "email": {"value": "test@example.com"}}
    {"id": "2", "email": {"reason": "User has no email"}}

방법 4: separate nullable 필드
  type User {
    id: ID!
    name: String!
    email: String          # nullable, 값이 있으면 그것, null이면 "없음"
    emailProvidedAt: DateTime  # null이면 이메일 설정 안 함
  }

결론: 
  Optional만으로는 불가능. GraphQL 스키마 설계 단계에서
  null의 의미를 명확히 정의하고, 필요하면 Union 타입이나
  명시적 플래그를 추가해야 한다.
  
  Optional은 Java 도메인의 안전성 도구일 뿐,
  API 설계의 모호성은 해결하지 못한다.

</details>

---

<div align="center">

**[⬅️ 이전: Optional 안티패턴](./04-optional-antipatterns.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: CompletableFuture 구조 ➡️](../chapter05-completable-future/01-completable-future-structure.md)**

</div>
