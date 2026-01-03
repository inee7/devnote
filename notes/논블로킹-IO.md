---
tags: [논블로킹IO, 이벤트루프, 리액터, 성능최적화, 백엔드]
---

## 한 줄 요약

논블로킹 I/O는 소수의 스레드로 I/O 대기 없이 다음 작업을 수행하여 높은 동시성을 달성한다.

## 핵심 정리

- 경량 스레드(Virtual Thread, Coroutine)도 결국 한계가 옴
- 논블로킹 I/O는 입출력 대기 없이 다음 작업 수행
- 소수의 스레드로 대량의 요청 처리 가능
- 채널을 CPU 개수만큼 만들어 스레드 할당
- 리액터(Reactor) = 이벤트 루프(Event Loop)

## 상세 내용

### 경량 스레드의 한계

#### Virtual Thread / Coroutine

Java의 Virtual Thread, Kotlin의 Coroutine은 경량 스레드로 많은 동시 작업을 처리할 수 있습니다:

```kotlin
// Kotlin Coroutine 예시
suspend fun handleRequest() {
    val result = withContext(Dispatchers.IO) {
        externalApi.call()  // I/O 작업
    }
    processResult(result)
}
```

장점:
- OS 스레드보다 훨씬 가벼움
- 수천~수만 개의 경량 스레드 생성 가능
- 컨텍스트 스위칭 비용 낮음

하지만 한계:
- 경량 스레드도 많아지면 메모리 사용량 증가
- 스케줄링 오버헤드 발생
- I/O 대기 중에는 결국 스레드가 블로킹됨 (내부적으로)

### 논블로킹 I/O의 필요성

**블로킹 I/O의 문제**

```kotlin
fun handleRequest() {
    val data = database.query()  // 블로킹: DB 응답 대기
    val result = externalApi.call(data)  // 블로킹: API 응답 대기
    return result
}
```

문제:
- I/O 작업 중 스레드가 대기 상태
- 대기 중인 스레드는 다른 작업 처리 불가
- 많은 요청 처리하려면 많은 스레드 필요

**논블로킹 I/O의 해결**

```kotlin
// 논블로킹 방식 (개념적 예시)
fun handleRequestNonBlocking() {
    database.queryAsync { data ->
        externalApi.callAsync(data) { result ->
            sendResponse(result)
        }
    }
    // I/O 대기 없이 즉시 리턴
    // 다음 요청 처리 가능
}
```

장점:
- I/O 작업을 시작하고 즉시 다음 작업으로 이동
- I/O 완료 시 콜백으로 후속 처리
- 소수의 스레드로 대량 요청 처리

### 이벤트 루프 (Event Loop) / 리액터 (Reactor)

#### 동작 원리

```
이벤트 루프 (단일 스레드)
    ↓
[요청1] → I/O 시작 → 콜백 등록
[요청2] → I/O 시작 → 콜백 등록
[요청3] → I/O 시작 → 콜백 등록
    ↓
I/O 완료 감지
    ↓
콜백 실행 → 응답 전송
```

핵심:
- 하나의 이벤트 루프가 여러 I/O 작업 동시 관리
- I/O 대기 중에도 다른 작업 처리
- I/O 완료되면 콜백 실행

#### 채널과 스레드 할당

**효율적인 동시성 처리**

```
CPU 4코어인 경우:

채널 1 (스레드 1) → 이벤트 루프
채널 2 (스레드 2) → 이벤트 루프
채널 3 (스레드 3) → 이벤트 루프
채널 4 (스레드 4) → 이벤트 루프
```

원칙:
- 채널 수 = CPU 코어 수
- 각 채널에 스레드 하나 할당
- 각 스레드가 이벤트 루프 실행
- CPU를 최대한 활용하면서 컨텍스트 스위칭 최소화

### 논블로킹 I/O 프레임워크

#### Spring WebFlux (Project Reactor)

```kotlin
@RestController
class UserController(
    private val userService: UserService
) {
    @GetMapping("/users/{id}")
    fun getUser(@PathVariable id: Long): Mono<User> {
        return userService.findById(id)  // 논블로킹
    }

    @GetMapping("/users")
    fun getUsers(): Flux<User> {
        return userService.findAll()  // 논블로킹 스트림
    }
}
```

특징:
- `Mono<T>`: 0~1개 결과 (비동기)
- `Flux<T>`: 0~N개 결과 스트림 (비동기)
- Netty 기반 이벤트 루프 사용
- 소수의 스레드로 높은 처리량

#### Ktor (Kotlin 비동기 프레임워크)

```kotlin
fun Application.module() {
    routing {
        get("/users/{id}") {
            val id = call.parameters["id"]?.toLong()
                ?: throw BadRequestException("Invalid ID")

            val user = userService.findById(id)  // suspend 함수
            call.respond(user)
        }
    }
}
```

특징:
- Kotlin Coroutine 기반
- suspend 함수로 논블로킹 I/O
- 경량하고 직관적인 API

### 언제 논블로킹 I/O를 사용할까?

**논블로킹 I/O가 효과적인 경우**
- I/O 대기 시간이 긴 작업 (외부 API, DB 조회)
- 동시 연결 수가 매우 많은 경우 (수만~수십만)
- CPU 사용률이 낮고 I/O가 병목인 경우

**블로킹 I/O가 나은 경우**
- CPU 집약적 작업 (계산, 암호화 등)
- 코드 복잡도를 낮추고 싶은 경우
- 팀의 비동기 프로그래밍 경험이 부족한 경우

**실무 권장 접근**
1. 먼저 경량 스레드 사용 (Virtual Thread, Coroutine)
2. 성능 이슈가 발생하면 논블로킹 I/O 고려
3. 부분적으로 논블로킹 적용 (예: 외부 API 호출만)

## 실무 적용

### 도입 체크리스트

- [ ] I/O 대기 시간이 전체 응답 시간의 대부분인가?
- [ ] 동시 연결 수가 수만 개 이상인가?
- [ ] 팀이 비동기 프로그래밍에 익숙한가?
- [ ] 기존 블로킹 라이브러리를 논블로킹으로 교체할 수 있는가?

### 주의사항

**블로킹 코드 혼재 금지**

```kotlin
// 잘못된 예: WebFlux에서 블로킹 호출
@GetMapping("/users/{id}")
fun getUser(@PathVariable id: Long): Mono<User> {
    return Mono.fromCallable {
        jdbcTemplate.queryForObject(...)  // 블로킹!
        // 이벤트 루프 스레드가 블로킹됨 → 전체 성능 저하
    }
}

// 올바른 예: 블로킹 작업은 별도 스레드 풀에서
@GetMapping("/users/{id}")
fun getUser(@PathVariable id: Long): Mono<User> {
    return Mono.fromCallable {
        jdbcTemplate.queryForObject(...)
    }.subscribeOn(Schedulers.boundedElastic())  // 블로킹 전용 스레드 풀
}
```

**디버깅의 어려움**
- 비동기 스택 트레이스는 읽기 어려움
- 로깅과 모니터링 강화 필요
- Reactor나 Coroutine의 디버깅 도구 활용

### 금융권에서의 고려사항

- 논블로킹 I/O는 코드 복잡도를 높이므로 신중히 도입
- 핵심 비즈니스 로직은 동기 방식 유지하고, 부하가 큰 부분만 비동기 처리
- 기존 블로킹 인프라(DB 드라이버 등)와의 호환성 확인
- 예외 처리와 트랜잭션 경계를 명확히 설계

---

**출처**
- 주니어 백엔드 개발자가 반드시 알아야 할 실무 지식 (최범균)
