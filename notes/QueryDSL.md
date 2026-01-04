---
tags: [querydsl, jpa, dynamic-query, type-safe]
---

## 한 줄 요약

QueryDSL은 타입 안전한 쿼리를 작성할 수 있는 프레임워크로, 컴파일 시점 오류 검증과 동적 쿼리 작성에 강력하다.

## 핵심 정리

- 타입 안전 쿼리로 컴파일 시점 오류 검증
- 문자열 JPQL 대비 IDE 자동완성 지원
- 동적 쿼리 작성이 간결 (BooleanBuilder, BooleanExpression)
- 복잡한 조인, 서브쿼리, 집계 쿼리 가능
- Q클래스 자동 생성 (Annotation Processor)
- 다중 데이터소스 환경에서도 사용 가능

## 상세 내용

### 설정

#### Gradle 설정 (Kotlin DSL)

```kotlin
plugins {
    kotlin("kapt") version "1.9.0"
}

dependencies {
    // QueryDSL
    implementation("com.querydsl:querydsl-jpa:5.0.0:jakarta")
    kapt("com.querydsl:querydsl-apt:5.0.0:jakarta")
    kapt("jakarta.annotation:jakarta.annotation-api")
    kapt("jakarta.persistence:jakarta.persistence-api")
}

// Q클래스 생성 경로 설정
kotlin {
    sourceSets.main {
        kotlin.srcDir("build/generated/source/kapt/main")
    }
}
```

#### Q클래스 생성

```bash
# Gradle 빌드 시 자동 생성
./gradlew clean build

# build/generated/source/kapt/main 경로에 Q클래스 생성
```

생성된 Q클래스:

```kotlin
// Member 엔티티 → QMember 생성
@Entity
class Member(
    @Id @GeneratedValue
    val id: Long? = null,
    var name: String,
    var age: Int
)

// 자동 생성: QMember
val qMember = QMember.member
```

#### QueryDSL Config

```kotlin
@Configuration
class QueryDslConfig(
    private val em: EntityManager
) {

    @Bean
    fun jpaQueryFactory(): JPAQueryFactory {
        return JPAQueryFactory(em)
    }
}
```

### 기본 사용법

#### Repository에서 사용

```kotlin
@Repository
class MemberRepositoryImpl(
    private val queryFactory: JPAQueryFactory
) : MemberRepositoryCustom {

    override fun findByName(name: String): List<Member> {
        return queryFactory
            .selectFrom(QMember.member)
            .where(QMember.member.name.eq(name))
            .fetch()
    }
}
```

#### 조회

```kotlin
val member = QMember.member

// 단건 조회
val result = queryFactory
    .selectFrom(member)
    .where(member.id.eq(1L))
    .fetchOne()  // 단건 (결과 없으면 null, 2개 이상이면 예외)

// 리스트 조회
val results = queryFactory
    .selectFrom(member)
    .where(member.age.gt(20))
    .fetch()  // List<Member>

// 처음 1건
val first = queryFactory
    .selectFrom(member)
    .fetchFirst()  // limit(1).fetchOne()

// 카운트 쿼리
val count = queryFactory
    .select(member.count())
    .from(member)
    .fetchOne()  // Long
```

#### 조건절

```kotlin
val member = QMember.member

queryFactory
    .selectFrom(member)
    .where(
        member.name.eq("홍길동"),          // =
        member.age.ne(30),                 // !=
        member.age.gt(20),                 // >
        member.age.goe(20),                // >=
        member.age.lt(30),                 // <
        member.age.loe(30),                // <=
        member.age.between(20, 30),        // BETWEEN
        member.age.`in`(20, 30, 40),       // IN
        member.name.like("%홍%"),          // LIKE
        member.name.contains("홍"),        // LIKE '%홍%'
        member.name.startsWith("홍"),      // LIKE '홍%'
        member.name.isNotNull              // IS NOT NULL
    )
    .fetch()
```

**여러 조건은 AND**

```kotlin
queryFactory
    .selectFrom(member)
    .where(
        member.name.eq("홍길동"),
        member.age.eq(30)  // AND로 연결
    )
    .fetch()
```

#### 정렬

```kotlin
queryFactory
    .selectFrom(member)
    .orderBy(
        member.age.desc(),
        member.name.asc()
    )
    .fetch()
```

#### 페이징

