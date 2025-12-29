
# Kafka 핵심 개념 정리

> 참고자료  
> - [Apache Kafka Quickstart](https://kafka.apache.org/quickstart)  
> - [Spring for Apache Kafka Reference](https://docs.spring.io/spring-kafka/reference/)

---

## 1. Kafka 개요

Apache Kafka는 대용량 실시간 데이터를 처리하기 위한 고성능 **분산 메시징 플랫폼**입니다.  

### 주요 특징

- **고성능**: 높은 처리량, 낮은 지연시간  
- **확장성**: 브로커를 수평 확장 가능  
- **내구성**: 장애 발생 시에도 메시지 유실 없이 재처리 가능  
- **고가용성**: 리플리케이션을 통해 안정성 확보  
- **느슨한 결합**: 컨슈머 완전 분리  
- **생태계**: Kafka Connect, Schema Registry 등 도구 제공  

---

## 2. Kafka 아키텍처

Kafka는 여러 구성 요소가 계층적으로 결합된 구조입니다.  

> 구조 흐름: **Cluster → Broker → Topic → Partition → Segment → Record**

### 주요 구성 요소

| 구성 요소       | 설명 |
|----------------|------|
| **Kafka Cluster** | 여러 브로커로 구성된 시스템 전체 |
| **Broker**     | Kafka 서버 프로세스 |
| **Topic**      | 메시지 스트림 단위 (논리 단위), 이메일 주소 정도의 개념 |
| **Partition**  | 병렬 처리를 위한 단위 (물리 저장) |
| **Segment**    | Partition 내부 로그 파일 |
| **Offset**     | Partition 내 메시지의 위치 (64-bit) |
| **Producer**   | Kafka로 메시지 전송 |
| **Consumer**   | Kafka에서 메시지 소비 |
| **Zookeeper**  | (기존 구조) 메타데이터 관리 및 브로커 상태 확인 |
| **KRaft**      | (신규 구조) Zookeeper 없이 Kafka만으로 클러스터 관리 |

---

## 3. 메시지 흐름: Producer → Kafka → Consumer

Kafka에서 메시지는 다음 흐름을 따릅니다.

### 3-1. Producer
![[resources/images/kafka-producer.png]]
- 메시지를 Kafka(파티션선택하여)에 전송  
- **Topic, Value**는 필수 / **Key, Partition**은 선택  

#### 파티션 선택 전략

| 방식        | 설명 |
|-------------|------|
| Key 기반     | 같은 Key → 같은 Partition → 순서 보장 |
| Round-robin | 무작위 분산, 순서 보장 안 됨 (기본값) |
| 직접 지정    | 특정 Partition으로 지정 전송 |

#### 전송 보장 수준
- **At least once** (기본): 중복 가능  
- **At most once**: 손실 가능성 존재  
- **Exactly once**: PID + 메시지 번호로 중복 방지  

#### 배치 전송

- 메시지를 모아서 한 번에 전송 → 네트워크 오버헤드 감소  
- 내부적으로 `send()` 후 파티션별로 모아서 배치 처리  

---

### 3-2. Partition & Segment

#### Partition
![[resources/images/kafka-topic-partition.png]]
- Topic을 병렬 처리 단위로 나눈 구조  
- 병렬성과 처리량을 높이기 위한 핵심 구성  
- **주의**: Partition 수는 줄일 수 없으며 초기 설정 중요  

> 권장: 2~4개로 시작 후 점진적으로 조정  
> Key 기반 전송 시 Partition 수 변경에 주의 (Hash 충돌)

#### Segment
![[resources/images/kafka-segment.png]]
- Partition 내부의 실제 저장 단위 (로그 파일)  
- 디스크 경로 예: `/data/kafka-logs/{topic}-{partition}/`

---

### 3-3. Consumer
![[resources/images/kafka-consumer.png]]
- Kafka는 **Pull 방식** (컨슈머가 직접 메시지 읽음)  
- Offset을 기반으로 메시지 위치 추적  
- **Consumer Group**으로 묶어 병렬 소비 가능  

#### Consumer Group 특징

- 동일 Group ID를 가진 컨슈머들로 구성  
- 하나의 Partition은 하나의 Consumer만 소비  
- Group 내 컨슈머 수 ≤ Partition 수가 이상적  

#### Offset 관리

- **Auto Commit**: 주기적으로 자동 커밋 (성능 좋지만 중복 가능)  
- **Sync Commit**: 안전하지만 성능 저하  
- **Async Commit**: 빠르지만 실패 시 재시도 안 함  

---

## 4. Kafka 고급 기능

### 4-1. 리플리케이션

- Partition 별 복제본 유지 (기본 3개)  `--replication-factor 3`
- 토픽 전체가 아닌 **파티션 단위로 복제**됨  
- 장애 발생 시 다른 브로커로 대체 가능  

| 역할     | 설명 |
|----------|------|
| **리더**   | 모든 읽기/쓰기 처리 |
| **팔로워** | 리더 복제만 수행 |

---

### 4-2. 압축 전송

Kafka는 성능 최적화를 위해 메시지를 압축할 수 있습니다.

| 알고리즘     | 장점 |
|--------------|------|
| `gzip`, `zstd`   | 높은 압축률 (CPU 사용 높음) |
| `lz4`, `snappy` | 빠른 전송 속도 (압축률 낮음) |

---

### 4-3. 페이지 캐시(Page Cache)
![[resources/images/kafka-page-cache.png]]
- Kafka는 디스크 직접 I/O 대신 **OS의 Page Cache**를 활용  
- 디스크 I/O를 줄이고 메시지 처리 성능 향상  

---

### 4-4. Retry 토픽과 Bean 생성

Spring Kafka에서는 `@RetryableTopic` 어노테이션 사용 시 자동으로 Retry용 `NewTopic` 빈을 생성합니다.
```java
public RetryTopicConfiguration findRetryConfigurationFor(String[] topics, Method method, Object bean) {
	RetryableTopic annotation = AnnotationUtils.findAnnotation(method, RetryableTopic.class);
	return annotation != null
			? new RetryableTopicAnnotationProcessor(this.beanFactory, this.resolver, this.expressionContext)
					.processAnnotation(topics, method, annotation, bean)
			: maybeGetFromContext(topics);
}

```
- 어노테이션으로 찾거나 maybeGetFromContext에서는 RetryTopicConfiguration를 찾는다 
- 그럼 어노테이션이 두개라면 같은 토픽으로 만들수 없다 중복빈 에러가 나기 때문이다 
- 빈을 만들지 않게 설정하면 된다 근데 그 빈이 뭘까 
- NewTopic을 빈으로 만드는건데 이건 스프링에서 브로커에 자동으로 만들게 할때 쓰는데 카프카 설정으로 auto create 꺼져있으면 의미 없다 
- 브로커 설정에 따라 (`auto.create.topic.enable=false`) 자동 생성되지 않을 수 있음  

---

## 5. Kafka 발전 히스토리

- 2013년: 리플리케이션 기능 도입 (내구성 강화)  
- Schema Registry 도입 → 데이터 스키마 합의  
- Kafka Connect 도입 → 다양한 시스템과 연결 가능  
- KRaft 구조 도입 → Zookeeper 의존도 제거  

---

## 마무리

Kafka는 이벤트 기반 시스템, 실시간 로그 수집, 데이터 스트리밍 처리 등 다양한 곳에서 활약하고 있는 **핵심 인프라 기술**입니다.  
Spring Kafka와 함께 사용하면 운영 및 개발 편의성도 높아지므로 백엔드 시스템에서의 적용 가치가 매우 큽니다.

#kafka 