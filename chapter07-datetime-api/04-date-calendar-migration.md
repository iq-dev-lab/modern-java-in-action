# Date/Calendar → New API 마이그레이션 전략

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `java.util.Date`와 `Instant` 간 변환 API는?
- `Calendar.to/from Instant` 변환의 함정은?
- JPA 엔티티에서 `@Temporal` 대신 `LocalDateTime`을 쓸 때의 주의사항은?
- Jackson에서 날짜 필드를 직렬화할 때 타임존 문제는 어떻게 처리하는가?
- 레거시 코드와의 경계에서 변환을 어디서 할 것인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Java 8 이전의 레거시 코드는 여전히 많다. `java.util.Date`, `Calendar`, Joda-Time 등이 섞여 있을 수 있다. 마이그레이션은 한 번에 완료되지 않으므로, 경계에서의 변환이 중요하다. 특히 JPA와 JSON 직렬화는 복잡함. 잘못된 변환은 타임존 오류(24시간 차이)를 만든다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: java.util.Date를 그대로 LocalDate로 변환
  Date oldDate = new Date();  // 2024-02-14 22:30:45 (UTC)
  LocalDate newDate = oldDate.toInstant().atZone(ZoneId.systemDefault()).toLocalDate();
  // → SystemDefault가 UTC가 아니면 날짜가 달라짐!
  // → 2024-02-14 22:30:45 UTC → 2024-02-15 07:30 (KST) → LocalDate 2024-02-15
  // → 원래 Date의 날짜(14일)와 다른 날짜(15일)로 변환됨!

실수 2: Calendar를 LocalDateTime으로 변환
  Calendar cal = Calendar.getInstance();
  cal.set(2024, 1, 14);  // 월은 0 기반! (1 = 2월)
  
  LocalDateTime ldt = cal.toInstant()  // ← 어라, 시간대는?
      .atZone(ZoneId.systemDefault())
      .toLocalDateTime();
  // → Calendar의 타임존과 systemDefault가 다르면 오류!

실수 3: JPA에서 LocalDateTime을 @Temporal로 표시
  @Entity
  public class Event {
      @Temporal(TemporalType.TIMESTAMP)
      private LocalDateTime eventTime;  // ← 타입 불일치 (LocalDateTime은 @Temporal 불필요)
  }
  // → Hibernate가 자동 변환 시도, 타임존 정보 손실 가능

실수 4: JSON 직렬화에서 타임존 무시
  @Entity
  public class Order {
      @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
      private LocalDateTime createdAt;
  }
  
  // JSON 응답: {"createdAt": "2024-02-14 22:30:45"}
  // → 클라이언트 타임존이 모름, 해석 불가능

실수 5: Joda-Time과 java.time 혼합
  import org.joda.time.DateTime;
  import java.time.LocalDateTime;
  
  // 변환 없이 섞임
  DateTime joda = new DateTime();
  LocalDateTime java8 = joda.toLocalDateTime();  // 메서드 자체 없음!
  // → 컴파일 오류

실수 6: 레거시 API 호출 시 반복 변환
  public LocalDateTime processOrder(Order order) {
      Date legacyDate = order.getCreatedDate();
      
      // 함수 호출 시마다 변환
      Instant instant = legacyDate.toInstant();
      LocalDateTime ldt = instant.atZone(ZoneId.systemDefault()).toLocalDateTime();
      
      // 다시 Date로 변환해서 반환
      return ldt;  // ← 여러 번 변환 반복
  }
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
import java.time.*;
import java.time.temporal.ChronoUnit;
import javax.persistence.*;
import com.fasterxml.jackson.annotation.*;

// 1. java.util.Date ↔ Instant 변환
public class DateConversion {
    // Date → Instant (안전)
    public static Instant dateToInstant(java.util.Date date) {
        return date == null ? null : date.toInstant();
    }
    
    // Instant → Date (안전)
    public static java.util.Date instantToDate(Instant instant) {
        return instant == null ? null : java.util.Date.from(instant);
    }
    
