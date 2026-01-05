---
tags: [jpa, 영속성컨텍스트, entitymanager, 플러시, 트랜잭션]
---

## 한 줄 요약

JPA는 EntityManager와 영속성 컨텍스트를 통해 엔티티의 생명주기를 관리하며, 1차 캐시, 쓰기 지연, 변경 감지 등의 이점을 제공한다.

## 핵심 정리

- EntityManagerFactory는 Thread-Safe하며 애플리케이션 전체에서 하나만 생성
- EntityManager는 Thread-Safe하지 않으며 트랜잭션마다 생성
- 영속성 컨텍스트는 엔티티를 영구 저장하는 메모리상의 환경
- 1차 캐시, 동일성 보장, 쓰기 지연, 변경 감지(Dirty Checking)가 주요 장점
- 플러시는 영속성 컨텍스트의 변경 내용을 DB에 반영 (커밋 시점, JPQL 실행 시)
- GenerationType.IDENTITY는 persist 시점에 즉시 INSERT 발생

## 상세 내용

### EntityManagerFactory

#### 생성과 특징

EntityManagerFactory는 데이터베이스 연결을 관리하고 커넥션 풀을 생성하는 무거운 객체다.

```java
// 애플리케이션 전체에서 하나만 생성
EntityManagerFactory emf = Persistence.createEntityManagerFactory("myPersistenceUnit");
```

특징:
- **Thread-Safe**: 여러 스레드가 동시 접근 가능
- **싱글톤**: 데이터베이스 하나당 하나만 생성
- **생성 비용 큼**: 초기화 시 데이터베이스 연결, 커넥션 풀 생성
- **Hibernate에서는 SessionFactory**

### EntityManager

#### 역할

EntityManager는 엔티티를 저장하는 **메모리상의 데이터베이스**라고 생각하면 된다.

```java
// EntityManagerFactory에서 생성
EntityManager em = emf.createEntityManager();
```

주요 역할:
- 커넥션 풀에서 DB 연결 획득
- 엔티티 저장, 수정, 삭제, 조회 (CRUD)
- 영속성 컨텍스트 관리

#### Thread-Safety

**주의: EntityManager는 Thread-Safe하지 않음!**

```java
// 나쁜 예: 여러 스레드가 같은 EntityManager 공유
private static EntityManager em;  // 절대 금지!

// 좋은 예: Spring에서 관리
@PersistenceContext  // Thread-Safe한 Proxy로 감싸서 주입
private EntityManager em;
```

Spring의 `@PersistenceContext`:
- Spring이 EntityManager를 Proxy로 감싸서 주입
- 요청마다 올바른 EntityManager에 위임
- Thread-Safety 보장

#### 트랜잭션과 커넥션

```
HTTP Request → EntityManagerFactory가 EntityManager 생성
→ EntityManager가 트랜잭션 시작
→ 커넥션 풀에서 데이터베이스 연결
→ 처리
```

**중요**: 트랜잭션 시작 시점에 커넥션을 획득한다.

### 영속성 컨텍스트 (Persistence Context)

#### 정의

영속성 컨텍스트란 **엔티티를 영구 저장하는 환경**을 말한다.

```kotlin
// 엔티티를 영속성 컨텍스트에 저장
em.persist(member)  // member가 영속 상태가 됨
```

특징:
- EntityManager 생성 시 하나 만들어짐
- EntityManager를 통해 영속성 컨텍스트에 접근
- 여러 EntityManager가 같은 영속성 컨텍스트에 접근할 수도 있음 (트랜잭션 범위 전략)

### 엔티티 생명주기

```
비영속 (new/transient)
    ↓ em.persist()
영속 (managed)
    ↓ em.detach() / 트랜잭션 종료
준영속 (detached)
    ↓ em.remove()
삭제 (removed)
```

**1. 비영속 (New/Transient)**

