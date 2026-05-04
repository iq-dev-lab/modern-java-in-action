# Optional.of vs ofNullable vs empty — 정확한 사용 시점

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Optional.of(null)`이 `NullPointerException`을 던지는 것은 오류인가 아니면 설계인가?
- `ofNullable()`이 안전한 래핑을 보장하는 메커니즘은?
- `Optional.empty()`가 기본 선택지가 아닌 이유는?
- "이 값이 null이면 버그"와 "null일 수 있음"의 의도를 코드로 정확히 표현하는 방법은?
- 코드 리뷰에서 `ofNullable()` 남용을 어떻게 지적할 것인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Optional의 세 가지 팩토리 메서드는 단순해 보이지만, 의도적으로 다른 시멘틱을 갖는다. `Optional.of(null)`이 NPE를 던지는 설계는 "이 값이 절대 null이 아니어야 한다"는 계약을 명시하는 것이고, `ofNullable()`은 "null일 수 있다"는 선언이다. 이를 구분하지 못하면 코드의 의도가 불명확해지고, 버그 발생 시 원인 파악이 어려워진다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 항상 ofNullable() 사용
  // 안전하다고 생각하지만 의도 불명확
  public Optional<String> getUserName(User user) {
      return Optional.ofNullable(user.getName());
  }
  
  문제: user.getName()이 절대 null이 아니어야 한다면?
        ofNullable()은 "null일 수 있다"는 거짓 신호
        코드 리뷰어는 이 필드의 계약을 파악할 수 없음

실수 2: of()로 외부 입력 받기
  public Optional<String> findUserEmail(String email) {
      return Optional.of(email);  // 외부 입력인데 NPE 던짐
  }
  
  호출:
  Optional<String> result = findUserEmail(userInput);  // userInput이 null?
  // → NullPointerException 가능성
  
  해결: Optional.ofNullable(email)

실수 3: of()로 감싼 후 다시 null 체크
  Optional<String> name = Optional.of(user.getName());  // NPE 위험
  if (name != null) { ... }  // Optional을 null 체크? 모순!
  
  Optional의 의도 무시

실시 4: ofNullable()의 오버헤드 무시
  // 루프에서 매번 ofNullable() 호출
  for (User user : users) {
      Optional.ofNullable(user.getEmail())  // null check 매번
          .ifPresent(System.out::println);
  }
  
  소수의 null인 경우 불필요한 Optional 생성 오버헤드
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 올바른 Optional 팩토리 메서드 사용 패턴

// 패턴 1: 필드가 절대 null이 아닐 때 → of()
class User {
    private final String id;      // 생성자에서 검증, null 불가
    private final String email;   // final, 항상 초기화됨
    
    public User(String id, String email) {
        this.id = Objects.requireNonNull(id);
        this.email = Objects.requireNonNull(email);
    }
    
    public Optional<String> getId() {
        return Optional.of(id);  // NPE 불가능, 의도 명확
    }
}

// 패턴 2: null일 수 있는 값 → ofNullable()
class UserRepository {
    public Optional<User> findById(String id) {
        // DB 쿼리가 결과 없을 수 있음
        User user = database.executeQuery("SELECT * FROM users WHERE id = ?");
        return Optional.ofNullable(user);  // null 가능, 의도 명확
    }
}

// 패턴 3: 조건부 wrapping
public Optional<String> getUserPhone(User user, boolean includePrivate) {
    String phone = includePrivate ? user.getFullPhone() : user.getPublicPhone();
    return Optional.ofNullable(phone);  // phone이 null일 수도, 아닐 수도
}

// 패턴 4: 체인에서 중간값 처리
Optional<User> user = findUserById(userId)
    .flatMap(u -> {
        String email = u.getEmail();
        // email이 항상 null이 아니면:
        return Optional.of(email);  // of() 사용, 명확함
    })
    .filter(e -> e.contains("@"))
    .map(e -> e.toLowerCase());

// 패턴 5: Stream에서의 올바른 사용
users.stream()
    .map(user -> {
        // user.getPhone()이 null일 수 있으면
        return Optional.ofNullable(user.getPhone());  // ofNullable()
    })
    .flatMap(Optional::stream)  // Optional → Stream 변환
    .forEach(System.out::println);

// 패턴 6: API 설계 가이드
public interface UserService {
    // 명확: 항상 User를 반환 (없으면 Optional.empty())
    Optional<User> findById(String id);
    
    // 명확: 리스트는 빈 리스트로 반환 (Optional 불필요)
    List<User> findByName(String name);
    
    // 명확: Email은 항상 존재하고, 공개 여부는 boolean
    String getEmail(User user);  // 항상 non-null 또는 exception
    
    // 명확: 공개 가능한 이메일이면 Optional로 반환
    Optional<String> getPublicEmail(User user);
}
```

