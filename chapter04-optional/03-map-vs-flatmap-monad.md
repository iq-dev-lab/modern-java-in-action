# map vs flatMap — Functor와 Monad 패턴

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `map()`이 `Optional<T> → Optional<U>`로 변환하는 Functor인 반면, `flatMap()`이 평탄화(Monad)인 이유는?
- `Optional<Optional<T>>`가 왜 발생하고, 어떻게 방지하는가?
- 함수형 카테고리 이론의 Functor/Monad 법칙이 Optional에 어떻게 적용되는가?
- 실무에서 `map` 체인과 `flatMap` 체인의 성능 차이는?
- Stream의 `map`과 `Optional`의 `map`은 같은 원리인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Optional은 단순한 null 체크 도구가 아니라 Functor와 Monad의 구체적 구현이다. `map`과 `flatMap`의 차이를 이해하지 못하면, Optional을 제대로 사용할 수 없을 뿐 아니라, 함수형 프로그래밍의 핵심 원리(합성, 체이닝)를 놓치게 된다. Stream, CompletableFuture 등의 고차 추상화도 같은 원리를 따르므로, Optional에서 마스터하면 Java 전체의 함수형 설계를 이해할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: flatMap 대신 map 사용
  Optional<User> user = findUserById(id)
      .map(u -> findEmailById(u.getId()))
      // findEmailById(...)가 Optional<Email>을 반환
      // 결과: Optional<Optional<Email>> ❌
  
  이제 Optional<Optional<Email>>을 처리하려면?
  result.get().get()  // get() 이중 호출 → NPE 위험
  
  해결:
  Optional<Email> email = findUserById(id)
      .flatMap(u -> findEmailById(u.getId()))
      // flatMap()은 자동으로 Optional<Optional<Email>> 평탄화
      // 결과: Optional<Email> ✅

실수 2: map() 체인에서 null 처리 누락
  Optional<String> result = Optional.of(user)
      .map(u -> u.getEmail())  // String 또는 null?
      .map(e -> e.toLowerCase())  // null이면 NPE!
  
  u.getEmail()이 null을 반환할 수 있으면:
      .map(u -> u.getEmail())  // Optional을 벗기고 plain String
      // String이 null이면? → NPE
  
  해결: ofNullable()으로 래핑
  .map(u -> Optional.ofNullable(u.getEmail()))
      .flatMap(Function.identity())  // Optional<Optional<String>> → Optional<String>

실수 3: flatMap의 람다에서 직접 값 반환
  Optional<String> email = findUser(id)
      .flatMap(user -> user.getEmail());  // user.getEmail()이 String이면?
  
  flatMap(Function<T, Optional<U>>)은 Optional 반환을 기대
  String을 반환하면 컴파일 에러
  
  올바른 사용:
  .flatMap(user -> Optional.ofNullable(user.getEmail()))

실수 4: Optional<Optional<T>> 다중 언래핑
  Optional<Optional<Optional<String>>> nested = ...
  
  String value = nested.get().get().get();  // get() 삼중 호출, 모두 NPE 위험
  
  대신:
  Optional<String> value = nested
      .flatMap(Function.identity())
      .flatMap(Function.identity());
  
  또는 Stream으로 변환:
  Stream<String> stream = nested.stream()
      .flatMap(Optional::stream)
      .flatMap(Optional::stream);
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// Functor (map) vs Monad (flatMap) 올바른 사용

// 패턴 1: map() — 값 변환 (Functor)
Optional<User> user = findUserById(1L);
Optional<Integer> age = user.map(u -> u.getAge());  // User → Integer
// 결과: Optional<Integer>

// 패턴 2: flatMap() — 값 변환 후 Optional 반환 (Monad)
Optional<Email> email = findUserById(1L)
    .flatMap(user -> findEmailByUserId(user.getId()));
// User → Optional<Email> → Optional<Email> (평탄화)

// 패턴 3: 체이닝 조합
Optional<String> domain = findUserById(1L)
    .flatMap(user -> findEmailByUserId(user.getId()))  // Optional<Email>
    .map(email -> email.getDomain())  // Email → String
    .map(d -> d.toLowerCase());  // String → String

// 패턴 4: filter와 map의 조합
Optional<String> verified = findUserById(1L)
    .flatMap(user -> findEmailByUserId(user.getId()))
    .filter(email -> email.isVerified())  // filter는 Optional 반환
    .map(email -> email.getAddress());  // map으로 값 추출

