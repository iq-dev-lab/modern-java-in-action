# LocalDate · LocalDateTime · ZonedDateTime · Instant — 의미와 사용 시점

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `LocalDate`는 타임존이 없는데, 언제 사용하고 언제 피해야 하는가?
- "회의는 2월 14일 오후 2시"와 "로켓 발사는 UTC 1709023200"은 각각 어떤 타입이어야 하는가?
- `ZonedDateTime`과 `OffsetDateTime`의 차이는 무엇인가?
- DB에 저장할 때 `LocalDateTime`을 쓰면 왜 위험한가?
- `Instant.now()`는 왜 UTC 타임라인의 한 점일까?

---

## 🔍 왜 이 개념이 실무에서 중요한가

시간 표현을 잘못 선택하면 국제 서비스에서 24시간 오차가 발생한다. 예: 한국(UTC+9) 자정을 `LocalDateTime.of(2024, 2, 14, 0, 0)`으로 저장하면, UTC 타임라인에서는 2월 13일 15:00이다. JPA 엔티티에서 타임존 정보 없이 저장된 시간을 읽으면, 시스템 타임존에 따라 다른 시간으로 해석될 수 있다. 국제 이벤트(SaaS, 결제, 로그 수집)에서는 반드시 `Instant` 또는 `ZonedDateTime`을 사용해야 한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: LocalDateTime을 DB에 저장
  // 회의 시간을 저장
  @Column LocalDateTime meetingTime = LocalDateTime.of(2024, 2, 14, 14, 0);
  // DB에 "2024-02-14 14:00:00" 저장
  // → 타임존 정보 없음
  // → 다른 타임존 클라이언트가 조회 시 "2024-02-14 14:00:00"을 자신의 로컬 시간으로 해석
  // → UTC-8 시스템에서는 14:00 = 현지 14:00 (실제로는 UTC 22:00인데!)

실수 2: 무조건 LocalDate 쓰기
  LocalDate birthday = LocalDate.of(1990, 6, 15);
  // 생일은 타임존 무관? 아니다!
  // "1990-06-15" = 어느 나라의 자정?
  // 국제선 비행기 탈 때: 출생지 자정과 현재지 자정이 다름
  // → 비자 검증 시스템이 나이를 잘못 계산할 수 있음

실수 3: Instant를 사용자 시간으로 표시
  Instant createdAt = Instant.now();  // 2024-02-14T22:30:45Z
  // "작성 시간: " + createdAt  // "2024-02-14T22:30:45Z"
  // 사용자는 UTC를 모르므로 혼란스러움
  // → UI에서는 ZonedDateTime으로 변환 필수

실수 4: ZonedDateTime 간 비교 오류
  ZonedDateTime seoul = ZonedDateTime.of(2024, 2, 14, 14, 0, 0, 0, 
      ZoneId.of("Asia/Seoul"));
  ZonedDateTime london = ZonedDateTime.of(2024, 2, 14, 5, 0, 0, 0, 
      ZoneId.of("Europe/London"));
  // 실제로 같은 시각인데
  seoul.equals(london)  // false ← 타임존이 다르면 equals 실패!
  // → 비교: isBefore(), isAfter(), compareTo() 사용
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 1. 회의 시간 (사람이 참석하는 로컬 이벤트)
// DB: Instant + 표시 시 사용자 타임존으로 변환

@Entity
public class MeetingRoom {
    @Id private Long id;
    
    // ✅ 저장: UTC 절대 시점
    @Column(name = "scheduled_time")
    private Instant scheduledTime;
    
    // 편의 메서드: 서울 시간으로 표시
    public LocalDateTime getSeoulTime() {
        return scheduledTime.atZone(ZoneId.of("Asia/Seoul"))
            .toLocalDateTime();
    }
    
    public static MeetingRoom book(LocalDateTime seoulTime) {
        // 입력: 사용자가 "14:00" 입력 (한국 시간)
        ZonedDateTime zoned = seoulTime.atZone(ZoneId.of("Asia/Seoul"));
        return new MeetingRoom(zoned.toInstant());
    }
}

// 2. 생일 (연/월/일만 중요, 시간 무관)
@Entity
public class Person {
    @Column(name = "birth_date")
    private LocalDate birthDate;  // ✅ 로컬 날짜만 저장
    
    public int getAge(LocalDate today) {
        // 나이 계산: 타임존 무관 (일만 비교)
        return Period.between(birthDate, today).getYears();
    }
}

