---
tags: [jpa, 연관관계, onetomany, manytoone, onetoone, manytomany, fetchtype]
---

## 한 줄 요약

JPA 연관관계는 방향성, 다중성, 연관관계 주인을 고려하여 설계하며, 외래 키가 있는 쪽을 주인으로 정해야 한다.

## 핵심 정리

- 외래 키가 있는 곳을 연관관계의 주인으로 설정
- @ManyToOne, @OneToOne은 기본 EAGER, @OneToMany, @ManyToMany는 기본 LAZY
- 실무에서는 모든 연관관계를 LAZY로 설정하고 필요 시 Fetch Join 사용
- 양방향 매핑 시 mappedBy로 주인이 아닌 쪽 표시
- @ManyToMany는 실무 사용 금지, 중간 엔티티로 풀어내기
- OneToMany 단방향보다 ManyToOne 양방향 권장

## 상세 내용

### 연관관계의 기본 개념

#### 방향성

- **단방향**: 한쪽만 참조
- **양방향**: 양쪽 모두 참조 (사실은 단방향 2개)

#### 다중성

- **@ManyToOne**: 다대일
- **@OneToMany**: 일대다
- **@OneToOne**: 일대일
- **@ManyToMany**: 다대다

#### 연관관계의 주인

**핵심 원칙: 외래 키가 있는 곳을 주인으로!**

```kotlin
@Entity
class Member(
    @Id @GeneratedValue
    val id: Long? = null,

    @ManyToOne  // 외래 키 관리 (연관관계 주인)
    @JoinColumn(name = "team_id")
    var team: Team?
)

@Entity
class Team(
    @Id @GeneratedValue
    val id: Long? = null,

    @OneToMany(mappedBy = "team")  // 주인 아님 (읽기 전용)
    val members: MutableList<Member> = mutableListOf()
)
```

주의:
- **주인만 외래 키 관리** (등록, 수정)
- **주인이 아닌 쪽은 읽기만 가능**
- mappedBy는 주인이 아닌 쪽에 표시

### @ManyToOne (다대일)

#### 단방향

```kotlin
@Entity
class Order(
    @Id @GeneratedValue
    val id: Long? = null,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id")
    var member: Member?
)
```

특징:
- 가장 많이 사용하는 연관관계
- 기본 FetchType은 EAGER (권장: LAZY로 변경)

#### 양방향

```kotlin
@Entity
class Order(
    @Id @GeneratedValue
    val id: Long? = null,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id")
    var member: Member?
)

@Entity
class Member(
    @Id @GeneratedValue
    val id: Long? = null,

    @OneToMany(mappedBy = "member")
    val orders: MutableList<Order> = mutableListOf()
)
```

양방향 연관관계 설정:

```kotlin
fun setMember(member: Member) {
    this.member = member
    member.orders.add(this)  // 양쪽 모두 설정
}
```

### @OneToMany (일대다)

#### 단방향 (비권장)

```kotlin
@Entity
class Team(
    @Id @GeneratedValue
    val id: Long? = null,

    @OneToMany
    @JoinColumn(name = "team_id")  // Member 테이블의 외래 키
    val members: MutableList<Member> = mutableListOf()
)
```

문제점:
- 자신이 관리하지 않는 테이블의 외래 키 수정
- 추가 UPDATE 쿼리 발생
- 성능 및 관리 문제

**권장: ManyToOne 양방향 사용**

#### 양방향

```kotlin
@Entity
class Team(
    @Id @GeneratedValue
    val id: Long? = null,

    @OneToMany(mappedBy = "team")
    val members: MutableList<Member> = mutableListOf()
)

@Entity
class Member(
    @Id @GeneratedValue
    val id: Long? = null,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "team_id")
    var team: Team?
)
```

### @OneToOne (일대일)

#### 주 테이블에 외래 키

```kotlin
@Entity
class Member(
    @Id @GeneratedValue
    val id: Long? = null,

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "locker_id", unique = true)
    var locker: Locker?
)

@Entity
class Locker(
    @Id @GeneratedValue
    val id: Long? = null,

    @OneToOne(mappedBy = "locker")
    var member: Member?
)
```

장점:
- 주 테이블만 조회해도 대상 테이블 확인 가능
- 객체지향 개발자 선호

단점:
- 대상 테이블에 외래 키가 있는 구조로 변경 시 테이블 구조 변경 필요

#### 대상 테이블에 외래 키

