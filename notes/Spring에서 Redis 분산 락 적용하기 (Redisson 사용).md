
## 분산 락(Distributed Lock)이란?

- 다중 스레드 환경에서, 어떤 자원에 대해 동시에 접근 가능한 스레드가 단 한 개만 되도록 강제하는 것을 Lock (혹은 Mutex, 상호 배제)이라고 한다.
- Java 기반 웹 애플리케이션에서 Lock을 구현하는데 가장 쉬운 방법은 `synchronized` 키워드 (Kotlin의 경우 `@Synchronized` 어노테이션)을 사용하는 것이다.
- 그러나, 부하 분산 등을 위해 서버가 다중화된 환경에서는 `synchronized`가 지정된 코드에 대해 다른 서버 간의 배제성이 보장되지 않는다.
    - 따라서, 다중화된 서버 사이에 배제성을 확보하기 위한 공유된 자원을 사용해야 한다.
    - 이는 분산된 자원에 대한 잠금(Lock)이며, "분산 락(Distributed Lock)"이라고 한다.
- 외부 자원을 통한 Lock 구현은 여러 가지 방법이 있지만, DB의 부하를 줄이고 메모리 기반으로 빠른 입출력이 특징인 Redis를 사용하여 적용하는 방법을 기술한다.

## 문제가 되는 상황

- 재고 정보는 물류 센터의 적재된 위치와 수량을 나타낸다.
- 분산 부하를 위한 서버 다중화로 서버는 2개
- A 상품의 총 재고는 5개
    - A 상품이 적재된 1~5번 위치에 각각 1개씩 적재
    - 유통기한이 빠른 순으로 1번 위치부터 5번 위치에 적재
    - 1번 위치부터 5번 위치로 차례대로 출고한다고 가정
- 2개의 서버에 동시에 각각 A 상품 1개 출고가 요청되었을 경우
    - 1번 서버의 상황
        - 재고 정보를 읽으면, 1번 위치부터 5번 위치까지 1개씩 있고, 1번 위치의 재고부터 나가야 하므로 1번 위치 재고 1개 할당
    - 2번 서버의 상황
        - 상호 배제가 되지 않으므로, 재고 정보를 읽을 경우 1번 서버와 같은 정보를 읽게 됨 -> 마찬가지로 1번 위치 재고 1개 할당
- 요청 처리 후 상황
    - 결과적으로 1번 위치에 2개 재고가 할당되어 1번 위치의 재고는 -1, 2~5번 위치는 재고 각각 1개
    - 재고 총합은 3이지만 잘못된 위치의 재고 할당으로 마이너스 재고 유발
- 옳게 처리가 되려면,
    - 1번 서버든 2번 서버든, 하나의 서버가 재고 정보를 한 서버씩 접근하여, 1번 위치 재고 0, 2번 위치 재고 0, 3~5번 위치는 재고 1개로 나타나야 함.

## 서버 환경

- SpringBoot 3.0.2, Kotlin 1.7.22 
- EKS(k8s)

## 구현

### Dependency 추가

```kotlin
    implementation("org.redisson:redisson-spring-boot-starter:3.21.1")
```

- redisson dependency 안에 spring-data-redis가 포함되어 있음.

#### 왜 spring-data-redis의 기본 구현체인 Lettuce가 아닌 Redisson를 사용하는가

- Lettuce
    - spring-data-redis의 기본 구현체
    - 기본적으로 Spin Lock을 사용한다.
        - 이는 Lock을 대기하는 상황에서, Lock을 획득할 수 있는지 계속 요청을 보낸다.
        - 따라서 Lock을 획득하려는 스레드가 많을 경우 Redis에 부하가 집중된다.
    - Lock에 대한 타임아웃이 없어, Unlock(잠금 해제) 호출을 하지 못한 경우 Dead Lock을 유발할 수 있다.
- Redisson
    - pub/sub 방식을 사용한다.
        - Lock을 당장 획득할 수 없으면 대기한다.
        - Lock이 획득 가능할 경우 Redis에서 클라이언트로 획득 가능함을 알린다.
    - Lock의 lease time 설정이 가능하다.
        - 즉, 설정된 lease time이 지난 경우 자동으로 Lock의 소유권을 회수하여 Dead Lock을 방지한다.

### Redis 설정