```kotlin
val results = queryFactory
    .selectFrom(member)
    .orderBy(member.name.asc())
    .offset(10)   // 10번째부터
    .limit(20)    // 20개
    .fetch()

// count 쿼리
val total = queryFactory
    .select(member.count())
    .from(member)
    .fetchOne()
```

**Spring Data와 통합**

```kotlin
data class MemberSearchCondition(
    val name: String?,
    val ageGoe: Int?,
    val ageLoe: Int?
)

override fun searchMembers(
    condition: MemberSearchCondition,
    pageable: Pageable
): Page<Member> {
    val content = queryFactory
        .selectFrom(member)
        .where(
            nameEq(condition.name),
            ageGoe(condition.ageGoe),
            ageLoe(condition.ageLoe)
        )
        .offset(pageable.offset)
        .limit(pageable.pageSize.toLong())
        .fetch()

    val total = queryFactory
        .select(member.count())
        .from(member)
        .where(
            nameEq(condition.name),
            ageGoe(condition.ageGoe),
            ageLoe(condition.ageLoe)
        )
        .fetchOne() ?: 0L

    return PageImpl(content, pageable, total)
}
```

### 동적 쿼리

#### BooleanBuilder

```kotlin
data class MemberSearchCondition(
    val name: String?,
    val ageGoe: Int?,
    val ageLoe: Int?
)

fun searchMembers(condition: MemberSearchCondition): List<Member> {
    val builder = BooleanBuilder()

    // 조건 동적 추가
    condition.name?.let {
        builder.and(member.name.eq(it))
    }

    condition.ageGoe?.let {
        builder.and(member.age.goe(it))
    }

    condition.ageLoe?.let {
        builder.and(member.age.loe(it))
    }

    return queryFactory
        .selectFrom(member)
        .where(builder)
        .fetch()
}
```

단점:
- 가독성 떨어짐
- 조건이 많아지면 복잡

#### BooleanExpression (권장)

```kotlin
fun searchMembers(condition: MemberSearchCondition): List<Member> {
    return queryFactory
        .selectFrom(member)
        .where(
            nameEq(condition.name),
            ageGoe(condition.ageGoe),
            ageLoe(condition.ageLoe)
        )
        .fetch()
}

// 조건 메소드 분리
private fun nameEq(name: String?): BooleanExpression? {
    return name?.let { member.name.eq(it) }
}

private fun ageGoe(ageGoe: Int?): BooleanExpression? {
    return ageGoe?.let { member.age.goe(it) }
}

private fun ageLoe(ageLoe: Int?): BooleanExpression? {
    return ageLoe?.let { member.age.loe(it) }
}
```

장점:
- 가독성 좋음
- 조건 재사용 가능
- 조합 가능

**조건 조합**

```kotlin
// 나이 범위 조건 조합
private fun ageBetween(ageGoe: Int?, ageLoe: Int?): BooleanExpression? {
    val goe = ageGoe(ageGoe)
    val loe = ageLoe(ageLoe)

    return when {
        goe != null && loe != null -> goe.and(loe)
        goe != null -> goe
        loe != null -> loe
        else -> null
    }
}

// 사용
queryFactory
    .selectFrom(member)
    .where(
        nameEq(condition.name),
        ageBetween(condition.ageGoe, condition.ageLoe)
    )
    .fetch()
```

### 조인

#### 기본 조인

```kotlin
val member = QMember.member
val team = QTeam.team

// Inner Join
queryFactory
    .selectFrom(member)
    .join(member.team, team)
    .where(team.name.eq("TeamA"))
    .fetch()

// Left Join
queryFactory
    .selectFrom(member)
    .leftJoin(member.team, team)
    .fetch()

// Fetch Join
queryFactory
    .selectFrom(member)
    .join(member.team, team).fetchJoin()
    .fetch()
```

#### Theta Join

```kotlin
// 연관관계 없는 조인
queryFactory
    .select(member)
    .from(member, team)
    .where(member.name.eq(team.name))
    .fetch()
```

#### On 절

```kotlin
// 조인 대상 필터링
queryFactory
    .selectFrom(member)
    .leftJoin(member.team, team)
    .on(team.name.eq("TeamA"))
    .fetch()

// 연관관계 없는 엔티티 외부 조인
queryFactory
    .selectFrom(member)
    .leftJoin(team).on(member.name.eq(team.name))
    .fetch()
```

### DTO 조회

#### Projections.bean

