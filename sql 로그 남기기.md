# sql 로그 옵션

## 하이버네이트 로그 

```yml
spring:
  jpa:
    properties:
      hibernate:
        show_sql: true
```

```sql 
Hibernate: select user0_.id as id1_0_, user0_.age as age2_0_, user0_.name as name3_0_ from user user0_ where user0_.name=?
```

위에것 보다 

```yml
logging.level.org.hibernate.SQL: debug
```

가 더 좋은듯하다. 

- **spring.jpa.properties.hibernate.show_sql: true**는 단순히 SQL 쿼리를 콘솔에 출력하는 데 사용되며, 설정이 매우 간단하지만 제어가 제한적입니다.

- **logging.level.org.hibernate.SQL: debug**는 Spring Boot의 로깅 시스템과 통합되어 더 유연하고 강력한 로깅을 제공합니다. SQL 쿼리뿐만 아니라 로그 포맷이나 출력 위치를 더 세밀하게 제어할 수 있습니다.

### 포맷시키기

```yml
spring:
  jpa:
    properties:
      hibernate:
        format_sql: true
logging.level.org.hibernate.SQL: debug
```

```sql
Hibernate: 
    select
        user0_.id as id1_0_,
        user0_.age as age2_0_,
        user0_.name as name3_0_ 
    from
        user user0_ 
    where
        user0_.name=?
```



## ?에 어떤 값인지까지 확인

```yml
spring:
  jpa:
    properties:
      hibernate:
        format_sql: true
logging.level.org.hibernate.SQL: debug

logging.level.org.hibernate.type.descriptor.sql: trace // 하이버네이트5 
logging.level.org.hibernate.orm.jdbc.bind: trace // 하이버네이트6 
```

```sq;
Hibernate: 
    select
        user0_.id as id1_0_,
        user0_.age as age2_0_,
        user0_.name as name3_0_ 
    from
        user user0_ 
    where
        user0_.name=?
2019-07-28 22:00:32.673 TRACE 33555 --- [nio-8080-exec-3] o.h.type.descriptor.sql.BasicBinder      : binding parameter [1] as [VARCHAR] - [김철수]
2019-07-28 22:00:32.676 TRACE 33555 --- [nio-8080-exec-3] o.h.type.descriptor.sql.BasicExtractor   : extracted value ([id1_0_] : [BIGINT]) - [1]
2019-07-28 22:00:32.680 TRACE 33555 --- [nio-8080-exec-3] o.h.type.descriptor.sql.BasicExtractor   : extracted value ([age2_0_] : [INTEGER]) - [15]
2019-07-28 22:00:32.680 TRACE 33555 --- [nio-8080-exec-3] o.h.type.descriptor.sql.BasicExtractor   : extracted value ([name3_0_] : [VARCHAR]) - [김철수]

```



### spring.jpa.hibernate와 spring.jpa.properties.hibernate 차이

`spring.jpa.hibernate`와 `spring.jpa.properties.hibernate`는 Spring Boot에서 `Hibernate`와 관련된 설정을 다루는 두 가지 방법으로, 각각의 설정 범위와 적용 방식에서 차이가 있습니다.

### 1. **`spring.jpa.hibernate`**
`spring.jpa.hibernate`는 Spring Boot에서 `Hibernate`와 관련된 설정을 쉽게 관리하기 위해 제공하는 기본 설정 옵션입니다. Spring Boot에서 자동 설정된 `JPA` 속성에 대한 일부 `Hibernate` 옵션을 직접 제어하는 데 사용됩니다.

예를 들어:

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: update
      naming:
        physical-strategy: org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl
```

- **`ddl-auto`**: 데이터베이스 스키마 생성 및 업데이트 전략을 지정합니다.
- **`naming.physical-strategy`**: Hibernate의 물리적인 네이밍 전략을 설정합니다.

이와 같은 설정들은 Hibernate의 특정 기능을 제어하는 데 사용되며, 기본적으로 Spring Boot에서 제공하는 간단한 설정 옵션입니다.

### 2. **`spring.jpa.properties.hibernate`**
`spring.jpa.properties.hibernate`는 `Hibernate`의 더 세부적인 설정을 제어하기 위해 사용됩니다. 이는 Hibernate의 기본 설정을 직접적으로 설정할 수 있도록 해줍니다. 이 방식은 Hibernate의 `hibernate.properties` 파일에 설정하는 것과 같은 역할을 합니다.

예를 들어:

```yaml
spring:
  jpa:
    properties:
      hibernate:
        show_sql: true
        format_sql: true
```

- **`show_sql`**: Hibernate가 실행하는 SQL 쿼리를 로그에 출력합니다.
- **`format_sql`**: 출력된 SQL을 보기 좋게 포맷합니다.

이와 같은 설정들은 Hibernate의 세부적인 동작 방식을 직접 제어할 수 있도록 해줍니다.

### 주요 차이점

- **설정 범위**:
  - `spring.jpa.hibernate`: Spring Boot에서 제공하는 일부 `Hibernate` 관련 기본 설정만을 제어합니다.
  - `spring.jpa.properties.hibernate`: Hibernate의 전체 설정 옵션을 세부적으로 제어할 수 있습니다.

- **유연성**:
  - `spring.jpa.hibernate`: Spring Boot의 자동 설정 기능과 결합하여 사용하기에 편리하지만, 설정할 수 있는 항목이 제한적입니다.
  - `spring.jpa.properties.hibernate`: Hibernate의 고급 설정까지 모두 제어할 수 있어, 더 세부적이고 유연한 설정이 가능합니다.

- **사용 목적**:
  - `spring.jpa.hibernate`는 일반적인 설정을 위해 사용되며, 대부분의 경우 충분합니다.
  - `spring.jpa.properties.hibernate`는 특별한 Hibernate 설정이 필요한 경우, 또는 Spring Boot에서 제공하지 않는 고급 설정을 해야 할 때 사용됩니다.

### 요약
- **`spring.jpa.hibernate`**: Spring Boot에서 제공하는 간단한 Hibernate 설정 옵션을 제어.
- **`spring.jpa.properties.hibernate`**: Hibernate의 모든 세부 설정을 제어할 수 있는 옵션.

일반적으로, Spring Boot의 기본 설정을 활용하면서 필요한 경우 `spring.jpa.hibernate`를 사용하고, 더 세부적인 제어가 필요할 때 `spring.jpa.properties.hibernate`를 사용하는 것이 좋습니다.

#jpa #log 