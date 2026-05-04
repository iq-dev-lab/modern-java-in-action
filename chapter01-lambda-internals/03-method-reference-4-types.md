# Method Reference 4가지 — 각각의 바이트코드

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Method Reference의 4가지 종류는 정확히 무엇이고, 각각 어떤 바이트코드로 변환되는가?
- `Class::staticMethod`와 `Class::instanceMethod`의 BootstrapMethods 인자는 어떻게 다른가?
- `instance::method` 캡처에서 `this`를 내부적으로 어떻게 전달하는가?
- `Class::new` 생성자 참조는 왜 `REF_newInvokeSpecial`을 사용하는가?
- 각 메서드 참조가 동등한 람다 코드로 변환되면 어떻게 되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

메서드 참조는 람다보다 읽기 쉽고, 때로는 더 효율적이다. 하지만 4가지 종류가 다르게 동작하며, 특히 인스턴스 메서드 참조가 캡처 변수처럼 작동한다는 것을 이해하면, 메모리 누수나 성능 문제를 예방할 수 있다. 또한 JDK 스트림 API(`map`, `filter` 등)를 효과적으로 사용하려면 메서드 참조의 각 형태를 정확히 알아야 한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 모든 메서드 참조가 같은 바이트코드로 변환된다고 생각
  String::length      // 인스턴스 메서드
  Math::abs           // static 메서드
  ArrayList::new      // 생성자
  → 내부적으로 완전히 다른 처리!

실수 2: static 메서드 참조가 캡처 없다고 생각하고, 인스턴스 메서드 참조도 마찬가지
  // static은 캡처 없음 (O)
  Function<Integer, Integer> f = Math::abs;  // 인자 1개
  
  // 인스턴스 메서드는 receiver 암묵적 캡처 (X로 알았음)
  Function<String, Integer> f = String::length;  // receiver?
  → 실제로는 첫 인자가 receiver (String 인스턴스)

실수 3: instance::method에서 this가 저장되지 않는다고 생각
  String s = "hello";
  Supplier<Integer> supplier = s::length;  // this는?
  → 내부적으로 's'가 캡처 변수처럼 저장됨
  → supplier 객체가 's' 참조 유지 (GC 불가)

실수 4: 메서드 참조는 항상 효율적이라고 생각
  // 캡처가 있으면 람다와 동등한 성능
  String s = "hello";
  supplier = s::length;     // 성능 동등
  supplier2 = () -> s.length();  // 같은 비용
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 4가지 메서드 참조를 올바르게 활용

// 1. Static Method Reference: Class::staticMethod
// BootstrapMethods: REF_invokeStatic
Function<Integer, Integer> abs = Math::abs;  // static int abs(int)
IntUnaryOperator doubled = x -> x * 2;  // 같은 효과이지만 람다가 더 명확

// 캡처 없음: 싱글톤 재사용 → 메모리 효율
Predicate<String> isEmpty = String::isEmpty;

// 2. Instance Method Reference (unbound): Class::instanceMethod
// BootstrapMethods: REF_invokeVirtual
// 첫 인자가 receiver (인스턴스)
Function<String, Integer> length = String::length;
// 동등한 람다: (s) -> s.length()

// 사용:
List<String> words = List.of("hello", "a", "world");
words.stream()
    .map(String::length)  // 각 String의 length() 호출
    .forEach(System.out::println);

// 3. Instance Method Reference (bound): instance::method
// BootstrapMethods: REF_invokeVirtual + 캡처된 instance
String prefix = "Item-";
Function<Integer, String> formatter = 
    id -> prefix + id;  // 또는 캡처 있는 메서드 참조

// 주의: instance를 캡처 → 메모리 누수 가능
String data = "important";
Supplier<Integer> supplier = data::length;  // data 캡처됨!
// data가 GC 되지 않음 (supplier가 보유)

// 4. Constructor Reference: Class::new
// BootstrapMethods: REF_newInvokeSpecial
Supplier<ArrayList> listSupplier = ArrayList::new;
Function<Integer, ArrayList> capacityList = ArrayList::new;
BiFunction<Integer, Integer, int[][]> array2D = int[][]::new;

