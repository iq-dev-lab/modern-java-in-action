# Optional 안티패턴 — 필드·매개변수·Collection 금지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Optional을 필드로 사용하면 왜 안 되는가? (직렬화, 메모리, 의미론)
- 메서드 매개변수로 Optional을 받으면 호출자의 부담이 왜 증가하는가?
- `Optional<List<T>>`가 `List<T>`와 의미가 중복인 이유는?
- JDK API에서 Optional이 메서드 반환 타입에만 사용되는 설계 일관성은?
- Optional 안티패턴을 정적 분석으로 감지할 수 있는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Optional의 설계자 Brian Goetz는 Optional을 "메서드 반환 타입으로만 사용"하도록 명시했다. 이 원칙을 무시하면 직렬화 불가, ORM 매핑 오류, API 혼란 등 실무 문제가 대량 발생한다. 특히 엔티티 필드, REST API 매개변수, 컬렉션 원소로 사용할 때 문제가 심각하다. 안티패턴을 인식하는 것이 실무 코드 품질의 첫걸음이다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Optional을 엔티티 필드로 사용
  @Entity
  class User {
      @Id
      long id;
      Optional<String> email;  // ❌ 직렬화 불가!
      Optional<String> phone;
  }
  
  문제:
    - JPA 직렬화 불가: NotSerializableException
    - Hibernate는 Optional 컬럼 타입 지원 안 함
    - JSON 직렬화 시 Optional 래핑 제거 필요
    - DB 스키마와 불일치 (Optional은 no concept in SQL)

실수 2: Optional을 메서드 매개변수로 받기
  public void updateEmail(Long userId, Optional<String> email) {
      // email이 null이면? Optional.empty()?
      // 호출자 입장: email.orElse(null)로 감싸야 함
  }
  
  호출:
  service.updateEmail(1L, Optional.of("test@example.com"));  // 불편
  service.updateEmail(1L, Optional.empty());  // 수정하지 않겠다는 뜻?
  
  문제:
    - 호출 쪽에서 Optional을 만들어야 함
    - email이 null 가능한지 명확하지 않음
    - API 사용자 혼란 (Optional이 필요한가?)

실수 3: Optional<List<T>> 사용
  public Optional<List<String>> getNames() {
      List<String> names = repository.findAll();
      return names.isEmpty() 
          ? Optional.empty() 
          : Optional.of(names);
  }
  
  문제:
    - Optional.empty() vs 빈 리스트의 의미 중복
    - 호출자: result.orElse(Collections.emptyList())는 불필요
    - 리스트는 이미 "없음"을 표현 (빈 리스트)

실수 4: Collection<Optional<T>> 사용
  List<Optional<String>> emails = users.stream()
      .map(user -> Optional.ofNullable(user.getEmail()))
      .collect(Collectors.toList());
  
  // 순회:
  for (Optional<String> email : emails) {
      email.ifPresent(System.out::println);
  }
  
  문제:
    - Optional 래핑의 추가 레이어
    - 메모리 오버헤드 (Optional 객체 수백만 개)
    - Optional이 아니라 Stream 사용이 정답

실수 5: Optional로 "없음" 상태를 여러 의미로 사용
  Optional<String> result = cache.get(key);
  // result == Optional.empty()
  // 의미: 캐시에 없다? 또는 값이 null이었다?
  
  // 구분 불가:
  if (result.isEmpty()) {
      // 캐시 재로드인가? 값 자체가 없는 건가?
  }
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// Optional 안티패턴 해결 패턴

// 패턴 1: Optional 필드 → 메서드 반환
@Entity
class User {
    @Id
    long id;
    
    @Column(nullable = true)
    String email;  // 필드는 일반 타입
    
    @Transient  // JPA에서 처리 제외
    public Optional<String> getEmail() {
        return Optional.ofNullable(email);
    }
    
    public void setEmail(String email) {
        this.email = email;  // null 허용
    }
}

// 패턴 2: Optional 매개변수 → 오버로드 또는 null 매개변수
// ❌ 나쁜 설계
// public void updateEmail(Long userId, Optional<String> email) { }

// ✅ 좋은 설계 1: 오버로드
public void updateEmail(Long userId, String email) {
    // email이 null이면 버그 (호출자 책임)
}

public void updateEmail(Long userId) {
    // 이메일 수정 안 함
}

