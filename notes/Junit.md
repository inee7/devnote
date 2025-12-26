# JUnit + MockK + AssertJ 통합 가이드
---

## 📘 1. JUnit

### ✅ 기본 구조 & 네이밍

```kotlin
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.DisplayName

class DiscountServiceNamingTest {

    // 백틱 문장형
    @Test
    fun `VIP는 10% 할인이 적용된다`() {
        val sut = DiscountService()
        assertThat(sut.apply("VIP", 10_000)).isEqualTo(9_000)
    }

    // DisplayName 사용
    @DisplayName("VIP는 10% 할인이 적용된다")
    @Test
    fun vip_discount() {
        val sut = DiscountService()
        assertThat(sut.apply("VIP", 10_000)).isEqualTo(9_000)
    }
}
```

---

### ✅ 중첩 네이밍(@Nested) — 계층적 테스트 구조

```kotlin
@DisplayName("프로모션 API")
class PromotionApiTest {

    @Nested
    @DisplayName("POST /promotions")
    inner class Create {

        @Nested
        @DisplayName("유효한 요청일 때")
        inner class WhenValid {
            @Test
            @DisplayName("201 Created와 생성된 ID를 반환한다")
            fun returns_created_with_id() {}
        }

        @Nested
        @DisplayName("잘못된 요청일 때")
        inner class WhenInvalid {
            @Test
            @DisplayName("400 Bad Request를 반환한다")
            fun returns_bad_request() {}
        }
    }
}
```

출력 예:  
`프로모션 API > POST /promotions > 유효한 요청일 때 > 201 Created와 생성된 ID를 반환한다`

---

### ✅ 데이터 기반 테스트 (파라미터라이즈드)

```kotlin
@ParameterizedTest(name = "[{index}] role={0} -> rate={1}")
@CsvSource("VIP,0.10", "GOLD,0.05", "SILVER,0.02")
fun discount_rate(role: String, expected: Double) {
    assertThat(sut.rateOf(role)).isEqualTo(expected)
}
```

복잡한 경우:

```kotlin
companion object {
    @JvmStatic
    fun cases() = listOf(
        Arguments.of("VIP", Money.of(10_000), Money.of(9_000)),
        Arguments.of("GOLD", Money.of(10_000), Money.of(9_500))
    )
}

@ParameterizedTest(name = "{0} {1} -> {2}")
@MethodSource("cases")
fun calc(role: String, price: Money, expected: Money) {
    assertThat(sut.calc(role, price)).isEqualTo(expected)
}
```

---

### ✅ 수명주기 (Fixture)

|Spock|JUnit5|설명|
|---|---|---|
|`setup()` / `cleanup()`|`@BeforeEach` / `@AfterEach`|각 테스트 전후 실행|
|`setupSpec()` / `cleanupSpec()`|`@BeforeAll` / `@AfterAll`|클래스 단 1회 실행|

#### 예 1) 테스트마다 초기화/정리

```kotlin
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.*

class CartTest {

    private lateinit var cart: MutableList<String>

    @BeforeEach
    fun setup() {            // 각 테스트 직전
        cart = mutableListOf()
    }

    @AfterEach
    fun cleanup() {          // 각 테스트 직후
        cart.clear()
    }

    @Test
    fun `상품 추가`() {
        cart += "A"
        assertThat(cart).hasSize(1)
    }

    @Test
    fun `여러 상품 추가`() {
        cart += "A"; cart += "B"
        assertThat(cart).containsExactly("A", "B")
    }
}
```

#### 예 2) 클래스당 1회 초기화/정리 — `companion object` + `@JvmStatic`

```kotlin
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.*

class DbTest {

    companion object {
        private lateinit var db: FakeDb

        @JvmStatic
        @BeforeAll
        fun setupAll() {          // 클래스 시작 전 1회
            db = FakeDb().connect()
        }

        @JvmStatic
        @AfterAll
        fun cleanupAll() {        // 클래스 종료 후 1회
            db.close()
        }
    }

    @Test
    fun `쿼리 동작`() {
        assertThat(db.query("select 1")).isEqualTo(1)
    }
}

/** 데모용 페이크 */
class FakeDb {
    fun connect(): FakeDb = this
    fun close() {}
    fun query(sql: String): Int = 1
}
```

