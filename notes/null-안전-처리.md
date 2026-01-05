---
tags: [kotlin, null-safety, effective-kotlin, best-practices]
---

## 한 줄 요약

null은 최대한 의미를 가져야 하며, safe call(?.), elvis(?)와 스마트 캐스팅을 적극 활용하고, !! 연산자는 미래의 불확실성 때문에 최대한 피해야 한다.

## 핵심 정리

**null 처리 원칙**
- null은 명확한 의미를 가져야 함 ("없음", "아직 초기화 안됨" 등)
- 예상치 못한 null은 즉시 오류 발생시켜야 함
- !! 연산자는 가능한 피하기 (미래에 null이 될 수 있음)

**안전한 null 처리 방법**
- Safe call (?.) 사용
- Elvis 연산자 (?:)로 기본값 제공
- 스마트 캐스팅 활용
- Contracts 기능 활용
- 방어적 프로그래밍 (require, check, requireNotNull, checkNotNull)

**nullable 제거 전략**
- 설계 단계에서 nullable을 최소화
- lateinit, lazy 활용
- sealed class로 상태 표현

## 상세 내용

### null이 의미를 가져야 하는 이유

**명확한 의미의 null**

```kotlin
// ✅ null이 "선택 안함"을 의미
data class UserProfile(
    val name: String,
    val bio: String?,  // 자기소개를 작성하지 않을 수 있음
    val profileImage: String?  // 프로필 이미지를 설정하지 않을 수 있음
)

// ✅ null이 "아직 로드 안됨"을 의미
var cachedData: Data? = null  // 캐시가 아직 초기화되지 않음
```

**의미 없는 null (안티패턴)**

```kotlin
// ❌ null의 의미가 불명확
fun getUser(id: Long): User? {
    // User가 없을 때 null? 에러일 때 null? DB 연결 실패일 때 null?
}

// ✅ 명확한 의미
fun getUser(id: Long): User {
    return userRepository.findById(id)
        ?: throw NoSuchElementException("User not found: $id")
}

// 또는 Result 타입 사용
fun getUser(id: Long): Result<User> {
    return try {
        Result.success(userRepository.findById(id))
    } catch (e: Exception) {
        Result.failure(e)
    }
}
```

### Safe Call (?.)

가장 기본적인 null 안전 처리 방법이다.

```kotlin
val user: User? = getUser()

// ✅ Safe call
user?.updateProfile()  // user가 null이면 실행되지 않음

// user가 null이면 전체 체인이 null
val nameLength = user?.profile?.name?.length  // Int?
```

**메서드 체이닝**

```kotlin
data class Order(
    val customer: Customer?,
    val items: List<Item>
)

data class Customer(
    val address: Address?
)

data class Address(
    val city: String
)

// Safe call 체이닝
val city: String? = order?.customer?.address?.city
```

### Elvis 연산자 (?:)

null일 때 기본값을 제공한다.

```kotlin
val user: User? = getUser()

// ✅ 기본값 제공
val name = user?.name ?: "Unknown"

// ✅ early return
fun processUser(user: User?) {
    val validUser = user ?: return
    // 이후 코드에서 validUser는 non-null
}

// ✅ 예외 던지기
fun processUser(user: User?) {
    val validUser = user ?: throw IllegalArgumentException("User is null")
}
```

**실용 예시**

```kotlin
// 설정값 기본값 처리
val timeout = config.getTimeout() ?: 30000

// 캐시 미스 처리
val data = cache[key] ?: fetchFromDb(key).also { cache[key] = it }

// 인증 실패 처리
val currentUser = session.getUser()
    ?: throw UnauthorizedException("로그인이 필요합니다")
```

### 스마트 캐스팅

null 체크 후 자동으로 non-null 타입으로 캐스팅된다.

