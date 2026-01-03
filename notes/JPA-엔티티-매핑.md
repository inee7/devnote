---
tags: [jpa, entity, embeddable, ddl, 매핑]
---

## 한 줄 요약

JPA는 @Entity, @Embeddable 등의 어노테이션으로 자바 객체와 데이터베이스 테이블을 매핑하고, DDL 자동 생성 기능을 제공한다.

## 핵심 정리

- @Entity는 기본 생성자 필수 (프록시 패턴 지원 위함)
- final, enum, interface, inner 클래스는 엔티티로 사용 불가
- @Embeddable로 값 타입 객체를 만들어 재사용성 향상
- @GeneratedValue 전략은 IDENTITY, SEQUENCE, TABLE, AUTO 중 선택
- DDL 자동 생성은 개발 환경에만 사용, 운영에서는 절대 금지
- 값 타입은 불변으로 설계해야 함

## 상세 내용

### @Entity

#### 기본 요구사항

```kotlin
@Entity
class Member(
    @Id @GeneratedValue
    val id: Long? = null,

    var name: String,
    var age: Int
) {
    // 기본 생성자 필수 (JPA가 리플렉션으로 객체 생성)
    constructor() : this(null, "", 0)
}
```

**제약사항:**
- **기본 생성자 필수** (public 또는 protected)
  - JPA가 프록시 패턴으로 지연 로딩 지원
  - 리플렉션으로 객체 생성 시 필요
- **final 클래스 불가** (프록시 생성 불가)
- **enum, interface, inner 클래스 불가**
- **필드에 final 사용 불가** (변경 감지 불가)

#### 특수 타입 매핑

**Enum 타입**

```kotlin
@Entity
class Member(
    @Id @GeneratedValue
    val id: Long? = null,

    @Enumerated(EnumType.STRING)  // 권장: 문자열로 저장
    var role: Role
)

enum class Role {
    USER, ADMIN
}
```

주의:
- `EnumType.ORDINAL`은 사용 금지 (순서가 바뀌면 데이터 오염)
- `EnumType.STRING` 권장 (실제 enum 이름 저장)

**날짜 타입**

```kotlin
@Entity
class Post(
    @Id @GeneratedValue
    val id: Long? = null,

    @Temporal(TemporalType.TIMESTAMP)
    var createdAt: Date,

    // Java 8 이상은 어노테이션 불필요
    var updatedAt: LocalDateTime  // 자동 매핑
)
```

**큰 문자열**

```kotlin
@Entity
class Article(
    @Id @GeneratedValue
    val id: Long? = null,

    @Lob  // CLOB, BLOB 매핑
    var content: String  // 길이 제한 없음
)
```

### 기본 키 생성 전략

#### IDENTITY

```kotlin
@Entity
class Member(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long? = null
)
```

특징:
- DB가 자동으로 ID 생성 (AUTO_INCREMENT)
- MySQL, PostgreSQL, SQL Server 등
- **persist 시점에 즉시 INSERT 발생** (영속성 컨텍스트에 ID 필요)
- Batch Insert 불가

#### SEQUENCE

```kotlin
@Entity
@SequenceGenerator(
    name = "member_seq_generator",
    sequenceName = "member_seq",
    initialValue = 1,
    allocationSize = 50  // 성능 최적화
)
class Member(
    @Id
    @GeneratedValue(
        strategy = GenerationType.SEQUENCE,
        generator = "member_seq_generator"
    )
    val id: Long? = null
)
```

특징:
- 시퀀스 오브젝트 사용 (Oracle, PostgreSQL 등)
- persist 시점에 시퀀스만 조회 (INSERT는 commit 시)
- `allocationSize`로 성능 최적화 (미리 여러 개 할당)

#### TABLE

```kotlin
@Entity
@TableGenerator(
    name = "member_table_generator",
    table = "my_sequences",
    pkColumnValue = "member_seq",
    allocationSize = 50
)
class Member(
    @Id
    @GeneratedValue(
        strategy = GenerationType.TABLE,
        generator = "member_table_generator"
    )
    val id: Long? = null
)
```

특징:
- 키 생성 전용 테이블 사용
- 모든 DB에서 사용 가능
- 성능이 좋지 않음 (잘 사용 안 함)

### @Embeddable (값 타입)

#### 기본 사용

```kotlin
@Embeddable
class Address(
    var city: String,
    var street: String,
    var zipcode: String
) {
    // 기본 생성자 필수
    protected constructor() : this("", "", "")
}

@Entity
class Member(
    @Id @GeneratedValue
    val id: Long? = null,

    var name: String,

    @Embedded
    var address: Address?
)
```

