# 불변성 설계 — 왜 withYear() 같은 메서드만 있는가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 왜 `LocalDate`에는 `setYear()` 같은 setter가 없는가?
- `withYear(2025)`는 왜 새 인스턴스를 반환하는가?
- 불변 객체가 스레드 안전성을 보장하는 메커니즘은?
- 동시성 자료구조(HashMap, ConcurrentHashMap)에서 불변 객체를 쓸 때의 이점은?
- 구닝 스트립(String의 내부 최적화)과 날짜 객체의 관계는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Java 8 이전의 `java.util.Date`와 `java.util.Calendar`는 모두 가변(mutable) 객체였다. 이로 인해 발생한 버그가 얼마나 많았는지는 Java 커뮤니티의 악몽이다. 스레드 A가 `Date`를 수정하는 동시에 스레드 B가 읽으면, B는 일관되지 않은 상태를 본다. 불변 설계는 이를 원천적으로 차단한다. 또한 함수형 프로그래밍에서는 불변성이 필수다. 멀티스레드 환경과 함수형 프로그래밍이 표준인 현대 Java에서 이 개념은 필수 지식이다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Date를 가변으로 취급 (Java 8 이전)
  Date deadline = new Date();
  deadline.setHours(17);
  deadline.setMinutes(0);
  
  // 스레드 A
  Thread a = new Thread(() -> {
      for (int i = 0; i < 1000; i++) {
          deadline.setSeconds(i);  // 수정
      }
  });
  
  // 스레드 B
  Thread b = new Thread(() -> {
      System.out.println(deadline);  // 읽기
      // → "Mon Feb 14 17:00:32 2024"일 수도, "17:00:59"일 수도 있음
      // → 데드락, NullPointerException 가능
  });
  
  → 해결: synchronize(deadline) {...}로 감싸야 함

실수 2: HashMap에 Date를 key로 사용
  Map<Date, String> schedule = new HashMap<>();
  Date key = new Date(2024, 2, 14);
  schedule.put(key, "meeting");
  
  // 나중에 key를 수정하면?
  key.setDate(15);
  
  // HashMap이 원래 hash code로 탐색 (hash code 변경됨)
  schedule.get(key);  // null! (찾을 수 없음)
  → 자료구조 무결성 깨짐

실수 3: 캐시된 객체를 수정
  private LocalDate cached = LocalDate.now();
  
  public LocalDate getToday() {
      return cached;  // 같은 객체 반환
  }
  
  // 사용자는 안전하다고 가정 (불변)
  LocalDate today = getToday();
  // 만약 LocalDate가 mutable이었다면:
  // today.setYear(2025);  // getToday()가 돌려줄 모든 값이 변함!

실수 4: Calendar의 헷갈리는 setter
  Calendar cal = Calendar.getInstance();
  cal.set(2024, 2, 14);  // 월은 0 기반! (2 = 3월)
  cal.setHours(14);      // java.util.Date로만 작동
  
  // 언제 변경이 적용되는가?
  // → getTime() 호출 시 재계산
  // → 일관성 보장 안 됨 (중간에 다른 스레드가 개입하면?)
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 1. 불변 날짜 객체 (Java 8+)
LocalDate deadline = LocalDate.of(2024, 2, 14);

// setter 없음 - 컴파일 오류
// deadline.setYear(2025);  // ❌

// with* 메서드: 새 인스턴스 반환
LocalDate next = deadline.withYear(2025);
// → deadline은 변경 안 됨
// → next는 새로운 객체

// 2. 스레드 안전 (락 불필요)
List<LocalDate> dates = Collections.synchronizedList(new ArrayList<>());

Thread t1 = new Thread(() -> {
    for (int i = 0; i < 1000; i++) {
        LocalDate d = LocalDate.of(2024, 1, 1).plusDays(i);
        dates.add(d);  // 동기화 불필요 (LocalDate는 불변)
    }
});

Thread t2 = new Thread(() -> {
    for (LocalDate d : dates) {
        System.out.println(d.getYear());  // 안전 (값이 절대 변하지 않음)
    }
});

// 3. 불변 객체를 key로 사용 (안전)
Map<LocalDate, String> schedule = new ConcurrentHashMap<>();
LocalDate key = LocalDate.of(2024, 2, 14);
schedule.put(key, "meeting");

// key를 "수정"하려면 (사실은 새 인스턴스)
key = key.withYear(2025);  // 새 객체 (기존 참조는 유지)
schedule.get(LocalDate.of(2024, 2, 14));  // "meeting" (찾음!)