// 패턴 5: Optional을 Stream으로 변환
List<String> emails = userIds.stream()
    .map(id -> findUserById(id))  // Optional<User>
    .flatMap(Optional::stream)  // Optional<User> → Stream<User> (0개 또는 1개)
    .flatMap(user -> findEmailByUserId(user.getId()).stream())
    .collect(Collectors.toList());

// 패턴 6: ifPresentOrElse로 분기
findUserById(id)
    .ifPresentOrElse(
        user -> System.out.println("사용자: " + user),
        () -> System.out.println("사용자 없음")
    );

// 패턴 7: 중첩 Optional 평탄화
Optional<Optional<String>> nested = Optional.of(Optional.of("value"));
Optional<String> flat = nested
    .flatMap(Function.identity());  // Optional<Optional<T>> → Optional<T>

// 패턴 8: Monad 체인으로 복잡한 계산
Optional<String> result = getUserId(1L)
    .flatMap(userId -> getUser(userId))
    .flatMap(user -> getPreferences(user.getId()))
    .flatMap(prefs -> validateEmail(prefs.getEmail()))
    .map(email -> email.toUpperCase());
```

---

## 🔬 내부 동작 원리

### 1. map() — Functor 패턴

```java
// OpenJDK Optional.map(Function<? super T, ? extends U> mapper)
public <U> Optional<U> map(Function<? super T, ? extends U> mapper) {
    Objects.requireNonNull(mapper);
    if (!isPresent())
        return empty();  // Optional이 empty면 empty 반환
    else
        return Optional.ofNullable(mapper.apply(value));
        //    ↑ mapper 결과가 null이면 empty 반환
}

동작:
  Optional<User> user = Optional.of(new User("John"));
  
  Optional<Integer> age = user.map(u -> u.getAge());
  
  내부:
    1. isPresent() = true
    2. mapper.apply(user) = 30 (age 값)
    3. Optional.ofNullable(30) = Optional.of(30)
    4. 반환: Optional[30]

  Functor 법칙:
    f: User → Integer (순수 함수)
    Optional<User> → Optional<Integer> 변환
    원소가 없으면 그냥 empty 반환

메모리:
  Optional[User] --map(mapper)--> Optional[Integer]
  
  새 Optional 객체 생성 (여전히 하나의 Optional)
  Optional<Optional<Integer>> 아님!
```

### 2. flatMap() — Monad 패턴

```java
// OpenJDK Optional.flatMap(Function<? super T, ? extends Optional<? extends U>> mapper)
public <U> Optional<U> flatMap(Function<? super T, ? extends Optional<? extends U>> mapper) {
    Objects.requireNonNull(mapper);
    if (!isPresent())
        return empty();
    else
        return Objects.requireNonNull(mapper.apply(value));
        //    ↑ mapper 결과는 Optional<U>
        //    ↑ 평탄화: Optional<Optional<U>> → Optional<U>
}

동작:
  Optional<User> user = Optional.of(new User(1L, "John"));
  
  Optional<Email> email = user.flatMap(
      u -> Optional.ofNullable(emailDb.findByUserId(u.getId()))
      // findByUserId는 Email 또는 null을 반환
  );
  
  내부:
    1. isPresent() = true
    2. mapper.apply(user)
       = Optional.ofNullable(emailDb.findByUserId(1L))
       = Optional[Email] (이미 Optional)
    3. requireNonNull() = Optional[Email]
       평탄화: Optional<Optional<Email>> → Optional<Email>
    4. 반환: Optional[Email]

메모리:
  Optional[User]
    --flatMap(mapper)-->
  mapper.apply(user) = Optional[Email]  // mapper가 이미 Optional 반환
  
  결과: Optional[Email] (중첩 없음, 평탄화됨)
  
  vs map()으로 같은 일을 하면:
  Optional<Optional<Email>> ❌ (중첩)
```

### 3. Functor 법칙 (Optional에서)

```
Functor 법칙: 범주론의 구조 보존 원리

Optional은 Functor이다:
  - 오브젝트: User, Integer, String, ...
  - 모피즘(화살표): 함수 f: User → Integer
  - Functor: Optional f: Optional[User] → Optional[Integer]

