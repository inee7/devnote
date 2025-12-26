# Java 클래스와 Jackson 직렬화/역직렬화 정리 (Lombok 포함)

Jackson은 Java 객체를 직렬화(객체 → JSON) 및 역직렬화(JSON → 객체)할 때 특정 규칙을 따릅니다.  
여기서는 기본 규칙과 Lombok 사용 시 주의사항, 그리고 `-parameters` 옵션까지 정리합니다.

---

## 1. 직렬화 (객체 → JSON)

Jackson은 **getter / public 필드**를 직렬화 대상으로 삼습니다.

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

---

## 2. 역직렬화 (JSON → 객체)

Jackson은 기본적으로 **기본 생성자 + setter** 방식으로 객체를 만듭니다.

1. **생성자 선택**
   - no-arg constructor 있으면 객체 생성 후 setter 호출
   - `@JsonCreator`가 붙은 생성자 있으면 해당 생성자 사용
   - record / 불변 객체 → 생성자 기반 사용

2. **프로퍼티 매핑**
   - JSON key ↔ setter (`setXxx()`)
   - JSON key ↔ public 필드
   - `@JsonProperty("alias")`로 매핑 이름 지정

3. **우선순위**
   - `@JsonCreator` 생성자 > setter > public 필드

---

## 3. Lombok 사용 시 주의사항

### (1) @Data / @Getter / @Setter
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

### (2) @Builder
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

### (3) @Value (불변 객체)
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

### (4) @NoArgsConstructor(access = PROTECTED)
- JPA 엔티티에서 자주 사용
- Jackson도 호출 가능 → 문제 없음

---

## 4. 기본 생성자 없이 역직렬화 가능한 경우

- **불변 객체 (final 필드 + 생성자)**  
  → `-parameters` 옵션 + `jackson-module-parameter-names` 필요
- **@JsonCreator + @JsonProperty**  
  → 옵션 없이도 가능 (명시적 매핑)
- **Java Record (JDK 16+)**  
  → 자동 지원 (Jackson 2.12+)
- **Lombok @Value**  
  → 생성자 기반 역직렬화 (보통 @JsonCreator 필요)

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

JSON:
```json
{ "id": 1, "name": "친구" }
```

→ `new User(1, "친구")` 호출 성공 (단, -parameters 필요)

---

## 5. Java 컴파일 옵션: -parameters

생성자 기반 역직렬화에서 JSON key와 파라미터 이름을 매핑하려면  
**컴파일 시 파라미터 이름을 바이트코드에 포함**해야 합니다.

### Maven
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

### Gradle
```kotlin
tasks.withType<JavaCompile> {
    options.compilerArgs.add("-parameters")
}
```

### Jackson 모듈
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

---

## 6. 요약

| 구분 | 직렬화 (객체 → JSON) | 역직렬화 (JSON → 객체) |
|------|----------------------|------------------------|
| **public getter** | JSON에 포함 | setter 필요 |
| **public 필드** | JSON에 포함 | setter 없이 매핑 가능 |
| **private 필드** | 출력 X (단, @JsonProperty로 가능) | 매핑 불가 |
| **final 필드** | getter 있으면 출력 | 생성자 필요 |
| **기본 생성자 없음** | 직렬화 OK | 생성자 기반 매핑 (조건 충족 시) |
| **Lombok @Data** | 직렬화/역직렬화 가능 | no-arg 필요 |
| **Lombok @Builder** | 직렬화 O | 역직렬화 불가 (설정 필요) |
| **Lombok @Value** | 직렬화 O | 생성자 기반 매핑 필요 |
| **Java Record** | 직렬화 O | 자동 매핑 |

---

## 결론

- 직렬화: **getter/public 필드** 기준  
- 역직렬화: **기본 생성자 + setter** 또는 **생성자 기반 매핑**  
- `-parameters` 옵션 + `jackson-module-parameter-names` → 불변 객체 매핑 지원  
- Lombok:  
  - `@Data`는 대부분 문제 없음, no-arg 필요시 추가  
  - `@Builder`는 역직렬화에 추가 설정 필요  
  - `@Value`는 `@JsonCreator` 생성자 권장  
  - `Record`는 가장 깔끔한 불변 객체 매핑 방식  