---

## 🔬 내부 동작 원리

### 1. of() — Non-null 검증

```java
// OpenJDK Optional.of(T value)
public static <T> Optional<T> of(T value) {
    return new Optional<>(Objects.requireNonNull(value));
    //    ↑ NPE 발생 (null이면)
}

// Objects.requireNonNull의 구현
public static <T> T requireNonNull(T obj) {
    if (obj == null)
        throw new NullPointerException();
    return obj;
}

// 더 상세한 버전 (Java 9+)
public static <T> T requireNonNull(T obj, String message) {
    if (obj == null)
        throw new NullPointerException(message);
    return obj;
}

의도:
  Optional.of(value)는 "value가 절대 null이 아니다"는 프로그래머의 확신
  → null이면 프로그래머 실수 또는 입력 검증 실패
  → 즉시 NPE를 던져 버그를 조기에 포착

Examples:
  String name = "John";
  Optional<String> opt1 = Optional.of(name);  // OK, 'John'으로 wrapping
  
  String email = null;
  Optional<String> opt2 = Optional.of(email);  // NullPointerException!
```

### 2. ofNullable() — 안전한 래핑

```java
// OpenJDK Optional.ofNullable(T value)
public static <T> Optional<T> ofNullable(T value) {
    return value == null ? empty() : of(value);
    //    null이면 EMPTY  / 아니면 of()로 wrapping
}

동작:
  value == null → Optional.empty() 반환 (EMPTY 싱글톤)
  value != null → Optional.of(value) 호출 (new Optional(value))

메모리 구조:
  Optional<String> opt1 = Optional.ofNullable("test");
    → new Optional<>("test")
    
  Optional<String> opt2 = Optional.ofNullable(null);
    → EMPTY 반환 (싱글톤)
  
  opt2의 내부:
    value = null (EMPTY는 private final T value = null)

의도:
  "null일 수 있지만 처리하겠다"는 선언
  → null을 허용하고, 안전하게 처리
  → 검증 불필요 (이미 수행됨)

Examples:
  String input = getUserInput();  // null 가능
  Optional<String> opt = Optional.ofNullable(input);
  // input이 null → EMPTY 반환
  // input이 "value" → of(input) 반환
  
  opt.ifPresent(v -> System.out.println(v));
  // null이면 아무 것도 출력 안 함
```

### 3. empty() — 명시적 빈 Optional

```java
// OpenJDK Optional.empty()
public static<T> Optional<T> empty() {
    return (Optional<T>) EMPTY;
    //    정적 싱글톤 상수 반환
}

// EMPTY 상수
private static final Optional<?> EMPTY = new Optional<>(null);

사용:
  Optional<String> empty = Optional.empty();
  // EMPTY 싱글톤 반환
  
  empty.isEmpty();  // true (Java 11+)
  empty.isPresent();  // false

메모리:
  empty()를 몇 번 호출해도 같은 EMPTY 객체 반환
  (가비지 컬렉션 오버헤드 없음)

언제 사용:
  Optional<String> result;
  if (someCondition) {
      result = Optional.of(value);
  } else {
      result = Optional.empty();  // 명시적으로 "없음"
  }

  대안: Optional.ofNullable(someCondition ? value : null);
```

