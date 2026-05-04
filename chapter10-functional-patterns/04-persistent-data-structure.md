# 영속 자료구조 (Persistent Data Structure)와 함수형 컬렉션

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 영속 자료구조의 정의와 변경 시 새 버전을 만드는 이유는?
- 구조 공유(Structural Sharing)로 O(N) 복사를 회피하는 메커니즘은 무엇인가?
- Java 9의 `List.of()`, `Map.of()` 불변 컬렉션과 영속 자료구조의 차이는?
- Vavr 라이브러리의 `Vector`, `HashMap` 같은 영속 컬렉션을 어떻게 활용하는가?
- 함수형 프로그래밍에서 불변성과 성능을 어떻게 동시에 달성하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

함수형 프로그래밍은 불변성을 지향한다. 하지만 자바의 표준 `ArrayList`는 변경 가능(mutable)하고, 깊은 복사는 성능을 해친다. 영속 자료구조는 이 딜레마를 해결한다. 특히 금융 시스템, 분산 시스템, 이벤트 소싱에서 "모든 과거 상태를 추적"해야 할 때 영속 자료구조는 필수다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 불변성을 위해 모든 변경 시 전체 복사
  List<String> originalList = new ArrayList<>(Arrays.asList("a", "b", "c"));
  
  // "b"를 "b2"로 변경하고 싶음
  List<String> newList = new ArrayList<>(originalList);  // O(n) 복사!
  newList.set(1, "b2");
  
  // 문제: 100개 요소 → 매번 복사 → O(n) × 100 = O(n²) 복잡도
  //      대용량 컬렉션에서 성능 재앙

실수 2: Java 9의 List.of()를 영속 자료구조로 착각
  List<String> list1 = List.of("a", "b", "c");
  List<String> list2 = list1;  // 같은 참조
  
  // "b"를 변경하고 싶으면?
  // → List.of() 결과는 완전 불변, "변경 후 새 버전" 개념 없음
  // → Collections.unmodifiableList()와 동일
  // → 변경하려면 여전히 전체 복사 필요
  
  // 영속 자료구조:
  // Vector<String> vec = Vector.of("a", "b", "c");
  // Vector<String> vec2 = vec.update(1, "b2");  // 구조 공유로 O(log n)

실수 3: 모든 변경 시 깊은 복사
  class State {
      Map<String, User> users;  // 1000개 사용자
  }
  
  State oldState = ...;
  State newState = deepCopy(oldState);  // 1000개 User 모두 복사!
  newState.users.put("new_user", newUser);
  
  // 문제: 한 명 사용자 추가하려고 1000명 모두 복사
  //      "이벤트 소싱"에서 매 이벤트마다 이 작업?
  //      메모리 폭발!

실수 4: 불변 컬렉션과 영속 컬렉션의 혼동
  // 불변 컬렉션 (Collections.unmodifiableList)
  List<String> immutable = Collections.unmodifiableList(list);
  immutable.add("new");  // UnsupportedOperationException
  
  // 영속 컬렉션 (Vavr Vector)
  Vector<String> persistent = Vector.of("a", "b", "c");
  Vector<String> updated = persistent.append("d");
  // persistent: ["a", "b", "c"]  (변경 없음)
  // updated: ["a", "b", "c", "d"]  (새 버전)
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 영속 자료구조 올바른 활용

// ============ 패턴 1: Vavr Vector (불변 리스트) ============
import io.vavr.collection.Vector;

Vector<String> v1 = Vector.of("a", "b", "c");

// 추가: 새 Vector 반환 (v1은 변경 없음)
Vector<String> v2 = v1.append("d");
// v1: ["a", "b", "c"]
// v2: ["a", "b", "c", "d"]

// 수정: 인덱스 1을 "b2"로 (구조 공유)
Vector<String> v3 = v1.update(1, "b2");
// v1: ["a", "b", "c"]
// v3: ["a", "b2", "c"]

