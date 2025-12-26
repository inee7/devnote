---
tags: [jpa, performance, n-plus-1]
---
# JPA 성능최적화

## N+1 문제
성능상 가장 주의해야 하는것이 이 N+1 문제이다. 아래와 같은 엔티티가 있다고 가정한다.
```java
 @Entity
 class Member{
     @Id @GeneratedValue
     private Long id;

     @OneToMany(mappedBy = "member", fetch = FetchType.EAGER)
     private List<Order> orders = new ArrayList<>();
 }

 @Entity
 class Order{
     @Id @GeneratedValue
     private Long id;

     @ManyToOne
     private Member member;
 }
```

### 즉시 로딩
```java
 List<Member> members = 
     em.createQuery("SELECT m FROM Member", Member.class)
     .getResultList();
```
JPA는 fetchType을 전혀 신경쓰지 않고 충실하게 JPQL에 맞춰 SQL을 생성한다.
따라서 Member를 전체 조회하는 쿼리가 먼저 실행되고, Member의 개수만큼 Order를 조회하게 될 것이다
```java
 SELECT * FROM Member; -- if result is 5
 SELECT * FROM Order_ WHERE MEMBER_ID = 1;
 SELECT * FROM Order_ WHERE MEMBER_ID = 2;
 SELECT * FROM Order_ WHERE MEMBER_ID = 3;
 SELECT * FROM Order_ WHERE MEMBER_ID = 4;
 SELECT * FROM Order_ WHERE MEMBER_ID = 5;
```
이처럼  ~처음 실행한 SQL의 결과 수만큼 추가로 SQL을 실행~하는 것
을 N+1 문제라고 한다.
### 지연 로딩
즉시로딩을 지연로딩으로 바꿔도 N+1에서 자유로울수는 없다. 즉시로딩이 아니라서 Member 조회와 동시에 Member 건수만큼 Order를 조회해오진 않겠지만, Member에서 Order를 사용하는 시점에는 똑같이 불러오게 된다.
```java
 for(Member member : membres){
     System.out.println(member.getOrders().size());
 }
```
members 개수만큼 order 조회 SQL이 실행될 것이다. 즉, 이것도 결국 N+1 문제다.
### 해결법
#### 1. 페치 조인
가장 일반적인 방법이다. ~SQL 조인을 이용해서 연관된 엔티티를 함께 조회~하므로 N+1 문제가 발생하지 않는다. JPQL은 아래와 같다.
```sql 
SELECT m FROM Member m JOIN FETCH m.orders
```
JOIN으로 같이 조회해서 Member 엔티티의 orders 속성에 초기화 하였기 때문에 더이상 N+1이 발생하지 않는다. 참고로 위 예제는 일대다 조인이므로 결과가 늘어날 수 있다.  DISTINCT를 써줘야한다.
#### 2. 하이버네이트 @BatchSize
하이버네이트가 제공하는  org.hibernate.annotations.BatchSize
 어노테이션을 이용하면 연관된 엔티티를 조회할 때 지정된 size 만큼 SQL의 IN절을 사용해서 조회한다.
```java
 @Entity
 class Member{
     @Id @GeneratedValue
     private Long id;

     @org.hibernate.annotations.BatchSize(size = 5)
     @OneToMany(mappedBy = "member", fetch = FetchType.EAGER)
     private List<Order> orders = new ArrayList<>();
 }
```
즉시로딩이므로 Member를 조회하는 시점에 Order를 같이 조회한다.  @BatchSize
가 있으므로 Member의 건수만큼 추가 SQL을 날리지 않고, 조회한 Member 의 id들을 모아서 SQL IN 절을 날린다.
```sql
 SELECT * FROM
 ORDER_ WHERE MEMBER_ID IN(
     ?, ?, ?, ?, ?
 )
```
size는 IN절에 올수있는 최대 인자 개수를 말한다. 만약 Member의 개수가 10개라면 위의 IN절이 2번 실행될것이다.
그리고 만약 지연로딩이라면 지연로딩된 엔티티 최초 사용시점에 5건을 미리 로딩해두고, 6번째 엔티티 사용 시점에 다음 SQL을 추가로 실행한다.
`hibernate.default_batch_fetch_size` 속성을 사용하면 애플리케이션 전체에 기본으로 @BatchSize를 적용할 수 있다.
#### 3. 하이버네이트 @Fetch(FetchMode.SUBSELECT)
연관된 데이터를 조회할 때 서브쿼리를 사용해서 N+1 문제를 해결한다
```java
 @Entity
 class Member{
     @Id @GeneratedValue
     private Long id;

     @org.hibernate.annotations.Fetch(FetchMode.SUBSELECT)
     @OneToMany(mappedBy = "member", fetch = FetchType.EAGER)
     private List<Order> orders = new ArrayList<>();
 }
```
아래와 같이 실행된다.
```sql
 SELECT * FROM Member;
 SELECT * FROM Order_
     WHERE MEMBER_ID IN(
         SELECT ID
         FROM Member
     )
```
즉시로딩으로 설정하면 조회시점에, 지연로딩으로 설정하면 지연로딩된 엔티티를 사용하는 시점에 위의 쿼리가 실행된다.

**모두 지연로딩으로 설정하고 성능 최적화가 필요한 곳에는 JPQL 페치 조인을 사용하는 것이 추천**되는 전략이다.

