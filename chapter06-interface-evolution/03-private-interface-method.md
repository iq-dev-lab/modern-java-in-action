# Private Interface Method (Java 9) — 코드 중복 제거

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Java 9에서 `private` 인터페이스 메서드를 도입한 이유는?
- `private`, `private static` 인터페이스 메서드의 차이는 무엇인가?
- Default method 간 공통 로직을 private 메서드로 캡슐화하는 방식은?
- 기존 `static` helper 클래스 패턴 대비 응집도가 어떻게 향상되는가?
- JDK 표준 라이브러리(예: `Collection`, `Stream`)에서 private 메서드는 어디에 쓰이는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Java 8에서 Default Method를 도입했지만, 여러 default method가 공통 로직을 공유할 때 문제가 생겼다. 예를 들어, `java.util.Collection`의 `removeIf()`, `forEach()` 같은 default method들이 내부적으로 반복 로직을 중복 구현했다. Java 9의 `private` 인터페이스 메서드는 이 중복을 제거하고 인터페이스 내에서 캡슐화를 강화했다. 이를 모르면 대규모 라이브러리 코드에서 불필요한 중복을 만들거나, 코드 변경 시 여러 곳을 수정해야 하는 상황이 발생한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 여러 default method의 공통 로직을 중복으로 구현
  interface Processor {
      default void validateAndProcess(Data d) {
          if (d == null) throw new NullPointerException();
          // 복잡한 검증 로직 100줄
          process(d);
      }
      
      default void processMultiple(List<Data> list) {
          for (Data d : list) {
              if (d == null) throw new NullPointerException();
              // 동일한 검증 로직 100줄 (복붙!)
              process(d);
          }
      }
      
      void process(Data d);
  }
  
  // 문제: 검증 로직 변경 시 두 곳 수정 필요
  // → 한 곳만 수정하면 버그 발생 가능

실수 2: Helper 클래스를 공개로 노출
  public class CollectionHelper {
      // 모든 인터페이스 구현체가 사용하는 유틸리티
      public static void validateNotNull(Object o) { ... }
      public static void checkCapacity(int current, int required) { ... }
  }
  
  // 문제:
  // - 공개 API로 보이지만 내부용 클래스
  // - 버전 호환성 유지 부담
  // - 사용자가 이 클래스에 의존할 수 있음

실수 3: 인터페이스와 분리된 클래스에 로직 분산
  interface Collection { ... }
  class CollectionHelper { ... }
  class StreamHelper { ... }
  
  // 문제:
  // - 관련 로직이 물리적으로 떨어져 있음
  // - 유지보수성 저하
  // - 응집도 낮음
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 올바른 패턴 1: private 메서드로 공통 로직 캡슐화
interface Processor {
    default void validateAndProcess(Data d) {
        validateInput(d);
        process(d);
    }
    
    default void processMultiple(List<Data> list) {
        for (Data d : list) {
            validateInput(d);
            process(d);
        }
    }
    
    // private 메서드로 공통 검증 로직 추출
    private void validateInput(Data d) {
        if (d == null) throw new NullPointerException();
        if (d.isEmpty()) throw new IllegalArgumentException("Empty data");
    }
    
    void process(Data d);
}

// 올바른 패턴 2: private static 메서드로 유틸리티 함수 제공
interface StreamFactory {
    default <T> Stream<T> asParallelStream() {
        return createStream().parallel();
    }
    
    default <T> Stream<T> asSequentialStream() {
        return createStream().sequential();
    }
    
    // private static으로 스트림 생성 로직 통합
    private static <T> Stream<T> createStream() {
        // 복잡한 스트림 생성 로직 (한 곳에만)
        return StreamSupport.stream(createSpliterator(), false);
    }
    
    // 인스턴스 정보 필요 없는 헬퍼
    Spliterator<?> createSpliterator();
}

// 올바른 패턴 3: private instance method와 private static method 혼합
interface DataCollection {
    // public interface 메서드
    default void forEach(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        forEachImpl(action);
    }
    
    default void removeIf(Predicate<? super E> filter) {
        Objects.requireNonNull(filter);
        removeIfImpl(filter);
    }
    
    // private instance method (null 체크 후 공통 로직)
    private void forEachImpl(Consumer<? super E> action) {
        for (E e : this) {
            action.accept(e);
        }
    }
    
    private void removeIfImpl(Predicate<? super E> filter) {
        iterator().forEachRemaining(e -> {
            if (filter.test(e)) {
                iterator().remove();
            }
        });
    }
    
