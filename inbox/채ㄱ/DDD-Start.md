# DDD-Start

# Domain Driven Development

## 도메인
구현해야할 소프트웨어의 대상 , 해결해야할 문제 영역

한 도메인은 다시 하위 도메인으로 나눔

온라인 서점 도메인은 주문, 회원, 혜택, 결제, 카탈로그… 등으로 도메인 나뉨

도메인은 자체 시스템이 아니라 외부 시스템을 연동하기도 한다

### 도메인모델
특정 도메인을 개념적으로 표현

클래스 다이어그램, 상태 다이어그램 등 uml을 쓰기도 한다

도메인에 따라 용어의 의미가 결정되므로 여러 하위 도메인을 하나의 다이어그램에 모델링하면 안된다. 한 도메인 별로 모델링

### 도메인모델 패턴
엔터프라이즈 애플리케이션 아키텍처 패턴 책에서 나옴 - 마틴파울러

’출고전에 배송지를 변경할 수 있다’와 같은 규칙을 구현한 코드가 도메인 계층에 위치하게 된다. 이런 도메인 규칙을 객체 지향 기법으로 구현하는 패턴이 도메인 모델 패턴.

### 도메인모델 도출
기획서,유스케이스,사용자스토리 같은 요구사항 분석을 통해 도메인을 이해하고 이를 바탕으로 도메인 모델 초안을 만들어야 비로소 코드로 작성 가능하다.

화이트보드,종이,툴 어떻게든 초기 모델을 만들어 보면 된다.

모델링할 때 기본이 되는 작업은 모델을 구성하는 핵심 `구성요소`,`규칙`, `기능`을 찾는 것.

그렇게 도메인 클래스를 규칙과 구성요소 기능에 맞게 만들고 도메인에 대한 이해가 더 깊게 되면 해당 메서드명도 더 도메인에 맞게 구체화 될것이다.

ex) 최초에는 ’배송지 정보 변경 가능 여부 확인’을 의미하는 isShippingChangeable라는 이름을 사용했다가 요구사항을 분석하면서 배송지 정보 변경과 주문 취소가 둘 다’출고전에 가능’하다는 제약 조건을 알게 되면서 verifyNotYetShipped로 변경

> 문서화를 하는 주된 이유는 지식을 공유하기 위함.코드를 보면서 도메인을 깊게 이해하게 되므로 코드 자체도 문서화의 대상이 됨.도메인 지식이 잘 묻어나도록 코드를 작성하면 코드의 가독성도 높아져 문서로서 의미를 갖는다

도출한 모델은 크게 엔티티와 벨류로 구분

### 엔티티와 벨류

### 엔티티

* `식별` 가능한 객체이다.
* 식별자를 통해 equals()와 hashCode()메서드로 구분한다.
* 식별자는 도메인 특징과 기술에 따라 달라진다.
  * 특정 규칙에 따라 생성
  * UUID 사용
  * 값을 직접 입력
  * 일련번호 (DB 자동 증가값)

### 벨류

* 개념적으로 완전히 하나를 표현
* 식별자 없음
* `불변(immutable)`
  * 불변객체는 참조 투명성과 스레드에 안전한 특징을 가지고 있음
* 속성값을 통해 equals()와 hashCode()메서드로 구분한다.

엔티티의 식별자가 벨류타입이면 변수명을 id라 해도 타입 자체로 의미를 알 수 있다.

ex) long orderNo-> OrderNo id

### 도메인 모델에 set 메서드 넣지 말자
도메인 객체를 생성할때 불완전한 상태가 되기 때문에 무의미한 set을 피하라.

생성자를 통해 필요한 데이터를 받고 그안에 내부 메소드로 validate 할 수 있다.

> DTO와 get/set프레젠테이션 계층과 도메인 계층이 데이터를 서로 주고 받을때 사용하는 DTO는 오래전에 set메서드를 필요로 했기 때문에 어쩔수 없이 get/set을 구현했다.DTO가 도메인 로직을 담고 있지는 않는다면 get/set 메서드가 도메인 객체의 일관성에 영향을 줄 가능성은 약하다.그래도 요즘 프레임워크에서 필드에 직접 접근해서 할당하는 기능이 있기때문에 set을 만들지 말자. 그렇게 하면 DTO는 불변 객체가 되어 이점이 있다.DTO레이어-도메인레이어 분리 필요

### 도메인용어
코드를 작성할 때 도메인에서 사용하는 용어는 매우 중요하다.

도메인에서 사용하는 용어를 코드에 반영하지 않으면 의미를 해석해야하는 부담을 준다.

