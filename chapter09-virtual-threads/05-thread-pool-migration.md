# Thread Pool → Virtual Thread 전환 전략

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 기존 `ThreadPoolExecutor` 코드를 `Executors.newVirtualThreadPerTaskExecutor()`로 어떻게 마이그레이션하는가?
- 왜 Virtual Thread에서 스레드 풀링이 안티패턴인가?
- ThreadLocal 메모리 누수를 마이그레이션 과정에서 어떻게 회피하는가?
- Spring Boot `spring.threads.virtual.enabled=true`로 전환하면 어떤 부분이 자동으로 바뀌는가?
- 레거시 코드와 새로운 VT 코드를 함께 실행할 때의 주의사항은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

대부분의 Java 서버는 Platform Thread 기반이다. Virtual Thread로 전환하려면 코드 변경과 아키텍처 조정이 필요하다. 잘못 마이그레이션하면 VT의 이점을 살리지 못하거나, 오히려 성능이 떨어질 수 있다. 점진적 전환 전략과 마이그레이션 체크리스트가 필수다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: "Virtual Thread로 바꾸면 자동으로 빨라진다"
  // 코드 변경 없이 JVM 플래그만 변경
  java -XX:+UseVirtualThreads MyApp.java  // (가상의 플래그)
  → VT가 기본 설정으로 모든 task 처리
  → synchronized 때문에 Pinning 발생
  → Platform Thread보다 느려질 수도 있음

실수 2: "ThreadPoolExecutor.corePoolSize = 1000으로 설정하고 VT 사용"
  ExecutorService pool = Executors.newFixedThreadPool(1000);
  // Virtual Thread는 풀링이 무의미
  // 매번 새로운 VT를 만드는 것이 비용 거의 0이므로
  → pool 유지 비용이 오버헤드
  → newVirtualThreadPerTaskExecutor()가 권장

실수 3: "ThreadLocal은 VT에서도 안전하다"
  ThreadLocal<Connection> connPool = ThreadLocal.withInitial(() -> 
      createConnection());
  
  for (VirtualThread vt : million_vts) {
      vt.run(() -> {
          Connection conn = connPool.get();  // 각 VT마다 독립적
          query(conn);
      });
  }
  → 100만 VT × Connection 객체 = 메모리 폭발
  → ThreadLocal 정리가 VT 종료 시 자동이지만, cleanup 로직 누락 시 누수

실제 마이그레이션 버그:

// 레거시 코드 (Platform Thread)
ExecutorService executor = Executors.newFixedThreadPool(200);

for (Request req : requests) {
    executor.submit(() -> {
        handleRequest(req);  // 각 스레드가 요청 처리
    });
}

// 단순 변경 (잘못된 마이그레이션)
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

for (Request req : requests) {
    executor.submit(() -> {
        synchronized(cache) {  // ← Pinning!
            if (!cache.contains(req.id)) {
                cache.put(req.id, computeExpensive());
            }
        }
        handleRequest(req);
    });
}

→ Pinning 때문에 VT의 이점 상실
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
import java.util.concurrent.*;
import java.util.concurrent.locks.ReentrantLock;

public class ThreadPoolMigration {
    
    // 1단계: 레거시 ThreadPoolExecutor 기반
    static class LegacyService {
        ExecutorService pool = Executors.newFixedThreadPool(200);
        
        public void processRequests(List<Request> requests) throws Exception {
            for (Request req : requests) {
                pool.submit(() -> {
                    try {
                        handleRequest(req);
                    } catch (Exception e) {
                        logger.error("Error", e);
                    }
                });
            }
        }
        
        void handleRequest(Request req) throws Exception {
            // 비즈니스 로직
        }
    }
    
    // 2단계: VirtualThreadPerTaskExecutor로 전환 (최소 변경)
    static class Step2Service {
        ExecutorService pool = Executors.newVirtualThreadPerTaskExecutor();
        // ↑ newFixedThreadPool() 대신 newVirtualThreadPerTaskExecutor()
        
        public void processRequests(List<Request> requests) throws Exception {
            for (Request req : requests) {
                pool.submit(() -> {
                    try {
                        handleRequest(req);
                    } catch (Exception e) {
                        logger.error("Error", e);
                    }
                });
            }
        }
        