#### 예 3) 클래스당 1회 초기화/정리 — `@TestInstance(PER_CLASS)` (비정적 허용)

```kotlin
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.*

@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class DbPerClassTest {

    private lateinit var db: FakeDb

    @BeforeAll
    fun setupAll() {         // 인스턴스 메서드로 사용 가능
        db = FakeDb().connect()
    }

    @AfterAll
    fun cleanupAll() {
        db.close()
    }

    @BeforeEach
    fun setupEach() { /* 테스트마다 초기화 필요 시 */ }

    @AfterEach
    fun cleanupEach() { /* 테스트마다 정리 필요 시 */ }

    @Test
    fun `select 1`() {
        assertThat(db.query("select 1")).isEqualTo(1)
    }
}
```

> **실행 순서 요약**  
> `@BeforeAll` → (반복) `@BeforeEach` → `@Test` → `@AfterEach` → … → `@AfterAll`

---

### ✅ 예외 및 비활성화 테스트

```kotlin
@Test
fun `잔액이 부족하면 예외`() {
    assertThrows<InsufficientBalance> {
        sut.pay(userId, amount)
    }
}

@Timeout(1)
@Disabled("일시적으로 비활성화")
@Test
fun flaky_test() { }
```

---

## 🧱 2. MockK

### ✅ Mock 객체 생성 방식

|구분|설명|초기화 필요|권장 상황|
|---|---|---|---|
|**팩토리 함수**|`mockk()`, `spyk()` 즉시 생성|❌|단일/소규모 테스트|
|**어노테이션 기반**|`@MockK`, `@SpyK`, `@InjectMockKs`|✅|클래스 단위 자동 주입|

```kotlin

// 팩토리 함수
val repo = mockk<UserRepository>()
val service = spyk<RealService>()

// 어노테이션
@ExtendWith(MockKExtension::class)
class OrderServiceTest {
    @MockK lateinit var repo: OrderRepository
    @RelaxedMockK lateinit var mailer: Mailer
    @SpyK lateinit var calc: Calculator
    @InjectMockKs lateinit var sut: OrderService
}
```

---

### ✅ Stub & Verify

#### Stub — `every { } returns/throws`

##### 1️⃣ 기본 패턴

```kotlin
every { repo.findById(1) } returns User(1, "Alice")
every { repo.findById(999) } returns null
every { repo.save(any()) } throws IllegalStateException("Duplicate")
```

|구문|의미|
|---|---|
|`returns`|항상 **고정된 값**을 반환|
|`throws`|호출 시 **예외**를 던짐|
|`answers`|**인자/상태 기반**으로 동적으로 결과 생성|

---

##### 2️⃣ `answers` — 인자 기반 동적 처리

```kotlin
every { repo.save(any()) } answers {
    val arg = firstArg<User>()
    arg.copy(id = 42) // 인자 기반으로 반환 값 만들기
}
```

`answers` 블록에서 자주 쓰는 도구들:

|도구|설명|
|---|---|
|`firstArg<T>()`, `secondArg()`, `lastArg()`|전달된 인자 꺼내기|
|`invocation.args[n]`|n번째 인자 직접 접근|
|`invocation.invocationStr`|호출 문자열|
|`callOriginal()`|원본 메서드 호출(Spy에서만)|

---

##### 3️⃣ 다중 Stub — `returnsMany`, `andThen`

```kotlin
every { random.nextInt() } returnsMany listOf(1, 2, 3)
// 이후 호출은 마지막 값(3) 유지

every { service.call() } returns "A" andThen "B" andThen "C"
```

> 여러 번 호출되는 메서드의 반환을 **순차적으로** 구성할 때 유용합니다.

---

##### 4️⃣ Nullable 대응 (`returns null`)

```kotlin
every { repo.findOrNull(any()) } returns null // 반환 타입이 T? 여야 함
```

> Kotlin에서는 `returns null` 시 **함수 반환 타입이 nullable**이어야 합니다.

---

##### 5️⃣ `Unit`/void 함수 Stub — `just Runs`

```kotlin
every { mailer.send(any(), any()) } just Runs
```

