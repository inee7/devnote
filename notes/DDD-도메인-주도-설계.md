---
tags: [ddd, domain-driven-design, dip, clean-architecture, hexagonal]
---

## 한 줄 요약

DDD(Domain-Driven Design)는 데이터 중심이 아닌 도메인(비즈니스 로직) 중심으로 개발하는 방법론으로, DIP(의존성 역전)를 통해 도메인을 프레임워크로부터 독립시켜 비즈니스 변화에 유연하게 대응한다.

## 핵심 정리

- DDD는 도메인(해결해야 할 문제 영역) 중심 개발 방법론
- MVC의 서비스 계층 중복과 복잡성 문제를 도메인 객체 강화로 해결
- DDD는 4Tier 아키텍처: 프레젠테이션 - 애플리케이션 서비스 - 도메인 - 인프라스트럭처
- DIP(의존성 역전)로 도메인을 프레임워크(JPA, Spring)로부터 독립
- 멀티모듈 구조: API → Domain ← JPA (Domain이 중심)
- 트레이드오프: 레이어 분리로 코드량 증가, 매핑 코드 필요
- 장점: 비즈니스 변화에 유연, SOLID 원칙 준수, 프레임워크 교체 용이

## 상세 내용

### DDD란 무엇인가

#### 정의

**도메인(Domain)**: 해결해야 할 문제 영역

DDD는 데이터 기반 개발에서 벗어나 **우리가 풀어야 할 도메인 중심으로 개발**하고자 하는 방법론입니다.

역사:
- Eric Evans가 최초 주장
- Spring 2.0에서 DDD 방식 지원 시작

#### 아키텍처 진화

**1Tier, 2Tier Architect**
- 프리젠테이션과 비즈니스 모델이 서로 뒤엉킴
- 하나의 기능을 수행하기 위해 로우레벨 기술(Data Access, Transaction)이 프리젠테이션 영역까지 침범

**MVC**
- 복잡한 로우레벨 기술을 처리하는 비즈니스 모델을 뷰(프리젠테이션)와 분리
- 개발자와 디자이너 사이의 업무 영역을 효율적으로 분할
- 프리젠테이션 계층에서 DB 정보 누출 방지, 코드 중복 제거

**MVC의 한계**

> 너무 과도한 역할 분담 때문에 과거 1Tier, 2Tier 계층이 갖고 있었던 직관적인 설계의 장점은 송두리째 지워버렸다는 게 문제

문제점:
- **서비스 계층의 중복과 복잡성 증가** (SRP, OCP 지키기 힘듦)
- 대량 처리만을 목적으로 하여 세밀한 상태 변화나 디테일한 설정에 매우 약함
- 여러 계층을 통한 과정을 이해해야 함

**DDD (Domain-Driven Design)**

DDD는 다른 어떤 계층보다도 **도메인 계층, 즉 객체의 역할을 충분히 강화**합니다.

장점:
- **비즈니스 변화에 유연**: 세밀한 변화에 매우 민감하게 반응
- **객체지향 프로그래밍의 근간**: SOLID 원칙 준수
- **직관적인 설계**: 4Tier 아키텍처 구현
- **도메인에만 집중**: 디자이너, 기획자의 요구사항은 도메인(객체)에만 집중하면 됨

### DDD 아키텍처

#### 4Tier 구조

```
프레젠테이션 (Presentation)
    ↓
애플리케이션 서비스 (Application Service)
    ↓
도메인 (Domain) ← 핵심!
    ↓
인프라스트럭처 (Infrastructure)
```

각 계층의 역할:
- **프레젠테이션**: HTTP 요청/응답, UI
- **애플리케이션 서비스**: 트랜잭션, 도메인 조합
- **도메인**: 비즈니스 로직, 엔티티, 값 객체
- **인프라스트럭처**: 데이터베이스, 외부 API, 메시징

### 도메인과 프레임워크 분리

#### 문제: 도메인의 프레임워크 의존

Spring 프레임워크가 POJO를 지키기 위해 등장했지만, 언젠가부터 **우리의 도메인엔 Spring이 침투**하고 있습니다.

일반적인 코드:

```kotlin
// JPA 엔티티가 도메인 객체의 역할을 한다
@Entity
@Table(name = "member")
class Member(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long? = null,

    var name: String,
    var email: String
) {
    // 도메인 로직
    fun changeEmail(newEmail: String) {
        this.email = newEmail
    }
}
```

문제점:
- 도메인 객체가 JPA에 의존
- 프레임워크 변경 시 도메인 수정 필요
- 순수한 비즈니스 로직 테스트 어려움

#### 해결: DIP(의존성 역전 원칙)

**일반적인 의존관계**

```
API → Domain → JPA
```

Domain이 JPA에 의존하면:
- JPA 엔티티가 도메인 객체 역할
- 프레임워크에 종속
- Domain과 JPA 모듈 분리가 무의미

**의존성 역전 적용**

```
API → Domain ← JPA
```

Domain이 중심이 되고, JPA가 Domain에 의존:
- 도메인은 인터페이스만 정의
- JPA는 인터페이스 구현체 제공
- 프레임워크로부터 독립

### 멀티모듈 구조 예시

#### 모듈 구성

```
project/
├─ api/          # 실제 서버 애플리케이션 (main, HTTP 진입점)
├─ domain/       # 도메인 로직 (순수 비즈니스)
└─ jpa/          # JPA 설정 및 구현
```

#### 일반적인 의존관계 (AS-IS)

```gradle
// api
dependencies {
    implementation project(":domain")
}

// domain
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'
    implementation project(":jpa")  // ❌ 도메인이 JPA에 의존
}

// jpa
dependencies {
    api 'org.springframework.boot:spring-boot-starter-data-jpa'
}
```

문제: Domain → JPA 의존으로 도메인이 프레임워크에 종속

#### DIP 적용 의존관계 (TO-BE)

```gradle
// api
dependencies {
    implementation project(":domain")
    implementation project(":jpa")  // API가 JPA 의존성 처리
}

// domain
dependencies {
    implementation 'org.springframework:spring-tx'  // 최소한의 의존
}

// jpa
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation project(":domain")  // ✅ JPA가 Domain에 의존
}
```

장점: Domain이 중심, JPA는 구현체일 뿐

#### 코드 구조

**Domain 모듈 (순수 도메인)**

```kotlin
// 도메인 객체 (프레임워크 의존 없음)
data class Member(
    val id: MemberId,
    val name: String,
    val email: String
) {
    // 순수 비즈니스 로직
    fun changeEmail(newEmail: String): Member {
        require(newEmail.contains("@")) { "유효하지 않은 이메일" }
        return copy(email = newEmail)
    }
}

// Value Object
@JvmInline
value class MemberId(val value: Long)

// 리포지토리 인터페이스 (도메인이 정의)
interface MemberRepository {
    fun save(member: Member): Member
    fun findById(id: MemberId): Member?
}
```

**JPA 모듈 (구현체)**

```kotlin
// JPA 엔티티 (도메인과 분리)
@Entity
@Table(name = "member")
class MemberEntity(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long? = null,

    val name: String,
    val email: String
) {
    companion object {
        fun from(member: Member): MemberEntity {
            return MemberEntity(
                id = member.id.value,
                name = member.name,
                email = member.email
            )
        }
    }

    fun toDomain(): Member {
        return Member(
            id = MemberId(id!!),
            name = name,
            email = email
        )
    }
}

// 리포지토리 구현 (도메인 인터페이스 구현)
@Repository
class MemberRepositoryImpl(
    private val jpaRepository: MemberJpaRepository
) : MemberRepository {

    override fun save(member: Member): Member {
        val entity = MemberEntity.from(member)
        val saved = jpaRepository.save(entity)
        return saved.toDomain()
    }

    override fun findById(id: MemberId): Member? {
        return jpaRepository.findById(id.value)
            .map { it.toDomain() }
            .orElse(null)
    }
}

interface MemberJpaRepository : JpaRepository<MemberEntity, Long>
```

### 트레이드오프

#### 코드량 증가

레이어를 분리하면 분리할수록 **레이어를 넘나들 때마다 매핑 코드가 발생**합니다.