// ✅ 좋은 설계 2: @Nullable 어노테이션 (명시적)
public void updateEmail(Long userId, @Nullable String email) {
    if (email != null) {
        // 이메일 수정
    }
}

// 패턴 3: Optional<List<T>> → List<T> (빈 리스트 사용)
// ❌ 나쁜 설계
// public Optional<List<String>> getNames() { }

// ✅ 좋은 설계
public List<String> getNames() {
    return repository.findAllNames();  // 빈 리스트면 empty List
}

// 패턴 4: Collection<Optional<T>> → Stream 사용
// ❌ 나쁜 설계
// List<Optional<String>> emails = ...
// for (Optional<String> email : emails) { ... }

// ✅ 좋은 설계 1: null 필터링
List<String> emails = users.stream()
    .map(User::getEmail)
    .filter(Objects::nonNull)  // null 제거
    .collect(Collectors.toList());

// ✅ 좋은 설계 2: Optional을 Stream으로 변환
List<String> emails = users.stream()
    .map(user -> Optional.ofNullable(user.getEmail()))
    .flatMap(Optional::stream)  // Optional → Stream (0 또는 1개)
    .collect(Collectors.toList());

// 패턴 5: 캐시의 "없음" 상태 명확화
class Cache<K, V> {
    // ❌ 혼동: Optional.empty()가 뜻하는 게 뭔가?
    // Optional<V> get(K key) { }
    
    // ✅ 명확: 존재 여부를 명시적으로
    Optional<V> getIfPresent(K key) {
        return Optional.ofNullable(data.get(key));  // 캐시에 없으면 empty
    }
}

// 패턴 6: REST API - JSON 직렬화
@RestController
class UserController {
    
    // ❌ Optional을 매개변수로 받음
    // public User getUser(@RequestParam Optional<Long> id) { }
    
    // ✅ 기본값이나 조건부 처리
    public User getUser(@RequestParam(required = false) Long id) {
        if (id == null) {
            return getCurrentUser();
        }
        return userService.findById(id).orElseThrow(UserNotFoundException::new);
    }
}

