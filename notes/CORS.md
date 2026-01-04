---
tags: [security, web, http, cors, sop]
---

# CORS

## 한 줄 요약

CORS는 SOP(동일 출처 정책)을 우회하여 다른 출처의 리소스 접근을 허용하는 HTTP 헤더 기반 메커니즘이다

## 핵심 정리

- SOP(Same-Origin Policy): 프로토콜, 호스트, 포트가 모두 같아야 동일 출처
- CORS(Cross-Origin Resource Sharing): 서버가 다른 출처의 접근을 허용하는 메커니즘
- Preflight Request: OPTIONS 메서드로 사전 검증
- Access-Control-Allow-Origin 헤더로 허용 출처 지정
- Spring에서는 `@CrossOrigin` 또는 WebMvcConfigurer로 설정

## 상세 내용

### SOP(Same-Origin Policy)

브라우저 초기에 보안상의 이유로 스크립트 내에서 시작된 **교차 출처 HTTP 요청을 제한**하는데, 이를 **SOP(Same-Origin Policy, 동일 출처 정책)** 라 한다.

**동일 출처 조건:**
- 프로토콜 (http vs https)
- 호스트 (example.com vs api.example.com)
- 포트 (3000 vs 8080)

세 가지가 **모두 같아야** 동일 출처다.

**예시:**

```
https://example.com:443/app
```

| URL | 동일 출처 여부 | 이유 |
|-----|------------|------|
| https://example.com:443/api | ✅ | 프로토콜, 호스트, 포트 동일 |
| https://example.com/api | ✅ | https는 443 포트 생략 가능 |
| http://example.com/api | ❌ | 프로토콜 다름 (http) |
| https://api.example.com/api | ❌ | 호스트 다름 (서브도메인) |
| https://example.com:8080/api | ❌ | 포트 다름 |

### CORS(Cross-Origin Resource Sharing)

SOP를 우회하기 위한 방법:
1. **브라우저 측**: JSONP 사용 (레거시)
2. **서버 측**: CORS 설정 (권장)

**CORS란?**
웹 서버가 도메인 간 액세스 제어 기능을 제공하여 **서로 다른 출처 간 데이터 전송을 가능하게** 해주는 메커니즘이다.

### CORS 동작 방식

#### 1. Simple Request (단순 요청)

다음 조건을 모두 만족하는 경우:
- 메서드: GET, HEAD, POST 중 하나
- 헤더: Accept, Accept-Language, Content-Language, Content-Type만 사용
- Content-Type: application/x-www-form-urlencoded, multipart/form-data, text/plain

```
Browser → Server: GET /api/users
                  Origin: https://frontend.com

Server → Browser: Access-Control-Allow-Origin: https://frontend.com
                  (실제 데이터)
```

#### 2. Preflight Request (사전 요청)

복잡한 요청의 경우 OPTIONS 메서드로 사전 검증:

```
Browser → Server: OPTIONS /api/users
                  Origin: https://frontend.com
                  Access-Control-Request-Method: POST
                  Access-Control-Request-Headers: Content-Type

Server → Browser: Access-Control-Allow-Origin: https://frontend.com
                  Access-Control-Allow-Methods: GET, POST, PUT, DELETE
                  Access-Control-Allow-Headers: Content-Type
                  Access-Control-Max-Age: 86400

Browser → Server: POST /api/users (실제 요청)
```

## 실무 적용

### Spring Boot 설정

**방법 1: @CrossOrigin 애노테이션**

```java
@RestController
@RequestMapping("/api")
@CrossOrigin(origins = "https://frontend.com")
public class UserController {

    @GetMapping("/users")
    public List<User> getUsers() {
        return userService.findAll();
    }
}
```

**방법 2: WebMvcConfigurer (글로벌 설정)**

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://frontend.com", "https://admin.frontend.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

**방법 3: Filter 사용**

```java
@Component
public class CorsFilter implements Filter {

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
        throws IOException, ServletException {

        HttpServletResponse response = (HttpServletResponse) res;
        HttpServletRequest request = (HttpServletRequest) req;

        response.setHeader("Access-Control-Allow-Origin", "https://frontend.com");
        response.setHeader("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS");
        response.setHeader("Access-Control-Allow-Headers", "Content-Type, Authorization");
        response.setHeader("Access-Control-Max-Age", "3600");

        if ("OPTIONS".equalsIgnoreCase(request.getMethod())) {
            response.setStatus(HttpServletResponse.SC_OK);
        } else {
            chain.doFilter(req, res);
        }
    }
}
```

### 보안 고려사항

**❌ 잘못된 설정:**
```java
.allowedOrigins("*")  // 모든 출처 허용 - 보안 위험
.allowCredentials(true)  // 쿠키 포함과 *는 함께 사용 불가
```

**✅ 올바른 설정:**
```java
.allowedOrigins("https://frontend.com")  // 명시적으로 허용
.allowedOriginPatterns("https://*.example.com")  // 패턴 사용
.allowCredentials(true)  // 쿠키 포함 시 명시적 출처 필요
```

### Credentials 포함 요청

```javascript
// Frontend
fetch('https://api.example.com/users', {
    method: 'GET',
    credentials: 'include'  // 쿠키 포함
})
```

```java
// Backend
.allowCredentials(true)
.allowedOrigins("https://frontend.com")  // * 불가
```

## 관련 노트

- [[API requestresponse 로깅]]
- [[Spring Boot 테스트 전략과 테스트 더블]]