    // Date → LocalDateTime (타임존 명시)
    public static LocalDateTime dateToLocalDateTime(
        java.util.Date date, ZoneId zoneId) {
        if (date == null) return null;
        return date.toInstant()
            .atZone(zoneId)
            .toLocalDateTime();
    }
    
    public static void main(String[] args) {
        java.util.Date oldDate = new java.util.Date();
        
        // 절대값은 안전 (타임존 무관)
        Instant instant = oldDate.toInstant();
        
        // 로컬 날짜 필요 시: 타임존 명시
        LocalDateTime ldt = dateToLocalDateTime(
            oldDate, 
            ZoneId.of("Asia/Seoul")
        );
        
        // 역변환
        java.util.Date restored = java.util.Date.from(instant);
    }
}

// 2. Calendar ↔ Instant 변환
public class CalendarConversion {
    public static Instant calendarToInstant(Calendar cal) {
        return cal == null ? null : cal.toInstant();
    }
    
    public static Calendar instantToCalendar(Instant instant) {
        if (instant == null) return null;
        Calendar cal = Calendar.getInstance();
        cal.setTime(java.util.Date.from(instant));
        return cal;
    }
    
    // Calendar → LocalDateTime (타임존 유지)
    public static LocalDateTime calendarToLocalDateTime(Calendar cal) {
        if (cal == null) return null;
        
        // Calendar의 타임존 추출
        ZoneId zoneId = cal.getTimeZone().toZoneId();
        
        // Instant → ZonedDateTime → LocalDateTime
        return cal.toInstant()
            .atZone(zoneId)
            .toLocalDateTime();
    }
    
    public static void main(String[] args) {
        Calendar cal = Calendar.getInstance();
        
        Instant instant = calendarToInstant(cal);
        LocalDateTime ldt = calendarToLocalDateTime(cal);
    }
}

// 3. JPA 마이그레이션
@Entity
@Table(name = "orders")
public class Order {
    @Id private Long id;
    
    // ❌ 레거시 (타임존 정보 손실 위험)
    // @Temporal(TemporalType.TIMESTAMP)
    // private Date createdDate;
    
    // ✅ 신규 표준: UTC 절대값
    @Column(name = "created_at_utc")
    private Instant createdAt;
    
    // ✅ 신규 선택: 타임존 포함 표시
    // @Column(name = "event_time")
    // private ZonedDateTime eventTime;
    
    // 편의 메서드: 특정 타임존으로 표시
    public LocalDateTime getCreatedAtInSeoul() {
        return createdAt.atZone(ZoneId.of("Asia/Seoul"))
            .toLocalDateTime();
    }
    
    // 레거시 코드와의 호환성: getter
    public java.util.Date getCreatedDate() {
        return java.util.Date.from(createdAt);
    }
    
    // 레거시 코드와의 호환성: setter
    public void setCreatedDate(java.util.Date date) {
        this.createdAt = date.toInstant();
    }
}

// 4. Jackson 직렬화/역직렬화
@Entity
public class Invoice {
    @Id private Long id;
    
    // Instant는 자동으로 ISO-8601 UTC로 직렬화
    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'")
    private Instant issuedAt;
    
    // LocalDateTime은 타임존 정보 없으므로 명시 필수
    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd HH:mm:ss")
    @JsonSerialize(using = LocalDateTimeSerializer.class)
    @JsonDeserialize(using = LocalDateTimeDeserializer.class)
    private LocalDateTime dueDate;
    
    // ZonedDateTime은 타임존과 함께 직렬화
    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd'T'HH:mm:ss.SSSXXX")
    private ZonedDateTime dueDateWithZone;
}

// 커스텀 Jackson 직렬화 (타임존 명시)
public class LocalDateTimeSerializer extends StdSerializer<LocalDateTime> {
    private final ZoneId zoneId;
    
