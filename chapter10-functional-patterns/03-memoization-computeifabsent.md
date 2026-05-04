# 메모이제이션 패턴 — ConcurrentHashMap.computeIfAbsent 활용

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 메모이제이션의 정의와 순수 함수의 역할은 무엇인가?
- `ConcurrentHashMap.computeIfAbsent`를 사용한 thread-safe 메모이제이션 구현 방법은?
- `Function<T, R>`을 메모이제이션 데코레이터로 감싸는 일반화 기법은?
- 캐시 크기 제한과 메모리 누수를 회피하기 위해 `Caffeine` 라이브러리를 어떻게 통합하는가?
- 단순 ConcurrentHashMap 메모이제이션의 한계와 Caffeine의 장점은 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Fibonacci, 팩토리얼, API 호출 캐싱, 권한 검증 등 "같은 입력에 대해 같은 결과를 반복 계산"하는 상황은 흔하다. 메모이제이션은 성능을 극적으로 향상시킬 수 있지만, 잘못 구현하면 메모리 누수나 데이터 레이스가 발생한다. 함수형 프로그래밍에서 순수 함수의 결과를 안전하게 캐싱하는 방법을 이해하면, 응답 시간 개선과 리소스 절약을 동시에 달성할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 동기화 없는 HashMap으로 메모이제이션
  Map<Integer, Integer> cache = new HashMap<>();
  
  public int fibonacci(int n) {
      if (cache.containsKey(n)) {
          return cache.get(n);  // 데이터 레이스 가능!
      }
      int result = ... 계산 ...;
      cache.put(n, result);
      return result;
  }
  // 문제: 멀티스레드 환경에서 동시 put/get으로 일관성 깨짐
  // 나쁜 경우: NullPointerException 또는 무한 재계산

실수 2: 전체 메서드를 synchronized로 보호
  Map<Integer, Integer> cache = new HashMap<>();
  
  public synchronized int fibonacci(int n) {  // 전체 메서드 동기화
      if (cache.containsKey(n)) {
          return cache.get(n);
      }
      int result = ... 오래 걸리는 계산 ...;
      cache.put(n, result);
      return result;
  }
  // 문제: 첫 번째 호출이 계산 중일 때 다른 스레드는 대기
  //      계산이 10초 걸리면 모든 호출이 직렬화됨

실수 3: 제한 없는 무한 캐시
  ConcurrentHashMap<String, byte[]> apiCache = new ConcurrentHashMap<>();
  
  public byte[] fetchAndCache(String url) {
      return apiCache.computeIfAbsent(url, k -> {
          return callExternalAPI(k);  // API 결과 캐싱
      });
  }
  // 문제: 며칠 간 서로 다른 URL 호출하면 메모리 누적
  //      가비지 컬렉션 회피: 캐시는 절대 삭제되지 않음
  //      OutOfMemoryError 발생 가능

실수 4: Exception을 캐싱
  ConcurrentHashMap<String, Result> cache = new ConcurrentHashMap<>();
  
  public Result getResult(String key) {
      return cache.computeIfAbsent(key, k -> {
          try {
              return computeExpensive(k);
          } catch (IOException e) {
              return null;  // null을 캐싱!
          }
      });
  }
  // 문제: 첫 실패를 캐싱하면, 나중에 성공해도 null 반환
  //      예: API가 일시적으로 다운됐다가 복구되면?
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 메모이제이션 올바른 패턴

// ============ 패턴 1: computeIfAbsent 기본 사용 ============
public class MemoizedFibonacci {
    private final ConcurrentHashMap<Integer, Integer> cache = 
        new ConcurrentHashMap<>();
    
    public int fibonacci(int n) {
        if (n <= 1) return n;
        
        return cache.computeIfAbsent(n, k -> {
            // 캐시에 없으면 한 번만 계산됨 (같은 키에 대해)
            return fibonacci(k - 1) + fibonacci(k - 2);
        });
    }
}

// ============ 패턴 2: Function을 메모이제이션 데코레이터로 감싸기 ============
public static <T, R> Function<T, R> memoize(Function<T, R> f) {
    ConcurrentHashMap<T, R> cache = new ConcurrentHashMap<>();
    
    return t -> cache.computeIfAbsent(t, f);
    // 매우 간단하고 강력함!
}

// 사용:
Function<Integer, Integer> fibonacci = memoize(n -> {
    if (n <= 1) return n;
    return fibonacci.apply(n - 1) + fibonacci.apply(n - 2);
});

