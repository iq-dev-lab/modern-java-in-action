# Static Interface Method와 Helper Method 패턴

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Java 8 이전에는 왜 `Collections.unmodifiableList()` 같은 유틸리티가 분리된 클래스에 있었는가?
- `List.of()`, `Optional.empty()`처럼 static factory method를 인터페이스에 정의한 이유는?
- 인터페이스의 static 메서드는 왜 상속되지 않는가?
- Static interface method vs static helper class의 응집도 차이는?
- JDK 컬렉션 API의 진화에서 정적 팩토리 메서드의 역할은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Java 8 이전의 컬렉션 API는 인터페이스와 유틸리티 클래스로 나뉘었다: `List` 인터페이스와 `Collections` 헬퍼 클래스. Java 8에서 `List.of()`, `Map.entry()` 같은 정적 팩토리 메서드를 인터페이스에 직접 정의할 수 있게 되면서, API의 응집도와 발견성(discoverability)이 크게 개선되었다. 이를 모르면 여전히 분리된 헬퍼 클래스 패턴을 따르거나, 인터페이스 기반의 깔끔한 API를 설계할 기회를 놓친다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 인터페이스의 static 메서드가 상속된다고 착각
  interface Creatable {
      static Creatable create() {
          return new CreateableImpl();
      }
  }
  
  class Subclass implements Creatable { }
  
  Subclass obj = Subclass.create();  // 컴파일 에러!
  // static 메서드는 상속 안 됨
  
  // 올바른 호출:
  Creatable obj = Creatable.create();

실수 2: Static factory method를 구현체에서 오버라이드 시도
  interface Factory {
      static Factory getInstance() {
          return new FactoryImpl();
      }
  }
  
  class FactoryImpl implements Factory {
      public static Factory getInstance() {  // 컴파일 에러!
          // 인터페이스의 static 메서드를 오버라이드할 수 없음
          return new FactoryImpl();
      }
  }

실수 3: Helper 클래스 vs 인터페이스 static 메서드 혼용
  // Java 8 이전 스타일 (여전히 사용)
  public class ListHelper {
      public static List<?> empty() { return Collections.emptyList(); }
      public static List<?> singleton(Object e) { return Collections.singletonList(e); }
  }
  
  // Java 8+ 스타일 (인터페이스에 static)
  interface List {
      static <E> List<E> of(E... elements) { ... }
  }
  
  // 혼용하면 API 일관성 저하
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 올바른 패턴 1: Static factory method를 인터페이스에
interface Shape {
    // 추상 메서드
    double getArea();
    
    // Static factory method (Java 8+)
    static Shape circle(double radius) {
        return new Circle(radius);
    }
    
    static Shape rectangle(double width, double height) {
        return new Rectangle(width, height);
    }
    
    static Shape square(double side) {
        return rectangle(side, side);  // 위임
    }
    
    // 구현 클래스들
    class Circle implements Shape {
        private double radius;
        Circle(double radius) { this.radius = radius; }
        @Override public double getArea() { return Math.PI * radius * radius; }
    }
    
    class Rectangle implements Shape {
        private double width, height;
        Rectangle(double w, double h) { this.width = w; this.height = h; }
        @Override public double getArea() { return width * height; }
    }
}

// 올바른 패턴 2: 불변 컬렉션 생성
interface List<E> {
    static <E> List<E> of() { return new UnmodifiableList<>(); }
    static <E> List<E> of(E e1) { return new UnmodifiableList<>(e1); }
    static <E> List<E> of(E e1, E e2) { return new UnmodifiableList<>(e1, e2); }
    // ... varargs 오버로딩
    static <E> List<E> of(E... elements) { return new UnmodifiableList<>(elements); }
    
    // 필터링하여 새 리스트 반환
    default <E> List<E> filter(Predicate<? super E> pred) {
        List<E> result = List.of();  // static factory 사용
        for (E e : (List<E>) this) {
            if (pred.test(e)) result.add(e);
        }
        return result;
    }
}