```kotlin
// 도메인 객체
data class Member(val id: MemberId, val name: String, val email: String)

// JPA 엔티티
@Entity
class MemberEntity(val id: Long?, val name: String, val email: String) {
    // 도메인 ↔ 엔티티 매핑 코드 필요
    fun toDomain(): Member = ...
    companion object {
        fun from(member: Member): MemberEntity = ...
    }
}
```

거의 동일한 모양의 클래스들이 생겨나고, 매핑 로직도 필요합니다.

#### 현실적인 고민들

- 정말 도메인에서 프레임워크를 제거할 수 있나?
- 이렇게 짜면 코드가 너무 많아지는데?
- 현업에서 레퍼런스를 찾기 어려움
- 동료들과의 타협 필요

#### 그럼에도 분리하는 이유

**프레임워크로 인해 지저분해지는 코드를 분리할 수 있습니다.**

- "이렇게 하는 게 바람직한 건 아닌 것 같은데 프레임워크 때문에 이렇게 할 수밖에 없는" 경우가 종종 있음
- 도메인을 프레임워크로부터 격리시키면 프레임워크와 상관없이 더 바람직한 방향으로 코드 작성 가능
- 프레임워크로 인해 지저분해지는 코드는 다른 쪽으로 분리 가능

### 더 나아가서

#### Spring 의존성 완전 제거

예제에서 Domain 모듈은 `spring-tx`에 의존하고 있습니다.

```gradle
// domain
dependencies {
    implementation 'org.springframework:spring-tx'  // 트랜잭션 때문에
}
```

더 순수한 도메인을 위해서는:
- 컴포넌트 스캔에 대한 책임 분리
- 트랜잭션에 대한 책임 분리

물론 그렇게 하면 코드는 더 늘어납니다.

## 실무 적용

### DDD 도입 체크리스트

- [ ] 도메인 모듈을 프레임워크로부터 독립시켰는가?
- [ ] 도메인 객체가 순수한 비즈니스 로직만 포함하는가?
- [ ] 인프라 계층(JPA, Redis 등)이 도메인에 의존하는가?
- [ ] 매핑 코드가 인프라 계층에 위치하는가?
- [ ] 도메인 로직 테스트 시 프레임워크 없이 가능한가?

### 도입 전략

**1단계: 핵심 도메인부터**
- 전체를 한 번에 바꾸려 하지 말 것
- 가장 중요한 비즈니스 로직부터 도메인 분리
- 레거시는 점진적으로 리팩토링

**2단계: 멀티모듈 구성**
- api, domain, infrastructure 모듈 분리
- domain은 infrastructure에 의존하지 않음
- api가 의존성 연결

**3단계: 인터페이스 정의**
- 도메인에서 필요한 인터페이스 정의
- 인프라에서 구현체 제공
- DIP 적용

**4단계: 팀 합의**
- 코드량 증가에 대한 이해 필요
- 장기적 이점 공유 (유지보수성, 테스트 용이성, 프레임워크 독립성)

### 금융권에서의 고려사항

- **비즈니스 로직 중심**: 복잡한 금융 규칙은 도메인에 집중
- **감사 추적**: 도메인 이벤트로 모든 변경 추적
- **테스트 커버리지**: 도메인 로직은 프레임워크 없이 단위 테스트 필수
- **규제 대응**: 비즈니스 규칙 변경 시 도메인만 수정하면 됨
- **레거시 전환**: 점진적 마이그레이션 전략 (핵심 도메인부터)
- **성능**: 매핑 코드로 인한 오버헤드 측정 필요
- **JPA 특화 기능**: Dirty Checking 등 활용 불가, 명시적 save 필요

### 참고 아키텍처

- **클린 아키텍처** (Clean Architecture)
- **헥사고날 아키텍처** (Hexagonal Architecture / Ports and Adapters)
- **어니언 아키텍처** (Onion Architecture)

모두 본질은 같습니다: **도메인을 중심에 두고 프레임워크를 외부로**

---

**출처**
- Domain-Driven Design (Eric Evans)
- [domain 에서 프레임워크 의존을 제거할 수 있을까](https://multifrontgarden.tistory.com/296)
- [이터너티님의 Domain Driven Design](http://aeternum.egloos.com/category/Domain-Driven%20Design)
- [이일민(토비)님의 스프링 프레임워크와 DDD](http://www.dbguide.net/knowledge.db?cmd=view&boardUid=127395)

