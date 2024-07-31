# kafka retry토픽 빈

다음 코드에서 어노테이션 있으면 토픽 빈을 만든다 

```java
public RetryTopicConfiguration findRetryConfigurationFor(String[] topics, Method method, Object bean) {
	RetryableTopic annotation = AnnotationUtils.findAnnotation(method, RetryableTopic.class);
	return annotation != null
			? new RetryableTopicAnnotationProcessor(this.beanFactory, this.resolver, this.expressionContext)
					.processAnnotation(topics, method, annotation, bean)
			: maybeGetFromContext(topics);
}

```

어노테이션으로 찾거나 maybeGetFromContext에서는 RetryTopicConfiguration를 찾는다 
그럼 어노테이션이 두개라면 같은 토픽으로 만들수 없다 중복빈 에러가 나기 때문이다 
빈을 만들지 않게 설정하면 된다 
근데 그 빈이 뭘까 
NewTopic을 빈으로 만드는건데 이건 스프링에서 브로커에 자동으로 만들게 할때 쓰는데 카프카 설정으로 auto create 꺼져있으면 의미 없다 