        void handleRequest(Request req) throws Exception {
            // 비즈니스 로직 (동일)
        }
    }
    
    // 3단계: synchronized → ReentrantLock 교체 (Pinning 해결)
    static class Step3Service {
        ExecutorService pool = Executors.newVirtualThreadPerTaskExecutor();
        ReentrantLock cacheLock = new ReentrantLock();  // ← synchronized 대신
        Cache<String, String> cache = new Cache<>();
        
        public void processRequests(List<Request> requests) throws Exception {
            for (Request req : requests) {
                pool.submit(() -> {
                    try {
                        handleRequest(req);
                    } catch (Exception e) {
                        logger.error("Error", e);
                    }
                });
            }
        }
        
        void handleRequest(Request req) throws Exception {
            // 캐시 검사 (lock 짧게)
            cacheLock.lock();
            try {
                if (cache.contains(req.id)) {
                    handleCached(req, cache.get(req.id));
                    return;
                }
            } finally {
                cacheLock.unlock();
            }
            
            // 계산 (lock 없음)
            String result = computeExpensive(req);
            
            // 캐시 저장 (lock)
            cacheLock.lock();
            try {
                cache.put(req.id, result);
            } finally {
                cacheLock.unlock();
            }
            
            handleResult(req, result);
        }
        
        void handleCached(Request req, String cached) {}
        void handleResult(Request req, String result) {}
        String computeExpensive(Request req) { return ""; }
    }
    
    // 4단계: StructuredTaskScope로 부모-자식 관계 구조화
    static class Step4Service {
        public void processRequests(List<Request> requests) throws Exception {
            for (List<Request> batch : batchRequests(requests, 100)) {
                processBatch(batch);
            }
        }
        
        void processBatch(List<Request> batch) throws Exception {
            try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
                for (Request req : batch) {
                    scope.fork(() -> {
                        handleRequest(req);
                        return null;
                    });
                }
                scope.join();
                scope.throwIfFailed();
            }
        }
        
        void handleRequest(Request req) throws Exception { }
    }
    
    // 5단계: Spring Boot 자동 통합 (설정만 변경)
    // application.properties:
    // spring.threads.virtual.enabled=true
    // server.tomcat.threads.max=200
    // server.tomcat.threads.min-spare=10
    
    // application.yml:
    // spring:
    //   threads:
    //     virtual:
    //       enabled: true
    // server:
    //   tomcat:
    //     threads:
    //       max: 200
    //       min-spare: 10
}
```

---

## 🔬 내부 동작 원리

### 1. 마이그레이션 전략 단계

```
단계별 마이그레이션:

┌──────────────────────────────────────────────────┐
│ Phase 0: 현재 상태 분석                          │
│ ├─ ThreadPoolExecutor 사용 코드 위치 찾기         │
│ ├─ synchronized vs ReentrantLock 통계            │
│ ├─ ThreadLocal 사용 현황                         │
│ ├─ JNI/native call 여부                         │
│ └─ 병목 지점 파악 (profiling)                    │
└──────────────────────────────────────────────────┘
             ↓
┌──────────────────────────────────────────────────┐
│ Phase 1: 격리된 영역부터 전환                     │
│ ├─ 기존 pool과 VT pool 공존                      │
│ │   ExecutorService legacyPool = ...             │
│ │   ExecutorService vtPool = newVirtualThread... │
│ │                                               │
│ ├─ 새로운 기능부터 VT 적용                       │
│ │   @RequestMapping("/new-endpoint")            │
│ │   public void newEndpoint() {                 │
│ │       vtPool.submit(() -> ...);               │
│ │   }                                           │
│ │                                               │
│ └─ 이전 기능은 legacyPool 사용                   │
│     @RequestMapping("/old-endpoint")            │
│     public void oldEndpoint() {                 │
│         legacyPool.submit(() -> ...);           │
│     }                                           │
└──────────────────────────────────────────────────┘
             ↓
┌──────────────────────────────────────────────────┐
│ Phase 2: synchronized 제거/ReentrantLock 도입    │
│ ├─ synchronized 블록 찾기                        │
│ ├─ 각각을 ReentrantLock으로 교체                │
│ ├─ lock 범위 최소화 (lock time 단축)             │
│ └─ 테스트 (Pinning 진단)                        │
│   java -Djdk.tracePinnedThreads=full Test.java │
└──────────────────────────────────────────────────┘
             ↓
