---
tags: [jackson, json, serialization, deserialization, java, kotlin, spring-boot, lombok]
---

## 한 줄 요약

Jackson은 Java/Kotlin 객체와 JSON 간 변환을 담당하는 라이브러리로, 직렬화는 getter 기준, 역직렬화는 생성자+setter 기준으로 동작하며, Kotlin/Lombok/Spring Boot 환경에 따라 추가 설정이 필요하다.

## 핵심 정리

- **직렬화**(객체 → JSON): public getter 또는 public 필드 기준
- **역직렬화**(JSON → 객체): 기본 생성자 + setter 또는 생성자 기반 매핑
- Java: Lombok 사용 시 @Data는 문제 없으나, @Builder/@Value는 추가 설정 필요
- Kotlin: data class는 기본 지원, isXXX 프로퍼티는 jackson-module-kotlin 2.10.1부터 스펙 변경
- Spring Boot: JavaTimeModule, null 값 제외, FAIL_ON_UNKNOWN_PROPERTIES=false가 기본 설정
- 불변 객체 역직렬화: `-parameters` 컴파일 옵션 + jackson-module-parameter-names 필요
- 어노테이션: @JsonProperty, @JsonIgnore, @JsonInclude, @JsonCreator로 세밀한 제어 가능

## 상세 내용

### Jackson 기본 동작 원리

#### 직렬화 (객체 → JSON)

Jackson은 **getter / public 필드**를 직렬화 대상으로 삼습니다.

**대상**:
- **public getter 메서드**
  - `getXxx()` 또는 `isXxx()` (boolean)
  - JSON key로 출력됨
  - `@JsonProperty("alias")`로 이름 변경 가능
  - `@JsonIgnore`로 제외 가능

- **public 필드**
  - getter 없어도 출력됨 (단, `MapperFeature.AUTO_DETECT_FIELDS` 영향)

- **private 필드**
  - 출력되지 않음
  - `@JsonProperty` 붙이면 강제 출력 가능

#### 역직렬화 (JSON → 객체)

Jackson은 기본적으로 **기본 생성자 + setter** 방식으로 객체를 만듭니다.

**생성자 선택 우선순위**:
1. `@JsonCreator`가 붙은 생성자
2. no-arg constructor (객체 생성 후 setter 호출)
3. record / 불변 객체 → 생성자 기반 사용

**프로퍼티 매핑**:
- JSON key ↔ setter (`setXxx()`)
- JSON key ↔ public 필드
- `@JsonProperty("alias")`로 매핑 이름 지정

### Java에서의 Jackson 사용

#### 기본 예시

```java
public class User {
    private Long id;
    private String name;

    public Long getId() { return id; }
    public String getName() { return name; }
}
```

출력:
```json
{ "id": 1, "name": "친구" }
```

#### 기본 생성자 없이 역직렬화 가능한 경우

**불변 객체 (final 필드 + 생성자)**

```java
public class User {
    private final Long id;
    private final String name;

    public User(Long id, String name) { // no-arg 없음
        this.id = id;
        this.name = name;
    }

    public Long getId() { return id; }
    public String getName() { return name; }
}
```

→ `new User(1, "친구")` 호출 성공 (단, `-parameters` 옵션 필요)

**지원 방법**:
- `-parameters` 옵션 + `jackson-module-parameter-names` 필요
- `@JsonCreator + @JsonProperty` (옵션 없이도 가능, 명시적 매핑)
- Java Record (JDK 16+, Jackson 2.12+에서 자동 지원)

#### -parameters 컴파일 옵션

생성자 기반 역직렬화에서 JSON key와 파라미터 이름을 매핑하려면 **컴파일 시 파라미터 이름을 바이트코드에 포함**해야 합니다.

**Maven**:
```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <version>3.11.0</version>
  <configuration>
    <source>17</source>
    <target>17</target>
    <compilerArgs>
      <arg>-parameters</arg>
    </compilerArgs>
  </configuration>
</plugin>
```

**Gradle**:
```kotlin
tasks.withType<JavaCompile> {
    options.compilerArgs.add("-parameters")
}
```

**Jackson 모듈**:
```xml
<dependency>
  <groupId>com.fasterxml.jackson.module</groupId>
  <artifactId>jackson-module-parameter-names</artifactId>
  <version>2.17.2</version>
</dependency>
```