## 아키텍처
전형적으로 [표현-응용-도메인-인프라] 이다.

표현 영역은 흔히 스프링MVC의 기술을 이용한다.

표현영역은 HTTP 요청 파라미터를 객체타입으로 받아서 응용영역에 전달하고 응용영역이 리턴한 결과를 표현영역이 JSON으로 변환해 HTTP응답한다.

응용 서비스는 도메인 모델에 로직 수행을 위임한다.

도메인 영역은 도메인 모델을 구현한다. 도메인 모델을 도메인의 핵심 로직을 구현.

인프라스트럭처 영역은 구현 기술에 대한 것을 다룬다.

RDBMS 연동이나 메시징 큐, 몽고DB등 연동한다.

응용영역에서 DB에 보관된 데이터가 필요하면 인프라스트럭쳐영역의 DB모듈을 사용해서 데이터를 읽어온다.

### 레이어
네 영역을 구성할 때 많이 사용하는 아키텍처가 표현->응용->도메인->인프라

도메인복잡도가 크면 응용->도메인으로 분리하고 작으면 도메인 하나를 두기도 한다.

상위계층만 하위계층에 의존한다. 하위계층은 상위계층을 모든다.

하지만 구현의 편리함을 위해 유연하게 응용계층이 바로 인프라계층에 의존 하기도 한다.

그런데 응용계층에서 인프라계층을 의존해서 쓰면 `테스트어려움`과 `기능확장의어려움`이라는 문제가 발생한다.

해답은 DIP이다.

### DIP
의존성 역전의 원칙

고수준모듈(응용) <- 저수준모듈(인프라) 로 의존성을 바꿔버리는 것이다.

인터페이스로 저수준모듈이 고수준모듈에 의존하도록하며 고수준모듈이 저수준모듈을 사용하도록 한다.

응용계층에서 인터페이스를 만들고 인프라스트럭처계층에서 그 인터페이스를 의존하여 구현된 클래스를 만들면 된다.

그리고 DI를 통해 구현 모델을 선택 하면 된다.

### DIP와 아키텍처
인프라스트럭쳐의 구현체들은 응용영역과 도메인영영에 영향을 최소화 하면서 변경 할 수 있게 된다.

### 도메인영역의 주요 구성요소

### - 엔티티
고유의 식별자를 갖는 객체로 자신의 라이프사이클을 갖음.

도메인 모델의 데이터를 포함하며 해당 데이터와 관련된 기능을 함께 제공

### -밸류
식별자가 없고 하나의 도메인 객체의 속성을 표현할 때 사용.

불변객체

도메인 엔티티와 DB테이블의 엔티티와는 다르다.

도메인 엔티티는 데이터와 기능을 제공한다.

RDBMS는 밸류를 표현하기 어렵다.

밸류는 불변이라 수정할때 완전히 새로운 객체로 변경 되어야한다.

### -애그리거트
관련된 엔티티와 밸류객체를 개념적으로 하나로 묶는것. (한 라이프스타일 트랜잭션)

도메인이 커질수록 개발할 도메인 모델도 커지면서 많은 엔티티와 밸류가 출현한다.

이 때 전체 구조를 파악하기 어려워 지는데 애그리거트가 있으면 전체 구조를 파악하면서 엔티티와 밸류에 집중 할 수 있다.

군집에 속한 객체들을 관리하는 루트 엔티티를 갖는다.

`루트엔티티` 는 애그리거트에 속해 있는 엔티티와 밸류 객체를 이용해서 애그리거트가 구현해야할 기능을 제공

이는 애그리거트이 내부 구현을 숨겨서 애그리거트 단위로 구현을 캡슐화 할 수 있게 돕는다.

하위 엔티티에 대해 알고 싶더라도 항상 `루트엔티티` 를 접근해서 확인한다.

### -리포지터리
도메인 모델의 영속성 처리

예를 들어 DBMS 테이블에 엔티티 객체를 가져오거나 저장한다.

리포지터리는 애그리거트 단위로 도메인 객체를 저장하고 조회하는 기능을 정의

리포지터리는 인터페이스로 만들고 응용계층에서 DI받아서 인프라스트럭처로 DB접근을 하게 된다.

스프링에서 다음과 같이 적용할 것이다.

```java
@Configuration
public class OrderSerServiceConfig {  
    @Autowired  
    private OrderRepository orderRepository;  

    @Bean 
    public CancelOrderService cancelOrderService() {    
        return new CancelOrderService(orderRepository); 
    }
}

@Configuration
public class RepositoryConfig {  
@Bean  
public JpaOrderRepository orderRepository() {  
  return new JpaOrderRepository();
  } 
 }
```