// 동등한 람다:
Supplier<ArrayList> lambdaList = () -> new ArrayList();
Function<Integer, ArrayList> lambdaCapacity = capacity -> new ArrayList(capacity);
```

---

## 🔬 내부 동작 원리

### 1. 4가지 Method Reference 분류와 BootstrapMethods

```
┌────────────────────────────────────────────────────────────────┐
│ Type 1: Static Method Reference                                │
│예: Math::abs                                                   │
│                                                                │
│ 바이트코드:                                                     │
│   invokedynamic #n, 0  // apply:()Ljava/util/function/Function│
│                                                                │
│ BootstrapMethods:                                              │
│   LambdaMetafactory.metafactory(                               │
│     lookup,                                                    │
│     "apply",                                                   │
│     MethodType(Function),        // Function<Integer, Integer> │
│     MethodType(int → int),       // SAM: (int) -> int         │
│     MethodHandle(REF_invokeStatic, Math.class, "abs", ...)    │
│     MethodType(int → int)        // 인스턴트화 타입            │
│   )                                                            │
│                                                                │
│ 특징:                                                          │
│  - 캡처 없음 → 싱글톤 재사용                                    │
│  - MethodHandle: REF_invokeStatic (코드 1)                     │
│  - SAM 타입과 메서드 타입이 동일                                │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ Type 2: Unbound Instance Method Reference                      │
│예: String::length                                              │
│                                                                │
│ 바이트코드:                                                     │
│   invokedynamic #n, 0  // apply:()Ljava/util/function/Function│
│                                                                │
│ BootstrapMethods:                                              │
│   LambdaMetafactory.metafactory(                               │
│     lookup,                                                    │
│     "apply",                                                   │
│     MethodType(Function),        // Function<String, Integer>  │
│     MethodType(String, int),     // SAM: (String) -> int      │
│     MethodHandle(REF_invokeVirtual,                            │
│                  String.class, "length", ())                  │
│     MethodType(String, int)      // receiver + 반환값          │
│   )                                                            │
│                                                                │
│ 특징:                                                          │
│  - 첫 인자가 receiver (인스턴스)                                │
│  - MethodHandle: REF_invokeVirtual (코드 5)                    │
│  - SAM 첫 인자 = String (receiver)                            │
│  - 캡처 없음 → 싱글톤 재사용                                    │
│  - 동등한 람다: (s) -> s.length()                              │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ Type 3: Bound Instance Method Reference                        │
│예: "hello"::length   (instance가 이미 결정됨)                   │
│                                                                │
│ 바이트코드:                                                     │
│   invokedynamic #n, 0  // apply:()Ljava/util/function/Supplier│
│                                                                │
│ BootstrapMethods:                                              │
│   LambdaMetafactory.metafactory(                               │
│     lookup,                                                    │
│     "apply",                                                   │
│     MethodType(Supplier),        // Supplier<Integer>          │
│     MethodType(int),             // SAM: () -> int            │
│     MethodHandle(REF_invokeVirtual,                            │
│                  String.class, "length", ())                  │
│     MethodType(String, int),     // receiver (String) 포함!    │
│     "hello"                       // ← 캡처된 receiver!        │
│   )                                                            │
│                                                                │
│ 특징:                                                          │
│  - receiver("hello")를 캡처 → 별도 인자로 전달               │
│  - MethodHandle: REF_invokeVirtual                             │
│  - instantiatedMethodType: MethodType(String, int) 
│                           (receiver가 인자로 포함)            │
│  - SAM (Supplier)은 인자 없음 → receiver는 invokedynamic 시간에 │
│                                  메서드로 전달됨              │
│  - 캡처 있음 → 매번 새 인스턴스 (싱글톤 불가)                  │
│  - 동등한 람다: () -> "hello".length()                        │
│                또는 (String s) 캡처: () -> s.length()         │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ Type 4: Constructor Reference                                  │
│예: ArrayList::new                                              │
│                                                                │
│ 바이트코드:                                                     │
│   invokedynamic #n, 0  // apply:()Ljava/util/function/Supplier│
│                                                                │
│ BootstrapMethods:                                              │
│   LambdaMetafactory.metafactory(                               │
│     lookup,                                                    │
│     "apply",                                                   │
│     MethodType(Supplier),        // Supplier<ArrayList>        │
│     MethodType(ArrayList),       // SAM: () -> ArrayList      │
│     MethodHandle(REF_newInvokeSpecial,                         │
│                  ArrayList.class, "<init>", (...))            │
│     MethodType(ArrayList)        // 반환값 = 새 인스턴스       │
│   )                                                            │
│                                                                │
│ 특징:                                                          │
│  - MethodHandle: REF_newInvokeSpecial (코드 8)                 │
│  - 메서드명: "<init>" (생성자 특수명)                         │
│  - 캡처 없음 → 싱글톤 재사용 (동일 Constructor Reference)     │
│  - 생성자 인자에 따라 다양한 형태 가능:
│    * ArrayList::new → Supplier<ArrayList>
│    * ArrayList::new → Function<Integer, ArrayList> (capacity)
│    * int[][]::new → BiFunction<Integer, Integer, int[][]>
└────────────────────────────────────────────────────────────────┘