### 4. 세 메서드의 선택 플로우차트

```
값을 Optional로 wrapping하려고 한다.
  ↓
이 값이 절대 null이 아니어야 하는가? (프로그래머의 계약)
  ↓ YES
  값이 정말 null이 아니면 안 되는가? (외부 입력 검증 済)
    ↓ YES
    → Optional.of(value)  ✅
    (null → NullPointerException)
    
    ↓ NO
    값이 null일 수도 있는데 of()로 감싸야 하나?
      ↓ YES
      → 사전 검증: value = Objects.requireNonNull(value)
        후 Optional.of(value)  ✅
      
      ↓ NO
      → Optional.ofNullable(value)  ✅
  
  ↓ NO
  값이 null일 수 있다.
    ↓
    분명히 absent할 수 있는가?
      ↓ YES
      → Optional.ofNullable(value)  ✅
      
      ↓ NO
      → null인 경우 default로 처리?
        → Optional.of(value != null ? value : default)  또는
        → Optional.ofNullable(value).orElse(default)  ✅
```

---

## 💻 실전 실험

### 실험 1: of() vs ofNullable() 비교

```java
public class OfVsOfNullableTest {
    public static void main(String[] args) {
        // of() 사용: null이면 NPE
        System.out.println("=== Optional.of() ===");
        try {
            Optional<String> opt = Optional.of(null);
            System.out.println("성공: " + opt);
        } catch (NullPointerException e) {
            System.out.println("NullPointerException 발생 (기대 동작)");
        }
        
        // ofNullable() 사용: null이면 EMPTY
        System.out.println("\n=== Optional.ofNullable() ===");
        Optional<String> opt = Optional.ofNullable(null);
        System.out.println("결과: " + opt);  // Optional.empty
        System.out.println("isEmpty: " + opt.isEmpty());  // true
        
        // ofNullable()의 메모리 효율성
        System.out.println("\n=== EMPTY 싱글톤 검증 ===");
        Optional<String> empty1 = Optional.ofNullable(null);
        Optional<String> empty2 = Optional.ofNullable(null);
        System.out.println("empty1 == empty2: " + (empty1 == empty2));  // true!
        System.out.println("같은 객체: " + (System.identityHashCode(empty1) == 
                                            System.identityHashCode(empty2)));
    }
}
```

### 실험 2: 팩토리 메서드 선택의 영향

```java
public class FactoryMethodChoiceTest {
    
    static class User {
        String id;        // 생성자에서 검증됨
        String email;     // null 가능
        String phone;     // 항상 존재
        
        public User(String id, String email, String phone) {
            this.id = Objects.requireNonNull(id);
            this.email = email;  // null 가능
            this.phone = Objects.requireNonNull(phone);
        }
    }
    
    // ✅ 올바른 설계
    static class WellDesigned {
        User user;
        
        // id는 항상 존재 → of() 사용
        Optional<String> getId() {
            return Optional.of(user.id);  // NPE 불가능
        }
        
        // email은 null 가능 → ofNullable() 사용
        Optional<String> getEmail() {
            return Optional.ofNullable(user.email);  // null 안전
        }
        
        // phone은 항상 존재 → of() 사용
        Optional<String> getPhone() {
            return Optional.of(user.phone);  // NPE 불가능
        }
    }
    
    // ❌ 나쁜 설계 (모두 ofNullable 사용)
    static class PoorDesign {
        User user;
        
        Optional<String> getId() {
            return Optional.ofNullable(user.id);  // 거짓 신호: id가 null일 수 없는데
        }
        
        Optional<String> getEmail() {
            return Optional.ofNullable(user.email);  // 맞음
        }
        
        Optional<String> getPhone() {
            return Optional.ofNullable(user.phone);  // 거짓 신호: phone이 null일 수 없는데
        }
    }
    
    public static void main(String[] args) {
        User user = new User("123", "test@example.com", "010-1234-5678");
        
        System.out.println("=== WellDesigned ===");
        WellDesigned good = new WellDesigned() {{ this.user = user; }};
        System.out.println("id: " + good.getId());
        System.out.println("email: " + good.getEmail());
        System.out.println("phone: " + good.getPhone());
        
        // 코드 리뷰어가 볼 수 있는 것:
        // getId() → of() → "id는 절대 null이 아니다"
        // getEmail() → ofNullable() → "email은 null일 수 있다"
        // getPhone() → of() → "phone은 절대 null이 아니다"
    }
}
```