Functor 제1 법칙: 항등원소 보존
  optional.map(Function.identity()) == optional
  
  확인:
  Optional<String> opt = Optional.of("hello");
  Optional<String> result = opt.map(x -> x);
  opt.equals(result)  // true

Functor 제2 법칙: 합성 보존
  optional.map(f).map(g) == optional.map(f.andThen(g))
  
  확인:
  Optional<String> opt = Optional.of("hello");
  
  방법 1: 체이닝
  Optional<Integer> r1 = opt.map(String::length).map(n -> n * 2);
  
  방법 2: 합성
  Function<String, Integer> f = String::length;
  Function<Integer, Integer> g = n -> n * 2;
  Optional<Integer> r2 = opt.map(f.andThen(g));
  
  r1.equals(r2)  // true

실무 의미:
  map() 체이닝은 수학적으로 안전하다
  optional.map(a).map(b).map(c)
    = optional.map(a.andThen(b).andThen(c))
  
  순서는 보존된다 (함수 합성)
  중간에 null이 나타나도 자동으로 empty 처리된다
```

### 4. Monad 법칙 (Optional에서)

```
Monad 법칙: Functor + 평탄화 원리

Optional은 Monad이다:
  - 값을 감싸기: Optional.of(value) / ofNullable(value)
  - 감싼 값을 변환: flatMap(f)
  - 평탄화: Optional<Optional<T>> → Optional<T>

Monad 제1 법칙: Left Identity (왼쪽 항등원소)
  Optional.of(a).flatMap(f) == f.apply(a)
  
  확인:
  Integer a = 42;
  Function<Integer, Optional<String>> f = n -> Optional.of("answer: " + n);
  
  Optional<String> left = Optional.of(a).flatMap(f);
  Optional<String> right = f.apply(a);
  
  left.equals(right)  // true

Monad 제2 법칙: Right Identity (오른쪽 항등원소)
  m.flatMap(Optional::of) == m
  
  확인:
  Optional<String> m = Optional.of("hello");
  Optional<String> result = m.flatMap(Optional::of);
  
  m.equals(result)  // true

Monad 제3 법칙: Associativity (결합법칙)
  m.flatMap(f).flatMap(g) == m.flatMap(x -> f.apply(x).flatMap(g))
  
  확인:
  Optional<Integer> m = Optional.of(5);
  Function<Integer, Optional<Integer>> f = n -> Optional.of(n * 2);
  Function<Integer, Optional<Integer>> g = n -> Optional.of(n + 1);
  
  Optional<Integer> left = m.flatMap(f).flatMap(g);
  // = Optional.of(5).flatMap(n -> Optional.of(n * 2))  // Optional[10]
  // .flatMap(n -> Optional.of(n + 1))  // Optional[11]
  
  Optional<Integer> right = m.flatMap(x -> f.apply(x).flatMap(g));
  // = Optional.of(5).flatMap(n -> Optional.of(n * 2).flatMap(n2 -> Optional.of(n2 + 1)))
  // = Optional.of(5).flatMap(n -> Optional.of(n * 2 + 1))
  
  left.equals(right)  // true

실무 의미:
  flatMap() 체이닝은 수학적으로 안전하다
  optional.flatMap(a).flatMap(b).flatMap(c)
    = optional.flatMap(x -> a(x).flatMap(b).flatMap(c))
  
  순서는 보존된다
  중간의 Optional<Optional<T>>도 자동으로 평탄화된다
```

### 5. 중첩 Optional 제거 메커니즘

```
문제: 중첩 Optional 발생

Optional.map() 사용 시:
  Optional<User> user = Optional.of(new User(1, "John"));
  Optional<Optional<Email>> nested = user.map(
      u -> emailDb.findByUserId(u.getId())  // Optional<Email> 반환
  );
  // map은 변환 결과를 Optional로 래핑
  // 결과: Optional<Optional<Email>>

flatMap() 사용 시:
  Optional<Email> flat = user.flatMap(
      u -> emailDb.findByUserId(u.getId())  // Optional<Email> 반환
  );
  // flatMap은 변환 결과가 이미 Optional이면 평탄화
  // 결과: Optional<Email>

평탄화의 내부 원리:
  mapper.apply(value)가 Optional<Email>을 반환
  
  flatMap: Optional.of(Optional<Email>) → requireNonNull() → Optional<Email>
                                           └─→ unwrap ────→
  
  map:     Optional.of(Optional<Email>) → 그냥 반환
           └─→ wrap in another Optional

