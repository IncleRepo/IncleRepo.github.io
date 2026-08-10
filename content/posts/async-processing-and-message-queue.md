+++
title = '@Async는 언제 충분하고 언제 메시지 큐가 필요한가'
slug = '12'
aliases = ['/posts/012/']
date = 2026-05-12T19:30:00+09:00
lastmod = 2026-08-10T16:00:27+09:00
draft = false
description = '느린 부수 작업을 분리하는 @Async부터 작업 유실, Kafka, Transactional Outbox, 순서와 중복 처리까지 단계적으로 알아봅니다.'
categories = ['비동기 처리']
tags = ['Spring', 'Async', 'Kafka', '메시지 큐']
+++

주문 API가 다음 순서로 동작한다고 가정해 보자.

```text
POST /orders

1. 주문 저장
2. 주문 완료 이메일 발송
3. HTTP 응답
```

주문 저장에는 50ms가 걸리지만 이메일 서버의 응답에는 3초가 걸린다.

```text
주문 저장        50ms
이메일 발송    3,000ms
HTTP 응답
```

주문이 이미 정상적으로 저장됐다면 사용자가 이메일 전송까지 3초 동안 기다릴 필요가 있을까? 이메일이 주문 성공의 필수 조건이 아니라면 요청 처리와 이메일 발송을 분리할 수 있다. 여기서 비동기 처리가 시작된다.

## 비동기는 작업을 빠르게 하지 않고 기다림을 옮긴다

동기 방식에서는 이메일 발송이 끝나야 다음 단계로 이동한다.

```text
HTTP 요청
  ↓
주문 저장
  ↓
이메일 발송 3초
  ↓
HTTP 응답
```

비동기 방식에서는 이메일 작업을 다른 실행 흐름에 넘긴 뒤 요청에 먼저 응답할 수 있다.

```text
HTTP 요청
  ↓
주문 저장
  ↓
이메일 작업 제출 ─────→ 별도 Thread에서 이메일 발송 3초
  ↓
HTTP 응답
```

이메일 전송 시간이 3초에서 0초가 된 것은 아니다. 줄어든 것은 사용자의 대기 시간이고, 실제 이메일 처리 시간과 필요한 Network·외부 시스템 처리량은 그대로다. 비동기는 작업량을 없애는 기술이 아니라 **기다림의 위치를 바꾸는 기술**이다.

이 지점을 이해하면 동기·비동기와 Blocking·Non-Blocking도 구분하기 쉬워진다. 일반적인 Spring MVC 요청은 다음처럼 흘러간다.

```text
요청 Thread
  ↓
DB 호출
  ↓
DB 응답을 기다림
  ↓
다음 코드 실행
```

여기에는 서로 다른 두 질문이 있다.

- 작업이 끝날 때까지 호출 흐름이 결과를 기다리는가? 이는 동기와 비동기를 나누는 기준이다.
- 결과를 기다리는 동안 현재 Thread가 진행하지 못하는가? 이는 Blocking과 Non-Blocking을 나누는 기준이다.

한 요청 Thread가 DB 응답을 기다린다고 서버 전체가 멈추는 것은 아니다. 다른 요청은 다른 Thread에서 처리할 수 있다.

```text
Request 1 → Thread 1 → DB 응답 대기
Request 2 → Thread 2 → 별도 처리
```

다만 기다리는 요청이 계속 늘어나면 사용 가능한 요청 Thread가 줄어들고, 결국 새 요청까지 대기할 수 있다. 따라서 느린 부수 작업을 요청 Thread에서 분리하면 사용자 응답 시간을 줄이고 요청 Thread를 더 빨리 반환할 수 있다.

아래 그림에서는 두 축을 한 번에 외우기보다, **작업 완료를 누가 기다리는지**와 **기다리는 동안 현재 Thread가 멈추는지**를 나누어 보면 된다.

![동기·비동기와 Blocking·Non-Blocking의 조합](/images/posts/async-processing-and-message-queue/legacy-01.png "동기·비동기와 Blocking·Non-Blocking은 서로 다른 기준이다")

그러면 Spring에서는 이메일 같은 부수 작업을 가장 간단하게 어떻게 분리할 수 있을까?

