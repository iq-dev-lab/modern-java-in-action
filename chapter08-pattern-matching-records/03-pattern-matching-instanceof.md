# Pattern Matching for instanceof (Java 16) — 캐스팅 제거

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `if (obj instanceof String s) { ... s.length() ... }` 문법이 어떻게 바이트코드 상 자동 캐스팅으로 변환되는가?
- 패턴 변수 `s`의 스코프 결정 규칙은? (then-branch에서만 가시성)
- Negation 시 변수 가시성이 어떻게 변하는가? (`!(obj instanceof String s)`)
- Switch expression과 결합했을 때의 의미는?
- 중첩 instanceof 패턴의 컴파일 시간 복잡도는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

다형성 기반 코드에서 타입 검사 후 캐스팅은 매우 빈번하다. 방문자 패턴(Visitor), 직렬화/역직렬화, UI 이벤트 처리 등에서 `instanceof ... cast` 체인이 반복된다. Pattern matching for instanceof는 이 보일러플레이트를 제거해서 코드 가독성을 높이고, 버그(잘못된 캐스트)를 방지한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 패턴 변수의 스코프를 과대평가
  public void process(Object obj) {
      if (obj instanceof String s) {
          System.out.println(s.length());
      }
      System.out.println(s);  // ❌ 컴파일 에러: s는 if 블록에서만 가시
  }

실수 2: else 블록에서 패턴 변수 사용
  if (obj instanceof String s) {
      System.out.println("String: " + s.length());
  } else {
      System.out.println("Not string, length = " + s.length());  // ❌ 컴파일 에러
  }

실수 3: Negation 시 패턴 변수 혼동
  if (!(obj instanceof String s)) {
      System.out.println(s.length());  // ❌ 컴파일 에러
  }
  // !(instanceof String s)는 String이 아닐 때 true
  // → s는 if 블록에서 가시성 없음

실수 4: 중첩 instanceof 시 변수 충돌
  if (obj instanceof String s) {
      if (obj instanceof String s) {  // ⚠️ 같은 변수명
          System.out.println(s);  // inner s (shadowing)
      }
  }

실수 5: 패턴 마칭의 컴파일 시간 복잡도 무시
  if (obj instanceof A a) { ... }
  else if (obj instanceof B b) { ... }
  else if (obj instanceof C c) { ... }
  // N개 조건: O(N) 컴파일 시간
  // switch expression (Java 21): O(log N)에 최적화
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 패턴 1: 기본 instanceof pattern
Object obj = "Hello";

if (obj instanceof String s) {
    System.out.println("String length: " + s.length());  // s는 String으로 이미 캐스팅됨
    // instanceof 내에서 String으로 처리 (자동 unboxing 같은 개념)
}
// s는 이 스코프 밖에서 접근 불가

// 패턴 2: else-if 체인
public String classify(Object obj) {
    if (obj instanceof String s) {
        return "String(" + s.length() + ")";
    } else if (obj instanceof Integer i) {
        return "Integer(" + i + ")";
    } else if (obj instanceof Double d) {
        return "Double(" + d + ")";
    } else {
        return "Unknown";
    }
}

// 패턴 3: 부정(negation) 패턴
public void processNonString(Object obj) {
    if (!(obj instanceof String s)) {
        // 이 블록에 진입 = obj는 String이 아님
        // → s는 가시성 없음 (String이 아니므로 의미 없음)
        System.out.println("Not a string");
    } else {
        // 이 블록 = obj는 반드시 String
        System.out.println("String: " + s.length());  // ✅ s 가시성 있음
    }
}

// 패턴 4: switch expression과 결합 (Java 21)
public String classify(Object obj) {
    return switch (obj) {
        case String s -> "String(" + s.length() + ")";
        case Integer i -> "Integer(" + i + ")";
        case Double d -> "Double(" + d + ")";
        case null -> "null";
        default -> "Unknown";
    };
}

// 패턴 5: 복잡한 타입 검사 제거
// Before: 여러 instanceof + cast
if (obj instanceof String) {
    String s = (String) obj;
    if (s.length() > 10) { ... }
}

// After: 패턴 매칭
if (obj instanceof String s && s.length() > 10) {
    // s는 캐스팅 없이 String으로 인식
}

// 패턴 6: 메서드 체인 (cascading guards)
public void handle(Object obj) {
    if (obj instanceof String s && !s.isEmpty()) {
        System.out.println("Non-empty string: " + s);
    }
    if (obj instanceof Integer i && i > 0) {
        System.out.println("Positive integer: " + i);
    }
}