* 응용 서비스는 필요한 도메인 객체를 구하거나 저장할 때 리포지터리를 사용한다.
* 응용서비스는 트랜잭션을 관리 하는데 , 트랜잭션 처리는 리포지터리 구현 기술에 영향을 받는다.

리포지터리의 사용 주체가 응용 서비스이기 때문에 리포지터리는 응용 서비스가 필요로 하는 메서드를 제공한다.

* 애그리거트를 저장하는 메서드
* 애그리거트 루트 식별자로 애그리거트를 조회하는 메서드

```java
public interface SomeRepository {  
void save(Some some);  
Some findById(SomeId id);
}
```

### -도메인서비스
특정 엔티티에 속하지 않은 도메인 로직을 제공

예를들어 할인금액계산은 상품,쿠폰,회원등급,구매액 등 다양한 조건을 이용해 구현하는데 이렇게 다양한 엔티티와 밸류를 필요로 할 경우 도메인 서비스에서 로직을 구현

### 요청처리 흐름

```
Browser->Controller: HTTP요청
Controller->Controller: 요청데이터를 응용서비스에 맞게 변환
Controller->Application Service: 기능 실행 
Application Service->Repository: find
Repository-->Application Service: 도메인 객체 반환
Application Service->Domain Object: 도메인 로직 실행
Application Service-->Controller: 리턴
Controller-->Browser:HTTP응답

```

### 인프라스트럭처 개요
인프라스트럭처는 표현,응용,도메인 레이어에 지원한다.

도메인객체의 영속성 처리, 트랜잭션, SMTP클라이언트,Rest클라이언트등 다른 영역에서 필요하는 프레임워크, 기술, 보조 기능을 지원.

인프라스트럭처에 대해 의존성을 없앤다고 했는데 프레임워크에 따라서 인프라스트럭처가 응용과 도메인 레이어에 의존 되서 사용하는 경우도 있다.

스프링을 사용하면 응용 서비스는 트랜잭션 처리를 위해 스프링이 제공하는 @Transactional 사용하는것이 편리하다.

JPA는 @Entity와 @Table은 도메인 모델 클래스에 사용하는 것이 XML 매핑 설정을 이용하는것보다 편리하다.

구현의 편리함은 DIP가 주는 다른 장점만큼 중요하기 때문에 DIP 장점을 해치지 않는 범위에서 응용 영역과 도메인 영역에서 구현 기술에 대한 의존을 가져가는것이 현명.

표현영역은 항상 인프라스트럭처 영역과 쌍을 이룬다. 스프링MVC를 사용해 웹요청을 처리하면 스프링이 제공하는 MVC에 맞게 표현영역을 구현해야한다.

### 패키지구성
레이어별로 패키지를 구성할 수 있는데

도메인이 크면 도메인 별로 각 영역을 가질 수 있다.

도메인 모듈은 도메인에 속한 애그리거트를 기준으로 구성

또한 한 도메인이 특별히 크다면 거기에서 패키지를 세분화 해도된다.

ex)

com.myshop.catalog.ui

com.myshop.catalog.application

com.myshop.catalog.domain.product

com.myshop.catalog.domain.category

com.myshop.catalog.infrastructure

com.myshop.order.ui

com.myshop.order.application

com.myshop.order.domain

com.myshop.order.infrastructure

com.myshop.member.ui

com.myshop.member.application

com.myshop.member.domain

com.myshop.member.infrastructure

보통 패키지에 10개 미만 클래스 유지 권장

## 애그리거트 깊이 탐구

## ID를 이용한 애그리거트 참조
한 객체가 다른 객체를 참조하는것 처럼 애그리거트도 다른 애그리거트를 참조한다. 애그리거트의 관리 주체가 애그리거트 루트이므로 애그리거트에서 다른 애그리거트를 참조한다는것은 루트를 참조한다는것.

애그리거트간 참조는 필드를 통해서 쉽게 구현할 수 있다.

```
public class Member {}public class Orderer {    private Member member;}
```

필드를 통해서 다른 애그리거트를 **직접** 참조하는것은 구현의 편리함이 있다. JPA를 사용하면 `@OneToMany`, `@OneToOne`같은 어노테이션으로 연관된 객체를 로딩하는 기능을 제공하므로 필드를 이용해서 다른 애그리거트를 쉽게 참조 할 수 있다.