```kotlin
data class MemberDto(
    var name: String? = null,
    var age: Int? = null
)

queryFactory
    .select(
        Projections.bean(
            MemberDto::class.java,
            member.name,
            member.age
        )
    )
    .from(member)
    .fetch()
```

Setter 사용 (기본 생성자 + Setter 필요)

#### Projections.fields

```kotlin
queryFactory
    .select(
        Projections.fields(
            MemberDto::class.java,
            member.name,
            member.age
        )
    )
    .from(member)
    .fetch()
```

필드 직접 주입 (Reflection)

#### Projections.constructor

```kotlin
data class MemberDto(
    val name: String,
    val age: Int
)

queryFactory
    .select(
        Projections.constructor(
            MemberDto::class.java,
            member.name,
            member.age
        )
    )
    .from(member)
    .fetch()
```

생성자 사용 (가장 안전, 권장)

#### @QueryProjection (권장)

```kotlin
data class MemberDto @QueryProjection constructor(
    val name: String,
    val age: Int
)

// QMemberDto 자동 생성됨
queryFactory
    .select(QMemberDto(member.name, member.age))
    .from(member)
    .fetch()
```

장점:
- 컴파일 시점 타입 체크
- 파라미터 순서 오류 방지

단점:
- DTO가 QueryDSL 의존

### 집계 함수

```kotlin
val tuple = queryFactory
    .select(
        member.count(),
        member.age.sum(),
        member.age.avg(),
        member.age.max(),
        member.age.min()
    )
    .from(member)
    .fetchOne()

tuple?.get(member.count())  // count
tuple?.get(member.age.sum())  // sum
```

#### Group By

```kotlin
val results = queryFactory
    .select(team.name, member.age.avg())
    .from(member)
    .join(member.team, team)
    .groupBy(team.name)
    .having(member.age.avg().gt(20))
    .fetch()
```

### 서브쿼리

```kotlin
val memberSub = QMember("memberSub")

// WHERE 서브쿼리
queryFactory
    .selectFrom(member)
    .where(
        member.age.eq(
            JPAExpressions
                .select(memberSub.age.max())
                .from(memberSub)
        )
    )
    .fetch()

// IN 서브쿼리
queryFactory
    .selectFrom(member)
    .where(
        member.age.`in`(
            JPAExpressions
                .select(memberSub.age)
                .from(memberSub)
                .where(memberSub.age.gt(20))
        )
    )
    .fetch()

// SELECT 서브쿼리
queryFactory
    .select(
        member.name,
        JPAExpressions
            .select(memberSub.age.avg())
            .from(memberSub)
    )
    .from(member)
    .fetch()
```

주의:
- FROM 절 서브쿼리 불가 (JPQL 한계)
- 해결: 조인으로 변경 또는 쿼리 분리

### 다중 데이터소스 설정

#### 설정 클래스

```kotlin
@Configuration
@EnableJpaRepositories(
    basePackages = ["com.example.primary"],
    entityManagerFactoryRef = "primaryEntityManagerFactory",
    transactionManagerRef = "primaryTransactionManager"
)
class PrimaryDataSourceConfig {

    @Primary
    @Bean
    @ConfigurationProperties("spring.datasource.primary")
    fun primaryDataSource(): DataSource {
        return DataSourceBuilder.create().build()
    }

    @Primary
    @Bean
    fun primaryEntityManagerFactory(
        builder: EntityManagerFactoryBuilder,
        @Qualifier("primaryDataSource") dataSource: DataSource
    ): LocalContainerEntityManagerFactoryBean {
        return builder
            .dataSource(dataSource)
            .packages("com.example.primary.entity")
            .persistenceUnit("primary")
            .build()
    }

    @Primary
    @Bean
    fun primaryTransactionManager(
        @Qualifier("primaryEntityManagerFactory")
        entityManagerFactory: EntityManagerFactory
    ): PlatformTransactionManager {
        return JpaTransactionManager(entityManagerFactory)
    }

    @Primary
    @Bean
    fun primaryQueryFactory(
        @Qualifier("primaryEntityManagerFactory")
        entityManagerFactory: EntityManagerFactory
    ): JPAQueryFactory {
        return JPAQueryFactory(entityManagerFactory.createEntityManager())
    }
}

@Configuration
@EnableJpaRepositories(
    basePackages = ["com.example.secondary"],
    entityManagerFactoryRef = "secondaryEntityManagerFactory",
    transactionManagerRef = "secondaryTransactionManager"
)
class SecondaryDataSourceConfig {

    @Bean
    @ConfigurationProperties("spring.datasource.secondary")
    fun secondaryDataSource(): DataSource {
        return DataSourceBuilder.create().build()
    }

    @Bean
    fun secondaryEntityManagerFactory(
        builder: EntityManagerFactoryBuilder,
        @Qualifier("secondaryDataSource") dataSource: DataSource
    ): LocalContainerEntityManagerFactoryBean {
        return builder
            .dataSource(dataSource)
            .packages("com.example.secondary.entity")
            .persistenceUnit("secondary")
            .build()
    }

    @Bean
    fun secondaryTransactionManager(
        @Qualifier("secondaryEntityManagerFactory")
        entityManagerFactory: EntityManagerFactory
    ): PlatformTransactionManager {
        return JpaTransactionManager(entityManagerFactory)
    }

    @Bean
    fun secondaryQueryFactory(
        @Qualifier("secondaryEntityManagerFactory")
        entityManagerFactory: EntityManagerFactory
    ): JPAQueryFactory {
        return JPAQueryFactory(entityManagerFactory.createEntityManager())
    }
}
```