// 패턴 7: 여러 instanceof 조건
if ((obj instanceof String s || obj instanceof StringBuilder sb)) {
    // s는 이 블록에서 불확실 (String일 수도, 아닐 수도)
    // → 컴파일 에러: s는 if 블록에서 단일 스코프에서만 가시
}
```

---

## 🔬 내부 동작 원리

### 1. Pattern Variable의 스코프 결정

```
Java 소스:
  if (obj instanceof String s) {
      System.out.println(s.length());
  }
  System.out.println(s);  // ❌ 에러

컴파일러의 분석:
  ① instanceof 조건이 true인 경우만 s가 정의됨
  ② s의 스코프 = true 브랜치의 블록 전체
  ③ false 브랜치, 블록 밖에서는 s 정의되지 않음

바이트코드 (의사코드):
  if (obj instanceof String) {
      String s = (String) obj;
      System.out.println(s.length());  // s 사용 가능
  }
  // s는 블록 밖에서 어라이 불가능

Scoping rule:
  - if (expr instanceof Type pattern) { A } else { B }
    → pattern은 A에서만 가시성
    → B에서는 pattern 접근 불가능

  - if (!(expr instanceof Type pattern)) { A }
    → A에서는 pattern 불가시 (expr이 Type이 아니므로)
    → pattern이 정의되지 않음

  - if (expr instanceof Type pattern && expr2 instanceof Type2 pattern2) { A }
    → A에서 pattern, pattern2 모두 가시성
```

### 2. Negation과 가시성

```
Pattern matching negation (Java 17+):

정상 케이스:
  if (obj instanceof String s) {
      // s가 String이 맞음
      System.out.println(s.length());  // ✅ s 가시성
  }

부정 케이스:
  if (!(obj instanceof String s)) {
      // obj가 String이 아님을 확인했지만
      // s는 정의되지 않음 (String이 아니므로)
      System.out.println(s);  // ❌ 컴파일 에러
  }

그래서 else로 처리:
  if (!(obj instanceof String s)) {
      System.out.println("Not string");
  } else {
      // !(false) = true, 즉 obj는 반드시 String
      System.out.println(s.length());  // ✅ s 가시성
  }

OR 조건:
  if (obj instanceof String s || obj instanceof StringBuilder sb) {
      // s와 sb 모두 이 블록에서 불확실
      // → 컴파일 에러: 경로에 따라 정의가 다름
  }

AND 조건:
  if (obj instanceof String s && s.length() > 10) {
      // s가 정의되어 있고, s.length() > 10 확인됨
      // → 두 조건 모두 만족할 때만 블록 진입
  }
```

### 3. Switch Expression과의 결합

```
Switch expression (Java 21)에서 pattern matching:

구문:
  switch (expr) {
      case Pattern p1 -> handleP1(p1);
      case Pattern p2 -> handleP2(p2);
      case null -> handleNull();
      default -> handleDefault();
  }

컴파일러 최적화:
  - instanceof 체인 (if-else-if)는 O(N) 비교
  - switch + pattern은 O(log N)으로 최적화 가능
  - JVM이 다양한 dispatch 방식 선택 (tableswitch, lookupswitch, invokedynamic)

예시:
  public String describe(Object obj) {
      return switch (obj) {
          case String s -> "String of length " + s.length();
          case Integer i && i > 0 -> "Positive " + i;
          case Integer i -> "Non-positive " + i;
          case Double d -> "Double " + d;
          case null -> "null";
          default -> obj.getClass().getName();
      };
  }

바이트코드:
  switch의 dispatch 테이블을 사용해 직접 case로 점프
  → if-else 체인보다 빠름 (특히 case 많을 때)
```

### 4. 타입 검사 바이트코드 비교

```
Before (Java 15):
  public void process(Object obj) {
      if (obj instanceof String) {
          String s = (String) obj;
          System.out.println(s.length());
      }
  }

바이트코드:
  0: aload_0       // obj 로드
  1: instanceof 2  // String 체크 → boolean (0/1)
  4: ifeq 15       // false면 15로 점프 (블록 스킵)
  7: aload_0
  8: checkcast 2   // (String) 캐스팅 (이미 instanceof 확인했는데도 재검사)
  11: astore_1
  12: ...          // s.length() 호출
  
  두 번 검사: instanceof + checkcast

