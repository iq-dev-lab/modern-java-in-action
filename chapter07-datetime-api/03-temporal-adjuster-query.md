# TemporalAdjuster · TemporalQuery — 활용 패턴

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `TemporalAdjuster`는 함수형 인터페이스인가, 전략 패턴인가?
- `TemporalAdjusters.lastDayOfMonth()`의 구현은 어떻게 되어 있는가?
- "다음 월요일"과 "다음 영업일"을 어떻게 표현하는가?
- `TemporalQuery`는 언제 사용하고, 람다로 쓸 수 있는가?
- Custom `TemporalAdjuster`를 만들 때 주의할 점은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

복잡한 날짜 계산(월말, 다음 영업일, 반기 시작일 등)은 비즈니스 로직의 핵심이다. Java 8 이전에는 이런 계산을 직접 구현했고, 버그의 온상이었다. `TemporalAdjuster`는 이를 함수형으로 표현하고, `TemporalAdjusters` 라이브러리는 자주 쓰는 패턴을 제공한다. 금융, HR, 스케줄링 시스템에서 필수다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 날짜 계산을 조건문으로 직접 구현
  // "다음 월요일" 구하기 (지옥의 코드)
  LocalDate date = LocalDate.now();
  int dayOfWeek = date.getDayOfWeek().getValue();  // 1=월, 7=일
  
  int daysToAdd;
  if (dayOfWeek == 7) {           // 일요일 = 7
      daysToAdd = 1;
  } else if (dayOfWeek == 6) {    // 토요일 = 6
      daysToAdd = 2;
  } else {
      daysToAdd = (8 - dayOfWeek) % 7;
  }
  
  LocalDate nextMonday = date.plusDays(daysToAdd);
  // → 버그: 월요일 자신도 "다음" 월요일에 포함되는가?
  // → 일요일이면 다음날? 8일?

실수 2: 반복되는 계산을 메서드로 만들지만 일관성 없음
  static LocalDate nextMonday(LocalDate date) {
      int dow = date.getDayOfWeek().getValue();
      return date.plusDays(8 - dow);
  }
  
  static LocalDate nextFriday(LocalDate date) {
      // 다른 로직... (복사/붙여넣기 버그)
      int dow = date.getDayOfWeek().getValue();
      return date.plusDays(12 - dow);  // 오류!
  }

실제 3: 월말을 쿼리로 구하지 않고 매직 넘버 사용
  LocalDate eom = LocalDate.of(year, month, 28);  // 2월 대비
  while (eom.getMonth() == month) {
      eom = eom.plusDays(1);
  }
  eom = eom.minusDays(1);
  // → 복잡, 성능 낭비

실수 4: TemporalQuery의 목적을 모름
  LocalDate date = LocalDate.of(2024, 2, 14);
  
  // 쿼리: "이 날짜가 윤년인가?"
  boolean isLeapYear = date.query(something);  // something은?
  
  // TemporalQuery가 있는데도 직접 구현
  boolean isLeap = (date.getYear() % 4 == 0 && 
                    date.getYear() % 100 != 0) || 
                   (date.getYear() % 400 == 0);
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
import java.time.*;
import java.time.temporal.*;

// 1. 표준 TemporalAdjuster 사용
public class StandardAdjusters {
    public static void main(String[] args) {
        LocalDate date = LocalDate.of(2024, 2, 14);  // 수요일
        
        // 다음 월요일
        LocalDate nextMonday = date.with(
            TemporalAdjusters.next(DayOfWeek.MONDAY)
        );
        System.out.println(nextMonday);  // 2024-02-19
        
        // 이번 월요일 또는 다음 월요일 (오늘 포함)
        LocalDate nextOrSame = date.with(
            TemporalAdjusters.nextOrSame(DayOfWeek.MONDAY)
        );
        System.out.println(nextOrSame);  // 2024-02-19
        
        // 이전 월요일
        LocalDate prevMonday = date.with(
            TemporalAdjusters.previous(DayOfWeek.MONDAY)
        );
        System.out.println(prevMonday);  // 2024-02-12
        
        // 월의 첫 날
        LocalDate firstDay = date.with(TemporalAdjusters.firstDayOfMonth());
        System.out.println(firstDay);  // 2024-02-01
        
        // 월의 마지막 날
        LocalDate lastDay = date.with(TemporalAdjusters.lastDayOfMonth());
        System.out.println(lastDay);  // 2024-02-29 (윤년)
        
        // 연의 첫 날
        LocalDate jan1 = date.with(TemporalAdjusters.firstDayOfYear());
        System.out.println(jan1);  // 2024-01-01
        
        // 월의 n번째 특정 요일
        LocalDate thirdMonday = date.with(
            TemporalAdjusters.dayOfWeekInMonth(3, DayOfWeek.MONDAY)
        );
        System.out.println(thirdMonday);  // 2024-02-19
    }
}

