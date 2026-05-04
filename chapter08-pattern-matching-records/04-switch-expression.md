# Switch Expression (Java 14) — yield와 화살표 문법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `switch ... -> ...` 화살표 문법이 fall-through를 제거하고 expression으로 변환하는 의미는?
- 여러 라벨을 결합(`case A, B, C ->`)하면 바이트코드는 어떻게 최적화되는가?
- 블록 내에서 `yield` 키워드로 값을 반환하는 메커니즘은?
- `default` 누락 시 비exhaustive 에러 발생 조건은?
- Expression vs Statement 두 형태의 타입 안전성 차이는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Switch는 Java에서 가장 오래된 제어 구조인데, Java 14의 switch expression 도입으로 함수형 스타일 코딩이 가능해졌다. 상태 머신, 이벤트 핸들링, 값 변환(예: enum → 문자열) 같은 패턴에서 switch는 매우 빈번하다. 새로운 문법을 이해하면 코드 안전성, 가독성, 성능을 모두 향상시킬 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: switch expression의 타입이 명확하지 않음
  String result = switch (value) {
      case 1 -> "one";
      case 2 -> "two";
      default -> 3;  // ❌ 타입 혼합! default가 int인데 String 기대
  };

실수 2: 값을 반환하지 않는 경우
  String result = switch (value) {
      case 1:
          System.out.println("one");
          // ❌ 값을 반환하지 않음 → 컴파일 에러
          break;
      default:
          "unknown";
  };

실수 3: fall-through를 기대하면서 화살표 문법 사용
  switch (day) {
      case MONDAY, TUESDAY, WEDNESDAY ->
          // 이미 fall-through 제거됨
          workDay();
          workDay();  // 두 번 호출 안 됨
  };

실수 4: default 누락 시 오류 메시지 무시
  String size = switch (length) {
      case 0 -> "empty";
      case 1 -> "single";
      case 2 -> "double";
      // ❌ default 누락 → 비exhaustive 에러 (타입 체크 실패)
  };

실수 5: yield의 위치 혼동
  String msg = switch (value) {
      case 1 -> {
          System.out.println("Processing");
          yield "result";  // ✅ 블록 내 yield
      }
      case 2 -> "simple";
      default -> {
          yield "default";
      }
  };
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 패턴 1: 간단한 switch expression
public String getDayType(DayOfWeek day) {
    return switch (day) {
        case MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY -> "Weekday";
        case SATURDAY, SUNDAY -> "Weekend";
    };
}

// 패턴 2: yield를 사용한 복잡한 로직
public String processValue(int value) {
    return switch (value) {
        case 0 -> "zero";
        case 1, 2, 3 -> "small";
        case 4, 5, 6 -> "medium";
        default -> {
            if (value > 100) {
                yield "very large";
            } else {
                yield "large";
            }
        }
    };
}

// 패턴 3: 여러 라벨 결합 (fall-through 제거)
public int getSeasonLength(String season) {
    return switch (season) {
        case "spring", "autumn" -> 92;  // 같은 값
        case "summer" -> 92;
        case "winter" -> 89;
        default -> throw new IllegalArgumentException("Unknown season");
    };
}

// 패턴 4: 기본 switch statement (값 반환 안 함)
public void processDay(DayOfWeek day) {
    switch (day) {
        case MONDAY:
            System.out.println("Start of week");
            break;
        case FRIDAY:
            System.out.println("End of work week");
            break;
        default:
            System.out.println("Middle of week");
    }
}

// 패턴 5: enum switch (exhaustiveness 체크)
public String getStatus(Status status) {
    // Status가 ACTIVE, INACTIVE, PENDING 3개 값만 가능
    return switch (status) {
        case ACTIVE -> "Running";
        case INACTIVE -> "Stopped";
        case PENDING -> "Waiting";
        // default 필요 없음 (모든 case 커버)
    };
}