## `@Async`는 작업을 다른 Thread에 넘긴다

Spring에서는 메서드에 `@Async`를 붙여 비동기 작업으로 실행할 수 있다.

```java
@Async("notificationExecutor")
public void sendOrderCompleted(long orderId) {
    mailClient.sendOrderCompleted(orderId);
}
```

Annotation 하나를 붙였다고 Java 메서드 안에서 갑자기 Thread가 바뀌는 것은 아니다. 다른 Spring Bean이 이 메서드를 호출하면 앞에 놓인 Spring Proxy가 호출을 먼저 받는다. Proxy는 메서드를 직접 실행하지 않고 실행할 작업을 `TaskExecutor`에 제출한다.

```text
Caller
  ↓
Spring Proxy
  ↓
TaskExecutor에 작업 제출
  ↓
Caller는 먼저 복귀

Worker Thread
  ↓
실제 메서드 실행
```

각 구성 요소의 역할은 다르다.

- **Proxy**는 `@Async`가 붙은 호출을 가로챈다.
- **TaskExecutor**는 비동기 작업을 실행할 Thread Pool을 관리한다.
- **Queue**는 Worker Thread가 바로 처리하지 못한 작업을 잠시 보관한다.

내부적으로 메서드 호출은 `Runnable`이나 `Callable` 형태의 작업으로 Executor에 제출된다. 호출자는 작업을 맡긴 뒤 먼저 돌아가고, Worker Thread가 나중에 실제 메서드를 실행한다.

### 같은 클래스에서 호출하면 왜 동작하지 않을까

Proxy를 거쳐야 한다는 사실은 `@Async`가 적용되지 않는 대표적인 상황도 설명해 준다.

```java
@Service
public class MailService {

    public void orderCompleted() {
        sendMail();
    }

    @Async
    public void sendMail() {
        // 이메일 발송
    }
}
```

다른 Bean에서 호출하면 Bean 앞의 Proxy를 지나간다.

```text
다른 Bean → Proxy → sendMail()
```

하지만 같은 객체 안의 `sendMail()` 호출은 사실상 `this.sendMail()`이다.

```text
this.sendMail() → Proxy를 다시 지나지 않음 → 현재 Thread에서 실행
```

같은 클래스 내부 호출에서 `@Async`가 적용되지 않는 이유는 Annotation이 잘못되어서가 아니라 Proxy를 통과하지 않았기 때문이다. 비동기 작업을 별도 Spring Bean으로 분리하는 방식이 가장 이해하기 쉽고 안전하다.

### `void` 작업의 예외는 원래 요청으로 돌아가지 않는다

비동기 메서드가 `void`를 반환한다면 Worker Thread에서 발생한 예외를 원래 요청에 돌려줄 수 없다.

```text
요청 Thread                 Worker Thread

Async 작업 제출
  ↓
HTTP 응답
  ↓
요청 종료                    나중에 작업 실행
                               ↓
                            Exception 발생
```

예외가 발생했을 때 원래 요청은 이미 끝났을 수 있다. `Future`나 `CompletableFuture`를 반환하면 호출자가 완료와 예외를 관찰할 수 있지만, Fire-and-forget 형태의 `void` 작업은 로그·모니터링·실패 저장·재시도 정책을 따로 마련해야 한다.

작업을 다른 Thread로 넘기는 것만으로 충분하다면 `@Async`는 간단하고 유용하다. 하지만 이메일 작업 500개가 한꺼번에 들어오면 어디에서 기다리게 될까?

## 작업이 몰리면 Thread Pool과 Queue에 쌓인다

다음은 비동기 이메일 작업에 사용할 Executor 설정 예시다.

```java
@Bean
public ThreadPoolTaskExecutor notificationExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(4);
    executor.setMaxPoolSize(8);
    executor.setQueueCapacity(200);
    executor.setThreadNamePrefix("notification-");
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    executor.initialize();
    return executor;
}
```

작업은 다음 순서로 받아들여진다.

```text
corePoolSize까지 Worker 사용
  ↓ 모두 바쁨
Queue에 작업 저장
  ↓ Queue도 가득 참
maxPoolSize까지 Worker 추가
  ↓ Thread와 Queue가 모두 가득 참
RejectedExecutionHandler 실행
```

