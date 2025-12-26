# @ConfigurationProperties

**설정파일의 값을 객체로 바인딩 가능하게 해주는 애노테이션**

`@ConfigurationProperties`는 특정 프리픽스(prefix)를 기준으로 매핑

```java
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private String name;
    private int version;

    // Getters and Setters
    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getVersion() {
        return version;
    }

    public void setVersion(int version) {
        this.version = version;
    }
}
```

**설정 파일 예시:**

```yaml
app:
  name: MyApp
  version: 1
```

`@ConfigurationProperties` 는 bean이 되어야한다
`@Componenet`로 bean이 되거나 `@EnableConfigurationProperties(AppProperties.class)`

```java
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableConfigurationProperties(AppProperties.class)
public class AppConfig {
}
```

springboot 2.1 까지는 기본생성자와 setter를 통해 주입되었다
2.2부터 `@ConstructorBinding`를 통해 생성자로 주입 가능해졌다
```java
@ConstructorBinding
@ConfigurationProperties("config.person")
@RequiredArgsConstructor @ToString
public class Person {
    private final String name;
    private final int age;
}
```

3.0부터는 @Target이 생성자와 애노테이션에만 붙일수 있게 바꼈다 
```java
@ConfigurationProperties("config.person")
@ToString
public class Person {
    private final String name;
    private final int age;

    @ConstructorBinding
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
}
```

그런데 @Autowired처럼 생략가능하게 되어서 생성자만 있어도 `@ConstructorBinding`없이 사용가능
```java
@ConfigurationProperties("config.person")
@RequiredArgsConstructor @ToString
public class Person {
    private final String name;
    private final int age;
}
```

`@ConfigurationProperties`는 다양한 타입을 바인딩할 수 있습니다. 기본적인 타입(String, int, boolean 등)뿐만 아니라, 복잡한 객체나 리스트도 바인딩할 수 있습니다.

```java
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private String name;
    private int version;
    private List<String> servers;

    // Getters and Setters
}
```

**설정 파일 예시:**

```yaml
app:
  name: MyApp
  version: 1
  servers:
    - server1.example.com
    - server2.example.com
```

이 경우, `servers` 리스트는 설정 파일의 `servers` 항목을 그대로 반영하게 됩니다.

`@ConfigurationProperties`와 함께 Spring의 `JSR-303/JSR-380` 표준 애너테이션(`@NotNull`, `@Min`, `@Max`, `@Pattern` 등)을 사용하여 바인딩된 프로퍼티에 대한 유효성 검사를 할 수 있습니다.
```java
import javax.validation.constraints.NotNull;

@ConfigurationProperties(prefix = "app")
public class AppProperties {

    @NotNull
    private String name;

    private int version;

    // Getters and Setters
}
```

이 경우, `app.name`이 설정 파일에 누락되어 있거나 `null`일 경우 애플리케이션이 시작되지 않고 오류가 발생합니다.

#spring 