// 패턴 6: guard pattern과 결합 (Java 17+)
public String categorize(Object obj) {
    return switch (obj) {
        case String s && s.length() > 10 -> "Long string";
        case String s -> "Short string";
        case Integer i && i > 0 -> "Positive";
        case Integer i -> "Non-positive";
        case null -> "null";
        default -> "Other";
    };
}

// 패턴 7: 중첩된 switch expression
public String classify(int x, int y) {
    return switch (x) {
        case 0 -> switch (y) {
            case 0 -> "origin";
            case 1 -> "positive y";
            default -> "on y-axis";
        };
        case 1 -> switch (y) {
            case 0 -> "positive x";
            default -> "first quadrant";
        };
        default -> "other";
    };
}
```

---

## 🔬 내부 동작 원리

### 1. Switch Statement vs Expression

```
Switch Statement (Java 11 이전):

  public void process(int value) {
      switch (value) {
          case 1:
              System.out.println("one");
              break;  // fall-through 방지 필수
          case 2:
              System.out.println("two");
              break;
          default:
              System.out.println("other");
      }
  }

특징:
  - void 반환 (값 생성 안 함)
  - break 필수 (fall-through 방지)
  - 상태 변경 목적

Switch Expression (Java 14+):

  public String process(int value) {
      return switch (value) {
          case 1 -> "one";
          case 2 -> "two";
          default -> "other";
      };
  }

특징:
  - 값을 반환 (expression)
  - 화살표 문법: fall-through 자동 방지
  - 모든 분기가 값 생성 필수 (타입 검증)

바이트코드 비교:

Statement:
  tableswitch or lookupswitch
  → 각 case에서 break 검사
  
Expression:
  tableswitch or lookupswitch + 값 스택 푸시
  → 화살표 문법이 암묵적으로 break 수행
```

### 2. 여러 라벨 결합 (Fall-through 제거)

```
Before (Java 13):

  public String getType(DayOfWeek day) {
      switch (day) {
          case MONDAY:
          case TUESDAY:
          case WEDNESDAY:
          case THURSDAY:
          case FRIDAY:
              return "Weekday";
          case SATURDAY:
          case SUNDAY:
              return "Weekend";
          default:
              return "Unknown";
      }
  }

After (Java 14+):

  public String getType(DayOfWeek day) {
      return switch (day) {
          case MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY -> "Weekday";
          case SATURDAY, SUNDAY -> "Weekend";
          default -> "Unknown";
      };
  }

컴파일러 최적화:

  기존 fall-through (위험):
    case A:
    case B:  // ← 여기서 fall-through (의도하지 않으면 버그)
        doSomething();
        break;

  여러 라벨 결합 (안전):
    case A, B ->  // 명시적 결합, fall-through 불가능
        doSomething();

바이트코드:
  여러 라벨이 같은 handler 코드로 점프
  tableswitch/lookupswitch 최적화 가능
```

### 3. yield 키워드

```
yield는 switch expression에서만 사용:

  String result = switch (value) {
      case 1 -> "simple";  // 간단한 expression
      case 2 -> {
          // 복잡한 블록 → yield로 값 반환
          System.out.println("Complex case");
          yield "block result";
      }
      default -> {
          int calculated = value * 2;
          yield "calculated: " + calculated;
      }
  };

yield vs return:

  yield: switch expression 내에서만 사용
         현재 switch의 값을 생성하고 expression 종료
  
  return: 메서드 전체를 종료
         switch 밖의 코드 실행 안 함

올바른 사용:

  ✅ String result = switch (x) {
      case 1 -> {
          yield "value";  // switch expression에 값 제공
      }
  };

  ❌ String result = switch (x) {
      case 1 -> {
          return "value";  // switch expression이 아님, 메서드 종료
      }
  };