```kotlin
val member = Member(name = "회원1")  // 비영속 상태
// 영속성 컨텍스트와 무관
```

**2. 영속 (Managed)**

```kotlin
em.persist(member)  // 영속 상태
// 영속성 컨텍스트가 관리
```

**3. 준영속 (Detached)**

```kotlin
em.detach(member)  // 준영속 상태
// 또는 트랜잭션 커밋 후 자동으로 준영속
```

**4. 삭제 (Removed)**

```kotlin
em.remove(member)  // 삭제 상태
// DB에서도 삭제 예정
```

### 영속성 컨텍스트의 이점

#### 1. 1차 캐시

```kotlin
// 1차 캐시에 저장
em.persist(member)

// 1차 캐시에서 조회 (DB 접근 안 함)
val found = em.find(Member::class.java, member.id)
println(found === member)  // true (같은 객체)
```

장점:
- 네트워크, 디스크 I/O 없이 조회
- 같은 트랜잭션 내에서 반복 조회 시 성능 향상

#### 2. 동일성 보장

```kotlin
val member1 = em.find(Member::class.java, 1L)
val member2 = em.find(Member::class.java, 1L)

println(member1 === member2)  // true
```

같은 식별자를 가진 엔티티는 항상 같은 객체를 반환 (Repeatable Read 격리 수준)

#### 3. 쓰기 지연 (Transactional Write-Behind)

```kotlin
em.persist(member1)  // 쿼리 저장소에 INSERT 추가
em.persist(member2)  // 쿼리 저장소에 INSERT 추가
em.persist(member3)  // 쿼리 저장소에 INSERT 추가

// 여기까지 DB에 쿼리 전송 안 함

transaction.commit()  // 커밋 시점에 한 번에 전송
```

장점:
- 네트워크 왕복 횟수 감소
- JDBC Batch를 활용한 성능 최적화 가능

#### 4. 변경 감지 (Dirty Checking)

```kotlin
@Transactional
fun updateMember(id: Long, newName: String) {
    val member = em.find(Member::class.java, id)
    member.name = newName
    // em.persist() 불필요!
}
```

동작 원리:
1. 영속성 컨텍스트에 엔티티 저장 시 **스냅샷** 복사
2. 플러시 시점에 스냅샷과 현재 엔티티 비교
3. 변경 사항 발견 시 UPDATE 쿼리 생성
4. 쿼리 저장소에 보내고 트랜잭션 커밋

**변경 시 모든 필드 UPDATE**

```sql
-- 기본적으로 모든 필드 업데이트
UPDATE member SET name = ?, age = ?, email = ?, ...
WHERE id = ?
```

이유:
- 쿼리 재사용 가능 (PreparedStatement 캐싱)
- 애플리케이션 로딩 시점에 미리 생성

필드가 너무 많으면:

```kotlin
@Entity
@DynamicUpdate  // 변경된 필드만 UPDATE
class Member { ... }
```

#### 5. 삭제도 지연

```kotlin
em.remove(member)  // 쿼리 저장소에 DELETE 추가
// 커밋 시점에 DELETE 실행
```

주의:
- 영속성 컨텍스트에서도 제거됨
- GC가 되려면 다른 곳에서 참조하지 않아야 함

### 플러시 (Flush)

#### 플러시란?

영속성 컨텍스트의 변경 내용을 데이터베이스에 **반영**하는 것

동작:
1. 변경 감지 (Dirty Checking)
2. 수정된 엔티티를 쿼리 저장소에 등록
3. 쿼리 저장소의 쿼리를 DB에 전송

**주의**: 플러시해도 영속성 컨텍스트는 지워지지 않음

#### 플러시 발생 시점

**1. em.flush() 직접 호출**

```kotlin
em.persist(member)
em.flush()  // 즉시 DB에 반영
// 테스트 외에는 거의 사용 안 함
```

**2. 트랜잭션 커밋 시 자동**

