---
tags: [java, railway-oriented-programming, either, result, error-handling, functional-programming, sealed-class]
---

## 한 줄 요약

Java 17+의 sealed interface와 record를 활용하면 Vavr 없이도 레일웨이 패턴을 깔끔하게 구현할 수 있으며, pattern matching으로 가독성 높은 에러 처리가 가능하다.

## 핵심 정리

**순수 Java 구현의 장점**
- Vavr 같은 외부 라이브러리 불필요
- Java 17+ sealed interface로 타입 안전성 확보
- record로 불변 데이터 구조 간단히 구현
- Java 21+ pattern matching으로 가독성 향상

**핵심 타입**
- Either<L, R>: 성공(Right) 또는 실패(Left)
- Result<T>: 성공(Success) 또는 실패(Failure)
- sealed interface로 컴파일 타임 타입 체크

**주요 연산**
- flatMap: Either를 반환하는 함수 체이닝
- map: 일반 함수 체이닝
- fold: 성공/실패 분기 처리

## 상세 내용

### Either 타입 구현

**sealed interface로 Either 정의**

```java
public sealed interface Either<L, R>
    permits Either.Left, Either.Right {

    record Left<L, R>(L value) implements Either<L, R> {}
    record Right<L, R>(R value) implements Either<L, R> {}

    // 생성 메서드
    static <L, R> Either<L, R> left(L value) {
        return new Left<>(value);
    }

    static <L, R> Either<L, R> right(R value) {
        return new Right<>(value);
    }

    // 값 추출
    default boolean isLeft() {
        return this instanceof Left<L, R>;
    }

    default boolean isRight() {
        return this instanceof Right<L, R>;
    }

    default L getLeft() {
        if (this instanceof Left<L, R> left) {
            return left.value();
        }
        throw new NoSuchElementException("Not a Left");
    }

    default R getRight() {
        if (this instanceof Right<L, R> right) {
            return right.value();
        }
        throw new NoSuchElementException("Not a Right");
    }
}
```

**flatMap과 map 구현**

```java
public sealed interface Either<L, R>
    permits Either.Left, Either.Right {

    // ... 위 코드 계속

    // flatMap: Either를 반환하는 함수 체이닝
    default <T> Either<L, T> flatMap(Function<R, Either<L, T>> mapper) {
        if (this instanceof Right<L, R> right) {
            return mapper.apply(right.value());
        }
        return (Either<L, T>) this;
    }

    // map: 일반 함수 체이닝
    default <T> Either<L, T> map(Function<R, T> mapper) {
        if (this instanceof Right<L, R> right) {
            return Either.right(mapper.apply(right.value()));
        }
        return (Either<L, T>) this;
    }

    // mapLeft: 에러 변환
    default <T> Either<T, R> mapLeft(Function<L, T> mapper) {
        if (this instanceof Left<L, R> left) {
            return Either.left(mapper.apply(left.value()));
        }
        return (Either<T, R>) this;
    }

    // fold: 성공/실패 분기
    default <T> T fold(
        Function<L, T> leftMapper,
        Function<R, T> rightMapper
    ) {
        if (this instanceof Left<L, R> left) {
            return leftMapper.apply(left.value());
        }
        if (this instanceof Right<L, R> right) {
            return rightMapper.apply(right.value());
        }
        throw new IllegalStateException("Unreachable");
    }

    // getOrElse: 실패 시 기본값
    default R getOrElse(R defaultValue) {
        if (this instanceof Right<L, R> right) {
            return right.value();
        }
        return defaultValue;
    }

    // getOrElse with supplier
    default R getOrElse(Supplier<R> defaultSupplier) {
        if (this instanceof Right<L, R> right) {
            return right.value();
        }
        return defaultSupplier.get();
    }
}
```

### 비즈니스 에러 정의

**sealed interface로 도메인 에러**