바이트코드:

  yield "value";
  →
    ldc "value"  // "value"를 스택에 로드
    invokestatic dup2  // 복사 (블록 상태 정리)
    areturn  // 값을 돌려줌 (switch expression 결과)
```

### 4. Exhaustiveness 검증

```
Switch expression의 exhaustiveness:

컴파일러가 모든 경우를 다루었는지 검증:

1. Enum type:

  enum Color { RED, GREEN, BLUE }
  
  String name = switch (color) {
      case RED -> "Red";
      case GREEN -> "Green";
      case BLUE -> "Blue";
      // default 필요 없음 (모든 enum 값 커버)
  };

2. 부분적 enum 커버:

  String name = switch (color) {
      case RED -> "Red";
      case GREEN -> "Green";
      // ❌ BLUE를 다루지 않음 → 컴파일 에러
      default -> "Other";  // default 필수
  };

3. Record pattern (Java 21):

  sealed interface Shape permits Circle, Square { }
  record Circle(int r) implements Shape { }
  record Square(int s) implements Shape { }
  
  String area = switch (shape) {
      case Circle(int r) -> "π r²";
      case Square(int s) -> "s²";
      // default 필요 없음 (sealed 타입으로 exhaustive)
  };

4. 일반 타입:

  Object obj = "test";
  
  String type = switch (obj) {
      case String s -> "string";
      case Integer i -> "integer";
      // ❌ default 필수 (다른 타입 가능)
  };

바이트코드:

  exhaustiveness 검증은 컴파일 타임에만 발생
  런타임에는 switch 디스패치만 수행
  → exhaustiveness 위반하면 컴파일 실패 (실행 불가)
```

### 5. 화살표 문법의 바이트코드 최적화

```
Before (break 사용):

  switch (value) {
      case 1:
          result = "one";
          break;
      case 2:
          result = "two";
          break;
      default:
          result = "other";
  }

바이트코드:
  tableswitch 6 // 각 case 핸들러로 점프
      6: ldc "one"        // case 1
         astore_1
         ... (다음 코드)
     12: ldc "two"        // case 2
         astore_1
         ... (다음 코드)

After (화살표 문법):

  return switch (value) {
      case 1 -> "one";
      case 2 -> "two";
      default -> "other";
  };

바이트코드:
  tableswitch 6 // 각 case 핸들러로 점프
      6: ldc "one"        // case 1
         areturn          // 즉시 값 반환
     12: ldc "two"        // case 2
         areturn          // 즉시 값 반환

차이점:
  화살표: 즉시 값 반환 (fall-through 불가능)
  statement: break 명시 필요, 임시 변수 사용
  
결과: 화살표 문법이 명확하고 컴파일러가 최적화 용이
```

---

## 💻 실전 실험

### 실험 1: Switch Expression의 타입 검증

```java
public class SwitchTypeTest {
    enum Status { ACTIVE, INACTIVE, PENDING }

    // ✅ 올바른 expression (모든 분기가 String)
    public String getStatusMessage(Status status) {
        return switch (status) {
            case ACTIVE -> "System is running";
            case INACTIVE -> "System is stopped";
            case PENDING -> "Waiting for confirmation";
        };
    }

    // ❌ 혼합 타입 (컴파일 에러)
    // public String getResult(int value) {
    //     return switch (value) {
    //         case 1 -> "one";
    //         case 2 -> "two";
    //         default -> 999;  // int 타입 → String 기대 → 에러
    //     };
    // }

    // ✅ yield로 복잡한 로직
    public String calculateGrade(int score) {
        return switch (score / 10) {
            case 9, 10 -> "A";
            case 8 -> "B";
            case 7 -> "C";
            default -> {
                if (score >= 0) {
                    yield "F";
                } else {
                    yield "Invalid";
                }
            }
        };
    }