하지만 필드를 이용한 애그리거트 참조는 아래 문제점이 있다

* 편한 탐색 오용
* 성능에 대한 고민
* 확장 어려움

한 애그리거트 내부에서 다른 애그리거트의 객체를 쉽게 접근할 수 있으므로 다른 애그리거트의 상태를 쉽게 변경할 수 있게 된다. 한 애그리거트가 관리하는 범위는 자기 자신으로 한정해야 한다. 다른 애그리거트의 상태를 변경하는것은 애그리거트간의 의존 결합도를 높여서 결과적으로 애그리거트의 변경을 어렵게 만든다.

성능에 대한 고민으로는 편집에 유용한 lazy loading과 조회에 유용한 eager loading 사이에서 하나만 선택 가능하다는것이다.

단일 DBMS로 서비스를 제공하다가 하위 도메인별로 시스템을 분리하면서 도메인마다 서로다른 DBMS를 사용하면 더이상 다른 애그리거트 루트를 참조하기 위해 JPA같은 단일 기술을 사용할 수 없다.

그래서 ID를 이용해서 다른 애그리거트를 참조하면 좋다.

```
class Member {     private MemberId id;}class Orderer {    private MemberId memberId;}
```

DB테이블에서의 외래키를 사용해서 참조하는 것과 비슷하게 다른 애그리거트를 참조할때 ID참조를 사용한다는 점. 단 애그리거트 내의 엔티티를 참조할때는 객체 레퍼런스로 참조한다.

이렇게 하면 모든객체가 참조로 연결되지 않고 한 애그리거트에 속한 객체들만 참조로 연결된다. 이는 애그리거트의 경계를 명확히 하고 애그리거트간 물리적인 연결을 제거하기 때문에 복잡도를 낮춰준다. 또 애그리거트간 의존을 제거하므로 응집도를 높여준다. 또 구현 복잡도도 낮아진다. 직접 참조하지 않으므로 참조하는 애그리거트가 필요하면 응용서비스에서 아이디를 이용해서 로딩하면 된다. 직접 참조하지 않기 때문에 애그리거트 내부에서 다른 애그리거트의 상태를 변경할 수 없다. 애그리거트별로 다른 기술을 사용하는것도 가능하다.

이렇게 아이디를 이용한 참조를 하게되면 조인쿼리로 한번에 가져올 수 있는것을 N+1쿼리를 날려야 되서 성능이 후지게 된다. 이때 조회 전용 쿼리를 사용하면 된다. 데이터 조회를 위한 별도 DAO를 만들고 DAO의 조회 매서드에서 세타 조인(?)을 이용해서 한 번의 쿼리로 필요한 데이터를 로딩한다.

## 애그리거트 간 집합 연관
…

## 애그리거트를 팩토리로 사용하기
상점 계정이 차단 상태가 아닌 경우에만 상품을 생성하도록 구현할때

```java
public class RegisterProductService {
    public ProductId registerNewProduct(NewProductRequest req) {
        Store account = accountRepository.findStoreById(req.getStoreId());
        checkNull(account);
        if (!account.isBlocked()) {
            throw new StoreBlockException();
        }
        productId id = productRepository.nextId();
        Product product = new Product(id, account.getId, ....);
        productRepository.save(product);
        return id;
    }
}
```

Product를 생성 가능한지 판단하는 코드와 Product를 생성하는 코드가 분리되어 있다. 중요한 도메인 로직 처리가 응용 서비스에 노출되어 있다. Store 가 Product를 생성할 수 있는지 여부를 판단하고 Product를 생성하는 것은 논리적으로 하나의 도메인 기능인데 이 도메인 기능을 응용 서비스에서 구현하고 있는것이다. 이 도메인 기능을 넣기 위한 별도의 도메인 서비스나 팩토리 클래스를 만들수도 있지만 이기능을 구현하기 더 좋은 장소는 Store 애그리거트이다.

```java
public Store extends Member {    
			public Product createProduct(ProductId productId) {   
			     if (isBlocked()) throw new StoreBlockedException();   
			     return new Product(newProduct, getId, .....);   
			 }
}
```

Store 애그리거트의 createProduct()는 Product 애그리거트를 생성하는 팩토리 역할을 한다. 팩토리 역할을 하면서도 중요한 도메인 로직을 구현하고 있다. 이제 응용 서비스는 팩토리 기능을 이용해서 Product를 생성하면 된다.