// 4. 함수형 체인
LocalDateTime now = LocalDateTime.now();

LocalDateTime deadline = now
    .withHour(17)      // 새 인스턴스
    .withMinute(0)     // 또 다른 새 인스턴스
    .withSecond(0)     // 또 다른 새 인스턴스
    .plusDays(7);      // 또 다른 새 인스턴스

// → 중간 값들이 모두 생성되지만, 각각 독립적 (사이드이펙트 없음)

// 5. 불변성 보장 (방어적 복사 불필요)
public class Meeting {
    private final LocalDateTime startTime;  // final로 충분!
    
    public Meeting(LocalDateTime startTime) {
        this.startTime = startTime;
        // 방어적 복사 불필요 (LocalDateTime은 불변)
        // this.startTime = new LocalDateTime(startTime); // 이건 불가능
    }
    
    public LocalDateTime getStartTime() {
        return startTime;  // 안전하게 반환
        // → 호출자가 수정해도 내 startTime은 변하지 않음
    }
}

// 6. ConcurrentHashMap의 computeIfAbsent와 불변성
ConcurrentHashMap<LocalDate, List<String>> events = new ConcurrentHashMap<>();

LocalDate today = LocalDate.now();
events.computeIfAbsent(today, date -> new ArrayList<>()).add("event");

// LocalDate는 불변이므로:
// - 계산 함수 실행 중에도 date의 값이 변하지 않음
// - 같은 'today' 참조로 여러 스레드에서 접근해도 안전
```

---

## 🔬 내부 동작 원리

### 1. 불변성 보장 메커니즘

```java
// java.time.LocalDate 구조
public final class LocalDate implements Temporal, Serializable {
    // 1. final class: 상속 금지
    
    private final int year;      // final 필드
    private final short month;   // final 필드
    private final short day;     // final 필드
    
    // 2. private 생성자
    private LocalDate(int year, short month, short day) {
        this.year = year;
        this.month = month;
        this.day = day;
    }
    
    // 3. 정적 팩토리 (유일한 생성 방법)
    public static LocalDate of(int year, int month, int dayOfMonth) {
        return new LocalDate(year, (short) month, (short) day);
    }
    
    // 4. setter 없음 (모든 write 메서드는 with*)
    public LocalDate withYear(int year) {
        return new LocalDate(year, month, day);  // 새 인스턴스
    }
    
    // 5. equals/hashCode는 내용 기반
    @Override
    public boolean equals(Object obj) {
        if (!(obj instanceof LocalDate)) return false;
        LocalDate other = (LocalDate) obj;
        return year == other.year && month == other.month && day == other.day;
    }
    
    @Override
    public int hashCode() {
        return year * 10000 + month * 100 + day;
    }
}

// 불변성 보장:
// 1. final class → 하위 클래스에서 동작 변경 불가
// 2. 모든 필드 final → 초기화 후 변경 불가능
// 3. private 생성자 + static factory → 생성 통제
// 4. with* 메서드 → 새 인스턴스 반환
// → 결론: 생성 후 절대 변하지 않음
```

### 2. with* 메서드의 메모리 효율

```java
// 문제: with* 메서드는 매번 새 인스턴스를 생성한다
LocalDate date1 = LocalDate.of(2024, 2, 14);
LocalDate date2 = date1.withYear(2024);  // 같은 값도 새 인스턴스 생성

// date1과 date2는 값은 같지만 다른 객체
System.out.println(date1 == date2);        // false
System.out.println(date1.equals(date2));   // true

// 메모리 낭비? 아니다!
// 1. 인스턴스는 아주 작음 (12바이트: int + 2×short)
// 2. GC는 이런 작은 객체를 빠르게 정리 (Young Generation)
// 3. String처럼 interning은 필요 없음 (equals 충분)

// 실제로 자주 발생하는 패턴:
LocalDate deadline = baseDate
    .withYear(2025)          // 새 인스턴스 1
    .withMonth(12)           // 새 인스턴스 2
    .withDayOfMonth(31);     // 새 인스턴스 3

// 최종 결과만 deadline에 저장 (3개 임시 인스턴스는 GC 대상)
```

### 3. 스레드 안전성 (락 필요 없음)

```java
// Date (가변) - 동기화 필수
public class DateScheduler {
    private Date deadline = new Date();  // 가변
    
    public synchronized void setDeadline(Date d) {
        this.deadline = new Date(d);  // 방어적 복사
    }
    