    // private static method (재사용 가능한 유틸리티)
    private static <T> T validateNotNull(T obj, String message) {
        if (obj == null) throw new NullPointerException(message);
        return obj;
    }
}

// 올바른 패턴 4: 구현체에서도 private 사용 (Java 9+)
class MyList<E> extends AbstractList<E> {
    private E[] elements;
    
    // 공개 메서드들
    @Override
    public E get(int index) {
        validateIndex(index);
        return elements[index];
    }
    
    @Override
    public void set(int index, E element) {
        validateIndex(index);
        validateNotNull(element);
        elements[index] = element;
    }
    
    // private 메서드로 검증 로직 집중화
    private void validateIndex(int index) {
        if (index < 0 || index >= size()) {
            throw new IndexOutOfBoundsException();
        }
    }
    
    private void validateNotNull(E element) {
        if (element == null) {
            throw new NullPointerException();
        }
    }
}
```

---

## 🔬 내부 동작 원리

### 1. private 인터페이스 메서드의 컴파일

```
자바 코드:
  interface IService {
      default void publicMethod() {
          privateHelper();
      }
      
      private void privateHelper() {
          System.out.println("Only visible to interface");
      }
  }

바이트코드:
  public interface IService {
      public default void publicMethod();
          Code:
            0: aload_0
            1: invokespecial IService.privateHelper ()V
            4: return
      
      private void privateHelper();
          Code:
            0: getstatic java.lang.System.out
            3: ldc "Only visible to interface"
            5: invokevirtual java.io.PrintStream.println
            ...
  }

특징:
  - private 메서드는 바이트코드에 synthetic 플래그 없음 (실제 private)
  - invokespecial로 컴파일 (정적 바인딩)
  - 구현체에서 호출 불가 (컴파일 타임에 체크)
  - 인터페이스 내에서만 접근 가능
```

### 2. private vs private static 메서드

```
private instance method:
  interface DataSource {
      default void readData() {
          validateConnection();  // this 필요
      }
      
      private void validateConnection() {
          // 인스턴스 상태 접근 가능
          if (isClosed()) throw new IllegalStateException();
      }
      
      boolean isClosed();
  }
  
  바이트코드:
    private void validateConnection();
      Code:
        0: aload_0                    // this 로드
        1: invokeinterface IDataSource.isClosed()Z
        6: ...

private static method:
  interface Utilities {
      default void process(String data) {
          String validated = validateInput(data);  // static 호출
          doProcess(validated);
      }
      
      private static String validateInput(String s) {
          // 인스턴스 상태 불필요
          return s == null ? "" : s.trim();
      }
      
      void doProcess(String s);
  }
  
  바이트코드:
    private static java.lang.String validateInput(java.lang.String);
      Code:
        0: aload_0                    // 매개변수 로드 (this 없음)
        1: ifnonnull 10
        4: ldc ""
        6: goto 16
        10: aload_0
        11: invokevirtual java.lang.String.trim()
        ...

선택 기준:
  instance method: 인터페이스 메서드 구현 상태 필요
  static method: 순수 유틸리티 함수 (상태 무관)
```

### 3. 캡슐화 강화

```
Java 8 (default method만 있던 시대):

  interface Collection<E> {
      default void forEach(Consumer<? super E> action) {
          for (E e : this) action.accept(e);  // 반복 로직
      }
      
      default void forEachOrdered(Consumer<? super E> action) {
          for (E e : this) action.accept(e);  // 동일 반복 로직 (중복!)
      }
      
      // 문제: 로직 변경 시 여러 곳 수정 필요
  }

Java 9 (private method 추가):

  interface Collection<E> {
      default void forEach(Consumer<? super E> action) {
          iterateWith(action);
      }
      
      default void forEachOrdered(Consumer<? super E> action) {
          iterateWith(action);  // 같은 로직 호출
      }
      
      // private으로 구현 상세 숨김
      private void iterateWith(Consumer<? super E> action) {
          for (E e : this) action.accept(e);
      }
  }

캡슐화 이점:
  1. 중복 제거 (DRY 원칙)
  2. 구현 상세 내부화 (인터페이스 계약 영향 없음)
  3. 변경 영향도 감소
  4. 버전 호환성 유지 용이
```

### 4. JDK에서의 활용 사례

```
java.util.Collection (Java 9+):

