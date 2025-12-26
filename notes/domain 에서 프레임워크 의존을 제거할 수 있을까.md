# domain 에서 프레임워크 의존을 제거할 수 있을까
![](domain%20%EC%97%90%EC%84%9C%20%ED%94%84%EB%A0%88%EC%9E%84%EC%9B%8C%ED%81%AC%20%EC%9D%98%EC%A1%B4%EC%9D%84%20%EC%A0%9C%EA%B1%B0%ED%95%A0%20%EC%88%98%20%EC%9E%88%EC%9D%84%EA%B9%8C/R800x0.jpg) 

모두다 등장한지 오래된 개념들이지만 몇년 전부터 DDD 의 바람이 부는듯 하더니 요즘엔 클린 아키텍처, 헥사고날 아키텍처 등에 대한 관심이 많은 것 같다. 관심이 많은 것과 잘 하는건 분명 다르지만 잘하기 위해서는 일단 관심을 가져야하기에 마냥 나쁘게만 볼 현상은 아니라고 생각한다. 나도 관심을 갖고있는 사람 중 하나인데, 이런저런 이름으로 소통되고 있지만 이들이 주장하는건 결국 도메인에서 기술을 분리하는 것이다. 애초에 spring 프레임워크가 등장했던 것 자체가 POJO 를 지키기 위함이었는데 언젠가부터 우리의 도메인엔 spring 이 침투하고 있다.

이런 내용들을 책이나 자료를 통해 공부하고, 구두로 의견을 나누는건 어렵지 않은데 막상 현업에서 하려고하면 다양한 고민거리들을 마주하게 된다.

- 정말 도메인에서 프레임워크(사실상 spring)를 제거할 수 있나?

- 이렇게짜면 코드가 너무 많아지는데?

등등...  그리고 현업에서 프레임워크로부터 독립적인 도메인 레이어를 갖고있는 애플리케이션을 보는것도 쉽지 않다보니 마땅한 레퍼런스도 없다.

아키텍처를 부르는 다양한 이름들이 있지만 본질은 도메인 레이어를 프레임워크로부터 독립시키는 것이고, 그것을 위해서는 DIP 를 잘 활용해야한다. DIP 라니, 요즘엔 DIP 모르는 사람 찾기도 어려울 지경이고 면접시에 SOLID 에 대한 질문은 신입들도 꿰차고 온다. 하지만 실제 코드에서 DIP 를 활용하는건 의외로 보기 어렵다.

이번 포스팅의 목적은 공부를 하다가 혼자만의 고민거리와 나름의 해석들을 적는 것이다. 애플리케이션이라고 부르기도 민망한 아주 작은 애플리케이션(그냥 API 하나..)을 요즘 많이 사용하는 그레이들 멀티 모듈 구조로 만들고, 일반적인 의존성 흐름을 갖는 형태와 DIP 를 적용해 순수한 도메인 레이어를 갖는 형태를 살펴보자. 도메인 모듈은 spring 프레임워크 전부를 걷어내진 않을 것이고, JPA 를 걷어내는걸 목표로 한다.

#### **# 애플리케이션**

먼저 애플리케이션 구조에 대해 설명한다. 이번 예제는 그레이들 멀티 모듈로 구성되며 총 3개의 모듈이 등장한다.

![](domain%20%EC%97%90%EC%84%9C%20%ED%94%84%EB%A0%88%EC%9E%84%EC%9B%8C%ED%81%AC%20%EC%9D%98%EC%A1%B4%EC%9D%84%20%EC%A0%9C%EA%B1%B0%ED%95%A0%20%EC%88%98%20%EC%9E%88%EC%9D%84%EA%B9%8C/img.png) 

DIP-API: 실제적인 서버 애플리케이션. main 메서드를 포함하고있으며, HTTP 진입 포인트를 제공.

DIP-DOMAIN: 도메인 로직을 담아두는 모듈.

DIP-JPA: JPA 관련 설정과 클래스들을 담아두는 모듈.

#### **# 일반적인 의존성의 흐름**

처음에는 위 다이어그램에서 3개의 모듈을 일렬로 배치했었는데, 그러고보니 의도치않게 의존성의 순서를 나타내는 듯하여 삼각배치로 배치를 변경했다. 일반적으로는 저런 형태의 3개의 모듈이 있다고하면 이런식으로 의존하게 된다.

![](domain%20%EC%97%90%EC%84%9C%20%ED%94%84%EB%A0%88%EC%9E%84%EC%9B%8C%ED%81%AC%20%EC%9D%98%EC%A1%B4%EC%9D%84%20%EC%A0%9C%EA%B1%B0%ED%95%A0%20%EC%88%98%20%EC%9E%88%EC%9D%84%EA%B9%8C/img_2.png) 

사실 도메인이 JPA 에 의존하게 되면, JPA 엔티티가 도메인 객체 역할을 하게되기 때문에 예제와 같이 DOMAIN 과 JPA 모듈을 분리한게 무의미해진다. 그래서 도메인과 JPA 의 경계가 사라지고 이런 형태로 운영되는 경우가 많다.