// 3. 로그/이벤트 (전 세계 수집, UTC 기준)
@Entity
public class AuditLog {
    @Column(name = "event_time_utc")
    private Instant eventTime;  // ✅ Instant = UTC 타임라인의 한 점
    
    public static AuditLog create(String action) {
        return new AuditLog(Instant.now(), action);
    }
}

// 4. 타임존 인식 시간
public class EventScheduler {
    // 사용자별 선호 타임존
    private Map<String, ZoneId> userZones = new ConcurrentHashMap<>();
    
    public void scheduleForUser(String userId, LocalDateTime localTime) {
        ZoneId zone = userZones.get(userId);
        Instant when = localTime.atZone(zone).toInstant();
        // → 항상 Instant로 저장
    }
    
    public ZonedDateTime getTimeInUserZone(String userId, Instant time) {
        ZoneId zone = userZones.get(userId);
        return time.atZone(zone);
    }
}

// 5. 타임존 변환 체인
Instant utc = Instant.now();  // 2024-02-14T22:30:45Z

ZonedDateTime seoul = utc.atZone(ZoneId.of("Asia/Seoul"));
// 2024-02-15T07:30:45+09:00 (다음날!)

ZonedDateTime london = utc.atZone(ZoneId.of("Europe/London"));
// 2024-02-14T22:30:45Z (같은 날)

// 다시 로컬만 추출 (타임존 버림)
LocalDateTime seoulLocal = seoul.toLocalDateTime();
// 2024-02-15T07:30:45 (타임존 정보 없음)
```

---

## 🔬 내부 동작 원리

### 1. LocalDate — 타임존 없는 날짜

```java
// java.time.LocalDate 구조
public final class LocalDate implements Temporal, Serializable {
    private final int year;    // 1-999,999,999
    private final short month;  // 1-12
    private final short day;    // 1-31
    
    // 타임존 정보 없음!
    // 단순히 (year, month, day) 트리플
}

// 메모리상 표현: (2024, 2, 14) = 그냥 숫자
LocalDate d = LocalDate.of(2024, 2, 14);
// 내부: year=2024, month=2, day=14
// "한국 2024-02-14"인지 "뉴욕 2024-02-14"인지 구별 불가능
```

### 2. LocalDateTime — 타임존 없는 일시

```java
public final class LocalDateTime implements Temporal, Serializable {
    private final LocalDate date;
    private final LocalTime time;
    // LocalTime = (hour, minute, second, nanosecond)
}

// 예: 회의 시작 시간 "2024-02-14 14:00:00" (타임존 미지정)
LocalDateTime meeting = LocalDateTime.of(2024, 2, 14, 14, 0, 0);
// 문제: "오후 2시"가 어느 나라의 오후 2시?
```

### 3. ZonedDateTime — 타임존 포함 일시

```java
public final class ZonedDateTime implements Temporal, Serializable {
    private final LocalDateTime dateTime;
    private final ZoneOffset offset;      // ±HH:MM (예: +09:00)
    private final ZoneId zone;            // 규칙 기반 (예: Asia/Seoul)
}

// 한국의 2024-02-14 14:00:00
ZonedDateTime seoul = ZonedDateTime.of(
    LocalDateTime.of(2024, 2, 14, 14, 0, 0),
    ZoneId.of("Asia/Seoul")
);
// 내부:
//   dateTime = 2024-02-14 14:00:00
//   zone = Asia/Seoul
//   offset = +09:00 (현재)

// UTC로 변환 (Instant 추출)
Instant utc = seoul.toInstant();
// 2024-02-14 05:00:00 UTC (14:00 KST - 9시간)

// OffsetDateTime과의 차이
OffsetDateTime fixed = OffsetDateTime.of(
    LocalDateTime.of(2024, 2, 14, 14, 0, 0),
    ZoneOffset.of("+09:00")
);
// 차이:
//   ZoneId: 규칙 기반 (서머타임 자동 조정)
//   ZoneOffset: 고정된 UTC 오프셋 (± 조정 없음)
```

### 4. Instant — UTC 타임라인의 한 점

```java
public final class Instant implements Temporal, Serializable {
    private final long seconds;       // 에포크 이후 초 (1970-01-01T00:00:00Z)
    private final int nanos;          // 0-999,999,999
}

// Instant.now() = 현재 UTC 시각
Instant now = Instant.now();
// 내부: seconds=1708058445, nanos=123456789
// → 전 세계 모든 사람이 같은 값 (타임존 무관)

