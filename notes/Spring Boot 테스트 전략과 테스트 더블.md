---
tags: [spring, test, junit, mockito, testing-strategy]
---

# Spring Boot 테스트 전략과 테스트 더블

## 한 줄 요약

Spring Boot에서 슬라이스 테스트, 단위 테스트, 통합 테스트를 구분하여 사용하고, 테스트 더블(Mock, Stub, Spy 등)로 외부 의존성을 격리한다

## 핵심 정리

- **컨트롤러 책임 분리**: Controller는 HTTP 요청/응답 변환만, 흐름 제어는 Service 계층에서
- **슬라이스 테스트**: `@WebMvcTest`, `@DataJpaTest`로 계층별 격리 테스트
- **단위 테스트**: JUnit + Mock으로 비즈니스 로직 검증, 도메인 모델은 순수 테스트 가능
- **테스트 더블**: Dummy, Fake, Stub, Spy, Mock 각각의 용도 구분
- **AAA 패턴**: Arrange-Act-Assert 구조로 테스트 작성
- **FIRST 원칙**: Fast, Isolated, Repeatable, Self-Validating, Timely

## 상세 내용

### 컨트롤러 책임 분리

Controller가 흐름 제어까지 책임을 가지면 안 되는 이유:
- 계층 간 책임이 흐려짐
- 단위 테스트가 어려워짐

**원칙:** Controller는 HTTP 요청/응답 변환만, 비즈니스 흐름 제어는 Service 계층에서 담당

### 슬라이스 테스트

#### 컨트롤러 테스트 (@WebMvcTest)

```kotlin
@WebMvcTest(OrderController::class)
class OrderControllerTest {

    @Autowired
    private lateinit var mockMvc: MockMvc

    @MockBean
    private lateinit var orderService: OrderService

    @Test
    fun `주문_취소_요청시_정상응답을_반환한다`() {
        // given
        val orderId = 1L
        doNothing().`when`(orderService).cancelOrder(orderId)

        // when & then
        mockMvc.perform(post("/orders/{id}/cancel", orderId))
            .andExpect(status().isOk)
    }
}
```

#### 레포지토리 테스트 (@DataJpaTest)

```kotlin
@DataJpaTest
class OrderRepositoryTest {

    @Autowired
    private lateinit var orderRepository: OrderRepository

    @Test
    fun `주문을_저장하고_조회할_수_있다`() {
        val order = Order.create("상품A", 2)
        orderRepository.save(order)

        val found = orderRepository.findById(order.id).orElseThrow()
        assertThat(found.productName).isEqualTo("상품A")
    }
}
```

### 단위 테스트 (Unit Test)

**정의:** 가장 작은 단위(클래스/함수)를 독립적으로 검증

**특징:**
- 외부 시스템(DB, 네트워크 등)에 의존하면 안 됨
- 빠르고 고립적이어야 함

**비즈니스 흐름:** JUnit + Mock 단위 테스트
**도메인 모델:** 순수 언어 테스트로도 가능

### 통합 테스트

주요 시나리오를 end-to-end로 테스트해야 할 때 사용

### 테스트 더블 (Test Double)

테스트 코드에서 실제 객체를 대신해서 사용되는 대체 객체

#### 1. Dummy
- **목적**: 값 채우기용. 실제 로직은 없음
- **예시**: `new DummyUser()`

#### 2. Fake
- **목적**: 단순 구현체. 실제 동작하지만 가벼움
- **예시**: `InMemoryUserRepository` (DB 대신 메모리 리스트 사용)

#### 3. Stub
- **목적**: 고정된 값을 반환
- **예시**: `when(user.getName()).thenReturn("Grimoire")`

#### 4. Spy
- **목적**: 실제 동작하면서 호출 여부, 인자 기록
- **예시**: `verify(spyService).run()`

#### 5. Mock
- **목적**: 사전에 정의한 동작 및 기대를 검증
- **예시**: `verify(mock).calledWith("param")`

### AAA 패턴 (Arrange-Act-Assert)

테스트의 3단계 구조:

```java
// Arrange: 테스트 준비
Calculator calc = new Calculator();

// Act: 실제 코드 실행
int result = calc.add(2, 3);

// Assert: 결과 검증
assertEquals(5, result);
```

### FIRST 원칙

좋은 테스트의 특징:

- **Fast**: 빠르고
- **Isolated**: 고립되며
- **Repeatable**: 반복 가능하고
- **Self-Validating**: 결과 자동 검증
- **Timely**: 코드 작성 직후 바로 작성됨

## 실무 적용

### Spock vs JUnit

**Spock:**
- BDD 기반의 자연어적인 Groovy 기반 테스트 프레임워크
- 매개변수가 많고 여러 데이터에 따른 테스트에 유리
- 요즘 Java/Kotlin과 충돌 발생하기도 함
- Spring Boot 기본은 JUnit이며, 현재는 기피하는 추세

**권장:** Spring Boot 기본 스택인 JUnit 5 + Mockito 조합 사용

### 테스트 전략

1. **도메인 모델**: 순수 단위 테스트 (외부 의존성 없음)
2. **비즈니스 로직**: JUnit + Mock으로 Service 계층 테스트
3. **계층별 검증**: `@WebMvcTest`, `@DataJpaTest` 슬라이스 테스트
4. **주요 시나리오**: 통합 테스트로 end-to-end 검증

## 관련 노트

- [[@Transactional]]
- [[단일 책임 원칙]]

#spring #test #junit
