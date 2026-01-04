---
tags: [jpa, spring-data-jpa, query]
---

# @Modifying

## 한 줄 요약

Spring Data JPA에서 `@Query`로 UPDATE/DELETE/INSERT 작업을 수행할 때 필수로 선언해야 하는 애노테이션

## 핵심 정리

- `@Query`는 기본적으로 SELECT만 지원
- UPDATE, DELETE, INSERT 작업에는 `@Modifying` 필수 (없으면 예외 발생)
- `@Transactional`과 함께 사용해야 함 (JpaRepository는 기본이 readOnly)
- 플러시 타이밍 제어 가능 (`clearAutomatically`, `flushAutomatically`)

## 상세 내용

### 기본 사용법

```java
public interface UserRepository extends JpaRepository<User, Long> {

    @Modifying
    @Transactional
    @Query("UPDATE User u SET u.status = :status WHERE u.id = :id")
    int updateUserStatus(@Param("id") Long id, @Param("status") String status);

    @Modifying
    @Transactional
    @Query("DELETE FROM User u WHERE u.createdDate < :date")
    int deleteOldUsers(@Param("date") LocalDateTime date);
}
```

### 왜 @Transactional이 필요한가?

Spring Data JPA의 `SimpleJpaRepository`는 기본적으로 **읽기 전용 트랜잭션**(`@Transactional(readOnly = true)`)으로 동작한다.

- SELECT 작업: readOnly 트랜잭션으로 충분
- **UPDATE/DELETE**: 쓰기 가능한 트랜잭션 필요
- `@Modifying` 메서드에 `@Transactional`을 명시하지 않으면 readOnly 트랜잭션에서 실행되어 예외 발생

```java
// ❌ 잘못된 예시
@Modifying
@Query("UPDATE User u SET u.name = :name")
int updateName(@Param("name") String name);  // TransactionRequiredException 발생

// ✅ 올바른 예시
@Modifying
@Transactional
@Query("UPDATE User u SET u.name = :name")
int updateName(@Param("name") String name);
```

### 플러시와 클리어 옵션

```java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Transactional
@Query("UPDATE User u SET u.status = 'INACTIVE' WHERE u.lastLoginDate < :date")
int deactivateInactiveUsers(@Param("date") LocalDateTime date);
```

**옵션 설명:**
- `clearAutomatically`: 쿼리 실행 후 영속성 컨텍스트를 자동으로 clear
- `flushAutomatically`: 쿼리 실행 전 변경사항을 DB에 flush

**기본값은 둘 다 false**

### clearAutomatically가 필요한 이유

```java
// 1. 엔티티 조회 (영속성 컨텍스트에 캐시됨)
User user = userRepository.findById(1L).get();
System.out.println(user.getStatus()); // "ACTIVE"

// 2. 벌크 UPDATE 실행
userRepository.updateUserStatus(1L, "INACTIVE");

// 3. 다시 조회
User sameUser = userRepository.findById(1L).get();
System.out.println(sameUser.getStatus()); // 여전히 "ACTIVE" ❌
```

벌크 연산은 영속성 컨텍스트를 거치지 않고 DB에 직접 쿼리를 실행하므로, 1차 캐시와 DB 상태가 불일치하게 된다.

**해결 방법:**
```java
@Modifying(clearAutomatically = true)  // 쿼리 실행 후 영속성 컨텍스트 clear
@Transactional
@Query("UPDATE User u SET u.status = :status WHERE u.id = :id")
int updateUserStatus(@Param("id") Long id, @Param("status") String status);
```

## 실무 적용

### 반환 타입

```java
// 영향받은 행 수 반환
@Modifying
@Transactional
@Query("DELETE FROM User u WHERE u.status = 'DELETED'")
int deleteMarkedUsers();

// void도 가능
@Modifying
@Transactional
@Query("UPDATE User u SET u.lastModified = :now")
void updateAllLastModified(@Param("now") LocalDateTime now);
```

### 벌크 연산 vs 개별 연산

**벌크 연산 (권장):**
```java
@Modifying
@Transactional
@Query("UPDATE User u SET u.point = u.point + :amount WHERE u.id IN :ids")
int addPointsBulk(@Param("ids") List<Long> ids, @Param("amount") int amount);
```

**개별 연산 (비효율):**
```java
@Transactional
public void addPoints(List<Long> ids, int amount) {
    for (Long id : ids) {
        User user = userRepository.findById(id).get();
        user.addPoint(amount);
    }
    // N번의 UPDATE 쿼리 발생
}
```

### 주의사항

1. **영속성 컨텍스트 불일치 문제**
   - 벌크 연산 후 엔티티 재조회 시 주의
   - `clearAutomatically = true` 사용 또는 수동 `entityManager.clear()` 호출

2. **성능**
   - 대량 데이터 수정 시 벌크 연산이 유리
   - 소량 데이터는 변경 감지(Dirty Checking)가 더 안전할 수 있음

3. **auditing 미적용**
   - `@LastModifiedDate` 등의 Auditing 기능이 작동하지 않음
   - 필요시 쿼리에 직접 명시

## 관련 노트

- [[@Transactional]]
- [[JPA원리]]
- [[영속성컨텍스트]]
- [[JPA-성능-최적화]]

#jpa #spring-data-jpa #bulk-operation