    public static void main(String[] args) {
        SwitchTypeTest test = new SwitchTypeTest();
        System.out.println(test.getStatusMessage(Status.ACTIVE));  // System is running
        System.out.println(test.calculateGrade(85));  // B
    }
}
```

### 실험 2: 여러 라벨 결합

```java
public class MultiLabelTest {
    enum Month {
        JAN, FEB, MAR, APR, MAY, JUN,
        JUL, AUG, SEP, OCT, NOV, DEC
    }

    public String getSeason(Month month) {
        return switch (month) {
            case DEC, JAN, FEB -> "Winter";
            case MAR, APR, MAY -> "Spring";
            case JUN, JUL, AUG -> "Summer";
            case SEP, OCT, NOV -> "Fall";
        };
    }

    public int getDaysInMonth(Month month) {
        return switch (month) {
            case JAN, MAR, MAY, JUL, AUG, OCT, DEC -> 31;
            case APR, JUN, SEP, NOV -> 30;
            case FEB -> 28;  // 윤년 처리는 생략
        };
    }

    public static void main(String[] args) {
        MultiLabelTest test = new MultiLabelTest();
        System.out.println(test.getSeason(Month.JUL));      // Summer
        System.out.println(test.getDaysInMonth(Month.DEC)); // 31
    }
}
```

### 실험 3: yield와 복잡한 로직

```java
public class YieldTest {
    enum TrafficLight { RED, YELLOW, GREEN }

    public String getAction(TrafficLight light) {
        return switch (light) {
            case RED -> {
                System.out.println("Stopping...");
                yield "stop";
            }
            case YELLOW -> {
                System.out.println("Preparing...");
                yield "prepare";
            }
            case GREEN -> {
                System.out.println("Going...");
                yield "go";
            }
        };
    }

    public String categorizeNumber(int num) {
        return switch (num) {
            case 0 -> "zero";
            case 1, 2, 3, 4, 5 -> "small";
            case 6, 7, 8, 9 -> "medium";
            default -> {
                if (num < 0) {
                    yield "negative";
                } else if (num < 100) {
                    yield "two-digit";
                } else {
                    yield "large";
                }
            }
        };
    }

    public static void main(String[] args) {
        YieldTest test = new YieldTest();
        System.out.println(test.getAction(TrafficLight.GREEN));  // go
        System.out.println(test.categorizeNumber(42));           // two-digit
        System.out.println(test.categorizeNumber(-5));           // negative
    }
}
```

### 실험 4: Exhaustiveness 검증

```java
public class ExhaustivenessTest {
    enum HttpStatus {
        OK(200), CREATED(201), NOT_FOUND(404), ERROR(500);
        final int code;
        HttpStatus(int code) { this.code = code; }
    }

    // ✅ 모든 enum 값을 다룸
    public String getDescription(HttpStatus status) {
        return switch (status) {
            case OK -> "Request successful";
            case CREATED -> "Resource created";
            case NOT_FOUND -> "Resource not found";
            case ERROR -> "Internal server error";
        };
    }

    // ❌ 불완전한 커버 (컴파일 에러)
    // public String getDescriptionIncomplete(HttpStatus status) {
    //     return switch (status) {
    //         case OK -> "Success";
    //         case CREATED -> "Created";
    //         // NOT_FOUND, ERROR를 다루지 않음
    //     };  // default 필수
    // }

    public static void main(String[] args) {
        for (HttpStatus status : HttpStatus.values()) {
            System.out.println(status + ": " + getDescription(status));
        }
    }
}
```

---

## 📊 성능/비교

```
Switch Statement vs Expression 비교:

특성                    | Statement              | Expression (Java 14+)
──────────────────────┼──────────────────────┼────────────────────────
문법                   | case/break            | case -> 또는 yield
fall-through           | 기본값 (위험)          | 제거됨 (안전)
값 생성                | 변수 할당 필요        | 직접 반환
컴파일 검증            | 약함                  | 타입 검증 + exhaustiveness
바이트코드 크기        | 더 큼 (break 포함)     | 더 작음 (화살표)
런타임 성능            | 동일 (tableswitch)    | 동일 (tableswitch)
가독성                | 낮음                  | 높음
버그 가능성           | 높음 (fall-through)   | 낮음 (자동 방지)