```java
public sealed interface AppError
    permits AppError.ValidationError,
            AppError.DuplicateUserError,
            AppError.DatabaseError,
            AppError.NotFoundError {

    String message();

    record ValidationError(
        String field,
        String reason
    ) implements AppError {
        @Override
        public String message() {
            return "유효성 검사 실패: " + field + " - " + reason;
        }
    }

    record DuplicateUserError(
        String email
    ) implements AppError {
        @Override
        public String message() {
            return "이미 존재하는 이메일: " + email;
        }
    }

    record DatabaseError(
        Throwable cause
    ) implements AppError {
        @Override
        public String message() {
            return "데이터베이스 오류: " + cause.getMessage();
        }
    }

    record NotFoundError(
        String resourceId
    ) implements AppError {
        @Override
        public String message() {
            return "리소스를 찾을 수 없음: " + resourceId;
        }
    }
}
```

### 서비스 계층 구현

**회원가입 예시**

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    // 타입 별칭 대신 제네릭 사용
    public Either<AppError, User> registerUser(String email, String name) {
        return validateInput(email, name)
            .flatMap(ignored -> checkEmailDuplication(email))
            .flatMap(validEmail -> saveUser(validEmail, name));
    }

    private Either<AppError, Void> validateInput(String email, String name) {
        if (!isEmailValid(email)) {
            return Either.left(
                new AppError.ValidationError("email", "유효하지 않은 형식")
            );
        }
        if (name == null || name.isBlank()) {
            return Either.left(
                new AppError.ValidationError("name", "비어있을 수 없음")
            );
        }
        return Either.right(null);
    }

    private Either<AppError, String> checkEmailDuplication(String email) {
        User existing = userRepository.findByEmail(email);
        if (existing != null) {
            return Either.left(new AppError.DuplicateUserError(email));
        }
        return Either.right(email);
    }

    private Either<AppError, User> saveUser(String email, String name) {
        try {
            User user = new User(email, name);
            User saved = userRepository.save(user);
            return Either.right(saved);
        } catch (Exception e) {
            return Either.left(new AppError.DatabaseError(e));
        }
    }

    private boolean isEmailValid(String email) {
        return email != null &&
               email.contains("@") &&
               email.contains(".");
    }
}
```

### 컨트롤러에서 Either 처리

**Java 21+ pattern matching 활용**

```java
@RestController
@RequestMapping("/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @PostMapping("/register")
    public ResponseEntity<?> register(@RequestBody RegisterUserRequest request) {
        Either<AppError, User> result =
            userService.registerUser(request.email(), request.name());

        // Java 21+ pattern matching
        return switch (result) {
            case Either.Right<AppError, User> right ->
                ResponseEntity.ok(UserResponse.from(right.value()));
            case Either.Left<AppError, User> left ->
                handleError(left.value());
        };
    }

    private ResponseEntity<?> handleError(AppError error) {
        return switch (error) {
            case AppError.ValidationError ve ->
                ResponseEntity.badRequest().body(Map.of(
                    "type", "VALIDATION_ERROR",
                    "field", ve.field(),
                    "message", ve.message()
                ));
            case AppError.DuplicateUserError due ->
                ResponseEntity.status(HttpStatus.CONFLICT).body(Map.of(
                    "type", "DUPLICATE_USER",
                    "message", due.message()
                ));
            case AppError.DatabaseError de ->
                ResponseEntity.internalServerError().body(Map.of(
                    "type", "DATABASE_ERROR",
                    "message", "서버 오류가 발생했습니다"
                ));
            case AppError.NotFoundError nfe ->
                ResponseEntity.notFound().build();
        };
    }
}

record RegisterUserRequest(String email, String name) {}

record UserResponse(Long id, String email, String name) {
    static UserResponse from(User user) {
        return new UserResponse(user.getId(), user.getEmail(), user.getName());
    }
}
```

**Java 17 (pattern matching 없이)**

```java
@PostMapping("/register")
public ResponseEntity<?> register(@RequestBody RegisterUserRequest request) {
    Either<AppError, User> result =
        userService.registerUser(request.email(), request.name());

    return result.fold(
        // 실패 처리
        error -> handleError(error),
        // 성공 처리
        user -> ResponseEntity.ok(UserResponse.from(user))
    );
}