```kotlin
fun processUser(user: User?) {
    if (user != null) {
        // 이 블록 안에서 user는 User 타입 (User?가 아님)
        println(user.name)
        user.updateProfile()
    }
}

// when에서도 사용 가능
fun describe(value: Any?): String {
    return when (value) {
        null -> "null"
        is String -> "String of length ${value.length}"  // 스마트 캐스트
        is Int -> "Int: $value"
        else -> "Unknown"
    }
}
```

**스마트 캐스트가 작동하지 않는 경우**

```kotlin
var user: User? = getUser()

if (user != null) {
    // 스마트 캐스트 안됨! (var는 중간에 변경될 수 있음)
    // println(user.name)  // 컴파일 에러
}

// ✅ 해결: 로컬 val로 복사
var user: User? = getUser()
val userSnapshot = user
if (userSnapshot != null) {
    println(userSnapshot.name)  // OK
}
```

**커스텀 getter는 스마트 캐스트 불가**

```kotlin
val name: String?
    get() = computeName()  // 매번 다른 값을 반환할 수 있음

if (name != null) {
    // 스마트 캐스트 안됨
    // println(name.length)  // 컴파일 에러
    println(name?.length)  // Safe call 필요
}
```

### Contracts 기능

Kotlin의 contracts는 함수의 동작을 컴파일러에게 알려주는 기능이다.

```kotlin
// isNullOrBlank의 구현
@kotlin.internal.InlineOnly
public inline fun CharSequence?.isNullOrBlank(): Boolean {
    contract {
        returns(false) implies (this@isNullOrBlank != null)
    }
    return this == null || this.isBlank()
}

// 사용
val name: String? = readLine()
if (!name.isNullOrBlank()) {
    // contract 덕분에 name이 non-null로 스마트 캐스트
    println("Length: ${name.length}")
}
```

**커스텀 contract**

```kotlin
fun requireNotNull(value: String?): String {
    contract {
        returns() implies (value != null)
    }
    return value ?: throw IllegalArgumentException("Value is null")
}

// 사용
val name: String? = getName()
val validName = requireNotNull(name)
// validName은 String 타입
```

### !! 연산자를 피해야 하는 이유

`!!`는 "나는 이 값이 null이 아니라고 확신한다"는 의미지만, 미래는 불확실하다.

```kotlin
// ❌ 지금은 확실하지만...
val max = listOf(1, 2, 3, 4, 5).max()!!  // 지금은 null이 아님
println(max)

// 리팩토링 후...
val numbers = getNumbers()  // 빈 리스트를 반환할 수 있음
val max = numbers.max()!!  // NPE 발생! 추적하기 어려움
```

**!! 대신 안전한 대안**

```kotlin
// ✅ Elvis 사용
val max = numbers.maxOrNull() ?: 0

// ✅ requireNotNull 사용
val max = requireNotNull(numbers.maxOrNull()) {
    "리스트가 비어있습니다"
}

// ✅ checkNotNull 사용
val max = checkNotNull(numbers.maxOrNull()) {
    "리스트가 비어있습니다"
}
```

### 방어적 프로그래밍

#### require - 인자 검증

```kotlin
fun setAge(age: Int?) {
    require(age != null) { "나이는 필수입니다" }
    require(age > 0) { "나이는 양수여야 합니다" }
    // 이후 age는 non-null Int
}
```

#### check - 상태 검증

```kotlin
class Connection {
    private var socket: Socket? = null

    fun send(data: String) {
        check(socket != null) { "연결이 초기화되지 않았습니다" }
        socket.write(data)  // socket은 non-null
    }
}
```

#### requireNotNull

```kotlin
fun processUser(userId: Long?) {
    val validId = requireNotNull(userId) { "User ID는 필수입니다" }
    // validId는 Long 타입
}
```

#### checkNotNull

```kotlin
class UserService {
    private var database: Database? = null

    fun getUser(id: Long): User {
        val db = checkNotNull(database) { "Database not initialized" }
        return db.query(id)
    }
}
```

### nullable을 최대한 제거하는 방법

#### lateinit 사용