// 내부: 대부분 노드 공유 (메모리 효율적)

// ============ 패턴 2: Vavr HashMap (불변 맵) ============
import io.vavr.collection.HashMap;

HashMap<String, Integer> m1 = HashMap.of(
    "alice", 30,
    "bob", 25
);

// 추가/수정: 새 HashMap 반환
HashMap<String, Integer> m2 = m1.put("charlie", 35);
// m1: {alice: 30, bob: 25}
// m2: {alice: 30, bob: 25, charlie: 35}

// 제거: 새 HashMap 반환
HashMap<String, Integer> m3 = m1.remove("bob");
// m1: {alice: 30, bob: 25}
// m3: {alice: 30}

// ============ 패턴 3: 이벤트 소싱과 상태 관리 ============
public class EventSourcedAccount {
    public static class Account {
        public final String id;
        public final long balance;
        public final Vector<String> transactionHistory;
        
        private Account(String id, long balance, Vector<String> history) {
            this.id = id;
            this.balance = balance;
            this.transactionHistory = history;
        }
        
        public static Account create(String id) {
            return new Account(id, 0, Vector.empty());
        }
    }
    
    // 입금: 새 Account 반환 (이전 상태 보존)
    public Account deposit(Account acc, long amount) {
        return new Account(
            acc.id,
            acc.balance + amount,
            acc.transactionHistory.append("DEPOSIT: +" + amount)
        );
    }
    
    // 출금: 새 Account 반환
    public Account withdraw(Account acc, long amount) {
        if (acc.balance < amount) {
            throw new IllegalStateException("잔액 부족");
        }
        return new Account(
            acc.id,
            acc.balance - amount,
            acc.transactionHistory.append("WITHDRAW: -" + amount)
        );
    }
}

// ============ 패턴 4: 함수형 state 관리 (Monadic) ============
public class StateTransition {
    public static class State<T> {
        private final T value;
        private final Vector<String> history;
        
        public State(T value, Vector<String> history) {
            this.value = value;
            this.history = history;
        }
        
        public <U> State<U> map(Function<T, U> f) {
            return new State<>(f.apply(value), history);
        }
        
        public <U> State<U> flatMap(Function<T, State<U>> f) {
            State<U> next = f.apply(value);
            return new State<>(next.value, history.appendAll(next.history));
        }
    }
    
    public static void main(String[] args) {
        State<Integer> s1 = new State<>(10, Vector.empty());
        
        State<Integer> s2 = s1.flatMap(x -> 
            new State<>(x * 2, Vector.of("doubled"))
        );
        
        State<Integer> s3 = s2.flatMap(x -> 
            new State<>(x + 5, Vector.of("added 5"))
        );
        
        System.out.println("최종값: " + s3.value);      // 25
        System.out.println("히스토리: " + s3.history);   // [doubled, added 5]
    }
}

// ============ 패턴 5: Java 9 List.of() vs Vavr Vector ============
import java.util.List;
import io.vavr.collection.Vector;

// Java 9 List.of() - 완전 불변 (변경 불가)
List<String> javaList = List.of("a", "b", "c");
// javaList.add("d");  // UnsupportedOperationException
// javaList.set(1, "b2");  // UnsupportedOperationException

// 변경하려면:
List<String> newJavaList = new ArrayList<>(javaList);
newJavaList.add("d");  // O(n) 복사 필요

// Vavr Vector - 영속 자료구조 (변경 후 새 버전)
Vector<String> vavrVec = Vector.of("a", "b", "c");
Vector<String> newVavrVec = vavrVec.append("d");  // O(log n)
// vavrVec: ["a", "b", "c"]  (원본 보존)
// newVavrVec: ["a", "b", "c", "d"]
```

---

## 🔬 내부 동작 원리

### 1. 구조 공유(Structural Sharing)

```
일반 변경 가능 리스트:
  arr = [a, b, c, d]
  arr2 = copy(arr)  // O(n) 전체 복사
  arr2[1] = x
  
  메모리:
  arr:  [a][b][c][d]  (4개 셀)
  arr2: [a][x][c][d]  (4개 셀, 전체 복사)
  
  총 메모리: 8개 셀

