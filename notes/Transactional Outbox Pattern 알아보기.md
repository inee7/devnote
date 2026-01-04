---
tags: [kafka, event, transactional-outbox, distributed-system, microservice, cdc]
---
# Transactional Outbox Pattern 알아보기
![[resources/images/transactional-outbox-01.png]]

*메시지를 DB 트랜잭션과 묶어서 커밋하여 비동기적으로 메시지 브로커에게 전달하는 패턴*

## Overview

이벤트로 기반으로 분리된 분산환경에서 우리는 특정 DB 상태를 변경하는 트랜잭션과 함께 이벤트를 발행해야할 때가 있습니다. 가령 웹 서비스에 회원 탈퇴 요청이 왔을 때 회원의 탈퇴가 발생했다는 이벤트를 발행시켜 해당 메시지를 구독하는 서비스들에게 회원 탈퇴 이벤트를 전달하는 경우입니다. 회원 탈퇴 이벤트를 구독하는 서비스들은 탈퇴된 회원에게 메일을 보낸다던지, 적재된 회원의 개인정보를 삭제하는 로직을 진행시킨다던지, 회원이 작성한 게시글을 비공개를 돌린다던지 목표하는 비즈니스 로직을 진행시킵니다.

하지만 이벤트로 분리된 애플리케이션 환경에서는 고민해야할 문제가 있습니다. 메시지 발행과 트랜잭션 커밋 시점에 대한 문제입니다. 애플리케이션 로직상 트랜잭션이 완료되기 전에 이벤트 메시지를 발행하게 됩니다. 그러면 그 이후에 비즈니스 로직에서 예외가 발생하거나 DB에 쿼리를 날린 이후에 특정한 이유 때문에 예외가 발생해 롤백이 될 수 있습니다. 이 경우에 트랜잭션이 성공하지 않았는데 메시지가 발행이 되는 경우가 생길 수 있고 시스템의 안정성이 보장되지 못합니다.

글에서는 우리가 마주할 수 있는 이 문제를 정의하고 이 문제를 해결하기 위해 이전에 사용했던 분산 트랜잭션의 문제점과 이를 해결하는 방법 중 하나인 **Transaction Outbox Pattern**에 대해서 소개하겠습니다.

## 트랜잭션널 이벤트(Transactional Event)

우리가 개발하는 서비스는 보통 DB를 업데이트하는 트랜잭션과 함께 메시지를 발행합니다. 이 때 DB를 업데이트와 메시지 전송을 한 트랜잭션으로 묶지 않으면 문제가 발생합니다. 이 두 작업은 하나의 애플리케이션 서비스 내에서 원자적으로 수행되지 않으면 시스템이 실패할 경우 문제가 발생하게 되는 것입니다.