```java
public class RegisterProductService {    
		public ProductId registerNewProduct(NewProductRequest req) {   
		     Store account = accountRepository.findStoreById(req.getStoreId());   
		     checkNull(account);     
			   ProductId id = productRepository.nextId();  
		     Product product = account.createProduct(id, ...);   
			   productRepository.save(product);        return id;   
		 }
}
```

앞선 코드와 차이점은 응용 서비스에서 더이상 Store의 상태를 확인하지 않는다는것이다. Store가 Product를 생성할 수 있는지 여부를 확인하는 도메인 로직은 Store에서 구현하고 있다. 이제 Product 생성 가능 여부를 확인하는 도메인 로직을 변경해도 도메인 영역의 Store 만 변경하면 되고 응용 서비스는 영향을 받지 않는다. 도메인의 응집도도 높아졌다.

애그리거트가 갖고 있는 데이터를 이용해서 다른 애그리거트를 생성해야한다면 애그리거트에 팩토리 매서드를 구현하는것을 고려해보자.

## Repository와 모델(JPA중심)

### Repository 구현
repository 구현 클래스를 domain.impl과 같은 패키지에 위치 시키는것은 구현체를 분리함을 위한거지 좋은 설계 원칙이 아니다. 가능하면 repository구현 클래스를 인프라스트럭쳐에 위치시켜서 인프라스트럭처에 대한 의존을 낮춰야한다.

Repository의 기본기능은

* 아이디로 애그리거트 조회
* 애그리거트 저장

```
public interface SomeRepository {    void save(Some some);    Some findById(SomeId id);}
```

루트엔티티를 기준으로 repository 인터페이스를 작성한다.

```
@Repositorypublic class JpaOrderRepository implements OrderRepository {  @PersistenceContext  private EntityManager entityManager;    @Override  public Order findById(OrderNo id) {    return entityManager.find(Order.class, id);  }    @Override  public void save(Order order) {    entityManager.persist(order);  }    @Override  public void remove(Order order) {    entityManager.remove(order);  }}
```

```
public class ChangeOrderService {  @Transactional  public void changeShippingInfo(OrderNo no, ShippingInfo newShippingInfo) {    Order order = orderRepository.findById(no);    if(ordr == null) {      throw new OrderNotFoundException();         }    order.changeShippingInfo(newShippingInfo);  }}
```

서비스에서 repository의 애그리거트를 가져와서 변경하면 트랜젝션 안에서 update쿼리까지 실행한다.

id외에 다른 조건으로 애그리거트를 조회할 때에는 JPA의 Criteria나 JPQL을 사용한다. (QueryDSL이 더 좋다)

### 매핑구현

* 애그리거트 루트는 엔티티이므로 @Entity
* 밸류는 @Embeddable
* 타입 파라미터가 밸류면 @Embedded
  * @AttributeOverrides()를 써서 매핑설정과 다른 칼럼 이름을 사용할 수 있다
    * name은 객체 프로퍼티명, column은 컬럼명

**기본생성자**

* JPA를 위해 기본 생성자 필요. JPA가 상속해서 쓰기 때문에 protected로 다른 코드에서 기본 생성자로 생성 못하게 한다.

### 필드접근방식

* @Access(AccessType.FIELD)로 명시적으로 접근 방식을 바꿀 수 있다. 하지만 필드에 놓으면 필드접근방식이 되고 메서드에 위치하면 메서드 접근 방식을 선택한다.

### 밸류 타입의 프로퍼티를 한 개의 컬럼에 매핑

* ## 프로퍼티와 DB컬럼 사이에 타입을 변환하고 싶을 때 AttributeConverter 인터페이스를 상속한다. <밸류타입,DB타입> 그리고 @Converter(autoApply=true) 클래스에 붙인다.

```java
@Converter(autoApply = true)
public class MoneyConverter implements AttributeConverter<Money, Integer> {
    @Override
    public Integer convertToDatabaseColumn(Money money) {
        if (money == null) return null;
        else return money.getValue();
    }

    @Override
    public Money convertToEntityAttribute(Integer value) {
        if... else return new Money(value)
    }
}
```
* **@Converter(autoApply=true)** 는 모든 Money 프로퍼티에 대해 converter적용 , false면 프로퍼티에 @Converter(converter=MoneyConverter.class)

### 밸류컬렉션: 별도 테이블 매핑

* 밸류 컬렉션을 별도 테이블로 매핑할 때는 @ElementCollection을 달고 @CollectionTable로 밸류를 저장할 테이블을 지정한다. name속성으로 테이블 이름을 지정하고 joinColumns로 외부키 컬럼을 지정. @OrderColumn으로 지정한 컬럼에 리스트의 인덱스값을 저장