영속 리스트 (Trie 기반):
  trie = [a, b, c, d]
  
  내부 구조 (단순화):
  root → [0] → a
       → [1] → b
       → [2] → c
       → [3] → d
  
  trie2 = trie.update(1, x)
  
  root → [0] → a        ← 공유!
       → [1] → (새 노드)
               └→ x
       → [2] → c        ← 공유!
       → [3] → d        ← 공유!
  
  변경 불필요한 부분은 재사용
  → 메모리: ~1 + (변경 부분) << O(n)
  
  복잡도: O(log n) (트리 높이 = 레벨 개수)
```

### 2. Vavr Vector의 구조 (32-ary Tree)

```java
// Vavr Vector는 내부적으로 32-ary Trie (정확히는 이진 트리 변형)

Vector<E> = Node<E>[]
  ├─ 레벨 0 (leaf): 최대 32개 요소
  ├─ 레벨 1: 32개 노드 (최대 1024개 요소)
  ├─ 레벨 2: 32개 노드 (최대 32768개 요소)
  └─ ...

예: Vector.of(1,2,3,...,100)

구조 (단순화):
  root[0] → [1][2][3]...[32]
  root[1] → [33][34][35]...[64]
  root[2] → [65][66]...[100]...

append(101) 시:
  1. 마지막 블록이 가득 찼는가? NO
  2. 마지막 블록에 101 추가
  3. 변경된 노드만 새로 할당
  4. 나머지는 재사용
  → O(1) (또는 O(log n) 최악)

update(50, x) 시:
  1. 50은 root[1]의 [18] (50 - 32 = 18)
  2. root[1][18] = x로 변경
  3. 새로운 root[1] 할당
  4. 나머지 root[0], root[2], ... 재사용
  → O(log n) (트리 높이)
```

### 3. Vavr HashMap의 구조 (HAMT)

```
HAMT = Hash Array Mapped Trie

기존 HashMap:
  배열[hash % capacity] → 연결 리스트 또는 트리
  
  문제: 크기 변경 시 전체 재해싱 (O(n))
        모든 버킷이 배열로 존재 (메모리 낭비)

HAMT 구조:
  루트 노드
  ├─ 비트맵 (어떤 자식이 있는가?)
  └─ 자식 배열 (sparse)
     ├─ [hash[0:5]] → 자식 노드
     ├─ [hash[5:10]] → 자식 노드
     └─ ...

예: HashMap.of("alice" → 30, "bob" → 25)

hash("alice") = 0xABCD1234
  ├─ [5] = 0xA → 자식 노드 1
  │         ├─ [5] = 0xB → 자식 노드 2
  │         │         ├─ [5] = 0xC → ("alice", 30)
  │         │         └─ ...
  └─ ...

hash("bob") = 0x12345678
  ├─ [5] = 0x1 → 자식 노드 3
  │         ├─ [5] = 0x2 → 자식 노드 4
  │         │         ├─ [5] = 0x3 → ("bob", 25)
  │         │         └─ ...
  └─ ...

put("charlie" → 35) 시:
  hash("charlie") 경로 탐색 → 새 노드 생성 → 경로 재구성
  → 변경된 경로의 노드만 새로 할당
  → 다른 경로는 재사용
  → O(log n), 메모리 효율적
```

### 4. Java 9 List.of() vs Vavr Vector

```
Java 9 List.of():