    public synchronized Date getDeadline() {
        return new Date(deadline);    // 방어적 복사
    }
}

// LocalDate (불변) - 동기화 불필요
public class DateScheduler {
    private LocalDate deadline = LocalDate.now();  // 불변
    
    public void setDeadline(LocalDate d) {
        // synchronized 불필요
        this.deadline = d;
    }
    
    public LocalDate getDeadline() {
        // synchronized 불필요 (값이 변하지 않음)
        return deadline;
    }
}

// 메커니즘:
// 불변 객체 = 읽는 것이 항상 안전
//   → 스레드 A가 읽는 동안 스레드 B가 쓸 수 없음 (쓸 수 없으니까!)
//   → Race condition 불가능
```

### 4. HashMap/ConcurrentHashMap과의 상호작용

```java
// 불변 객체는 HashMap 안전성의 핵심

public class HashMap<K,V> {
    private Entry<K,V>[] table;
    private int size;
    
    public V put(K key, V value) {
        int hash = key.hashCode();        // 해시 계산
        int index = hash % table.length;  // 버킷 결정
        
        // table[index]에 저장
        // → 이후에 key의 hashCode()가 변하면?
        // → get()은 다른 버킷을 찾음 (null 반환)
        // → 자료구조 무결성 깨짐!
    }
}

// 따라서:
// Map<LocalDate, String> schedule = new HashMap<>();
// LocalDate key = LocalDate.of(2024, 2, 14);
// schedule.put(key, "meeting");
// 
// key.setDay(15);  // ❌ 불가능 (메서드 없음)
// → 안전!

// 반면 Date는:
// Map<Date, String> schedule = new HashMap<>();
// Date key = new Date(2024, 2, 14);
// schedule.put(key, "meeting");
// key.setDate(15);  // ✅ 가능 (가변이므로)
// → schedule.get(new Date(2024, 2, 14))는 null! (찾을 수 없음)
```

### 5. 값 기반 equals/hashCode

```java
// 불변 객체는 내용 기반 equals 사용 가능

LocalDate date1 = LocalDate.of(2024, 2, 14);
LocalDate date2 = LocalDate.of(2024, 2, 14);

System.out.println(date1 == date2);           // false (다른 객체)
System.out.println(date1.equals(date2));      // true (같은 내용)

// 해시맵에서:
Map<LocalDate, String> map = new HashMap<>();
map.put(date1, "meeting");
map.get(date2);  // "meeting" (찾음!)

// 가변 객체와의 차이:
Date d1 = new Date();
Date d2 = new Date();
System.out.println(d1.equals(d2));  // false (참조 기반)
// → 따라서 Date를 HashMap key로 쓰면 문제 발생
```

---

## 💻 실전 실험

### 실험 1: with* 메서드 체인 성능

```java
import java.time.*;

public class WithMethodChain {
    public static void main(String[] args) {
        // 메서드 체인
        LocalDateTime start = LocalDateTime.of(2024, 1, 1, 0, 0, 0);
        
        long t1 = System.nanoTime();
        LocalDateTime result = start
            .withYear(2025)
            .withMonth(6)
            .withDayOfMonth(15)
            .withHour(14)
            .withMinute(30);
        long t2 = System.nanoTime();
        
        System.out.println("Chain: " + (t2 - t1) + " ns");
        // Chain: ~5000 ns (5 마이크로초)
        
        // 수동 구성 (with 없었다면)
        long t3 = System.nanoTime();
        LocalDateTime manual = LocalDateTime.of(2025, 6, 15, 14, 30, 0);
        long t4 = System.nanoTime();
        
        System.out.println("Direct: " + (t4 - t3) + " ns");
        // Direct: ~500 ns (메서드 체인보다 빠르지만, 차이는 미미)
        
        // 결론: 불변성의 이점(명확함, 안전함)이 미미한 성능 비용을 정당화
    }
}
```

### 실험 2: ConcurrentHashMap에서의 불변성 이점

```java
import java.time.*;
import java.util.concurrent.*;