### 밸류컬렉션: 한개 컬럼에 매핑

* 밸류컬렉션을 한개 컬럼에 매핑 해야할때 (이메일 주소 목록을 set으로 보관하고 DB에는 한 개 컬럼에 콤마로 구분해서 저장) AttributeConverter 사용하면 한 개 컬럼에 쉽게 매핑 가능. 단, 밸류컬렉션을 표현하는 새로운 밸류 타입을 추가 해야함.

### 밸류를 이용한 아이디 매핑

* 식별자를 기본 타입이 아닌 별도 밸류타입으로 만들때는 @Id대신 @EmbeddedId 사용. 단, Serializable 타입이어야함. 이때 장점은 식별자에 기능을 추가할 수 있음. 이런 식별자를 가지는 엔티티는 equals와 hashcode에 신경 써야한다.

### 별도 테이블에 저장하는 밸류 매핑

* 

애그리거트에서 루트 엔티티를 뺸 나머지는 대부분 밸류. 다른 엔티티가 있으면 진짜 엔티티인지 의심. 단지 별도 테이블에 데이터를 저장한다고 해서 엔티티인 것은아님. 밸류를 별도 테이블에 저장할수 있었음.

* 

밸류가 아니라 엔티티가 확실하다면 다른 애그리거트는 아닌지 확인해야함. 특히

독자적인 라이프 사이클을 갖는다면 다른 애그리거트일 가능성 높음

* 상품과 리뷰가 같은 애그리거트로 생각할 수 있지만 아니다. 상품과 리뷰는 따로 생성되고 따로 변경된다.
* 

매핑되는 테이블의 식별자와 애그리거트의 식별자가 다를 수 있다.

* Article과 ArticleContent를 보면 1대1매핑으로 생각하고 엔티티를 두개 만들수있는데 틀렸다.
* ArticleContent를 밸류로 봐야한다. AriticleContent의 ID는 식별하긴 하지만 Article테이블의 데이터와 연결을 위함이다. 즉, 특정 프로퍼티를 별도 테이블에 보관한 것으로 접근해야한다.
* ArticleContent는 밸류이므로 @Embeddable로 매핑하고 Article에는 밸류를 매핑한 테이블을 지정한다.
  * `@SecondaryTable(name = "article_content", pkJoinColumns = @PrimaryKeyJoinColumn(name = "id"))`
  * `@AttributeOverrides({ @AttributeOverride(name ="content", column = @Column(table="article_content")),@AttributeOverride(name ="contentType", column = @Column(table="article_content")) }) private ArticleContent content`
  * @SecondaryTable사용하면 매핑한 테이블 조인해서 가져온다. 만약 밸류의 테이블까지 값을 읽어오기 싫으면 지연로딩을 하면되는데 이는 밸류를 엔티티로 만드는것이므로 옳지 못하다. 대신 조회 전용 쿼리를 사용한다.

### 밸류 컬렉션을 @Entity로 매핑하기

* 개념적으로 밸류이지만 상속을 표현하기 위해 @Entity사용할 때가 있다.
* entity인 컬렉션을 clear하면 한번씩 delete쿼리가 발생해서 성능에 위협을 준다.
* @Embeddable타입이면 한번에 delete한다. 다만 상속은 포기하고 if-else로 타입 구분 필요하다.

### ID참조와 조인 테이블을 이용한 단방향 M:N

* todo..

### 애그리거트 로딩 전략

* 

즉시로딩(FetchType.EAGER)은 애그리거트 루트를 로딩하면 루트에 속한 모든 객체가 완전한 상태이다.

* 

애그리거트가 크면 즉시로딩 성능이 나빠진다.

* 

애그리거트는 개념적으로 완전해야하는 이유 중 하나는

표현영역에서 애그리거트이 상태를 보여줄 때 필요하기 때문인데 이건 별도의 조회 전용 기능을 구현하는 방식으로 하는게 유리.

* 

상태 변경 기능을 위해 완전해야 할 수 도 있는데 하지만

트랜잭션 범위내에서 지연로딩을 허용하기 때문에 실제로 상태를 변경하는 시점에 필요한 구성요소만 로딩해도 문제가 되지 않음

* 

보통 상태 변경보다 조회가 많으므로 상태 변경을 위해 지연 로딩 사용한다고 발생하는 추가 쿼리는 속도 저하에 문제가 되지 않는다.

* 

그래서 애그리거트 모든 연관을 즉시 로딩할 필요는 없다.


### 애그리거트 저장/삭제시 영속성 전파