> 아무 동작도 하지 않는 `Unit` 반환을 표현 (Mockito의 `doNothing()` 유사).

---

##### 6️⃣ suspend 함수 Stub — `coEvery`

```kotlin
import io.mockk.coEvery
coEvery { repo.load("u1") } returns User("u1")
coEvery { repo.save(any()) } answers { firstArg<User>().copy(id = 99) }
```

> 코루틴 함수는 반드시 `coEvery`/`coAnswers`를 사용하세요.

---

##### 7️⃣ `throws` vs `answers` (조건부 예외)

|상황|예시|특징|
|---|---|---|
|**고정된 예외 발생**|`throws IllegalStateException()`|항상 같은 예외 발생|
|**조건부 발생**|`answers { if (firstArg<Int>() < 0) throw ... else 1 }`|인자/상태에 따라 분기|

---

##### 8️⃣ 콜백/람다와 함께 Stub (부작용 시뮬레이션)

```kotlin
val cbSlot = slot<(Boolean) -> Unit>()

every { mailer.send(any(), capture(cbSlot)) } answers {
    cbSlot.captured(true) // 콜백을 테스트에서 직접 실행
}
```

> Stub은 반환뿐 아니라 **콜백 실행**과 같은 **부작용 시뮬레이션**에도 사용됩니다.

---

##### 9️⃣ Return/Throw/Answer 종합 예시

```kotlin
every { repo.find(any()) } returns User("A")
every { repo.find("B") } throws NoSuchElementException("B not found")
every { repo.save(any()) } answers { firstArg<User>().copy(saved = true) }
```

---

##### 🔟 실무 팁 요약

|패턴|권장 상황|
|---|---|
|`returns`|고정값 Stub|
|`answers`|인자 기반 동적 처리 · 콜백 실행 · 부작용|
|`throws`|예외 시나리오|
|`returnsMany`/`andThen`|반복 호출 시 순차 반환|
|`just Runs`|`Unit` 반환|
|`coEvery`|suspend 함수|

---

#### Mock — `verify` / `confirmVerified`

##### 1️⃣ 기본 검증 패턴

```kotlin
val mailer = mockk<Mailer>()
every { mailer.send(any(), any()) } returns Unit

// act
sendWelcome(mailer, "alice")

// verify
verify(exactly = 1) { 
    mailer.send("alice", match { it.contains("welcome") }) 
}
confirmVerified(mailer) // 추가 호출 없음 보장
```

---

##### 2️⃣ `wasNot Called` — “호출 자체가 없어야 함”

```kotlin
val mailer = mockk<Mailer>()

// act
processWithoutSending(mailer)

// verify
verify { mailer wasNot Called } // ✅ mailer의 어떤 메서드도 호출되면 실패
```

📌 특정 메서드만 0회 호출인지 검증하고 싶다면:

```kotlin
verify(exactly = 0) { mailer.send(any(), any()) }
```

|구분|설명|
|---|---|
|`verify { mock wasNot Called }`|mock 전체 호출 없음|
|`verify(exactly = 0)`|특정 메서드 호출 없음|

---

##### 3️⃣ 순서 검증 (`verifyOrder`, `verifySequence`)

```kotlin
verifyOrder {              // 지정 호출 순서만 보장(사이 호출 허용)
    repo.lock()
    repo.save(any())
}

verifySequence {           // 지정 호출이 정확히 이 순서로만 실행되어야 함
    repo.lock()
    repo.save(any())
    repo.unlock()
}
```

|메서드|설명|
|---|---|
|`verifyOrder`|지정된 호출 순서만 중요 (중간에 다른 호출 가능)|
|`verifySequence`|순서 + 호출 집합 모두 일치해야 함 (중간 호출 불가)|

---

##### 4️⃣ verify 옵션 정리

|옵션|설명|예시|
|---|---|---|
|`exactly = n`|정확히 n회 호출|`verify(exactly = 1) { mailer.send(..) }`|
|`atLeast = n` / `atMost = n`|최소/최대 호출 횟수|`verify(atLeast = 1) { repo.flush() }`|
|`timeout = ms`|지정 시간 내 호출 검증 (비추천)|`verify(timeout = 200) { queue.offer(..) }`|
|`wasNot Called`|mock 전체 무호출 보장|`verify { mock wasNot Called }`|
|`confirmVerified(mock)`|검증된 호출 외 잔여 호출 없음|`confirmVerified(repo)`|
|`inverse = true`|블록 내 호출 없어야 함|`verify(inverse = true) { mailer.send(..) }`|