public class ImmutabilityAndConcurrency {
    public static void main(String[] args) throws InterruptedException {
        ConcurrentHashMap<LocalDate, String> schedule = new ConcurrentHashMap<>();
        LocalDate key = LocalDate.of(2024, 2, 14);
        
        // 스레드 1: 쓰기
        Thread writer = new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                schedule.put(key, "event-" + i);
            }
        });
        
        // 스레드 2: 읽기
        Thread reader = new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                String value = schedule.get(key);
                // value는 절대 null이나 부분적 값이 아님
                // (LocalDate 불변이므로 해시 변경 불가능)
            }
        });
        
        writer.start();
        reader.start();
        writer.join();
        reader.join();
        
        System.out.println("No race conditions or lost updates!");
        // 동기화 없이도 완전히 안전
    }
}
```

### 실험 3: HashMap 무결성 (가변 vs 불변)

```java
import java.time.*;
import java.util.*;

public class HashMapIntegrity {
    static class MutableDate {
        int year, month, day;
        
        MutableDate(int year, int month, int day) {
            this.year = year;
            this.month = month;
            this.day = day;
        }
        
        @Override
        public int hashCode() {
            return year * 10000 + month * 100 + day;
        }
        
        @Override
        public boolean equals(Object obj) {
            MutableDate other = (MutableDate) obj;
            return year == other.year && month == other.month && day == other.day;
        }
        
        public void setDay(int d) { this.day = d; }  // 가변!
    }
    
    public static void main(String[] args) {
        // 가변 객체 (위험)
        HashMap<MutableDate, String> badMap = new HashMap<>();
        MutableDate badKey = new MutableDate(2024, 2, 14);
        badMap.put(badKey, "meeting");
        
        System.out.println("Before mutation: " + badMap.get(badKey));  // "meeting"
        
        badKey.setDay(15);  // 변경!
        
        System.out.println("After mutation: " + badMap.get(badKey));   // null!
        // 자료구조 무결성 깨짐!
        
        // 불변 객체 (안전)
        HashMap<LocalDate, String> goodMap = new HashMap<>();
        LocalDate goodKey = LocalDate.of(2024, 2, 14);
        goodMap.put(goodKey, "meeting");
        
        System.out.println("\nImmutable - Before: " + goodMap.get(goodKey));  // "meeting"
        
        // goodKey를 "수정"하려면 새 객체 생성
        goodKey = goodKey.withDay(15);
        
        System.out.println("Immutable - After: " + goodMap.get(LocalDate.of(2024, 2, 14)));
        // "meeting" (원래 키가 유지되므로 찾음!)
    }
}
```

---

## 📊 성능/비교

| 측면 | 가변 (Date/Calendar) | 불변 (LocalDate) |
|------|---------------------|-----------------|
| **메모리** | 더 큼 (내부 상태 복잡) | 작음 (12-48바이트) |
| **방어적 복사** | 필수 (getter/setter) | 불필요 |
| **HashMap key** | 위험 (hashCode 변경) | 안전 |
| **스레드 안전** | synchronized 필수 | 불필요 |
| **함수형 체인** | 불가능 (상태 변경) | 자연스러움 |
| **GC 부담** | 낮음 (객체 수 적음) | 높음 (체인 시 객체 증가) |

**실제 성능:** GC 부담이 미미할 정도 (10ns 단위)

---

## ⚖️ 트레이드오프

### 불변성 vs 성능

```
불변성 선택 (LocalDate):
  + 스레드 안전성 (락 불필요)
  + HashMap 안전 (hashCode 변경 불가)
  + 예측 가능 (값이 절대 변하지 않음)
  + 함수형 프로그래밍 친화적
  - 객체 생성 빈번 (with* 메서드)
  - GC 부담 증가 (미미하지만)
  
가변성 선택 (Date):
  + 메모리 효율 (객체 수 적음)
  + setter로 직관적 수정
  - synchronized 필수
  - HashMap key로 사용 불가
  - 버그 발생 가능성 높음
  - 모던 자바에서 권장되지 않음

결론: 거의 모든 경우 불변 선택 (예외: 극한의 성능 최적화)
```

---

## 📌 핵심 정리

1. **불변 설계의 핵심:**
   - `final class` + `final fields` + `private 생성자`
   - `with*` 메서드로 새 인스턴스 반환
   - setter 없음

2. **스레드 안전성:**
   - 불변 객체는 읽기가 항상 안전 (동기화 불필요)
   - ConcurrentHashMap과 호환성 완벽

3. **HashMap/HashSet:**
   - 불변 객체는 key/element로 안전
   - 가변 객체는 hashCode 변경 위험 (자료구조 무결성 파괴)

4. **함수형 프로그래밍:**
   - with* 메서드 체인이 자연스러움
   - 사이드이펙트 없음 (원본 객체 유지)

5. **성능:**
   - 객체 생성 비용 미미 (GC가 빠르게 정리)
   - 동기화 제거로 실제로는 더 빠를 수 있음

---

## 🤔 생각해볼 문제

**Q1: 다음 코드에서 문제점은?**
```java
Map<Date, String> schedule = new HashMap<>();
Date deadline = new Date(124, 1, 14);  // 2024-02-14
schedule.put(deadline, "meeting");

