---
tags: [spring-data-jpa, repository, query-method, pagination, dto-projection]
---

## 한 줄 요약

Spring Data JPA는 Repository 인터페이스와 메소드 네이밍 규칙으로 반복적인 데이터 접근 코드를 제거하고, 페이징/정렬/DTO 변환을 쉽게 처리할 수 있게 한다.

## 핵심 정리

- JpaRepository는 기본 CRUD, 페이징, 배치 작업 메소드 제공
- 메소드 이름으로 쿼리 자동 생성 (findBy, countBy, existsBy 등)
- @Query로 JPQL 직접 작성 가능 (복잡한 쿼리, Native SQL)
- 페이징은 Pageable, 정렬은 Sort 사용
- DTO 변환은 인터페이스 프로젝션 또는 클래스 프로젝션 활용
- @Modifying으로 벌크 연산 수행 (UPDATE, DELETE)
- 사용자 정의 리포지토리로 복잡한 로직 분리

## 상세 내용

### Repository 인터페이스

#### 계층 구조

```
Repository (마커 인터페이스)
  ↓
CrudRepository (기본 CRUD)
  ↓
PagingAndSortingRepository (페이징, 정렬)
  ↓
JpaRepository (JPA 특화 기능)
```

#### JpaRepository

```kotlin
interface MemberRepository : JpaRepository<Member, Long> {
    // 메소드 선언만으로 구현체 자동 생성
}
```

**기본 제공 메소드:**

```kotlin
// 저장
save(entity)          // 단건 저장
saveAll(entities)     // 여러 건 저장

// 조회
findById(id)          // ID로 조회 (Optional)
findAll()             // 전체 조회
findAllById(ids)      // 여러 ID로 조회

// 삭제
delete(entity)        // 엔티티 삭제
deleteById(id)        // ID로 삭제
deleteAll()           // 전체 삭제

// 기타
count()               // 개수
existsById(id)        // 존재 여부
flush()               // 플러시
```

**JPA 특화 메소드:**

```kotlin
// 배치 작업
saveAllAndFlush(entities)      // 저장 후 즉시 flush
deleteAllInBatch(entities)     // 배치 삭제 (쿼리 1개)
deleteAllByIdInBatch(ids)      // ID로 배치 삭제

// 참조 반환
getReferenceById(id)           // 프록시 반환 (지연 로딩)
```

### 쿼리 메소드 네이밍 규칙

#### 기본 패턴

**findBy, readBy, queryBy, getBy**

```kotlin
interface MemberRepository : JpaRepository<Member, Long> {
    // 단순 조회
    fun findByName(name: String): List<Member>

    // 여러 조건 (And)
    fun findByNameAndAge(name: String, age: Int): List<Member>

    // Or 조건
    fun findByNameOrEmail(name: String, email: String): List<Member>
}
```

**조건 키워드**

```kotlin
// Like
fun findByNameLike(name: String): List<Member>
// 사용: findByNameLike("%홍%")

// Containing (양쪽 %)
fun findByNameContaining(keyword: String): List<Member>
// 사용: findByNameContaining("홍")  → LIKE '%홍%'

// StartingWith, EndingWith
fun findByNameStartingWith(prefix: String): List<Member>
fun findByNameEndingWith(suffix: String): List<Member>

// Between
fun findByAgeBetween(start: Int, end: Int): List<Member>

// LessThan, GreaterThan
fun findByAgeLessThan(age: Int): List<Member>
fun findByAgeGreaterThanEqual(age: Int): List<Member>

// In
fun findByAgeIn(ages: List<Int>): List<Member>

// IsNull, IsNotNull
fun findByEmailIsNull(): List<Member>
fun findByEmailIsNotNull(): List<Member>

// OrderBy
fun findByAgeOrderByNameDesc(age: Int): List<Member>
```

**countBy, existsBy, deleteBy**

