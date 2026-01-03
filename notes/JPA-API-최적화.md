---
tags: [jpa, performance, n-plus-1, fetch-join, dto, osiv]
---

## 한 줄 요약

JPA API 최적화는 N+1 문제를 Fetch Join과 BatchSize로 해결하고, DTO 직접 조회로 필요한 데이터만 가져오며, xToOne/xToMany 관계를 적절히 최적화하는 것이 핵심이다.

## 핵심 정리

- N+1 문제는 Fetch Join, @BatchSize, @Fetch(SUBSELECT)로 해결
- 모든 연관관계는 LAZY로 설정하고 필요 시 Fetch Join 사용
- DTO 직접 조회로 필요한 컬럼만 SELECT (메모리 최적화)
- 컬렉션 Fetch Join 시 페이징 불가 → BatchSize 활용
- 읽기 전용 쿼리는 @Transactional(readOnly = true) 사용
- OSIV는 트레이드오프 이해 후 선택 ([[OSIV]] 참고)

## 상세 내용

### N+1 문제

#### 문제 발생

**즉시 로딩 (EAGER)에서 발생**

```kotlin
@Entity
class Member(
    @Id @GeneratedValue
    val id: Long? = null,

    var name: String,

    @OneToMany(mappedBy = "member", fetch = FetchType.EAGER)
    val orders: MutableList<Order> = mutableListOf()
)

@Entity
class Order(
    @Id @GeneratedValue
    val id: Long? = null,

    @ManyToOne
    @JoinColumn(name = "member_id")
    var member: Member?
)
```

```kotlin
// JPQL 실행
val members = em.createQuery("SELECT m FROM Member m", Member::class.java)
    .resultList
```

실행되는 SQL:

```sql
-- 1. Member 전체 조회
SELECT * FROM member;  -- 결과 5건

-- 2. 각 Member의 Order 조회 (N번)
SELECT * FROM orders WHERE member_id = 1;
SELECT * FROM orders WHERE member_id = 2;
SELECT * FROM orders WHERE member_id = 3;
SELECT * FROM orders WHERE member_id = 4;
SELECT * FROM orders WHERE member_id = 5;
```

총 6개의 쿼리 발생 (1 + N)

**지연 로딩 (LAZY)에서도 발생**

```kotlin
@OneToMany(mappedBy = "member", fetch = FetchType.LAZY)
val orders: MutableList<Order> = mutableListOf()
```

```kotlin
val members = em.createQuery("SELECT m FROM Member m", Member::class.java)
    .resultList

// 지연 로딩된 컬렉션 사용 시점에 N+1 발생
for (member in members) {
    println("${member.name}: ${member.orders.size}")  // 여기서 N번 쿼리
}
```

LAZY 로딩은 조회 시점만 지연할 뿐, 결국 사용 시 N+1 발생

#### 해결법 1: Fetch Join (권장)

**기본 사용**

```kotlin
val members = em.createQuery(
    "SELECT m FROM Member m JOIN FETCH m.orders",
    Member::class.java
).resultList
```

실행 SQL:

```sql
SELECT m.*, o.*
FROM member m
INNER JOIN orders o ON m.id = o.member_id
```

단 1개의 쿼리로 해결

**일대다 조인 시 DISTINCT 필수**

```kotlin
val members = em.createQuery(
    "SELECT DISTINCT m FROM Member m JOIN FETCH m.orders",
    Member::class.java
).resultList
```

이유:
- 일대다 조인은 결과 수가 "다(Many)" 쪽 기준으로 증가
- Member 1명이 Order 3개 가지면 결과는 3행
- DISTINCT로 중복 제거 (JPA가 애플리케이션 레벨에서도 중복 제거)

**컬렉션 Fetch Join의 한계**

```kotlin
// 페이징 불가!
val members = em.createQuery(
    "SELECT DISTINCT m FROM Member m JOIN FETCH m.orders",
    Member::class.java
)
    .setFirstResult(0)
    .setMaxResults(10)  // 메모리에서 페이징!
    .resultList
```