등록:
```java
ObjectMapper mapper = JsonMapper.builder()
    .addModule(new ParameterNamesModule())
    .build();
```

### Lombok과 Jackson

#### @Data / @Getter / @Setter

- getter/setter 자동 생성 → 직렬화/역직렬화 잘 동작
- 단, **기본 생성자 필요** → 없으면 `@NoArgsConstructor` 추가

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class User {
    private Long id;
    private String name;
}
```

#### @Builder

- Builder만 있으면 역직렬화 불가 → Jackson이 setter/생성자를 못 찾음
- 해결: `@JsonDeserialize(builder=...)` + `@JsonPOJOBuilder` 필요

```java
@Data
@Builder
@JsonDeserialize(builder = User.UserBuilder.class)
public class User {
    private Long id;
    private String name;

    @JsonPOJOBuilder(withPrefix = "")
    public static class UserBuilder {}
}
```

#### @Value (불변 객체)

- 모든 필드 `private final`, setter 없음
- 생성자 기반 역직렬화 필요 → `@JsonCreator` 지정 권장

```java
@Value
@AllArgsConstructor(onConstructor=@__(@JsonCreator))
public class User {
    Long id;
    String name;
}
```

#### @NoArgsConstructor(access = PROTECTED)

- JPA 엔티티에서 자주 사용
- Jackson도 호출 가능 → 문제 없음

### Kotlin에서의 Jackson 사용

#### data class 기본 동작

Kotlin `data class`는 자동으로 **`val`/`var` 프로퍼티마다 public getter**가 생성되므로 대부분 JSON에 출력됩니다.

```kotlin
data class User(
    val id: Long,     // JSON 출력 O
    var name: String  // JSON 출력 O
)
```

출력:
```json
{ "id": 1, "name": "친구" }
```

#### val / var

- **val (불변)** → getter만 존재 → JSON에 포함
- **var (가변)** → getter + setter → JSON에 포함

#### private val / private var

- getter/setter가 **생성되지 않으므로** JSON 출력에서 제외됨
- 출력하고 싶다면 `@get:JsonProperty`를 붙여야 함

```kotlin
data class User(
    val id: Long,
    private val secret: String
)
```

출력:
```json
{ "id": 1 }
```

출력 강제:
```kotlin
data class User(
    val id: Long,
    @get:JsonProperty("secret")
    private val secret: String
)
```

출력:
```json
{ "id": 1, "secret": "mypw" }
```

#### 본문 프로퍼티 (생성자 외부 정의)

- 생성자와 관계없이 **getter가 public이면 직렬화 대상**
- `val`/`var` 둘 다 동일하게 적용

```kotlin
class User(val id: Long) {
    val createdAt: String = "2025-08-28"
}
```

출력:
```json
{ "id": 1, "createdAt": "2025-08-28" }
```

#### lateinit var

- getter는 있으므로 JSON에 포함됨
- 값이 초기화되지 않은 상태에서 직렬화하면 NPE 발생 가능 → 주의 필요

#### isXXX 프로퍼티 주의사항 (jackson-module-kotlin 2.10.1 스펙 변경)

**변경 전 (2.10.0까지)**:

```kotlin
data class Person(
    val isDeveloper: Boolean
)
```

직렬화 결과:
```json
{"developer": true}
```

`is`를 제거한 key로 serialize 했습니다.

**변경 후 (2.10.1부터)**:

```json
{"isDeveloper": true}
```

`is`를 포함하게 변경되었습니다.

**영향**:
- Spring Boot 버전 업그레이드 시 API 스펙이 변경될 여지가 있음
- API 응답 DTO에 isXXX 프로퍼티가 존재한다면 주의 필요

**호환성 유지 방법**:

기존 API 스펙을 유지하려면 `@JsonProperty`를 사용해 명시적으로 key를 작성:

```kotlin
data class Person(
    @JsonProperty("developer")
    val isDeveloper: Boolean
)
```

**배경**:
- serialize/deserialize 간에 이슈가 있었고, Java에서는 `is`가 포함돼서 serialize 되고 있었기 때문에 스펙 변경이라기보다는 **버그 픽스**에 가까움
- 관련 이슈: https://github.com/FasterXML/jackson-module-kotlin/issues/80

### 어노테이션 제어

#### @JsonIgnore

해당 필드/프로퍼티 직렬화 제외

#### @JsonProperty("alias")

JSON key 이름 변경

#### @JsonInclude

값이 `null`이거나 기본값이면 제외 가능

```kotlin
data class User(
    val id: Long,
    @JsonIgnore
    val password: String,
    @JsonProperty("nick")
    val nickname: String,
    @JsonInclude(JsonInclude.Include.NON_NULL)
    val email: String? = null
)
```

출력:
```json
{
  "id": 1,
  "nick": "친구"
  // password 제외, email이 null이므로 제외
}
```

### Spring Boot에서의 Jackson 설정

Spring Boot는 `spring-boot-starter-json`에 의해 Jackson을 포함하고 있으며, 이는 `spring-boot-starter-web`의 일부로 포함됩니다. 따라서 별도의 추가 설정 없이도 Jackson을 통해 JSON 처리가 가능합니다.

#### JacksonAutoConfiguration

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(ObjectMapper.class)
@AutoConfigureAfter(GsonAutoConfiguration.class)
@Import(JacksonAutoConfiguration.JacksonObjectMapperConfiguration.class)
public class JacksonAutoConfiguration {

    // 자동 설정된 ObjectMapper를 정의하고 빈으로 등록
    @Bean
    @ConditionalOnMissingBean
    public ObjectMapper jacksonObjectMapper(Jackson2ObjectMapperBuilder builder) {
        return builder.createXmlMapper(false).build();
    }

    @Configuration(proxyBeanMethods = false)
    @ConditionalOnClass(ObjectMapper.class)
    @Import(JacksonAutoConfiguration.StandardJacksonConfiguration.class)
    static class JacksonObjectMapperConfiguration {
    }

    @Configuration(proxyBeanMethods = false)
    static class StandardJacksonConfiguration {

        @Bean
        @ConditionalOnMissingBean
        public Jackson2ObjectMapperBuilderCustomizer standardJacksonObjectMapperBuilderCustomizer() {
            return builder -> {
                builder.modulesToInstall(new JavaTimeModule());
                builder.featuresToDisable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
            };
        }

    }

}
```