#### 사용

```kotlin
@Repository
class PrimaryMemberRepository(
    @Qualifier("primaryQueryFactory")
    private val queryFactory: JPAQueryFactory
) {
    fun findByName(name: String): List<Member> {
        return queryFactory
            .selectFrom(QMember.member)
            .where(QMember.member.name.eq(name))
            .fetch()
    }
}

@Repository
class SecondaryMemberRepository(
    @Qualifier("secondaryQueryFactory")
    private val queryFactory: JPAQueryFactory
) {
    fun findByName(name: String): List<Member> {
        return queryFactory
            .selectFrom(QMember.member)
            .where(QMember.member.name.eq(name))
            .fetch()
    }
}
```

## 실무 적용

### QueryDSL 설계 체크리스트

- [ ] 동적 쿼리는 BooleanExpression으로 작성했는가?
- [ ] DTO 조회는 @QueryProjection 또는 constructor 사용했는가?
- [ ] 복잡한 쿼리는 사용자 정의 리포지토리로 분리했는가?
- [ ] FROM 절 서브쿼리를 사용하지 않았는가?
- [ ] Fetch Join 시 페이징 문제를 고려했는가?

### 권장 패턴

```kotlin
@Repository
class MemberRepositoryImpl(
    private val queryFactory: JPAQueryFactory
) : MemberRepositoryCustom {

    // 동적 쿼리 - BooleanExpression
    override fun searchMembers(condition: MemberSearchCondition): List<Member> {
        return queryFactory
            .selectFrom(member)
            .leftJoin(member.team, team).fetchJoin()
            .where(
                nameEq(condition.name),
                teamNameEq(condition.teamName),
                ageBetween(condition.ageGoe, condition.ageLoe)
            )
            .fetch()
    }

    // 조건 메소드 분리
    private fun nameEq(name: String?): BooleanExpression? {
        return name?.let { member.name.eq(it) }
    }

    private fun teamNameEq(teamName: String?): BooleanExpression? {
        return teamName?.let { team.name.eq(it) }
    }

    private fun ageBetween(ageGoe: Int?, ageLoe: Int?): BooleanExpression? {
        return ageGoe(ageGoe)?.and(ageLoe(ageLoe))
    }

    private fun ageGoe(ageGoe: Int?): BooleanExpression? {
        return ageGoe?.let { member.age.goe(it) }
    }

    private fun ageLoe(ageLoe: Int?): BooleanExpression? {
        return ageLoe?.let { member.age.loe(it) }
    }
}
```

### 금융권에서의 고려사항

- 복잡한 검색 조건은 QueryDSL로 구현 (동적 쿼리 필수)
- DTO 조회로 민감 정보 노출 방지
- 타입 안전성으로 런타임 오류 감소
- 페이징 필수 (대용량 데이터 처리)
- FROM 절 서브쿼리 불가 시 Native Query 고려

## 관련 노트

- [[JPA-Spring-Data-JPA]] - Spring Data JPA와 QueryDSL 통합
- [[JPA-API-최적화]] - 동적 쿼리와 DTO 조회 최적화

---

**출처**
- Practical QueryDSL (김영한)
- QueryDSL 공식 문서
