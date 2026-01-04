---
tags: [spring, configuration, best-practice, conditional]
---

# @Profile 쓰지 말고 Property로 제어하는 방법

## 한 줄 요약

`@Profile`은 환경별 Bean 설정을 한눈에 파악하기 어려우므로, `@ConditionalOnProperty`를 사용하여 명시적인 프로퍼티 기반 제어를 권장

## 핵심 정리

- `@Profile`의 문제점: 전체 검색 필요, 부정 조건(`!test`) 파악 어려움
- `@ConditionalOnProperty`: 환경별 설정을 yml에 명시적으로 정의
- 가독성과 유지보수성 향상
- 설정 변경 시 코드 수정 없이 프로퍼티만 변경 가능

## 상세 내용

### @Profile의 문제점

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Profile("dev")
    public DataSource devDataSource() { }

    @Bean
    @Profile("test")
    public DataSource testDataSource() { }

    @Bean
    @Profile("!test")  // ❌ 부정 조건은 찾기 어려움
    public DataSource prodDataSource() { }
}
```

**문제점:**
1. **전체 검색 필요**: 특정 프로파일에 어떤 Bean이 등록되는지 파악하려면 프로젝트 전체를 검색해야 함
2. **부정 조건 파악 어려움**: `@Profile("!test")`처럼 부정 조건은 IDE 검색으로 찾기 힘듦
3. **조합 복잡도**: `@Profile({"dev", "qa"})`, `@Profile("dev & mysql")` 등 복잡한 조합 시 가독성 저하
4. **명시성 부족**: yml 파일만 봐서는 어떤 Bean이 활성화되는지 알 수 없음

### 권장 방식: @ConditionalOnProperty 사용

```yaml
# application-dev.yml
service:
  mock: true
  cache:
    enabled: false

# application-prod.yml
service:
  mock: false
  cache:
    enabled: true
```

```java
@Configuration
public class ServiceConfig {

    @Bean
    @ConditionalOnProperty(name = "service.mock", havingValue = "true")
    public Service mockService() {
        return new MockService();
    }

    @Bean
    @ConditionalOnProperty(name = "service.mock", havingValue = "false")
    public Service realService() {
        return new RealService();
    }

    @Bean
    @ConditionalOnProperty(name = "service.cache.enabled", havingValue = "true")
    public CacheManager cacheManager() {
        return new RedisCacheManager();
    }
}
```

**장점:**
1. **명시적**: yml 파일만 보고도 어떤 Bean이 생성될지 예측 가능
2. **검색 용이**: 프로퍼티 키로 검색하면 관련 Bean을 모두 찾을 수 있음
3. **중앙 관리**: 환경별 설정이 yml 파일에 명확히 정의됨
4. **테스트 용이**: 테스트 시 프로퍼티만 오버라이드하면 됨

### matchIfMissing 옵션 활용

```java
@Bean
@ConditionalOnProperty(
    name = "service.mock",
    havingValue = "false",
    matchIfMissing = true  // 프로퍼티가 없으면 기본값으로 이 Bean 생성
)
public Service realService() {
    return new RealService();
}
```

**사용 시나리오:**
- Production 환경이 기본일 때, 프로퍼티가 명시되지 않으면 실제 서비스 Bean을 생성
- Mock은 명시적으로 설정해야만 활성화됨

## 실무 적용

### 복잡한 조건 처리

```yaml
# application.yml
external:
  api:
    enabled: true
    mock: false
    retry:
      enabled: true
      max-attempts: 3
```

```java
@Configuration
public class ExternalApiConfig {

    @Bean
    @ConditionalOnProperty(name = "external.api.enabled", havingValue = "true")
    @ConditionalOnProperty(name = "external.api.mock", havingValue = "false", matchIfMissing = true)
    public ExternalApiClient realApiClient() {
        return new RealExternalApiClient();
    }

    @Bean
    @ConditionalOnProperty(name = "external.api.enabled", havingValue = "true")
    @ConditionalOnProperty(name = "external.api.mock", havingValue = "true")
    public ExternalApiClient mockApiClient() {
        return new MockExternalApiClient();
    }
}
```

### 테스트에서의 활용

```java
@SpringBootTest
@TestPropertySource(properties = {
    "service.mock=true",
    "service.cache.enabled=false"
})
class ServiceTest {
    @Autowired
    Service service;  // mockService가 주입됨

    @Test
    void test() {
        assertInstanceOf(MockService.class, service);
    }
}
```

### 언제 @Profile을 써도 괜찮은가?

다음 경우에는 `@Profile` 사용도 합리적:

1. **간단한 환경 분리**: dev/prod 두 개뿐이고 설정이 단순할 때
2. **Spring Boot 자동 구성 활용**: `spring.profiles.active`로 전체 환경을 한 번에 전환
3. **레거시 프로젝트**: 이미 @Profile로 구성되어 있고 변경 비용이 클 때

## 관련 참고 자료

[Don't Use Spring @Profile Annotation](https://reflectoring.io/dont-use-spring-profile-annotation/)

## 관련 노트

- [[@ConfigurationProperties]]
- [[스프링부트와 스프링프레임워크 관계]]