```kotlin
@Configuration
@ConfigurationProperties(prefix = RedisConfigConst.PROPERTIES_PREFIX) // "spring.data.redis"
class RedisConfig {

    lateinit var host: String
    var port: Int = 0

    @Bean(destroyMethod = "shutdown")
    fun redissonClient(): RedissonClient {
        val config: Config = Config()
        config.useSingleServer()
            .setAddress("redis://$host:$port")
            .setDnsMonitoringInterval(-1)
        return Redisson.create(config)
    }

    @Bean
    fun redisConnectionFactory(redissonClient: RedissonClient): RedisConnectionFactory {
        return RedissonConnectionFactory(redissonClient)
    }
}
```

- host가 하나인 Redis를 사용하므로 SingleServer를 사용했다.
- `setDnsMonitorinInterval(-1)` : Redisson에 포함된 DNSMonitor 라는 객체에서 주기적으로 DNS를 체크하는데, 현재 버전에서는 정상적으로 연결되어 작동함에도 `io.netty.resolver.dns.DnsNameResolverTimeoutException ... query via UDP timed out after 5000 milliseconds`이 계속 발생하는 버그가 있어 해당 DNS 체크를 하지 않도록 설정 (-1 = 비활성화)

### RedisLockService 구현

```kotlin
@Service
class RedisLockService(
    private val redissonClient: RedissonClient
) {

    @Value("\${lock.wait-time}")
    var waitTime: Long = 0

    @Value("\${lock.lease-time}")
    var leaseTime: Long = 0

    fun <R> tryLockWith(
        lockName: String,
        task: () -> R,
    ): R = tryLockWith(
        lockName = lockName,
        waitTime = waitTime,
        leaseTime = leaseTime,
        task = task
    )

    fun <R> tryLockWith(
        lockName: String,
        waitTime: Long,
        leaseTime: Long,
        task: () -> R,
    ): R {
        val rLock: RLock = redissonClient.getLock(RedisConfigConst.LOCK_PREFIX + lockName) // Lock 호출
        val available: Boolean = rLock.tryLock(waitTime, leaseTime, TimeUnit.SECONDS) // Lock 획득 시도
        if (!available) { // 획득 시도를 실패했을 경우 Exception 처리
            throw RedisLockTimeoutException(REDIS_LOCK_WAIT_TIMEOUT_EXCEPTION)
        }
        try {
            return task() // 전달 받은 람다 실행
        } finally {
            if (rLock.isHeldByCurrentThread) { // 해당 스레드가 Lock을 소유 중인지 확인
                rLock.unlock() // Lock 반환
            } else { // 스레드가 Lock을 소유 중이지 않을 경우, Exception (leaseTime을 넘은 경우)
                throw RedisLockTimeoutException(REDIS_LOCK_FORCE_LEASED_EXCEPTION)
            }
        }
    }
}
```

- timeout 관련 설정은 운영 환경에서 변경의 여지가 있으므로 프로퍼티로 주입받도록 설정
    - waitTime을 leaseTime보다 길게 설정하여, timeout Exception이 자주 발생하지 않도록 설계 
- 제네릭과 람다를 사용하여, 분산 락을 설정할 코드를 전달받음.
- `rLock.tryLock(waitTime, leaseTime, TimeUnit.SECONDS)`
    - waitTime은 Lock 획득을 위해 기다리는 시간을 설정한다.
    - leaseTime은 Lock 획득한 후 설정한 시간이 넘어가면 자동으로 Lock 소유권이 회수된다.
        - 이 설정을 통해 Dead Lock을 방지한다.
- tryLock 결과 false인 경우, wait timeout이 발생했음을 Exception 처리
- `rLock.isHeldByCurrentThread`
    - 해당 값을 통해 현재 스레드가 락을 소유하고 있는지 확인이 가능하다.
    - 체크를 하는 이유는, 현재 스레드가 Lock을 소유하지 않은 채로 `rLock.unlock()`를 호출하면 Exception이 발생한다.
        - waitTime 과 leaseTime 중 무엇을 초과했는지 구분하는 것이 필요 없다면, if문을 모두 제거하여도 상관없다.
- `rLock.unlock()`
    - Lock을 반환한다.
    - 현재 스레드에서 Lock 소유권이 없을 때 호출되면 Exception을 발생시킨다.
    - 특히, 트랜잭션과 연관된 경우 잘못된 사용으로 원치 않은 롤백이 발생할 수 있으므로 조심해야 한다.

### 클라이언트 코드

```null
val result: List<ResultDto> = redisLockService.tryLockWith(lockName = model.customCode)
        { inventoryAssignService.assignInventory(model, requestDto) }
```

출저 : https://velog.io/@profoundsea25/Spring에서-Redis-분산-락-적용하기-Redisson-사용

#spring #redis