## 읽기 전용 쿼리의 성능 최적화
JPA의 [[영속성컨텍스트]]는 변경 감지를 위해 스냅샷 인스턴스를 보관하는 특징이 있다. 
단순 조회 화면에서는 조회한 엔티티를 다시 조회할 필요도 없고, 수정할 필요도 없어서 이때는 스냅샷 인스턴스를 위한 메모리가 낭비된다. 이럴 경우 아래의 방법으로 메모리 사용량을 최적화할 수 있다.

### 스칼라 타입으로 조회
엔티티가 아닌 스칼라 타입으로 모든 필드를 조회하는 것이다. 알다시피 스칼라 타입은 영속성 컨텍스트가 관리하지 않는다.
```sql
SELECT m.id, m.name, m.age FROM Member m
```

### 읽기 전용 쿼리 힌트 사용
하이버네이트 전용 힌트인  `org.hibernate.readOnly`를 사용하면 엔티티를 읽기 전용으로 조회할 수 있다. 읽기 전용이므로 영속성 컨텍스트가 스냅샷을 저장하지 않으므로 메모리 사용량을 최적화 할 수 있다.
```java
 Member member = em.createQuery("SELECT m FROM Member m", Member.class)
     .setHint("org.hibernate.readOnly", true)
     .getSingleResult();
```
스냅샷이 없으므로 member의 값을 수정해도 update 쿼리가 발생하지 않는다.
스냅샷만 저장하지 않는 것이지, 1차 캐시에는 그대로 저장한다 똑같은 식별자로 2번 조회했을 경우 반환되는 엔티티의 주소가 같다

### 읽기 전용 트랜잭션 사용
스프링 프레임워크를 사용하면 트랜잭션을 읽기 전용 모드로 설정할 수 있다.
`[[@Transactional]](readOnly = true)`
트랜잭션을 읽기 전용으로 설정하면 스프링 프레임워크가 하이버네이트 세션의 플러시 모드를  MANUAL로 설정한다. 이렇게하면 강제로 플러시 호출을 하지 않는 한 플러시가 일어나지 않는다.
엔티티의 플러시 모드는 AUTO, COMMIT 모드만 있다. MANUAL 모드는 하이버네이트 세션에 있는 플러시모드이다. 이는 강제로 플러시를 호출하지 않으면 절대 플러시가 일어나지 않는 특징을 가지고 있다. 하이버네이트 세션은 JPA 엔티티 매니저를 하이버네이트로 구현한 구현체이다.
플러시를 수행하지 않으므로 플러시할 때 일어나는 스냅샷 비교와 같은 무거운 로직들이 실행되지 않으므로 성능이 향상된다. (그래도 스냅샷은 그대로 저장하는 듯. 단지 플러시만 일어나지 않는 것 같다.) 물론 트랜잭션을 시작했으므로 트랜잭션 시작, 수행, 커밋의 과정은 이루어진다.
### 트랜잭션 밖에서 읽기
트랜잭션 없이 엔티티를 조회하는 것을 의미한다. JPA에서 엔티티를 변경하려면 트랜잭션이 필수이므로, 조회가 목적일 때만 사용해야 한다.
JPA는 기본적으로 아래의 2가지 특성이 있다
* 트랜잭션이 커밋될 때 영속성 컨텍스트를 플러시한다
* 영속성 컨텍스트만 있으면 트랜잭션 없이 읽기가 가능하다
```java
// 스프링을 사용하지 않을 때
 EntityManager em = emf.createEntityManger();
 // EntityTransaction tx = em.getTransaction();

 try {
     // tx.begin();

     Some some = em.find(Some.class, 1L);
     SomeChild someChild = some.getSomeChildren().get(0);
 }
```
트랜잭션을 시작하는 부분 없이 엔티티매니저만 가져와서 조회를 수행해도 정상 동작하고, 보다시피 lazy 로딩까지도 동작한다.
참고로 여기서 스프링이 @PersistenceContext를 통해 가져온 EntityManager를 사용할 경우 예외가 발생한다(org.hibernate.LazyInitializationException) 위처럼 메서드내에서 생성된 entityManager에만 유효하다(@PersistenceContext는 공유되는 애라서 그런가…) 공유되는 애인게 왜 문제가 되지?

## 스프링을 사용할 때
스프링을 사용할때는 [[OSIV]]를 사용하거나, 직접 EntityManager를 생성시켜주는 방법을 사용해야 한다. 기본적으로 @Transactional 을 통해 트랜잭션이 생성될 때 영속성 컨텍스트를 생성하는 구조이기 때문이다. 
트랜잭션을 시작하지 않을거라면 @Transactional 어노테이션을 제거해줘야하는데, 그러면 스프링 입장에서 언제 영속성 컨텍스트를 생성해줘야 할지 알지 못하게 된다 그러므로 OSIV를 사용하지 않을거면EntityManagerFactory를 가져온 뒤 직접 엔티티 매니저를 생성해줘야만 트랜잭션 없이 읽기가 가능하다. 이 과정 없이 트랜잭션 없이 읽기를 사용하려면(@Transactional 어노테이션을 제거해서) OSIV를 켜줘야한다.

#jpa 

## 관련 노트
- [[JPA원리]]