중첩 제거 (Stream 방식):
  Optional<Optional<Optional<String>>> triple = ...
  
  // Option 1: flatMap 연쇄
  Optional<String> single = triple
      .flatMap(Function.identity())
      .flatMap(Function.identity());
  
  // Option 2: Stream 변환
  Optional<String> single = triple.stream()
      .flatMap(Optional::stream)
      .flatMap(Optional::stream)
      .findFirst();
  
  // Option 3: custom unwrapper
  static <T> Optional<T> flatten(Optional<Optional<T>> opt) {
      return opt.flatMap(Function.identity());
  }
```

---

## 💻 실전 실험

### 실험 1: map() vs flatMap() 비교

```java
import java.util.Optional;
import java.util.function.Function;

public class MapVsFlatMapTest {
    
    static class User {
        long id;
        String name;
        User(long id, String name) { this.id = id; this.name = name; }
        long getId() { return id; }
        String getName() { return name; }
    }
    
    static Optional<User> findUserById(long id) {
        if (id == 1) return Optional.of(new User(1, "Alice"));
        return Optional.empty();
    }
    
    static Optional<String> findEmailByUserId(long userId) {
        if (userId == 1) return Optional.of("alice@example.com");
        return Optional.empty();
    }
    
    public static void main(String[] args) {
        System.out.println("=== map() 사용 (Functor) ===");
        
        // map: User → Optional<String>
        Optional<Optional<String>> nested = findUserById(1)
            .map(user -> findEmailByUserId(user.getId()));
        
        System.out.println("타입: Optional<Optional<String>>");
        System.out.println("값: " + nested);  // Optional[Optional[alice@example.com]]
        System.out.println("값 추출: " + nested.get().get());  // 이중 get() 필요
        
        System.out.println("\n=== flatMap() 사용 (Monad) ===");
        
        // flatMap: User → Optional<String>, 자동 평탄화
        Optional<String> flat = findUserById(1)
            .flatMap(user -> findEmailByUserId(user.getId()));
        
        System.out.println("타입: Optional<String>");
        System.out.println("값: " + flat);  // Optional[alice@example.com]
        System.out.println("값 추출: " + flat.get());  // 단일 get()
        
        System.out.println("\n=== 체이닝 비교 ===");
        
        // map 체인: Optional 레벨에서 변환
        Optional<String> result1 = findUserById(1)
            .map(user -> user.getName())
            .map(name -> name.toUpperCase());
        System.out.println("map 체인 (동일 타입): " + result1);  // Optional[ALICE]
        
        // flatMap 체인: 각 단계마다 Optional 처리
        Optional<String> result2 = findUserById(1)
            .flatMap(user -> Optional.of(user.getName()))
            .flatMap(name -> Optional.of(name.toUpperCase()));
        System.out.println("flatMap 체인: " + result2);  // Optional[ALICE]
    }
}

// 출력:
// === map() 사용 (Functor) ===
// 타입: Optional<Optional<String>>
// 값: Optional[Optional[alice@example.com]]
// 값 추출: alice@example.com
//
// === flatMap() 사용 (Monad) ===
// 타입: Optional<String>
// 값: Optional[alice@example.com]
// 값 추출: alice@example.com
//
// === 체이닝 비교 ===
// map 체인 (동일 타입): Optional[ALICE]
// flatMap 체인: Optional[ALICE]
```

### 실험 2: Monad 법칙 검증

```java
import java.util.Optional;
import java.util.function.Function;

public class MonadLawTest {
    