public interface Collection<E> extends Iterable<E> {
    default boolean removeIf(Predicate<? super E> filter) {
        Objects.requireNonNull(filter);
        boolean removed = false;
        final Iterator<E> each = iterator();
        while (each.hasNext()) {
            if (filter.test(each.next())) {
                each.remove();
                removed = true;
            }
        }
        return removed;
    }
    
    // Java 9에서 각 default method의 공통 검증을 private 메서드로
    private void checkNotNull(Object obj) {
        if (obj == null) throw new NullPointerException();
    }
}

java.util.stream.Stream (Java 9+):

public interface Stream<T> extends BaseStream<T, Stream<T>> {
    default void forEach(Consumer<? super T> action) {
        forEachInternal(action);
    }
    
    // 내부 구현 (공개하지 않음)
    private void forEachInternal(Consumer<? super T> action) {
        // 복잡한 스트림 처리 로직
    }
}

java.util.List (Java 9+):

public interface List<E> extends Collection<E> {
    default void replaceAll(UnaryOperator<E> operator) {
        Objects.requireNonNull(operator);
        final ListIterator<E> li = this.listIterator();
        while (li.hasNext()) {
            li.set(operator.apply(li.next()));
        }
    }
    
    default void sort(Comparator<? super E> c) {
        Object[] a = this.toArray();
        Arrays.sort(a, (Comparator) c);
        ListIterator<E> i = this.listIterator();
        for (Object e : a) {
            i.next();
            i.set((E) e);
        }
    }
    
    // 공통 검증은 private으로
    private void validateComparator(Comparator<?> c) {
        // 모든 정렬 관련 default method가 공유
    }
}
```

---

## 💻 실전 실험

### 실험 1: private instance method 캡슐화

```java
interface Logger {
    default void logInfo(String msg) {
        log("INFO", msg);
    }
    
    default void logError(String msg) {
        log("ERROR", msg);
    }
    
    default void logWarning(String msg) {
        log("WARNING", msg);
    }
    
    private void log(String level, String msg) {
        String formatted = formatMessage(level, msg);
        writeToStream(formatted);
    }
    
    private String formatMessage(String level, String msg) {
        return String.format("[%s] %s at %s",
            level, msg, System.currentTimeMillis());
    }
    
    void writeToStream(String msg);
}

class ConsoleLogger implements Logger {
    @Override
    public void writeToStream(String msg) {
        System.out.println(msg);
    }
}

public class LoggerTest {
    public static void main(String[] args) {
        Logger logger = new ConsoleLogger();
        logger.logInfo("Application started");
        logger.logError("Error occurred");
        
        // 출력:
        // [INFO] Application started at 1234567890
        // [ERROR] Error occurred at 1234567891
    }
}
```

### 실험 2: private static method 유틸리티

```java
interface Validator {
    default <T> boolean validate(T obj, Predicate<T> rule) {
        validateNotNull(obj, "Object cannot be null");
        validateNotNull(rule, "Rule cannot be null");
        return rule.test(obj);
    }
    
    default <T> List<T> filterValid(List<T> items, Predicate<T> rule) {
        validateNotNull(items, "Items cannot be null");
        validateNotNull(rule, "Rule cannot be null");
        return items.stream()
                   .filter(rule)
                   .collect(Collectors.toList());
    }
    
    // 공유 유틸리티 (static)
    private static <T> void validateNotNull(T obj, String msg) {
        if (obj == null) throw new NullPointerException(msg);
    }
}

class DataValidator implements Validator {
}

public class ValidatorTest {
    public static void main(String[] args) {
        Validator v = new DataValidator();
        
        boolean valid = v.validate(42, n -> n > 0);
        System.out.println("Valid: " + valid);  // true
        
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
        List<Integer> evens = v.filterValid(numbers, n -> n % 2 == 0);
        System.out.println("Evens: " + evens);  // [2, 4]
    }
}
```

### 실험 3: 구현체 내 private 메서드

```java
class OptimizedList<E> extends AbstractList<E> {
    private Object[] elements;
    private int size;
    
    @Override
    public boolean add(E e) {
        ensureCapacity(size + 1);
        elements[size++] = e;
        return true;
    }
    
    @Override
    public E get(int index) {
        validateIndex(index);
        return (E) elements[index];
    }
    
    @Override
    public int size() { return size; }
    
    // private 메서드 그룹화
    private void ensureCapacity(int minCapacity) {
        if (minCapacity > elements.length) {
            resize(minCapacity * 2);
        }
    }
    