문제:
- 일대다 조인으로 결과 수가 예측 불가
- LIMIT를 걸면 데이터 정합성 문제
- JPA가 경고 로그를 남기고 메모리에서 페이징 (전체 데이터 로드 → OOM 위험)

**컬렉션 Fetch Join은 1개만**

```kotlin
// 잘못된 예
SELECT m
FROM Member m
    JOIN FETCH m.orders
    JOIN FETCH m.coupons  // MultipleBagFetchException 발생
```

이유:
- 카테시안 곱으로 데이터 정합성 문제
- Hibernate는 컬렉션 Fetch Join 2개 이상 금지

#### 해결법 2: @BatchSize (컬렉션 페이징 시 권장)

**엔티티 레벨**

```kotlin
@Entity
class Member(
    @Id @GeneratedValue
    val id: Long? = null,

    var name: String,

    @BatchSize(size = 100)  // IN절에 최대 100개
    @OneToMany(mappedBy = "member")
    val orders: MutableList<Order> = mutableListOf()
)
```

**글로벌 설정 (권장)**

```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 100  # 전역 설정
```

**동작 방식**

```kotlin
val members = em.createQuery("SELECT m FROM Member m", Member::class.java)
    .setFirstResult(0)
    .setMaxResults(10)
    .resultList

// 지연 로딩 컬렉션 사용
for (member in members) {
    println(member.orders.size)
}
```

실행 SQL:

```sql
-- 1. Member 페이징 조회
SELECT * FROM member LIMIT 10;  -- 10건

-- 2. IN절로 Order 한 번에 조회
SELECT * FROM orders
WHERE member_id IN (1, 2, 3, 4, 5, 6, 7, 8, 9, 10);  -- 1번만!
```

장점:
- 페이징 가능
- 쿼리 수 = 1 + 1 (Member 조회 + Order 일괄 조회)
- size 조절로 IN절 크기 제어

권장 size:
- 100 ~ 1000 (DB IN절 최적화 고려)
- 너무 크면 IN절 성능 저하
- 너무 작으면 쿼리 수 증가

#### 해결법 3: @Fetch(FetchMode.SUBSELECT)

```kotlin
@Entity
class Member(
    @Id @GeneratedValue
    val id: Long? = null,

    var name: String,

    @Fetch(FetchMode.SUBSELECT)
    @OneToMany(mappedBy = "member")
    val orders: MutableList<Order> = mutableListOf()
)
```

**동작 방식**

```kotlin
val members = em.createQuery("SELECT m FROM Member m", Member::class.java)
    .resultList

// 지연 로딩 컬렉션 사용
for (member in members) {
    println(member.orders.size)
}
```

실행 SQL:

```sql
-- 1. Member 조회
SELECT * FROM member;

-- 2. 서브쿼리로 Order 조회
SELECT * FROM orders
WHERE member_id IN (
    SELECT id FROM member
);
```

특징:
- 서브쿼리 1번으로 모든 Order 조회
- BatchSize와 유사하지만 서브쿼리 사용
- 실무에서는 BatchSize가 더 범용적

### DTO 직접 조회

#### 엔티티 조회 vs DTO 조회

**엔티티 조회**

```kotlin
val members = em.createQuery(
    "SELECT m FROM Member m JOIN FETCH m.orders",
    Member::class.java
).resultList
```

장점:
- 영속성 컨텍스트 관리 (변경 감지)
- 코드 재사용성

단점:
- 모든 컬럼 SELECT
- 불필요한 데이터 조회
- 메모리 사용량 증가 (스냅샷 저장)

**DTO 직접 조회**

```kotlin
data class MemberOrderDto(
    val memberId: Long,
    val memberName: String,
    val orderCount: Int
)

val dtos = em.createQuery("""
    SELECT new com.example.dto.MemberOrderDto(
        m.id,
        m.name,
        COUNT(o)
    )
    FROM Member m
    LEFT JOIN m.orders o
    GROUP BY m.id, m.name
""", MemberOrderDto::class.java).resultList
```