#### 기본 설정

**날짜와 시간**:
- Java 8 날짜 및 시간 API (LocalDate, LocalDateTime)를 지원하기 위해 `JavaTimeModule`을 자동으로 등록
- 기본적으로 날짜와 시간은 **ISO-8601 문자열 형식**으로 직렬화됨

**NULL 값 처리**:
- 기본적으로 null 값은 직렬화되지 않음
- 필요한 경우 null 값을 포함하도록 설정 가능

**기타 설정**:
- `FAIL_ON_UNKNOWN_PROPERTIES = false`: JSON에 매핑되지 않는 필드가 있어도 무시
- `INDENT_OUTPUT = false`: JSON 출력이 기본적으로 포맷팅되지 않음

#### 커스텀 설정

**ObjectMapper 전역 설정**:

```kotlin
val mapper = ObjectMapper()
    .registerModule(KotlinModule())
    .setSerializationInclusion(JsonInclude.Include.NON_NULL) // null 제외
    .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS) // 날짜 포맷 변경
```

**application.yml (Spring Boot 2.5+)**:

Spring Boot 2.5부터 yml에서 ObjectMapper global custom이 가능합니다.

```yaml
spring:
  jackson:
    serialization:
      write-dates-as-timestamps: false
      indent-output: true
    deserialization:
      fail-on-unknown-properties: false
    default-property-inclusion: non_null
```

## 실무 적용

### 직렬화/역직렬화 체크리스트

- [ ] 직렬화는 getter 기준, 역직렬화는 생성자+setter 기준임을 이해했는가?
- [ ] Kotlin data class의 isXXX 프로퍼티 스펙 변경(2.10.1)을 인지하고 있는가?
- [ ] 불변 객체 사용 시 `-parameters` 옵션과 jackson-module-parameter-names를 설정했는가?
- [ ] Lombok @Builder/@Value 사용 시 추가 설정을 적용했는가?
- [ ] Spring Boot의 기본 Jackson 설정(JavaTimeModule, null 제외 등)을 활용하고 있는가?

### Java 환경