각 설정의 의미는 이 흐름에 이름을 붙인 것이다.

- `corePoolSize`는 평소 유지하며 먼저 사용할 Worker 수다.
- `queueCapacity`는 Worker가 즉시 실행하지 못한 작업이 기다릴 수 있는 수다.
- `maxPoolSize`는 Queue까지 가득 찼을 때 늘릴 수 있는 최대 Worker 수다.
- `RejectedExecutionHandler`는 더는 작업을 받을 수 없을 때의 처리 방법이다.

위 예제의 `CallerRunsPolicy`는 작업을 버리는 대신 제출한 Thread가 직접 실행하게 한다. 요청 Thread가 제출자라면 포화 상태에서 요청이 다시 느려질 수 있다. 즉 Rejection 정책도 시스템의 부하 전달 방식을 바꾸는 중요한 선택이다.

### `maxPoolSize`만 높인다고 Thread가 바로 늘지는 않는다

다음 설정을 생각해 보자.

```text
corePoolSize = 4
maxPoolSize = 100
queueCapacity = 10000
```

작업이 조금 몰렸을 때 Executor는 100개의 Thread를 바로 만들지 않는다. Core Worker 4개가 바쁘면 나머지 작업을 큰 Queue에 먼저 넣는다. Queue가 가득 차기 전까지는 `maxPoolSize`를 향해 Worker를 늘리지 않는다.

따라서 `maxPoolSize=100`이라는 숫자만 보고 동시 작업 100개가 곧바로 실행된다고 판단하면 안 된다. 반대로 Queue가 너무 작으면 순간적인 요청 증가에도 Thread가 빠르게 늘고 Rejection이 발생할 수 있다.

### Thread 수는 작업과 상대 시스템을 함께 보고 정한다

CPU 계산이 중심인 작업에서 Thread를 지나치게 늘리면 CPU Core보다 많은 Thread가 실행 기회를 다투면서 Context Switching 비용만 커질 수 있다. 반면 이메일이나 외부 API 호출처럼 기다리는 시간이 긴 I/O 작업은 CPU 작업보다 더 많은 Thread를 활용할 여지가 있다.

하지만 외부 Mail Server가 동시에 20개 요청만 처리할 수 있다면 Application Thread를 200개로 늘려도 병목이 사라지지 않는다. 오히려 외부 시스템에 더 많은 요청을 한꺼번에 보내 장애를 키울 수 있다.

Thread Pool 크기는 애플리케이션만 보고 정하는 숫자가 아니다. 작업의 CPU·I/O 성격, 평균 처리 시간, 요청 도착 속도, 외부 시스템이 감당할 수 있는 동시 요청 수를 함께 봐야 한다.

이제 더 근본적인 질문이 남는다. Worker가 아직 이메일 작업을 처리하지 못했다면 그 작업은 어디에 있을까?

## `@Async` 작업은 애플리케이션 메모리에 머문다

대기 중인 `@Async` 작업은 보통 Application Process 내부의 Thread Pool Queue에 있다.

```text
Application Process

TaskExecutor
  ├─ Worker Thread
  └─ Memory Queue
       ├─ 이메일 작업 A
       ├─ 이메일 작업 B
       └─ 이메일 작업 C
```

애플리케이션이 정상 종료될 때 대기 작업을 기다리도록 설정할 수는 있다. 그러나 프로세스가 강제로 종료되거나 서버가 갑자기 사라지면 실행 중이거나 대기 중인 작업은 유실될 수 있다. 메모리 Queue는 작업을 영구적으로 보존하는 저장소가 아니기 때문이다.

짧은 부수 작업이고 잃어도 다시 만들 수 있다면 이 단순함이 장점이다. 반대로 주문 후속 처리처럼 반드시 실행해야 한다면 작업을 애플리케이션 프로세스 밖에 남길 방법이 필요하다.

## 메시지 큐는 작업을 프로세스 밖에 남긴다

메시지 큐는 더 빠른 `@Async`가 아니다. 핵심 차이는 실행 Thread가 아니라 작업을 남기는 위치다.