```kotlin
@Entity
class Member(
    @Id @GeneratedValue
    val id: Long? = null,

    @OneToOne(mappedBy = "member")
    var locker: Locker?
)

@Entity
class Locker(
    @Id @GeneratedValue
    val id: Long? = null,

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id", unique = true)
    var member: Member?
)
```

장점:
- 일대일에서 일대다로 변경 시 테이블 구조 유지 가능
- DBA 선호

단점:
- 프록시 기능의 한계로 지연 로딩 불가능 (항상 즉시 로딩)

### @ManyToMany (다대다) - 실무 사용 금지

#### 문제점

```kotlin
// 사용 금지!
@Entity
class Product(
    @Id @GeneratedValue
    val id: Long? = null,

    @ManyToMany
    @JoinTable(name = "member_product")
    val members: MutableList<Member> = mutableListOf()
)
```

문제:
- 중간 테이블에 컬럼 추가 불가
- 세밀한 쿼리 실행 어려움
- 숨겨진 중간 테이블로 인한 예상치 못한 쿼리 발생

#### 해결: 중간 엔티티 사용

```kotlin
@Entity
class Member(
    @Id @GeneratedValue
    val id: Long? = null,

    @OneToMany(mappedBy = "member")
    val memberProducts: MutableList<MemberProduct> = mutableListOf()
)

@Entity
class MemberProduct(
    @Id @GeneratedValue
    val id: Long? = null,

    @ManyToOne
    @JoinColumn(name = "member_id")
    var member: Member?,

    @ManyToOne
    @JoinColumn(name = "product_id")
    var product: Product?,

    var orderCount: Int,  // 추가 정보 가능
    var orderDate: LocalDateTime
)

@Entity
class Product(
    @Id @GeneratedValue
    val id: Long? = null,

    @OneToMany(mappedBy = "product")
    val memberProducts: MutableList<MemberProduct> = mutableListOf()
)
```

### @JoinColumn

```kotlin
@ManyToOne
@JoinColumn(
    name = "team_id",  // 외래 키 컬럼명
    referencedColumnName = "id",  // 참조 컬럼명 (기본값: 상대 PK)
    foreignKey = ForeignKey(name = "fk_member_team")  // DDL 생성 시 제약조건명
)
var team: Team?
```

### FetchType (페치 전략)

#### 기본값

- **@ManyToOne, @OneToOne**: EAGER (즉시 로딩)
- **@OneToMany, @ManyToMany**: LAZY (지연 로딩)

#### 권장 설정

**모든 연관관계를 LAZY로!**

```kotlin
@ManyToOne(fetch = FetchType.LAZY)  // 명시적으로 LAZY
@JoinColumn(name = "team_id")
var team: Team?
```

이유:
- EAGER는 예상치 못한 SQL 발생
- N+1 문제 발생
- JPQL에서 성능 튜닝 어려움

필요 시 Fetch Join 사용:

```kotlin
// JPQL Fetch Join
val members = em.createQuery(
    "SELECT m FROM Member m JOIN FETCH m.team",
    Member::class.java
).resultList
```

## 실무 적용

### 연관관계 설정 체크리스트

- [ ] 외래 키가 있는 곳을 주인으로 설정했는가?
- [ ] 모든 연관관계를 LAZY로 설정했는가?
- [ ] 양방향 관계 시 연관관계 편의 메소드를 작성했는가?
- [ ] @ManyToMany를 사용하지 않았는가?

### 연관관계 편의 메소드

```kotlin
@Entity
class Member(
    @Id @GeneratedValue
    val id: Long? = null,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "team_id")
    private var _team: Team? = null
) {
    var team: Team?
        get() = _team
        set(value) {
            // 기존 팀과의 관계 제거
            _team?.members?.remove(this)

            // 새 팀 설정
            _team = value

            // 새 팀에 멤버 추가
            value?.members?.add(this)
        }
}
```

### 무한 루프 방지

```kotlin
@Entity
class Member(
    @Id @GeneratedValue
    val id: Long? = null,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "team_id")
    var team: Team?
) {
    // toString에 연관관계 필드 제외
    override fun toString(): String {
        return "Member(id=$id)"
    }
}
```

주의:
- toString, equals, hashCode에 양방향 연관관계 필드 포함 금지
- JSON 직렬화 시 @JsonIgnore 사용

### 금융권에서의 고려사항

- 복잡한 연관관계는 피하고 필요 시 별도 조회
- 양방향 연관관계는 최소화 (단방향 선호)
- Cascade는 신중히 사용 (데이터 정합성 중요)
- 지연 로딩 필수, 성능 이슈는 Fetch Join으로 해결


---

**출처**
- JPA 프로그래밍 (김영한)