![](domain%20%EC%97%90%EC%84%9C%20%ED%94%84%EB%A0%88%EC%9E%84%EC%9B%8C%ED%81%AC%20%EC%9D%98%EC%A1%B4%EC%9D%84%20%EC%A0%9C%EA%B1%B0%ED%95%A0%20%EC%88%98%20%EC%9E%88%EC%9D%84%EA%B9%8C/img_3.png) 

일단 DIP-JPA 가 존재한다는 가정하에 그레이들에 나타나는 의존성은 아래와 같다.

```
// DIP-API
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation project(":dip-domain")
}

// DIP-DOMAIN
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'
    implementation project(":dip-jpa")
}

// DIP-JPA
dependencies {
    api 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'com.h2database:h2'
}
```

어떻게보면 아름답기까지한 모습이다. 순차적으로 각 모듈에 의존성을 갖게된다. DIP-JPA 에서 spring data jpa 에 대한 의존성을 api 로 참조하는 이유는, JPA 엔티티를 도메인 객체로 이용하기 때문에 필연적으로 애플리케이션 전역에서 JPA 의존이 필요해지기 때문이다.(converter, page 등) api 와 implementation 에 대한 내용은 여기선 하지 않는다.

```java
// JPA 엔티티가 도메인 객체의 역할을 하게 된다.
@Entity
@Table(name = "dip")
@AllArgsConstructor
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Getter
public class Dip {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "dip_id")
    private Long id;
    @Column(name = "message")
    private String message;
    
    // domain logics
}
```

너무나도 당연해보이는 이런 의존관계는 결과적으로 도메인 레이어가 분리되기는 커녕 프레임워크에 의존하게 되고, 프레임워크의 변경에 매우 큰 영향을 받는다. 우리의 애플리케이션 흐름의 종단에 도메인이 존재하는 것이 아니라 spring data jpa 가 위치하게 되는 것이다.

#### **# 모듈간 의존성 역전**

이런저런 아키텍처 관련 서적들은 하나같이 도메인 레이어의 독립을 얘기한다. 그리고 그 독립의 중심엔 인터페이스가 있다. 인터페이스는 자바를 공부하는 사람들이라면 언어 공부를 할때부터 공부하는 내용이고, 언어 공부를 넘어 프레임워크에 대한 공부를 하게되도 익히 그 중요성을 공부하게 된다. 하지만 정작 코드를 작성할때는 인터페이스를 잘 활용하지 않는다. 이는 추상화에 대한 훈련이 부족한 것과 프레임워크 생태계가 인터페이스 없이도 너무 훌륭하게 모든 것들을 지원해주기 때문이다. 실제로 일반적인 의존관계 아키텍처에서는 인터페이스가 하나도 없어도 아무런 지장없이 코드를 작성할 수 있다. 하지만 의존성 역전을 위해서는 인터페이스가 필수다. 의존성 역전을 하게되면 의존관계는 이렇게 변경된다.

![](domain%20%EC%97%90%EC%84%9C%20%ED%94%84%EB%A0%88%EC%9E%84%EC%9B%8C%ED%81%AC%20%EC%9D%98%EC%A1%B4%EC%9D%84%20%EC%A0%9C%EA%B1%B0%ED%95%A0%20%EC%88%98%20%EC%9E%88%EC%9D%84%EA%B9%8C/img_4.png) 

차이가 보이는가? DOMAIN 에서 JPA 로 향하던 화살표가 JPA 에서 DOMAIN 으로 향한다. 화살표의 방향이 완전히 역전되기 때문에 DIP 를 의존성 역전이라고 부르는 것이다. 위 의존관계에서는 더 이상 JPA 엔티티가 도메인 객체의 책임을 가질 수 없다. 그렇기에 DIP-JPA 가 필수가 된다.

```
// 프레임워크 기술이 전혀 보이지 않는 도메인 객체
@Getter
@AllArgsConstructor
public class Dip {
    private DipId id;
    private String message;
    
    // domain logics
}
```

위 클래스는 깨알같이 id 도 VO 를 만들어서 사용한다. JPA 엔티티가 도메인 객체 역할을 할때는 id VO 를 만들기가 어렵다.

그레이들에는 이런식으로 의존관계가 표현된다.

```
// DIP-DOMAIN
dependencies {
    implementation 'org.springframework:spring-tx'
}

// DIP-JPA
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation project(":dip-domain")
    runtimeOnly 'com.h2database:h2'
}
```

대략적으로 얘기하면 도메인 객체의 로직은 모두 DIP-DOMAIN 모듈에서 책임지게 되고, 영속성 관련 로직은 DIP-JPA 가 책임진다. 도메인에는 영속성 관련한 인터페이스들만 위치하게 된다. 그럼 DIP-JPA 모듈에서 어찌어찌 영속성 구현체들을 만든다고쳐도 위와 같은 의존관계에서 어디선가에서는 DIP-JPA 에 대한 의존을 가져야하지 않을까? 물론이다.