// 올바른 패턴 3: 동작 정의 (Functional Interface의 정적 팩토리)
@FunctionalInterface
interface Converter<I, O> {
    O convert(I input);
    
    // Static composition 메서드
    static <A, B, C> Converter<A, C> compose(
        Converter<A, B> first,
        Converter<B, C> second
    ) {
        return a -> second.convert(first.convert(a));
    }
    
    // Static identity
    static <T> Converter<T, T> identity() {
        return t -> t;
    }
    
    // 사용
    static void example() {
        Converter<String, Integer> stringToInt = Integer::parseInt;
        Converter<Integer, String> intToString = String::valueOf;
        Converter<String, String> composed = compose(stringToInt, intToString);
    }
}

// 올바른 패턴 4: Map.Entry와 같은 간단한 객체 생성
interface Map<K, V> {
    interface Entry<K, V> {
        K getKey();
        V getValue();
        
        // Static factory
        static <K, V> Entry<K, V> entry(K k, V v) {
            return new SimpleEntry<>(k, v);
        }
        
        class SimpleEntry<K, V> implements Entry<K, V> {
            private K key;
            private V value;
            SimpleEntry(K k, V v) { this.key = k; this.value = v; }
            @Override public K getKey() { return key; }
            @Override public V getValue() { return value; }
        }
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Static method는 상속되지 않는 이유

```
자바 코드:
  interface Animal {
      static void sound() {
          System.out.println("Generic sound");
      }
  }
  
  class Dog implements Animal { }
  
  Dog.sound();  // 컴파일 에러!
  Animal.sound();  // OK

이유:

1. Static binding (정적 바인딩)
   static 메서드는 런타임에 다형성 불가능
   → 컴파일 타임에 타입으로 결정
   
   class Dog implements Animal {
       static void sound() { print("Woof"); }  // 다른 메서드
   }
   
   Animal a = new Dog();
   a.sound();  // 어느 것을 호출? (ambiguous)
   // 따라서 Java는 static 메서드 상속 금지

2. 메서드 테이블 처리
   Virtual method (인스턴스):
     - 메서드 테이블에 포함 (다형성 지원)
     - 자식 클래스가 오버라이드 가능
   
   Static method:
     - 메서드 테이블 없음 (정적 바인딩)
     - invokestatic으로 직접 호출
     - 자식에서 "숨김" 불가 (다른 정적 메서드)

3. 설계 의도
   static method는 인터페이스 계약의 일부가 아님
   → 상속 대상이 아님 (구현 상세)

바이트코드 비교:

Virtual method (interface default):
  0: aload_0
  1: invokevirtual Animal.sound()V  // 동적 디스패치
  4: return

Static method:
  0: invokestatic Animal.sound()V   // 정적 바인딩
  3: return
  (receiver 없음 - this를 전달하지 않음)
```

### 2. Java 8 이전: Collections Helper Class

```
Java 8 이전 설계:

interface Collection<E> {
    // 추상 메서드들만
    boolean add(E e);
    boolean remove(Object o);
    int size();
    // ...
}

public final class Collections {
    // static helper method들
    public static <T extends Comparable<? super T>> void sort(List<T> list) { }
    
    public static <T> List<T> unmodifiableList(List<? extends T> list) {
        return new UnmodifiableList<>(list);
    }
    
    public static <T> List<T> synchronizedList(List<T> list) {
        return new SynchronizedList<>(list);
    }
    
    public static final List EMPTY_LIST = new UnmodifiableList<>(new ArrayList());
    public static <T> List<T> emptyList() {
        return (List<T>) EMPTY_LIST;
    }
}

// 사용자 입장: 이게 Collection과 관련된 메서드인지 어떻게 알지?
List<Integer> empty = Collections.emptyList();
List<String> unmod = Collections.unmodifiableList(list);
```

### 3. Java 8+: Static Method in Interface

```
Java 8+ 설계:

interface List<E> extends Collection<E> {
    // 추상 메서드
    E get(int index);
    E set(int index, E element);
    
    // default method
    default boolean removeIf(Predicate<? super E> filter) { ... }
    
    // Static factory method (새로움!)
    @SafeVarargs
    static <E> List<E> of(E... elements) {
        return new UnmodifiableList<>(elements);
    }
    
    static <E> List<E> copyOf(Collection<? extends E> coll) {
        return new UnmodifiableList<>(coll);
    }
}

// 사용자 입장: List의 정적 메서드 → 발견성 개선
List<Integer> empty = List.of();
List<String> list = List.of("a", "b", "c");
List<String> copy = List.copyOf(existing);

API 응집도:
  - List와 관련된 모든 메서드가 List에 있음
  - IDE 자동완성도 효과적 ("List."를 입력하면 모든 옵션 보임)
```

### 4. Static Factory Method의 장점

```java
// 예 1: 명확한 의도 표현
class Configuration {
    private Config data;
    
    // 이름 있는 생성자 (불가능)
    // public Configuration.withDefaults() { }
    
    // 대신 static factory
    static Configuration withDefaults() {
        Configuration c = new Configuration();
        c.setDefaultValues();
        return c;
    }
    
    static Configuration empty() {
        return new Configuration();
    }
}

// 사용자: Configuration.withDefaults() 의도가 명확

// 예 2: 인스턴스 재사용 (불변 객체)
interface Number {
    static Number ZERO = new Number(0);
    static Number ONE = new Number(1);
    static Number TWO = new Number(2);
    
    static Number of(int value) {
        return switch(value) {
            case 0 -> ZERO;
            case 1 -> ONE;
            case 2 -> TWO;
            default -> new Number(value);
        };
    }
}

// 사용자가 모르는 사이에 인스턴스 풀 구현

// 예 3: 서브타입 반환
interface Shape {
    static Shape circle(double radius) {
        return new Circle(radius);  // Shape의 구현체
    }
    
    static Shape rectangle(double w, double h) {
        return new Rectangle(w, h);  // 다른 구현체
    }
}

// 사용자는 Shape만 알면 됨 (Circle, Rectangle은 비공개 가능)
```

### 5. Static vs Default Method의 호출 메커니즘

```
바이트코드 차이:

Default method (invokeinterface):
  interface Service {
      default void process() { }
  }
  
  Service s = new ServiceImpl();
  s.process();  // invokeinterface Service.process()V
                // 런타임 디스패치 (다형성)

Static method (invokestatic):
  interface Service {
      static void process() { }
  }
  
  Service.process();  // invokestatic Service.process()V
                      // 정적 바인딩 (다형성 불가)

호출 시점:
  invokeinterface:
    1. 런타임에 Service.process()의 메서드 테이블 인덱스 조회
    2. 실제 인스턴스의 메서드 테이블에서 해당 슬롯 확인
    3. 구현체의 메서드 호출
  
  invokestatic:
    1. 컴파일 시 정확한 메서드 결정
    2. 런타임에 Service.process()를 직접 호출
    3. 해석 불필요 (이미 결정됨)
```

---

## 💻 실전 실험

### 실험 1: Static Factory Method 구현

```java
interface Logger {
    void log(String message);
    
    // Static factory method
    static Logger console() {
        return message -> System.out.println("[CONSOLE] " + message);
    }
    
    static Logger file(String filename) {
        return message -> {
            try (FileWriter fw = new FileWriter(filename, true)) {
                fw.write("[FILE] " + message + "\n");
            } catch (IOException e) { e.printStackTrace(); }
        };
    }
    
    static Logger composite(Logger... loggers) {
        return message -> {
            for (Logger l : loggers) {
                l.log(message);
            }
        };
    }
}

public class LoggerTest {
    public static void main(String[] args) {
        Logger console = Logger.console();
        Logger file = Logger.file("/tmp/app.log");
        Logger combined = Logger.composite(console, file);
        
        combined.log("Application started");
        
        // 복합 로거가 두 곳 모두에 기록
    }
}
```

### 실험 2: Static Method는 상속되지 않음 확인

```java
interface Base {
    static void staticMethod() {
        System.out.println("Base static");
    }
    
    default void instanceMethod() {
        System.out.println("Base instance");
    }
}

class Child implements Base {
    // static 메서드는 "숨김" (재정의)
    static void staticMethod() {
        System.out.println("Child static");
    }
    
    @Override  // instance는 오버라이드 가능
    public void instanceMethod() {
        System.out.println("Child instance");
    }
}

public class InheritanceTest {
    public static void main(String[] args) {
        Base.staticMethod();      // Base static
        Child.staticMethod();     // Child static (다른 메서드)
        
        Base base = new Child();
        base.staticMethod();      // Base static (Base 참조 타입)
        base.instanceMethod();    // Child instance (다형성)
        
        Child child = new Child();
        child.staticMethod();     // Child static
        child.instanceMethod();   // Child instance
    }
}
```

### 실험 3: JDK 컬렉션 Static Factory

```java
import java.util.*;

public class StaticFactoryTest {
    public static void main(String[] args) {
        // Java 9+ List.of() - static factory
        List<String> list = List.of("a", "b", "c");
        System.out.println(list);  // [a, b, c]
        
        // 불변 리스트
        try {
            list.add("d");  // UnsupportedOperationException
        } catch (UnsupportedOperationException e) {
            System.out.println("List.of() returns immutable list");
        }
        
        // Map.of() - static factory
        Map<String, Integer> map = Map.of(
            "one", 1,
            "two", 2,
            "three", 3
        );
        System.out.println(map);
        
        // Set.of() - static factory
        Set<String> set = Set.of("a", "b", "c");
        System.out.println(set);
        
        // 비교: 이전 스타일 (Collections helper)
        List<String> oldStyle = Collections.unmodifiableList(
            Arrays.asList("x", "y", "z")
        );
        System.out.println(oldStyle);
    }
}
```

---

## 📊 성능/비교

```
Static factory vs Constructor 성능:

메모리 오버헤드:
  Constructor:
    - new 키워드로 매번 새 인스턴스 생성
    - 힙에 객체 할당

  Static factory:
    - 인스턴스 재사용 가능 (불변 객체)
    - Flyweight 패턴 구현 가능
    
    예: Integer.valueOf(5)는 -128~127 범위 캐싱
    Integer i1 = Integer.valueOf(5);
    Integer i2 = Integer.valueOf(5);
    i1 == i2  // true (동일 객체)

호출 성능:
  Constructor (invokenew + invokespecial):
    - 객체 할당: ~50ns
    - 초기화: ~10ns

  Static factory (invokestatic):
    - 메서드 호출: ~20ns
    - 캐시 히트 시: ~5ns (인스턴스 재사용)

결과: Static factory가 빠를 수 있음 (캐싱 통해)

메서드 호출 성능:
  invokestatic (static method):
    - 정적 바인딩
    - 항상 인라인 가능
    - ~3ns (JIT 후)

  invokevirtual (instance method):
    - 다형성
    - JIT 인라인 가능 (단형성)
    - ~3ns (JIT 후)

  성능 차이: 무시할 수 있음

일반적으로:
  - API 설계 관점에서 static factory 선호
  - 성능은 JIT 최적화가 담당
  - 메모리는 불변 객체 재사용으로 이득
```

---

## ⚖️ 트레이드오프

```
Static Factory Method의 트레이드오프:

장점:
  ✓ API 발견성 개선 (IDE 자동완성)
  ✓ 응집도 향상 (관련 메서드가 같은 곳에)
  ✓ 명확한 의도 표현 (메서드 이름)
  ✓ 인스턴스 캐싱 가능 (성능)
  ✓ 서브타입 반환 가능 (구현 숨김)
  ✓ 호출 시점에 구현 결정 가능 (유연성)

단점:
  ✗ 생성자 문법 사용 불가 (관례 깨짐)
  ✗ Reflection에서 찾기 어려움
  ✗ 상속되지 않음 (명시적 호출 필요)
  ✗ 이름 선택의 부담 (of? newInstance? create?)

비교: Constructor vs Static Factory

Constructor 방식:
  List<String> list = new ArrayList<>(Arrays.asList("a", "b"));
  + 언어 표준
  - 의도 불명확 (새 ArrayList?)
  - 구현체 노출

Static factory 방식:
  List<String> list = List.of("a", "b");
  + 명확한 의도
  + 구현체 숨김
  - 언어 표준이 아님
  - 생성자 관례 깨짐

타협: 둘 다 제공
  class MyClass {
      // Constructor (표준)
      public MyClass(String value) { ... }
      
      // Static factory (편의)
      public static MyClass of(String value) {
          return new MyClass(value);
      }
      
      public static MyClass empty() {
          return new MyClass("");
      }
  }
```

---

## 📌 핵심 정리

```
Static Interface Method의 역할:

1. 진화 역사
   Java 7: Helper 클래스 필수
     Collections.emptyList()
     Collections.unmodifiableList()
   
   Java 8+: 인터페이스에 static 메서드
     List.of()
     Map.entry()

2. 특징
   - 상속 불가 (정적 바인딩)
   - invokestatic으로 컴파일
   - 오버라이드 불가 (메서드 테이블 없음)
   - 다형성 불가능 (정적 해석)

3. 사용 목적
   - Static factory method (생성)
   - Utility function (변환, 검증)
   - Constant 정의 (final static)
   - Composition (함수형 조합)

4. API 설계
   응집도: Helper class << Static interface method
   발견성: Helper class << Static interface method
   일관성: Helper class << Static interface method

5. 성능
   - 인스턴스 재사용 가능 (메모리)
   - 정적 바인딩 (캐싱 용이)
   - JIT 인라인 (호출 비용)

6. JDK 활용
   java.util.List, Set, Map
   java.util.Collections (여전히 사용)
   java.util.Optional
   java.time.* (LocalDate.of() 등)
```

---

## 🤔 생각해볼 문제

**Q1.** 왜 `static` 메서드는 인터페이스에서 상속되지 않는데, `default` 메서드는 상속되는가?

<details>
<summary>해설 보기</summary>

**다형성의 가능성** 때문이다:

```java
// default method (상속됨)
interface Animal {
    default void sound() { print("Generic"); }
}

class Dog implements Animal {
    @Override
    public void sound() { print("Woof"); }
}

Animal a = new Dog();
a.sound();  // "Woof" (다형성)
```

반대로:

```java
// static method (상속 안 됨)
interface Animal {
    static void sleep() { print("Zzz"); }
}

class Dog implements Animal { }

Dog.sleep();  // 컴파일 에러!
Animal.sleep();  // OK

Dog dog = new Dog();
dog.sleep();  // 컴파일 에러! (인스턴스로 호출 불가)
```

왜?

1. **다형성 불가능**
   - static method는 런타임 다형성 불가능
   - 타입이 정확히 결정되어야 호출 가능
   - 상속되면 어느 것을 호출할지 불명확

2. **메서드 테이블의 부재**
   - default method: vtable에 슬롯 할당 (오버라이드 가능)
   - static method: 정적 바인딩 (메서드 테이블 없음)
   - 상속할 것이 없음

3. **의도의 명확성**
   - default: 계약의 일부 (구현체가 해석)
   - static: 유틸리티 (구현 상세, 계약 아님)

결론: Static method는 "선택적 상속" 개념 불가능. 명시적으로 호출해야 한다.

</details>

---

**Q2.** `List.of()`가 항상 새로운 인스턴스를 반환하지 않는데, 이것을 사용자가 어떻게 알 수 있는가?

<details>
<summary>해설 보기</summary>

**Javadoc과 관례**다:

```java
public interface List<E> extends Collection<E> {
    /**
     * Returns an <a href="#unmodifiable">unmodifiable list</a> containing...
     * The list is <i>immutable</i>.
     *
     * @param elements the elements to be contained in the list
     * @return an unmodifiable list containing the elements
     * ...
     */
    @SafeVarargs
    static <E> List<E> of(E... elements) {
        // 구현: 캐싱 여부는 내부 결정
    }
}
```

사용자는 **명세만 믿음**:
- "unmodifiable list 반환"
- "immutable"
- 인스턴스 재사용 여부는 구현 상세 (명세에 없음)

실제 구현 (Java 9+):
```java
// 0개 요소
if (elements.length == 0) {
    return EMPTY_LIST;  // 캐시된 인스턴스
}
// 1개 요소
if (elements.length == 1) {
    return new List1<>(elements[0]);  // 전용 클래스
}
// 2개 이상
return new ListN<>(elements);  // 공용 클래스
```

이것이 가능한 이유:
- `List.of()`는 불변 리스트 **명세**만 제시
- "몇 개의 인스턴스 타입을 사용하는가"는 구현 상세
- 사용자는 오직 List 인터페이스만 의존

따라서:
```java
List<String> empty1 = List.of();
List<String> empty2 = List.of();
empty1 == empty2  // true (동일 캐시 인스턴스)

// 하지만 사용자는 이것을 알 필요 없음
// List 인터페이스만 사용하면 됨
```

**핵심**: Static factory method를 통해 구현을 완전히 숨기고, 명세(계약)만 노출. 이것이 최대의 자유도를 제공한다.

</details>

---

**Q3.** Java 컬렉션 API에서 여전히 `Collections` 헬퍼 클래스를 사용하는 이유는?

<details>
<summary>해설 보기</summary>

**하위 호환성과 추가 기능**:

```java
// Collections는 여전히 광범위하게 사용됨

// 1. 기존 코드와의 호환성
List<String> list = new ArrayList<>();
List<String> unmodifiable = Collections.unmodifiableList(list);

// 2. List.of()와의 차이
// Collections.unmodifiableList: 기존 리스트를 감싸기 (변경 감지)
List<String> original = new ArrayList<>(Arrays.asList("a", "b"));
List<String> view = Collections.unmodifiableList(original);
original.add("c");
view.size();  // 3 (변경이 반영됨!)

// List.of: 새로운 불변 리스트 (독립적)
List<String> immutable = List.of("a", "b");
// original을 수정해도 immutable은 영향 없음

// 3. Synchronized collection
Map<String, String> map = Collections.synchronizedMap(new HashMap<>());
// thread-safe wrapper (성능 관점에서 ConcurrentHashMap과 다름)

// 4. 역순 반복자, 이진 검색 등
Collections.reverse(list);
Collections.binarySearch(list, "target");
Collections.shuffle(list);
Collections.sort(list);  // (List.sort()와 유사)
```

API 설계의 진화:

Java 1.2 (1998):
  ```java
  List list = Collections.unmodifiableList(new ArrayList());
  ```

Java 5-7:
  ```java
  List<String> list = Collections.unmodifiableList(new ArrayList<>());
  ```

Java 8-11:
  ```java
  // List.of() 추가하지만 Collections 여전히 사용
  List<String> list = List.of("a", "b");
  List<String> wrapped = Collections.unmodifiableList(existing);
  ```

Java 16+:
  ```java
  // 완전한 전환은 아직 (호환성)
  // Collections의 일부 메서드는 deprecated 안 됨
  ```

결론:
- **List.of()**: 새로운 불변 리스트 생성
- **Collections.unmodifiableList()**: 기존 리스트를 읽기 전용으로 감싸기
- 둘 다 서로 다른 사용 사례 지원
- Collections는 100% 대체되지 않음 (호환성)

</details>

---

<div align="center">

**[⬅️ 이전: Private Interface Method](./03-private-interface-method.md)** | **[홈으로 🏠](../README.md)** | **[다음: Sealed Interface ➡️](./05-sealed-interface.md)**

</div>