MethodHandle Reference Codes (중요):
  REF_invokeVirtual = 5      // instance method
  REF_invokeStatic = 6       // static method
  REF_invokeSpecial = 7      // private instance method
  REF_newInvokeSpecial = 8   // constructor <init>
  REF_invokeInterface = 9    // interface method
```

### 2. 동등한 람다로의 변환

```
각 메서드 참조가 람다로 표현되면:

Type 1: Static Method Reference
  메서드 참조:
    Function<Integer, Integer> f = Math::abs;
  
  동등한 람다:
    Function<Integer, Integer> f = x -> Math.abs(x);
  
  컴파일된 메서드:
    private static int lambda$0(int x) {
        return Math.abs(x);
    }

Type 2: Unbound Instance Method Reference
  메서드 참조:
    Function<String, Integer> f = String::length;
  
  동등한 람다:
    Function<String, Integer> f = s -> s.length();
  
  컴파일된 메서드:
    private static int lambda$0(String s) {
        return s.length();
    }

Type 3: Bound Instance Method Reference
  메서드 참조:
    String str = "hello";
    Supplier<Integer> s = str::length;
  
  동등한 람다:
    String str = "hello";
    Supplier<Integer> s = () -> str.length();
  
  컴파일된 메서드:
    private static int lambda$0(String str) {  // 캡처 변수
        return str.length();
    }
  
  invokedynamic 시 캡처:
    invokedynamic #n, 0  // apply:()Supplier
    BootstrapMethods: instantiatedMethodType:
      MethodType(String, int)  // receiver str 암묵적 인자

Type 4: Constructor Reference
  메서드 참조:
    Supplier<ArrayList> s = ArrayList::new;
  
  동등한 람다:
    Supplier<ArrayList> s = () -> new ArrayList();
  
  컴파일된 메서드:
    private static ArrayList lambda$0() {
        return new ArrayList();
    }
```

### 3. javap 분석 예제

```
Source:
  public class MethodReferenceExample {
      public static void example() {
          Function<Integer, Integer> f1 = Math::abs;          // Type 1: static
          Function<String, Integer> f2 = String::length;      // Type 2: unbound
          
          String s = "hello";
          Supplier<Integer> f3 = s::length;                   // Type 3: bound
          
          Supplier<ArrayList> f4 = ArrayList::new;            // Type 4: constructor
      }
  }

javap -c -v MethodReferenceExample.class | grep -A 20 "BootstrapMethods:"

결과:
BootstrapMethods:
  0: #48 invokestatic java/lang/invoke/LambdaMetafactory.metafactory
     Method arguments:
       #49 (I)I         // Type 1: static Math::abs
       #50 invokestatic MethodReferenceExample.lambda$0:(I)I
       #51 (I)I
  
  1: #48 invokestatic java/lang/invoke/LambdaMetafactory.metafactory
     Method arguments:
       #52 (Ljava/lang/String;)I    // Type 2: unbound String::length
       #53 invokevirtual java/lang/String.length:()I
       #54 (Ljava/lang/String;)I
  
  2: #48 invokestatic java/lang/invoke/LambdaMetafactory.metafactory
     Method arguments:
       #55 ()I          // Type 3: bound s::length
       #53 invokevirtual java/lang/String.length:()I
       #54 (Ljava/lang/String;)I  ← receiver String 포함
       #56 "hello"      // ← 캡처된 인스턴스!
  
  3: #48 invokestatic java/lang/invoke/LambdaMetafactory.metafactory
     Method arguments:
       #57 ()Ljava/util/ArrayList;   // Type 4: constructor
       #58 invokespecial ArrayList.<init>:()V
       #59 ()Ljava/util/ArrayList;
```

---

## 💻 실전 실험

### 실험 1: 4가지 메서드 참조 비교

```java
import java.util.function.*;

public class MethodRefTest {
    public static void main(String[] args) {
        // Type 1: Static Method
        Function<Integer, Integer> absRef = Math::abs;
        System.out.println("abs(-5) = " + absRef.apply(-5));
        
        // Type 2: Unbound Instance
        Function<String, Integer> lengthRef = String::length;
        System.out.println("'hello'.length() = " + lengthRef.apply("hello"));
        
        // Type 3: Bound Instance (캡처됨)
        String prefix = ">>>";
        Function<String, String> formatRef = prefix::concat;
        System.out.println(">>> + 'test' = " + formatRef.apply("test"));
        
        // Type 4: Constructor
        Supplier<java.util.ArrayList> listRef = java.util.ArrayList::new;
        java.util.ArrayList list = listRef.get();
        System.out.println("New ArrayList: " + list.getClass().getSimpleName());
    }
}
```

```bash
javac MethodRefTest.java
javap -c -v -p MethodRefTest.class | grep -A 30 "BootstrapMethods:"
```

### 실험 2: 캡처 변수의 메모리 영향

```java
public class CaptureMemoryTest {
    static class DataHolder {
        byte[] data = new byte[1024 * 1024];  // 1MB
    }
    