```kotlin
// 개수
fun countByAge(age: Int): Long

// 존재 여부
fun existsByEmail(email: String): Boolean

// 삭제
fun deleteByAge(age: Int): Long  // 삭제된 개수 반환
```

#### 제한된 결과

```kotlin
// Top, First
fun findTop3ByOrderByAgeDesc(): List<Member>
fun findFirst10ByAge(age: Int): List<Member>

// Distinct
fun findDistinctByName(name: String): List<Member>
```

### @Query 어노테이션

#### JPQL

```kotlin
interface MemberRepository : JpaRepository<Member, Long> {

    @Query("SELECT m FROM Member m WHERE m.name = :name")
    fun findByNameJpql(@Param("name") name: String): List<Member>

    // 위치 기반 파라미터 (비권장)
    @Query("SELECT m FROM Member m WHERE m.name = ?1 AND m.age = ?2")
    fun findByNameAndAgePositional(name: String, age: Int): List<Member>

    // 컬렉션 파라미터
    @Query("SELECT m FROM Member m WHERE m.age IN :ages")
    fun findByAges(@Param("ages") ages: List<Int>): List<Member>
}
```

**복잡한 조인 쿼리**

```kotlin
@Query("""
    SELECT m FROM Member m
    JOIN FETCH m.team t
    WHERE t.name = :teamName
""")
fun findByTeamNameWithFetch(@Param("teamName") teamName: String): List<Member>
```

#### Native SQL

```kotlin
@Query(
    value = "SELECT * FROM member WHERE name = :name",
    nativeQuery = true
)
fun findByNameNative(@Param("name") name: String): List<Member>
```

주의:
- Native SQL은 DB 종속적
- 엔티티 그래프 기능 사용 불가
- 정렬이 제대로 작동하지 않을 수 있음

#### 벌크 연산

```kotlin
@Modifying  // 필수!
@Query("UPDATE Member m SET m.age = m.age + 1 WHERE m.age >= :age")
fun bulkAgeIncrease(@Param("age") age: Int): Int  // 영향받은 행 수

@Modifying
@Query("DELETE FROM Member m WHERE m.age < :age")
fun deleteBulkByAge(@Param("age") age: Int): Int
```

주의사항:

```kotlin
@Service
@Transactional
class MemberService(
    private val memberRepository: MemberRepository,
    private val em: EntityManager
) {
    fun updateAges() {
        // 벌크 연산
        memberRepository.bulkAgeIncrease(20)

        // 영속성 컨텍스트 초기화 필수!
        em.clear()

        // 또는 @Modifying(clearAutomatically = true) 사용
    }
}
```

이유:
- 벌크 연산은 영속성 컨텍스트 무시하고 DB 직접 수정
- 1차 캐시와 DB 불일치 발생
- 반드시 영속성 컨텍스트 초기화 필요

### 페이징과 정렬

#### Pageable

```kotlin
interface MemberRepository : JpaRepository<Member, Long> {
    fun findByAge(age: Int, pageable: Pageable): Page<Member>
}
```

**사용**

```kotlin
@Service
@Transactional(readOnly = true)
class MemberService(
    private val memberRepository: MemberRepository
) {
    fun getMembers(page: Int, size: Int): Page<Member> {
        val pageable = PageRequest.of(page, size, Sort.by("name").descending())
        return memberRepository.findByAge(20, pageable)
    }
}
```

**Page vs Slice**

```kotlin
// Page: 전체 개수 조회 (count 쿼리 발생)
fun findByAge(age: Int, pageable: Pageable): Page<Member>

// Slice: 다음 페이지 존재 여부만 확인 (count 쿼리 없음)
fun findByAge(age: Int, pageable: Pageable): Slice<Member>

// List: 페이징 메타데이터 없음
fun findByAge(age: Int, pageable: Pageable): List<Member>
```

**Page 사용**