    private void validateIndex(int index) {
        if (index < 0 || index >= size) {
            throw new IndexOutOfBoundsException(index);
        }
    }
    
    private void resize(int newCapacity) {
        Object[] newElements = new Object[newCapacity];
        System.arraycopy(elements, 0, newElements, 0, size);
        elements = newElements;
    }
}
```

---

## 📊 성능/비교

```
private method vs public helper class 성능 비교:

메모리 오버헤드:
  public helper class:
    - 별도 클래스 파일 (~1KB)
    - 메서드 테이블 항목
    - 클래스 메타데이터

  private interface method:
    - 인터페이스 메서드 테이블에 통합
    - 추가 메모리 없음 (인터페이스 메모리에 포함)

호출 성능:
  invokespecial (private method):
    - 정적 바인딩
    - 인라인 가능
    - ~3ns (JIT 후)

  invokestatic (helper class static method):
    - 정적 바인딩
    - 인라인 가능
    - ~3ns (JIT 후)

  성능 차이: 무시할 수 있음 (JIT 최적화 후)

메서드 테이블 크기:
  큰 인터페이스 (50+ default method):
    private helper 방식: 메서드 테이블 크기 증가
    public helper 방식: 별도 클래스 (메타데이터 증가)

일반적으로:
  - default method 많음 + 공통 로직 많음 → private 메서드 가장 효율적
  - 간단한 인터페이스 → 차이 미미
```

---

## ⚖️ 트레이드오프

```
private interface method 도입의 트레이드오프:

장점:
  ✓ 캡슐화 강화 (구현 상세 숨김)
  ✓ 코드 중복 제거 (DRY 원칙)
  ✓ 응집도 향상 (관련 로직 한곳에)
  ✓ 메모리 효율적 (별도 클래스 불필요)
  ✓ 유지보수성 개선

단점:
  ✗ Java 9+ 필수 (하위 호환성)
  ✗ 버전 9 미만에서 사용 불가
  ✗ 초기 학습 곡선 (새로운 개념)

비교: Helper Class vs Private Method

Helper Class 패턴 (Java 8):
  + 명확한 관심사 분리
  + Java 8 이하에서도 사용 가능
  - 공개 API처럼 보임
  - 여러 클래스 파일 관리 필요
  - 버전 호환성 부담

Private Interface Method (Java 9+):
  + 캡슐화 강화
  + 인터페이스와 함께 배포
  - Java 9 이상 필수
  - 인터페이스가 복잡해질 수 있음

선택 기준:
  Java 8 지원 필요 → Helper class
  Java 9+ only → Private interface method
  대규모 라이브러리 → Private interface method (유지보수 용이)
  간단한 인터페이스 → 둘 다 괜찮음
```

---

## 📌 핵심 정리

```
Private Interface Method의 역할:

1. 캡슐화 강화
   - default method 간 공통 로직을 private으로 추상화
   - 구현 상세가 계약에 영향 없음
   - 라이브러리 내부 최적화 가능

2. 두 가지 형태
   private void method() { ... }
     - 인스턴스 상태 접근 필요
     - 구현체 메서드와 협력

   private static void method() { ... }
     - 상태 무관 유틸리티
     - 순수 함수형 로직

3. 중복 제거
   여러 default method의 공통 로직을 한 곳에서 관리
   - 버그 수정 시 한 곳만 수정
   - 로직 변경이 일관되게 적용

4. 응집도 향상
   관련 메서드들이 물리적으로 같은 인터페이스에 위치
   - 메서드들의 의도가 분명
   - 유지보수자 입장에서 탐색 용이

5. JDK 활용
   java.util.Collection, java.util.List, Stream 등에서
   광범위하게 사용 중 (Java 9+)
```

---

## 🤔 생각해볼 문제

**Q1.** `private static` 메서드를 사용해야 할 때와 `private` 인스턴스 메서드를 사용해야 할 때의 기준은 무엇인가?

<details>
<summary>해설 보기</summary>

**private static**: 인스턴스 상태 무관

```java
interface Formatter {
    default String format(String input) {
        return toUpperCase(input);  // 순수 함수
    }
    
    private static String toUpperCase(String s) {
        return s == null ? "" : s.toUpperCase();
    }
}
```

**private 인스턴스 메서드**: 인스턴스 상태 필요

```java
interface DataProcessor {
    default void processData(Data d) {
        if (isConnected()) {  // 상태 체크
            sendToBackend(d);
        }
    }
    