// Unix 타임스탬프와 호환
long epochSecond = now.getEpochSecond();
// 1708058445 (1970-01-01T00:00:00Z 이후 초)

// 계산에 유리
Instant tenSecondsLater = now.plusSeconds(10);
```

### 5. 변환 체인 (메모리 효율)

```
LocalDateTime (타임존 없음)
    ↓ atZone(ZoneId)
ZonedDateTime (타임존 포함)
    ↓ toInstant()
Instant (UTC 절대 시점)
    ↓ atZone(ZoneId)
ZonedDateTime (다른 타임존)
    ↓ toLocalDateTime()
LocalDateTime (타임존 없음)

// 주의: LocalDateTime → Instant 직접 변환 불가능
// Instant ist = LocalDateTime.now().toInstant();  // 컴파일 오류
// → 반드시 atZone() 거쳐야 함
```

---

## 💻 실전 실험

### 실험 1: 타임존 변환에 따른 시간 차이

```java
import java.time.*;

public class TimezoneExperiment {
    public static void main(String[] args) {
        // 한 순간 (UTC)
        Instant universal = Instant.parse("2024-02-14T22:30:45Z");
        
        // 같은 순간을 여러 타임존에서
        ZoneId seoul = ZoneId.of("Asia/Seoul");
        ZoneId london = ZoneId.of("Europe/London");
        ZoneId newyork = ZoneId.of("America/New_York");
        
        System.out.println("UTC: " + universal);
        // UTC: 2024-02-14T22:30:45Z
        
        System.out.println("Seoul: " + universal.atZone(seoul));
        // Seoul: 2024-02-15T07:30:45+09:00 (다음날!)
        
        System.out.println("London: " + universal.atZone(london));
        // London: 2024-02-14T22:30:45Z (같은 시간)
        
        System.out.println("New York: " + universal.atZone(newyork));
        // New York: 2024-02-14T17:30:45-05:00 (이전 시간)
        
        // 지역별 로컬 시간만 비교 (타임존 버림)
        LocalDateTime seoulLocal = universal.atZone(seoul).toLocalDateTime();
        LocalDateTime londonLocal = universal.atZone(london).toLocalDateTime();
        
        System.out.println("seoulLocal.getDayOfMonth() = " + 
            seoulLocal.getDayOfMonth());  // 15
        System.out.println("londonLocal.getDayOfMonth() = " + 
            londonLocal.getDayOfMonth());  // 14
    }
}
```

### 실험 2: DB 저장 시 타임존 손실

```java
import javax.persistence.*;
import java.time.*;

@Entity
@Table(name = "events")
public class EventBad {
    @Id private Long id;
    
    // 위험: 타임존 정보 없음
    @Column(name = "event_time")
    private LocalDateTime eventTime;
    
    // 저장: LocalDateTime.of(2024, 2, 14, 14, 0)
    // DB에 저장된 값: "2024-02-14 14:00:00"
    
    // 문제: 어느 타임존의 14:00인가?
    // → JVM 기본 타임존에서 해석됨
    // → JVM이 UTC 가정 시: "2024-02-14 14:00:00"은 UTC 14:00
    // → 실제로는 서울 14:00 (= UTC 05:00)이었으므로 9시간 오차!
}

@Entity
@Table(name = "events_good")
public class EventGood {
    @Id private Long id;
    
    // 올바름: Instant 저장 (항상 UTC)
    @Column(name = "event_time_utc")
    private Instant eventTime;
    
    // 저장: Instant.parse("2024-02-14T05:00:00Z")
    // DB에 저장된 값: "2024-02-14 05:00:00" (UTC)
    