[2PC(two-phase commit)](https://ebrary.net/64872/computer_science/introduction_phase_commit) 

기존에는 이 원자성을 보장하기 위해 분산 트랜잭션을 사용했습니다. 분산 트랜잭션은 [2PC(two-phase commit, 2단계 커밋)](https://github.com/eastperson/TIL/blob/main/Database/2PC(two-phase%20commit).md)을 이용하여 트랜잭션 참여자가 커밋 혹은 롤백을 할 수 있도록 원자성을 보장하는 방법입니다. 분산 트랜잭션은 단일 db에 비해서 10배 이상의 성능 저하가 발생할 수 있습니다. 또한 동기 IPC(Inter-Process Communication) 형태라 가용성이 떨어지는 문제가 있습니다. 현대의 아키텍처는 일관성보다 가용성을 우선시하고 있습니다. 이러한 이유로 분산 트랜잭션 방식은 현대 애플리케이션과 맞지 않으며 최신 메시지 브로커(카프카 등)는 기능 자체를 지원하지 않습니다.

![[resources/images/transactional-outbox-02.png]]

우리는 다음 2가지를 보장해야합니다.

* 데이터베이스 트랜잭션이 커밋되면 메시지가 발행되어야 합니다. 반대로 트랜잭션이 롤백되면 메시지를 보내지 않아야 합니다.
* 메시지 서비스는 보낸 순서대로 브로커로 보내야 합니다.

⠀
## Transaction Outbox Pattern

이 문제를 해결하기 위한 방법으로 우리는 DB를 업데이트하는 트랜잭션의 일부로 데이터베이스에 메시지를 저장하는 방법이 있습니다. 그런 다음에 별도의 프로세스가 저장된 이벤트를 읽어 메시지 브로커에 전송하는 것입니다. 이 방법이 Outbox Pattern 입니다.

애플리케이션은 데이터베이스의 outbox 테이블에 메시지 내용을 저장합니다. 다른 애플리케이션이나 프로세스는 outbox 테이블에서 데이터를 읽고 해당 데이터를 사용하여 작업을 수행할 수 있습니다. 실패시 완료될 때까지 다시 시도할 수 있습니다. 따라서 outbox pattern은 적어도 한 번 이상(at-least once) 메시지가 성공적으로 전송되었는지 확인할 수 있습니다.

![[resources/images/transactional-outbox-03.png]] ![[resources/images/transactional-outbox-04.png]] [https://en.wikipedia.org/wiki/Inbox_and_outbox_pattern](https://en.wikipedia.org/wiki/Inbox_and_outbox_pattern) 

여기서 outbox는 '보낼 편지함'이라는 뜻이 있습니다. 전송되지 않았거나 전송에 실패한 메시지들이 모여있는 보관함이라는 뜻입니다. 메시지를 보낼 데이터를 저장하는 저장소를 따로 두는 것이죠. 이제는 크리스 리처드슨(Chris Richardson's)의 [microservices.io](http://microservices.io) 에서 정의하는 아웃박스 패턴을 보겠습니다.
![[resources/images/transactional-outbox-05.png]]

이 패턴의 구성요소는 다음과 같습니다.

* Sender - 메시지를 보내는 서비스
* Database - 엔티티 및 메시지 outbox를 저장하는 데이터베이스
* Message outbox - 관계형 데이터베이스인 경우 보낼 메시지를 저장하는 테이블. NoSQL의 경우 각 데이터베이스 record(document or item)의 프로퍼티
* Message relay - outbox에 저장된 메시지를 메시지 브로커로 보내는 서비스

⠀
이 패턴에서는 Message Relay라는 별도의 프로세스가 추가됐습니다. outbox 테이블은 임시 메시지 큐 역할을 하며 엔티티 업데이트와 함께 트랜잭션으로 묶입니다. Message Relay는 outbox 테이블에 저장하는 데이터를 비동기적으로 읽어서 메시지를 발행하여 메시지 브로커에게 전달하는 역할을 하게 됩니다. outbox pattern의 Message Relay을 구현하는데는 Polling publisher, Transaction log tailing 두 가지 방식이 있습니다.

## 폴링 발행기 패턴(Polling Publisher Pattern)

RDBMS를 사용하는 애플리케이션에서 outbox 테이블에 삽입된 메시지를 발행하는 간단한 방법으로 테이블을 폴링해서 미발행된 메시지를 조회하는 것입니다. 메시지 릴레이는 이렇게 조회한 메시지를 하나씩 각자의 목적지 채널로 보내서 메시지 브로커에 발행합니다. 그리고 나중에 outbox 테이블에서 메시지를 삭제합니다.

![](resources/images/transactional-outbox-06.png)
[https://krishnakrmahto.com/transactional-messaging-in-microservices](https://krishnakrmahto.com/transactional-messaging-in-microservices)

DB 폴링은 규모가 작을 경우 쓸 수 있는 쉬운 방법입니다. 하지만 DB를 자주 폴링하면 비용이 발생하고 NoSQL DB는 쿼리 능력에 따라 사용 가능 여부가 결정됩니다.

## 트랜잭션 로그 테일링 패턴(Transaction log tailing Pattern)

메시지 릴레이로 DB 트랜잭션 로그(커밋 로그)를 테일링(tailing)하는 방법입니다. 애플리케이션에서 커밋된 업데이트는 각 DB의 트랜잭션 로그 항목(log entry, 로그 엔트리)으로 남습니다. 트랜잭션 로그 마이너(transaction log miner)로 트랜잭션 로그를 읽어 변경분을 하나씩 메시지로 메시지 브로커에 발행하는 방법입니다.
![](resources/images/transactional-outbox-07.png)
[https://microservices.io/patterns/data/transaction-log-tailing.html](https://microservices.io/patterns/data/transaction-log-tailing.html)

트랜잭션 로그 마이너는 트랜잭션 로그 항목을 읽고 삽입된 메시지에 대응되는 각 로그 항목을 메시지로 전환하여 메시지 브로커에 발행합니다. RDBMS의 outbox 테이블에 출력된 메시지 또는 NoSQL DB에 레코드에 추가된 메시지를 발행할 수 있습니다.

mysql의 경우 mysqlbinlog, PostgreSQL WAL, Oracle redolog 등을 활용하여 변경사항을 읽어서 구현할 수 있습니다. 구현 난이도가 높아 관련 툴을 사용하는 경우가 많습니다. 관련 툴은 디비지움(Debezium), 링크드인 데이터 버스(LinkdIn Databus), DynamoDB 스트림즈, 이벤추에이트 트램 등이 있습니다.

## Outbox Pattern with Kafka Connect

Kafka-Connect는 Kafka 브로커 외에 별도의 서비스로 실행됩니다. 아래의 그림에서는 Postgres 데이터 베이스를 사용하였는고 엔티티 업데이트가 발생할 때 oubox 테이블에 레코드를 추가하는 모습입니다. Kafka-Connect 는 런타임시점에서 데이터베이스의 변경사항을 캡쳐하기 위해 Debezium 커넥터가 배포됩니다. Debezium 커넥터는 outbox 테이블에서 데이터베이스 트랜잭션 로그(write ahead log)를 추적하고 사용자 정의 커넥터에 의해 정의된 토픽에 메시지를 발행합니다.

![](resources/images/transactional-outbox-08.png)
[https://dzone.com/articles/implementing-the-outbox-pattern](https://dzone.com/articles/implementing-the-outbox-pattern)

이 방법은 적어도 한 번(at-least once) 전달을 보장합니다. 커넥터가 다운되고 시작될 때 동일한 이벤트를 여러 번 게시할 때가 있습니다. 따라서 consumer는 멱등성 상태여야 하며 중복 이벤트가 다시 처리되지 않도록 해야합니다.

이와같이 로그의 변경된 데이터를 감지하는 작업을 **CDC(Change Data Capture)**라고 합니다. Debezium MySQL Connector는 bonlog를 읽어 INSERT, UPDATE, DELETE 연산에 대한 변경 이벤트를 만들어 Kafka 토픽으로 전송해줍니다. 따라서 DB에서 수행된 모든 이벤트가 안정적으로 수집되고 이벤트 발행시 정확한 순서가 보장됩니다.

## Conclusion

아웃박스 패턴은 일관성이 필요하고 요청을 정확하게 포착해야하는 데이터로 작업하는 경우 필요합니다. 또한 데이터베이스 업데이트와 메시지 전송의 원자성(Atomic)을 보장하기 위해서 사용합니다. 주로 금융 비즈니스에서 트랜잭션 이벤트(Transactional Event)를 다루거나 외부 시스템과 결제 등의 메시지를 전달을 보장할 때가 주로 그 경우죠. 또한 메시지의 최소 한 번 전달을 보장하였습니다. 이 방법을 쓰면서 비용이 높은 2PC를 사용하지 않았습니다.

로그의 변경감지를 캡처하여 메시지를 발행하는 Tailing 기법을 자세히 알고 싶으면 CDC(Change Data Capture)를 최소 1회 보장(ALO, At-Least Once) 등의 메시지 전달에 대한 내용을 알고 싶으시면 메시지 전달 보장(Message Delivery Gurantee) 키워드를 통해 더 깊은 내용을 학습하실 수 있습니다.

## Reference

[Transactional Outbox Pattern 알아보기](https://velog.io/@eastperson/Transaction-Outbox-Pattern-%EC%95%8C%EC%95%84%EB%B3%B4%EA%B8%B0)

