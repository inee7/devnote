---
tags: [kotlin, railway-oriented-programming, result, error-handling, functional-programming]
---

## 한 줄 요약

레일웨이 프로그래밍은 함수의 실행 흐름을 '성공'과 '실패' 두 트랙으로 나누어 관리하는 패턴으로, Kotlin의 Result 타입을 활용하면 예외 처리를 선언적이고 안전하게 구현할 수 있다.

## 핵심 정리

**레일웨이 프로그래밍의 핵심**
- 모든 함수가 Result<T> (성공 또는 실패) 컨테이너를 반환
- 성공 트랙: 다음 함수로 값 전달
- 실패 트랙: 나머지 함수는 실행되지 않고 실패 객체가 끝까지 전달

**Kotlin의 Result<T>**
- Result.Success<T>: 성공한 값
- Result.Failure<Throwable>: 실패한 예외
- runCatching으로 쉽게 Result 생성

**주요 연산**
- flatMap: Result를 반환하는 함수 체이닝
- map: 일반 함수 체이닝
- fold: 성공/실패에 따른 분기 처리

**장점**
- 명시적인 오류 처리 (함수 시그니처로 실패 가능성 표현)
- try-catch 중첩 제거로 가독성 향상
- 컴파일 타임에 오류 처리 강제
- 함수 조합 용이

 ## 상세 내용

### 기존 예외 처리의 문제점

**try-catch의 문제**

```kotlin
// ❌ 가독성 저하
fun registerUser(email: String, name: String): User {
    try {
        validateEmail(email)
    } catch (e: Exception) {
        throw IllegalArgumentException("이메일 검증 실패")
    }

    try {
        val existing = userRepository.findByEmail(email)
        if (existing != null) {
            throw IllegalStateException("중복 이메일")
        }
    } catch (e: Exception) {
        throw DatabaseException(e)
    }

    try {
        return userRepository.save(User(email, name))
    } catch (e: Exception) {
        throw DatabaseException(e)
    }
}
```

문제점:
- try-catch가 비즈니스 로직 중간에 끼어들어 가독성 저하
- 예외 처리를 잊거나 누락하기 쉬움
- 제어 흐름이 불명확 (예외가 어디서 발생할지 예측 어려움)

### 레일웨이 프로그래밍 개념

**두 개의 트랙**

```
성공 트랙:  [검증] → [중복체크] → [저장] → Success(User)
                ↓          ↓         ↓
실패 트랙:  Failure ──────────────────→ Failure
```

한 단계라도 실패하면 실패 트랙으로 전환되고, 나머지 단계는 건너뛴다.

### Kotlin의 Result<T>

**기본 사용법**

```kotlin
// 성공 케이스
val success: Result<Int> = runCatching {
    10
}
// Result.Success(10)

// 실패 케이스
val failure: Result<Int> = runCatching {
    throw IllegalArgumentException("오류")
}
// Result.Failure(IllegalArgumentException)
```

**Result의 주요 메서드**

```kotlin
val result: Result<Int> = runCatching { 10 }

// 값 추출
result.getOrNull()  // 10 또는 null
result.getOrDefault(0)  // 10 또는 기본값
result.getOrElse { 0 }  // 10 또는 람다 결과

// 예외 추출
result.exceptionOrNull()  // null 또는 예외

// 성공/실패 확인
result.isSuccess  // true
result.isFailure  // false
```

### flatMap을 통한 체이닝

**flatMap 확장 함수**

표준 라이브러리에 없으므로 직접 구현한다.

```kotlin
inline fun <T, R> Result<T>.flatMap(
    transform: (T) -> Result<R>
): Result<R> {
    return when (val value = getOrNull()) {
        null -> Result.failure(exceptionOrNull()!!)
        else -> transform(value)
    }
}
```

**사용 예시**

```kotlin
fun registerUser(email: String, name: String): Result<User> {
    return validateInput(email, name)
        .flatMap { checkEmailDuplication(email) }
        .flatMap { saveUser(email, name) }
}

private fun validateInput(email: String, name: String): Result<Unit> =
    runCatching {
        require(isEmailValid(email)) { "유효하지 않은 이메일 형식" }
        require(name.isNotBlank()) { "이름은 비어있을 수 없음" }
    }

private fun checkEmailDuplication(email: String): Result<String> =
    runCatching {
        val existing = userRepository.findByEmail(email)
        check(existing == null) { "이미 존재하는 이메일: $email" }
        email  // 성공 시 이메일 반환
    }

private fun saveUser(email: String, name: String): Result<User> =
    runCatching {
        userRepository.save(User(email, name))
    }
```

**동작 원리**

1. `validateInput` 실행 → Success(Unit)
2. flatMap으로 `checkEmailDuplication` 실행 → Success(email)
3. flatMap으로 `saveUser` 실행 → Success(User)

