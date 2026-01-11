---
tags: [java, stream, flatmap, functional-programming]
---

# Java Stream flatMap 완벽 가이드

> Java 8+ Stream API의 `flatMap`을 깊이 이해하기 위한 문서

---

## 1. flatMap이란?

**flatMap = map + flatten**

- 각 요소를 **스트림으로 변환**하고
- 모든 스트림을 **하나로 합치는** 연산

### 시그니처

```java
<R> Stream<R> flatMap(Function<T, Stream<R>> mapper)
```

| 구성 요소 | 설명 |
|-----------|------|
| `T` | 입력 스트림의 요소 타입 |
| `Stream<R>` | mapper가 반환해야 하는 타입 (반드시 스트림) |
| `R` | 최종 결과 스트림의 요소 타입 |

---

## 2. map vs flatMap 비교

### map: 1:1 변환

```java
List<String> names = List.of("kim", "lee", "park");

names.stream()
    .map(String::toUpperCase)
    .toList();
// [KIM, LEE, PARK]
// 3개 → 3개
```

```
kim  ──▶ KIM
lee  ──▶ LEE
park ──▶ PARK
```

### flatMap: 1:N 변환 후 평탄화

```java
List<Order> orders = List.of(
    new Order(1L, List.of("사과", "바나나")),
    new Order(2L, List.of("우유")),
    new Order(3L, List.of("빵", "잼", "버터"))
);

orders.stream()
    .flatMap(order -> order.items().stream())
    .toList();
// [사과, 바나나, 우유, 빵, 잼, 버터]
// 3개 → 6개
```

```
Order1 ──▶ [사과, 바나나]  ──┐
Order2 ──▶ [우유]          ──┼──▶ [사과, 바나나, 우유, 빵, 잼, 버터]
Order3 ──▶ [빵, 잼, 버터]  ──┘
```

---

## 3. flatMap 내부 동작 원리

### 핵심: 스트림은 파이프라인

flatMap은 **모아서 나중에 합치는 게 아니라**, 요소가 생성될 때마다 **즉시 흘려보냅니다**.

### 순서도

```
┌─────────────────────────────────────────────────────────────────┐
│  원본 스트림: [A, B, C]                                          │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 1: 첫 번째 요소 A 처리                                     │
│                                                                 │
│    A → mapper 함수 → Stream[1, 2]                               │
│                           │                                     │
│                           ▼                                     │
│                    결과에 1 추가 → [1]                           │
│                    결과에 2 추가 → [1, 2]                        │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 2: 두 번째 요소 B 처리                                     │
│                                                                 │
│    B → mapper 함수 → Stream[] (빈 스트림)                        │
│                           │                                     │
│                           ▼                                     │
│                    꺼낼 게 없음 → 결과 변화 없음                   │
│                    현재 결과: [1, 2]                             │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 3: 세 번째 요소 C 처리                                     │
│                                                                 │
│    C → mapper 함수 → Stream[3, 4, 5]                            │
│                           │                                     │
│                           ▼                                     │
│                    결과에 3 추가 → [1, 2, 3]                     │
│                    결과에 4 추가 → [1, 2, 3, 4]                  │
│                    결과에 5 추가 → [1, 2, 3, 4, 5]               │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  최종 결과: [1, 2, 3, 4, 5]                                      │
└─────────────────────────────────────────────────────────────────┘
```

### 비유: 컨베이어 벨트와 상자

```
     [상자 A]      [상자 B]      [상자 C]
        │             │             │
        ▼             ▼             ▼
     ┌─────┐      ┌─────┐      ┌─────┐
     │열기 │      │열기 │      │열기 │
     └─────┘      └─────┘      └─────┘
        │             │             │
        ▼             ▼             ▼
     [1] [2]       (빈 상자)    [3] [4] [5]
        │                           │
        └───────────┬───────────────┘
                    ▼
    ════════════════════════════════════════
         컨베이어 벨트로 순서대로 흘러감
    ════════════════════════════════════════
                    │
                    ▼
              [1, 2, 3, 4, 5]
```

- **상자를 열어서** 내용물을 컨베이어 벨트에 올리는 것이 flatMap
- **빈 상자**는 올릴 게 없으니 그냥 지나감

---

## 4. 실무 예제

### 예제 1: 주문에서 모든 상품 추출

```java
record Order(Long id, List<String> items) {}

List<Order> orders = List.of(
    new Order(1L, List.of("맥북", "아이패드")),
    new Order(2L, List.of("갤럭시")),
    new Order(3L, List.of("에어팟", "애플워치", "아이폰"))
);

List<String> allItems = orders.stream()
    .flatMap(order -> order.items().stream())
    .toList();

// [맥북, 아이패드, 갤럭시, 에어팟, 애플워치, 아이폰]
```