// 패턴 7: 빈 Optional 감지를 위한 전용 메서드
class OptionalUtils {
    static <T> Optional<T> ofNonEmpty(Collection<T> collection) {
        return collection.isEmpty() 
            ? Optional.empty() 
            : Optional.of(collection.stream().findFirst().orElse(null));
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Optional 필드의 직렬화 문제

```java
// Optional은 Serializable을 구현하지 않음
public final class Optional<T> {
    // implements Serializable 없음!
    private final T value;
}

필드로 사용 시:
  class User implements Serializable {
      Optional<String> email;  // Optional이 Serializable 아님
  }
  
  ObjectOutputStream oos = new ObjectOutputStream(file);
  oos.writeObject(user);  // NotSerializableException!

JPA 직렬화:
  Hibernate는 필드를 직렬화하려고 시도:
  - Optional 타입 인식 실패
  - 또는 value 필드만 직렬화하는 커스텀 처리 필요
  - 데이터베이스 매핑 불명확

메모리 구조:
  class User {
      String email;  // 8 바이트 참조
      // 추가 메모리: 0
  }
  
  class User {
      Optional<String> email;  // 8 바이트 참조
      // 추가 메모리: Optional 객체 16 바이트
  }
  
  10,000 User 객체: 160 KB 추가 메모리

직렬화 형식:
  // Optional 필드 없는 User
  {
    "id": 1,
    "email": "test@example.com"  // 직접 값
  }
  
  // Optional 필드 있는 User (Jackson + jackson-datatype-jdk8)
  {
    "id": 1,
    "email": "test@example.com"  // Optional 래핑 제거됨 (모듈이 처리)
  }
  
  vs Optional이 null:
  {
    "id": 1,
    "email": null  // null로 처리
  }
  
  문제: null과 "값 없음"의 의미 차이를 JSON이 구분 못함
```

### 2. Optional 매개변수의 호출자 부담

```java
// Optional 매개변수의 문제

// ❌ Optional 매개변수 API
public void updateEmail(Long userId, Optional<String> email) {
    if (email.isPresent()) {
        // 이메일 수정
        db.update(userId, email.get());
    } else {
        // 이메일 수정 안 함
    }
}

호출 시:
  // 호출자가 Optional을 만들어야 함
  service.updateEmail(1L, Optional.of("new@example.com"));
  service.updateEmail(1L, Optional.empty());  // 수정하지 않음
  
  // 문제: 호출자가 Optional의 의미를 알아야 함
  // API 문서가 명확하지 않으면 혼란

// ✅ 올바른 API
public void updateEmail(Long userId, String email) {
    // email이 null이면 프로그래머 실수 (InputValidationException)
    db.update(userId, Objects.requireNonNull(email));
}

public void clearEmail(Long userId) {
    // 의도 명확: 이메일 제거
    db.update(userId, null);  // 또는 DELETE 쿼리
}

호출 시:
  service.updateEmail(1L, "new@example.com");  // 명확
  service.clearEmail(1L);  // 명확

API 설계 원칙:
  매개변수에 Optional 사용 안 함
  → 대신 오버로드, 빌더 패턴, @Nullable 어노테이션 사용
  → 호출자 부담 감소, API 명확성 향상
```

### 3. Optional<Collection<T>>의 중복 의미

```java
// Optional<List<T>>의 문제

// ❌ 나쁜 설계
public Optional<List<String>> getNames() {
    List<String> result = database.query("SELECT name FROM users");
    return result.isEmpty() 
        ? Optional.empty() 
        : Optional.of(result);
}

호출 시:
  Optional<List<String>> opt = getNames();
  
  opt.isPresent()의 의미:
    - 리스트가 비어있지 않다? (isPresent() = true)
    - 또는 쿼리 성공?
  
  opt.get()의 위험:
    - NoSuchElementException 가능 (empty())
    - 하지만 쿼리가 성공했으면 항상 빈 List 이상을 반환해야 함
  
  opt.orElse(Collections.emptyList())
    - 불필요한 orElse (List는 이미 empty 처리 가능)

의미 중복:
  Optional.empty() = 리스트 없음 (쿼리 실패? 데이터 없음?)
  empty List = 데이터 없음
  
  둘의 구분이 불명확

데이터 구조적 분석:
  List는 이미 0개 원소를 지원
  → Optional<List<T>>는 불필요 (중복)
  → null vs empty List도 의미 중복

// ✅ 올바른 설계
public List<String> getNames() {
    return database.query("SELECT name FROM users");
    // 빈 리스트면 empty List 반환
    // "데이터 없음" = empty List
}

호출 시:
  List<String> names = getNames();
  
  names.isEmpty()
    - 명확: 데이터 없음
  
  names.forEach(System.out::println);
    - 루프가 0회 실행 (안전)
  
  names.stream()
      .filter(s -> s.length() > 3)
      .collect(Collectors.toList());
    - Stream API와 자연스럽게 결합

원칙:
  Collection이 이미 "없음"을 표현하면 Optional 불필요
  → List, Set, Map: empty 컬렉션으로 표현
  → Map: containsKey() 체크, get() null 반환
  → Optional: 단일 값의 부재만
```

### 4. Collection<Optional<T>>의 메모리/성능 문제

```java
// Collection<Optional<T>>의 문제

// ❌ 나쁜 구조
List<Optional<String>> emails = users.stream()
    .map(user -> Optional.ofNullable(user.getEmail()))
    .collect(Collectors.toList());

메모리 구조:
  List (배열)
    [0] → Optional 객체 (16 바이트)
          value → "alice@example.com"
    [1] → Optional 객체 (16 바이트)
          value → null
    [2] → Optional 객체 (16 바이트)
          value → "charlie@example.com"
    ...

1,000,000 사용자 (30% null):
  Optional 래핑: 1M × 16 바이트= 16 MB
  실제 효율: 70만 개 Optional만 데이터 포함, 30만 개는 null
  추가 메모리: 30만 × 16 바이트 = 4.8 MB (낭비)

읽기 성능:
  for (Optional<String> email : emails) {
      email.ifPresent(e -> process(e));
  }
  
  매 반복마다 Optional 객체 접근 → CPU 캐시 미스 증가

// ✅ 올바른 구조 1: null 필터링
List<String> emails = users.stream()
    .map(User::getEmail)
    .filter(Objects::nonNull)
    .collect(Collectors.toList());

메모리: null 제거, String만 저장 (메모리 절약)
성능: 직접 String 접근 (캐시 효율 향상)

// ✅ 올바른 구조 2: Optional → Stream 변환
List<String> emails = users.stream()
    .flatMap(user -> Optional.ofNullable(user.getEmail()).stream())
    .collect(Collectors.toList());

동작:
  각 user에 대해:
  - Optional.ofNullable(email) 생성
  - .stream() → 0 또는 1개 원소 Stream
  - flatMap으로 평탄화 → 전체 Stream에 추가
  - 결과: null이 자동으로 필터링됨

메모리: List에는 String만 저장 (Optional 래핑 없음)
성능: Stream 연쇄 처리 (JIT 최적화 가능)
```

### 5. JDK API 설계의 일관성

```java
// JDK 표준 라이브러리에서 Optional 사용 패턴

1. 메서드 반환 타입 (✅ 권장)

public interface Optional<T> {
    public static <T> Optional<T> of(T value) { ... }
    public static <T> Optional<T> ofNullable(T value) { ... }
    public static <T> Optional<T> empty() { ... }
    
    public Optional<U> map(Function<T, U> mapper) { ... }
    public Optional<U> flatMap(Function<T, Optional<U>> mapper) { ... }
}

public interface Stream<T> {
    public <R> Stream<R> map(Function<T, R> mapper) { ... }
    // Stream 메서드 반환도 Optional 지원
    public Optional<T> findFirst() { ... }
    public Optional<T> findAny() { ... }
    public Optional<T> max(Comparator<T> cmp) { ... }
}

2. 매개변수로 사용 안 함 (❌ 금지)

Java 표준 라이브러리에서:
  - Collections.empty*(): 매개변수 Optional 없음
  - Stream 메서드: 매개변수 Optional 없음
  - ExecutorService: 매개변수 Optional 없음

3. 필드로 사용 안 함 (❌ 금지)

Exception 클래스들:
  class IOException extends Exception {
      // cause 필드는 Throwable (Optional 아님)
      private Throwable cause;
      
      public Optional<Throwable> getCause() {
          return Optional.ofNullable(cause);
      }
  }

엔티티/DAO:
  JDK의 Record 타입도 Optional 필드 사용 안 함
  record User(long id, String name, String email) { }
  // email은 null 가능해도 Optional로 선언 안 함

일관성의 의미:
  JDK 설계자들이 Optional을 메서드 반환에만 사용
  → 업계 표준 (de facto standard)
  → 우리도 따라야 함
```

---

## 💻 실전 실험

### 실험 1: Optional 필드 직렬화 실패

```java
import java.io.*;
import java.util.Optional;

public class OptionalFieldSerializationTest {
    
    static class BadUser implements Serializable {
        long id;
        Optional<String> email;  // Serializable 아님
        
        BadUser(long id, String email) {
            this.id = id;
            this.email = Optional.ofNullable(email);
        }
    }
    
    static class GoodUser implements Serializable {
        long id;
        String email;  // 일반 필드
        
        GoodUser(long id, String email) {
            this.id = id;
            this.email = email;
        }
        
        Optional<String> getEmail() {
            return Optional.ofNullable(email);
        }
    }
    
    public static void main(String[] args) throws Exception {
        System.out.println("=== BadUser 직렬화 시도 ===");
        try {
            BadUser bad = new BadUser(1, "test@example.com");
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            ObjectOutputStream oos = new ObjectOutputStream(baos);
            oos.writeObject(bad);
            System.out.println("성공!");
        } catch (NotSerializableException e) {
            System.out.println("실패: " + e.getMessage());  // java.util.Optional
        }
        
        System.out.println("\n=== GoodUser 직렬화 ===");
        try {
            GoodUser good = new GoodUser(1, "test@example.com");
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            ObjectOutputStream oos = new ObjectOutputStream(baos);
            oos.writeObject(good);
            System.out.println("성공!");
            
            ByteArrayInputStream bais = new ByteArrayInputStream(baos.toByteArray());
            ObjectInputStream ois = new ObjectInputStream(bais);
            GoodUser deserialized = (GoodUser) ois.readObject();
            System.out.println("역직렬화 후 email: " + deserialized.getEmail());
        } catch (Exception e) {
            System.out.println("실패: " + e.getMessage());
        }
    }
}

// 출력:
// === BadUser 직렬화 시도 ===
// 실패: java.util.Optional
//
// === GoodUser 직렬화 ===
// 성공!
// 역직렬화 후 email: Optional[test@example.com]
```

### 실험 2: Optional 매개변수 vs 오버로드

```java
import java.util.Optional;

public class OptionalParameterTest {
    
    // ❌ Optional 매개변수
    static class ServiceWithOptional {
        void updateUser(long id, Optional<String> name) {
            if (name.isPresent()) {
                System.out.println("이름 업데이트: " + name.get());
            } else {
                System.out.println("이름 업데이트 안 함");
            }
        }
    }
    
    // ✅ 오버로드 사용
    static class ServiceWithOverload {
        void updateUser(long id, String name) {
            if (name == null) {
                throw new IllegalArgumentException("name is required");
            }
            System.out.println("이름 업데이트: " + name);
        }
        
        void updateUserOptionally(long id) {
            System.out.println("이름 업데이트 안 함");
        }
    }
    
    public static void main(String[] args) {
        System.out.println("=== Optional 매개변수 사용 (혼동) ===");
        ServiceWithOptional service1 = new ServiceWithOptional();
        
        service1.updateUser(1, Optional.of("Alice"));  // OK
        service1.updateUser(1, Optional.empty());      // OK, 하지만 의도 불명확
        service1.updateUser(1, null);                  // 컴파일 에러 또는 런타임 NPE
        
        System.out.println("\n=== 오버로드 사용 (명확) ===");
        ServiceWithOverload service2 = new ServiceWithOverload();
        
        service2.updateUser(1, "Alice");       // 명확: 이름 업데이트
        service2.updateUserOptionally(1);      // 명확: 업데이트 안 함
        
        // service2.updateUser(1, null);  // 컴파일 오류 가능성 낮음
    }
}

// 출력:
// === Optional 매개변수 사용 (혼동) ===
// 이름 업데이트: Alice
// 이름 업데이트 안 함
//
// === 오버로드 사용 (명확) ===
// 이름 업데이트: Alice
// 이름 업데이트 안 함
```

### 실험 3: Collection<Optional<T>> vs 올바른 방법

```java
import java.util.*;
import java.util.stream.Collectors;

public class CollectionOptionalTest {
    
    static class User {
        String name;
        String email;
        User(String name, String email) {
            this.name = name;
            this.email = email;
        }
    }
    
    public static void main(String[] args) {
        List<User> users = List.of(
            new User("Alice", "alice@example.com"),
            new User("Bob", null),
            new User("Charlie", "charlie@example.com")
        );
        
        System.out.println("=== ❌ Collection<Optional<T>> (나쁜 방법) ===");
        List<Optional<String>> badEmails = users.stream()
            .map(u -> Optional.ofNullable(u.email))
            .collect(Collectors.toList());
        
        System.out.println("크기: " + badEmails.size());
        badEmails.forEach(e -> e.ifPresent(System.out::println));
        
        System.out.println("\n=== ✅ filter + null 제거 (좋은 방법 1) ===");
        List<String> goodEmails1 = users.stream()
            .map(u -> u.email)
            .filter(Objects::nonNull)
            .collect(Collectors.toList());
        
        System.out.println("크기: " + goodEmails1.size());
        goodEmails1.forEach(System.out::println);
        
        System.out.println("\n=== ✅ flatMap(Optional::stream) (좋은 방법 2) ===");
        List<String> goodEmails2 = users.stream()
            .flatMap(u -> Optional.ofNullable(u.email).stream())
            .collect(Collectors.toList());
        
        System.out.println("크기: " + goodEmails2.size());
        goodEmails2.forEach(System.out::println);
        
        System.out.println("\n=== 메모리 비교 ===");
        System.out.println("Collection<Optional<T>>: 3개 Optional 객체 생성");
        System.out.println("filter(nonNull): Optional 없음, null 체크만");
        System.out.println("flatMap(stream): Optional 중간 생성, 최종 결과에 없음");
    }
}

// 출력:
// === ❌ Collection<Optional<T>> (나쁜 방법) ===
// 크기: 3
// alice@example.com
// charlie@example.com
//
// === ✅ filter + null 제거 (좋은 방법 1) ===
// 크기: 2
// alice@example.com
// charlie@example.com
//
// === ✅ flatMap(Optional::stream) (좋은 방법 2) ===
// 크기: 2
// alice@example.com
// charlie@example.com
//
// === 메모리 비교 ===
// Collection<Optional<T>>: 3개 Optional 객체 생성
// filter(nonNull): Optional 없음, null 체크만
// flatMap(stream): Optional 중간 생성, 최종 결과에 없음
```

---

## 📊 성능/비교

```
Optional 안티패턴의 성능 영향:

필드로 Optional 사용:
  메모리: 객체당 16 바이트 추가
  1,000,000 사용자: 16 MB 낭비
  캐시: Optional 필드 접근 → 추가 인더렉션

매개변수로 Optional 사용:
  호출 비용: Optional 객체 생성 + 언래핑
  API 복잡도: 호출자가 Optional 관리
  가독성: 낮음 (의도 불명확)

Collection<Optional<T>>:
  메모리: 원소 × 16 바이트 추가 메모리
  1,000,000 원소: 16 MB 낭비
  순회: Optional 객체 접근 → 캐시 미스

올바른 설계와 비교:

설계          | 메모리     | 캐시 효율  | 가독성  | API 명확성
──────────────┼────────────┼───────────┼───────┼──────────
Optional 필드 | +16 MB    | 나쁨      | 낮음  | 낮음
좋은 설계     | 0        | 좋음      | 높음  | 높음

Collection<Optional<T>>:
  병렬 처리: Optional 접근 → 동기화 오버헤드
  필터링: filter(nonNull)이 flatMap(Optional::stream)보다 빠름
  메모리 캐시: 가능한 한 작은 객체 구조 (Optional 제외)
```

---

## ⚖️ 트레이드오프

```
Optional 필드:
  장점: 명시적으로 "없을 수 있음" 표현
  단점: 직렬화 불가, JPA 미지원, 메모리 오버헤드

Optional 매개변수:
  장점: 명시적으로 "선택적" 매개변수 표현
  단점: 호출자 부담 증가, API 복잡도

Optional<Collection<T>>:
  장점: "빈 컬렉션" vs "없음" 구분 가능?
  단점: 의미 중복, 메모리 낭비, 사용 복잡도

Collection<Optional<T>>:
  장점: 각 원소의 존재 여부 명시적 표현?
  단점: 메모리 낭비, 캐시 효율 저하, 코드 복잡도

설계 원칙:
  메서드 반환: Optional 권장 (null 가능 명시)
  필드: null 또는 기본값 (Optional 금지)
  매개변수: null 불가 또는 오버로드 (Optional 금지)
  컬렉션: 빈 컬렉션 (Optional 금지)
```

---

## 📌 핵심 정리

```
Optional 안티패턴 정리:

❌ 금지된 사용:
  1. 필드: @Transient + 메서드 반환으로 변경
  2. 매개변수: 오버로드 또는 @Nullable 사용
  3. Optional<List<T>>: List<T> (빈 리스트 사용)
  4. Optional<Map<T>>: Map<T> (containsKey 사용)
  5. Collection<Optional<T>>: filter(nonNull) 또는 flatMap(stream)

✅ 권장 사용:
  1. 메서드 반환: Optional<T> 사용
  2. 메서드 체인: map/flatMap 사용
  3. 조건부 처리: ifPresent/ifPresentOrElse 사용

설계 가이드:
  "Optional은 메서드 반환에만 사용한다"
  이것이 Brian Goetz의 원래 설계 의도

정적 분석:
  SpotBugs, Checker Framework, IntelliJ IDEA
  Optional 필드/매개변수 사용 감지 가능
```

---

## 🤔 생각해볼 문제

**Q1.** `Optional<List<T>>`와 `List<T>`가 의미적으로 왜 중복인가?

<details>
<summary>해설 보기</summary>

List는 이미 "없음"을 표현할 수 있다:
- 빈 리스트 (size == 0) = 데이터 없음
- null (참조 없음) = 쿼리 실패 또는 오류

Optional<List<T>>의 경우:
- Optional.empty() = 데이터 없음 또는 쿼리 실패?
- Optional.of(emptyList()) = 데이터 없음 (빈 리스트)
- Optional.of(nonEmptyList()) = 데이터 있음

의미 모호성:
  Optional.empty()의 의미:
    1. 쿼리 실패
    2. 데이터 없음 (Optional.of(emptyList()) 대신)
    3. null 값

  구분 불가능 → API 사용자 혼동

List 단독 사용:
  emptyList() = 데이터 없음 (명확)
  nonEmptyList() = 데이터 있음 (명확)
  null이 필요하면? → 쿼리 실패 예외를 던짐

결론: "없음"을 표현하는 두 가지 방법(Optional, 빈 컬렉션)이 있으면 혼동이 생긴다. 어느 하나를 선택해야 함. List는 이미 충분하므로 Optional은 불필요.

</details>

---

**Q2.** Optional 필드 대신 메서드 반환을 사용해야 하는 근본 이유는?

<details>
<summary>해설 보기</summary>

세 가지 근본 이유:

1. 직렬화 호환성
   Optional은 Serializable을 구현하지 않음
   → JPA, JSON, RPC 직렬화 불가능
   → 필드이면 엔티티 직렬화 실패
   
   메서드 반환이면:
   - 필드는 일반 타입으로 유지
   - 직렬화 가능
   - getter에서 Optional로 래핑 (메모리 효율적, 호출 시만 생성)

2. 값 기반 클래스의 의미
   Optional은 @ValueBased (값 기반)
   → == 비교 정의되지 않음
   → 필드로 사용하면 equals/hashCode 혼동
   
   메서드 반환이면:
   - 임시 객체로만 존재
   - 비교/해싱 필요 없음

3. JPA와 데이터베이스 매핑
   Optional은 SQL 타입이 없음
   → Hibernate가 매핑 불가능
   → @Lob, @Convert 같은 커스텀 처리 필요
   
   메서드 반환이면:
   - 필드는 String, Long 같은 SQL 타입
   - JPA가 자동 매핑
   - getter에서 Optional로 변환

결론: Optional은 "일시적 컨테이너", 필드는 영구적 저장소. 필드는 영구성(직렬화, JPA)을 고려해야 하므로 Optional이 부적합.

</details>

---

**Q3.** `Optional` 안티패턴을 정적 분석으로 감지하는 방법은?

<details>
<summary>해설 보기</summary>

자동화된 감지 도구들:

1. SpotBugs (구 FindBugs)
   ```
   플러그인: SpotBugs for Eclipse, Maven
   패턴: Optional 필드 감지
   규칙: OPTIONAL_FIELD_VALIDATION
   
   감지 예:
   class User {
       Optional<String> email;  // ❌ 경고
   }
   ```

2. IntelliJ IDEA (기본 제공)
   ```
   Inspection: "Optional field"
   설정: Settings → Editor → Inspections → Java
   
   감지 및 제안:
   - Optional 필드 사용 경고
   - 자동 리팩토링: 메서드 반환으로 변경
   ```

3. Checker Framework
   ```
   플러그인: @Nullable, @Nonnull 검증
   설정: javac 플러그인으로 컴파일 타임 검증
   
   감지 예:
   class User {
       Optional<String> email;  // ❌ 타입 시스템 위반
   }
   ```

4. SonarQube
   ```
   규칙: "java:S4970" (Optional 필드 감지)
   규칙: "java:S4971" (Optional 매개변수 감지)
   
   리포트:
   - 위반 항목 목록
   - 심각도: MAJOR
   - 제안: 리팩토링 패턴
   ```

5. ErrorProne (Google)
   ```
   플러그인: Google's Error Prone
   검사: OptionalAssign, OptionalParameter
   
   빌드 타임 검증:
   mvn clean compile
   → Optional 안티패턴 감지
   ```

자동화 스택:
  1. IDE: IntelliJ IDEA 기본 검사
  2. 빌드: SonarQube or ErrorProne
  3. CI/CD: 빌드 실패로 merge 차단
  4. 코드 리뷰: 자동 리뷰 코멘트

결론: Optional 안티패턴은 충분히 자동화되어 있다. 팀의 빌드 파이프라인에 SpotBugs or SonarQube를 추가하면 대부분의 위반을 사전에 차단할 수 있다.

</details>

---

<div align="center">

**[⬅️ 이전: map vs flatMap](./03-map-vs-flatmap-monad.md)** | **[홈으로 🏠](../README.md)** | **[다음: 직렬화·ORM·Optional ➡️](./05-optional-serialization-orm.md)**

</div>