┌──────────────────────────────────────────────────┐
│ Phase 3: ThreadLocal 마이그레이션                │
│ ├─ ThreadLocal 변수 식별                        │
│ ├─ 로컬 변수 또는 파라미터로 변경                 │
│ ├─ StructuredTaskScope로 스코프 관리            │
│ └─ cleanup 로직 추가 (필요시)                   │
└──────────────────────────────────────────────────┘
             ↓
┌──────────────────────────────────────────────────┐
│ Phase 4: Spring Boot 설정 변경 (자동화)          │
│ ├─ spring.threads.virtual.enabled=true          │
│ ├─ Tomcat이 자동으로 VT 스케줄러 사용           │
│ ├─ @Async가 VT 기반 실행                        │
│ └─ 코드 변경 최소화                              │
└──────────────────────────────────────────────────┘
             ↓
┌──────────────────────────────────────────────────┐
│ Phase 5: 완전 전환                              │
│ ├─ legacyPool 제거                              │
│ ├─ 모든 ExecutorService를 VT로 통합             │
│ ├─ 성능 검증                                     │
│ └─ 운영 모니터링 (JFR, Prometheus)              │
└──────────────────────────────────────────────────┘
```

### 2. 풀링 vs Per-Task 생성

```
Platform Thread:

생성 비용 = ~1ms
스레드 500개 풀 유지:
  메모리: 500 × 1MB = 500MB (항상 할당)
  재사용: 같은 스레드가 다음 task 처리
  → 풀 유지가 경제적

Virtual Thread:

생성 비용 = ~10μs (0.01ms)
스레드 풀 없이 매번 생성:
  메모리: 필요한 VT만 생성 (~1-2KB)
  스코핑: VT가 종료되면 GC 회수
  → 풀 유지가 오버헤드

예시: 1,000,000 동시 요청

Platform Thread + Pool:
  풀 크기 N = 200
  각 스레드 응답 시간 = 160ms
  초당 처리량 = 200 / 0.160 = 1,250 req/s
  메모리: 200MB (스택) + 관리 오버헤드

Virtual Thread (Per-Task):
  캐리어 스레드 = 8 (CPU 코어)
  각 VT 응답 시간 = 160ms
  초당 처리량 = (1,000,000 × 0.160) / 8 ≈ 20,000 req/s
  메모리: ~4GB (100만 VT × 2KB + 8MB 캐리어)

결론: VT는 풀이 불필요 (오히려 해로움)
```

### 3. ThreadLocal 마이그레이션 패턴

```
패턴 1: ThreadLocal → 로컬 변수

// Before (Platform Thread)
ThreadLocal<Connection> connLocal = new ThreadLocal<>();

String query(String sql) {
    Connection conn = connLocal.get();
    if (conn == null) {
        conn = createConnection();
        connLocal.set(conn);
    }
    return conn.executeQuery(sql);
}

// After (Virtual Thread)
String query(String sql) {
    Connection conn = createConnection();  // 매번 생성 (낮은 비용)
    try {
        return conn.executeQuery(sql);
    } finally {
        conn.close();
    }
}

패턴 2: ThreadLocal → StructuredTaskScope 매개변수

// Before (Platform Thread)
ThreadLocal<Context> contextLocal = new ThreadLocal<>();

class Handler {
    public void handle(Request req) {
        contextLocal.set(new Context(req));
        processRequest();
    }
    
    private void processRequest() {
        Context ctx = contextLocal.get();  // ThreadLocal 접근
        ...
    }
}

// After (Virtual Thread)
public void handleRequest(Request req) throws Exception {
    Context ctx = new Context(req);
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        scope.fork(() -> processRequest(ctx));  // 파라미터로 전달
        scope.join();
    }
}

private String processRequest(Context ctx) {  // 로컬 변수
    ...
    return result;
}

패턴 3: ThreadLocal → Scoped Values (Java 21+)

// 예상되는 미래 API (현재 미리보기)
static final ScopedValue<Connection> CONNECTION = ScopedValue.newInstance();

String query(String sql) {
    return CONNECTION.get().executeQuery(sql);
}