// fibonacci(35) 는 재귀 호출에서 캐시 히트로 매우 빠름

// ============ 패턴 3: Caffeine 캐시로 크기 제한 ============
// Maven: com.github.ben-manes.caffeine:caffeine:3.1.8

import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import java.util.concurrent.TimeUnit;

public class CaffeineMemoizedCache {
    private final Cache<String, Object> cache = Caffeine.newBuilder()
        .maximumSize(1000)                    // 최대 1000개 항목
        .expireAfterWrite(10, TimeUnit.MINUTES) // 10분 후 만료
        .recordStats()                        // 통계 기록
        .build();
    
    public <T> T compute(String key, Function<String, T> f) {
        return (T) cache.get(key, k -> f.apply(k));
    }
    
    public void printStats() {
        System.out.println(cache.stats());  // 히트율, 미스율 등
    }
}

// ============ 패턴 4: Exception 처리 ============
public class SafeMemoizedCache {
    private final ConcurrentHashMap<String, Try<Object>> cache = 
        new ConcurrentHashMap<>();
    
    public <T> T computeOrThrow(String key, Function<String, T> f) {
        Try<T> result = (Try<T>) cache.computeIfAbsent(key, k -> {
            // Try로 성공/실패 모두 캐싱
            try {
                return Try.success(f.apply(k));
            } catch (Exception e) {
                return Try.failure(e);
            }
        });
        return result.get();  // 실패 시 예외 재발생
    }
}

// ============ 패턴 5: TTL(Time To Live) 메모이제이션 ============
public class TTLMemoizedCache {
    private static class CachedValue<T> {
        final T value;
        final long expiryTime;
        
        CachedValue(T value, long ttlMillis) {
            this.value = value;
            this.expiryTime = System.currentTimeMillis() + ttlMillis;
        }
        
        boolean isExpired() {
            return System.currentTimeMillis() > expiryTime;
        }
    }
    
    private final ConcurrentHashMap<String, CachedValue<?>> cache = 
        new ConcurrentHashMap<>();
    private final long ttlMillis;
    
    public TTLMemoizedCache(long ttlMillis) {
        this.ttlMillis = ttlMillis;
    }
    
    public <T> T get(String key, Function<String, T> f) {
        while (true) {
            CachedValue<T> cached = (CachedValue<T>) cache.get(key);
            
            if (cached != null && !cached.isExpired()) {
                return cached.value;
            }
            
            // 캐시 없음 또는 만료됨 → 재계산
            T value = f.apply(key);
            CachedValue<T> newValue = new CachedValue<>(value, ttlMillis);
            
            // 다른 스레드가 이미 계산했을 수도 있음
            CachedValue<T> prev = (CachedValue<T>) cache.putIfAbsent(key, newValue);
            if (prev != null && !prev.isExpired()) {
                return prev.value;
            }
            // 만료되었으면 다시 루프 → 재계산
        }
    }
}
```

---

## 🔬 내부 동작 원리

### 1. 메모이제이션의 수학적 배경

```
순수 함수 (Pure Function):
  f(x) = y
  f(x) = y  (항상 같은 결과)
  
  → 같은 입력에 대해 항상 같은 출력
  → 결과를 캐싱 가능!

메모이제이션:
  cache[x] = f(x)
  
  다음 호출 f(x):
    cache에 있으면 → 캐시 반환 (계산 생략)
    cache에 없으면 → 계산 후 캐시 저장

복잡도 개선 예:
  Fibonacci(n) 순진한 재귀: O(2^n)
  Fibonacci(n) 메모이제이션: O(n)
  
  Fibonacci(40): 
    순진한 방식: 약 2~3초
    메모이제이션: 마이크로초
```

### 2. computeIfAbsent의 원자성

```java
// ConcurrentHashMap.computeIfAbsent의 구조 (단순화):
public V computeIfAbsent(K key, Function<? super K, ? extends V> mappingFunction) {
    // 1. 버킷 해시값으로 위치 결정
    int hash = spread(key.hashCode());
    Node[] tab = table;
    
    // 2. 버킷이 비어있으면 CAS로 삽입 시도 (락 없음!)
    if (tab[index] == null) {
        Node newNode = new Node(hash, key, null);
        if (compareAndSwapObject(tab, offset, null, newNode)) {
            // 삽입 성공, 그러면 계산 실행
            V value = mappingFunction.apply(key);
            newNode.val = value;
            return value;
        }
    }
    
    // 3. 버킷에 이미 노드가 있으면 synchronized로 보호
    synchronized (tab[index]) {
        Node node = tab[index];
        
        // 다시 확인: 다른 스레드가 이미 계산했을 수도
        if (node.val == null) {
            V value = mappingFunction.apply(key);
            node.val = value;
            return value;
        } else {
            // 이미 계산됨 → 그 값 반환
            return node.val;
        }
    }
}