After (Java 16+):
  public void process(Object obj) {
      if (obj instanceof String s) {
          System.out.println(s.length());
      }
  }

바이트코드:
  0: aload_0
  1: instanceof 2  // 타입 체크
  4: ifeq 13       // false면 13으로 점프
  7: aload_0
  8: checkcast 2   // 안전한 캐스팅 (이미 instanceof로 검증)
  12: areturn      // s로 바로 사용
  
  JIT 컴파일 시:
  → instanceof 성공 후 checkcast 제거 가능
  → 패턴 변수는 컴파일러가 자동으로 타입 좁혀줌
```

### 5. Pattern Variable Binding의 컴파일 메커니즘

```
컴파일러가 수행하는 작업:

1. Parsing: instanceof String s 인식
2. Type Checking: String s의 타입 확인
3. Scope Analysis: s의 가시 범위 계산
4. Code Generation:
   a. instanceof 체크 (boolean 결과)
   b. true 경로에서 checkcast (선택적, 생략 가능)
   c. pattern 변수 s를 Type String으로 선언
   d. 블록 내에서 s는 String으로 취급

5. Verification: 패턴 변수 사용이 스코프 내인지 확인

컴파일 타임 복잡도:
  N개 패턴: O(N) 스코프 분석 필요
  → switch + pattern: O(log N) 디스패치 테이블로 최적화
```

---

## 💻 실전 실험

### 실험 1: 패턴 변수 스코프 테스트

```java
public class PatternScopeTest {
    public static void main(String[] args) {
        Object obj = "Hello";

        // 기본 스코프
        if (obj instanceof String s) {
            System.out.println("Length: " + s.length());  // ✅ s 가시
        }
        // System.out.println(s);  // ❌ 컴파일 에러: s는 이 스코프에서 미정의

        // else 블록에서 검사
        if (obj instanceof String) {
            System.out.println("String");
        } else {
            // 이 경로는 obj가 String이 아님
            // 하지만 다음은 불가능:
            // System.out.println(s);
        }

        // AND 조건
        if (obj instanceof String s && s.length() > 5) {
            System.out.println("Long string: " + s);  // ✅ s 가시
        }

        // OR 조건 (컴파일 불가)
        if (obj instanceof String s || obj instanceof StringBuilder) {
            // s는 항상 정의되지 않음 (StringBuilder 경로에서)
            // System.out.println(s);  // ❌ 컴파일 에러
        }

        // 부정
        if (!(obj instanceof String s)) {
            // s는 미정의 (obj가 String이 아니므로)
            // System.out.println(s);  // ❌ 컴파일 에러
        } else {
            // 반대 경로: obj는 반드시 String
            System.out.println("String: " + s);  // ✅ s 가시
        }
    }
}
```

### 실험 2: 메서드 체인에서의 패턴 적용

```java
public class ProcessingChain {
    interface Processor {
        void process(Object obj);
    }

    static class StringProcessor {
        void handle(Object obj) {
            // Before: 여러 번 캐스팅
            if (obj instanceof String) {
                String s = (String) obj;
                System.out.println("String: " + s.toUpperCase());
                if (s.length() > 10) {
                    System.out.println("Long string!");
                }
            }

            // After: 패턴 매칭
            if (obj instanceof String s && s.length() > 0) {
                System.out.println("Trimmed: " + s.trim());
                processString(s);
            }
        }

        void processString(String s) {
            System.out.println("Length: " + s.length());
        }
    }

    static class MultiTypeHandler {
        void handle(Object obj) {
            if (obj instanceof String s) {
                handleString(s);
            } else if (obj instanceof Integer i) {
                handleInteger(i);
            } else if (obj instanceof Double d) {
                handleDouble(d);
            } else {
                handleUnknown(obj);
            }
        }

        void handleString(String s) { System.out.println("S: " + s); }
        void handleInteger(Integer i) { System.out.println("I: " + i); }
        void handleDouble(Double d) { System.out.println("D: " + d); }
        void handleUnknown(Object o) { System.out.println("?"); }
    }

    public static void main(String[] args) {
        StringProcessor sp = new StringProcessor();
        sp.handle("Hello World");
        sp.handle(123);

        MultiTypeHandler mth = new MultiTypeHandler();
        mth.handle("Test");
        mth.handle(42);
        mth.handle(3.14);
    }
}
```

### 실험 3: Switch Expression으로 변환

```java
public class SwitchPatternTest {
    static String classifyOld(Object obj) {
        if (obj instanceof String s) {
            return "String(" + s.length() + ")";
        } else if (obj instanceof Integer i) {
            return "Integer(" + i + ")";
        } else if (obj instanceof Double d) {
            return "Double(" + d + ")";
        } else {
            return "Other";
        }
    }