private ResponseEntity<?> handleError(AppError error) {
    if (error instanceof AppError.ValidationError ve) {
        return ResponseEntity.badRequest().body(Map.of(
            "type", "VALIDATION_ERROR",
            "field", ve.field(),
            "message", ve.message()
        ));
    }
    if (error instanceof AppError.DuplicateUserError due) {
        return ResponseEntity.status(HttpStatus.CONFLICT).body(Map.of(
            "type", "DUPLICATE_USER",
            "message", due.message()
        ));
    }
    if (error instanceof AppError.DatabaseError de) {
        return ResponseEntity.internalServerError().body(Map.of(
            "type", "DATABASE_ERROR",
            "message", "서버 오류가 발생했습니다"
        ));
    }
    if (error instanceof AppError.NotFoundError nfe) {
        return ResponseEntity.notFound().build();
    }
    return ResponseEntity.internalServerError().build();
}
```

### 복잡한 비즈니스 로직 체이닝

**주문 처리 예시**

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    private final InventoryService inventoryService;
    private final PaymentService paymentService;

    public Either<AppError, Order> createOrder(
        Long userId,
        Long productId,
        int quantity
    ) {
        return validateQuantity(quantity)
            .flatMap(ignored -> findProduct(productId))
            .flatMap(product -> checkInventory(product, quantity))
            .flatMap(product -> reserveInventory(product.getId(), quantity)
                .map(ignored -> product))
            .flatMap(product -> processPayment(userId, product, quantity))
            .flatMap(payment -> saveOrder(userId, productId, quantity, payment));
    }

    private Either<AppError, Void> validateQuantity(int quantity) {
        if (quantity <= 0) {
            return Either.left(
                new AppError.ValidationError("quantity", "양수여야 함")
            );
        }
        return Either.right(null);
    }

    private Either<AppError, Product> findProduct(Long productId) {
        return productRepository.findById(productId)
            .map(Either::<AppError, Product>right)
            .orElse(Either.left(
                new AppError.NotFoundError("product:" + productId)
            ));
    }

    private Either<AppError, Product> checkInventory(
        Product product,
        int quantity
    ) {
        int available = inventoryService.getAvailableQuantity(product.getId());
        if (available < quantity) {
            return Either.left(new AppError.ValidationError(
                "quantity",
                String.format("재고 부족: 요청 %d, 가용 %d", quantity, available)
            ));
        }
        return Either.right(product);
    }

    private Either<AppError, Void> reserveInventory(
        Long productId,
        int quantity
    ) {
        try {
            inventoryService.reserve(productId, quantity);
            return Either.right(null);
        } catch (Exception e) {
            return Either.left(new AppError.DatabaseError(e));
        }
    }

    private Either<AppError, Payment> processPayment(
        Long userId,
        Product product,
        int quantity
    ) {
        long amount = product.getPrice() * quantity;
        try {
            Payment payment = paymentService.charge(userId, amount);
            return Either.right(payment);
        } catch (PaymentFailedException e) {
            // 재고 롤백
            inventoryService.release(product.getId(), quantity);
            return Either.left(
                new AppError.ValidationError("payment", "결제 실패")
            );
        }
    }

    private Either<AppError, Order> saveOrder(
        Long userId,
        Long productId,
        int quantity,
        Payment payment
    ) {
        try {
            Order order = new Order(userId, productId, quantity, payment.getId());
            Order saved = orderRepository.save(order);
            return Either.right(saved);
        } catch (Exception e) {
            // 결제 및 재고 롤백
            paymentService.refund(payment.getId());
            inventoryService.release(productId, quantity);
            return Either.left(new AppError.DatabaseError(e));
        }
    }
}
```

### Result 타입 구현 (Optional 스타일)

Either 외에 성공/실패만 구분하는 Result 타입도 유용하다.

```java
public sealed interface Result<T>
    permits Result.Success, Result.Failure {

    record Success<T>(T value) implements Result<T> {}
    record Failure<T>(Throwable error) implements Result<T> {}

    static <T> Result<T> success(T value) {
        return new Success<>(value);
    }

    static <T> Result<T> failure(Throwable error) {
        return new Failure<>(error);
    }

    static <T> Result<T> of(Supplier<T> supplier) {
        try {
            return success(supplier.get());
        } catch (Exception e) {
            return failure(e);
        }
    }

    default <U> Result<U> flatMap(Function<T, Result<U>> mapper) {
        if (this instanceof Success<T> success) {
            return mapper.apply(success.value());
        }
        return (Result<U>) this;
    }

    default <U> Result<U> map(Function<T, U> mapper) {
        if (this instanceof Success<T> success) {
            return Result.success(mapper.apply(success.value()));
        }
        return (Result<U>) this;
    }

    default <U> U fold(
        Function<Throwable, U> failureMapper,
        Function<T, U> successMapper
    ) {
        if (this instanceof Failure<T> failure) {
            return failureMapper.apply(failure.error());
        }
        if (this instanceof Success<T> success) {
            return successMapper.apply(success.value());
        }
        throw new IllegalStateException("Unreachable");
    }

    default T getOrElse(T defaultValue) {
        if (this instanceof Success<T> success) {
            return success.value();
        }
        return defaultValue;
    }

    default T getOrThrow() {
        if (this instanceof Success<T> success) {
            return success.value();
        }
        if (this instanceof Failure<T> failure) {
            if (failure.error() instanceof RuntimeException re) {
                throw re;
            }
            throw new RuntimeException(failure.error());
        }
        throw new IllegalStateException("Unreachable");
    }
}
```