// 2. Custom TemporalAdjuster
public class CustomAdjusters {
    // 영업일 기준 (주말 제외)
    public static TemporalAdjuster nextBusinessDay() {
        return temporal -> {
            LocalDate date = LocalDate.from(temporal);
            do {
                date = date.plusDays(1);
            } while (date.getDayOfWeek().getValue() > 5);  // 토일 건너뜀
            return date;
        };
    }
    
    // 반기 시작 (1월, 7월)
    public static TemporalAdjuster firstDayOfHalfYear() {
        return temporal -> {
            LocalDate date = LocalDate.from(temporal);
            return date.getMonthValue() <= 6
                ? LocalDate.of(date.getYear(), 1, 1)
                : LocalDate.of(date.getYear(), 7, 1);
        };
    }
    
    // 다음 월말
    public static TemporalAdjuster nextMonthEnd() {
        return temporal -> LocalDate.from(temporal)
            .with(TemporalAdjusters.lastDayOfMonth())
            .plusMonths(1)
            .with(TemporalAdjusters.lastDayOfMonth());
    }
    
    public static void main(String[] args) {
        LocalDate date = LocalDate.of(2024, 2, 14);
        
        LocalDate nextBiz = date.with(nextBusinessDay());
        System.out.println(nextBiz);  // 2024-02-15
        
        LocalDate halfYear = date.with(firstDayOfHalfYear());
        System.out.println(halfYear);  // 2024-01-01
    }
}

// 3. TemporalQuery로 정보 추출
public class TemporalQueries {
    // 윤년 여부
    public static TemporalQuery<Boolean> isLeapYear() {
        return temporal -> {
            Year year = Year.from(temporal);
            return year.isLeap();
        };
    }
    
    // 월의 마지막 날짜
    public static TemporalQuery<Integer> dayOfLastDayOfMonth() {
        return temporal -> {
            YearMonth ym = YearMonth.from(temporal);
            return ym.lengthOfMonth();
        };
    }
    
    // 분기 (1-4)
    public static TemporalQuery<Integer> quarter() {
        return temporal -> {
            Month month = Month.from(temporal);
            return (month.getValue() - 1) / 3 + 1;
        };
    }
    
    public static void main(String[] args) {
        LocalDate date = LocalDate.of(2024, 2, 14);
        
        boolean leap = date.query(isLeapYear());
        System.out.println("Leap year: " + leap);  // true
        
        int lastDay = date.query(dayOfLastDayOfMonth());
        System.out.println("Last day: " + lastDay);  // 29
        
        int q = date.query(quarter());
        System.out.println("Quarter: " + q);  // 1
    }
}

// 4. 실무 패턴: 급여 지급일
public class PayrollScheduler {
    // 매월 마지막 영업일 (금요일 선호)
    public static TemporalAdjuster lastBusinessDayOfMonth() {
        return temporal -> {
            LocalDate date = LocalDate.from(temporal)
                .with(TemporalAdjusters.lastDayOfMonth());
            
            while (date.getDayOfWeek().getValue() > 5) {
                date = date.minusDays(1);
            }
            return date;
        };
    }
    
    public static void main(String[] args) {
        LocalDate[] months = {
            LocalDate.of(2024, 1, 1),
            LocalDate.of(2024, 2, 1),
            LocalDate.of(2024, 3, 1),
        };
        
        for (LocalDate month : months) {
            LocalDate payday = month.with(lastBusinessDayOfMonth());
            System.out.println(month.getMonth() + ": " + payday);
        }
        // JANUARY: 2024-01-31
        // FEBRUARY: 2024-02-29
        // MARCH: 2024-03-29
    }
}

// 5. 빌더 패턴과 함께
public class FluentDateCalculation {
    public static void main(String[] args) {
        LocalDate date = LocalDate.of(2024, 2, 14);
        
        LocalDate result = date
            .with(TemporalAdjusters.firstDayOfMonth())  // 2024-02-01
            .plus(1, ChronoUnit.MONTHS)                  // 2024-03-01
            .with(TemporalAdjusters.lastDayOfMonth());   // 2024-03-31
        
        System.out.println(result);  // 2024-03-31
    }
}
```

---

## 🔬 내부 동작 원리

### 1. TemporalAdjuster 인터페이스

```java
@FunctionalInterface
public interface TemporalAdjuster {
    Temporal adjustInto(Temporal temporal);
}