// 결과:
// 1. 같은 키에 대해 여러 스레드가 호출해도
// 2. 최대 한 번의 계산이 보장됨 (대부분의 경우)
// 3. 나머지 스레드는 결과 대기
```

### 3. 캐시 무한 증가의 문제

```
메모리 누수 시나리오:

Week 1: 1000개의 서로 다른 URL 캐싱 → 100MB
Week 2: 1000개의 새로운 URL 캐싱 → 100MB (합계 200MB)
...
Month 1: 4000개 URL → 400MB
...
Year 1: 52,000개 URL → 5.2GB!

무한 캐시의 문제:
  1. 메모리 계속 증가
  2. GC 압력 증가 (Full GC 시간 증가)
  3. OutOfMemoryError 가능

해결책:
  ① 최대 크기 제한 (LRU)
     Caffeine: maximumSize(1000)
     
  ② TTL (Time To Live) 설정
     Caffeine: expireAfterWrite(10, TimeUnit.MINUTES)
     
  ③ 수동 정리
     // 1시간마다 오래된 항목 제거
     new Timer().scheduleAtFixedRate(
       () -> cache.asMap().entrySet().removeIf(...),
       0, 3600_000
     );
```

### 4. Caffeine의 구조

```
Caffeine (com.github.ben-manes.caffeine):
  ConcurrentHashMap 기반
  + 자동 제거 정책 (eviction)
  + TTL 관리
  + 통계 수집
  + 비동기 로딩

구성 요소:

Cache<K, V>
├─ ConcurrentHashMap (실제 저장소)
├─ Deque (LRU 순서 추적)
├─ ExpirationQueue (TTL 관리)
└─ StatsCounter (히트/미스 통계)

동작:
1. get(key, loader)
   ├─ 캐시에 있고 만료 안 됨 → 반환
   ├─ 캐시에 없음 → loader.apply(key)
   └─ 결과 저장 + LRU 업데이트

2. 만료된 항목:
   ├─ 지연 삭제 (lazy deletion)
   └─ 정기적 배경 정리 (background cleanup)

3. 크기 초과:
   ├─ LRU 정책으로 가장 오래된 항목 제거
   └─ 새 항목 삽입
```

---

## 💻 실전 실험

### 실험 1: 메모이제이션 성능 비교

```java
public class MemoizationBenchmark {
    
    // 메모이제이션 없음
    static int fib_naive(int n) {
        if (n <= 1) return n;
        return fib_naive(n - 1) + fib_naive(n - 2);
    }
    
    // ConcurrentHashMap 메모이제이션
    static class MemoizedFib {
        private final ConcurrentHashMap<Integer, Integer> cache = 
            new ConcurrentHashMap<>();
        
        int compute(int n) {
            if (n <= 1) return n;
            return cache.computeIfAbsent(n, k -> 
                compute(k - 1) + compute(k - 2)
            );
        }
    }
    
    // Caffeine 메모이제이션
    static class CaffeineFib {
        private final Cache<Integer, Integer> cache = 
            Caffeine.newBuilder()
                .maximumSize(1000)
                .build();
        
        int compute(int n) {
            if (n <= 1) return n;
            return cache.get(n, k -> 
                compute(k - 1) + compute(k - 2)
            );
        }
    }
    