public void handleRequest() throws Exception {
    Connection conn = createConnection();
    ScopedValue.where(CONNECTION, conn)
        .run(() -> {
            processRequest();
        });
}
```

### 4. Spring Boot 자동 통합

```
spring.threads.virtual.enabled=true 활성화 시 변경사항:

Tomcat Servlet Container:
  Before:
    - ThreadPoolExecutor (기본 200 스레드)
    - Platform Thread로 각 요청 처리
    - 동시성 한계: 200 req

  After:
    - ForkJoinPool 캐리어 스레드 (CPU 코어 수)
    - Virtual Thread로 각 요청 처리
    - 동시성 한계: 수만 req

Spring @Async:
  Before:
    @Async
    public void asyncTask() {
        // SimpleAsyncTaskExecutor (Platform Thread 생성)
        Thread.sleep(1000);  // 블로킹
    }

  After:
    @Async
    public void asyncTask() {
        // VirtualThreadTaskExecutor (Virtual Thread)
        Thread.sleep(1000);  // yield 가능
    }

RestClient / WebClient:
  Before:
    RestClient: 동기 + ThreadPoolExecutor 풀
    WebClient: 비동기 + Reactor

  After:
    RestClient: 동기 + Virtual Thread (블로킹 I/O 자동 최적화)
    WebClient: 동기 + Virtual Thread (동시성 향상)

Database Connection Pool (HikariCP):
  Before:
    hikaricp.maximum-pool-size = 20  // Platform Thread를 위한 풀
    각 스레드가 connection 점유

  After:
    hikaricp.maximum-pool-size = 20  // 동일하게 설정 가능
    하지만 100만 VT가 동시 접근 가능
    → 데이터베이스 연결 풀이 병목이 될 수 있음
    → hikari 풀 크기를 더 크게 설정 검토
```

---

## 💻 실전 실험

### 실험 1: ThreadPoolExecutor → VirtualThreadPerTaskExecutor

```java
import java.util.concurrent.*;

public class MigrationTest {
    static class Task implements Runnable {
        int id;
        Task(int id) { this.id = id; }
        public void run() {
            try {
                System.out.println("[Task-" + id + "] 시작");
                Thread.sleep(100);  // 블로킹 I/O 시뮬레이션
                System.out.println("[Task-" + id + "] 완료");
            } catch (InterruptedException e) {}
        }
    }
    
    public static void main(String[] args) throws Exception {
        // Platform Thread 기반
        System.out.println("=== Platform Thread (FixedThreadPool) ===");
        long start = System.currentTimeMillis();
        ExecutorService platformPool = Executors.newFixedThreadPool(8);
        
        for (int i = 0; i < 1000; i++) {
            platformPool.submit(new Task(i));
        }
        platformPool.shutdown();
        platformPool.awaitTermination(1, TimeUnit.MINUTES);
        long elapsed = System.currentTimeMillis() - start;
        System.out.println("총 시간: " + elapsed + "ms");
        
        // Virtual Thread 기반
        System.out.println("\n=== Virtual Thread (PerTaskExecutor) ===");
        start = System.currentTimeMillis();
        ExecutorService vtPool = Executors.newVirtualThreadPerTaskExecutor();
        
        for (int i = 0; i < 1000; i++) {
            vtPool.submit(new Task(i));
        }
        vtPool.shutdown();
        vtPool.awaitTermination(1, TimeUnit.MINUTES);
        elapsed = System.currentTimeMillis() - start;
        System.out.println("총 시간: " + elapsed + "ms");
    }
}

// 예상 결과:
// Platform Thread: ~12.5s (1000 tasks × 100ms / 8 threads)
// Virtual Thread: ~100ms (모든 task가 병렬 실행)
```

### 실험 2: Spring Boot 자동 전환

```properties
# application.properties

# Virtual Thread 활성화
spring.threads.virtual.enabled=true

# 톰캣 설정
server.tomcat.threads.max=200
server.tomcat.threads.min-spare=10

# 로깅
logging.level.org.springframework.web=DEBUG
```

```java
// 변경 없이 자동 적용됨
@RestController
public class ApiController {
    
    @GetMapping("/data")
    public String getData() {
        // Tomcat이 자동으로 Virtual Thread 할당
        return blockingIO();  // I/O가 자동으로 yield 가능
    }
    
    @GetMapping("/async")
    @Async  // 자동으로 Virtual Thread 기반 실행
    public CompletableFuture<String> asyncData() {
        return CompletableFuture.completedFuture(blockingIO());
    }
    