```kotlin
class UserService {
    // ❌ nullable로 선언
    private var database: Database? = null

    fun init(db: Database) {
        database = db
    }

    fun getUser(id: Long): User {
        return database?.query(id)
            ?: throw IllegalStateException("Database not initialized")
    }
}

// ✅ lateinit 사용
class UserService {
    private lateinit var database: Database

    fun init(db: Database) {
        database = db
    }

    fun getUser(id: Long): User {
        return database.query(id)  // null 체크 불필요
    }
}
```

#### lazy 사용

```kotlin
class ExpensiveResource {
    // ❌ nullable + 수동 초기화
    private var cache: Cache? = null

    fun getCache(): Cache {
        if (cache == null) {
            cache = createCache()
        }
        return cache!!
    }
}

// ✅ lazy 사용
class ExpensiveResource {
    private val cache: Cache by lazy {
        createCache()
    }

    fun getCache(): Cache = cache  // null 체크 불필요
}
```

#### sealed class로 상태 표현

```kotlin
// ❌ nullable로 상태 표현
data class Response(
    val data: Data?,
    val error: Error?
)

// 문제: data와 error 둘 다 null이거나 둘 다 non-null일 수 있음

// ✅ sealed class로 명확한 상태 표현
sealed class Response {
    data class Success(val data: Data) : Response()
    data class Failure(val error: Error) : Response()
}

// 사용
when (response) {
    is Response.Success -> println(response.data)  // data는 항상 존재
    is Response.Failure -> println(response.error)  // error는 항상 존재
}
```

## 실무 적용

### Spring에서의 null 처리

```kotlin
@RestController
class UserController(
    private val userService: UserService
) {
    @GetMapping("/users/{id}")
    fun getUser(@PathVariable id: Long): UserResponse {
        // ✅ Elvis로 404 응답
        val user = userService.findById(id)
            ?: throw ResponseStatusException(HttpStatus.NOT_FOUND)

        return UserResponse.from(user)
    }

    @PostMapping("/users")
    fun createUser(@RequestBody request: CreateUserRequest): UserResponse {
        // ✅ require로 검증
        require(!request.email.isNullOrBlank()) { "이메일은 필수입니다" }

        val user = userService.createUser(request)
        return UserResponse.from(user)
    }
}
```

### Repository 패턴

```kotlin
@Repository
interface UserRepository : JpaRepository<User, Long>

@Service
class UserService(
    private val userRepository: UserRepository
) {
    // ✅ null 대신 Optional 또는 예외
    fun getUser(id: Long): User {
        return userRepository.findById(id)
            .orElseThrow { NoSuchElementException("User not found: $id") }
    }

    // ✅ nullable 반환 (조회 실패가 정상적인 경우)
    fun findUserByEmail(email: String): User? {
        return userRepository.findByEmail(email)
    }
}
```

## 금융권에서의 고려사항

**필수 데이터는 non-null**
- 계좌번호, 거래금액, 고객ID 등은 절대 null이 되어서는 안됨
- require/check로 엄격하게 검증

**선택 데이터는 명확한 의미**
- null이 "선택 안함"인지 "아직 입력 안함"인지 명확히
- 비고, 메모 등은 nullable로 허용

**에러 처리**
- 예상치 못한 null은 즉시 예외 발생
- 복구 가능한 경우만 nullable 반환

```kotlin
// 금융 거래 예시
data class Transaction(
    val id: Long,  // 필수
    val accountId: Long,  // 필수
    val amount: Long,  // 필수
    val memo: String?  // 선택
)

fun processTransaction(transaction: Transaction) {
    // 필수 필드 검증
    require(transaction.amount > 0) { "거래 금액은 양수여야 합니다" }

    // 선택 필드는 safe call
    val memoText = transaction.memo ?: "메모 없음"
    log.info("거래 처리: ${transaction.id}, 메모: $memoText")
}
```

---

**출처**
- Effective Kotlin (마르친 모스칼라)