// TemporalAdjuster는 함수형 인터페이스
// → 람다로 구현 가능

// 사용 방식:
LocalDate date = LocalDate.now();
LocalDate adjusted = date.with(temporal -> {
    // 임시 수정 로직
    return temporal;
});

// 또는 메서드 참조
Temporal result = temporal.with(TemporalAdjusters::firstDayOfMonth);
```

### 2. TemporalAdjusters 라이브러리 구현

```java
public class TemporalAdjusters {
    // next(DayOfWeek) 구현
    public static TemporalAdjuster next(DayOfWeek dayOfWeek) {
        return temporal -> {
            DayOfWeek dow = DayOfWeek.from(temporal);
            int daysToAdd = (dayOfWeek.getValue() - dow.getValue() + 7) % 7;
            if (daysToAdd == 0) {
                daysToAdd = 7;  // 같은 요일이면 다음 주
            }
            return temporal.plus(daysToAdd, ChronoUnit.DAYS);
        };
    }
    
    // lastDayOfMonth 구현
    public static TemporalAdjuster lastDayOfMonth() {
        return temporal -> {
            YearMonth ym = YearMonth.from(temporal);
            return ym.atEndOfMonth();
        };
    }
    
    // dayOfWeekInMonth(n, dayOfWeek) 구현
    public static TemporalAdjuster dayOfWeekInMonth(int ordinal, DayOfWeek dayOfWeek) {
        return temporal -> {
            LocalDate date = LocalDate.from(temporal)
                .with(TemporalAdjusters.firstDayOfMonth())  // 월 초
                .with(TemporalAdjusters.nextOrSame(dayOfWeek));  // 첫 해당 요일
            
            for (int i = 1; i < ordinal; i++) {
                date = date.plus(1, ChronoUnit.WEEKS);  // +1주
            }
            
            return date;
        };
    }
}
```

### 3. TemporalQuery 인터페이스

```java
@FunctionalInterface
public interface TemporalQuery<R> {
    R queryFrom(TemporalAccessor temporal);
}

// 사용 방식:
LocalDate date = LocalDate.now();

// 표준 쿼리
Integer iso = date.query(ChronoField.YEAR::getFrom);

// 커스텀 쿼리 (람다)
Integer daysInMonth = date.query(temporal -> {
    YearMonth ym = YearMonth.from(temporal);
    return ym.lengthOfMonth();
});

// 또는 메서드 참조
TemporalQuery<Integer> daysQuery = YearMonth::lengthOfMonth;
```

### 4. with() vs plus() 메서드

```java
LocalDate date = LocalDate.of(2024, 2, 14);

// with(): TemporalAdjuster 적용
// → 날짜 필드 변경 (새 값으로 설정)
LocalDate adjusted = date.with(TemporalAdjusters.firstDayOfMonth());
// 2024-02-01 (월을 1로 변경)

// plus(): ChronoUnit으로 더하기
// → 날짜 더하기
LocalDate future = date.plus(7, ChronoUnit.DAYS);
// 2024-02-21

// 조합
LocalDate complex = date
    .with(TemporalAdjusters.lastDayOfMonth())  // 월말로 변경
    .plus(1, ChronoUnit.MONTHS);                // +1개월
// 2024-03-31
```

### 5. 체이닝 메커니즘

```java
public abstract class Temporal {
    public Temporal with(TemporalAdjuster adjuster) {
        return adjuster.adjustInto(this);
    }
    
    public Temporal with(TemporalField field, long newValue) {
        return field.adjustInto(this, newValue);
    }
    
    public Temporal plus(long amountToAdd, TemporalUnit unit) {
        return unit.addTo(this, amountToAdd);
    }
}

// 메서드 체인 = 메서드 합성
// date.with(a).with(b).plus(c)
// = date에 a 적용 → 결과에 b 적용 → 결과에 c 적용
```

---

## 💻 실전 실험

### 실험 1: TemporalAdjuster 성능 비교

```java
import java.time.*;
import java.time.temporal.*;