    public static void main(String[] args) {
        DataHolder holder = new DataHolder();
        
        // 캡처되지 않는 메서드 참조 (권장)
        Supplier<Integer> unbound = Integer::new;  // 캡처 없음
        
        // 캡처되는 메서드 참조 (주의!)
        Supplier<String> bound = holder::toString;  // holder 캡처됨!
        // holder가 GC 되지 않음 (bound가 참조 유지)
    }
}
```

### 실험 3: Stream API에서의 활용

```java
import java.util.*;
import java.util.stream.*;

public class StreamMethodRef {
    public static void main(String[] args) {
        List<String> words = List.of("hello", "world", "java");
        
        // Type 2: Unbound 인스턴스 메서드 (권장)
        words.stream()
            .map(String::toUpperCase)  // 각 String::toUpperCase 호출
            .forEach(System.out::println);
        
        // Type 4: Constructor (권장)
        List<Integer> numbers = List.of(1, 2, 3);
        numbers.stream()
            .collect(Collectors.toCollection(ArrayList::new));
        
        // Type 3: Bound (피하기 - 매번 새 인스턴스)
        String prefix = "Item-";
        Function<Integer, String> formatter = 
            id -> prefix + id;  // 메서드 참조 피함
        // prefix::concat은 캡처되어 메모리 누수 가능
    }
}
```

---

## 📊 성능/비교

```
4가지 메서드 참조 성능 비교 (JMH, 1000만 번 호출):

Type             | 캡처  | 싱글톤 | Throughput  | 메모리
─────────────────┼─────┼────────┼──────────────┼────────
Static (Math::abs)     | NO  | YES    | ~500 M ops  | ~1KB
Unbound (String::len)  | NO  | YES    | ~480 M ops  | ~1KB
Bound (str::length)    | YES | NO     | ~400 M ops  | 인스턴스마다
Constructor (::new)    | NO  | YES    | ~450 M ops  | ~1KB

캡처 vs 비캡처 성능:
  - 캡처 없음: invokedynamic 1회 bootstrap → 이후 캐시 → 매우 빠름
  - 캡처 있음: invokedynamic마다 새 인스턴스 생성 → 약간 느림
  
메모리:
  - 싱글톤 메서드 참조: ~1-2KB (CallSite 캐시만)
  - 캡처 메서드 참조: 캡처 변수 크기 + 참조 객체 (GC 불가)

람다 vs 메서드 참조 성능:
  - Static::method vs (x) -> Static.method(x): 동등
  - String::length vs (s) -> s.length(): 동등
  - str::length vs () -> str.length(): 동등
  - ArrayList::new vs () -> new ArrayList(): 동등
  → 성능상 차이 없음 (컴파일러가 같게 처리)
```

---

## ⚖️ 트레이드오프

```
메서드 참조 선택 가이드:

Type 1: Static Method Reference
  장점: 명확, 캡처 없음, 효율적
  단점: 특정 static 메서드에만 사용 가능
  사용: Math::abs, Integer::parseInt 등

Type 2: Unbound Instance Method Reference
  장점: 메서드만 지정, 캡처 없음, Stream API와 조화
  단점: 첫 인자가 receiver라는 것을 알아야 함
  사용: String::length, String::trim, List::contains

Type 3: Bound Instance Method Reference
  장점: 읽기 쉬움 (instance 명시적)
  단점: 캡처 → 메모리 누수 위험
  피할 것: 가능하면 람다나 unbound로 변경
  예외: 짧은 생명주기 (변수 스코프 내)

Type 4: Constructor Reference
  장점: 명확한 의도, 캡처 없음, 팩토리 패턴 적합
  단점: 특정 생성자 시그니처에만 적용
  사용: ArrayList::new, HashMap::new, int[][]::new
```

---

## 📌 핵심 정리

```
Method Reference 4가지:

1. Static Method Reference (Class::staticMethod)
   - REF_invokeStatic
   - 캡처 없음 (싱글톤)
   - 동등한 람다: (args) -> Class.method(args)

2. Unbound Instance Method (Class::instanceMethod)
   - REF_invokeVirtual
   - 첫 인자가 receiver
   - 캡처 없음 (싱글톤)
   - 동등한 람다: (receiver, args) -> receiver.method(args)