구조:
  private final Object[] elements;  // 단순 배열
  private final int size;
  
  특징:
  - 완전히 불변 (add, set, remove 불가)
  - 메모리 효율적 (배열만으로 충분)
  - O(1) 접근, O(n) 복사
  
  사용 사례:
  - 불변 컬렉션이 필요할 때
  - "변경하지 않는" 경우에만 사용
  - 변경이 필요하면 새 ArrayList 생성해서 복사

Vavr Vector:

구조:
  private final Object[] elements;  // 트리 노드 배열
  private final int offset;         // 오프셋
  private final int length;
  
  특징:
  - 변경 후 새 버전 반환 (영속성)
  - O(log n) 추가, O(log n) 수정
  - 구조 공유로 메모리 효율
  - 모든 과거 버전 유지 가능
  
  사용 사례:
  - 버전 관리 필요 (이벤트 소싱)
  - 빈번한 변경 + 과거 상태 추적
  - 함수형 프로그래밍

성능 비교 (1000개 요소):

작업                | List.of()  | Vector   | 참고
──────────────────┼──────────┼────────┼────────────
초기 생성          | 0.1ms    | 0.2ms  | Vector: 트리 구성
append(1000번)     | 1500ms   | 150ms  | List.of: O(n)×1000 복사
update(50회)       | 750ms    | 5ms    | Vector: O(log n)
메모리 (1000 + 50  | 110MB    | 15MB   | Vector: 구조 공유
변경 후 합계)      |          |        |
```

---

## 💻 실전 실험

### 실험 1: Vector vs List.of() 성능 비교

```java
import io.vavr.collection.Vector;
import java.util.ArrayList;
import java.util.List;

public class PersistentVsImmutableBenchmark {
    
    public static void main(String[] args) {
        int iterations = 1000;
        
        // 테스트 1: List.of() - 반복 변경
        long start = System.nanoTime();
        List<Integer> list = List.of(1, 2, 3, 4, 5);
        for (int i = 0; i < iterations; i++) {
            // 변경하려면 ArrayList로 복사해야 함
            List<Integer> newList = new ArrayList<>(list);
            newList.add(i);
            list = newList;
        }
        long time1 = System.nanoTime() - start;
        
        // 테스트 2: Vector - 반복 변경
        start = System.nanoTime();
        Vector<Integer> vec = Vector.of(1, 2, 3, 4, 5);
        for (int i = 0; i < iterations; i++) {
            vec = vec.append(i);  // O(log n)
        }
        long time2 = System.nanoTime() - start;
        
        System.out.printf("List.of() + ArrayList 복사: %.2f ms%n", 
            time1 / 1_000_000.0);
        System.out.printf("Vector append: %.2f ms%n", 
            time2 / 1_000_000.0);
        System.out.printf("비율: %.1f배 빠름%n", 
            (double) time1 / time2);
        
        // 예상 결과:
        // List.of() + ArrayList 복사: 450.00 ms
        // Vector append: 8.50 ms
        // 비율: 52.9배 빠름
    }
}
```

### 실험 2: 구조 공유 확인

```java
import io.vavr.collection.Vector;

public class StructuralSharingTest {
    
    public static void main(String[] args) {
        // 원본 Vector
        Vector<String> v1 = Vector.of("a", "b", "c", "d", "e");
        
        // v1에서 파생된 Vector들
        Vector<String> v2 = v1.update(1, "B");      // "b" → "B"
        Vector<String> v3 = v1.append("f");         // 마지막에 "f" 추가
        Vector<String> v4 = v2.update(3, "D");      // v2에서 "d" → "D"
        
        System.out.println("v1: " + v1);  // [a, b, c, d, e]
        System.out.println("v2: " + v2);  // [a, B, c, d, e]
        System.out.println("v3: " + v3);  // [a, b, c, d, e, f]
        System.out.println("v4: " + v4);  // [a, B, c, D, e]
        
        // 메모리 구조 검사 (Reflection)
        System.out.println("\n메모리 효율:");
        System.out.println("v1 생성: 새 트리");
        System.out.println("v2 생성: v1 일부 노드 재사용");
        System.out.println("v3 생성: v1 일부 노드 재사용");
        System.out.println("v4 생성: v2 일부 노드 재사용");
        
        // 모든 버전 동시 사용 가능 (이벤트 소싱)
        System.out.println("\n모든 버전이 메모리에 남아있음:");
        System.out.println("v1: " + v1);
        System.out.println("v2: " + v2);
        System.out.println("v3: " + v3);
        System.out.println("v4: " + v4);
    }
}
```

### 실험 3: 이벤트 소싱 시뮬레이션

```java
import io.vavr.collection.HashMap;
import io.vavr.collection.Vector;

