---
tags: [jpa, persistence-context]
---
# JPA의 원리

### EntityManagerFactory란

EntityManagerFactory는 데이터베이스 연동하고 커텍션 풀도 생성하여 생성 비용이 매우 크다.
데이터베이스를 하나만 사용하는 애플리케이션들은 일반적으로 _EntityManagerFactory_ 를 **한개만** 생성한다.
이 *EntityManagerFactory*로 *EntityManager*를 생성할 수 있다.
*EntityManagerFactory*는 **Thread Safe**해서 여러 스레드가 동시에 접근해도 안전하므로 서로 다른 스레드간에 공유해 사용한다.
(Hibernate에서는 SessionFactory)

### EntityManager란

EntityManager는 Entity를 저장하는 **메모리상의 데이터베이스**라고 생각하면 될 것같다.
EntityManager는 커넥션풀을 얻어 DB에 접근하여 Entity를 저장하고 수정하고 삭제하고 조회하는 등 엔터티와 관련된 모든일을 한다.
하지만 위의 EntityManagerFactory는 달리 Thread Safe하지 않기 때문에 동시성 문제가 발생하므로 스레드간에 절대 공유하면 안 된다.
그래서 일반적으로 EntityManager를 EntityManagerFactory를 이용하여 생성한것을 사용하는것이 아닌 스프링에서 관리하는 EntityManager를 아래와 같이 선언하여 사용한다.

```java
@PersistenceContext
//이렇게되면 스프링에서 알아서 EntityManager를 Proxy로 깜싼 EntityManager를 생성 하여 주입해주기 때문에 Thread-Safety를 보장 한다.
private EntityManager entityManager;
```

(Hibernate에서는 Session)
요청에 의해 EntityManager가 생성되면 [[@Transactional]]을 시작할때 커넥션풀에서 데이터베이스 연결을 한다.
http request thread -> EntityManager Factory 가 EM 생성 -> EM이 트랜잭션 시작하고 커텍션풀에서 데이타베이스 연결 -> 처리

### [[영속성컨텍스트]]란

- 영속성 컨텍스트란 엔티티(Entity)를 **영구 저장하는 환경**을 말한다. EntityManager로 엔티티를 저장하거나 조회하면 EntityManager는 PersistenceContext에 엔티티를 보관하고 관리한다.
- PersistenceContext는 **EntityManager(Session)를 생성할 때 하나 만들어진다.** 그리고 엔티티 매니저(Session)를 통해서 영속성 컨텍스트에 접근할 수 있고 영속성 컨텍스트를 관리 할 수 있다.
- 여러 엔티티 매니저(Session)가 같은 영속성 컨텍스트에 접근할 수도 있다.

### 엔티티 생명주기

- 비영속(new/transient) : 영속성 컨텍스트와 전혀 관계 없음
- 영속(merged): 컨텍스트에 저장됨
- 준영속(detached): 컨텍스트에 저장되었다가 분리됨
- 삭제(removed): 삭제된 상태

### 영속성 컨텍스트를 통한 엔티티 관리로 인한 장점

- 1차캐시
  - 영속성컨텍스트에 있으면 네트워크, 디스크 안 타도 된다.
- 동일성 보장
  - 영속성컨텍스트에 있는 객체면 같은 객체를 항상 리턴
- 쓰기지연
  - 트랜잭션 커밋시에 영속컨텍스트의 쿼리 저장소에 차곡 쌓인 Insert sql문을 엔티티를 역 DB에 보낸다.
  - 트랜잭션 커밋 전에는 쿼리가 절대 실행되지 않는다.
- 변경감지 (dirty checking)
  - 엔티티를 영속성 컨텍스트에 보관할 때, 최초 상태를 복사해서 저장해둔다. 스냅샷.
  - 플러시 시점에 스냅샷과 엔티티 상태를 비교해서 변경된 엔티티를 찾는다.
  - 변경된 엔티티의 수정 쿼리를 생성해서 쿼리 저장소에 보낸다.
  - 쿼리 저장소가 DB에 보내고 트랜잭션을 커밋한다.
  - 이러한 메커니즘은 [[JPA 성능최적화]]와 깊은 관련이 있다.
- 변경시 모든 필드 변경 됨.
  - 쿼리를 재사용 할 수 있는 장점.
  - 애플리케이션 로딩시점에 쿼리 만들어 놓는다.
  - 너무 많은 필드면 @DynamicUpdate사용 // 변경한 필드만 대응
- 삭제도 지연된다.
  - entitymanager.remove(e) 할 때 쿼리 저장소에 등록하고 커밋될 때 쿼리 전달.
  - 영속성컨텍스트에서 제거도 되는데 GC가 이뤄지려면 해당 객체를 참조 하지 않아야 한다.

### 플러시 시점

- em.flush() <- 테스트 외 사용하지 않음
- JPQL 실행시 자동으로 flush
  - 왜 이렇게 만들었냐면? 만약 트랜잭션이 끝나지 전에 영속화컨텍스트에만 있는 엔티티를 바로 SQL날리게 설계된 JPQL로 조회하면 DB에서 못 찾기 때문. 그래서 자동으로 flush (commit모드를 쓰고 싶을때는 트랜잭션 안에서 무결성이 지켜지는데 너무 많은 플러시가 일어날때 최적화)
- 식별자를 기준으로 조회하는 find()메소드는 flush 실행 되지 않음

#jpa 