---
### ✅ 인자 매처 (Argument Matchers)

|매처|설명|예시|
|---|---|---|
|`any()`|모든 값 (null 제외)|`every { repo.save(any()) }`|
|`anyNullable()`|null 포함 허용|`every { repo.save(anyNullable()) }`|
|`eq(value)`|특정 값|`every { repo.save(eq(user)) }`|
|`match { it.xxx }`|조건식|`every { repo.save(match { it.age > 20 }) }`|
|`isNull()` / `isNotNull()`|null 여부|`every { repo.save(isNull()) }`|

> **주의:** 매처와 실값을 **혼용할 수 없습니다.**  
> 모든 인자를 매처로 써야 합니다.

```kotlin
// ❌ 잘못된 예시
every { repo.save(User(1, any())) }
// ✅ 올바른 예시
every { repo.save(match { it.id == 1 }) }
```

---

### ✅ 인자 캡처

**verify 대신 “인자로 무엇이 전달되었는가”를 직접 검증**할 때 사용

```kotlin
val slot = slot<Order>()
every { repo.save(capture(slot)) } returns Order(id=1)
assertThat(slot.captured.amount).isEqualTo(10_000)
```

---

### ✅  콜백/람다 인자 캡처 및 실행 제어

##### 예제: 콜백 파라미터 테스트

```kotlin
interface Mailer {
    fun send(to: String, body: String, onComplete: (Boolean) -> Unit)
}

class Notifier(private val mailer: Mailer) {
    fun sendWelcome(to: String): Boolean {
        var ok = false
        mailer.send(to, "Welcome!", onComplete = { success -> ok = success })
        return ok
    }
}
```

**테스트 코드:**
```kotlin
val mailer = mockk<Mailer>()
val notifier = Notifier(mailer)

val cbSlot = slot<(Boolean) -> Unit>()
every { mailer.send(any(), any(), capture(cbSlot)) } answers {
    cbSlot.captured.invoke(true) // ✅ 콜백 직접 실행
}

val result = notifier.sendWelcome("alice")

verify(exactly = 1) { mailer.send("alice", match { it.contains("Welcome!") }, any()) }
assertThat(result).isTrue()
```

---

### ✅  실행 블록 캡처 (예: 트랜잭션)

```kotlin
interface TxManager { fun <T> inTransaction(block: () -> T): T }

class Service(private val tx: TxManager, private val repo: Repo) {
    fun save(name: String) {
        tx.inTransaction {
            repo.save(name)
        }
    }
}

val tx = mockk<TxManager>()
val repo = mockk<Repo>(relaxed = true)
val sut = Service(tx, repo)

val blockSlot = slot<() -> Unit>()
every { tx.inTransaction(capture(blockSlot)) } answers {
    blockSlot.captured.invoke() // ✅ 트랜잭션 블록 직접 실행
}

sut.save("alice")

verifyOrder {
    tx.inTransaction(any())
    repo.save("alice")
}
confirmVerified(tx, repo)
```

---

### ✅ Spy (부분 모킹)

```kotlin
val real = RealCalculator()
val calc = spyk(real)                  // 기본은 real 동작 수행
every { calc.divide(10, 0) } returns Int.MAX_VALUE  // 일부만 덮어쓰기

assertThat(calc.divide(10, 2)).isEqualTo(5)         // 실제 동작
assertThat(calc.divide(10, 0)).isEqualTo(Int.MAX_VALUE)
verify { calc.divide(any(), any()) }
```

> Spy는 **실제 동작**을 수행하므로 부작용이 없는 객체에 사용하세요.  
> 부작용(DB/파일/네트워크)이 있으면 Fake/Stub이 더 안전합니다.

---

### ✅ Relaxed Mock

```kotlin
val log = mockk<Logger>(relaxed = true)
```

> 설정되지 않은 함수 호출 시 0, null, false 등 기본값 반환.

