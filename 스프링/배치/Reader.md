DB, File, XML, JSON등 다양한 데이터 소스를 읽을수 있다. 스프링에서 지원하는 Reader를 쓸수 없을때는 직접 만들어 쓴다. ItemReader 인터페이스와 ItemStream 인터페이스가 중요한 부분을 담당하는데 ItemReader는 읽음 처리를 스펙이고 ItemStream은 상태저장과 복원을 담당하는 스펙이다. 둘을 구현하여 Reader를 쓰면 되는데 직접 구현할일은 드물다. (QueryDsl, Jooq 과 같은 경우에 필요할수도?)

DB Reader에 관해서만 보자

Spring의 JdbcTemplate을 쓰면 대용량 데이터를 limit, offset을 이용해 분할 처리해야한다. SpringBatch에서 이를 지원하며 쓸 수 있는 두가지 타입의 Reader가 있다.

### Cursor vs Paging

Cursor는 ResultSet이 open될 때마다 next() 메소드가 호출 되어 데이터를 스트리밍하여 반환 Paging은 Chunk 단위로 데이터를 조회 Cursor는 1Row, Paging은 PageSize만큼 Cursor는 DB 커넥션 맺은후 Cursor를 한칸씩 옮기면서 가져오고 다 읽어오면 커넥션을 끊는 반면 Paging은 PageSize만큼 가져오고 바로 커넥션을 끊는다 멀티쓰레드에서 사용할수 있는건 Paging이다

## Cursor

### JdbcCursorItemReader <!-- {"fold":true} -->

```java
@Slf4j
@RequiredArgsConstructor
@Configuration
public class JdbcCursorItemReaderJobConfiguration {

    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    private final DataSource dataSource; // DataSource DI

    private static final int chunkSize = 10;

    @Bean
    public Job jdbcCursorItemReaderJob() {
        return jobBuilderFactory.get("jdbcCursorItemReaderJob")
                .start(jdbcCursorItemReaderStep())
                .build();
    }

    @Bean
    public Step jdbcCursorItemReaderStep() {
        return stepBuilderFactory.get("jdbcCursorItemReaderStep")
                .<Pay, Pay>chunk(chunkSize)
                .reader(jdbcCursorItemReader())
                .writer(jdbcCursorItemWriter())
                .build();
    }

    @Bean
    public JdbcCursorItemReader<Pay> jdbcCursorItemReader() {
        return new JdbcCursorItemReaderBuilder<Pay>()
                .fetchSize(chunkSize)
                .dataSource(dataSource)
                .rowMapper(new BeanPropertyRowMapper<>(Pay.class))
                .sql("SELECT id, amount, tx_name, tx_date_time FROM pay")
                .name("jdbcCursorItemReader")
                .build();
    }

    private ItemWriter<Pay> jdbcCursorItemWriter() {
        return list -> {
            for (Pay pay: list) {
                log.info("Current Pay={}", pay);
            }
        };
    }
}

```