    private void sendToBackend(Data d) {
        // getConnection() 등 상태 기반 로직
    }
    
    boolean isConnected();
}
```

기준:
1. 메서드가 `this`의 추상 메서드를 호출하는가? → 인스턴스 메서드
2. 메서드가 멤버 변수에 접근하는가? → 인스턴스 메서드
3. 메서드가 순수 유틸리티 함수인가? → static 메서드
4. 매개변수만으로 처리 가능한가? → static 메서드

정리:
- **상태 의존적**: 인스턴스 메서드 (`private void`)
- **상태 독립적**: 정적 메서드 (`private static`)

</details>

---

**Q2.** Private interface method를 사용하면 구현체가 이 메서드를 오버라이드할 수 없다는 것이 제약이 아닌가?

<details>
<summary>해설 보기</summary>

맞다, 이는 **의도적인 제약**이다:

```java
interface Collection<E> {
    default void forEach(Consumer<? super E> action) {
        forEachHelper(action);
    }
    
    private void forEachHelper(Consumer<? super E> action) {
        // 구현체가 이를 오버라이드할 수 없음!
    }
}

class MyList<E> extends AbstractList<E> {
    // 아래는 컴파일 에러!
    // public void forEachHelper(...) { }  // 존재하지 않는 메서드
}
```

이것이 정당한 이유:

1. **계약 보호**
   - `forEach()`는 `forEachHelper()`를 호출하기로 약속
   - 구현체가 `forEachHelper()`를 바꾸면 계약 위반

2. **라이브러리 진화**
   ```java
   // Java 9에서 추가
   interface Collection<E> {
       default void forEach(...) {
           forEachHelper(...);
       }
       private void forEachHelper(...) { }
   }
   
   // 기존 구현체들: 영향 없음 (forEachHelper는 내부용)
   // 새로운 구현체: forEachHelper를 오버라이드할 수 없으므로
   //              라이브러리 업데이트의 의도가 자동 보호됨
   ```

3. **성능 최적화**
   - private 메서드는 정적 바인딩 (invokespecial)
   - 구현체 오버라이드 불가 → JIT가 항상 인라인 가능

원한다면:
```java
interface Collection<E> {
    // default가 아닌 public abstract로 남겨두면 구현체가 오버라이드 가능
    void myHelper();
}
```

하지만 이렇게 하면 모든 구현체가 구현해야 함 (번거로움).

**결론**: private 제약은 인터페이스 설계자의 명확한 의도를 나타낸다.

</details>

---

**Q3.** Java 표준 라이브러리에서 private 인터페이스 메서드를 가장 많이 사용하는 클래스는 어디인가?

<details>
<summary>해설 보기</summary>

**`java.util.Collection` 인터페이스**가 가장 광범위하게 사용한다:

```java
// java.util.Collection (Java 9 소스)
public interface Collection<E> extends Iterable<E> {
    
    default void forEach(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        for (E e : this) {
            action.accept(e);
        }
    }
    
    default boolean removeIf(Predicate<? super E> filter) {
        Objects.requireNonNull(filter);
        boolean removed = false;
        final Iterator<E> each = iterator();
        while (each.hasNext()) {
            if (filter.test(each.next())) {
                each.remove();
                removed = true;
            }
        }
        return removed;
    }
    
    // Java 9에서 공통 null 체크를 private으로 추출 (구상)
    private void checkNotNull(Object obj) {
        if (obj == null) throw new NullPointerException();
    }
}
```

다른 주요 사용처:

1. **java.util.List**
   - `replaceAll()`, `sort()`, `subList()` 등의 default method
   - 공통 반복 로직을 private으로 통합

2. **java.util.stream.Stream**
   - `forEach()`, `forEachOrdered()` 등 여러 터미널 연산
   - 공통 스트림 처리 로직을 private으로

3. **java.util.Map**
   - `forEach()`, `replaceAll()` 등의 default method
   - Entry 반복 로직 재사용

**이점**:
- Java 9 이전에는 각 구현체(`ArrayList`, `HashMap` 등)가 동일 로직 중복
- Java 9+: 인터페이스에서 정의, 모든 구현체가 일관되게 사용
- 버그 수정이 한 곳 (인터페이스)에서만 필요

</details>

---

<div align="center">

**[⬅️ 이전: 다이아몬드 상속 해결 규칙](./02-diamond-inheritance-rules.md)** | **[홈으로 🏠](../README.md)** | **[다음: Static Interface Method ➡️](./04-static-interface-method.md)**

</div>