    String blockingIO() {
        try {
            // JDBC, RestTemplate 등 블로킹 I/O
            return jdbcTemplate.queryForObject(
                "SELECT * FROM users LIMIT 1", 
                String.class
            );
        } catch (Exception e) {
            return "error";
        }
    }
}
```

### 실험 3: Pinning 진단 및 해결

```bash
# 1. Pinning이 발생하는 코드로 서버 실행
java -Djdk.tracePinnedThreads=full \
     -Dspring.threads.virtual.enabled=true \
     -cp target/app.jar \
     com.example.Application > pinning.log

# 2. 로그 분석
grep "pinned" pinning.log | head -20

# 3. 문제 코드 수정 (synchronized → ReentrantLock)
# git diff를 통해 변경사항 확인

# 4. 수정 후 재실행
java -Djdk.tracePinnedThreads=full \
     -Dspring.threads.virtual.enabled=true \
     -cp target/app.jar \
     com.example.Application > pinning-fixed.log

# 5. Pinning 감소 확인
wc -l pinning.log pinning-fixed.log
```

---

## 📊 성능/비교

| 항목 | Platform Thread | Virtual Thread |
|------|-----------------|----------------|
| 동시 요청 수 | 200 (풀 크기) | 100,000+ |
| 응답 시간 (160ms I/O) | 160ms | 160ms |
| 초당 처리량 | ~1,250 req/s | ~50,000 req/s |
| 메모리 (100만 req 대기) | 100GB+ | 2-4GB |
| 마이그레이션 난이도 | - | 중간 |
| Spring Boot 설정 변경 | - | 한 줄 |

---

## ⚖️ 트레이드오프

### 마이그레이션의 장점
- **성능**: 40배 이상 처리량 증가
- **메모리**: 1/100 이하로 감소
- **점진적 전환**: 기존 코드와 공존 가능
- **Spring 통합**: 설정 한 줄로 자동 적용

### 마이그레이션의 주의사항
- **Pinning 진단**: `-Djdk.tracePinnedThreads` 필수
- **ThreadLocal 정리**: 메모리 누수 위험
- **JNI 호출**: 완전 회피 불가능 (성능 저하 수용)
- **운영 모니터링**: 기존과 다른 메트릭 (VT 수가 아니라 캐리어 수 모니터링)

---

## 📌 핵심 정리

1. **Phase 별 마이그레이션**: 0단계 분석 → 1단계 격리 → 2단계 Pinning 해결 → 3단계 ThreadLocal → 4단계 Spring 통합
2. **풀 제거**: Virtual Thread는 per-task 생성 비용 거의 0 → 풀링 불필요
3. **synchronized 제거**: ReentrantLock으로 교체하여 Pinning 해결
4. **ThreadLocal 마이그레이션**: 로컬 변수 또는 StructuredTaskScope 파라미터 사용
5. **Spring Boot 자동화**: `spring.threads.virtual.enabled=true` 한 줄로 완전 전환 가능

---

## 🤔 생각해볼 문제

**Q1.** Virtual Thread로 마이그레이션 후 처리량이 늘었는데, 데이터베이스 연결 풀이 병목이 되었다. 어떻게 해결하는가?

<details>
<summary>해설 보기</summary>

Virtual Thread는 동시성은 무한대지만, 외부 리소스(DB connection)는 유한하다.

문제 상황:
```
Virtual Thread: 100만 개 생성 가능
DB Connection Pool: 20개 (HikariCP 기본값)