    public static void main(String[] args) {
        System.out.println("=== Monad 제1 법칙: Left Identity ===");
        
        Integer a = 42;
        Function<Integer, Optional<String>> f = 
            n -> Optional.of("number: " + n);
        
        Optional<String> left = Optional.of(a).flatMap(f);
        Optional<String> right = f.apply(a);
        
        System.out.println("Optional.of(a).flatMap(f): " + left);
        System.out.println("f.apply(a): " + right);
        System.out.println("동등: " + left.equals(right));
        
        System.out.println("\n=== Monad 제2 법칙: Right Identity ===");
        
        Optional<String> m = Optional.of("hello");
        Optional<String> result = m.flatMap(Optional::of);
        
        System.out.println("m: " + m);
        System.out.println("m.flatMap(Optional::of): " + result);
        System.out.println("동등: " + m.equals(result));
        
        System.out.println("\n=== Monad 제3 법칙: Associativity ===");
        
        Optional<Integer> opt = Optional.of(5);
        Function<Integer, Optional<Integer>> double_ = 
            n -> Optional.of(n * 2);
        Function<Integer, Optional<Integer>> addOne = 
            n -> Optional.of(n + 1);
        
        // 왼쪽: (m.flatMap(f)).flatMap(g)
        Optional<Integer> left3 = opt
            .flatMap(double_)
            .flatMap(addOne);
        
        // 오른쪽: m.flatMap(x -> f(x).flatMap(g))
        Optional<Integer> right3 = opt
            .flatMap(x -> double_.apply(x).flatMap(addOne));
        
        System.out.println("(m.flatMap(f)).flatMap(g): " + left3);
        System.out.println("m.flatMap(x -> f(x).flatMap(g)): " + right3);
        System.out.println("동등: " + left3.equals(right3));
    }
}

// 출력:
// === Monad 제1 법칙: Left Identity ===
// Optional.of(a).flatMap(f): Optional[number: 42]
// f.apply(a): Optional[number: 42]
// 동등: true
//
// === Monad 제2 법칙: Right Identity ===
// m: Optional[hello]
// m.flatMap(Optional::of): Optional[hello]
// 동등: true
//
// === Monad 제3 법칙: Associativity ===
// (m.flatMap(f)).flatMap(g): Optional[11]
// m.flatMap(x -> f(x).flatMap(g)): Optional[11]
// 동등: true
```

### 실험 3: Stream과의 비교

```java
import java.util.Optional;
import java.util.stream.Stream;

public class OptionalStreamTest {
    
    public static void main(String[] args) {
        System.out.println("=== Optional은 Stream처럼 동작 ===");
        
        Optional<String> opt = Optional.of("hello");
        
        // Optional.stream() → Stream (Java 9+)
        Stream<String> stream = opt.stream();
        long count = stream.count();
        System.out.println("Optional.stream() count: " + count);  // 1
        
        System.out.println("\n=== Optional vs Stream 비교 ===");
        
        // Optional은 0개 또는 1개 원소의 Stream
        Optional<String> present = Optional.of("value");
        Optional<String> empty = Optional.empty();
        
        System.out.println("present.stream().count(): " + present.stream().count());  // 1
        System.out.println("empty.stream().count(): " + empty.stream().count());  // 0
        
        System.out.println("\n=== Optional<Optional<T>>를 Stream으로 제거 ===");
        
        Optional<Optional<String>> nested = Optional.of(Optional.of("nested"));
        
        Optional<String> flat = nested.stream()
            .flatMap(Optional::stream)
            .findFirst();
        
        System.out.println("중첩된 Optional: " + nested);
        System.out.println("평탄화된 Optional: " + flat);
        
        System.out.println("\n=== flatMap(Optional::stream) 이용한 필터링 ===");
        
        java.util.List<Optional<String>> list = java.util.List.of(
            Optional.of("alice"),
            Optional.empty(),
            Optional.of("bob")
        );
        
        java.util.List<String> result = list.stream()
            .flatMap(Optional::stream)  // empty는 제거됨
            .toList();
        
        System.out.println("List<Optional<String>>: " + list);
        System.out.println("필터링 후: " + result);
    }
}

// 출력:
// === Optional은 Stream처럼 동작 ===
// Optional.stream() count: 1
//
// === Optional vs Stream 비교 ===
// present.stream().count(): 1
// empty.stream().count(): 0
//
// === Optional<Optional<T>>를 Stream으로 제거 ===
// 중첩된 Optional: Optional[Optional[nested]]
// 평탄화된 Optional: Optional[nested]
//
// === flatMap(Optional::stream) 이용한 필터링 ===
// List<Optional<String>>: [Optional[alice], Optional.empty, Optional[bob]]
// 필터링 후: [alice, bob]
```

---

## 📊 성능/비교

```
map() vs flatMap() 성능:

메모리:
  map(f) where f: T → U
    Optional<T> → Optional<U>
    Optional 객체 1개 생성
  
  flatMap(f) where f: T → Optional<U>
    Optional<T> → Optional<U>
    Optional 객체 최대 2개 임시 생성 (f의 결과 + 평탄화 과정)
    하지만 결과는 1개