public class AdjusterPerformance {
    public static void main(String[] args) {
        LocalDate date = LocalDate.of(2024, 2, 14);
        
        // 직접 구현 vs TemporalAdjuster
        long t1 = System.nanoTime();
        for (int i = 0; i < 1_000_000; i++) {
            // 직접: 다음 월요일
            int dow = date.getDayOfWeek().getValue();
            date.plusDays((8 - dow) % 7);
        }
        long t2 = System.nanoTime();
        
        date = LocalDate.of(2024, 2, 14);
        long t3 = System.nanoTime();
        for (int i = 0; i < 1_000_000; i++) {
            // TemporalAdjuster
            date.with(TemporalAdjusters.nextOrSame(DayOfWeek.MONDAY));
        }
        long t4 = System.nanoTime();
        
        System.out.println("Direct: " + (t2 - t1) / 1_000_000.0 + " ms");
        System.out.println("Adjuster: " + (t4 - t3) / 1_000_000.0 + " ms");
        // 거의 같음 (캐시 덕분)
    }
}
```

### 실험 2: Custom TemporalAdjuster 체인

```java
import java.time.*;
import java.time.temporal.*;

public class CustomAdjusterChain {
    // 한국의 공휴일 제외
    static class KoreanCalendar {
        static boolean isHoliday(LocalDate date) {
            return (date.getMonth() == Month.JANUARY && date.getDayOfMonth() == 1) ||
                   (date.getMonth() == Month.MARCH && date.getDayOfMonth() == 1) ||
                   (date.getMonth() == Month.MAY && date.getDayOfMonth() == 5) ||
                   (date.getMonth() == Month.AUGUST && date.getDayOfMonth() == 15) ||
                   (date.getMonth() == Month.OCTOBER && date.getDayOfMonth() == 3) ||
                   (date.getMonth() == Month.DECEMBER && date.getDayOfMonth() == 25);
        }
    }
    
    static TemporalAdjuster nextWorkingDay() {
        return temporal -> {
            LocalDate date = LocalDate.from(temporal).plusDays(1);
            while (date.getDayOfWeek().getValue() > 5 || 
                   KoreanCalendar.isHoliday(date)) {
                date = date.plusDays(1);
            }
            return date;
        };
    }
    
    public static void main(String[] args) {
        LocalDate date = LocalDate.of(2024, 3, 1);  // 금요일, 독립운동일
        
        LocalDate nextDay = date.with(nextWorkingDay());
        System.out.println(nextDay);  // 2024-03-04 (월요일)
    }
}
```

### 실험 3: TemporalQuery 조합

```java
import java.time.*;
import java.time.temporal.*;

public class CombinedQueries {
    static TemporalQuery<String> dateInfo() {
        return temporal -> {
            LocalDate date = LocalDate.from(temporal);
            int daysInMonth = date.query(YearMonth::lengthOfMonth);
            boolean isLeap = date.query(temporal2 -> 
                Year.from(temporal2).isLeap());
            DayOfWeek dow = DayOfWeek.from(temporal);
            
            return String.format(
                "%s: %d days in month, leap=%s, %s",
                date, daysInMonth, isLeap, dow
            );
        };
    }
    
    public static void main(String[] args) {
        LocalDate date = LocalDate.of(2024, 2, 14);
        System.out.println(date.query(dateInfo()));
        // 2024-02-14: 29 days in month, leap=true, WEDNESDAY
    }
}
```

---

## 📊 성능/비교

| 방식 | 코드 길이 | 가독성 | 성능 | 확장성 |
|------|---------|--------|------|--------|
| **직접 계산** | 짧음 | 낮음 (매직넘버) | 빠름 | 나쁨 (복사/붙여넣기) |
| **Helper 메서드** | 중간 | 중간 (일관성 필요) | 빠름 | 나쁨 (메서드 증가) |
| **TemporalAdjuster** | 길음 | 높음 (의도 명확) | 같음 | 좋음 (재사용 가능) |
| **라이브러리 (TemporalAdjusters)** | 매우 짧음 | 매우 높음 | 같음 | 매우 좋음 |

---

## ⚖️ 트레이드오프

### Custom TemporalAdjuster vs 직접 계산

```
Custom 선택:
  + 재사용 가능
  + 테스트 가능
  + 의도가 명확
  - 보일러플레이트 코드 증가
  - 람다 vs 별도 클래스 선택 필요

직접 계산 선택:
  + 간단한 로직은 빠름
  - 버그 가능성 높음
  - 재사용 어려움
  - 일관성 문제