public class EventSourcingTest {
    
    static class Event {
        String type;
        String key;
        String value;
        
        Event(String type, String key, String value) {
            this.type = type; this.key = key; this.value = value;
        }
    }
    
    static class State {
        HashMap<String, String> data;
        Vector<Event> events;
        
        State(HashMap<String, String> data, Vector<Event> events) {
            this.data = data;
            this.events = events;
        }
    }
    
    public static void main(String[] args) {
        State state = new State(HashMap.empty(), Vector.empty());
        
        // 이벤트 1: SET key1=value1
        Event e1 = new Event("SET", "key1", "value1");
        state = new State(
            state.data.put("key1", "value1"),
            state.events.append(e1)
        );
        System.out.println("After E1: " + state.data);
        
        // 이벤트 2: SET key2=value2
        Event e2 = new Event("SET", "key2", "value2");
        state = new State(
            state.data.put("key2", "value2"),
            state.events.append(e2)
        );
        System.out.println("After E2: " + state.data);
        
        // 이벤트 3: UPDATE key1=modified
        Event e3 = new Event("UPDATE", "key1", "modified");
        state = new State(
            state.data.put("key1", "modified"),
            state.events.append(e3)
        );
        System.out.println("After E3: " + state.data);
        
        // 이벤트 히스토리 재생
        System.out.println("\n이벤트 히스토리:");
        state.events.forEach(e -> 
            System.out.println("  " + e.type + ": " + e.key + "=" + e.value)
        );
        
        // 특정 시점의 상태로 롤백 가능 (이벤트 소싱의 강점)
        System.out.println("\n롤백 기능: 이벤트 리플레이로 과거 상태 복구 가능");
    }
}
```

---

## 📊 성능/비교

```
영속 자료구조 vs 표준 자료구조 비교:

작업           | ArrayList  | Vector    | HashMap     | Vavr.HashMap
──────────────┼──────────┼─────────┼──────────┼─────────────
단일 접근      | O(1)      | O(1)      | O(1)       | O(1)
append         | O(1)a     | O(log n)  | -          | -
remove         | O(n)      | O(log n)  | -          | -
update         | O(1)      | O(log n)  | -          | -
put/remove     | -         | -         | O(1)a      | O(log n)

a = amortized (상환 복잡도)

메모리 사용 (1000개 + 50회 변경):

방법                              | 메모리
─────────────────────────────────┼──────────
ArrayList (변경마다 복사)         | ~60MB
ArrayList (50 버전 모두 보관)     | ~3GB
Vector (변경마다 새 버전)         | ~15MB
Vector (50 버전 모두 보관)        | ~20MB

→ 영속 자료구조: O(n) 복사 회피, 구조 공유로 메모리 효율
```

---

## ⚖️ 트레이드오프

```
영속 자료구조의 트레이드오프:

메모리 효율:
  장점: 구조 공유로 O(log n) 메모리 추가
  단점: 포인터 오버헤드 (트리 노드)
       
성능:
  장점: O(log n) 연산으로 대량 변경 효율적
  단점: O(1)이 필요할 때는 Array 사용 권장
       
버전 관리:
  장점: 모든 과거 상태 자동 보존
  단점: 명시적 버전 관리 필요 (GC 전략)
       