**Result 사용 예시**

```java
public Result<User> registerUser(String email, String name) {
    return Result.of(() -> validateInput(email, name))
        .flatMap(ignored -> Result.of(() -> checkEmailDuplication(email)))
        .flatMap(validEmail -> Result.of(() -> saveUser(validEmail, name)));
}
```

## 실무 적용

### 기존 코드와의 통합

**점진적 도입**

```java
// 기존 코드 (예외 던지기)
public User registerUserLegacy(String email, String name) {
    validateInput(email, name);
    checkEmailDuplication(email);
    return saveUser(email, name);
}

// 새 코드 (Either 사용)
public Either<AppError, User> registerUser(String email, String name) {
    return validateInput(email, name)
        .flatMap(ignored -> checkEmailDuplication(email))
        .flatMap(validEmail -> saveUser(validEmail, name));
}

// 기존 코드를 호출해야 할 때
public User registerUserCompat(String email, String name) {
    return registerUser(email, name)
        .fold(
            error -> throw new BusinessException(error.message()),
            user -> user
        );
}
```

### 트랜잭션 처리

```java
@Service
@Transactional
public class OrderService {

    // @Transactional이 전체 체인을 감싸므로
    // 중간에 실패하면 자동으로 롤백
    public Either<AppError, Order> createOrder(
        Long userId,
        Long productId,
        int quantity
    ) {
        return validateQuantity(quantity)
            .flatMap(ignored -> findProduct(productId))
            .flatMap(product -> checkInventory(product, quantity))
            .flatMap(product -> reserveInventory(product.getId(), quantity)
                .map(ignored -> product))
            .flatMap(product -> processPayment(userId, product, quantity))
            .flatMap(payment -> saveOrder(userId, productId, quantity, payment));
    }
}
```

### Optional과의 상호운용

```java
// Optional → Either 변환
public Either<AppError, User> findUserById(Long userId) {
    return userRepository.findById(userId)
        .map(Either::<AppError, User>right)
        .orElse(Either.left(
            new AppError.NotFoundError("user:" + userId)
        ));
}

// Either → Optional 변환
public Optional<User> getUserOptional(String email) {
    Either<AppError, User> result = findUserByEmail(email);
    if (result instanceof Either.Right<AppError, User> right) {
        return Optional.of(right.value());
    }
    return Optional.empty();
}
```

### CompletableFuture와 통합

```java
public CompletableFuture<Either<AppError, User>> registerUserAsync(
    String email,
    String name
) {
    return CompletableFuture.supplyAsync(() ->
        registerUser(email, name)
    );
}

// 여러 비동기 작업 체이닝
public CompletableFuture<Either<AppError, Order>> createOrderAsync(
    Long userId,
    Long productId,
    int quantity
) {
    return CompletableFuture.supplyAsync(() -> validateQuantity(quantity))
        .thenCompose(result -> result.fold(
            error -> CompletableFuture.completedFuture(Either.left(error)),
            ignored -> findProductAsync(productId)
        ))
        .thenCompose(result -> result.fold(
            error -> CompletableFuture.completedFuture(Either.left(error)),
            product -> checkInventoryAsync(product, quantity)
        ));
}
```

## 금융권에서의 고려사항

**거래 처리 파이프라인**