    public LocalDateTimeSerializer(ZoneId zoneId) {
        super(LocalDateTime.class);
        this.zoneId = zoneId;
    }
    
    @Override
    public void serialize(LocalDateTime value, JsonGenerator gen, SerializerProvider provider) 
            throws IOException {
        // LocalDateTime을 특정 타임존으로 해석해서 Instant로 변환
        Instant instant = value.atZone(zoneId).toInstant();
        gen.writeString(instant.toString());
    }
}

// 5. 마이그레이션 경계 (Facade)
public class DateTimeFacade {
    // 레거시 API 호출 지점을 한 곳에 모음
    
    private static final ZoneId SERVER_ZONE = ZoneId.of("Asia/Seoul");
    
    // 레거시 Date ← 신규 Instant
    public static java.util.Date toDate(Instant instant) {
        return java.util.Date.from(instant);
    }
    
    // 신규 Instant ← 레거시 Date
    public static Instant toInstant(java.util.Date date) {
        return date.toInstant();
    }
    
    // 신규 LocalDateTime ← 레거시 Date (타임존 명시)
    public static LocalDateTime toLocalDateTime(java.util.Date date) {
        return toInstant(date)
            .atZone(SERVER_ZONE)
            .toLocalDateTime();
    }
    
    // 레거시 Date ← 신규 LocalDateTime (역변환)
    public static java.util.Date fromLocalDateTime(LocalDateTime ldt) {
        return java.util.Date.from(
            ldt.atZone(SERVER_ZONE).toInstant()
        );
    }
}

// 6. Joda-Time 마이그레이션
public class JodaTimeMigration {
    // Joda-Time DateTime ↔ java.time
    public static Instant fromJodaDateTime(org.joda.time.DateTime jodaDateTime) {
        return jodaDateTime.toGregorianCalendar().toInstant();
    }
    
    public static org.joda.time.DateTime toJodaDateTime(Instant instant) {
        return new org.joda.time.DateTime(java.util.Date.from(instant));
    }
    
    // 또는 라이브러리: joda-time-jsoda 사용
}

// 7. 점진적 마이그레이션 전략
public class GradualMigration {
    
    // Phase 1: DB 스키마는 그대로, 엔티티만 변환
    @Entity
    public class LegacyOrder {
        @Column(name = "created_date")
        private Instant createdAt;  // Instant로 읽지만, DB는 여전히 DATE
        
        // Hibernate 타입 매퍼가 자동 변환
        @Converter(autoApply = true)
        public static class InstantConverter implements AttributeConverter<Instant, java.sql.Timestamp> {
            @Override
            public java.sql.Timestamp convertToDatabaseColumn(Instant instant) {
                return instant == null ? null : java.sql.Timestamp.from(instant);
            }
            
            @Override
            public Instant convertToEntityAttribute(java.sql.Timestamp timestamp) {
                return timestamp == null ? null : timestamp.toInstant();
            }
        }
    }
    
    // Phase 2: 새로운 코드는 Instant/LocalDateTime만 사용
    // Phase 3: DB 컬럼 타입 변경 (DATETIME → TIMESTAMP UTC)
    // Phase 4: 레거시 호환성 제거
}

// 8. 외부 API와의 연동 (타임존 불일치 처리)
public class ExternalApiIntegration {
    
    // 외부 API가 ISO-8601 반환 (UTC)
    public class ExternalResponse {
        @JsonProperty("timestamp")
        @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss'Z'")
        private Instant timestamp;
    }
    
    // 외부 API가 Unix 타임스탬프 반환
    public class UnixTimestampResponse {
        @JsonProperty("created_at")
        private long unixSeconds;
        
        public Instant toInstant() {
            return Instant.ofEpochSecond(unixSeconds);
        }
    }
    
    // 외부 API가 로컬 날짜 반환 (타임존 미지정!)
    public class AmbiguousDateResponse {
        @JsonProperty("date")
        private String dateStr;  // "2024-02-14 14:30:00"
        