    static String classifyNew(Object obj) {
        return switch (obj) {
            case String s -> "String(" + s.length() + ")";
            case Integer i -> "Integer(" + i + ")";
            case Double d -> "Double(" + d + ")";
            case null -> "null";
            default -> "Other";
        };
    }

    public static void main(String[] args) {
        System.out.println(classifyOld("Hello"));    // String(5)
        System.out.println(classifyOld(42));         // Integer(42)
        System.out.println(classifyOld(3.14));       // Double(3.14)

        System.out.println(classifyNew("Hello"));    // String(5)
        System.out.println(classifyNew(42));         // Integer(42)
        System.out.println(classifyNew(3.14));       // Double(3.14)
        System.out.println(classifyNew(null));       // null
    }
}
```

---

## 📊 성능/비교

```
instanceof chain vs switch pattern:

특성                    | if-else-if instanceof | switch pattern (Java 21)
──────────────────────┼──────────────────────┼────────────────────────
컴파일 구조            | 선형 비교 (O(N))      | 디스패치 테이블 (O(1) lookup)
바이트코드 크기        | 증가 (N개 instanceof) | 최적화됨
런타임 성능 (1개 match)| ~1μs (평균)          | ~0.5μs (빠름)
런타임 성능 (마지막)   | ~10μs (N번 비교)      | ~0.5μs (일정)
메모리 오버헤드        | 적음                 | 디스패치 테이블 (작음)
가독성                | 나쁨 (보일러플레이트)  | 우수 (패턴 명확)

N=10개 case, 마지막 case match 시:
  if-else-if: 10번 instanceof 검사 → ~10μs
  switch: 디스패치 테이블로 직접 접근 → ~0.5μs
  
결론: switch pattern이 N이 크거나 분포가 균형잡혀 있을 때 현저히 빠름
```

---

## ⚖️ 트레이드오프

```
Pattern Matching for instanceof 도입:

장점:
  - 보일러플레이트 제거: instanceof + cast 체인 제거
  - 버그 방지: 잘못된 cast 불가능
  - 가독성: 의도가 명확한 코드
  - 타입 안전성: 컴파일러가 타입 검증

단점:
  - Java 16 이상 필수: 레거시 프로젝트 사용 불가
  - 스코프 규칙 이해 필요: 초보자 혼동 가능
  - 중첩 instanceof 시 복잡도 증가
  - switch pattern (Java 21)은 더 최근 버전 필요

Pattern variable scoping:

장점:
  - 변수 생명주기 명확: 필요한 범위에만 가시
  - 메모리: 불필요한 범위에서 참조 제거

단점:
  - 변수 이름 충돌 불가: 명시적 shadowing 불허
  - OR 조건에서 모호성: 경로마다 정의 불확실
  - 학습곡선: 스코프 규칙이 직관적이지 않을 수 있음
```

---

## 📌 핵심 정리

```
Pattern Matching for instanceof 핵심:

1. 기본 문법:
   if (obj instanceof Type pattern) { ... }
   → pattern은 then-branch에서만 가시, 자동 캐스팅

2. 스코프 규칙:
   - if에서만 가시성
   - else에서는 미정의
   - AND 조건에서는 모두 가시
   - OR 조건에서는 모호 (컴파일 에러)

3. Negation:
   if (!(obj instanceof String s)) { ... }
   → s는 이 블록에서 미정의
   → else에서 s 가시성 (obj가 String임을 보장)

4. Cascading guards:
   if (obj instanceof String s && s.length() > 10) { ... }
   → 두 조건 모두 만족할 때만 진입

5. Switch pattern (Java 21):
   switch (obj) {
       case Type pattern -> handlePattern();
       case null -> handleNull();
       default -> handleDefault();
   }
   → O(1) 디스패치 (if-else-if의 O(N) 개선)

6. 바이트코드:
   - instanceof 체크 (boolean)
   - true 경로에서 checkcast (이미 검증됨)
   - pattern 변수를 Type으로 바인딩