```text
@Async
→ 작업이 Application Memory에 머묾

Message Queue
→ 작업이 Application 밖 Broker에 기록됨
```

가장 단순한 주문 완료 흐름부터 보자.

```text
주문 서비스
  ↓ "주문 완료" Event 발행
Kafka
  ↓ Event 전달
이메일 Consumer
```

이 구조에 이름을 붙이면 다음과 같다.

- **Producer**는 메시지를 보내는 쪽이다.
- **Broker**는 메시지를 저장하고 Consumer에게 제공하는 중간 서버다.
- **Topic**은 같은 목적의 메시지를 모아두는 논리적인 이름이다.
- **Consumer**는 메시지를 읽어 실제 업무를 수행하는 쪽이다.

아래 그림에서는 메시지가 Producer에서 Broker의 Topic으로 들어가고 Consumer가 읽어 가는 큰 흐름을 먼저 보면 된다. Partition과 Consumer Group은 뒤에서 따로 살펴본다.

![Kafka의 Producer, Broker, Topic, Partition, Consumer 구조](/images/posts/async-processing-and-message-queue/legacy-02.png "Kafka 기본 구조")

Spring 애플리케이션에서는 Spring Kafka 의존성을 추가하고 Broker 주소를 설정한 뒤 `KafkaTemplate`로 메시지를 발행할 수 있다. 아래 세 이미지는 각각 **의존성 추가**, **Broker 연결 설정**, **Event 발행 코드**가 어느 부분에 해당하는지 보여 준다.

![Spring Kafka 의존성 구성](/images/posts/async-processing-and-message-queue/legacy-03.png "Spring Kafka 의존성")

![Kafka Broker 연결 설정](/images/posts/async-processing-and-message-queue/legacy-04.png "Kafka Broker 연결 설정")

![KafkaTemplate을 사용한 메시지 발행 예시](/images/posts/async-processing-and-message-queue/legacy-05.png "Spring에서 Kafka 메시지 발행")

Broker가 메시지를 보관하므로 Consumer가 잠시 중단되어도 보존 기간 안의 메시지를 다시 읽어 이어서 처리할 수 있다. 어디서부터 다시 읽을지는 뒤에서 살펴볼 Offset으로 관리한다. 다만 Broker에 보냈다는 사실만으로 무조건 안전한 것은 아니며 Producer의 확인 설정, 복제와 보존 정책이 요구 수준에 맞아야 한다.

메시지 큐는 순간적인 속도 차이도 흡수할 수 있다.

```text
Producer: 초당 1,000건
Consumer: 초당   500건

차이: 초당 500건이 Broker에 쌓임
```

갑작스러운 요청 증가가 짧게 끝나고 Consumer가 나중에 따라잡는다면 Broker가 완충 역할을 한다. 그러나 이 차이가 계속되면 밀린 메시지가 초당 500건씩 늘어난다. 메시지 큐는 처리량을 없애는 기술이 아니며, 장기적으로는 소비 처리량을 생산 처리량 이상으로 확보해야 한다.

그렇다면 같은 주문 완료 Event를 이메일과 통계가 모두 처리하려면 어떻게 해야 할까?

## Consumer Group은 처리 목적과 병렬 작업을 나눈다

`order-completed` Event를 이메일과 통계가 각각 받아야 한다고 가정하자.

```text
Email Consumer Group
→ 주문 완료 Event를 받아 이메일 발송

Statistics Consumer Group
→ 같은 주문 완료 Event를 받아 통계 반영
```

서로 다른 Consumer Group은 같은 Event를 각각 읽는다. 반대로 이메일 발송 처리량을 높이기 위해 Consumer를 세 대 실행한다면 같은 Group에 둔다.

```text
Email Consumer 1 ┐
Email Consumer 2 ├─ 같은 Email Consumer Group
Email Consumer 3 ┘  → 이메일 작업을 나누어 처리
```

Consumer Group은 같은 업무를 여러 Consumer가 나누어 처리하기 위한 묶음이라고 볼 수 있다. 그러면 Consumer 세 대는 하나의 Topic에서 어떤 메시지를 누가 가져갈까?

### Partition은 저장과 병렬 처리의 단위다