```java
@Service
@Transactional
public class TransactionService {

    private final AccountRepository accountRepository;
    private final TransactionRepository transactionRepository;
    private final AuditLogger auditLogger;

    public Either<AppError, Transaction> transfer(
        Long fromAccountId,
        Long toAccountId,
        Long amount
    ) {
        Either<AppError, Transaction> result = validateAmount(amount)
            .flatMap(ignored -> findAccount(fromAccountId))
            .flatMap(from -> checkBalance(from, amount))
            .flatMap(from -> findAccount(toAccountId)
                .map(to -> new AccountPair(from, to)))
            .flatMap(pair -> executeTransfer(pair, amount));

        // 성공/실패 모두 감사 로그 기록
        auditLogger.log(result);

        return result;
    }

    private Either<AppError, Void> validateAmount(Long amount) {
        if (amount == null || amount <= 0) {
            return Either.left(
                new AppError.ValidationError("amount", "양수여야 함")
            );
        }
        if (amount > 10_000_000L) {
            return Either.left(
                new AppError.ValidationError("amount", "1회 한도 초과")
            );
        }
        return Either.right(null);
    }

    private Either<AppError, Account> findAccount(Long accountId) {
        return accountRepository.findById(accountId)
            .map(Either::<AppError, Account>right)
            .orElse(Either.left(
                new AppError.NotFoundError("account:" + accountId)
            ));
    }

    private Either<AppError, Account> checkBalance(
        Account account,
        Long amount
    ) {
        if (account.getBalance() < amount) {
            return Either.left(new AppError.InsufficientBalanceError(
                account.getId(),
                amount,
                account.getBalance()
            ));
        }
        return Either.right(account);
    }

    private Either<AppError, Transaction> executeTransfer(
        AccountPair pair,
        Long amount
    ) {
        try {
            pair.from().withdraw(amount);
            pair.to().deposit(amount);

            accountRepository.save(pair.from());
            accountRepository.save(pair.to());

            Transaction transaction = new Transaction(
                pair.from().getId(),
                pair.to().getId(),
                amount
            );
            Transaction saved = transactionRepository.save(transaction);

            return Either.right(saved);
        } catch (Exception e) {
            return Either.left(new AppError.DatabaseError(e));
        }
    }

    record AccountPair(Account from, Account to) {}
}
```

**도메인 에러 확장**

```java
public sealed interface AppError
    permits AppError.ValidationError,
            AppError.DuplicateUserError,
            AppError.DatabaseError,
            AppError.NotFoundError,
            AppError.InsufficientBalanceError,
            AppError.TransactionLimitError {

    // ... 기존 에러들 ...

    record InsufficientBalanceError(
        Long accountId,
        Long requestedAmount,
        Long currentBalance
    ) implements AppError {
        @Override
        public String message() {
            return String.format(
                "잔액 부족: 계좌 %d, 요청 %d원, 현재 %d원",
                accountId, requestedAmount, currentBalance
            );
        }
    }

    record TransactionLimitError(
        Long accountId,
        Long dailyLimit,
        Long currentSum
    ) implements AppError {
        @Override
        public String message() {
            return String.format(
                "일일 한도 초과: 계좌 %d, 한도 %d원, 현재 %d원",
                accountId, dailyLimit, currentSum
            );
        }
    }
}
```

**에러 로깅 전략**

```java
@Component
public class AuditLogger {

    private final Logger log = LoggerFactory.getLogger(AuditLogger.class);

    public void log(Either<AppError, Transaction> result) {
        result.fold(
            // 실패 로깅
            error -> {
                log.warn("거래 실패: {}", error.message());
                saveAuditLog("FAILURE", error.message());
                return null;
            },
            // 성공 로깅
            transaction -> {
                log.info("거래 성공: {}", transaction.getId());
                saveAuditLog("SUCCESS", "거래 ID: " + transaction.getId());
                return null;
            }
        );
    }

    private void saveAuditLog(String status, String message) {
        // DB에 감사 로그 저장
    }
}
```

---

**참고 자료**
- [JEP 409: Sealed Classes (Java 17)](https://openjdk.org/jeps/409)
- [JEP 395: Records (Java 16)](https://openjdk.org/jeps/395)
- [JEP 441: Pattern Matching for switch (Java 21)](https://openjdk.org/jeps/441)

## 관련 노트

- [[레일웨이-패턴-Kotlin-구현]]
- [[모든 예외는 unchecked이다]]