        public Instant toInstant(ZoneId assumedZone) {
            LocalDateTime ldt = LocalDateTime.parse(dateStr, 
                DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
            return ldt.atZone(assumedZone).toInstant();
        }
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Date.toInstant() 메커니즘

```java
// java.util.Date.toInstant() 구현
public class Date implements Serializable {
    // 내부: long time (밀리초, 1970-01-01T00:00:00Z 이후)
    
    public Instant toInstant() {
        return Instant.ofEpochMilli(this.time);
    }
}

// Date는 정확히 Instant의 밀리초 단위 표현
Date date = new Date();  // 밀리초 정밀도
Instant instant = date.toInstant();  // 나노초 정밀도 (0 패딩)
```

### 2. Calendar.toInstant() 함정

```java
// Calendar는 내부에 타임존 정보 보유
Calendar cal = Calendar.getInstance();  // 시스템 기본 타임존
// 또는
Calendar cal = Calendar.getInstance(TimeZone.getTimeZone("UTC"));

// toInstant()는 Calendar의 타임존을 무시하고 절대값 반환
Instant i1 = cal.toInstant();

// 즉, Calendar의 타임존은 getTime() 호출 시만 사용됨
Date d = cal.getTime();  // ← 타임존 적용
Instant i2 = d.toInstant();  // i1 == i2 (항상 같은 Instant)
```

### 3. LocalDateTime의 "해석" 문제

```java
LocalDateTime ldt = LocalDateTime.of(2024, 2, 14, 14, 0, 0);

// ldt를 Instant로 변환하려면: 어느 타임존?
// (1) UTC라고 가정?
Instant i1 = ldt.atZone(ZoneId.of("UTC")).toInstant();
// i1 = 2024-02-14T14:00:00Z

// (2) 서울 시간이라고 가정?
Instant i2 = ldt.atZone(ZoneId.of("Asia/Seoul")).toInstant();
// i2 = 2024-02-14T05:00:00Z (14:00 KST - 9시간)

// → 같은 LocalDateTime이지만 다른 Instant!
// → atZone()에서 타임존을 반드시 명시해야 함
```

### 4. JPA @Temporal과의 호환성

```java
@Entity
public class Event {
    // 방식 1: java.util.Date with @Temporal
    @Temporal(TemporalType.TIMESTAMP)
    private Date eventDate;
    // → Hibernate: DATE 컬럼에 저장, 읽을 때 Date로 변환
    
    // 방식 2: LocalDateTime with @Temporal (권장 안 함)
    @Temporal(TemporalType.TIMESTAMP)
    private LocalDateTime eventTime;  // 타입 불일치
    // → Hibernate 자동 변환: LocalDateTime → Date → Instant
    // → 타임존 정보 손실 가능
    
    // 방식 3: Instant (권장)
    private Instant eventInstant;  // @Temporal 불필요
    // → Hibernate: 자동으로 TIMESTAMP UTC로 처리
    
    // 방식 4: 커스텀 Converter
    @Convert(converter = InstantConverter.class)
    @Column(name = "event_time", columnDefinition = "DATETIME")
    private Instant eventTime;
}
```

### 5. Jackson 직렬화 기본 동작

```java
// Instant는 ISO-8601 UTC로 자동 직렬화
@JsonSerialize(using = InstantSerializer.class)
public class InstantSerializer extends StdSerializer<Instant> {
    @Override
    public void serialize(Instant value, JsonGenerator gen, SerializerProvider provider) {
        gen.writeString(value.toString());  // "2024-02-14T22:30:45.123456789Z"
    }
}

// LocalDateTime은 타임존 정보 없이 직렬화
@JsonSerialize(using = LocalDateTimeSerializer.class)
public class LocalDateTimeSerializer extends StdSerializer<LocalDateTime> {
    @Override
    public void serialize(LocalDateTime value, JsonGenerator gen, SerializerProvider provider) {
        gen.writeString(value.toString());  // "2024-02-14T22:30:45.123456789"
    }
}

// JSON 응답:
// {"instant": "2024-02-14T22:30:45.123456789Z"}  // 타임존 명확
// {"localDateTime": "2024-02-14T22:30:45.123456789"}  // 타임존 모호
```

---

## 💻 실전 실험

### 실험 1: Date 변환의 함정

```java
import java.time.*;

public class DateConversionPitfall {
    public static void main(String[] args) {
        // 예: 한국 자정을 Date로 저장하려고 함
        LocalDateTime koreaToday = LocalDateTime.of(2024, 2, 14, 0, 0, 0);
        
        // 방식 1: UTC로 잘못 해석 (함정!)
        Instant wrongInstant = koreaToday.atZone(ZoneId.of("UTC"))
            .toInstant();
        System.out.println("Wrong (UTC): " + wrongInstant);
        // 2024-02-14T00:00:00Z (UTC 자정, 한국 오전 9시)
        
        // 방식 2: 올바른 해석
        Instant correctInstant = koreaToday.atZone(ZoneId.of("Asia/Seoul"))
            .toInstant();
        System.out.println("Correct (Seoul): " + correctInstant);
        // 2024-02-13T15:00:00Z (한국 자정 = UTC 15:00)
        
        // 변환 후 확인
        System.out.println("Wrong date: " + java.util.Date.from(wrongInstant));
        System.out.println("Correct date: " + java.util.Date.from(correctInstant));
    }
}
```

### 실험 2: JPA Hibernate 변환

```java
import javax.persistence.*;
import java.time.*;

@Entity
@Table(name = "events")
public class EventEntity {
    @Id private Long id;
    
    // 테스트: Instant vs LocalDateTime 저장
    @Column(name = "created_instant")
    private Instant createdAt;
    
    @Column(name = "created_datetime")
    @Convert(converter = LocalDateTimeToTimestampConverter.class)
    private LocalDateTime createdTime;
}

// Converter 구현
@Converter
public class LocalDateTimeToTimestampConverter 
        implements AttributeConverter<LocalDateTime, java.sql.Timestamp> {
    
    private static final ZoneId SERVER_ZONE = ZoneId.of("UTC");
    
    @Override
    public java.sql.Timestamp convertToDatabaseColumn(LocalDateTime ldt) {
        if (ldt == null) return null;
        // LocalDateTime을 UTC로 해석해서 저장
        Instant instant = ldt.atZone(SERVER_ZONE).toInstant();
        return java.sql.Timestamp.from(instant);
    }
    
    @Override
    public LocalDateTime convertToEntityAttribute(java.sql.Timestamp ts) {
        if (ts == null) return null;
        // 저장된 TIMESTAMP를 UTC로 해석해서 LocalDateTime으로 변환
        return ts.toInstant().atZone(SERVER_ZONE).toLocalDateTime();
    }
}
```

### 실험 3: 외부 API 응답 처리

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import java.time.*;

public class ExternalApiHandling {
    
    // 예: AWS S3 metadata (ISO-8601)
    static class S3Metadata {
        public String lastModified;  // "2024-02-14T22:30:45.000Z"
        
        public Instant getLastModifiedInstant() {
            return Instant.parse(lastModified);
        }
    }
    
    // 예: Slack API (Unix timestamp)
    static class SlackMessage {
        public long ts;  // 1707945045.123456
        
        public Instant getTimestamp() {
            // 초와 마이크로초 분리
            long seconds = ts / 1_000_000;
            long micros = ts % 1_000_000;
            return Instant.ofEpochSecond(seconds, micros * 1000);
        }
    }
    
    // 예: 국내 시스템 (로컬 타임스탬프, 타임존 불명)
    static class LocalSystemTime {
        public String createdAt;  // "2024-02-14 22:30:45"
        
        public Instant getInstant(ZoneId assumedZone) {
            LocalDateTime ldt = LocalDateTime.parse(createdAt,
                java.time.format.DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
            return ldt.atZone(assumedZone).toInstant();
        }
    }
    
    public static void main(String[] args) {
        // S3: 직접 parse
        Instant s3Time = Instant.parse("2024-02-14T22:30:45.000Z");
        
        // Slack: Unix 타임스탬프
        Instant slackTime = Instant.ofEpochSecond(1707945045, 123456000);
        
        // 로컬 시스템: 타임존 명시 필요
        LocalDateTime local = LocalDateTime.parse("2024-02-14 22:30:45",
            java.time.format.DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
        Instant localTime = local.atZone(ZoneId.of("Asia/Seoul")).toInstant();
    }
}
```

---

## 📊 성능/비교

| 변환 방식 | 속도 | 정확도 | 용도 |
|----------|------|--------|------|
| `Date.toInstant()` | 빠름 | 완벽 | 레거시 Date → 신규 |
| `Calendar.toInstant()` | 빠름 | 완벽 | 레거시 Calendar → 신규 |
| `LocalDateTime.atZone().toInstant()` | 보통 | 타임존 명시 필요 | LocalDateTime → 절대값 |
| `Instant.atZone().toLocalDateTime()` | 보통 | 타임존 명시 필요 | 절대값 → 로컬 |

---

## ⚖️ 트레이드오프

### DB에 Instant vs LocalDateTime

```
Instant 저장:
  + UTC 절대값, 타임존 무관
  + 모든 클라이언트에서 일관된 값
  - 시각적으로 읽기 어려움 (UTC)
  
LocalDateTime 저장:
  + 시각적으로 읽기 쉬움 (로컬 시간)
  + 타임존과 함께 저장 가능 (ZonedDateTime)
  - 변환 복잡, 오류 가능성
  - 타임존 정보 손실 위험
  
권장: Instant로 저장, 필요 시 atZone()으로 변환
```

### 마이그레이션 전략

```
한 번에 교체:
  + 깨끗한 코드
  - 큰 변경, 리스크 높음
  
단계적 마이그레이션:
  + 리스크 분산
  + 테스트 기회 많음
  - 일시적으로 코드 복잡도 증가
  - Facade 패턴으로 경계 관리
  
권장: 새 코드는 신규 API, 레거시는 Facade로 감싸기
```

---

## 📌 핵심 정리

1. **Date ↔ Instant:**
   - `Date.toInstant()` / `Instant.from(date)`
   - 항상 안전 (UTC 절대값)

2. **Calendar ↔ Instant:**
   - `Calendar.toInstant()` / 역변환
   - Calendar의 타임존은 자동 처리됨

3. **LocalDateTime 변환:**
   - `LocalDateTime.atZone(zoneId).toInstant()` (필수: 타임존 명시)
   - 역: `Instant.atZone(zoneId).toLocalDateTime()`

4. **JPA 마이그레이션:**
   - 신규: `Instant` (DB: TIMESTAMP UTC)
   - 레거시 호환: Converter로 자동 변환
   - 표시 필요 시: `atZone()` 사용

5. **Jackson 직렬화:**
   - Instant: ISO-8601 UTC (자동)
   - LocalDateTime: 타임존 명시 필수
   - 외부 API: 응답 형식에 맞춰 parse

6. **마이그레이션 전략:**
   - 경계에서만 변환 (Facade 패턴)
   - 새 코드는 신규 API만 사용
   - DB: Instant 기본, 필요 시 ZonedDateTime

---

## 🤔 생각해볼 문제

**Q1: 다음 JPA 쿼리에서 문제점은?**
```java
@Entity
public class Order {
    @Temporal(TemporalType.TIMESTAMP)
    private LocalDateTime createdAt;
}

// 쿼리
List<Order> orders = em.createQuery(
    "SELECT o FROM Order o WHERE o.createdAt > :date", Order.class)
    .setParameter("date", LocalDateTime.of(2024, 2, 14, 0, 0))
    .getResultList();
```

<details><summary>해설 보기</summary>

**문제: LocalDateTime이 타임존 정보 없이 저장되어 있음**

상황:
1. `createdAt = 2024-02-14 22:30:00` (DB에 저장)
2. 하지만 이것이 UTC인지 KST인지 불명확
3. Hibernate가 자동 변환할 때 시스템 타임존 가정
4. 쿼리 파라미터 `2024-02-14 00:00:00`도 시스템 타임존으로 해석

만약 시스템 타임존이 변경되면:
- UTC 기준: 2024-02-14 22:30 > 2024-02-14 00:00 (true)
- KST 기준: 같은 값이지만 UTC로는 다음 날 07:30 (다른 결과!)

**해결:**
```java
@Entity
public class Order {
    @Column(name = "created_at_utc")
    private Instant createdAt;  // 타임존 명시
}

// 쿼리
List<Order> orders = em.createQuery(
    "SELECT o FROM Order o WHERE o.createdAt > :date", Order.class)
    .setParameter("date", Instant.parse("2024-02-14T00:00:00Z"))
    .getResultList();
```

</details>

**Q2: 외부 API가 "2024-02-14T22:30:00" (타임존 없음)을 반환하는데, 이것을 어떻게 해석할 것인가?**

<details><summary>해설 보기</summary>

**답: API 문서를 확인하고, 기본값 명시하기**

가능한 해석:
1. UTC (많은 서버 API)
2. API 제공자의 로컬 타임존 (문서에 명시)
3. 클라이언트의 로컬 타임존 (명시되지 않음, 위험)

**안전한 방법:**
```java
public class ExternalApiResponse {
    @JsonProperty("timestamp")
    private String timestamp;  // "2024-02-14T22:30:00" (타임존 없음)
    
    // API 문서: "모든 타임스탐프는 UTC 기준"
    public Instant toInstant() {
        return LocalDateTime.parse(timestamp,
            java.time.format.DateTimeFormatter.ISO_LOCAL_DATE_TIME)
            .atZone(ZoneId.of("UTC"))  // ← API 문서의 기본값 명시
            .toInstant();
    }
}

// 또는 ISO-8601으로 요청해서 타임존 포함 강제
// GET /api/events?format=iso8601
// 응답: {"timestamp": "2024-02-14T22:30:00Z"}
```

</details>

**Q3: 마이그레이션 중 Date와 Instant가 섞여 있을 때, 언제 변환할 것인가?**

<details><summary>해설 보기</summary>

**권장: 경계(Facade)에서만 변환**

구조:
```
레거시 코드 (Date 사용)
    ↓ (Facade에서 변환)
비즈니스 로직 (Instant 사용)
    ↓ (Facade에서 변환)
영속성 계층 (DB: TIMESTAMP UTC)
```

예:
```java
// Facade: 경계 역할
public class DateTimeFacade {
    // 레거시 API로부터 받은 Date를 Instant로
    public static Instant receiveFromLegacy(Date date) {
        return date.toInstant();
    }
    
    // Instant를 레거시 API로 전달
    public static Date sendToLegacy(Instant instant) {
        return java.util.Date.from(instant);
    }
}

// 비즈니스 로직: Instant만 사용
public class OrderService {
    public void processOrder(Instant createdAt) {
        // Instant로 모든 작업 수행
    }
}

// 컨트롤러: Facade 사용
@PostMapping("/order")
public void createOrder(@RequestBody OrderDto dto) {
    Instant createdAt = DateTimeFacade.receiveFromLegacy(dto.getCreatedDate());
    service.processOrder(createdAt);
}
```

장점:
- 변환 지점이 명확
- 비즈니스 로직은 깨끗함
- 테스트 용이

</details>

---

<div align="center">

**[⬅️ 이전: TemporalAdjuster · TemporalQuery](./03-temporal-adjuster-query.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: Record 구조 ➡️](../chapter08-pattern-matching-records/01-record-internals.md)**

</div>