```kotlin
val page: Page<Member> = memberRepository.findByAge(20, pageable)

page.content           // 조회된 데이터
page.totalElements     // 전체 데이터 수
page.totalPages        // 전체 페이지 수
page.number            // 현재 페이지 번호 (0부터 시작)
page.size              // 페이지 크기
page.hasNext()         // 다음 페이지 존재 여부
page.isFirst()         // 첫 페이지 여부
```

**count 쿼리 최적화**

```kotlin
@Query(
    value = "SELECT m FROM Member m LEFT JOIN m.team t WHERE m.age = :age",
    countQuery = "SELECT COUNT(m) FROM Member m WHERE m.age = :age"
)
fun findByAge(@Param("age") age: Int, pageable: Pageable): Page<Member>
```

이유:
- 기본 count 쿼리는 본 쿼리에 LEFT JOIN 포함
- LEFT JOIN이 count에는 불필요하면 성능 낭비
- countQuery로 간단한 쿼리 분리

#### Sort

```kotlin
// 단일 정렬
val sort = Sort.by("name").descending()

// 여러 조건
val sort = Sort.by("age").descending()
    .and(Sort.by("name").ascending())

// Pageable에 포함
val pageable = PageRequest.of(0, 10, sort)
```

주의:
- 프로퍼티명은 엔티티 필드명 사용 (컬럼명 아님)
- 연관관계 정렬 시 `team.name` 가능

### DTO 변환

#### 인터페이스 프로젝션

```kotlin
interface MemberNameOnly {
    fun getName(): String
    fun getAge(): Int
}

interface MemberRepository : JpaRepository<Member, Long> {
    fun findByAge(age: Int): List<MemberNameOnly>
}
```

장점:
- 필요한 컬럼만 SELECT
- 구현 간단

**중첩 구조**

```kotlin
interface MemberWithTeam {
    fun getName(): String

    fun getTeam(): TeamInfo

    interface TeamInfo {
        fun getName(): String
    }
}
```

주의:
- 중첩 프로젝션은 최적화되지 않을 수 있음
- LEFT JOIN으로 모든 컬럼 가져올 수 있음

#### 클래스 프로젝션

```kotlin
data class MemberDto(
    val name: String,
    val age: Int
)

@Query("SELECT new com.example.dto.MemberDto(m.name, m.age) FROM Member m WHERE m.age = :age")
fun findDtoByAge(@Param("age") age: Int): List<MemberDto>
```

주의:
- 패키지명 포함 전체 경로 필수
- DTO에 생성자 필요

#### Native Query + DTO

```kotlin
interface MemberRepository : JpaRepository<Member, Long> {

    @Query(
        value = "SELECT m.name as name, m.age as age FROM member m WHERE m.age = :age",
        nativeQuery = true
    )
    fun findDtoNative(@Param("age") age: Int): List<MemberDto>
}

// DTO는 인터페이스 형태로
interface MemberDto {
    fun getName(): String
    fun getAge(): Int
}
```

### 사용자 정의 리포지토리

#### 구현 방법

**1. 사용자 정의 인터페이스**

```kotlin
interface MemberRepositoryCustom {
    fun findMembersWithComplexLogic(condition: SearchCondition): List<Member>
}
```

**2. 구현 클래스 (이름 규칙: {리포지토리}Impl)**

```kotlin
class MemberRepositoryImpl(
    private val em: EntityManager
) : MemberRepositoryCustom {

    override fun findMembersWithComplexLogic(condition: SearchCondition): List<Member> {
        // QueryDSL 또는 복잡한 JPQL 사용
        return em.createQuery("""
            SELECT m FROM Member m
            WHERE ...
        """, Member::class.java)
            .resultList
    }
}
```

**3. 기본 리포지토리에 상속**

```kotlin
interface MemberRepository : JpaRepository<Member, Long>, MemberRepositoryCustom {
    // 기본 메소드 + 사용자 정의 메소드 모두 사용 가능
}
```

#### 사용 예시