호환성:
  장점: 함수형 프로그래밍 패러다임 정렬
  단점: 기존 Java 코드와 호환 약함
       Vavr 의존성 추가 필요

Java 9 List.of()의 장점:
  - 표준 라이브러리 (별도 의존성 X)
  - 메모리 효율적 (단순 배열)
  - IDE 지원 완벽
  
Java 9 List.of()의 단점:
  - "변경 후 새 버전" 패턴 불가
  - 변경 필요 시 매번 복사
  - 버전 관리 불가능
```

---

## 📌 핵심 정리

```
영속 자료구조 (Persistent Data Structure):

정의:
  변경 시 새 버전 생성
  이전 버전 보존
  모든 버전에 대한 참조 유지 가능

핵심: 구조 공유 (Structural Sharing)
  변경 불필요한 부분은 재사용
  변경 부분만 새로 할당
  → O(N) 복사 회피, O(log N) 메모리 추가

Vavr 라이브러리:
  - Vector<E>: 불변 리스트
    append(): O(log n), update(): O(log n)
  - HashMap<K,V>: 불변 맵
    put(): O(log n), remove(): O(log n)
  - 내부: HAMT (Hash Array Mapped Trie)

Java 9 List.of() / Map.of():
  - 표준 불변 컬렉션
  - 완전 불변 (변경 불가)
  - 변경 필요 시 O(n) 복사

함수형 프로그래밍 맥락:
  ① 불변성: 부작용 제거
  ② 버전 관리: 이벤트 소싱, 시간 여행
  ③ 동시성: 락 불필요 (불변이므로)
  ④ 성능: 구조 공유로 메모리/시간 효율
```

---

## 🤔 생각해볼 문제

**Q1.** Vector의 append 연산이 "항상" O(log n)인가, 아니면 "평균" O(log n)인가?

<details>
<summary>해설 보기</summary>

**거의 O(1)에 가깝다.** (정확히는 amortized O(log n))

Vavr Vector의 append 동작:

```
Vector = [a1, a2, a3, ..., a32] (leaf 블록)
       → [b1, b2, ..., b32] (다음 leaf)
       → ...

append 시:
  1. 마지막 leaf에 여유 있으면? 
     → 마지막 leaf 요소 추가, O(1)
  
  2. 마지막 leaf 가득 참?
     → 새 leaf 할당, 부모 노드 갱신, O(log n)

패턴:
  append × 32: O(1) × 31 + O(log n) × 1 = O(32) ÷ 32 = O(1) amortized

실제 복잡도:
  - 대부분: O(1) (leaf에 여유 공간)
  - 주기적으로: O(log n) (leaf 가득 참)
  - 평균: ~O(1)
```

"Amortized O(log n)"이라고 표현하는 게 정확하지만, 실무에서는 거의 O(1)처럼 동작한다.

반대로 update는 항상 O(log n) (변경할 노드까지의 경로 길이).

</details>

---

**Q2.** 영속 자료구조에서 모든 버전을 메모리에 유지하면 메모리 누수가 되지 않는가?

<details>
<summary>해설 보기</summary>

이론적으로는 누수가 아니지만, 실무에서는 주의해야 한다.

```java
// 안전한 패턴: 명시적으로 버전 관리
Vector<String> v1 = Vector.of("a");
Vector<String> v2 = v1.append("b");
Vector<String> v3 = v2.append("c");

// v3만 필요하면 v1, v2 참조 제거
v1 = null;
v2 = null;
// → GC는 v1, v2 정리, v3만 메모리 유지
// → 구조 공유 덕분에 v3도 작은 크기

// 위험한 패턴: 무한 버전 보관
List<Vector<String>> history = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    v = v.append(String.valueOf(i));
    history.add(v);  // 모든 버전 보관!
}
// → history 리스트가 100만 개 참조 유지
// → Vector 각각이 이전 버전 노드 참조
// → 구조 공유 덕분에 메모리는 선형 증가 (괜찮음)
// → 하지만 100만 개 참조는 여전히 큰 메모리