* 

애그리거트 완전한 상태로 처리 한다면 와 일때 모든 객체를 처리하는 메서드를 가져야한다.

저장

삭제

* 

@Embeddable 타입은 자동으로 저장/삭제 되지만

* 

@Entity는 casecade={CascadeType.PERSIST, CascadeType.REMOVE} 해줘야 함게 저장/삭제 됨


### 식별자 생성 기능

* 사용자가 직접 생성
  * 이메일 주소처럼 사용자가 직접 식별자를 입력
* 도메인 로직으로 생성
  * 

식별자 생성규칙이 있는경우 엔티티를 생성할 때 이미 생성한 식별자를 전달하므로 엔티티가 식별자 생성기능을 제공하는 것보다는 별도 서비스로 식별자 생성 기능을 분리 해야함.

* 

식별자 생성 규칙은 도메인 규칙이므로 도메인 영역에 식별자 생성 기능을 위치시켜야한다.
```
public class ProductIdService {  public ProductId nextId() {    //정해진 규칙으로 식별자 생성   }}
```
* 

이 도메인 서비스를 이용해서 응용서비스에서 식별자를 구하고 엔티티를 생성한다.

* DB를 이용한 일련번호 사용

## 리포지터리의 조회 기능(JPA)
여러 애그리건트를 조합해서 한 화면에 보여주는 데이터 제공하거나 각종 통계 데이터 제공할때는 리포지터리를 사용하면 안된다.

JPA의 지연 로딩과 즉시 로딩, 연관 매핑으로 골치가 아픔.

통계는 특히 여러 테이블을 조인하거나 DBMS 전용 기능을 사용해야 구할 수 있는데 이는 JPQL이나 Criteria로 처리하기 힘듬.

이런 기능은 조회 전용쿼리로 처리해야함.

JPA와 하이버네이트를 사용하면 동적 인스턴스 생성, 하이버네이트의 @Subselect 확장 기능, 네이티브 쿼리, Mybatis를 이용해 조회 전용 기능을 구현할 수 있다.

queryDSL은 연관매핑이 되어있어서 join fetch 할수 있다. Mybatis는 쿼리 기반이라 매핑 지옥에서 벗어 날 수 있다.

## 응용서비스
응용서비스 기능

1. 라포지터리에서 애그리거트 조회
2. 도매인 기능 실행
3. 결과 반환

응용서비스 저장

1. 유효성 검사
2. 애그리거트 생성
3. 리포지터리에 애그리거트 저장
4. 결과 리턴

응용서비스에 비지니스로직 넣지 말기

응용서비스에 2-3개 기능만 담당해서 응집도 높이고 결합도 낮춤 SRP

인터페이스로 DIP 안해도 됨

TDD시 mokito로 목객체 활용

프레젠테이션 요청값은 클래스로 받고 응용서비스에 전달

응용서비스에서 애그리거트 반환하면 응집도 낮추기 때문에 필요한 데이터만 보내라

표현 영역의 일을 응용서비스에 넘기지 말아라

@Transaction 써라

도메인 로직이 발생할때 이벤트를 발생 시킬수 있다

도메인영역에서는 Events.raise

응용서비스에서는 Events.handle

응용서비스에 이벤트 날리는 로직 넣어도 되지만 도메인간의 의존성이나 외부 시스템에 대한 의존성을 낮춰주는 장점을 얻을 수 있고 시스템 확장에 이벤트가 핵심 역할을 수행할 수 있다 .

## 도메인서비스
도메인서비스는 특정 애그리거트에 넣기 애매한 로직이다

응용서비스에서 도메인서비스를 가지고 애그리거트의 메서드에 도메인서비스를 넘겨줘서 애그리거트가 도메인서비스 쓰게끔 한다. 혹은 반대로 도메인서비스에 애그리거트를 넣어 처리 하게 한다

## 애그리거트 트랜잭션 관리
한 애그리거트를 서로 다른 스레드에서 구해서 변경 하게 되면 서로 영향은 주지 않기 때문에 애그리거트의 규칙을 피할 수 있게 된다.

ex) 판매자 스레드가 배송을 해버리고 고객 스레드가 배송지를 바꿀때

### 비관적 잠금
비관적 잠금을 하면 선점하게 되서 블로킹 된다.

특정 레코드에 한 사용자만 접근 할 수 있게 하는 for update 같은걸 쓰는건데 JPA의 EntityManager에서는 `entityManager.find(Order.class, orderNo, LockModeType.PESSIMITIC_WRITE)` 로 쓸 수 있다.