체인 성능 (1,000,000회 체이닝):
  map() 체인:
    Optional<String> result = opt
        .map(String::toLowerCase)
        .map(String::trim)
        .map(String::length);  // 변환 성공, 실패 각 경우
    
    결과: 세 개의 Optional 객체 생성/GC
  
  flatMap() 체인 (같은 변환):
    Optional<String> result = opt
        .flatMap(s -> Optional.of(s.toLowerCase()))
        .flatMap(s -> Optional.of(s.trim()))
        .flatMap(s -> Optional.of(String.valueOf(s.length())));
    
    결과: 네 개의 Optional 객체 생성/GC
    불필요한 평탄화 비용

결론: 다음 변환이 Optional을 반환하지 않으면 map()이 낫다.

캐시 효율성:
  Optional 체인이 길수록 (10회 이상)
  각 단계마다 Optional 객체 생성 → CPU 캐시 미스 증가
  최신 JIT 컴파일러는 이를 최적화
```

---

## ⚖️ 트레이드오프

```
map() 사용:
  장점: 간단한 값 변환, 직관적
  단점: 중첩 Optional 가능 (f가 Optional 반환하면)

flatMap() 사용:
  장점: 중첩 Optional 자동 제거, 안전한 체이닝
  단점: 람다가 Optional을 반환해야 함 (더 복잡)

혼용:
  장점: map() + flatMap() 조합으로 복잡한 로직 표현
  단점: 코드 가독성 떨어질 수 있음

vs if-else null 체크:
  map/flatMap:
    장점: 함수형, 체이닝 가능, 간결
    단점: 람다 오버헤드, 중간값 접근 어려움
  
  if-else:
    장점: 명확한 제어 흐름, 성능 최적화 가능
    단점: 보일러플레이트, 중첩 조건 복잡도
```

---

## 📌 핵심 정리

```
map() vs flatMap():

map(Function<T, U>):
  - T → U 변환 (Optional 아님)
  - 결과: Optional<U>
  - Functor 패턴
  - 중첩 Optional 가능 (f가 Optional 반환하면)

flatMap(Function<T, Optional<U>>):
  - T → Optional<U> 변환 (Optional 반환)
  - 결과: Optional<U> (평탄화됨)
  - Monad 패턴
  - 중첩 자동 제거

Functor 법칙:
  1. map(identity) == 자신
  2. map(f).map(g) == map(f.andThen(g))

Monad 법칙:
  1. Optional.of(a).flatMap(f) == f(a)
  2. m.flatMap(Optional::of) == m
  3. m.flatMap(f).flatMap(g) == m.flatMap(x -> f(x).flatMap(g))

실무:
  - 다음 단계가 Optional을 반환? → flatMap()
  - 단순 값 변환? → map()
  - 중첩 Optional 제거? → flatMap(Function.identity()) 또는 stream()
