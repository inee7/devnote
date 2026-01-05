---
tags: [spring, logging, filter, mdc]
---

# API Request/Response 로깅

## 한 줄 요약

OncePerRequestFilter와 ContentCachingWrapper를 사용하여 HTTP 요청/응답을 로깅하고, MDC로 요청별 추적을 가능하게 한다

## 핵심 정리

- `OncePerRequestFilter`: 각 요청당 한 번만 실행되는 필터
- `ContentCachingRequestWrapper`/`ResponseWrapper`: 요청/응답 본문 캐싱
- MDC에 request_id 저장: 로그 추적 용이
- `copyBodyToResponse()`: 응답 본문 복원 필수
- 민감 정보 마스킹 필수

## 상세 내용

### OncePerRequestFilter 사용

`OncePerRequestFilter`를 상속한 필터를 Bean으로 등록한다.

**특징:**
- 각 요청에 대해 한 번만 실행
- 요청별로 고유한 처리를 할 때 유용
- 포워드나 리다이렉트 시에도 중복 실행 방지

### MDC(Mapped Diagnostic Context) 활용

MDC에 `request_id`를 넣어서 기록하면 로그 추적이 쉬워진다.

```java
MDC.put("request_id", UUID.randomUUID().toString());
```

### ContentCachingWrapper 사용

`ContentCachingRequestWrapper`와 `ContentCachingResponseWrapper`를 이용하면 요청과 응답 본문을 캐싱하여 로깅 후에도 유지 가능하다.

**주의:** 로깅하고 마지막에 **`copyBodyToResponse()`를 반드시 호출**해야 클라이언트에게 응답이 전달된다.

## 상세 내용

### 기본 구현

```java
@Component
public class RequestResponseLoggingFilter extends OncePerRequestFilter {

    private static final Logger log = LoggerFactory.getLogger(RequestResponseLoggingFilter.class);

    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain
    ) throws ServletException, IOException {

        // Request ID 생성 및 MDC 등록
        String requestId = UUID.randomUUID().toString();
        MDC.put("request_id", requestId);

        // ContentCaching Wrapper 생성
        ContentCachingRequestWrapper wrappedRequest = new ContentCachingRequestWrapper(request);
        ContentCachingResponseWrapper wrappedResponse = new ContentCachingResponseWrapper(response);

        try {
            // 요청 로깅
            logRequest(wrappedRequest, requestId);

            // 다음 필터 실행
            filterChain.doFilter(wrappedRequest, wrappedResponse);

            // 응답 로깅
            logResponse(wrappedResponse, requestId);

        } finally {
            // 응답 본문 복원 (중요!)
            wrappedResponse.copyBodyToResponse();

            // MDC 정리
            MDC.clear();
        }
    }

    private void logRequest(ContentCachingRequestWrapper request, String requestId) {
        String uri = request.getRequestURI();
        String method = request.getMethod();
        String body = new String(request.getContentAsByteArray(), StandardCharsets.UTF_8);

        log.info("[{}] Request: {} {} - Body: {}", requestId, method, uri, maskSensitiveData(body));
    }

    private void logResponse(ContentCachingResponseWrapper response, String requestId) {
        int status = response.getStatus();
        String body = new String(response.getContentAsByteArray(), StandardCharsets.UTF_8);

        log.info("[{}] Response: {} - Body: {}", requestId, status, maskSensitiveData(body));
    }

    private String maskSensitiveData(String data) {
        // 민감 정보 마스킹 로직
        return data.replaceAll("\"password\":\"[^\"]*\"", "\"password\":\"***\"");
    }
}
```

### Logback 설정 with MDC

```xml
<!-- logback-spring.xml -->
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] [%X{request_id}] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE" />
    </root>
</configuration>
```

## 실무 적용

### 성능 고려사항

- 대용량 파일 업로드/다운로드 시 메모리 부담
- 특정 경로는 로깅 제외 (health check, static resources)

```java
@Override
protected boolean shouldNotFilter(HttpServletRequest request) {
    String path = request.getRequestURI();
    return path.startsWith("/actuator") ||
           path.startsWith("/static") ||
           path.startsWith("/health");
}
```

### 민감 정보 마스킹

- 비밀번호
- 카드 번호
- 개인 식별 정보 (주민등록번호, 전화번호 등)

### 로그 저장소

- ELK Stack (Elasticsearch, Logstash, Kibana)
- CloudWatch Logs
- Splunk

request_id를 기준으로 요청 전체 흐름 추적 가능