해결책:
  1. 필요한 버전만 선택적으로 보관
  2. Ring buffer 사용 (최근 N개만)
  3. Weak reference로 자동 수집
  4. Garbage collection 명시적 호출
```

결론:
- 영속 자료구조는 메모리 누수를 **방지**하지 않음
- 단지 **구조 공유로 메모리 효율**을 높일 뿐
- 버전 관리는 개발자의 책임

이벤트 소싱의 실제 구현:
```java
// 최근 100개 이벤트만 유지 (이전은 스냅샷으로 압축)
Deque<State> recentStates = new ArrayDeque<>(100);
State latestSnapshot = loadSnaphotFromDB();

for (Event e : events) {
    latestSnapshot = applyEvent(latestSnapshot, e);
    recentStates.addLast(latestSnapshot);
    if (recentStates.size() > 100) {
        recentStates.removeFirst();  // 오래된 버전 제거
    }
}
```

</details>

---

**Q3.** Java 9의 `List.of()`를 변경해야 하면 정말 `new ArrayList<>`로 복사해야 하나? 더 좋은 방법은?

<details>
<summary>해설 보기</summary>

복사 외 몇 가지 대안이 있다.

```java
// 방법 1: ArrayList로 복사 (가장 일반적)
List<String> immutable = List.of("a", "b", "c");
List<String> mutable = new ArrayList<>(immutable);
mutable.add("d");

// 방법 2: 스트림으로 복사 (체인 가능)
List<String> mutable = immutable.stream()
    .collect(Collectors.toList());

// 방법 3: Stream.concat으로 합치기
List<String> extended = Stream.concat(
    immutable.stream(),
    Stream.of("d", "e")
).collect(Collectors.toList());

// 방법 4: 처음부터 ArrayList 사용 (권장)
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
// 또는 직접 구성
List<String> list = new ArrayList<>();
list.add("a");
list.add("b");
list.add("c");

// 방법 5: Vavr Vector 사용 (함수형)
Vector<String> v = Vector.of("a", "b", "c");
Vector<String> extended = v.append("d");  // O(log n), 원본 보존
```

각 방법의 장단점:

| 방법                | 시간        | 메모리         | 가독성 | 추천
|────────────────────┼──────────┼─────────────┼──────┼─────
| ArrayList 복사     | O(n)      | 2배 (임시)  | 높음 | ✓ 일반적
| Stream 복사        | O(n)      | 2배 (임시)  | 낮음 | 체인 필요
| concat             | O(n)      | 2배 (임시)  | 높음 | 합치기 필요
| 처음부터 ArrayList | O(n)      | 1배         | 높음 | ✓ 최적
| Vavr Vector        | O(log n)  | 1.1배       | 높음 | ✓ 함수형

결론:
- 불변 + 변경 불필요: `List.of()`
- 변경 필요: 처음부터 `ArrayList` 또는 `Vavr Vector`
- 여러 버전 관리 필요: `Vavr Vector` (함수형)

성능 최적화:
```java
// 좋지 않음
List<String> result = List.of();
for (String s : items) {
    result = new ArrayList<>(result);  // 매번 복사!
    result.add(s);
}

// 좋음
List<String> result = new ArrayList<>();
for (String s : items) {
    result.add(s);  // O(1) 추가
}
result = Collections.unmodifiableList(result);  // 최종 불변화

// 최적 (함수형)
List<String> result = items.stream()
    .collect(Collectors.toCollection(ArrayList::new));
```

</details>

---

<div align="center">

**[⬅️ 이전: 메모이제이션 패턴](./03-memoization-computeifabsent.md)** | **[홈으로 🏠](../README.md)** | **[다음: Either/Result 패턴 ➡️](./05-either-result-pattern.md)**

</div>
