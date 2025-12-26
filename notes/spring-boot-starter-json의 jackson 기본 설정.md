# spring-boot-starter-json의 jackson 기본 설정

스프링부트에서는 Jackson이 기본 JSON 직렬화 및 역직렬화 라이브러리로 사용되며, 글로벌 설정이 자동으로 구성됩니다. Spring Boot는 spring-boot-starter-json에 의해 Jackson을 포함하고 있으며, 이는 spring-boot-starter-web의 일부로 포함됩니다. 따라서 별도의 추가 설정 없이도 Jackson을 통해 JSON 처리가 가능합니다.
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

1. **날짜와 시간**:
   - 스프링부트는 Java 8 날짜 및 시간 API (e.g., LocalDate, LocalDateTime)를 지원하기 위해 JavaTimeModule을 자동으로 등록합니다.
   * 기본적으로 날짜와 시간은 ISO-8601 문자열 형식으로 직렬화됩니다.
2. **NULL 값 처리**:
   * 기본적으로 null 값은 직렬화되지 않습니다.
   * 필요한 경우 null 값을 포함하도록 설정할 수 있습니다.
3. **기타 설정**:
   * 기본적으로 FAIL_ON_UNKNOWN_PROPERTIES 설정이 false로 되어 있어, JSON에 매핑되지 않는 필드가 있어도 무시합니다.
   * INDENT_OUTPUT 설정이 false로 되어 있어, JSON 출력이 기본적으로 포맷팅되지 않습니다.


스프링 2.5부터 yml에서 objectmapper global custom 가능했다 

#spring #jackson