Kafka는 Topic을 여러 Partition으로 나눌 수 있다.

```text
order-completed Topic

Partition 0
Partition 1
Partition 2
```

같은 Consumer Group 안에서는 하나의 Partition을 동시에 둘 이상의 Consumer가 나누어 읽지 않는다. Kafka가 Partition을 Group의 Consumer에게 할당하고, Consumer는 자신이 맡은 Partition을 처리한다. Consumer 수가 Partition 수보다 많으면 남는 Consumer는 할당받을 Partition이 없어 쉬게 된다.

### Offset은 다시 읽을 위치를 나타낸다

Partition에 메시지가 다음처럼 저장되어 있다고 하자.

```text
Offset     0  1  2  3  4  5  6 ...
처리 완료  ─────────────┘
```

Consumer가 0번부터 4번까지 처리한 뒤 재시작하면 어디서부터 이어야 할까? Kafka Consumer는 Group별로 Commit한 Offset을 사용해 다시 읽을 위치를 관리한다. 정확히는 Commit한 값이 일반적으로 **다음에 읽을 메시지의 위치**를 뜻한다.

이제 Consumer 장애는 Broker와 Offset으로 복구할 수 있다. 하지만 Producer가 업무 DB를 바꾼 뒤 Broker에 Event를 남기기 전에 죽으면 어떻게 될까?

## DB 변경과 Event 발행 사이에는 틈이 있다

주문 취소는 보통 두 작업으로 이루어진다.

```text
1. DB에서 주문 상태를 취소로 변경
2. Kafka에 주문 취소 Event 발행
```

첫 번째는 Database, 두 번째는 Kafka라는 서로 다른 시스템에서 실행된다. DB를 먼저 Commit하면 다음 문제가 생길 수 있다.

```text
DB Commit 성공
  ↓
Application 종료 또는 Kafka 발행 실패
  ↓
DB에는 취소됐지만 Consumer는 취소 사실을 모름
```

Kafka에 먼저 발행해도 반대 문제가 생긴다.

```text
Kafka Event 발행
  ↓
DB Transaction Rollback
  ↓
Consumer는 취소됐다고 처리했지만 DB에는 취소되지 않음
```

일반적인 하나의 DB Transaction만으로 DB와 Kafka를 함께 원자적으로 Commit할 수 없기 때문에 생기는 틈이다. 그렇다면 Kafka 발행 자체가 아니라 **나중에 발행해야 할 Event가 있다는 사실**까지 DB 변경과 함께 저장할 수는 없을까?

## Transactional Outbox는 다시 시작할 위치를 남긴다

주문 상태와 발행할 Event 기록을 같은 DB Transaction에 저장한다.

```text
하나의 DB Transaction

주문 상태 변경
       +
Outbox INSERT
       ↓
     Commit
```

Commit이 끝난 뒤 별도 Publisher가 Outbox에서 아직 발행하지 않은 Event를 읽어 Kafka로 보낸다.

```text
Outbox Publisher
  ↓ Pending Event 조회
Kafka 발행
  ↓ 성공 처리
Outbox 상태 갱신 또는 삭제
```

이 구조를 **Transactional Outbox**라고 한다. DB 변경과 Kafka 발행을 하나의 Transaction으로 만드는 것이 아니다. 업무 상태와 “이 Event를 나중에 발행해야 한다”는 기록을 같은 DB Transaction에서 확정하는 방식이다.

Publisher가 실패해도 Pending Outbox가 DB에 남아 있으므로 다시 조회해 발행할 수 있다. Outbox의 핵심은 Event를 Kafka에 반드시 한 번만 보내는 데 있지 않다. **실패한 발행을 어디서 다시 시작할지 남기는 것**에 있다.

Outbox를 전달하는 방법은 크게 두 가지다.

- **Polling**은 애플리케이션이 일정 주기로 Pending Outbox를 조회해 발행한다. 이해하고 운영하기 쉽지만 조회 주기와 동시 처리 방식을 설계해야 한다.
- **CDC**는 Change Data Capture의 약자로, Debezium 같은 Connector가 DB 변경 로그를 읽어 Outbox INSERT를 Kafka로 전달하는 방식이다. 애플리케이션의 반복 조회를 줄일 수 있지만 Connector라는 별도 운영 대상이 생긴다.