    public static void main(String[] args) {
        int n = 35;
        
        // 테스트 1: 메모이제이션 없음
        long start = System.nanoTime();
        int result1 = fib_naive(n);
        long time1 = System.nanoTime() - start;
        
        // 테스트 2: ConcurrentHashMap
        start = System.nanoTime();
        int result2 = new MemoizedFib().compute(n);
        long time2 = System.nanoTime() - start;
        
        // 테스트 3: Caffeine
        start = System.nanoTime();
        int result3 = new CaffeineFib().compute(n);
        long time3 = System.nanoTime() - start;
        
        System.out.printf("순진한 재귀: %.2f ms (result=%d)%n", 
            time1 / 1_000_000.0, result1);
        System.out.printf("CHM 메모이제이션: %.2f ms (result=%d)%n", 
            time2 / 1_000_000.0, result2);
        System.out.printf("Caffeine: %.2f ms (result=%d)%n", 
            time3 / 1_000_000.0, result3);
        
        // 예상 결과:
        // 순진한 재귀: 2500.00 ms
        // CHM 메모이제이션: 0.50 ms (5000배 빠름!)
        // Caffeine: 0.55 ms
    }
}
```

### 실험 2: 멀티스레드 메모이제이션 검증

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class MemoizationThreadSafetyTest {
    static AtomicInteger computeCount = new AtomicInteger(0);
    
    static class MemoizedCompute {
        private final ConcurrentHashMap<Integer, Integer> cache = 
            new ConcurrentHashMap<>();
        
        int compute(int n) throws InterruptedException {
            return cache.computeIfAbsent(n, k -> {
                computeCount.incrementAndGet();
                try { Thread.sleep(100); } 
                catch (InterruptedException e) { throw new RuntimeException(e); }
                return k * k;
            });
        }
    }
    
    public static void main(String[] args) throws InterruptedException {
        MemoizedCompute memoized = new MemoizedCompute();
        
        // 100개 스레드가 동시에 같은 값을 요청
        Thread[] threads = new Thread[100];
        for (int i = 0; i < threads.length; i++) {
            threads[i] = new Thread(() -> {
                try { memoized.compute(42); }
                catch (InterruptedException e) { e.printStackTrace(); }
            });
            threads[i].start();
        }
        
        for (Thread t : threads) t.join();
        
        System.out.println("실제 계산 횟수: " + computeCount.get());
        System.out.println("예상: 1 (한 번만 계산)");
        
        // 출력:
        // 실제 계산 횟수: 1
        // 예상: 1
    }
}
```

### 실험 3: Caffeine 통계 및 제거 정책

```java
public class CaffeineStatsTest {
    public static void main(String[] args) throws InterruptedException {
        Cache<String, String> cache = Caffeine.newBuilder()
            .maximumSize(3)              // 최대 3개
            .expireAfterWrite(1, TimeUnit.SECONDS)  // 1초 후 만료
            .recordStats()               // 통계 기록
            .build();
        
        // 데이터 삽입
        cache.put("key1", "value1");
        cache.put("key2", "value2");
        cache.put("key3", "value3");
        
        System.out.println("크기 초과 전: " + cache.asMap().size());  // 3
        
        // 4번째 삽입 → LRU로 가장 오래된 항목 제거
        cache.put("key4", "value4");
        System.out.println("크기 초과 후: " + cache.asMap().size());  // 3 (key1 제거됨)
        
        // 접근 (히트)
        cache.getIfPresent("key2");  // 히트
        cache.getIfPresent("key99"); // 미스
        
        // TTL 테스트
        Thread.sleep(1100);  // 1.1초 대기
        System.out.println("TTL 만료 후: " + cache.asMap().size());  // 0 (모두 만료)
        
        // 통계 출력
        System.out.println(cache.stats());
        // CacheStats{hitCount=1, missCount=1, loadSuccessCount=0, 
        //            loadFailureCount=0, totalLoadTime=0, evictionCount=1}
    }
}
```

---

## 📊 성능/비교

```
메모이제이션 성능 비교 (Fibonacci(35)):

방법                           | 시간      | 계산 횟수
───────────────────────────────┼─────────┼─────────
순진한 재귀                    | 2500ms  | 29,860,703
ConcurrentHashMap 메모이제이션 | 0.5ms   | 35
Caffeine 메모이제이션          | 0.6ms   | 35
Caffeine (만료 정책 포함)      | 0.7ms   | 35

성능 개선:
  메모이제이션 없음 vs CHM: 5000배 빠름
  CHM vs Caffeine: 10% 더 느림 (오버헤드는 무시할 수준)

메모리 사용:

케이스                              | 메모리
─────────────────────────────────┼──────────
HashMap (100만 항목)               | 80MB
ConcurrentHashMap (100만 항목)    | 85MB (5% 오버헤드)
Caffeine (100만 항목, LRU)         | 90MB (10% 오버헤드)
Caffeine (최대 1000, 만료 정책)    | 2MB (자동 정리)
```

---

## ⚖️ 트레이드오프

