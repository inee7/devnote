---
tags: [kafka, message-queue, distributed-system, event-driven]
---

## 한 줄 요약

Apache Kafka는 대용량 실시간 데이터를 처리하는 분산 메시징 플랫폼으로, 높은 처리량과 확장성을 제공하며 이벤트 기반 아키텍처의 핵심 인프라다.

## 핵심 정리

- Kafka는 분산 메시징 플랫폼으로 고성능, 확장성, 내구성, 고가용성 제공
- 구조: Cluster → Broker → Topic → Partition → Segment → Record
- Producer는 메시지를 Partition에 전송 (Key 기반, Round-robin, 직접 지정)
- Consumer는 Pull 방식으로 메시지 소비 (Consumer Group 단위)
- Partition은 병렬 처리 단위이며, 초기 설정 중요 (줄일 수 없음)
- 리플리케이션으로 장애 대응 (Partition 단위로 복제)
- Page Cache 활용으로 디스크 I/O 최소화

## 상세 내용

### Kafka 개요

Apache Kafka는 대용량 실시간 데이터를 처리하기 위한 고성능 **분산 메시징 플랫폼**입니다.

#### 주요 특징

- **고성능**: 높은 처리량, 낮은 지연시간
- **확장성**: 브로커를 수평 확장 가능
- **내구성**: 장애 발생 시에도 메시지 유실 없이 재처리 가능
- **고가용성**: 리플리케이션을 통해 안정성 확보
- **느슨한 결합**: 컨슈머 완전 분리
- **생태계**: Kafka Connect, Schema Registry 등 도구 제공

### Kafka 아키텍처

Kafka는 여러 구성 요소가 계층적으로 결합된 구조입니다.

> 구조 흐름: **Cluster → Broker → Topic → Partition → Segment → Record**

#### 주요 구성 요소

| 구성 요소 | 설명 |
|----------------|------|
| **Kafka Cluster** | 여러 브로커로 구성된 시스템 전체 |
| **Broker** | Kafka 서버 프로세스 |
| **Topic** | 메시지 스트림 단위 (논리 단위), 이메일 주소 정도의 개념 |
| **Partition** | 병렬 처리를 위한 단위 (물리 저장) |
| **Segment** | Partition 내부 로그 파일 |
| **Offset** | Partition 내 메시지의 위치 (64-bit) |
| **Producer** | Kafka로 메시지 전송 |
| **Consumer** | Kafka에서 메시지 소비 |
| **Zookeeper** | (기존 구조) 메타데이터 관리 및 브로커 상태 확인 |
| **KRaft** | (신규 구조) Zookeeper 없이 Kafka만으로 클러스터 관리 |

### Producer

Producer는 메시지를 Kafka에 전송하는 역할을 합니다.

#### 필수/선택 항목

- **필수**: Topic, Value
- **선택**: Key, Partition

#### 파티션 선택 전략

| 방식 | 설명 |
|-------------|------|
| Key 기반 | 같은 Key → 같은 Partition → 순서 보장 |
| Round-robin | 무작위 분산, 순서 보장 안 됨 (기본값) |
| 직접 지정 | 특정 Partition으로 지정 전송 |

#### 전송 보장 수준

- **At least once** (기본): 중복 가능
- **At most once**: 손실 가능성 존재
- **Exactly once**: PID + 메시지 번호로 중복 방지

#### 배치 전송

메시지를 모아서 한 번에 전송하여 네트워크 오버헤드를 감소시킵니다.

- 내부적으로 `send()` 후 파티션별로 모아서 배치 처리
- 성능 향상의 핵심 메커니즘

### Partition & Segment

#### Partition

Partition은 Topic을 병렬 처리 단위로 나눈 구조입니다.

특징:
- 병렬성과 처리량을 높이기 위한 핵심 구성
- **주의**: Partition 수는 줄일 수 없으며 초기 설정 중요
- 브로커를 확장하면 파티션들이 여러 브로커에 분산됨

권장 사항:
- 2~4개로 시작 후 점진적으로 조정
- Key 기반 전송 시 Partition 수 변경에 주의 (Hash 충돌)

#### Segment

Segment는 Partition 내부의 실제 저장 단위(로그 파일)입니다.

- 디스크 경로 예: `/data/kafka-logs/{topic}-{partition}/`
- 시간 또는 크기 기준으로 로테이션

### Consumer

Consumer는 Kafka에서 메시지를 소비하는 역할을 합니다.

#### Pull 방식

- Kafka는 **Pull 방식** (컨슈머가 직접 메시지 읽음)
- Offset을 기반으로 메시지 위치 추적
- **Consumer Group**으로 묶어 병렬 소비 가능

#### Consumer Group 특징

- 동일 Group ID를 가진 컨슈머들로 구성
- 하나의 Partition은 하나의 Consumer만 소비
- Group 내 컨슈머 수 ≤ Partition 수가 이상적
- 컨슈머 수가 파티션 수보다 많으면 일부 컨슈머는 대기

#### Offset 관리

- **Auto Commit**: 주기적으로 자동 커밋 (성능 좋지만 중복 가능)
- **Sync Commit**: 안전하지만 성능 저하
- **Async Commit**: 빠르지만 실패 시 재시도 안 함