장점:
- 필요한 컬럼만 SELECT
- 메모리 사용량 감소
- 네트워크 트래픽 감소

단점:
- 재사용성 낮음
- 영속성 컨텍스트 관리 불가

#### xToOne 최적화

**기본 전략: Fetch Join**

```kotlin
data class OrderDto(
    val orderId: Long,
    val memberName: String,
    val orderDate: LocalDateTime
)

// 권장: Fetch Join
val orders = em.createQuery("""
    SELECT new com.example.dto.OrderDto(
        o.id,
        m.name,
        o.orderDate
    )
    FROM Order o
    JOIN o.member m
""", OrderDto::class.java).resultList
```

xToOne 관계는:
- 데이터 증가 없음 (1:1 관계)
- 페이징 가능
- Fetch Join 적극 활용

#### xToMany 최적화

**문제점**

```kotlin
// 안 좋은 예: 컬렉션 Fetch Join + 페이징
val members = em.createQuery(
    "SELECT DISTINCT m FROM Member m JOIN FETCH m.orders",
    Member::class.java
)
    .setFirstResult(0)
    .setMaxResults(10)  // 메모리 페이징 경고!
    .resultList
```

**해결 1: xToOne만 Fetch Join + BatchSize**

```kotlin
// 1. xToOne은 Fetch Join
val members = em.createQuery("""
    SELECT DISTINCT m
    FROM Member m
    JOIN FETCH m.team
""", Member::class.java)
    .setFirstResult(0)
    .setMaxResults(10)
    .resultList

// 2. xToMany는 BatchSize로 처리
// application.yml에 default_batch_fetch_size: 100 설정

// 지연 로딩 컬렉션 사용 시 IN절로 일괄 조회
for (member in members) {
    println(member.orders.size)
}
```

**해결 2: DTO 직접 조회 (복잡한 경우)**

```kotlin
// Step 1: 루트 조회
data class MemberDto(
    val memberId: Long,
    val memberName: String,
    val orders: MutableList<OrderDto> = mutableListOf()
)

val members = em.createQuery("""
    SELECT new com.example.dto.MemberDto(m.id, m.name)
    FROM Member m
""", MemberDto::class.java)
    .setFirstResult(0)
    .setMaxResults(10)
    .resultList

// Step 2: 컬렉션 IN절 조회
val memberIds = members.map { it.memberId }

val orderMap = em.createQuery("""
    SELECT new com.example.dto.OrderDto(o.id, o.member.id, o.orderDate)
    FROM Order o
    WHERE o.member.id IN :memberIds
""", OrderDto::class.java)
    .setParameter("memberIds", memberIds)
    .resultList
    .groupBy { it.memberId }

// Step 3: 매핑
members.forEach { member ->
    member.orders.addAll(orderMap[member.memberId] ?: emptyList())
}
```

장점:
- 쿼리 2번 (1 + 1)
- 페이징 가능
- 필요한 데이터만 조회

### 읽기 전용 쿼리 최적화

#### 문제점

```kotlin
// 단순 조회 화면
val members = em.createQuery("SELECT m FROM Member m", Member::class.java)
    .resultList
```

문제:
- 영속성 컨텍스트가 스냅샷 보관
- 메모리 낭비 (변경 감지 불필요)
- 플러시 시 스냅샷 비교 오버헤드

#### 해결 1: 스칼라 타입 조회

```kotlin
val result = em.createQuery("""
    SELECT m.id, m.name, m.age
    FROM Member m
""").resultList

// 영속성 컨텍스트 관리 안 함
```

#### 해결 2: @Transactional(readOnly = true)

```kotlin
@Service
@Transactional(readOnly = true)  // 읽기 전용
class MemberQueryService(
    private val memberRepository: MemberRepository
) {
    fun findMembers(): List<Member> {
        return memberRepository.findAll()
    }
}
```

효과:
- FlushMode를 MANUAL로 설정
- 플러시 발생하지 않음 (스냅샷 비교 생략)
- 성능 향상