### 실험 3: 외부 입력 처리

```java
import java.util.Optional;

public class ExternalInputTest {
    
    // API 엔드포인트에서 사용자 이름을 받는다
    public Optional<String> processUserInput(String userInput) {
        // userInput은 외부에서 오는 값 → null일 수 있음
        
        // ❌ 잘못된 처리
        // Optional<String> opt1 = Optional.of(userInput);  // NPE 위험!
        
        // ✅ 올바른 처리
        // 1. 먼저 입력 검증
        String trimmed = userInput != null ? userInput.trim() : null;
        
        // 2. null 체크 후 wrapping
        if (trimmed != null && !trimmed.isEmpty()) {
            return Optional.of(trimmed);  // 검증 후 of()
        } else {
            return Optional.empty();  // 또는 ofNullable(trimmed)
        }
        
        // 대안: 직접 ofNullable 사용
        // return Optional.ofNullable(userInput)
        //     .map(String::trim)
        //     .filter(s -> !s.isEmpty());
    }
    
    public static void main(String[] args) {
        ExternalInputTest test = new ExternalInputTest();
        
        System.out.println("=== 유효한 입력 ===");
        Optional<String> result1 = test.processUserInput("  John  ");
        System.out.println("결과: " + result1);  // Optional[John]
        
        System.out.println("\n=== null 입력 ===");
        Optional<String> result2 = test.processUserInput(null);
        System.out.println("결과: " + result2);  // Optional.empty
        
        System.out.println("\n=== 빈 문자열 입력 ===");
        Optional<String> result3 = test.processUserInput("  ");
        System.out.println("결과: " + result3);  // Optional.empty
    }
}
```

---

## 📊 성능/비교

```
팩토리 메서드 성능 특성:

Optional.of(T value):
  검증: Objects.requireNonNull(value)  O(1)
  객체 생성: new Optional(value)  O(1)
  메모리: 항상 새 Optional 객체 생성 (16 바이트)
  
Optional.ofNullable(T value):
  조건: value == null ? empty() : of(value)
  null인 경우: EMPTY 싱글톤 반환 (기존 객체 재사용)
  null이 아닌 경우: of()와 동일하게 새 객체 생성
  메모리: null → 기존 객체 (0 바이트), non-null → 새 객체 (16 바이트)

Optional.empty():
  반환: EMPTY 싱글톤 (기존 객체)
  메모리: 0 바이트 (재사용)

벤치마크 (JMH, 1,000,000회 호출):

of(non-null 값):
  가능한 가장 빠름 (검증 + 생성)
  
ofNullable(non-null 값):
  of()와 동일 성능 (null 체크 추가, CPU 파이프라인 최적화)
  
ofNullable(null):
  매우 빠름 (EMPTY 반환, 객체 생성 없음)

대규모 배열에서 ofNullable() 사용:
  User[] users = ...  (100,000명, 10% null)
  
  Optional<String>[] emails = new Optional[100000];
  for (int i = 0; i < users.length; i++) {
      emails[i] = Optional.ofNullable(users[i].email);
  }
  
  메모리:
    - null인 경우 90,000개: EMPTY 재사용 → ~0 바이트
    - non-null인 경우 10,000개: new Optional → ~160 KB
    - 전체: 최소 메모리 소비
    
  vs of(검증) 사용:
    - 모든 경우 new → 1.6 MB (100배 차이)
```