장점:
- 응집도 높은 설계
- 재사용 가능
- 의미 있는 메소드 추가 가능

```kotlin
@Embeddable
class Address(
    var city: String,
    var street: String,
    var zipcode: String
) {
    fun fullAddress(): String {
        return "$city $street $zipcode"
    }
}
```

#### 값 타입 불변성

**중요: 값 타입은 반드시 불변으로 설계**

```kotlin
@Embeddable
class Address(
    val city: String,  // val 사용
    val street: String,
    val zipcode: String
) {
    // Setter 제거
    // 생성자로만 값 설정
    protected constructor() : this("", "", "")
}
```

이유:
- 값 타입 공유로 인한 부작용 방지
- 여러 엔티티가 같은 값 타입 참조 시 문제 발생

```kotlin
// 나쁜 예
val address = Address("서울", "강남구", "12345")
member1.address = address
member2.address = address  // 같은 인스턴스 공유!

address.city = "부산"  // member1, member2 모두 영향!
```

#### @AttributeOverride

같은 값 타입을 여러 번 사용할 때 컬럼명 변경:

```kotlin
@Entity
class Member(
    @Id @GeneratedValue
    val id: Long? = null,

    @Embedded
    @AttributeOverrides(
        AttributeOverride(name = "city", column = Column(name = "home_city")),
        AttributeOverride(name = "street", column = Column(name = "home_street")),
        AttributeOverride(name = "zipcode", column = Column(name = "home_zipcode"))
    )
    var homeAddress: Address?,

    @Embedded
    @AttributeOverrides(
        AttributeOverride(name = "city", column = Column(name = "work_city")),
        AttributeOverride(name = "street", column = Column(name = "work_street")),
        AttributeOverride(name = "zipcode", column = Column(name = "work_zipcode"))
    )
    var workAddress: Address?
)
```

### DDL 자동 생성

#### 설정

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: create  # 또는 update, validate, none
```

옵션:
- **create**: 기존 테이블 삭제 후 생성
- **create-drop**: create + 종료 시 테이블 삭제
- **update**: 변경 사항만 반영 (컬럼 추가만 가능, 삭제는 안 됨)
- **validate**: 엔티티와 테이블이 정상 매핑되었는지만 확인
- **none**: 사용 안 함

#### 환경별 권장 설정

**개발 초기**
```yaml
ddl-auto: create
```

**개발 서버**
```yaml
ddl-auto: update  # 또는 validate
```

**테스트 서버**
```yaml
ddl-auto: validate
```

**스테이징/운영 서버**
```yaml
ddl-auto: validate  # 또는 none
# 절대 create, create-drop, update 사용 금지!
```

이유:
- 운영 DB는 DBA가 직접 관리
- 자동 DDL로 인한 데이터 손실 위험
- 테이블 Lock 등 성능 문제

#### DDL 생성 어노테이션

```kotlin
@Entity
@Table(
    name = "members",
    uniqueConstraints = [
        UniqueConstraint(columnNames = ["email"])
    ]
)
class Member(
    @Id @GeneratedValue
    val id: Long? = null,

    @Column(
        name = "user_name",
        nullable = false,
        length = 100,
        unique = true
    )
    var name: String,

    @Column(columnDefinition = "TEXT")
    var bio: String?
)
```

주의: 이러한 설정은 DDL 생성에만 영향, 런타임에는 영향 없음

## 실무 적용

### 엔티티 설계 체크리스트

- [ ] 기본 생성자 있는가? (protected 권장)
- [ ] final 클래스 아닌가?
- [ ] @GeneratedValue 전략 선택했는가?
- [ ] 값 타입은 불변으로 설계했는가?
- [ ] DDL 자동 생성은 개발 환경에만 사용하는가?

### Kotlin에서 주의사항

```kotlin
// 나쁜 예: data class 사용 금지
@Entity
data class Member(...)  // equals/hashCode 문제

// 좋은 예: 일반 class 사용
@Entity
class Member(
    @Id @GeneratedValue
    val id: Long? = null,
    var name: String
) {
    // equals/hashCode는 식별자 기반으로 직접 구현
    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (other !is Member) return false
        return id != null && id == other.id
    }

    override fun hashCode(): Int = id?.hashCode() ?: 0
}
```

### 금융권에서의 고려사항

- ID 생성 전략은 SEQUENCE 권장 (Batch Insert 가능)
- 금액 등 중요 데이터는 BigDecimal 사용
- 감사 정보(생성일시, 수정일시 등)는 @Embedded로 공통화
- DDL 자동 생성은 절대 사용 금지, 스크립트로 관리

---

**출처**
- JPA 프로그래밍 (김영한)