```kotlin
@Service
@Transactional(readOnly = true)
class MemberService(
    private val memberRepository: MemberRepository
) {
    fun searchMembers(condition: SearchCondition): List<Member> {
        // 사용자 정의 메소드 호출
        return memberRepository.findMembersWithComplexLogic(condition)
    }
}
```

### 기타 기능

#### @EntityGraph

연관관계 Fetch Join 간편화

```kotlin
interface MemberRepository : JpaRepository<Member, Long> {

    @EntityGraph(attributePaths = ["team"])
    fun findAll(): List<Member>

    @EntityGraph(attributePaths = ["team", "orders"])
    fun findByAge(age: Int): List<Member>
}
```

장점:
- JPQL 없이 Fetch Join 가능
- 간단한 경우 유용

단점:
- 복잡한 조건은 @Query + FETCH JOIN 권장

#### Auditing

엔티티 생성/수정 시점 자동 기록

```kotlin
@EntityListeners(AuditingEntityListener::class)
@MappedSuperclass
abstract class BaseEntity {

    @CreatedDate
    @Column(updatable = false)
    var createdAt: LocalDateTime? = null

    @LastModifiedDate
    var updatedAt: LocalDateTime? = null

    @CreatedBy
    @Column(updatable = false)
    var createdBy: String? = null

    @LastModifiedBy
    var updatedBy: String? = null
}

@Entity
class Member(
    @Id @GeneratedValue
    val id: Long? = null,
    var name: String
) : BaseEntity()
```

**활성화**

```kotlin
@Configuration
@EnableJpaAuditing
class JpaConfig {

    @Bean
    fun auditorProvider(): AuditorAware<String> {
        // 현재 사용자 정보 제공
        return AuditorAware {
            Optional.of(SecurityContextHolder.getContext().authentication?.name ?: "system")
        }
    }
}
```

## 실무 적용

### Repository 설계 체크리스트

- [ ] 복잡한 쿼리는 @Query 또는 사용자 정의 리포지토리로 분리했는가?
- [ ] 페이징 시 count 쿼리가 최적화되어 있는가?
- [ ] 벌크 연산 후 영속성 컨텍스트를 초기화했는가?
- [ ] DTO 변환 시 필요한 컬럼만 SELECT하는가?
- [ ] @EntityGraph는 간단한 경우만 사용하는가?

### 권장 패턴

```kotlin
@Repository
interface MemberRepository : JpaRepository<Member, Long>, MemberRepositoryCustom {

    // 간단한 쿼리는 메소드 네이밍
    fun findByName(name: String): List<Member>

    // 복잡한 쿼리는 @Query
    @Query("""
        SELECT m FROM Member m
        JOIN FETCH m.team
        WHERE m.age >= :age
    """)
    fun findActiveMembers(@Param("age") age: Int): List<Member>

    // 페이징 (count 쿼리 최적화)
    @Query(
        value = "SELECT m FROM Member m JOIN FETCH m.team WHERE m.age = :age",
        countQuery = "SELECT COUNT(m) FROM Member m WHERE m.age = :age"
    )
    fun findByAgeWithPaging(@Param("age") age: Int, pageable: Pageable): Page<Member>
}

// 사용자 정의 리포지토리: QueryDSL 등 복잡한 동적 쿼리
interface MemberRepositoryCustom {
    fun searchMembers(condition: MemberSearchCondition): List<MemberDto>
}
```

### 금융권에서의 고려사항

- 대용량 조회 시 Slice 사용 (count 쿼리 생략으로 성능 향상)
- 벌크 연산은 트랜잭션 분리 고려 (타임아웃 방지)
- Auditing으로 감사 정보 자동 기록 (규제 대응)
- 복잡한 쿼리는 Native SQL보다 JPQL 선호 (DB 벤더 독립성)
- DTO 변환으로 민감 정보 노출 방지

---

**출처**
- Spring Data JPA 공식 문서
- JPA 프로그래밍 (김영한)
