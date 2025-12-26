# Kotlin Data Class와 Jackson 역직렬화 동작 정리

Jackson을 Kotlin에서 사용할 때 가장 많이 쓰는 대상은 `data class`입니다.  
특히 **모든 프로퍼티가 `val`이고 생성자 기반으로만 정의된 경우** Jackson은 일반 Java POJO와는 다른 방식으로 역직렬화를 수행합니다.  
이 글에서는 Jackson이 Kotlin `data class`를 어떻게 역직렬화하는지, 생성자가 여러 개인 경우, `private val`, 본문 프로퍼티, setter 존재 여부 등에 따라 어떤 차이가 있는지 정리했습니다.

---

## 1. 기본 원리

- Kotlin `data class`의 경우 Jackson은 **기본 생성자(default constructor)** 를 찾지 않고  
  **primary constructor를 기반으로 역직렬화**를 수행합니다.
- JSON key → 생성자 파라미터 이름을 매핑해서 값을 넣고,  
  생성자를 호출하여 인스턴스를 생성합니다.
- `val` 프로퍼티는 setter가 없으므로, **생성자 시점에만 값이 설정**됩니다.

### 예시

```kotlin
data class User(
    val id: Long,
    val name: String,
    val email: String? = null // 기본값
)
```

```json
{ "id": 1, "name": "친구" }
```

→ 결과: `User(1, "친구", null)`

---

## 2. 생성자가 여러 개인 경우

- **우선순위**
  1. `@JsonCreator`가 붙은 생성자 (명시적 지정)
  2. Kotlin 모듈이 primary constructor를 선택
  3. 동일한 후보가 여러 개면 → **모호성 예외 발생**

- **@JvmOverloads**로 생긴 오버로드는 바이트코드에 여러 생성자가 생기지만,  
  Kotlin 모듈은 **전체 파라미터를 가진 primary constructor**를 사용합니다.

- **secondary constructor**를 JSON 입력으로 사용하려면 `@JsonCreator`를 붙여야 합니다.

---

## 3. `private val` 프로퍼티

- **생성자 파라미터가 `private val`** 인 경우에도 Jackson은 정상적으로 값을 주입합니다.  
  (생성자 호출 시 이미 전달되므로 접근 제한자는 문제되지 않음)
- 다만 **getter가 없으므로 직렬화 시 JSON에 출력되지 않음**.

### 예시

```kotlin
data class User(
    val id: Long,
    private val secret: String
)
```

역직렬화(JSON → 객체)는 가능하지만, 직렬화(객체 → JSON)에서는 `secret`이 빠집니다.  
출력까지 원하면 `@get:JsonProperty`를 추가해야 합니다.

---

## 4. 생성자에 없는 프로퍼티 (본문 정의)

생성자 파라미터가 아닌 클래스 본문 안에서 정의된 프로퍼티는 동작 방식이 다릅니다.

- **val (불변)**  
  - JSON에서 값을 주입할 수 없음 (항상 초기값 또는 `init` 블록 값 사용)
  - JSON에 해당 키가 있으면 → Jackson은 알 수 없는 프로퍼티로 간주할 수 있음
- **var (가변)**  
  - public setter가 있다면 생성 후 setter를 호출해 값을 세팅
- **lateinit var**  
  - JSON에 반드시 값이 있어야 하고, 주입 후 사용 가능

### 권장 방법
- JSON으로 받아야 한다면 → **생성자 파라미터로 올리는 것**이 가장 안전합니다.  
- 읽기 전용 계산 필드라면 → `@JsonProperty(access = READ_ONLY)`나 `@JsonIgnore` 사용.

---

## 5. 생성자 + Setter 동시 존재 시

- **동일한 프로퍼티**가 생성자에도 있고 setter도 있으면?
  - Jackson은 **생성자에서 이미 값이 주입된 경우 setter를 다시 호출하지 않습니다.**
- 즉, **중복으로 두 번 세팅되지 않는다**는 뜻입니다.
- 생성자에 없는 프로퍼티만 setter를 통해 값이 채워집니다.

---

## 6. 정리 (Cheat Sheet)

- ✅ 생성자 파라미터(`val`/`var`) → 생성자 호출 시 주입  
- ✅ `private val` → 역직렬화는 가능, 직렬화는 기본적으로 제외됨  
- ✅ 생성자에 없는 `var` 프로퍼티 → setter로 주입 가능  
- ❌ 생성자에 없는 `val` 프로퍼티 → 주입 불가 (항상 기본값 사용)  
- ✅ 생성자와 setter 동시 존재 → 생성자가 우선, setter는 중복 호출하지 않음  
- ⚠️ 생성자가 여러 개일 경우 → `@JsonCreator`로 단일 지정 권장  

---

## 참고 설정

알 수 없는 키 무시:
```kotlin
objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
```

불변 프로퍼티 직렬화 강제 노출:
```kotlin
data class User(
    @get:JsonProperty("secret")
    private val secret: String
)
```

---

## 결론

Kotlin `data class`와 Jackson을 함께 사용할 때 가장 중요한 점은  
**"모든 값은 생성자에서 주입된다는 것"**입니다.  

- 역직렬화 시에는 생성자 기반으로만 값을 세팅하고,  
- 생성자에 없는 프로퍼티만 setter를 통해 채웁니다.  
- 따라서 JSON으로부터 값을 안전하게 받으려면 **프로퍼티를 생성자 파라미터로 올리는 것**이 가장 깔끔한 방법입니다.