---

## ⚖️ 트레이드오프

```
of() 사용:
  장점: null 명시적 거부, 즉시 오류 감지
  단점: 호출 시 NPE 위험 (잘못된 값 전달 시)

ofNullable() 사용:
  장점: 안전한 래핑, null 처리 명확
  단점: 의도 모호 가능 (항상 null 가능하다는 거짓 신호)

empty() 명시적 호출:
  장점: "없음" 상태 명시적 표현
  단점: 불필요한 분기 (보통 ofNullable()로 충분)

의도 표현의 트레이드오프:
  
  Optional.of(value)
    → "value는 절대 null이 아니어야 한다" (강함)
    → NPE는 버그, 즉시 고쳐야 함
  
  Optional.ofNullable(value)
    → "value는 null일 수 있고, 안전하게 처리한다" (약함)
    → null은 정상, Optional.empty()로 처리
  
  코드 리뷰:
    ofNullable() 남용 → "이 값이 정말 null일 수 있나?"
    of() 과다 사용 → "이 값이 정말 null 불가능한가? (검증됐나?)"
```

---

## 📌 핵심 정리

```
Optional.of() vs ofNullable() vs empty() 선택:

of(T value):
  - value가 절대 null이 아니어야 할 때
  - null → NullPointerException (즉시 버그 감지)
  - 의도: "값을 신뢰할 수 있다"

ofNullable(T value):
  - value가 null일 수 있을 때
  - null → Optional.empty() (안전한 처리)
  - non-null → new Optional(value)
  - 의도: "값은 null일 수 있고, 안전하게 처리한다"

empty():
  - 명시적으로 "값 없음" 표현
  - 항상 EMPTY 싱글톤 반환
  - 메모리 효율적

선택 기준:
  - 필드 검증됨? → of()
  - DB/외부 입력? → ofNullable()
  - 조건부 absent? → ofNullable() 또는 empty()

메모리:
  - empty() 또는 ofNullable(null): EMPTY 재사용
  - of() 또는 ofNullable(non-null): 새 객체 생성

API 설계:
  - 메서드 반환: Optional.of() 또는 ofNullable()만 사용
  - 필드: null 또는 기본값 (Optional 금지)
  - 매개변수: null 불가 또는 기본값 (Optional 금지)
```

---

## 🤔 생각해볼 문제

**Q1.** `Optional.of(null)`이 `NullPointerException`을 던지는 설계의 철학은?

<details>
<summary>해설 보기</summary>

Optional.of(null)의 설계는 프로그래머의 계약 위반을 즉시 드러내는 것이다.

철학:
  "Optional.of()를 호출한다" = "나는 이 값이 절대 null이 아니라고 확신한다"
  
  만약 null이 전달되면:
    → 내 확신이 틀렸다 (입력 검증 실패, 로직 오류)
    → 즉시 NullPointerException으로 버그를 드러냄
    → 조용히 Optional.empty()를 반환하지 않음

이것이 null을 "무음 오류(silent error)"처럼 다루는 것을 회피하는 설계다.

비교:
  // ❌ 나쁜 설계: 조용히 무시
  Optional<String> opt = value == null ? Optional.empty() : new Optional(value);
  // 호출자는 왜 empty()인지 모름
  
  // ✅ 좋은 설계: 명시적 오류
  Optional<String> opt = Optional.of(value);  // null → NullPointerException
  // 호출자는 즉시 "아, value 검증을 빠뜨렸다" 깨닮음

실무적 의미:
  of()를 사용하려면 value가 null 불가능함을 증명해야 함:
    1. 생성자/파라미터에서 검증
    2. final 필드로 초기화
    3. Objects.requireNonNull() 사용
    4. if (value != null) 체크 후 호출