여러 Outbox와 Connector로 나누면 처리량과 장애 격리에는 도움이 될 수 있다. 하지만 같은 업무 흐름의 Event가 서로 다른 경로를 지나면 도착 순서를 맞추기 어려워질 수 있다. 같은 Aggregate의 Event 순서가 중요하다면 하나의 일관된 발행 경로와 Key 전략을 함께 설계해야 한다.

Outbox가 재발행할 수 있다는 것은 같은 Event가 두 번 전달될 수도 있다는 뜻이다. 중복을 보기 전에 먼저 Kafka가 보장하는 순서의 범위를 확인해 보자.

## Kafka의 순서는 Partition 안에서만 유지된다

Kafka는 Topic 전체의 전역 순서를 보장하지 않는다. 같은 Partition에 기록된 메시지의 순서만 유지한다.

```text
Partition 0: 결제 완료 → 주문 취소
Partition 1: 다른 주문의 결제 완료 → 배송 시작
```

같은 주문의 Event 순서가 중요하다면 `orderId`처럼 동일한 업무 ID를 메시지 Key로 사용해 같은 Partition으로 보내는 방식을 검토할 수 있다.

```text
Key = order-100

결제 완료(order-100)
  ↓ 같은 Partition
주문 취소(order-100)
```

서로 다른 주문 사이의 전역 순서를 맞추지 않아도 되므로 여러 Partition에서 병렬 처리할 수 있다. 여기서 중요한 것은 “Kafka가 알아서 업무 순서를 보장한다”가 아니라 **어떤 Event들이 순서를 공유해야 하는지 Key로 표현하는 것**이다.

## 다시 처리할 수 있다면 중복도 생길 수 있다

Consumer가 메시지를 처리하고 Offset을 Commit한다고 가정하자.

```text
1. Event 수신
2. 이메일 발송 성공
3. Offset Commit
```

2번은 성공했지만 3번 전에 Consumer가 종료되면 Kafka는 처리 완료 사실을 알지 못한다. Consumer가 재시작하면 같은 Event를 다시 받을 수 있다.

```text
이메일 발송 성공
  ↓
Offset Commit 전 장애
  ↓
재시작 후 같은 Event 재수신
```

이는 유실 가능성을 낮추는 대신 중복 가능성을 허용하는 **at-least-once** 처리에서 자연스럽게 생기는 Trade-off다. 한 번도 처리되지 않는 상황은 최대한 피하지만, 한 번 이상 처리될 수는 있다.

### 같은 Event가 다시 와도 결과가 중복되지 않게 만든다

`event-123`을 처음 처리한 뒤 Event ID를 기록했다고 하자.

```text
첫 수신
→ 알림 처리
→ event-123 처리 기록 저장

재수신
→ event-123 이미 처리됨
→ 건너뜀
```

같은 작업을 여러 번 요청해도 최종 결과가 한 번 실행한 것과 같도록 만드는 성질을 멱등성(Idempotency)이라고 한다.

```java
@KafkaListener(topics = "order-completed", groupId = "notification")
public void consume(OrderCompletedEvent event) {
    if (processedEventRepository.existsById(event.eventId())) {
        return;
    }

    notificationService.send(event.orderId());
    processedEventRepository.save(new ProcessedEvent(event.eventId()));
}
```

다만 이 코드는 개념을 보여 주는 단순한 예시일 뿐 완전한 보호 장치는 아니다.

```text
1. 처리 기록 없음 확인
2. 이메일 발송 성공
3. 처리 기록 저장 전에 장애
```

재시작 후 같은 Event를 받으면 이메일이 두 번 발송될 수 있다. 업무 DB 변경과 처리 기록이라면 같은 Transaction과 Unique Constraint로 묶을 수 있다. 이메일·결제처럼 외부 Side Effect가 있다면 외부 시스템이 제공하는 Idempotency Key를 사용하거나, 별도의 발송 상태와 재처리 절차를 설계해야 한다.