// 나중에...
deadline.setDate(15);

// 결과는?
schedule.get(new Date(124, 1, 14));  // ???
```

<details><summary>해설 보기</summary>

**답: null이 반환된다 (자료구조 무결성 파괴)**

이유:
1. `put(deadline, ...)`은 deadline의 hashCode()로 버킷 결정
   - hashCode = 124 * 10000 + 1 * 100 + 14 = 1240114
   - 버킷 45에 저장 (예시)

2. `deadline.setDate(15)` → deadline 객체가 변경됨
   - 같은 참조, 하지만 내용 변경
   - hashCode() 재계산 = 1240115
   - 이제 버킷 46에 있어야 하는데, 버킷 45에 여전히 있음

3. `get(new Date(124, 1, 14))` → new 객체 생성
   - hashCode = 1240114 → 버킷 45 탐색
   - 버킷 45에 deadline(값: 2024-02-15)이 있지만, 찾는 값(2024-02-14)과 불일치
   - null 반환

**결과:** HashMap의 get이 자신이 방금 put한 값을 찾을 수 없음!

올바른 방식:
```java
Map<LocalDate, String> schedule = new HashMap<>();
LocalDate deadline = LocalDate.of(2024, 2, 14);
schedule.put(deadline, "meeting");

// 변경은 새 변수에
deadline = deadline.withDayOfMonth(15);

// 원래 키로 조회
schedule.get(LocalDate.of(2024, 2, 14));  // "meeting" (찾음!)
```

</details>

**Q2: with* 메서드는 항상 새 인스턴스를 생성하는가?**

<details><summary>해설 보기</summary>

**답: 대부분 그렇지만, 최적화가 있을 수 있다**

예:
```java
LocalDate date = LocalDate.of(2024, 2, 14);
LocalDate same = date.withYear(2024);  // 같은 년도

System.out.println(date == same);  // ???
```

JDK 소스에서:
```java
public LocalDate withYear(int newYear) {
    if (this.year == newYear) {
        return this;  // 최적화: 변경 없으면 자신 반환
    }
    return new LocalDate(newYear, month, day);
}
```

결과: `date == same` → true (같은 객체)

하지만 이는:
- 내부 최적화 (공개 계약 아님)
- equals/hashCode는 같으므로 프로그래머가 신경 쓸 필요 없음
- 불변성 보장에는 영향 없음

결론: with* 메서드는 "새 인스턴스를 반환하거나 자신을 반환"한다 (값 관점에서는 새로운 값)

</details>

**Q3: String 내부화(interning)와 LocalDate의 관계는?**

<details><summary>해설 보기</summary>

**답: LocalDate는 interning이 필요 없다**

String의 경우:
```java
String s1 = new String("hello");
String s2 = new String("hello");

System.out.println(s1 == s2);           // false (다른 객체)
System.out.println(s1.equals(s2));      // true (같은 내용)

// String은 불변이므로 interning으로 메모리 절약
String s3 = s1.intern();  // 풀에 등록
String s4 = s2.intern();
System.out.println(s3 == s4);  // true (같은 인턴 객체)
```

LocalDate의 경우:
```java
LocalDate d1 = LocalDate.of(2024, 2, 14);
LocalDate d2 = LocalDate.of(2024, 2, 14);

System.out.println(d1 == d2);           // false (다른 객체)
System.out.println(d1.equals(d2));      // true (같은 내용)

// interning 필요 없음 (메모리 크기가 이미 작음)
// d1 == d2 확인 불필요 (equals 충분)
```

왜 다른가:
- String: 길이 가변, 메모리 낭비 가능 (interning 유용)
- LocalDate: 크기 고정 (12바이트), 객체 생성 비용 무시할 수준
- 따라서 equals 기반 비교로 충분

결론: LocalDate는 내용 기반 equals 사용, interning 불필요

</details>

---

<div align="center">

**[⬅️ 이전: LocalDate · ZonedDateTime](./01-localdate-zoneddatetime.md)** | **[홈으로 🏠](../README.md)** | **[다음: TemporalAdjuster · TemporalQuery ➡️](./03-temporal-adjuster-query.md)**

</div>