```
메모이제이션의 트레이드오프:

성능 개선:
  장점: 반복 계산 제거로 극적 성능 향상
  단점: 메모리 사용 증가
       캐시 관리 복잡도 증가

캐시 크기:
  장점: 무한 캐시는 구현 간단
  단점: 메모리 누수 위험
       GC 압력 증가

TTL vs LRU:
  TTL: 시간 기반 만료 (시간이 지나면 자동 제거)
       장점: 항상 최신 데이터 보장
       단점: 유효한 데이터도 만료될 수 있음
  
  LRU: 사용 빈도 기반 (오래된 항목부터 제거)
       장점: 자주 사용되는 항목 우선 유지
       단점: 크기가 정해져 있어야 함

동기화 오버헤드:
  ConcurrentHashMap: 버킷당 독립 락
                     읽기는 락 없음
  Caffeine: CHM 기반 + 추가 관리 (통계, 제거 정책)
           약간의 오버헤드 (무시할 수준)
```

---

## 📌 핵심 정리

```
메모이제이션 (Memoization):

정의: 함수의 계산 결과를 캐싱하여 중복 계산 제거

조건:
  - 순수 함수여야 함 (부작용 없음)
  - 같은 입력 = 항상 같은 출력

기본 구현:
  ConcurrentHashMap<K, V> cache = new ConcurrentHashMap<>();
  return cache.computeIfAbsent(key, f);

Function 데코레이터:
  public static <T, R> Function<T, R> memoize(Function<T, R> f) {
      ConcurrentHashMap<T, R> cache = new ConcurrentHashMap<>();
      return t -> cache.computeIfAbsent(t, f);
  }

computeIfAbsent의 보장:
  - 같은 키에 대해 최대 한 번 계산 (대부분의 경우)
  - 다른 스레드는 결과 대기
  - thread-safe (원자적 보장)

Caffeine 라이브러리:
  - 자동 크기 제한 (LRU)
  - TTL (Time To Live)
  - 통계 수집
  - 비동기 로딩 지원

메모리 관리:
  ① 최대 크기 설정: maximumSize(1000)
  ② TTL 설정: expireAfterWrite(10, TimeUnit.MINUTES)
  ③ 통계 모니터링: cache.stats()
```

---

## 🤔 생각해볼 문제

**Q1.** `computeIfAbsent`에서 계산 함수가 10초 걸린다면, 100개 스레드의 동시 호출은 어떻게 되는가?

<details>
<summary>해설 보기</summary>

이론적으로는 **첫 번째 스레드만 10초 계산, 나머지 99개는 결과 대기**이다. 하지만 실제로는 약간 다르다.

```
시나리오 1: 같은 버킷에 해시될 경우 (낮은 확률)
  T1: computeIfAbsent("key") 시작 → 10초 계산
  T2~T100: 같은 "key"에 computeIfAbsent 호출
  → T1이 계산 중일 때 T2~T100은 버킷 헤드의 synchronized에서 대기
  → T1이 완료하면 결과 반환, T2~T100은 캐시된 값 받음
  → 총 시간: ~10초

시나리오 2: 다른 버킷에 해시될 경우 (높은 확률, 리사이징 중)
  T1: "key1" 계산 시작
  T2~T100: 다른 키들 (다른 버킷)에서 계산 시작
  → 각각 독립적으로 계산 수행
  → 총 시간: ~10초 (병렬 실행)

실제 동작 (Java 8+):
  - 기본적으로 computeIfAbsent는 "한 번만" 실행을 보장하지 않음
  - Javadoc: "함수가 한 번만 호출된다는 보장은 없다"
  - 따라서 다른 버킷에 있으면 중복 계산 가능
  - 같은 버킷이면 대기 (원자성 보장)

예방:
  만약 "정확히 한 번 계산"이 중요하면:
  1. 별도 분산 락 사용 (Redis, Zookeeper)
  2. 계산 결과가 멱등성 있도록 설계
  3. 성능보다 정확성이 중요하면 synchronized 블록
```

</details>

---

**Q2.** Caffeine의 TTL과 LRU를 동시에 설정하면 어떤 항목이 먼저 제거되는가?

<details>
<summary>해설 보기</summary>

**둘 다 적용된다.** 두 가지 조건 중 하나를 만족하면 제거된다.

```java
Cache<String, String> cache = Caffeine.newBuilder()
    .maximumSize(1000)                          // LRU
    .expireAfterWrite(10, TimeUnit.MINUTES)     // TTL
    .build();
```

