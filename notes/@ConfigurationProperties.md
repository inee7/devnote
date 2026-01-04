---
tags: [spring, spring-boot, configuration]
---

# @ConfigurationProperties

## 한 줄 요약

Spring Boot에서 설정 파일(yml/properties)의 값을 타입 안전하게 객체로 바인딩하는 애노테이션

## 핵심 정리

- prefix 기반으로 설정 값을 객체 필드에 자동 매핑
- Bean으로 등록 필요 (`@Component` 또는 `@EnableConfigurationProperties`)
- 복잡한 객체, 리스트, 중첩 구조도 바인딩 가능
- JSR-303 유효성 검증 지원
- Spring Boot 2.2부터 생성자 바인딩 지원, 3.0부터는 `@ConstructorBinding` 생략 가능

## 상세 내용

### 기본 사용법

```java
@Component
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private String name;
    private int version;
    // Getters and Setters
}
```

```yaml
app:
  name: MyApp
  version: 1
```

### Bean 등록 방법

**방법 1: @Component 사용**
```java
@Component
@ConfigurationProperties(prefix = "app")
public class AppProperties { }
```

**방법 2: @EnableConfigurationProperties 사용**
```java
@Configuration
@EnableConfigurationProperties(AppProperties.class)
public class AppConfig { }
```

### 생성자 바인딩

**Spring Boot 2.1 이하**
- 기본 생성자 + setter 방식만 지원

**Spring Boot 2.2 ~ 2.x**
```java
@ConstructorBinding
@ConfigurationProperties("config.person")
@RequiredArgsConstructor
public class Person {
    private final String name;
    private final int age;
}
```

**Spring Boot 3.0 이상**
- `@ConstructorBinding`을 생성자 또는 클래스에 선언
- 생성자가 하나만 있으면 `@ConstructorBinding` 생략 가능 (Lombok `@RequiredArgsConstructor`와 함께 사용 시 편리)

```java
@ConfigurationProperties("config.person")
@RequiredArgsConstructor
public class Person {
    private final String name;
    private final int age;
}
```

### 복잡한 타입 바인딩

```java
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private List<String> servers;
    private Map<String, String> metadata;
}
```

```yaml
app:
  servers:
    - server1.example.com
    - server2.example.com
  metadata:
    env: production
    region: kr
```

### 유효성 검증

```java
@ConfigurationProperties(prefix = "app")
@Validated
public class AppProperties {
    @NotNull
    private String name;

    @Min(1)
    @Max(100)
    private int version;
}
```

설정 파일에 유효하지 않은 값이 있으면 애플리케이션 시작 시 예외 발생

## 실무 적용

### @Value vs @ConfigurationProperties

- **@Value**: 단일 값 주입에 적합, SpEL 표현식 사용 가능
- **@ConfigurationProperties**:
  - 여러 관련 설정을 그룹화할 때 유리
  - 타입 안전성 보장
  - 자동완성 및 IDE 지원 (spring-boot-configuration-processor 의존성 추가 시)
  - 유효성 검증 가능

### 주의사항

- 불변 객체로 만들고 싶다면 생성자 바인딩 + final 필드 조합 사용
- setter 방식은 런타임에 값이 변경될 수 있으므로 불변성이 필요하면 생성자 바인딩 권장
- `spring-boot-configuration-processor` 의존성 추가 시 IDE에서 자동완성 지원

## 관련 노트

- [[스프링부트와 스프링프레임워크 관계]]
- [[@Profile 쓰지 말고 Property로 제어하는 방법]]
