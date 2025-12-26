# @Profile 쓰지 말고 Property로 제어하는 방법

[https://reflectoring.io/dont-use-spring-profile-annotation/](https://reflectoring.io/dont-use-spring-profile-annotation/)

`@Profile("test")`, `@Profile("!test")` , `@Profile("h2")`

식으로 구성하면 전체검색으로 프로필을 하나하나 살펴야 어떻게 구성 되어있는지 확인 가능하다 특정 프로필을 한눈에 확인하기 어렵다 특히 !test는 너무 찾기 어렵다

yml에 환경별로 특정 프로퍼티를 넣고

```java
@Configuration
class MyConfiguration {
	@Bean
	@ConditionalOnProperty(name="service.mock", havingValue="true")
	Service mockService() {
		return new MockService();
	}

	@Bean
	@ConditionalOnProperty(name="service.mock", havingValue="false")
	Service realService() {
		return new RealService();
	}
}

```

처리하면 한눈에 확인이 가능하다