![](domain%20%EC%97%90%EC%84%9C%20%ED%94%84%EB%A0%88%EC%9E%84%EC%9B%8C%ED%81%AC%20%EC%9D%98%EC%A1%B4%EC%9D%84%20%EC%A0%9C%EA%B1%B0%ED%95%A0%20%EC%88%98%20%EC%9E%88%EC%9D%84%EA%B9%8C/img_5.png) 

```
// DIP-API
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation project(":dip-domain")
    implementation project(":dip-jpa")
}
```

그 역할은 DIP-API 모듈이 갖는다. 프레임워크에 대한 의존성 독립을 외쳤지만 결국 코드를 작성하고 애플리케이션이 구동되려면 프레임워크에 대한 의존성도 어디선가에서는 처리해야한다. 그 의존성에 대한 처리를 도메인에서 하는게 아니라 애플리케이션 영역(예제에서는 DIP-API)에서 하는 것이다.

#### **# 도메인 분리가 마냥 좋을까**

지금은 그저 다이어그램으로만 표현되다보니 체감하기 어렵지만 위와 같이 아키텍처를 구축하게 되면 현실적인 고민들이 생긴다.가장 큰 고민은 코드량의 증가다. 레이어를 분리하고 순수한 영역들 가져간다는 표현은 표현에서부터 뭔가 좋아보이지만 막상 레이어를 분리하면 분리할 수록 레이어를 넘나들때마다 매핑코드가 발생하게 된다. 거의 동일한 모양의 클래스들이 생겨나면 이게 정말 좋을까 라는 의문이 들 수 있다.

```
@Getter
@AllArgsConstructor
public class Dip {
    private DipId id;
    private String message;
    
    // domain logics
}

@Entity
@Table(name = "dip")
@AllArgsConstructor
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class DipEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "dip_id")
    private Long id;
    @Column(name = "message")
    private String message;

    public static DipEntity from(Dip dip) {
        if (dip.getId() == null) {
            return new DipEntity(null, dip.getMessage());
        }
        return new DipEntity(dip.getId().getValue(), dip.getMessage());
    }

    public Dip toDomain() {
        return new Dip(new DipId(id), message);
    }
}
```

좋은 점에 대해서만 얘기할때는 프레임워크 의존성이 없는 도메인 객체에 대해서만 얘기할뿐, JPA 엔티티에 대한 언급은 사라진다. 하지만 영속성 프레임워크로 JPA 를 포기하지 않는다면 결국 어딘가에는 JPA 엔티티가 필요하다. 도메인 코드에 대한 매핑코드와 함께 말이다.

#### **# 더 나아가서**

의존성 역전을 했다는 도메인 모듈에서도 spring-tx 에 대한 의존성을 갖고있다. spring 에 대한 의존을 놓지 못한 이유는 컴포넌트 스캔을 위해서고, tx 에 대한 의존을 놓지 못한 이유는 트랜잭션을 위해서다. 이번 예제는 JPA 에 대한 의존성을 놓는것에 집중하기 위해 JPA 에 집중했다. 더 순수한 도메인 모듈을 위해서는 컴포넌트 스캔과 트랜잭션에 대한 책임도 분리해야한다. 물론 그렇게하면 코드는 더 늘어난다.

#### **# 정리**

혹여나 싶어서 얘기하면 코드가 더 늘어나니 도메인 레이어 분리는 허상이다거나 하지말자는 얘기를 하고싶은게 아니다. 이런저런 자료들을 보고 좀 더 나은 애플리케이션을 작성해보고자 했을까 마주하는 현실적인 고민들에 대해서 적어놓은 것 뿐이다. 개인적으로는 매핑코드가 많아져도 분리하는게 더 낫다고 생각한다. 하지만 혼자 개발하는게 아닌이상 동료들과 타협이 필요할 것이고, 그때 마주하게 되는 가장 큰 허들이 코드량의 증가다. 이 부분에 대한 많은 논의가 필요할 것이다.

개발을 하다보면 "이렇게 하는게 바람직한건 아닌것 같은데 프레임워크때문에 이렇게 할 수 밖에 없는" 경우가 종종 있다. 그런 경우는 대부분 도메인이 프레임워크에 의존하고 있기 때문에 발생하는 문제다. 도메인을 프레임워크로부터 격리시켜 놓으면 프레임워크랑은 상관없이 더 바람직한 방향으로 코드를 작성할 수 있다. 프레임워크로 인해 지저분해지는 코드는 다른 쪽으로 분리할 수 있기 때문이다.

나도 지속적으로 공부하고 적용해보려고 노력하고 있으니 시간이 지나 좀 더 숙련도 높은 포스팅을 할 수 있는 날이 오길 기대한다.

순차적 의존관계: [https://github.com/LichKing-lee/dip-sample/tree/main](https://github.com/LichKing-lee/dip-sample/tree/main)

DIP 적용 의존관계: [https://github.com/LichKing-lee/dip-sample/tree/dip](https://github.com/LichKing-lee/dip-sample/tree/dip)

[domain 에서 프레임워크 의존을 제거할 수 있을까](https://multifrontgarden.tistory.com/296)

#ddd