### 고급 기능

#### 리플리케이션

Partition별 복제본을 유지하여 고가용성을 보장합니다.

설정:
```bash
--replication-factor 3  # 기본 3개
```

특징:
- 토픽 전체가 아닌 **파티션 단위로 복제**됨
- 장애 발생 시 다른 브로커로 대체 가능

역할:
| 역할 | 설명 |
|----------|------|
| **리더** | 모든 읽기/쓰기 처리 |
| **팔로워** | 리더 복제만 수행 |

#### 압축 전송

Kafka는 성능 최적화를 위해 메시지를 압축할 수 있습니다.

| 알고리즘 | 장점 |
|--------------|------|
| `gzip`, `zstd` | 높은 압축률 (CPU 사용 높음) |
| `lz4`, `snappy` | 빠른 전송 속도 (압축률 낮음) |

#### 페이지 캐시 (Page Cache)

Kafka는 디스크 직접 I/O 대신 **OS의 Page Cache**를 활용합니다.

장점:
- 디스크 I/O를 줄이고 메시지 처리 성능 향상
- OS 레벨 최적화 활용

### Spring Kafka

#### Retry 토픽과 Bean 생성

Spring Kafka에서는 `@RetryableTopic` 어노테이션 사용 시 자동으로 Retry용 `NewTopic` 빈을 생성합니다.

```kotlin
@RetryableTopic(
    attempts = "3",
    backoff = Backoff(delay = 1000)
)
@KafkaListener(topics = ["my-topic"])
fun listen(message: String) {
    // 처리 로직
}
```

동작 방식:
- 어노테이션 기반으로 `RetryTopicConfiguration` 생성
- `NewTopic` 빈을 자동으로 생성하여 브로커에 토픽 생성
- 같은 토픽에 대해 어노테이션이 2개 이상이면 중복 빈 에러 발생

주의사항:
- 브로커 설정 `auto.create.topic.enable=false`인 경우 자동 생성 안 됨
- 중복 생성 방지를 위해 `autoCreateTopics = false` 옵션 사용 가능

### FAQ

#### Q. 브로커 확장 시 토픽은 어떻게 되나요?

A. 파티션들을 여러 브로커에 나눔으로써, 하나의 토픽은 수평적으로 확장될 수 있고 이로 인해 브로커의 처리 능력보다 더 큰 성능을 제공할 수 있게 됩니다.

#### Q. 컨슈머 수가 파티션 수보다 많으면?

A. 일부 컨슈머는 대기 상태가 됩니다. 하나의 파티션은 하나의 컨슈머만 읽을 수 있기 때문입니다.

#### Q. 프로듀서도 파티션을 지정해서 보내나요?

A. 내부 파티셔너를 통해 자동으로 정해지거나, 직접 파티션을 지정해서 보낼 수 있습니다.

- Key 기반: 같은 Key는 같은 파티션으로 전송
- Round-robin: 파티션에 골고루 분산
- 직접 지정: 특정 파티션 번호 지정

## 실무 적용

### Kafka 설계 체크리스트

- [ ] Partition 수를 적절히 설정했는가? (초기 2~4개 권장)
- [ ] Consumer Group ID를 명확히 구분했는가?
- [ ] 순서 보장이 필요한 메시지는 Key 기반 전송을 사용하는가?
- [ ] 리플리케이션 팩터를 3 이상으로 설정했는가?
- [ ] Offset 커밋 전략을 명확히 했는가? (Auto vs Sync vs Async)

### 권장 설정

```yaml
# Producer 설정
spring:
  kafka:
    producer:
      acks: all  # 모든 리플리카가 수신 확인
      retries: 3  # 재시도 횟수
      batch-size: 16384  # 배치 크기
      compression-type: lz4  # 압축 알고리즘

# Consumer 설정
spring:
  kafka:
    consumer:
      enable-auto-commit: false  # 수동 커밋 권장
      auto-offset-reset: earliest  # 초기 Offset 전략
      max-poll-records: 500  # 한 번에 가져올 메시지 수
```

### 금융권에서의 고려사항

- **정확히 한 번 전송 (Exactly Once)**: 금액 관련 메시지는 중복 방지 필수
- **순서 보장**: 거래 순서가 중요한 경우 Key 기반 파티셔닝 사용
- **감사 로그**: 모든 메시지 전송/수신 로그 기록
- **장애 대응**: 리플리케이션 팩터 최소 3, 브로커 분산 배치
- **보안**: SSL/TLS 암호화, SASL 인증 필수
- **데이터 보관**: 민감 정보는 암호화 후 전송, 보관 기간 정책 수립

### Kafka 발전 히스토리

- 2013년: 리플리케이션 기능 도입 (내구성 강화)
- Schema Registry 도입 → 데이터 스키마 합의
- Kafka Connect 도입 → 다양한 시스템과 연결 가능
- KRaft 구조 도입 → Zookeeper 의존도 제거 (Kafka 2.8+)

---

**출처**
- Apache Kafka Quickstart
- Spring for Apache Kafka Reference

