---
tags: [spring, transaction, jpa]
---
# @Transactional 에 대하여 
### @Transactional은 인터페이스가 아닌 클래스에 걸기를 스프링 문서에서 권장하고 있다.

### Read Only

- @Transactional(readOnly=true)는 JDBC의 Connection.setReadOnly(true)를 호출한다.
- 이는 단지 “힌트”일 뿐이며 실제 데이터베이스 자체를 read-only 트랜잭션으로 생성할지 여부는 JDBC 드라이버에 따라 달라지고 행위도 데이터 계층 프레임워크의 구현여부에 따라 달라질 수 있다.
- 이 힌트로 정말로 read only 트랜잭션이 생성될 수도 있고, MySQL 리플리케이션 JDBC 드라이버의 경우 Write가 있을 경우에는 Master로 read only일 경우에는 Slave로 커넥션을 맺는 식의 선택을 하는 정도의 역할만 할 수도 있다.
- Oracle JDBC 드라이버는 Read-Only 트랜잭션을 지원하며, MySQL의 경우 5.6.5 버전이 되어서야 이를 지원하게 되었다.
- Spring-Hibernate 연동시에는 Session의 Flush Mode를 FLUSH_NEVER로 변경한다. 그러나 데이터가 수정되면 수정된 상태로 세션의 1차 캐시에는 수정된 상태로 데이터가 남아있게 되면, 명시적으로 flush()를 호출하면 쓰기 기능이 작동하게 된다.
- 단, 읽기 작업만 하더라도 트랜잭션을 걸어주는 것이 좋다. 트W랜잭션을 걸지 않으면 모든 SELECT 쿼리마다 commit을 하기 때문에 성능이 떨어진다. 명시적으로 트랜잭션을 걸어주면 마지막에 명시적으로 commit을 해주면 되며, commit 횟수가 줄어서 성능이 좋아진다. 이러한 읽기 전용 최적화에 대한 더 자세한 내용은 [[JPA 성능최적화]]에서 확인할 수 있다.

### Chained Transaction Manager

- [ChainedTransactionManager](http://docs.spring.io/spring-data/commons/docs/1.6.2.RELEASE/api/org/springframework/data/transaction/ChainedTransactionManager.html) Spring Data 에 추가된 다중 트랜잭션을 연결해서 관리하는 트랜잭션 매니저.
- JTA 처럼 트랜잭션이 완전히 동기화 되는 것은 아니고, 여러 트랜잭션을 묶어서 순서대로 커밋/롤백을 진행하는 방식.
- JTA 수준은 아니더라도 여러 트랜잭션이 생성되는 애플리케이션에서 각자 @Transactional을 지정하지 않고 하나로 통일하고자 하는 경우에 괜찮을 듯.
- Chained Transaction Manager로 묶이는 **각 트랜잭션의 Connection Pool 의 min/max 갯수는 동일하게** 가져가야한다.
- 경험상, 서로 다른 동기화 시점을 가지는 트랜잭션은 묶지 않는 것이 좋다.
    - MQ 사용시, MQ로 받은 데이터를 저장하는데 DB 접속이 끊겼을 경우 retry 를 하고자 한다면 MQ 트랜잭션과 DB 트랜잭션을 함께 묶으면 이 작업이 안된다. DB 트랜잭션이 롤백 될때 MQ 트랜잭션까지 롤백 되어 버려서 재시도할 트랜잭션이 없어지기 때문이다.

### Propagation.REQUIRED

- 특정 메서드의 트랜잭션이 Propagation.REQUIRED로 설정되었을 때의 트랜잭션 동작은 다음과 같다. 기본적으로 해당 메서드를 호출한 곳에서 별도의 트랜잭션이 설정되어 있지 않았다면 트랜잭션를 새로 시작한다.(새로운 연결을 생성하고 실행한다.) 만약, 호출한 곳에서 이미 트랜잭션이 설정되어 있다면 기존의 트랜잭션 내에서 로직을 실행한다.(동일한 연결 안에서 실행된다.) 예외가 발생하면 롤백이 되고 호출한 곳에도 롤백이 전파된다. 이러한 **Propagation.REQUIRED** 동작 방식을 원할 경우 기본값으로 설정되어 있기 때문에 생략해도 된다. ~[[관련 링크]](https://stackoverflow.com/a/10740405)~
- 만약, 해당 메써드가 호출한 곳와 별도의 쓰레드라면 어떤 동작이 일어날까? 답은 전파 레벨과 상관 없이 무조건 별도의 트랜잭션을 생성하여 해당 메써드를 실행한다. **Spirng**은 내부적으로 트랜잭션 정보를 **ThreadLocal** 변수에 저장하기 때문에 다른 쓰레드로 트랜잭션이 전파되지 않는다. ~[[관련 링크1]](https://dzone.com/articles/spring-and-threads-transactions)~ ~[[관련 링크2]](https://stackoverflow.com/a/11277682)~ [[관련 링크3]](~[](https://stackoverflow.com/a/11276195())[https://stackoverflow.com/a/11276195()](https://stackoverflow.com/a/11276195())~

### Propagation.REQUIRES_NEW

- Propagation.REQUIRES_NEW로 설정되었을 때에는 매번 새로운 트랜잭션을 시작한다.(새로운 연결을 생성하고 실행한다.) 만약, 호출한 곳에서 이미 트랜잭션이 설정되어 있다면(기존의 연결이 존재한다면) 기존의 트랜잭션은 메써드가 종료할 때까지 잠시 대기 상태로 두고 자신의 트랜잭션을 실행한다. 새로운 트랜잭션 안에서 예외가 발생해도 호출한 곳에는 롤백이 전파되지 않는다. 즉, 2개의 트랜잭션은 완전히 독립적인 별개로 단위로 작동한다. [[관련 링크1]](https://stackoverflow.com/a/25083505) [[관련 링크2]](https://stackoverflow.com/a/12391308)

### Propagation.NESTED

- Propagation.NESTED는 기본적으로 앞서 설명한 **Propagation.REQUIRED**와 동일하게 작동한다. 중요한 차이점은, **SAVEPOINT**를 지정한 시점까지 부분 롤백이 가능하다는 것이다. 유의할 점은, 데이터베이스가 **SAVEPOINT** 기능을 지원해야 사용이 가능하다. (대표적으로 **Oracle**이 해당한다.)

### 어노테이션이 중복으로 적용될 경우 메소드 < 클래스 < 인터페이스 순으로 우선순위

#spring #Transactional 

## 관련 노트
- [[JPA원리]]