이러한 비관적 잠금은 교착 상태로 빠지게 될 수 있다. 이럴때는 최대 대기 시간을 지정한다. JPA에서는

```java
Map<String,Object> hints = new HashMap<>(); 
hints.put("javax.persistence.lock.timeout", 2000); 
entityManager.find(Order.class, orderNo, LockModeType.PESSIMITIC_WRITE,hints)
```

### 낙관적 잠금
비관적잠금이 강력해 보이지만 모든 트랜잭션 충돌을 해결 할 수 없다.

위에는 판매자가 배송상태로 바꾸는 중에 고객이 배송지를 바꾸는건데 만약 판매자가 먼저 배송지 정보만 보고 그 후에 고객이 배송지를 바꾸고 판매자는 배송지를 미리 파악했으니깐 배송하고 상태를 변경 했다면 비관적으로 막더라도 판매자는 배송지 정보를 보고 나서 고객이 배송지를 바꾼거라 배송은 엉뚱한곳으로 가게 된다.

낙관적 잠금은 실제 DB에 반영할때 변경 가능한지 확인하는 방식이다.

이때 버전이 필요하다. 수정할 애그리거트와 매핑되는 테이블의 버전값이 현재 애그리거트와 같을때만 수정할 수 있게 한다.

JPA에서 @Version으로 지원한다.

만약 버전이 맞지 않아 트랜잭션이 충돌 되면 OptimisticLockingFailureException이 발생한다.

### 강제 버전 증가 필요
루트 엔티티 외에 다른 엔티티만 변경 된다면 루트엔티티에 달렸던 version을 바뀌지 않는다. 따라서 루트 엔티티의 버전을 강제로 증가 시켜야 한다. `entityManager.find(Order.class, id, LockModeType.OPTIMISTIC_FORCE_INCREMENT)` 를 쓰면 루트 엔티티 버전이 강제로 증가한다.

### 오프라인 선점 잠금
위 처럼 단일 트랜잭션에 멀티 스레드가 변경 할때는 해결할 수 있지만 멀티 트랜잭션에서는 막을 수 없다.

오프라인 선점 잠금은 첫번째 트랜잭션을 시작할 때 선점하고 마지막 트랜잭션에서 잠금을 해제한다. 다만 마지막 트랜잭션 끝나기 전에 종료할수도 있으니 유효 시간이 지나면 자동으로 해제 되게 해야하고 화면에서 나가기 전까지는 유효시간을 자동 갱신 할 수 있게 구현해야한다.

## CQRS
주문 상세 조회 화면 처럼 주문,상품,멤버를 조인해서 가져와야된다면

ID를 이용해 연관 관계를 참조하면 여러 도메인 조합할때 성능이 느려서 연관관계를 맺어 즉시 로딩해야 성능에 좋은데 즉시 로딩하면 단일 도메인 조회할때 불필요한 성능 저하가 있을것이다.

구현의 복잡도가 올라가기 때문에 이럴때는 상태변경을 위한 모델과 조회를 위한 모델을 분리 하는것이다.

상태를 변경하는 범위와 상태를 조회하는 범위가 정확하게 일치하지 않기 때문에 단일 모델로 두 종류의 기능을 구현하면 모델이 불필요하게 복잡해진다. 단일 모델을 상용할 때 발생하는 복잡도를 해결하기 위해 사용하는 방법이 CQRS

Command Query Responsibility Segregation

상태를 변경하는 Command를 위한 모델과 상태를 제공하는 Query를 위한 모델을 분리하는것이다.

CQRS는 복잡한 도메인에 적합. 복잡한 도메인일수록 명령과 조회 기능의 데이터 범위가 차이난다.

Command(상태변경)은 JPA

Query(복잡한 조회)는 MyBatis

조회 모델에는 응용서비스가 존재하지 않는다. 컨트롤러에서 바로 DAO를 실행 (물론, 몇가지 로직이 필요하다면 응용서비스를 두면 됨)

queryDSL은 연관매핑이 되어있어서 join fetch 할수 있다. Mybatis는 쿼리 기반이라 매핑 지옥에서 벗어 날 수 있다.

나의 생각 : **애그리거트 안에서는 queryDSL을 쓰고 애그리거트 간의 조회는 Mybatis를 쓰자**

명령 모델(RDB)과 조회 모델(NoSQL)이 서로 다른 데이터 저장소를 사용 할 수 있다.

이렇게 데이터 저장소가 달라지면 데이터 동기화는 이벤트를 통해 처리.

#dev/book
#ing