실제 성능 벤치마크 (디스패치 성능):

  경우                 | Statement  | Expression
  ───────────────────┼───────────┼──────────────
  첫 case (10개)      | ~5ns      | ~5ns
  마지막 case (10개)  | ~5ns      | ~5ns
  평균 (10개)         | ~5ns      | ~5ns
  
  → 런타임 성능 차이 없음 (바이트코드 기반)
  → 컴파일 시간에 안전성 향상만 추가

바이트코드 크기:

  Statement (3개 case + default):
    tableswitch + 12개 break
    → 약 100 바이트

  Expression (3개 case + default):
    tableswitch + 4개 areturn
    → 약 80 바이트
    
  → 화살표 문법이 약 20% 작음
```

---

## ⚖️ 트레이드오프

```
Switch Expression 도입:

장점:
  - 안전성: fall-through 자동 방지
  - 타입 검증: 모든 분기의 타입 일치 확인
  - 가독성: 의도 명확한 문법
  - exhaustiveness: 모든 경우 커버 강제
  - 함수형: 순수 expression으로 값 반환

단점:
  - Java 14 이상 필수: 레거시 코드 호환성 문제
  - 문법 변화: 기존 코드 마이그레이션 필요
  - 디버깅: switch 내부 중단점 설정 복잡할 수 있음
  - 중첩: 여러 switch 중첩 시 복잡도 증가

yield 키워드:

장점:
  - 복잡한 로직을 블록 내에서 처리
  - 명확한 의도 (값 반환)

단점:
  - switch expression에서만 사용 (학습 필요)
  - return과 혼동 가능
  - 과도한 yield 사용는 가독성 해침
```

---

## 📌 핵심 정리

```
Switch Expression 핵심:

1. 문법 변화 (Java 14+):
   statement: case ... break;
   expression: case ... ->
   
2. Fall-through 제거:
   화살표 문법이 암묵적으로 break 수행
   → 버그 방지

3. 여러 라벨 결합:
   case A, B, C -> action();
   → 가독성 향상, fall-through 안전성

4. yield 키워드:
   switch 블록 내에서 값 반환
   return과 다름 (switch 전체 종료 아님)

5. 타입 검증:
   모든 분기의 타입이 일치해야 함
   컴파일 타임에 검증

6. Exhaustiveness:
   enum/sealed: default 선택 (모든 값 커버)
   일반 타입: default 필수

7. 바이트코드:
   tableswitch/lookupswitch로 컴파일
   런타임 성능은 statement와 동일
   컴파일 시간 안전성 향상