Kafka의 Exactly-Once 기능도 외부 이메일이나 모든 DB 변경을 자동으로 정확히 한 번 실행해 주는 약속은 아니다. Broker 안의 읽기·처리·쓰기 보장과 외부 시스템의 Side Effect는 구분해서 봐야 한다.

중복을 감당하도록 만들었다고 모든 실패를 무한정 재시도해도 되는 것은 아니다.

## 실패를 무한히 재시도하면 정상 작업도 막힌다

Database 장애로 Message A가 실패했다고 가정하자.

```text
Message A 실패
  ↓ 즉시 Retry
실패
  ↓ 즉시 Retry
실패
  ↓ 무한 반복
```

이 방식은 CPU와 Consumer Thread를 계속 사용하고, 장애가 난 외부 시스템에 부하를 더한다. 같은 Partition의 뒤 메시지까지 처리하지 못할 수도 있다. 따라서 재시도 횟수를 제한하고, 시도 사이에 간격을 두며, 필요하면 간격을 점차 늘리는 Backoff를 적용한다.

```text
실패
  ↓ 제한된 횟수와 간격으로 Retry
그래도 실패
  ↓
별도 Queue 또는 Topic으로 분리
```

계속 실패하는 메시지를 정상 흐름에서 분리해 보관하는 경로를 일반적으로 DLQ(Dead Letter Queue)라고 부른다. Kafka에서는 별도의 Topic을 사용하는 DLT(Dead Letter Topic)라는 표현도 자주 쓴다.

DLQ나 DLT로 보냈다고 문제가 해결된 것은 아니다. 실패 원인을 확인하고, 데이터를 수정하거나 장애를 해소한 뒤, 누가 어떤 기준으로 다시 처리할지 운영 절차가 필요하다.

### HTTP 200 이후의 실패도 관찰해야 한다

비동기 구조에서는 HTTP 응답이 성공한 뒤 실제 작업이 실패할 수 있다. 요청 로그만으로 이메일 발송이 끝났는지 알 수 없으므로 다음 지표를 별도로 관찰한다.

- Consumer 처리 실패율
- Retry 횟수와 반복 실패 원인
- DLQ·DLT에 쌓인 메시지 수
- 처리되지 않고 뒤에 남은 Consumer Lag

예를 들어 Producer가 초당 1,000건을 만들지만 Consumer가 700건만 처리하면 초당 300건씩 밀린다. 이처럼 Consumer가 생산 속도를 따라가지 못해 남은 정도를 Lag로 관찰할 수 있다. Lag 증가는 Kafka 장애만을 뜻하지 않으며, 생산량 증가·느린 Consumer·외부 시스템 지연 등 여러 원인에서 생길 수 있다.

아래 화면에서는 그래프의 모양 자체보다 **Lag가 계속 증가하는지**, **실패와 Retry가 함께 늘어나는지**, **Consumer가 다시 따라잡는지**를 확인하면 된다.

![Kafka 처리 상태를 확인하는 Grafana 대시보드](/images/posts/async-processing-and-message-queue/legacy-06.png "Kafka 모니터링 예시")

지금까지 살펴보면 `@Async`와 메시지 큐의 차이는 처리 속도보다 실패했을 때 작업이 어디에 남고 누가 복구하는가에 있었다.

## 선택하기 전에 작업의 보장 수준을 정한다

느린 작업이면 `@Async`, 더 느리면 Kafka를 선택하는 문제가 아니다. 애플리케이션이 하나인지 여러 개인지도 결정적인 기준은 아니다. 하나의 애플리케이션에서도 메시지 큐를 사용할 수 있고, 여러 서비스가 있어도 짧은 로컬 작업에는 Executor를 사용할 수 있다.

`@Async`는 다음과 같은 작업에 잘 맞는다.

- 같은 애플리케이션 안에서 끝나는 짧은 부수 작업
- 유실되어도 다시 만들 수 있거나 업무 결과에 큰 영향을 주지 않는 작업
- 실패한 작업을 반드시 재처리해야 한다는 요구가 낮은 작업
- 별도 Broker를 운영할 만큼 복구·확장 요구가 크지 않은 작업

메시지 큐는 다음 요구가 생길수록 가치가 커진다.