    // 어느 타임존에서든 정확히 복원 가능
    public LocalDateTime getSeoulTime() {
        return eventTime.atZone(ZoneId.of("Asia/Seoul"))
            .toLocalDateTime();  // 2024-02-14 14:00:00
    }
}
```

### 실험 3: equals() vs compareTo()

```java
public class ZonedDateTimeComparison {
    public static void main(String[] args) {
        ZonedDateTime seoul = ZonedDateTime.of(
            LocalDateTime.of(2024, 2, 14, 14, 0),
            ZoneId.of("Asia/Seoul")
        );
        
        ZonedDateTime london = ZonedDateTime.of(
            LocalDateTime.of(2024, 2, 14, 5, 0),
            ZoneId.of("Europe/London")
        );
        
        // 실제로는 같은 시각 (UTC 2024-02-14T05:00:00Z)
        System.out.println("seoul.toInstant() = " + seoul.toInstant());
        // seoul.toInstant() = 2024-02-14T05:00:00Z
        
        System.out.println("london.toInstant() = " + london.toInstant());
        // london.toInstant() = 2024-02-14T05:00:00Z
        
        // equals()는 타임존까지 비교 (false)
        System.out.println("seoul.equals(london) = " + seoul.equals(london));
        // seoul.equals(london) = false (타임존 다름)
        
        // Instant로 변환 후 비교 (true)
        System.out.println("seoul.toInstant().equals(london.toInstant()) = " +
            seoul.toInstant().equals(london.toInstant()));
        // seoul.toInstant().equals(london.toInstant()) = true
        
        // compareTo()도 Instant로 비교
        System.out.println("seoul.compareTo(london) = " + seoul.compareTo(london));
        // seoul.compareTo(london) = 0 (같은 시각)
    }
}
```

---

## 📊 성능/비교

| 타입 | 메모리 | 변환비용 | 사용 시점 | 주의사항 |
|------|--------|---------|---------|---------|
| **LocalDate** | 12바이트 (int+2×short) | 없음 | 생일, 휴일 | 타임존 없음 = 해석 모호 |
| **LocalDateTime** | 24바이트 (LocalDate+LocalTime) | 없음 | UI 입력/출력 | DB 저장 위험 |
| **ZonedDateTime** | 48바이트 (LocalDateTime+ZoneId+ZoneOffset) | 높음 (규칙 계산) | 사용자 시간 표시 | DST 자동 조정 |
| **OffsetDateTime** | 40바이트 (LocalDateTime+ZoneOffset) | 없음 | 고정 오프셋 | DST 자동 조정 안 함 |
| **Instant** | 16바이트 (long+int) | 낮음 | DB 저장, 로그, 비교 | UTC만 가능 (변환 필요) |

**성능 팁:**
- Instant 간 비교: O(1) (숫자 비교)
- ZonedDateTime 간 비교: O(1) (Instant로 자동 변환)
- LocalDateTime 간 비교: O(1) (숫자 비교, 타임존 무관)
- ZoneId 규칙 계산: O(log n) (캐시됨)

---

## ⚖️ 트레이드오프

### LocalDate vs ZonedDateTime for 생일

```
LocalDate 선택:
  + 메모리 효율적 (12바이트)
  + 계산 빠름 (타임존 규칙 없음)
  - "어느 나라의 생일?"은 항상 자국 기준
  
ZonedDateTime 선택:
  + "생일이 어느 타임존인가"명확
  - 메모리 오버헤드
  - 거의 필요 없음 (생일은 로컬 날짜만 중요)
  
결론: LocalDate 권장 (타임존 없어도 의미 명확)
```

### Instant vs ZonedDateTime for 로그

```
Instant 선택:
  + DB 저장 간단 (UTC 절대값)
  + 메모리 효율 (16바이트)
  + 정렬/비교 빠름
  - UI에서 읽기 어려움 (사용자가 UTC 모름)
  
ZonedDateTime 선택:
  + UI 친화적 (사용자 시간대로 표시)
  - DB 저장 복잡 (오프셋/DST 처리)
  - 정렬/비교 느림
  
결론: DB는 Instant, UI는 atZone() 변환 권장
```

---

## 📌 핵심 정리

1. **LocalDate** = (year, month, day) 숫자 트리플, 타임존 없음
   - 생일, 휴일, 계약 종료일 등 "날짜"만 필요한 경우
   - 타임존 변환 불가능

2. **LocalDateTime** = LocalDate + LocalTime, 타임존 없음
   - UI 입력/출력, 사용자가 입력한 시간
   - DB 저장하면 위험 (타임존 정보 손실)

3. **ZonedDateTime** = LocalDateTime + ZoneId, 타임존 포함
   - 사용자별 선호 시간대 표시
   - DST 자동 조정, 메모리 오버헤드

4. **Instant** = UTC 절대 시점, 에포크 이후 나노초
   - DB 저장 (모든 시간값), 로그, 이벤트 타임스탬프
   - 전 세계 동일, 변환 필요 시 atZone() 사용

5. **OffsetDateTime** = LocalDateTime + ZoneOffset (고정 ±)
   - 고정된 UTC 오프셋 필요한 경우 (DST 없는 시스템)
   - 일반적으로 Instant 또는 ZonedDateTime 권장

---

## 🤔 생각해볼 문제

**Q1: 국제 쇼핑몰에서 주문 시간을 저장할 때, 다음 중 무엇을 선택해야 하는가?**
- 선택지: (A) LocalDateTime, (B) Instant, (C) ZonedDateTime, (D) LocalDate
- 요구사항: 한국 고객이 오후 3시에 주문 → 서버는 UTC 기준 저장 → 고객 대시보드에는 자신의 시간대로 표시

<details><summary>해설 보기</summary>

**정답: (B) Instant**

이유:
- 주문은 "전 세계 어느 시점에 발생했는가"가 중요
- DB: Instant로 저장 (UTC 절대값)
- 고객 대시보드: `Instant.atZone(userTimeZone).toLocalDateTime()`으로 변환
- (A) LocalDateTime: 타임존 정보 없어 9시간 오차 가능
- (C) ZonedDateTime: 복잡, DB 직렬화 문제
- (D) LocalDate: 시간 정보 손실

코드:
```java
@Entity
public class Order {
    @Column(name = "created_at_utc")
    private Instant createdAt;  // UTC 저장
    