→ 99만 개 VT가 DB 연결 대기
→ 병목: DB 연결 풀의 connection 수
```

해결책:

1. **연결 풀 크기 증가**:
   ```properties
   spring.datasource.hikari.maximum-pool-size=100
   ```
   하지만 DB 서버의 최대 연결 수 제약 (보통 500-1000)

2. **부하 조절 (Rate Limiting)**:
   ```java
   Semaphore limiter = new Semaphore(1000);
   
   limiter.acquire();
   try {
       procesRequest();
   } finally {
       limiter.release();
   }
   ```
   VT 수를 제한하여 DB 연결 수와 매칭

3. **연결 큐 타임아웃**:
   ```properties
   spring.datasource.hikari.connection-timeout=5000
   ```
   connection 획득 실패 시 exception → 클라이언트 재시도

4. **읽기 복제본(Read Replica) 활용**:
   ```java
   if (isReadOnly) {
       useReplicaPool();  // DB 읽기 전용 풀
   } else {
       usePrimaryPool();  // DB 쓰기 풀
   }
   ```

권장: 2번 + 3번 조합
- VT 동시성을 DB 연결 수에 맞춰 제한
- timeout으로 graceful degradation

</details>

---

**Q2.** 기존 Platform Thread 코드와 새로운 Virtual Thread 코드를 같은 프로세스에서 함께 실행할 수 있는가?

<details>
<summary>해설 보기</summary>

**그렇다. 공존 가능하고 일반적이다.**

```java
// Platform Thread 기반 (레거시)
ExecutorService legacyPool = Executors.newFixedThreadPool(50);

// Virtual Thread 기반 (신규)
ExecutorService vtPool = Executors.newVirtualThreadPerTaskExecutor();

// 동시 사용
legacyPool.submit(() -> {
    // Platform Thread에서 실행
    legacyService.handle();
});

vtPool.submit(() -> {
    // Virtual Thread에서 실행
    newService.handle();
});
```

공존 시 고려사항:

1. **리소스 경쟁**:
   - legacyPool: 50개 PT × 1MB = 50MB 메모리
   - vtPool: 100만 개 VT = 2-4GB 메모리
   - 합계: 2-4GB (메모리 충분해야 함)

2. **CPU 활용**:
   - PT: 50개 스레드가 CPU 타임 경쟁
   - VT: 8개 캐리어가 CPU 타임 경쟁
   - 우선순위 설정 고려

3. **ThreadLocal 격리**:
   - PT와 VT는 별개의 스레드 (ThreadLocal 충돌 없음)
   - 하지만 같은 cache/state 공유 시 synchronization 필요

4. **점진적 전환 전략**:
   ```
   t=0: 레거시 100%, 신규 0% (현황)
   t=1: 레거시 70%, 신규 30% (트래픽 점차 이동)
   t=2: 레거시 30%, 신규 70%
   t=3: 레거시 0%, 신규 100% (완전 전환)
   ```

권장: 공존하면서 점진적으로 전환
- 기존 시스템 안정성 유지
- 새로운 기능부터 VT 적용
- 성능 개선 확인 후 기존 기능 마이그레이션

</details>

---

**Q3.** Virtual Thread로 전환했는데 CPU 사용률이 이전보다 높다. 이유가 뭔가?

<details>
<summary>해설 보기</summary>

Virtual Thread는 처리량이 높아지므로, 같은 시간에 더 많은 작업을 한다.

**더 높은 CPU 사용률 = 정상적인 개선**

예시:
```
Platform Thread 기반:
  초당 1,250 req/s 처리
  CPU 활용도: 50%

Virtual Thread 기반:
  초당 50,000 req/s 처리 (40배)
  CPU 활용도: 80% (처리량 40배이므로 자연)
```

문제가 되는 경우:

1. **CPU 100% 지속**:
   - 대기 없이 계속 연산 중
   - CPU 집약적 작업이 VT에 섞여 있을 수 있음
   - 해결: CPU 작업을 별도 ForkJoinPool로 분리

2. **비효율적인 알고리즘**:
   ```java
   // O(n²) 알고리즘
   for (VirtualThread vt : 100만_vt) {
       for (VirtualThread vt2 : 100만_vt) {
           if (같음) break;
       }
   }
   ```
   해결: 알고리즘 최적화 (O(n log n) 등)

3. **컨텍스트 스위칭 오버헤드**:
   - 실제로는 없음 (JVM이 관리)
   - 하지만 대량 task 생성/삭제의 GC 오버헤드 가능
   - 해결: 배치 처리로 task 수 조절

권장 접근:
- CPU 80% 수준 유지 = 건강한 상태
- 100% 지속 = 과부하 (요청 제한 고려)
- 수직 확장(더 큰 서버) 또는 수평 확장(더 많은 인스턴스) 검토

</details>

---

<div align="center">

**[⬅️ 이전: Structured Concurrency](./04-structured-concurrency.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: 고차 함수 ➡️](../chapter10-functional-patterns/01-higher-order-function.md)**

</div>