만약 2번에서 실패하면:
1. `validateInput` 실행 → Success(Unit)
2. flatMap으로 `checkEmailDuplication` 실행 → Failure(예외)
3. flatMap은 Failure를 그대로 반환 (saveUser는 실행 안됨)

### 커스텀 비즈니스 에러

**sealed class로 도메인 에러 정의**

```kotlin
sealed class AppError(val message: String) {
    data class ValidationError(
        val field: String,
        val reason: String
    ) : AppError("유효성 검사 실패: $field - $reason")

    data class DuplicateUserError(
        val email: String
    ) : AppError("이미 존재하는 이메일: $email")

    data class DatabaseError(
        val cause: Throwable
    ) : AppError("데이터베이스 오류: ${cause.message}")

    data class NotFoundError(
        val resourceId: String
    ) : AppError("리소스를 찾을 수 없음: $resourceId")
}
```

**Either 타입 정의**

Result는 Throwable만 담을 수 있으므로, 커스텀 에러를 담는 Either 타입을 정의한다.

```kotlin
sealed class Either<out L, out R> {
    data class Left<out L>(val value: L) : Either<L, Nothing>()
    data class Right<out R>(val value: R) : Either<Nothing, R>()
}

// 비즈니스 로직용 타입 별칭
typealias ServiceResult<T> = Either<AppError, T>
```

**Either 확장 함수**

```kotlin
// flatMap (andThen)
inline fun <L, R, T> Either<L, R>.flatMap(
    transform: (R) -> Either<L, T>
): Either<L, T> {
    return when (this) {
        is Either.Right -> transform(this.value)
        is Either.Left -> this
    }
}

// map
inline fun <L, R, T> Either<L, R>.map(
    transform: (R) -> T
): Either<L, T> {
    return when (this) {
        is Either.Right -> Either.Right(transform(this.value))
        is Either.Left -> this
    }
}

// fold
inline fun <L, R, T> Either<L, R>.fold(
    onLeft: (L) -> T,
    onRight: (R) -> T
): T {
    return when (this) {
        is Either.Left -> onLeft(this.value)
        is Either.Right -> onRight(this.value)
    }
}
```

**커스텀 에러를 사용한 서비스**

```kotlin
@Service
class UserService(
    private val userRepository: UserRepository
) {
    fun registerUser(email: String, name: String): ServiceResult<User> {
        return validateInput(email, name)
            .flatMap { checkEmailDuplication(email) }
            .flatMap { saveUser(email, name) }
    }

    private fun validateInput(
        email: String,
        name: String
    ): ServiceResult<Unit> {
        return when {
            !isEmailValid(email) -> Either.Left(
                AppError.ValidationError("email", "유효하지 않은 형식")
            )
            name.isBlank() -> Either.Left(
                AppError.ValidationError("name", "비어있을 수 없음")
            )
            else -> Either.Right(Unit)
        }
    }

    private fun checkEmailDuplication(email: String): ServiceResult<String> {
        val existing = userRepository.findByEmail(email)
        return if (existing != null) {
            Either.Left(AppError.DuplicateUserError(email))
        } else {
            Either.Right(email)
        }
    }

    private fun saveUser(email: String, name: String): ServiceResult<User> {
        return try {
            val user = userRepository.save(User(email, name))
            Either.Right(user)
        } catch (e: Exception) {
            Either.Left(AppError.DatabaseError(e))
        }
    }

    private fun isEmailValid(email: String): Boolean {
        return email.contains("@") && email.contains(".")
    }
}
```

### 컨트롤러에서 Result 처리

**Result.fold로 HTTP 응답 생성**

```kotlin
@RestController
@RequestMapping("/users")
class UserController(
    private val userService: UserService
) {
    @PostMapping("/register")
    fun register(
        @RequestBody request: RegisterUserRequest
    ): ResponseEntity<*> {
        val result = userService.registerUser(request.email, request.name)

        return result.fold(
            onSuccess = { user ->
                ResponseEntity.ok(UserResponse.from(user))
            },
            onFailure = { error ->
                when (error) {
                    is IllegalArgumentException ->
                        ResponseEntity.badRequest().body(ErrorResponse(error.message))
                    is IllegalStateException ->
                        ResponseEntity.status(HttpStatus.CONFLICT)
                            .body(ErrorResponse(error.message))
                    else ->
                        ResponseEntity.internalServerError()
                            .body(ErrorResponse("알 수 없는 오류"))
                }
            }
        )
    }
}

data class RegisterUserRequest(
    val email: String,
    val name: String
)

data class UserResponse(
    val id: Long,
    val email: String,
    val name: String
) {
    companion object {
        fun from(user: User) = UserResponse(
            id = user.id!!,
            email = user.email,
            name = user.name
        )
    }
}

data class ErrorResponse(
    val message: String
)
```

