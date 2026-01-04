---
tags: [spring, transaction, jpa]
---

# @Transactional

## 한 줄 요약

Spring에서 선언적 트랜잭션 관리를 제공하는 애노테이션으로, 메서드/클래스 레벨에서 트랜잭션 경계와 전파 방식을 제어

## 핵심 정리

- 클래스 레벨 적용 권장 (인터페이스보다는 구현 클래스에)
- `readOnly`는 JDBC 힌트일 뿐, 실제 동작은 DB/드라이버마다 다름
- 전파 레벨: `REQUIRED` (기본), `REQUIRES_NEW`, `NESTED`
- ThreadLocal 기반이라 다른 스레드로 전파되지 않음
- 우선순위: 메서드 > 클래스 > 인터페이스

## 상세 내용

### 기본 사용법

Spring 문서에서는 **인터페이스가 아닌 구현 클래스에 `@Transactional`을 적용**하기를 권장한다.

### Read Only 옵션

- @Transactional(readOnly=true)는 JDBC의 Connection.setReadOnly(true)를 호출한다.
- 이는 단지 “힌트”일 뿐이며 실제 데이터베이스 자체를 read-only 트랜잭션으로 생성할지 여부는 JDBC 드라이버에 따라 달라지고 행위도 데이터 계층 프레임워크의 구현여부에 따라 달라질 수 있다.
- 이 힌트로 정말로 read only 트랜잭션이 생성될 수도 있고, MySQL 리플리케이션 JDBC 드라이버의 경우 Write가 있을 경우에는 Master로 read only일 경우에는 Slave로 커넥션을 맺는 식의 선택을 하는 정도의 역할만 할 수도 있다.
- Oracle JDBC 드라이버는 Read-Only 트랜잭션을 지원하며, MySQL의 경우 5.6.5 버전이 되어서야 이를 지원하게 되었다.
- Spring-Hibernate 연동시에는 Session의 Flush Mode를 FLUSH_NEVER로 변경한다. 그러나 데이터가 수정되면 수정된 상태로 세션의 1차 캐시에는 수정된 상태로 데이터가 남아있게 되면, 명시적으로 flush()를 호출하면 쓰기 기능이 작동하게 된다.
- **읽기 작업만 하더라도 트랜잭션을 거는 것이 좋다.** 트랜잭션을 걸지 않으면 모든 SELECT 쿼리마다 commit을 하기 때문에 성능이 떨어진다. 명시적으로 트랜잭션을 걸어주면 마지막에 한 번만 commit하면 되어 성능이 향상된다. 이러한 읽기 전용 최적화에 대한 더 자세한 내용은 [[JPA 성능최적화]]에서 확인할 수 있다.

### 트랜잭션 전파 (Propagation)

#### Propagation.REQUIRED (기본값)

- 호출한 곳에서 트랜잭션이 없으면 **새로운 트랜잭션 시작** (새 DB 연결 생성)
- 호출한 곳에서 이미 트랜잭션이 있으면 **기존 트랜잭션에 참여** (동일한 연결 사용)
- 예외 발생 시 **롤백이 호출한 곳에도 전파됨**
- 기본값이므로 생략 가능

#### Propagation.REQUIRES_NEW

- 항상 **새로운 트랜잭션을 시작** (새 DB 연결 생성)
- 호출한 곳에 트랜잭션이 있으면 **기존 트랜잭션은 일시 중단**되고 메서드 종료 후 재개
- 예외가 발생해도 **호출한 곳으로 롤백이 전파되지 않음**
- 두 트랜잭션은 **완전히 독립적**으로 작동

#### Propagation.NESTED

- 기본적으로 `REQUIRED`와 동일하게 작동
- **SAVEPOINT**를 지정하여 **부분 롤백** 가능
- 데이터베이스가 SAVEPOINT 기능을 지원해야 함 (Oracle 등)

### 스레드와 트랜잭션

별도의 스레드에서 호출되는 경우, 전파 레벨과 무관하게 **항상 새로운 트랜잭션을 생성**한다.

Spring은 내부적으로 트랜잭션 정보를 **ThreadLocal** 변수에 저장하기 때문에 다른 스레드로 트랜잭션이 전파되지 않는다.

## 실무 적용

### Chained Transaction Manager

[ChainedTransactionManager](http://docs.spring.io/spring-data/commons/docs/1.6.2.RELEASE/api/org/springframework/data/transaction/ChainedTransactionManager.html)는 Spring Data에 추가된 다중 트랜잭션을 연결해서 관리하는 트랜잭션 매니저다.

**특징:**
- JTA처럼 트랜잭션이 완전히 동기화되는 것은 아니고, 여러 트랜잭션을 묶어서 순서대로 커밋/롤백을 진행
- 여러 트랜잭션이 생성되는 애플리케이션에서 각자 `@Transactional`을 지정하지 않고 하나로 통일하고자 하는 경우에 사용 가능
- **각 트랜잭션의 Connection Pool의 min/max 개수는 동일하게** 유지해야 함

**주의사항:**
- 서로 다른 동기화 시점을 가지는 트랜잭션은 묶지 않는 것이 좋다
- 예: MQ 트랜잭션과 DB 트랜잭션을 함께 묶으면, DB 롤백 시 MQ 트랜잭션까지 롤백되어 재시도 불가

### 어노테이션 우선순위

`@Transactional`이 중복으로 적용될 경우: **메서드 > 클래스 > 인터페이스** 순으로 우선순위가 적용된다.

## 관련 노트

- [[JPA-영속성-관리]] - 트랜잭션과 영속성 컨텍스트의 범위
- [[OSIV]] - 트랜잭션 범위와 OSIV
- [[@Modifying]] - 벌크 연산과 트랜잭션