제거 순서:

1. **TTL 만료된 항목**: 10분이 지나면 자동 제거
2. **크기 초과 + LRU**: 1000개 초과 시 가장 오래 사용되지 않은 항목 제거

예:

```
초기 상태: 900개 항목 (모두 TTL 내)

시나리오 1: 새 항목 200개 추가
  → 1100개 (1000 초과)
  → LRU로 가장 오래된 100개 제거
  → 최종: 1000개

시나리오 2: 시간이 5분 지남, 새 항목 100개 추가
  → 1000개 (용량 맞음)
  → 5분 후 TTL 만료: 원래 항목들 자동 제거
  → 최종: 100개 (새 항목만 남음)

실제 동작:
  - TTL 만료: 지연 삭제 (lazy deletion)
    → get/put 시 확인, 만료되었으면 제거
  - LRU: 능동적 삭제 (eager eviction)
    → 크기 초과 시 즉시 제거
```

트레이드오프:

```
TTL만:
  장점: 자동으로 시간이 지난 데이터 제거
  단점: 공간 낭비 가능 (크기 제한 없음)

LRU만:
  장점: 공간 효율적 (크기 고정)
  단점: 시간이 오래된 데이터도 자주 사용되면 유지
       
동시 적용:
  장점: 시간 & 공간 모두 고려
  단점: 예측 어려움 (두 가지 조건)
```

</details>

---

**Q3.** 메모이제이션을 적용할 수 없는 함수의 특징은?

<details>
<summary>해설 보기</summary>

메모이제이션은 **순수 함수(Pure Function)**에만 적용 가능하다.

순수 함수의 조건:
1. **같은 입력 → 같은 출력** (결정적)
2. **부작용(side effect) 없음**
3. 외부 상태에 의존 X

메모이제이션 불가능한 함수들:

```java
// ❌ 불가능 1: 시간에 의존
LocalDateTime now() {  // 호출 시점마다 다른 값
    return LocalDateTime.now();
}

// ❌ 불가능 2: 난수 생성
int random() {  // 호출마다 다른 값
    return ThreadLocalRandom.current().nextInt();
}

// ❌ 불가능 3: 부작용 있음
int withdrawMoney(int amount) {  // DB 업데이트!
    database.update("balance", account - amount);
    return account - amount;
}

// ❌ 불가능 4: 외부 상태 의존
int balance(String accountId) {  // DB 값이 변할 수 있음
    return database.getBalance(accountId);
}

// ❌ 불가능 5: I/O 작업
String readFile(String path) throws IOException {  // 파일 수정 가능
    return new String(Files.readAllBytes(Paths.get(path)));
}

// ✓ 가능 1: 순수 계산
int factorial(int n) {  // 항상 같은 값
    return n <= 1 ? 1 : n * factorial(n - 1);
}

// ✓ 가능 2: 변환
String capitalize(String s) {  // 부작용 없음
    return s.substring(0, 1).toUpperCase() + s.substring(1);
}

// ✓ 가능 3: 검증 (읽기만, 부작용 X)
boolean isValid(String email) {
    return email.matches("^[^@]+@[^@]+\\.[^@]+$");
}
```

실제 적용 가능성:

```
API 호출 결과 캐싱:
  ❌ url → response
     이유: API 서버의 응답이 시간에 따라 변함
  
  ✓ (url, timestamp) → response
     이유: 특정 시점의 데이터 스냅샷
  
  또는 ✓ TTL 설정해서 주기적 갱신

권한 검증 캐싱:
  ❌ userId → hasPermission
     이유: 권한이 시간에 따라 변함
  
  ✓ (userId, timestamp) → hasPermission
     이유: 특정 시점의 권한 스냅샷

환율 계산 캐싱:
  ❌ (from, to) → rate
     이유: 환율이 시간마다 변함
  
  ✓ (from, to, date) → rate
     이유: 특정 날짜의 환율
```

결론:
- 메모이제이션은 **캐시 유효성 기간(TTL)**을 명확히 해야 함
- 그렇지 않으면 **stale data** 문제 발생
- 따라서 비즈니스 요구사항과 함께 설계해야 함
```

</details>

---

<div align="center">

**[⬅️ 이전: 커링과 부분 적용](./02-currying-partial-application.md)** | **[홈으로 🏠](../README.md)** | **[다음: 영속 자료구조 ➡️](./04-persistent-data-structure.md)**

</div>