**커스텀 에러로 더 명확한 응답**

```kotlin
@PostMapping("/register")
fun register(
    @RequestBody request: RegisterUserRequest
): ResponseEntity<*> {
    val result = userService.registerUser(request.email, request.name)

    return result.fold(
        onLeft = { error ->
            when (error) {
                is AppError.ValidationError ->
                    ResponseEntity.badRequest().body(mapOf(
                        "type" to "VALIDATION_ERROR",
                        "field" to error.field,
                        "message" to error.message
                    ))
                is AppError.DuplicateUserError ->
                    ResponseEntity.status(HttpStatus.CONFLICT).body(mapOf(
                        "type" to "DUPLICATE_USER",
                        "message" to error.message
                    ))
                is AppError.DatabaseError ->
                    ResponseEntity.internalServerError().body(mapOf(
                        "type" to "DATABASE_ERROR",
                        "message" to "서버 오류가 발생했습니다"
                    ))
                is AppError.NotFoundError ->
                    ResponseEntity.notFound().build<Any>()
            }
        },
        onRight = { user ->
            ResponseEntity.ok(UserResponse.from(user))
        }
    )
}
```

### 복잡한 비즈니스 로직 체이닝

**주문 처리 예시**

```kotlin
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val productRepository: ProductRepository,
    private val inventoryService: InventoryService,
    private val paymentService: PaymentService
) {
    fun createOrder(
        userId: Long,
        productId: Long,
        quantity: Int
    ): ServiceResult<Order> {
        return validateQuantity(quantity)
            .flatMap { findProduct(productId) }
            .flatMap { product -> checkInventory(product, quantity) }
            .flatMap { product -> reserveInventory(product.id, quantity) }
            .flatMap { product -> processPayment(userId, product, quantity) }
            .flatMap { payment -> saveOrder(userId, productId, quantity, payment) }
    }

    private fun validateQuantity(quantity: Int): ServiceResult<Unit> {
        return if (quantity > 0) {
            Either.Right(Unit)
        } else {
            Either.Left(AppError.ValidationError("quantity", "양수여야 함"))
        }
    }

    private fun findProduct(productId: Long): ServiceResult<Product> {
        val product = productRepository.findById(productId)
        return if (product != null) {
            Either.Right(product)
        } else {
            Either.Left(AppError.NotFoundError("product:$productId"))
        }
    }

    private fun checkInventory(
        product: Product,
        quantity: Int
    ): ServiceResult<Product> {
        val available = inventoryService.getAvailableQuantity(product.id)
        return if (available >= quantity) {
            Either.Right(product)
        } else {
            Either.Left(AppError.ValidationError(
                "quantity",
                "재고 부족: 요청 $quantity, 가용 $available"
            ))
        }
    }

    private fun reserveInventory(
        productId: Long,
        quantity: Int
    ): ServiceResult<Product> {
        return try {
            inventoryService.reserve(productId, quantity)
            findProduct(productId)
        } catch (e: Exception) {
            Either.Left(AppError.DatabaseError(e))
        }
    }

    private fun processPayment(
        userId: Long,
        product: Product,
        quantity: Int
    ): ServiceResult<Payment> {
        val amount = product.price * quantity
        return try {
            val payment = paymentService.charge(userId, amount)
            Either.Right(payment)
        } catch (e: PaymentFailedException) {
            // 재고 롤백
            inventoryService.release(product.id, quantity)
            Either.Left(AppError.ValidationError("payment", "결제 실패"))
        }
    }

    private fun saveOrder(
        userId: Long,
        productId: Long,
        quantity: Int,
        payment: Payment
    ): ServiceResult<Order> {
        return try {
            val order = Order(
                userId = userId,
                productId = productId,
                quantity = quantity,
                paymentId = payment.id
            )
            Either.Right(orderRepository.save(order))
        } catch (e: Exception) {
            // 결제 및 재고 롤백
            paymentService.refund(payment.id)
            inventoryService.release(productId, quantity)
            Either.Left(AppError.DatabaseError(e))
        }
    }
}
```

## 실무 적용

### Arrow-kt 라이브러리 활용

실무에서는 Either, Result 등을 제공하는 Arrow-kt 라이브러리를 사용하는 것이 일반적이다.

**의존성 추가**

```kotlin
// build.gradle.kts
dependencies {
    implementation("io.arrow-kt:arrow-core:1.2.0")
}
```

**Arrow의 Either 사용**