권장: 재사용 또는 복잡한 로직이면 Adjuster 사용
```

---

## 📌 핵심 정리

1. **TemporalAdjuster:**
   - 함수형 인터페이스: `Temporal adjustInto(Temporal)`
   - `with(adjuster)` 메서드로 적용
   - `TemporalAdjusters` 정적 팩토리 활용

2. **자주 쓰는 TemporalAdjusters:**
   - `next(dayOfWeek)`, `nextOrSame()`, `previous()`
   - `firstDayOfMonth()`, `lastDayOfMonth()`
   - `dayOfWeekInMonth(n, dayOfWeek)`

3. **TemporalQuery:**
   - 함수형 인터페이스: `R queryFrom(TemporalAccessor)`
   - `query(q)` 메서드로 정보 추출
   - 커스텀 쿼리는 람다로 간단히 구현

4. **체이닝:**
   - `with(adjuster1).with(adjuster2).plus(...)`로 복잡한 계산 표현
   - 함수형 프로그래밍 스타일

5. **성능:**
   - 직접 계산과 거의 같음 (캐시 덕분)
   - 가독성과 재사용성 이득이 큼

---

## 🤔 생각해볼 문제

**Q1: 다음 코드의 결과는?**
```java
LocalDate date = LocalDate.of(2024, 2, 14);  // 수요일

LocalDate result = date.with(
    TemporalAdjusters.next(DayOfWeek.WEDNESDAY)
);

System.out.println(result);  // ???
```

<details><summary>해설 보기</summary>

**답: 2024-02-21 (다음주 수요일)**

이유:
- `next(dayOfWeek)` = "다음" 해당 요일
- 오늘(2024-02-14)이 수요일이지만, "다음"이므로 7일 후

만약 "오늘 또는 다음"을 원했다면:
```java
LocalDate result = date.with(
    TemporalAdjusters.nextOrSame(DayOfWeek.WEDNESDAY)
);
// 2024-02-14 (오늘이 수요일이므로 자신 반환)
```

구현:
```java
public static TemporalAdjuster next(DayOfWeek dayOfWeek) {
    return temporal -> {
        DayOfWeek dow = DayOfWeek.from(temporal);
        int daysToAdd = (dayOfWeek.getValue() - dow.getValue() + 7) % 7;
        if (daysToAdd == 0) daysToAdd = 7;  // 같으면 7일 후
        return temporal.plus(daysToAdd, ChronoUnit.DAYS);
    };
}
```

</details>

**Q2: Custom TemporalAdjuster로 "2주 뒤 같은 요일"을 구현하려면?**

<details><summary>해설 보기</summary>

**정답:**
```java
TemporalAdjuster twoWeeksLater = temporal -> 
    temporal.plus(14, ChronoUnit.DAYS);

LocalDate date = LocalDate.of(2024, 2, 14);
LocalDate result = date.with(twoWeeksLater);
// 2024-02-28 (2주 뒤, 같은 수요일)
```

또는 ChronoUnit을 바로 쓸 수 있으므로:
```java
LocalDate result = date.plus(2, ChronoUnit.WEEKS);
// 같은 결과
```

TemporalAdjuster가 유용한 경우:
```java
TemporalAdjuster twoWeeksLaterButNotWeekend = temporal -> {
    LocalDate date = (LocalDate) temporal.plus(14, ChronoUnit.DAYS);
    while (date.getDayOfWeek().getValue() > 5) {
        date = date.minusDays(1);
    }
    return date;
};
// 복잡한 로직이므로 Adjuster로 감싸는 게 낫다
```

</details>

**Q3: TemporalQuery로 "연간 근무 일수 (주말/공휴일 제외)"를 구현하려면?**

<details><summary>해설 보기</summary>

**정답:**
```java
TemporalQuery<Integer> workingDaysInYear() {
    return temporal -> {
        Year year = Year.from(temporal);
        LocalDate jan1 = year.atDay(1);
        
        int count = 0;
        for (LocalDate date = jan1; date.getYear() == year.getValue();
             date = date.plusDays(1)) {
            if (date.getDayOfWeek().getValue() <= 5) {  // 평일만
                count++;
            }
        }
        return count;
    };
}

LocalDate date = LocalDate.of(2024, 1, 1);
Integer workingDays = date.query(workingDaysInYear());
// 2024: 253 근무일 (366 - 52×2 - 약 10 공휴일)
```

더 효율적인 방법:
```java
TemporalQuery<Integer> workingDaysEstimate() {
    return temporal -> {
        Year year = Year.from(temporal);
        int days = year.length();  // 365 또는 366
        int weekends = (days / 7) * 2;  // 주말
        if (days % 7 > 0) weekends++;  // 부분 주
        return days - weekends - 10;  // 공휴일 대략 10일
    };
}
```

</details>

---

<div align="center">

**[⬅️ 이전: 불변성 설계](./02-immutability-design.md)** | **[홈으로 🏠](../README.md)** | **[다음: Date/Calendar 마이그레이션 ➡️](./04-date-calendar-migration.md)**

</div>