```kotlin
em.persist(member)
transaction.commit()  // 자동으로 flush 후 commit
```

**3. JPQL 실행 시 자동**

```kotlin
em.persist(memberA)
em.persist(memberB)
em.persist(memberC)

// JPQL은 DB를 직접 조회하므로 자동 flush
val members = em.createQuery("SELECT m FROM Member m", Member::class.java)
    .resultList  // 3개 모두 조회됨
```

이유:
- JPQL은 SQL로 변환되어 DB를 직접 조회
- 영속성 컨텍스트에만 있으면 조회 안 됨
- 따라서 JPQL 실행 전 자동 flush

**find() 메소드는 flush 안 함**

```kotlin
em.persist(member)
val found = em.find(Member::class.java, member.id)  // 1차 캐시에서 조회
// flush 발생하지 않음
```

식별자 기준 조회는 1차 캐시에서 찾으므로 flush 불필요

#### 플러시 모드

```kotlin
em.setFlushMode(FlushModeType.AUTO)  // 기본값, 자동 flush
em.setFlushMode(FlushModeType.COMMIT)  // 커밋 시에만 flush
```

**COMMIT 모드 사용 케이스**:
- 트랜잭션 내에서 무결성이 지켜지지만
- 너무 많은 flush가 발생하여 성능 문제
- 최적화 목적으로 사용

### 특수한 경우들

#### GenerationType.IDENTITY와 즉시 INSERT

```kotlin
@Entity
class Member(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long? = null,
    var name: String
)

// persist 시점에 즉시 INSERT 발생!
em.persist(member)
println(member.id)  // 이미 ID가 할당됨
```

이유:
- 영속성 컨텍스트는 엔티티를 식별자로 관리
- IDENTITY 전략은 DB가 ID 생성
- ID를 얻으려면 DB에 INSERT 필수
- 따라서 persist 시점에 즉시 INSERT

#### JPQL은 1차 캐시를 먼저 찾지 않음

```kotlin
em.persist(member)  // 1차 캐시에 저장

// JPQL은 DB 조회 후 1차 캐시 확인
val found = em.createQuery("SELECT m FROM Member m WHERE m.id = :id", Member::class.java)
    .setParameter("id", member.id)
    .singleResult
```

동작:
1. DB 조회 (최신 데이터)
2. 1차 캐시에 같은 ID 있는지 확인
3. 있으면 DB 결과 버리고 캐시 반환 (동일성 보장)
4. 없으면 DB 결과를 캐시에 저장

이유:
- Unrepeatable Read 격리 수준 지원
- 같은 트랜잭션 내에서 동일한 결과 보장

## 실무 적용

### Spring에서 EntityManager 사용

```kotlin
@Repository
class MemberRepository(
    @PersistenceContext  // Thread-Safe한 Proxy 주입
    private val em: EntityManager
) {
    fun save(member: Member): Member {
        em.persist(member)
        return member
    }

    fun findById(id: Long): Member? {
        return em.find(Member::class.java, id)
    }
}
```

### 변경 감지 활용

```kotlin
@Service
@Transactional
class MemberService(
    private val memberRepository: MemberRepository
) {
    fun updateName(id: Long, newName: String) {
        val member = memberRepository.findById(id)
            ?: throw EntityNotFoundException()

        // 값만 변경, persist 불필요
        member.name = newName

        // 트랜잭션 커밋 시 자동으로 UPDATE
    }
}
```

### 금융권에서의 고려사항

- 트랜잭션 범위를 최소화 (커넥션 점유 시간 단축)
- 읽기 전용 조회는 `@Transactional(readOnly = true)` 사용
- IDENTITY 전략은 Batch Insert 불가하므로 Sequence 권장
- 1차 캐시는 트랜잭션 범위이므로 애플리케이션 레벨 캐시 별도 고려



---

**출처**
- JPA 프로그래밍 (김영한)
- Hibernate Documentation