주의:
- 스냅샷은 여전히 저장 (1차 캐시 유지)
- UPDATE 쿼리만 발생하지 않음

#### 해결 3: 읽기 전용 힌트

```kotlin
@Repository
interface MemberRepository : JpaRepository<Member, Long> {

    @QueryHints(QueryHint(name = "org.hibernate.readOnly", value = "true"))
    fun findAll(): List<Member>
}
```

효과:
- 스냅샷 저장 안 함
- 메모리 최적화

주의:
- 1차 캐시는 유지 (동일성 보장)
- 엔티티 수정해도 UPDATE 안 됨

#### 해결 4: 트랜잭션 밖에서 읽기

```kotlin
// EntityManagerFactory에서 직접 생성
val em = emf.createEntityManager()

try {
    // 트랜잭션 없이 조회
    val members = em.createQuery("SELECT m FROM Member m", Member::class.java)
        .resultList
} finally {
    em.close()
}
```

특징:
- 플러시 발생하지 않음
- 지연 로딩 동작 (영속성 컨텍스트는 존재)

주의:
- Spring의 @PersistenceContext는 불가
- 직접 생성한 EntityManager만 가능
- 실무에서는 OSIV 또는 readOnly 권장

### 최적화 전략 정리

#### 권장 전략

**1단계: 엔티티 조회 (Fetch Join + BatchSize)**

```kotlin
// xToOne은 Fetch Join
val orders = em.createQuery("""
    SELECT o
    FROM Order o
    JOIN FETCH o.member
    JOIN FETCH o.delivery
""", Order::class.java).resultList

// xToMany는 BatchSize (application.yml 설정)
// default_batch_fetch_size: 100
```

장점:
- 코드 간결
- 재사용성
- 대부분의 경우 충분

**2단계: DTO 직접 조회 (필요 시)**

```kotlin
// 필요한 데이터만
val dtos = em.createQuery("""
    SELECT new com.example.dto.OrderSimpleDto(
        o.id,
        m.name,
        o.orderDate
    )
    FROM Order o
    JOIN o.member m
""", OrderSimpleDto::class.java).resultList
```

사용 시기:
- 성능 최적화가 중요한 경우
- 네트워크 트래픽 감소 필요
- API 스펙이 고정된 경우

**3단계: Native SQL + DTO (극한 최적화)**

```kotlin
@Query(
    value = """
        SELECT o.id, m.name, o.order_date
        FROM orders o
        JOIN member m ON o.member_id = m.id
    """,
    nativeQuery = true
)
fun findOrdersNative(): List<OrderDto>
```

사용 시기:
- JPQL로 불가능한 쿼리
- DB 특화 기능 활용
- 극한의 성능 최적화

## 실무 적용

### 최적화 체크리스트

- [ ] 모든 연관관계를 LAZY로 설정했는가?
- [ ] default_batch_fetch_size를 100~1000으로 설정했는가?
- [ ] 컬렉션 Fetch Join 시 페이징을 사용하지 않았는가?
- [ ] 읽기 전용 API는 @Transactional(readOnly = true)를 사용했는가?
- [ ] DTO 조회가 필요한지 판단했는가?

### 성능 최적화 우선순위

```
1. LAZY 로딩 + default_batch_fetch_size
   → 가장 먼저 적용, 대부분 해결

2. xToOne Fetch Join
   → 간단하고 효과적

3. DTO 직접 조회
   → 필요 시 적용

4. Native SQL
   → 최후의 수단
```

### 금융권에서의 고려사항

- 대용량 조회는 DTO 직접 조회 권장 (메모리 절약)
- BatchSize는 1000 이하로 제한 (DB 성능 고려)
- 읽기 전용 트랜잭션으로 DB 부하 감소
- 페이징 필수 (전체 조회 금지)
- OSIV는 꺼야 함 ([[OSIV]] 참고)

---

**출처**
- JPA 프로그래밍 (김영한)
- 실전! 스프링 부트와 JPA 활용 2 (김영한)