```

---

## 🤔 생각해볼 문제

**Q1.** `Optional<Optional<T>>`가 발생하는 실제 상황은?

<details>
<summary>해설 보기</summary>

Optional<Optional<T>>는 여러 상황에서 발생한다:

상황 1: 데이터베이스 조회 체인
  public Optional<User> findUser(String email) { ... }  // null 가능
  public Optional<Address> findAddressByUserId(long id) { ... }  // null 가능
  
  Optional<Optional<Address>> nested = findUser(email)
      .map(user -> findAddressByUserId(user.getId()));  // map이 Optional 반환
  
  문제: Address가 없으면 Optional.empty(), User가 없으면 바깥쪽 Optional.empty()
  → Optional<Optional<Address>> 또는 Optional.empty() 구분 불명확

상황 2: 조건부 Optional 래핑
  Optional<String> email = getEmail(user);  // null 가능
  
  Optional<Optional<String>> confused = Optional.of(email);
  // email이 Optional이므로 double wrap
  
  올바른 방법:
  Optional<String> direct = Optional.ofNullable(email.orElse(null));

상황 3: 메서드 체인에서 실수
  userRepository.findById(id)
      .map(user -> Optional.ofNullable(user.getEmail()))
      // map은 Optional을 또 다른 Optional로 감싼다
  
  올바른 방법:
  userRepository.findById(id)
      .flatMap(user -> Optional.ofNullable(user.getEmail()))

결론: map()으로 Optional을 반환하는 함수를 호출하면 중첩 발생. flatMap()을 사용하거나 사전에 값을 추출해서 Optional로 만들어야 한다.

</details>

---

**Q2.** Functor 법칙과 Monad 법칙의 실무적 의미는?

<details>
<summary>해설 보기</summary>

이 법칙들은 수학적 보장이다. Optional 설계자가 이 법칙을 보장함으로써, 프로그래머는 map/flatMap 체인을 안전하게 리팩토링할 수 있다.

Functor 제2 법칙: map(f).map(g) == map(f.andThen(g))

실무 응용:
  // 원본 코드
  result = optional
      .map(user -> user.getEmail())
      .map(email -> email.toLowerCase());
  
  // 성능 최적화 (JVM이 할 수도, 프로그래머가 할 수도)
  Function<String, String> combined = ((Function<User, String>) User::getEmail)
      .andThen(String::toLowerCase);
  result = optional.map(combined);
  
  두 코드는 수학적으로 동등하다. 따라서:
  - 벤치마크에서 성능 차이가 나면 JVM 최적화 부족 → 버그 가능성
  - 리팩토링이 의미 보존한다 → 안전한 리팩토링

Monad 제3 법칙: 결합법칙

실무 응용:
  // 복잡한 체인
  result = optional
      .flatMap(a)
      .flatMap(b)
      .flatMap(c);
  
  // 최적화 (병렬 처리 등)
  result = optional.flatMap(x -> a(x).flatMap(b).flatMap(c));
  
  두 형태는 동등하다. 따라서:
  - 병렬화, 캐싱, 지연 평가 가능
  - 중간 결과 검사 후 조건부 처리 추가 가능
  
  예: 디버깅 추가
  result = optional
      .flatMap(x -> {
          Optional<T1> a1 = a(x);
          System.out.println("a 결과: " + a1);
          return a1.flatMap(b).flatMap(c);
      });

결론: 이 법칙들이 없으면 map/flatMap 체인을 리팩토링할 때 항상 버그 위험이 있다. 법칙이 있으므로, 안전하게 코드를 변형할 수 있다.

</details>

---

**Q3.** `Stream.flatMap(Optional::stream)`으로 `Optional<Optional<T>>`를 제거하는 원리는?

<details>
<summary>해설 보기</summary>

이것은 Optional을 Stream으로 변환한 후 stream을 다시 평탄화하는 기법이다.

원리:
  Optional<T>는 0개 또는 1개 원소의 Stream으로 볼 수 있다.
  
  Optional.of(x).stream() → Stream[x]  (1개)
  Optional.empty().stream() → Stream[]  (0개)

예:
  Optional<Optional<String>> nested = Optional.of(Optional.of("value"));
  
  // Option 1: flatMap으로 직접 제거
  nested.flatMap(Function.identity());  // Optional[value]
  
  // Option 2: Stream으로 변환해서 제거
  nested.stream()                       // Stream[Optional[value]]
        .flatMap(Optional::stream)      // Stream[value]
        .findFirst();                   // Optional[value]

동작:
  1. nested.stream() → Stream[Optional[value]]
     (1개의 Optional을 Stream으로 변환)
  
  2. .flatMap(Optional::stream)
     각 Optional을 stream으로 변환한 후 평탄화
     Optional[value].stream() → Stream[value]
     결과: Stream[value]
  
  3. .findFirst() → Optional[value]

언제 유용한가:
  List<Optional<String>> list = ...
  
  Stream API와의 조합이 필요할 때:
  list.stream()
      .flatMap(Optional::stream)  // List<Optional<T>> → Stream<T>
      .filter(s -> s.length() > 3)
      .map(String::toUpperCase)
      .collect(Collectors.toList());
  
  flatMap(Optional::stream)이 null 제거 역할도 하고, Stream 변환도 한다.

결론: Optional::stream은 Optional을 Stream으로 본다는 "구조적 사고"를 가능하게 한다. 이를 통해 Stream API의 강력한 조합성을 Optional에까지 확장할 수 있다.

</details>

---

<div align="center">

**[⬅️ 이전: Optional.of vs ofNullable vs empty](./02-of-vs-ofnullable-vs-empty.md)** | **[홈으로 🏠](../README.md)** | **[다음: Optional 안티패턴 ➡️](./04-optional-antipatterns.md)**

</div>