```

---

## 🤔 생각해볼 문제

**Q1.** 패턴 변수가 if의 then-branch에서만 가시성을 갖는 이유는?

<details>
<summary>해설 보기</summary>

스코프는 변수가 의미를 갖는 범위를 나타낸다. `instanceof Type pattern`은:

1. condition이 true일 때만 pattern이 정의됨 (타입 보장)
2. condition이 false면 pattern이 정의되지 않음

```java
if (obj instanceof String s) {  // true 경로
    s.length();  // ✅ s는 String으로 보장
} else {  // false 경로
    // obj가 String이 아님을 확인했으므로
    // s를 String으로 정의할 수 없음
}
// 블록 밖
// 어느 경로를 거쳤는지 불명확하므로 s 정의 불가
```

이것이 **condition이 true인 코드 경로에서만 pattern이 정의되는 이유**다. 컴파일러가 control flow를 분석해서 pattern 정의 범위를 결정한다.

else에서 pattern을 사용하려면:
```java
if (!(obj instanceof String s)) {
    // false: obj가 String이 아님 → s 미정의
} else {
    // true: obj가 String임 (부정의 부정) → s 정의
    s.length();  // ✅
}
```

</details>

---

**Q2.** switch pattern이 if-else-if instanceof 체인보다 성능이 나은 이유는?

<details>
<summary>해설 보기</summary>

CPU와 메모리 캐시의 특성 때문이다:

**if-else-if 체인 (선형 탐색, O(N)):**
```
if (instanceof String)    → miss (false)
  else if (instanceof Integer)  → miss (false)
    else if (instanceof Double)  → miss (false)
      else if (instanceof List)  → hit (true)
```

각 instanceof는 타입 체크 비용 (~1-2 사이클)이고, 평균적으로 N/2번 실행. CPU는 조건 분기를 예측(branch prediction)하려고 하지만, 계속 miss하면 파이프라인 플러시 발생.

**switch pattern (O(1) 디스패치):**
```
obj의 타입 → 해시 또는 인덱스 계산 → 디스패치 테이블로 직접 접근
```

JVM이 생성한 디스패치 테이블:
```
Type 인덱스:
  0: String handler
  1: Integer handler
  2: Double handler
  3: List handler
```

switch (obj)는:
1. obj의 런타임 타입 결정 (1회)
2. 테이블 인덱싱 (1회)
3. 직접 해당 case로 점프

**결과: 항상 O(1) + 분기 예측 성공률 높음 → 파이프라인 플러시 감소 → 빠름**

또한 JVM이 monomorphic call site 최적화:
```java
switch (obj) {
    case String s -> ...;
    case Integer i -> ...;
}
```
이 코드가 대부분의 경우 String으로 들어오면, JVM이 String 경로만 인라인 최적화 가능.

</details>

---

**Q3.** `if (obj instanceof String s || obj instanceof StringBuilder sb)`가 컴파일 에러인 이유는?

<details>
<summary>해설 보기</summary>

or 조건에서 패턴 변수는 **불확실성**을 갖는다:

```java
if (obj instanceof String s || obj instanceof StringBuilder sb) {
    // 컴파일러의 관점:
    // - 왼쪽 참: s 정의, sb 미정의
    // - 오른쪽 참: s 미정의, sb 정의
    // → 이 블록에 진입했을 때 s와 sb 중 어떤 것이 정의되었는지 알 수 없음
    System.out.println(s.length());  // ❌ s가 항상 정의되었다고 보장할 수 없음
    System.out.println(sb.length());  // ❌ sb가 항상 정의되었다고 보장할 수 없음
}
```

이는 **control flow 분석**의 안전성 원칙:
- 패턴 변수는 모든 경로에서 정의된 경우에만 가시

해결 방법:
```java
if (obj instanceof String s) {
    System.out.println(s.length());  // s만 이 블록에서 정의
} else if (obj instanceof StringBuilder sb) {
    System.out.println(sb.length());  // sb만 이 블록에서 정의
}

// 또는 switch pattern (Java 21)
switch (obj) {
    case String s -> System.out.println(s.length());
    case StringBuilder sb -> System.out.println(sb.length());
}
```

이것이 **OR 조건에서 패턴이 비추천**되고 **switch pattern이 권장**되는 이유다.

</details>

---

<div align="center">

**[⬅️ 이전: Record vs Lombok @Data](./02-record-vs-lombok.md)** | **[홈으로 🏠](../README.md)** | **[다음: Switch Expression ➡️](./04-switch-expression.md)**

</div>