```kotlin
import arrow.core.Either
import arrow.core.flatMap
import arrow.core.left
import arrow.core.right

fun registerUser(email: String, name: String): Either<AppError, User> {
    return validateInput(email, name)
        .flatMap { checkEmailDuplication(email) }
        .flatMap { saveUser(email, name) }
}

private fun validateInput(
    email: String,
    name: String
): Either<AppError, Unit> {
    return when {
        !isEmailValid(email) ->
            AppError.ValidationError("email", "유효하지 않음").left()
        name.isBlank() ->
            AppError.ValidationError("name", "비어있음").left()
        else -> Unit.right()
    }
}
```

### 트랜잭션 처리

레일웨이 패턴에서 트랜잭션은 가장 바깥쪽에서 관리한다.

```kotlin
@Service
@Transactional
class OrderService(
    private val orderRepository: OrderRepository,
    private val inventoryService: InventoryService,
    private val paymentService: PaymentService
) {
    // @Transactional이 전체 체인을 감싸므로
    // 중간에 실패하면 자동으로 롤백
    fun createOrder(
        userId: Long,
        productId: Long,
        quantity: Int
    ): ServiceResult<Order> {
        return validateQuantity(quantity)
            .flatMap { findProduct(productId) }
            .flatMap { product -> checkInventory(product, quantity) }
            .flatMap { product -> reserveInventory(product.id, quantity) }
            .flatMap { product -> processPayment(userId, product, quantity) }
            .flatMap { payment -> saveOrder(userId, productId, quantity, payment) }
    }
}
```

### 비동기 처리 (Coroutine)

```kotlin
suspend fun registerUserAsync(
    email: String,
    name: String
): ServiceResult<User> {
    return validateInput(email, name)
        .flatMap { checkEmailDuplicationAsync(email) }
        .flatMap { saveUserAsync(email, name) }
}

private suspend fun checkEmailDuplicationAsync(
    email: String
): ServiceResult<String> {
    return withContext(Dispatchers.IO) {
        val existing = userRepository.findByEmailAsync(email)
        if (existing != null) {
            AppError.DuplicateUserError(email).left()
        } else {
            email.right()
        }
    }
}
```

## 금융권에서의 고려사항

**거래 처리 파이프라인**
- 각 단계(검증, 한도 체크, 실행, 로깅)를 명확히 분리
- 실패 시 자동으로 롤백 처리
- 감사 로그에 성공/실패 원인 명확히 기록

**에러 분류**
- 비즈니스 에러 (복구 가능): Either의 Left로 처리
- 기술 에러 (복구 불가): 예외 발생 및 모니터링

**트랜잭션 일관성**
- @Transactional로 전체 파이프라인을 감싸서 원자성 보장
- 중간 단계 실패 시 자동 롤백

**명확한 실패 원인**
- sealed class로 실패 타입을 명확히 정의
- 감사 로그에서 정확한 실패 원인 추적 가능

```kotlin
// 금융 거래 예시
@Service
@Transactional
class TransactionService(
    private val accountRepository: AccountRepository,
    private val transactionRepository: TransactionRepository,
    private val auditLogger: AuditLogger
) {
    fun transfer(
        fromAccountId: Long,
        toAccountId: Long,
        amount: Long
    ): ServiceResult<Transaction> {
        return validateAmount(amount)
            .flatMap { findAccount(fromAccountId) }
            .flatMap { from -> checkBalance(from, amount) }
            .flatMap { from -> findAccount(toAccountId).map { to -> from to to } }
            .flatMap { (from, to) -> executeTransfer(from, to, amount) }
            .also { result ->
                auditLogger.log(result)  // 성공/실패 모두 로깅
            }
    }

    private fun validateAmount(amount: Long): ServiceResult<Unit> {
        return when {
            amount <= 0 -> AppError.ValidationError(
                "amount", "양수여야 함"
            ).left()
            amount > 10_000_000 -> AppError.ValidationError(
                "amount", "1회 한도 초과"
            ).left()
            else -> Unit.right()
        }
    }

    private fun checkBalance(
        account: Account,
        amount: Long
    ): ServiceResult<Account> {
        return if (account.balance >= amount) {
            account.right()
        } else {
            AppError.InsufficientBalanceError(
                accountId = account.id,
                requested = amount,
                current = account.balance
            ).left()
        }
    }

    private fun executeTransfer(
        from: Account,
        to: Account,
        amount: Long
    ): ServiceResult<Transaction> {
        return try {
            from.balance -= amount
            to.balance += amount
            accountRepository.save(from)
            accountRepository.save(to)

            val transaction = Transaction(
                fromAccountId = from.id,
                toAccountId = to.id,
                amount = amount
            )
            transactionRepository.save(transaction).right()
        } catch (e: Exception) {
            AppError.DatabaseError(e).left()
        }
    }
}
```

---

**참고 자료**
- [Railway Oriented Programming (F# for fun and profit)](https://fsharpforfunandprofit.com/rop/)
- [Arrow-kt Documentation](https://arrow-kt.io/)