이것이 Optional을 "안전한" 도구로 만드는 핵심이다.

</details>

---

**Q2.** 대규모 데이터 처리에서 `ofNullable()` 과다 사용의 영향은?

<details>
<summary>해설 보기</summary>

null이 많은 데이터에서 ofNullable()은 효율적이지만, 구조적 문제가 있다.

예: 100,000명 사용자, 이메일이 30% null인 경우

효율적 처리:
  Optional<String>[] emails = new Optional[100000];
  for (User user : users) {
      emails[i] = Optional.ofNullable(user.email);
  }
  
  메모리:
    - 30,000개: EMPTY 재사용 (0 바이트)
    - 70,000개: new Optional (70,000 × 16 = 1.12 MB)
    - 전체: ~1.12 MB
  
  읽기:
    emails[i].ifPresent(email -> process(email));
    → Optional 래핑/언래핑 오버헤드

비효율적 처리:
  // null 제거하고 Stream 사용
  users.stream()
      .map(User::getEmail)
      .filter(Objects::nonNull)  // null 필터링
      .forEach(email -> process(email));
  
  메모리: 필터링만 수행, Optional 래핑 없음
  읽기: 직접 String 처리, 오버헤드 없음

구조적 문제:
  if (데이터의 null이 예외적) → Optional.ofNullable() 좋음
  if (데이터의 null이 흔함 > 20%) → 다른 설계 고려:
    - 기본값 사용 (email = "unknown@example.com")
    - 필드 분리 (hasEmail: boolean, email: String)
    - Map<User, String> 사용 (null 키 미지원이지만 명시적)

결론: ofNullable()은 "가끔" null인 상황에 최적. null이 흔한 경우 구조적 재설계 고려.

</details>

---

**Q3.** 코드 리뷰에서 `Optional.of()`와 `ofNullable()` 사용을 어떻게 평가할 것인가?

<details>
<summary>해설 보기</summary>

리뷰 관점: 각 선택이 의도를 명확히 표현하는가?

of() 사용 리뷰:
  public Optional<String> getUserId() {
      return Optional.of(user.id);
  }
  
  질문: "user.id가 절대 null일 수 없나?"
    - 답: 생성자에서 Objects.requireNonNull() 한다 → ✅ OK
    - 답: final 필드, 초기화됨 → ✅ OK
    - 답: 모르겠다 → ❌ 검증 추가

ofNullable() 사용 리뷰:
  public Optional<String> getUserEmail() {
      return Optional.ofNullable(user.email);
  }
  
  질문: "user.email이 정말 null일 수 있나?"
    - 답: DB에서 nullable 컬럼 → ✅ OK
    - 답: 설정에서 선택사항 → ✅ OK
    - 답: 항상 입력되는데 실수로 ofNullable → ❌ of()로 변경

리뷰 원칙:
  1. of()와 ofNullable() 혼용 패턴 발견 → 의도 확인
  2. 모든 Optional 반환이 ofNullable() → 과다 사용 의심
  3. of()에서 NPE 발생 → "입력 검증하셨나?" 질문
  4. ofNullable()로 불필요한 Optional 생성 → 메모리 영향 검토

자동화:
  정적 분석 도구 설정:
    - of()에 잠재적 null 인자 감지 → 경고
    - 필드 검증 여부 확인 → of() 안전성 검증
    - Optional 과다 생성 패턴 감지 → 구조 리뷰

결론: 각 선택이 의도를 명확히 해야 함. 거의 모든 Optional은 ofNullable()이어야 정상.

</details>

---

<div align="center">

**[⬅️ 이전: Optional 내부 구조](./01-optional-internal-structure.md)** | **[홈으로 🏠](../README.md)** | **[다음: map vs flatMap ➡️](./03-map-vs-flatmap-monad.md)**

</div>