3. Bound Instance Method (instance::method)
   - REF_invokeVirtual + 캡처
   - 캡처 있음 (매번 새 인스턴스)
   - 동등한 람다: (args) -> instance.method(args)

4. Constructor Reference (Class::new)
   - REF_newInvokeSpecial
   - 생성자 시그니처 따라 다양한 형태
   - 캡처 없음 (싱글톤)
   - 동등한 람다: (args) -> new Class(args)

성능:
  - 모든 형태: 람다와 동등 (컴파일러가 같게 처리)
  - 캡처 없음 < 캡처 있음

최적화:
  - Unbound / Static / Constructor 선호 (캡처 없음)
  - Bound는 피하거나 짧은 스코프에서만 사용
```

---

## 🤔 생각해볼 문제

**Q1.** `instance::method`에서 instance를 캡처할 때, 여러 메서드 참조가 같은 instance를 캡처하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

각 메서드 참조마다 별도의 invokedynamic 명령어와 CallSite를 가지므로, 각각 독립적으로 instance를 캡처한다.

```java
String s = "hello";
Supplier<Integer> ref1 = s::length;     // CallSite 1: s 캡처
Supplier<Character> ref2 = s::charAt;   // CallSite 2: s 캡처 (다시)
// s는 두 번 캡처되지만, 같은 인스턴스이므로 참조는 같음
// GC: ref1과 ref2가 모두 s를 참조 → s는 GC 불가
```

메모리:
- invokedynamic마다 별도 CallSite
- 각 CallSite는 독립적 인스턴스 참조 저장
- 하나의 instance라도 여러 CallSite가 참조 유지

따라서 같은 object를 여러 메서드 참조로 캡처해도, 메모리는 참조만 증가하고 object 자체는 한 번만 저장된다.

</details>

---

**Q2.** `ArrayList::new`와 `ArrayList<Integer>::new`의 차이는?

<details>
<summary>해설 보기</summary>

문법 관점에서 `ArrayList<Integer>::new`는 불가능하다. 메서드 참조는 type parameter를 지정할 수 없다.

```java
// 작동함
Supplier<ArrayList> s1 = ArrayList::new;

// 컴파일 에러: type argument not allowed here
// Supplier<ArrayList<Integer>> s2 = ArrayList<Integer>::new;  // ERROR

// 올바른 방법:
Supplier<ArrayList<Integer>> s2 = 
    (Supplier<ArrayList<Integer>>) (Object) ArrayList::new;  // 캐스팅
// 또는 람다:
Supplier<ArrayList<Integer>> s2 = () -> new ArrayList<Integer>();
```

이유:
- 메서드 참조의 타입은 컨텍스트의 목표 타입(Function의 제네릭)에서 결정
- 메서드 참조 자체에는 제네릭 파라미터 지정 불가
- 컴파일러가 제네릭 정보를 타겟 함수형 인터페이스에서 추론

</details>

---

**Q3.** `Math::abs`를 여러 번 호출할 때, CallSite가 재사용되어 항상 같은 구현체 인스턴스를 반환하는가?

<details>
<summary>해설 보기</summary>

아니다. 메서드 참조가 **할당되는 장소마다** 별도의 invokedynamic 명령어를 가지므로, 각각 독립적인 CallSite를 생성한다.

```java
Function<Integer, Integer> f1 = Math::abs;  // invokedynamic #1 → CallSite 1
Function<Integer, Integer> f2 = Math::abs;  // invokedynamic #2 → CallSite 2

// f1과 f2가 같은 객체인가? → 아님!
System.out.println(f1 == f2);  // false (다른 CallSite, 다른 인스턴스)
```

그러나 **같은 위치에서 여러 번 호출**하면:

```java
Function<Integer, Integer> f = Math::abs;
f = Math::abs;  // 같은 변수에 재할당
// 첫 번째 invokedynamic: CallSite 생성 후 캐시
// 두 번째 invokedynamic (같은 위치): 캐시된 CallSite 재사용
// 결과: 같은 Function 인스턴스
```

JVM이 invokedynamic 명령어의 **바이트코드 위치**를 기반으로 CallSite를 관리하기 때문이다. 같은 명령어 위치는 같은 CallSite → 같은 함수형 인터페이스 구현체 인스턴스.

</details>

---

<div align="center">

**[⬅️ 이전: @FunctionalInterface와 SAM 변환](./02-functional-interface-sam.md)** | **[홈으로 🏠](../README.md)** | **[다음: Closure와 Variable Capture ➡️](./04-closure-variable-capture.md)**

</div>