---

### ✅ 코루틴 Stub/Verify

```kotlin
@Test
fun `suspend 함수 검증`() = runTest {
    coEvery { repo.load("u1") } returns User("u1")
    val result = sut.fetch("u1")
    coVerify { repo.load("u1") }
}
```

---

### ✅ 정적/싱글톤/생성자 모킹

```kotlin
mockkStatic("com.example.UtilKt")
every { topLevelFunc(any()) } returns 123

mockkObject(MySingleton)
every { MySingleton.call() } returns "mocked"
```

---

### ✅ 초기화 및 정리

```kotlin
@BeforeEach
fun setUp() { MockKAnnotations.init(this) }

@AfterEach
fun tearDown() { clearMocks(repo, answers = false) }
```

- `clearMocks()` : 기록만 초기화
- `unmockkAll()` : 전체 모킹 해제

---

## 🧩 3. AssertJ

> **JUnit 기본 단언보다 AssertJ를 권장**합니다.  
> 이유: **가독성, 풍부한 API, 명확한 실패 메시지, 체이닝, 컬렉션/맵/배열/optional을 위한 API 제공**
> 
> AssertJ는 실패 시 **기대값 vs 실제값**, **누락/추가 요소** 등을 자세히 출력합니다.

### ✅ 비교

|항목|JUnit|AssertJ|
|---|---|---|
|기본 단언|`assertEquals(a, b)`|`assertThat(b).isEqualTo(a)`|
|예외|`assertThrows<>()`|`assertThatThrownBy { }.isInstanceOf()`|
|컬렉션|`assertIterableEquals()`|`assertThat(list).containsExactly()`|

---

### ✅ 예외 단언

```kotlin
assertThatThrownBy { sut.run(-1) }
  .isInstanceOf(IllegalArgumentException::class.java)
  .hasMessage("negative!")
```

---

### ✅ 컬렉션 / 객체 비교

```kotlin
assertThat(actual)
  .containsExactly("A", "B")
  .doesNotContain("Z")
  .isSorted()
```

```kotlin
assertThat(actual)
  .usingRecursiveComparison()
  .ignoringFields("updatedAt", "id")
  .isEqualTo(expected)
```

---

### ✅ SoftAssertions

여러 단언 실패를 **한 번에 확인**할 수 있습니다.

```kotlin
val softly = SoftAssertions()
softly.assertThat(a).isEqualTo(1)
softly.assertThat(b).isGreaterThan(0)
softly.assertAll()
```

---

### ✅ 날짜/시간, 필터링, 추출

```kotlin
assertThat(orderTime).isBetween(start, end)
assertThat(users).filteredOn { it.active }.hasSize(2)
assertThat(users).extracting<String> { it.name }.contains("Alice")
```

---

### ✅ 커스텀 단언 (Custom Assertions)

AssertJ는 **도메인 특화 단언문**을 쉽게 확장할 수 있습니다.

```kotlin
import org.assertj.core.api.AbstractAssert
import org.assertj.core.api.Assertions.assertThat

class OrderAssert(actual: Order?) 
    : AbstractAssert<OrderAssert, Order?>(actual, OrderAssert::class.java) {

    fun hasStatus(expected: String): OrderAssert {
        isNotNull
        assertThat(actual!!.status)
            .withFailMessage("Expected status to be <%s> but was <%s>", expected, actual.status)
            .isEqualTo(expected)
        return this
    }

    fun hasAmountGreaterThan(min: Int): OrderAssert {
        isNotNull
        assertThat(actual!!.amount)
            .withFailMessage("Expected amount > %d but was %d", min, actual.amount)
            .isGreaterThan(min)
        return this
    }

    companion object {
        fun assertThatOrder(actual: Order?): OrderAssert = OrderAssert(actual)
    }
}


import org.junit.jupiter.api.Test

class OrderTest {

    @Test
    fun `주문 상태와 금액을 검증`() {
        val order = Order(id = 1, amount = 12_000, status = "PAID")

        OrderAssert.assertThatOrder(order)
            .hasStatus("PAID")
            .hasAmountGreaterThan(10_000)
    }
}
```

> Expected amount > 10000 but was 9000

#testcode #테스트코드 