- 프로세스가 종료되어도 작업이 남아야 함
- 실패한 작업의 Retry·DLQ·재처리가 필요함
- 여러 Consumer가 같은 Event를 각각 처리해야 함
- Producer와 Consumer의 처리량을 독립적으로 확장해야 함
- 처리 지연과 실패를 별도로 관찰해야 함

메시지 큐를 사용한다고 결합이 사라지는 것은 아니다. 직접 HTTP 호출처럼 실행 시점에 서로 기다리는 결합은 줄일 수 있지만, Consumer는 Producer가 발행하는 Event의 이름·필드·의미에 의존한다. Event도 변경 규칙이 필요한 계약이다.

실제 설계에서는 다음 질문을 순서대로 확인할 수 있다.

```text
HTTP 응답 전에 작업 결과가 꼭 필요한가?
  ├─ YES → 동기 처리 우선
  └─ NO  → 요청과 작업 분리 검토

작업을 잃어도 되거나 다시 만들 수 있는가?
  ├─ YES → @Async 같은 로컬 비동기 우선 검토
  └─ NO  → 프로세스 밖에 작업을 남길 Broker 검토

실패한 작업의 Retry·DLQ·재처리가 필요한가?
  └─ YES → 메시지 큐의 가치 증가

같은 Event를 여러 Consumer가 각각 처리해야 하는가?
  └─ YES → Consumer Group을 나눈 메시지 구조 검토

업무 DB 변경과 Event 발행 의도를 함께 확정해야 하는가?
  └─ YES → Transactional Outbox 검토

같은 업무의 Event 순서가 중요한가?
  └─ YES → Partition Key 설계

중복 전달이 가능한가?
  └─ YES 또는 가능성 존재 → Consumer 멱등성 설계
```

이는 정답을 자동으로 정해 주는 의사결정 트리가 아니라, 비동기 구조를 선택하기 전에 빠뜨리지 말아야 할 질문의 순서다.

## 정리

- 비동기는 작업을 더 빨리 끝내는 기술이 아니라 호출자가 완료를 기다리지 않게 실행 흐름을 분리하는 방법이다.
- `@Async`는 Spring Proxy가 작업을 Application 내부의 TaskExecutor에 넘겨 다른 Worker Thread에서 실행하게 한다.
- Thread Pool이 포화되면 작업은 Queue에서 기다리거나 거부되며, Queue의 작업은 프로세스 장애 때 유실될 수 있다.
- 메시지 큐는 작업을 Application 밖 Broker에 남겨 Producer와 Consumer의 생명주기를 분리한다.
- Broker가 복구 지점을 제공해도 DB 변경과 Event 발행 사이의 틈, 중복 전달, 순서, Retry와 운영 관측은 별도로 설계해야 한다.
- Transactional Outbox는 DB와 Kafka를 하나의 Transaction으로 묶는 방식이 아니라, 업무 변경과 발행 의도를 같은 DB Transaction에 기록해 다시 시작할 위치를 남기는 방식이다.

**`@Async`와 메시지 큐의 차이는 문법이나 처리 속도가 아니라 작업의 생명주기를 어디까지 애플리케이션 밖으로 분리할 것인가에 있다. 짧고 다시 만들 수 있는 부수 작업이라면 `@Async`가 단순한 선택일 수 있고, 프로세스 장애 뒤에도 작업을 남겨야 하거나 재시도·중복 처리·순서 보장·독립 확장이 필요하다면 메시지 큐를 검토할 이유가 커진다.**

## 참고 자료

### 공식 자료

- [Spring Framework - Task Execution and Scheduling](https://docs.spring.io/spring-framework/reference/integration/scheduling.html)
- [Java - ThreadPoolExecutor](https://docs.oracle.com/en/java/javase/18/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html)
- [Apache Kafka - Design](https://kafka.apache.org/41/design/design/)
- [Debezium - Outbox Event Router](https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html)
- [Spring Kafka - Retry Topic Features](https://docs.spring.io/spring-kafka/reference/retrytopic/features.html)

### 국내 기술 블로그

- [우아한형제들 - 우리 팀은 카프카를 어떻게 사용하고 있을까](https://techblog.woowahan.com/17386/)
