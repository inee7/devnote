# Kotlin Data Class와 Jackson 직렬화 동작 정리

앞서 역직렬화(JSON → 객체) 동작을 정리했으니, 이번에는 **직렬화(객체 → JSON)** 기준을 정리합니다.  
Jackson이 어떤 프로퍼티를 JSON 출력 대상으로 삼는지 Kotlin data class 기준으로 살펴봅니다.

---

## 1. 기본 규칙

- Jackson은 기본적으로 **public getter 메서드** 혹은 **public 필드**를 직렬화 대상으로 삼습니다.  
- Kotlin `data class`는 자동으로 **`val`/`var` 프로퍼티마다 public getter**가 생성되므로 대부분 JSON에 출력됩니다.

---

## 2. `val` / `var`

- **val (불변)** → getter만 존재 → JSON에 포함  
- **var (가변)** → getter + setter → JSON에 포함  
- **정리**: 생성자에 있든 없든, `public` getter가 있으면 직렬화됨.

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

---

## 3. `private val` / `private var`

- getter/setter가 **생성되지 않으므로** JSON 출력에서 제외됨.  
- 출력하고 싶다면 `@get:JsonProperty`를 붙여야 함.

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

---

## 4. 본문 프로퍼티 (생성자 외부 정의)

- 생성자와 관계없이 **getter가 public이면 직렬화 대상**.  
- `val`/`var` 둘 다 동일하게 적용.  
- 역직렬화 시에는 생성자 매핑/ setter 여부가 중요하지만, 직렬화는 단순히 getter 여부만 본다.

```kotlin
class User(val id: Long) {
    val createdAt: String = "2025-08-28"
}
```

출력:
```json
{ "id": 1, "createdAt": "2025-08-28" }
```

---

## 5. `lateinit var`

- getter는 있으므로 JSON에 포함됨.  
- 값이 초기화되지 않은 상태에서 직렬화하면 NPE 발생 가능 → 주의 필요.

---

## 6. 어노테이션 제어

- `@JsonIgnore` → 해당 필드/프로퍼티 직렬화 제외  
- `@JsonProperty("alias")` → JSON key 이름 변경  
- `@JsonInclude` → 값이 `null`이거나 기본값이면 제외 가능  

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

---

## 7. 커스텀 직렬화 기준

ObjectMapper 전역 설정:
```kotlin
val mapper = ObjectMapper()
    .registerModule(KotlinModule())
    .setSerializationInclusion(JsonInclude.Include.NON_NULL) // null 제외
    .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS) // 날짜 포맷 변경
```

---

## 📌 요약 (역직렬화 vs 직렬화)

| 구분 | 역직렬화 (JSON → 객체) | 직렬화 (객체 → JSON) |
|------|-----------------------|----------------------|
| **val 생성자 파라미터** | 생성자에서 주입 | getter 통해 출력됨 |
| **var 생성자 파라미터** | 생성자에서 주입 (추가로 setter도 가능) | getter 통해 출력됨 |
| **private val/var** | 생성자에서는 주입 가능, setter 불가 | 기본적으로 출력 안 됨 (필요 시 `@get:JsonProperty`) |
| **본문 정의 val/var** | val은 주입 불가, var은 setter로 주입 가능 | getter 있으면 출력됨 |
| **lateinit var** | setter로 주입 가능 | 초기화 후 getter 통해 출력됨 |
| **중복 (생성자+setter)** | 생성자가 우선, setter는 중복 호출 안 됨 | getter 기준으로 1번만 출력 |

---

## 결론

- **역직렬화**는 “생성자 → setter → 필드 접근” 순서로 값을 채움.  
- **직렬화**는 “getter/public 필드” 기준으로 JSON을 만듦.  
- 따라서 `private` 프로퍼티는 기본적으로 출력되지 않고, 필요 시 `@get:JsonProperty`로 노출해야 함.
