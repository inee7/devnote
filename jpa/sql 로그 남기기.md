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



### 포맷시키기

```yml
spring:
  jpa:
    properties:
      hibernate:
        show_sql: true
        format_sql: true
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
        show_sql: true
        format_sql: true
logging:
  level:
    org:
      hibernate:
        type:
          descriptor:
            sql: trace
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