### 예제 2: Optional 값들 중 존재하는 것만 모으기

```java
Optional<Payment> p1 = findById(1L);  // 값 있음
Optional<Payment> p2 = findById(2L);  // 값 없음 (empty)
Optional<Payment> p3 = findById(3L);  // 값 있음

List<Payment> payments = Stream.of(p1, p2, p3)
    .flatMap(Optional::stream)  // Java 9+
    .toList();

// p2는 빈 스트림이 되어 결과에서 제외됨
// [Payment1, Payment3]
```

### 예제 3: 문자열을 문자 단위로 분해

```java
List<String> words = List.of("hello", "world");

List<Character> chars = words.stream()
    .flatMap(word -> word.chars().mapToObj(c -> (char) c))
    .toList();

// [h, e, l, l, o, w, o, r, l, d]
```

### 예제 4: 중첩 컬렉션 평탄화

```java
List<List<Integer>> nested = List.of(
    List.of(1, 2, 3),
    List.of(4, 5),
    List.of(6, 7, 8, 9)
);

List<Integer> flat = nested.stream()
    .flatMap(List::stream)
    .toList();

// [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

### 예제 5: 부서별 직원 조회

```java
record Department(String name, List<Employee> employees) {}

List<Department> departments = getDepartments();

// 모든 부서의 모든 직원 중 연봉 5000만원 이상인 사람
List<Employee> highPaid = departments.stream()
    .flatMap(dept -> dept.employees().stream())
    .filter(emp -> emp.salary() >= 50_000_000)
    .toList();
```

---

## 5. flatMap 사용 시 주의사항

### 반드시 Stream을 반환해야 함

```java
// ❌ 컴파일 에러 - List 반환
orders.stream()
    .flatMap(order -> order.items())
    .toList();

// ✅ Stream으로 변환
orders.stream()
    .flatMap(order -> order.items().stream())
    .toList();
```

### 빈 스트림은 결과에서 제외됨

```java
Stream.of(
    List.of(1, 2),
    List.of(),        // 빈 리스트 → 빈 스트림 → 결과에 기여 없음
    List.of(3, 4)
)
.flatMap(List::stream)
.toList();

// [1, 2, 3, 4]  ← 빈 리스트는 사라짐
```

### null을 반환하면 안 됨

```java
// ❌ NullPointerException 발생 가능
orders.stream()
    .flatMap(order -> order.items() == null ? null : order.items().stream())
    .toList();

// ✅ 빈 스트림 반환
orders.stream()
    .flatMap(order -> order.items() == null 
        ? Stream.empty() 
        : order.items().stream())
    .toList();
```

---

## 6. Kotlin과 비교

| 상황 | Java | Kotlin |
|------|------|--------|
| 기본 flatMap | `.flatMap(x -> x.items().stream())` | `.flatMap { it.items }` |
| Optional 처리 | `.flatMap(Optional::stream)` | `.mapNotNull { }` 또는 `.filterNotNull()` |
| 빈 값 처리 | `Stream.empty()` | `emptyList()` |

```kotlin
// Kotlin - 더 간결함
val allItems = orders.flatMap { it.items }
```

---

## 7. 관련 메서드

| 메서드 | 설명 | 반환 타입 |
|--------|------|-----------|
| `map` | 1:1 변환 | `Stream<R>` |
| `flatMap` | 1:N 변환 + 평탄화 | `Stream<R>` |
| `flatMapToInt` | flatMap 후 IntStream | `IntStream` |
| `flatMapToLong` | flatMap 후 LongStream | `LongStream` |
| `flatMapToDouble` | flatMap 후 DoubleStream | `DoubleStream` |
| `mapMulti` (Java 16+) | 명령형 flatMap | `Stream<R>` |

---

## 8. 정리

### 핵심 공식

```
flatMap = 각 요소를 스트림으로 변환 + 모든 스트림을 하나로 합침
```

### 언제 사용하나?

- 중첩된 컬렉션을 평탄화할 때
- 1:N 관계의 데이터를 펼칠 때
- Optional 값들 중 존재하는 것만 모을 때
- 하나의 요소에서 여러 결과를 생성할 때

### 기억할 것

1. **mapper 함수는 반드시 Stream을 반환**해야 한다
2. **빈 스트림**은 결과에서 자연스럽게 제외된다
3. **null 대신 Stream.empty()**를 반환해야 한다
4. **스트림은 파이프라인**이다 - 요소가 즉시 흘러간다

---