~<Pay, Pay>chunk(chunkSize)~ 에서 첫번째는 Reader에서 반환될 타입이고 두번째는 Writer에 넘어올 타입이고 chunkSize만큼 트랜잭션을 묶는다. JdbcCursorItemReader에 ~.fetchSize(chunkSize)~ 는 쿼리와 독립적으로 JDBC 내부적으로 DB서버에서 fetch size만큼 가져오고 하나씩 read한다 ( 하지만 모든 데이터를 fetch해서 cursor를 옮기며 가져온다. mysql-connector-java에서 cursor fetch 기능을 기본적으로 막아놨다. 옵션으로 userCursorFetch를 추가해야함 [Spring batch Mysql + JdbcCursorReader 사용시 확인할 점 | Osori Blog](https://osoriandomori.github.io/posts/spring-batch-jdbc-cursor-reader/), JpaCursorItemReader는 모든 데이터 읽는다는 소리가 있는데 뭐가 맞는지 몰겠음 [if kakao 2022 Batch Performance를 고려한 최선의 Reader | 카카오페이 기술 블로그](https://tech.kakaopay.com/post/ifkakao2022-batch-performance-read/))

~DataSource~ 가 필요하긴하다. 쿼리 결과를 맵핑할 ~rowMapper~ 가 필요 하나씩 가져와서 writer로 던져주는게 특징이고 chunk size가 되면 commit한다.

### JpaCursorItemReader

```java
@Slf4j // log 사용을 위한 lombok 어노테이션
@RequiredArgsConstructor // 생성자 DI를 위한 lombok 어노테이션
@Configuration
public class JpaCursorItemReaderJobConfig {
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    private final EntityManagerFactory entityManagerFactory;

    private int chunkSize;
    @Value("${chunkSize:100}")
    public void setChunkSize(int chunkSize) {
        this.chunkSize = chunkSize;
    }

    @Bean
    public Job jpaCursorItemReaderJob() {
        return jobBuilderFactory.get("jpaCursorItemReaderJob")
                .start(jpaCursorItemReaderStep())
                .build();
    }

    @Bean
    public Step jpaCursorItemReaderStep() {
        return stepBuilderFactory.get("jpaCursorItemReaderStep")
                .<Pay, Pay>chunk(chunkSize)
                .reader(jpaCursorItemReader())
                .writer(jpaCursorItemWriter())
                .build();
    }

    @Bean
    public JpaCursorItemReader<Pay> jpaCursorItemReader() {
        return new JpaCursorItemReaderBuilder<Pay>()
                .name("jpaCursorItemReader")
                .entityManagerFactory(entityManagerFactory)
				   .maxItemCount(10)
				   .currentItemCount(0)
                .queryString("SELECT p FROM Pay p")
                .build();
    }

    private ItemWriter<Pay> jpaCursorItemWriter() {
        return list -> {
            for (Pay pay: list) {
                log.info("Current Pay={}", pay);
            }
        };
    }
}

```

**CursorItemReader 주의사항** DB서버와의 SocketTimeout을 충분히 큰 값으로 설정해야만 함 Cursor는 하나의 커넥션으로 배치가 종료될때 까지 유지한다. 배치 수행시간이 오래걸리면 PagingItemReader를 추천 (한 페이지를 읽을때마다 커넥션 끊기때문)

그외 HibernateCursorItemReader, StoredProcedureItemReader 가 있음

## Paging

### JdbcPagingItemReader

JdbcCursorItemReader과 같이 JdbcTemplate 인터페이스를 이용함

```java
@Slf4j
@RequiredArgsConstructor
@Configuration
public class JdbcPagingItemReaderJobConfiguration {

    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    private final DataSource dataSource; // DataSource DI

    private static final int chunkSize = 10;

    @Bean
    public Job jdbcPagingItemReaderJob() throws Exception {
        return jobBuilderFactory.get("jdbcPagingItemReaderJob")
                .start(jdbcPagingItemReaderStep())
                .build();
    }

    @Bean
    public Step jdbcPagingItemReaderStep() throws Exception {
        return stepBuilderFactory.get("jdbcPagingItemReaderStep")
                .<Pay, Pay>chunk(chunkSize)
                .reader(jdbcPagingItemReader())
                .writer(jdbcPagingItemWriter())
                .build();
    }

    @Bean
    public JdbcPagingItemReader<Pay> jdbcPagingItemReader() throws Exception {
        Map<String, Object> parameterValues = new HashMap<>();
        parameterValues.put("amount", 2000);

        return new JdbcPagingItemReaderBuilder<Pay>()
                .pageSize(chunkSize)
                .fetchSize(chunkSize)
                .dataSource(dataSource)
                .rowMapper(new BeanPropertyRowMapper<>(Pay.class))
                .queryProvider(createQueryProvider())
                .parameterValues(parameterValues)
                .name("jdbcPagingItemReader")
                .build();
    }

    private ItemWriter<Pay> jdbcPagingItemWriter() {
        return list -> {
            for (Pay pay: list) {
                log.info("Current Pay={}", pay);
            }
        };
    }

    @Bean
    public PagingQueryProvider createQueryProvider() throws Exception {
        SqlPagingQueryProviderFactoryBean queryProvider = new SqlPagingQueryProviderFactoryBean();
        queryProvider.setDataSource(dataSource); // Database에 맞는 PagingQueryProvider를 선택하기 위해
        queryProvider.setSelectClause("select id, amount, tx_name, tx_date_time");
        queryProvider.setFromClause("from pay");
        queryProvider.setWhereClause("where amount >= :amount");

        Map<String, Order> sortKeys = new HashMap<>(1);
        sortKeys.put("id", Order.ASCENDING);

        queryProvider.setSortKeys(sortKeys);

        return queryProvider.getObject();
    }
}

```

~.<Pay, Pay>chunk(chunkSize)~ 는 위와 같고 .fetchSize(chunkSize)외에 ~.pageSize(chunkSize)~ 도 있다. 이를 통해 limit 쿼리가 들어간다. 그리고 PagingQueryProvider를 통해 쿼리 작성 한다. 이렇게 작성을 해야 DB타입에 맞는 Paging 전략을 찾아 맞춰준다.

### JpaPagingItemReader

```java
@Slf4j // log 사용을 위한 lombok 어노테이션
@RequiredArgsConstructor // 생성자 DI를 위한 lombok 어노테이션
@Configuration
public class JpaPagingItemReaderJobConfiguration {
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    private final EntityManagerFactory entityManagerFactory;

    private int chunkSize = 10;

    @Bean
    public Job jpaPagingItemReaderJob() {
        return jobBuilderFactory.get("jpaPagingItemReaderJob")
                .start(jpaPagingItemReaderStep())
                .build();
    }

    @Bean
    public Step jpaPagingItemReaderStep() {
        return stepBuilderFactory.get("jpaPagingItemReaderStep")
                .<Pay, Pay>chunk(chunkSize)
                .reader(jpaPagingItemReader())
                .writer(jpaPagingItemWriter())
                .build();
    }

    @Bean
    public JpaPagingItemReader<Pay> jpaPagingItemReader() {
        return new JpaPagingItemReaderBuilder<Pay>()
                .name("jpaPagingItemReader")
                .entityManagerFactory(entityManagerFactory)
                .pageSize(chunkSize)
                .queryString("SELECT p FROM Pay p WHERE amount >= 2000")
                .build();
    }

    private ItemWriter<Pay> jpaPagingItemWriter() {
        return list -> {
            for (Pay pay: list) {
                log.info("Current Pay={}", pay);
            }
        };
    }
}

```

~.entityManagerFactory(entityManagerFactory)~ 가 기존 jdbc와 다르다

**PagingItemReader 주의점** 정렬이 무조건 포함되어야 업데이트되는 데이터 순서가 꼬이지 않는다

그외 HibernatePagingItemReader 있음

## 기타

JpaRepository를 써야하면 RepositoryItemReader를 써라

```java
  @Bean(name = "lotteryInfoReader")
    @StepScope
    public RepositoryItemReader<LotteryInfo> reader() {
        RepositoryItemReader<LotteryInfo> reader = new RepositoryItemReader<>();
        reader.setRepository(lotteryInfoRepository);
        reader.setMethodName("findAll");
        reader.setSort(Collections.singletonMap("name", Sort.Direction.ASC));
        return reader;
    }

```

Hibernate, JPA 등 영속성 컨텍스트가 필요한 Reader 사용시 fetchSize와 ChunkSize는 같은 값을 유지해야 합니다.