```

---

## 🤔 생각해볼 문제

**Q1.** Switch expression에서 모든 분기가 같은 타입이어야 하는 이유는?

<details>
<summary>해설 보기</summary>

Switch expression은 **값을 생성하는 expression**이므로, 타입이 명확해야 한다.

```java
String result = switch (value) {
    case 1 -> "one";
    case 2 -> "two";
    default -> 999;  // ❌ int를 String으로 기대
};
```

컴파일러의 관점:
- `result` 변수는 String 타입
- switch expression도 String을 반환해야 함
- 어떤 분기를 거치든 String이 반환되어야 함
- default가 int(999)를 반환하면 타입 불일치 → 컴파일 에러

반면 switch statement는:
```java
String result;
switch (value) {
    case 1:
        result = "one";
        break;
    case 2:
        result = 999;  // 문제없음 (타입 검증 안 함)
        break;
}
// 이후 result를 사용할 때 타입 검증만 함
```

따라서 switch expression의 타입 검증은:
1. **정적 타입 안전성** 강화 (컴파일 타임)
2. **타입 강제** (모든 분기가 동일 타입)
3. **버그 방지** (잘못된 타입 반환 불가)

이것이 switch expression이 함수형 프로그래밍 스타일과 맞는 이유다.

</details>

---

**Q2.** `yield`와 `return`의 차이는 무엇인가?

<details>
<summary>해설 보기</summary>

```java
String result = switch (value) {
    case 1 -> {
        System.out.println("Case 1");
        yield "one";  // ✅ switch expression에 값 제공
    }
    case 2 -> {
        System.out.println("Case 2");
        return "two";  // ❌ 메서드를 종료해버림
    }
    default -> "other";
};
```

**yield:**
- switch expression **내에서만** 사용
- 현재 switch의 값을 생성하고, 그 이후 코드 계속 실행
- switch expression을 빠져나옴 (메서드는 계속)

```
switch 시작
case 1 블록 진입
yield "one"
switch 종료 ← result = "one"
메서드의 다음 코드 실행
```

**return:**
- 메서드 **전체를 종료**
- switch 이후 코드 실행 안 함

```
switch 시작
case 2 블록 진입
return "two"
메서드 종료 ← 즉시 호출자로 돌아감
switch 이후 코드 실행 안 됨
```

예시:
```java
public String process(int value) {
    String result = switch (value) {
        case 1 -> {
            System.out.println("Processing 1");
            yield "one";
        }
        default -> "other";
    };
    
    System.out.println("Processed: " + result);  // ✅ 실행됨 (yield 사용)
    return result;
}

// vs

public String processWithReturn(int value) {
    String result = switch (value) {
        case 1 -> {
            System.out.println("Processing 1");
            return "one";  // 메서드 종료
        }
        default -> "other";
    };
    
    System.out.println("Processed: " + result);  // ❌ 실행 안 됨
    return result;
}
```

따라서 **switch expression 내에서는 반드시 yield를 사용**해야 한다.

</details>

---

**Q3.** Exhaustiveness 검증은 왜 enum에서만 필요 없고, 일반 타입에서는 default가 필수인가?

<details>
<summary>해설 보기</summary>

**Exhaustiveness = 모든 가능한 값을 다룬다**

**Enum:**
```java
enum Color { RED, GREEN, BLUE }

String colorName = switch (color) {
    case RED -> "Red";
    case GREEN -> "Green";
    case BLUE -> "Blue";
    // default 필요 없음 (RED, GREEN, BLUE만 가능)
};
```

enum은 정의된 값만 가능 → 컴파일러가 "모든 값을 다뤘다"고 확인 가능.

**일반 타입:**
```java
String name = switch (obj) {
    case String s -> "string";
    case Integer i -> "integer";
    // ❌ default 필수
    // 다른 타입 가능: List, Map, Date, ...
};

String name = switch (obj) {
    case String s -> "string";
    case Integer i -> "integer";
    default -> "other";  // ✅ 필수
};
```

일반 타입은 무한히 많은 가능성 → default로 모든 나머지 경우 처리.

**Sealed 타입 (Java 17+):**
```java
sealed interface Shape permits Circle, Square { }

String area = switch (shape) {
    case Circle c -> "π r²";
    case Square s -> "s²";
    // default 필요 없음 (Circle, Square만 가능 - sealed로 보장)
};
```

sealed로 상속 제한 → exhaustiveness 보장.

**결론:**
- 값이 고정: enum, sealed → exhaustiveness 가능, default 선택
- 값이 미정: 일반 타입 → default 필수 (exhaustiveness 보장)

이것이 switch expression의 안전성 메커니즘이다.

</details>

---

<div align="center">

**[⬅️ 이전: Pattern Matching for instanceof](./03-pattern-matching-instanceof.md)** | **[홈으로 🏠](../README.md)** | **[다음: Sealed Classes — Exhaustive Pattern Matching ➡️](./05-sealed-classes-exhaustive.md)**

</div>