**권장 패턴**:
- 일반 DTO: `@Data + @NoArgsConstructor + @AllArgsConstructor`
- 불변 DTO: `@Value + @JsonCreator` 또는 Java Record (JDK 16+)
- Builder 패턴: `@Builder + @JsonDeserialize(builder=...)`

**주의사항**:
- Lombok @Data는 기본 생성자가 필요하면 `@NoArgsConstructor` 명시
- JPA 엔티티는 `@NoArgsConstructor(access = PROTECTED)` 사용

### Kotlin 환경

**권장 패턴**:
- DTO: data class (기본 지원)
- 불변성: val 사용
- null 처리: nullable 타입 + @JsonInclude

**주의사항**:
- isXXX 프로퍼티는 @JsonProperty로 명시적 key 지정 권장
- private 필드는 기본적으로 직렬화 제외됨
- lateinit var은 초기화되지 않은 상태에서 직렬화 시 NPE 발생 가능

### Spring Boot 환경

**기본 설정 활용**:
- JavaTimeModule로 LocalDateTime 자동 처리
- null 값 자동 제외
- 알 수 없는 필드 무시

**커스텀 필요 시**:
- application.yml에서 전역 설정 (Spring Boot 2.5+)
- ObjectMapper 빈 커스터마이징
- @JsonComponent로 특정 타입 커스텀 직렬화/역직렬화

## 금융권에서의 고려사항

- **API 스펙 안정성**: isXXX 프로퍼티 스펙 변경처럼 라이브러리 버전 업그레이드 시 API 스펙 변경 주의
- **날짜/시간 처리**: ISO-8601 형식 사용, 타임존 명시 (UTC 권장)
- **금액 필드**: BigDecimal 사용, 직렬화 시 문자열로 처리하여 정밀도 손실 방지
- **민감 정보**: @JsonIgnore로 password, 주민번호 등 제외, 로그에 민감 정보 노출 방지
- **null 처리**: null 허용 여부를 명확히 정의, @JsonInclude로 제어
- **역직렬화 검증**: JSR-303 (@Valid)와 함께 사용하여 입력 검증
- **에러 처리**: FAIL_ON_UNKNOWN_PROPERTIES 설정 조정 (보안 vs 호환성 트레이드오프)
- **로깅**: ObjectMapper 커스터마이징으로 민감 정보 마스킹
- **버전 관리**: jackson-module-kotlin, jackson-module-parameter-names 버전 통일

### 요약 테이블

| 구분 | 직렬화 (객체 → JSON) | 역직렬화 (JSON → 객체) |
|------|----------------------|------------------------|
| **Java public getter** | JSON에 포함 | setter 필요 |
| **Java public 필드** | JSON에 포함 | setter 없이 매핑 가능 |
| **Java private 필드** | 출력 X (단, @JsonProperty로 가능) | 매핑 불가 |
| **Java final 필드** | getter 있으면 출력 | 생성자 필요 |
| **Java 기본 생성자 없음** | 직렬화 OK | 생성자 기반 매핑 (조건 충족 시) |
| **Kotlin val** | getter 통해 출력됨 | 생성자에서 주입 |
| **Kotlin var** | getter 통해 출력됨 | 생성자에서 주입 (추가로 setter도 가능) |
| **Kotlin private val/var** | 기본적으로 출력 안 됨 (@get:JsonProperty 필요) | 생성자에서는 주입 가능, setter 불가 |
| **Kotlin lateinit var** | 초기화 후 getter 통해 출력됨 | setter로 주입 가능 |
| **Lombok @Data** | 직렬화/역직렬화 가능 | no-arg 필요 |
| **Lombok @Builder** | 직렬화 O | 역직렬화 불가 (설정 필요) |
| **Lombok @Value** | 직렬화 O | 생성자 기반 매핑 필요 |
| **Java Record** | 직렬화 O | 자동 매핑 |

---

**출처**
- [Jackson Documentation](https://github.com/FasterXML/jackson)
- [jackson-module-kotlin isXXX 프로퍼티 스펙 변경](https://multifrontgarden.tistory.com/269)
- [Spring Boot Jackson Auto Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.json)

## 관련 노트

- [[kotlin_data_class 역직렬화]]
- [[spring-boot-starter-json의 jackson 기본 설정]]