    public LocalDateTime getCustomerTime(ZoneId zone) {
        return createdAt.atZone(zone).toLocalDateTime();
    }
}
```

</details>

**Q2: 다음 두 ZonedDateTime이 같은 시각을 나타내는가?**
```java
ZonedDateTime a = ZonedDateTime.of(
    LocalDateTime.of(2024, 3, 10, 2, 0),
    ZoneId.of("America/New_York")
);  // DST 전환 시점 (시간이 건너뜀)

ZonedDateTime b = ZonedDateTime.of(
    LocalDateTime.of(2024, 3, 10, 3, 0),
    ZoneId.of("America/New_York")
);  // DST 후
```

<details><summary>해설 보기</summary>

**답: 아니다. `a`는 존재하지 않는 시간이다.**

미국 동부시간 DST (매년 3월 두 번째 일요일):
- 02:00 → 03:00으로 건너뜀
- 02:00-02:59는 존재하지 않는 시간

> 💡 지역별 DST 전환 규칙이 다르다. 예: EU·영국(`Europe/London`)은 3월 마지막 일요일 01:00 UTC에 전환.
> JDK는 IANA TZ 데이터베이스(`tzdata`)로 자동 처리하므로 직접 계산할 필요 없음.

`ZonedDateTime.of()`는 유효하지 않은 시간을 입력하면:
- 1시간 앞당김 (03:00으로 조정)
- 또는 throw 옵션 사용

```java
ZonedDateTime a = ZonedDateTime.of(
    LocalDateTime.of(2024, 3, 10, 2, 0),
    ZoneId.of("America/New_York"),
    ZoneOffset.of("-04:00")  // 명시적 오프셋
);  // 아직도 모호함

// 올바른 방법: ZonedDateTime.of() 대신 atZone()
ZonedDateTime correct = LocalDateTime.of(2024, 3, 10, 3, 0)
    .atZone(ZoneId.of("America/New_York"));
// 03:00 EDT (-04:00)
```

</details>

**Q3: 다음 JPA 엔티티에서 뭐가 문제인가?**
```java
@Entity
public class ConferenceRoom {
    @Column(name = "reserved_from")
    private LocalDateTime reservedFrom;
    
    @Column(name = "reserved_to")
    private LocalDateTime reservedTo;
}
```

<details><summary>해설 보기</summary>

**문제: 타임존 정보 없음**

시나리오:
- 서울 개발자: "2024-02-14 14:00~15:00" 예약 입력
- 도쿄에서 조회: "2024-02-14 14:00" = 도쿄 14:00? 서울 14:00?
- 뉴욕에서 조회: "2024-02-14 14:00" = 뉴욕 14:00?

결과: 회의실 예약 충돌, 더블 북킹

**올바른 설계:**
```java
@Entity
public class ConferenceRoom {
    @Column(name = "reserved_from_utc")
    private Instant reservedFrom;  // UTC 저장
    
    @Column(name = "reserved_to_utc")
    private Instant reservedTo;
    
    @Column(name = "room_timezone")
    private String roomTimezone;  // "Asia/Seoul"
    
    // UI 표시용
    public LocalDateTime getReservedFromLocal() {
        return reservedFrom.atZone(ZoneId.of(roomTimezone))
            .toLocalDateTime();
    }
}
```

</details>

---

<div align="center">

**[⬅️ 이전 챕터: Sealed Interface](../chapter06-interface-evolution/05-sealed-interface.md)** | **[홈으로 🏠](../README.md)** | **[다음: 불변성 설계 ➡️](./02-immutability-design.md)**

</div>
