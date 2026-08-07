+++
title = '@Async는 언제 충분하고 언제 메시지 큐가 필요한가'
date = 2026-05-12T19:30:00+09:00
lastmod = 2026-08-07T16:49:40+09:00
draft = false
description = 'Spring @Async와 메시지 큐가 요청과 작업을 분리하는 방식, 실행 보장과 장애 복구의 차이를 정리합니다.'
categories = ['비동기 처리']
tags = ['Spring', 'Async', 'Kafka', '메시지 큐']
+++

비동기 처리는 작업을 무조건 빠르게 만드는 기술이 아니다. 호출자가 작업의 완료를 기다리지 않아도 되도록 **요청 처리와 실제 작업의 실행 시점을 분리하는 방법**이다.

Spring의 `@Async`와 Kafka 같은 메시지 큐는 모두 이 분리를 지원한다. 하지만 작업을 어디에 보관하고, 장애가 났을 때 어떻게 복구하는지가 다르다. 짧고 유실되어도 다시 수행할 수 있는 작업에는 `@Async`가 단순하고, 반드시 처리해야 하거나 여러 소비자가 독립적으로 받아야 하는 작업에는 메시지 큐가 더 잘 맞는다.

## 비동기 처리는 무엇을 줄이는가

![동기·비동기와 Blocking·Non-Blocking의 조합](/images/posts/async-processing-and-message-queue/legacy-01.png "동기·비동기와 Blocking·Non-Blocking은 서로 다른 기준이다")

비동기를 이해할 때 먼저 `동기·비동기`와 `Blocking·Non-Blocking`을 구분해야 한다. 두 쌍은 비슷하게 들리지만 바라보는 대상이 다르다.

- 동기·비동기는 **작업 완료를 현재 호출 흐름과 어떻게 맞출지**에 관한 구분이다.
- Blocking·Non-Blocking은 **결과를 기다리는 동안 현재 스레드가 멈추는지**에 관한 구분이다.

일반적인 Spring MVC 요청은 이해하기 쉬운 `동기 + Blocking` 형태다. 요청을 맡은 스레드가 Service를 호출하고, 데이터베이스나 외부 API의 응답이 돌아올 때까지 기다린 뒤 HTTP 응답을 만든다. 한 요청 스레드가 기다린다고 서버 전체가 멈추는 것은 아니다. 다른 요청은 다른 스레드가 처리한다. 다만 기다리는 요청이 많아지면 사용할 수 있는 요청 스레드가 점점 줄어든다.

동기 호출에서는 호출자가 결과를 받을 때까지 현재 흐름을 멈춘다.

```text
HTTP 요청 → 주문 저장 → 이메일 전송 → HTTP 응답
```

이메일 서버가 느려지면 요청 응답도 함께 느려진다. 이메일 전송을 비동기로 넘기면 요청 스레드는 작업을 제출한 뒤 먼저 응답할 수 있다.

```text
HTTP 요청 → 주문 저장 → 작업 제출 → HTTP 응답
                         └→ 별도 스레드에서 이메일 전송
```

여기서 줄어든 것은 이메일 전송 시간 자체가 아니다. 사용자가 그 시간을 기다리지 않게 된 것이다. 별도 스레드도 결국 같은 작업을 수행하므로 실행 자원과 외부 시스템의 처리량은 여전히 필요하다.

예를 들어 이메일 전송이 3초 걸린다면 비동기로 바꿔도 이메일은 여전히 3초 동안 전송된다. 달라지는 것은 HTTP 요청 스레드가 그 3초를 함께 기다리지 않는다는 점이다. 따라서 비동기는 **작업 시간 단축**보다 **기다림의 위치 변경**으로 이해하는 편이 정확하다.

그렇다면 Spring은 요청 흐름과 실제 작업을 어떤 방식으로 분리할까. 가장 가까운 출발점이 같은 애플리케이션 안에서 실행 스레드를 나누는 `@Async`다.

## Spring @Async는 무엇을 하는가

`@Async`가 붙은 메서드를 다른 Spring Bean이 호출하면 Spring AOP Proxy가 호출을 가로챈다. Proxy는 메서드를 바로 실행하는 대신 `TaskExecutor`에 작업을 제출하고 호출자에게 제어권을 돌려준다.

호출 순서를 한 단계씩 펼치면 다음과 같다.

```text
1. 요청 스레드가 Spring Bean의 메서드 호출
2. 실제 Bean 앞에 있는 AOP Proxy가 @Async 확인
3. 실행할 메서드를 Runnable 또는 Callable 작업으로 구성
4. TaskExecutor의 대기열에 작업 제출
5. 요청 스레드는 호출 지점으로 복귀
6. Executor의 작업 스레드가 나중에 실제 메서드 실행
```

즉 `@Async` 메서드 안에서 갑자기 스레드가 바뀌는 것이 아니다. **Bean 호출을 가로챈 Proxy가 실행 주체를 바꾸는 것**이다.

```java
@Service
@RequiredArgsConstructor
public class OrderNotificationService {

    private final MailClient mailClient;

    @Async("notificationExecutor")
    public void sendOrderCompleted(long orderId) {
        mailClient.sendOrderCompleted(orderId);
    }
}
```

이 구조에는 세 가지 조건이 있다.

- 같은 클래스 안에서 자기 메서드를 호출하면 Proxy를 통과하지 않아 동기적으로 실행된다.
- `void` 반환 작업의 예외는 원래 호출자에게 돌아가지 않는다. 로그와 모니터링, 필요한 재시도 정책을 별도로 마련해야 한다.
- 작업은 애플리케이션 메모리와 스레드 풀의 대기열에 머문다. 프로세스가 종료되면 실행 중이거나 대기 중인 작업을 잃을 수 있다.

스레드 풀은 처리량을 무한히 늘리는 장치도 아니다. 작업이 쌓이는 속도가 처리 속도보다 빠르면 대기열이 계속 늘거나 거부 정책이 동작한다.

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

`corePoolSize`, `queueCapacity`, `maxPoolSize`는 따로 떼어 결정할 수 없다. 작업 도착률, 한 작업의 평균 실행 시간, 외부 시스템이 감당할 수 있는 동시 요청 수를 함께 보고 정해야 한다.

기본적인 작업 수용 순서는 다음과 같다.

```text
corePoolSize까지 작업 스레드 사용
→ 모두 사용 중이면 queueCapacity까지 대기
→ 대기열도 가득 차면 maxPoolSize까지 스레드 추가
→ 최대 스레드와 대기열이 모두 가득 차면 거부 정책 실행
```

이 순서를 모르면 `maxPoolSize`를 크게 잡고도 왜 스레드가 늘지 않는지 이해하기 어렵다. 대기열이 매우 크면 작업이 먼저 큐에 쌓이므로 스레드는 오랫동안 `corePoolSize`에 머물 수 있다. 반대로 대기열이 너무 작으면 순간적인 요청 증가에도 스레드가 빠르게 늘고 거부가 발생할 수 있다.

작업 종류도 중요하다. CPU 계산이 중심인 작업은 스레드를 과도하게 늘리면 Context Switching만 증가할 수 있다. 이메일·외부 API처럼 기다림이 긴 I/O 작업은 더 많은 스레드를 활용할 여지가 있지만, 상대 시스템이 감당할 수 있는 동시 요청 수가 새로운 상한이 된다.

여기까지는 작업이 여전히 애플리케이션 프로세스 안에 있다는 전제가 있다. 프로세스가 종료되어도 작업을 남겨야 하거나 생산자와 소비자를 독립적으로 운영해야 한다면 실행 스레드가 아니라 작업의 저장 위치까지 분리해야 한다.

## 메시지 큐가 바꾸는 경계

![Kafka의 Producer, Broker, Topic, Partition, Consumer 구조](/images/posts/async-processing-and-message-queue/legacy-02.png "Kafka 기본 구조")

Spring에서 Kafka를 연결할 때도 구조는 같다. 의존성을 추가하고 Broker 주소를 설정한 뒤, Producer는 `KafkaTemplate`, Consumer는 `@KafkaListener`로 이 경계에 연결된다.

![Spring Kafka 의존성 구성](/images/posts/async-processing-and-message-queue/legacy-03.png "Spring Kafka 의존성")

![Kafka Broker 연결 설정](/images/posts/async-processing-and-message-queue/legacy-04.png "Kafka Broker 연결 설정")

![KafkaTemplate을 사용한 메시지 발행 예시](/images/posts/async-processing-and-message-queue/legacy-05.png "Spring에서 Kafka 메시지 발행")

메시지 큐를 사용하면 작업이 애플리케이션 프로세스 밖의 Broker에 기록된다.

```text
Producer → Broker의 Topic → Consumer
```

Producer는 메시지를 발행하고, Consumer는 자기 속도에 맞춰 메시지를 가져가 처리한다. Consumer가 잠시 중단되어도 Broker에 메시지가 남아 있다면 재시작 후 이어서 처리할 수 있다. 트래픽이 순간적으로 몰릴 때도 Broker가 작업을 버퍼링해 생산자와 소비자의 속도 차이를 흡수한다.

Kafka를 예로 들면 용어는 다음처럼 대응된다.

```text
Producer       메시지를 발행하는 애플리케이션
Topic          같은 목적의 메시지를 모아두는 논리적인 이름
Partition      Topic을 나눈 실제 저장·처리 단위
Consumer       메시지를 읽어 업무를 수행하는 애플리케이션
Consumer Group 같은 작업을 나눠 처리하는 Consumer의 묶음
Offset         Partition에서 Consumer가 어디까지 읽었는지 나타내는 위치
```

예를 들어 주문 서비스가 `order-completed` Topic에 이벤트를 발행하면 이메일 Consumer와 통계 Consumer는 서로 다른 Consumer Group으로 같은 이벤트를 각각 처리할 수 있다. 반대로 이메일 Consumer를 여러 대 띄워 같은 Group에 넣으면 Partition을 나눠 맡아 이메일 작업을 병렬 처리한다.

다만 큐를 두었다고 전체 부하가 사라지는 것은 아니다. 처리를 뒤로 미뤄 둘 수 있을 뿐이며, 장기적으로 소비 속도가 생산 속도보다 느리면 Lag가 계속 증가한다.

### 데이터 변경과 이벤트 발행 사이에도 틈이 있다

메시지를 Broker에 보냈다고 끝나는 것이 아니다. 업무 데이터 변경과 이벤트 발행이 서로 다른 시스템에서 일어나면 둘 사이에 실패할 틈이 생긴다.

```text
1. 데이터베이스에서 주문 취소 Commit
2. Kafka에 주문 취소 Event 발행
```

1번 뒤에 프로세스가 종료되거나 Kafka 발행이 실패하면 데이터베이스에는 취소된 주문이 남지만 Consumer는 취소 사실을 모른다. 반대로 Event를 먼저 보낸 뒤 데이터베이스가 Rollback되면 실제로는 취소되지 않은 주문의 Event가 전달될 수 있다. 데이터베이스 트랜잭션 하나로 Kafka까지 원자적으로 묶을 수 없기 때문에 생기는 문제다.

Transactional Outbox는 발행할 Event를 업무 데이터와 같은 데이터베이스에 먼저 기록해 이 틈을 줄인다.

```text
하나의 DB Transaction
├─ 주문 상태 변경
└─ Outbox 행 추가
        ↓ Commit 뒤
Outbox 전달기가 Kafka로 발행
```

두 데이터는 함께 Commit되거나 Rollback된다. Commit된 Outbox 행은 Polling으로 읽을 수도 있고, Binlog를 구독하는 CDC Connector로 Kafka에 전달할 수도 있다. 발행이 잠시 실패하더라도 전달기는 데이터베이스에 남은 기록을 다시 읽을 수 있다. 다만 **전달 재시도 때문에 같은 Event가 중복 발행될 수 있으므로 Consumer의 멱등성은 여전히 필요하다.**

CDC Connector 한 개의 처리량이 부족해 Outbox를 나눠야 할 때는 순서 보장 범위까지 함께 봐야 한다. 같은 주문의 Event가 서로 다른 Outbox와 Connector로 흩어지면 Kafka에 도착하는 순서도 달라질 수 있다. 따라서 같은 업무 식별자는 같은 Outbox로 보내고, 각 Outbox에는 독립된 Connector를 연결하는 식으로 처리량과 순서 경계를 함께 맞출 수 있다.

여기서 가져갈 핵심은 Kafka나 특정 Connector의 이름이 아니다. **업무 상태와 발행할 Event를 어느 경계에서 함께 확정하고, 실패한 발행을 어디서 다시 시작할지 정하는 것**이다.

### 순서가 필요하면 Partition 기준부터 정한다

Kafka에서는 Topic을 Partition으로 나누고, 같은 Consumer Group의 Consumer들이 Partition을 나눠 맡는다. 순서는 Topic 전체가 아니라 한 Partition 안에서만 유지된다.

예를 들어 같은 주문에서 `결제 완료`와 `주문 취소`의 순서가 중요하다면 주문 ID를 Key로 사용해 두 Event를 같은 Partition으로 보내야 한다. 서로 다른 주문끼리의 전역 순서까지 맞출 필요는 없으므로 Partition을 나눠 병렬 처리할 수 있다. 결국 순서 보장은 “Kafka가 알아서 해준다”가 아니라 **어떤 Event끼리 순서를 공유해야 하는지 Key로 표현하는 설계**다.

같은 Key의 순서를 지켰더라도 메시지가 한 번만 도착한다고 가정할 수는 없다. 전달 보장과 장애 복구를 얻는 대신 소비자는 재전달 가능성까지 감당해야 한다.

## 적어도 한 번 처리되면 중복을 고려해야 한다

Kafka의 일반적인 처리 흐름에서는 메시지를 처리한 뒤 Offset을 Commit한다. 처리는 끝났지만 Commit 전에 Consumer가 종료되면 같은 메시지를 다시 받을 수 있다. 이 때문에 일반적인 소비 구조는 중복 가능성을 포함한 at-least-once 관점으로 설계한다.

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

중복 검사를 추가했다고 항상 안전한 것은 아니다. 검사와 실제 처리 사이에 장애가 날 수 있으므로, 업무 데이터 변경과 처리 기록을 같은 트랜잭션으로 묶거나 외부 API가 멱등 키를 지원한다면 이를 활용하는 편이 낫다. 정확히 한 번이라는 표현도 Broker 내부 처리와 외부 데이터베이스 변경을 구분해서 봐야 한다.

처리에 실패했다고 같은 메시지를 즉시 무한 반복하는 것도 안전하지 않다. 데이터베이스나 외부 API가 장애 상태라면 실패 요청만 계속 쌓이면서 정상 메시지까지 처리하지 못할 수 있다. 보통은 재시도 횟수와 간격을 제한하고, 계속 실패한 메시지는 DLQ(Dead Letter Queue) 같은 별도 경로로 격리한다.

```text
메시지 처리
→ 실패
→ 잠시 기다린 뒤 제한된 횟수만큼 재시도
→ 계속 실패
→ DLQ로 이동
→ 원인 확인과 수정 뒤 재처리
```

운영할 때는 Consumer Lag, 재시도 횟수, DLQ 적재량을 함께 본다. Lag가 계속 증가한다면 Broker가 문제라기보다 생산 속도를 소비 속도가 따라가지 못하는 상태일 수 있다. 비동기 구조에서는 HTTP 요청이 이미 성공했기 때문에 이런 지표가 실제 작업 실패를 발견하는 창구가 된다.

결국 `@Async`와 메시지 큐의 차이는 문법보다 작업을 어디에 남기고 누가 실패를 복구하느냐에 있다. 이제 앞에서 확인한 실행 위치, 유실 가능성, 중복과 복구 요구를 선택 기준으로 묶어볼 수 있다.

## 무엇을 선택할까

메시지 큐를 선택했다면 전달 성공 여부만 볼 것이 아니라 Consumer Lag, 실패율과 재시도처럼 실제 처리 상태를 관찰해야 한다.

![Kafka 처리 상태를 확인하는 Grafana 대시보드](/images/posts/async-processing-and-message-queue/legacy-06.png "Kafka 모니터링 예시")

`@Async`가 잘 맞는 경우는 다음과 같다.

- 한 애플리케이션 안에서 짧게 끝나는 부수 작업
- 실패해도 사용자가 다시 시도하거나 재생성할 수 있는 작업
- 별도 Broker를 운영할 만큼 전달 보장과 확장 요구가 크지 않은 경우

메시지 큐가 잘 맞는 경우는 다음과 같다.

- 프로세스 재시작 뒤에도 작업이 남아 있어야 하는 경우
- 재시도와 실패 격리, 소비 지연 관측이 필요한 경우
- 여러 Consumer가 같은 이벤트를 서로 다른 목적으로 처리하는 경우
- 생산자와 소비자의 배포 및 처리량을 독립적으로 확장해야 하는 경우

비동기 처리의 선택 기준은 “느린가?” 하나가 아니다. 작업을 잃어도 되는지, 중복 처리가 가능한지, 실패를 누가 복구할지부터 정해야 한다.

## 정리

- `@Async`는 같은 프로세스 안에서 실행 스레드를 분리한다.
- 메시지 큐는 작업의 저장 위치와 생산자·소비자의 생명주기까지 분리한다.
- 비동기는 총 작업량을 없애지 않고 호출자의 대기만 줄인다.
- at-least-once 처리는 중복을 전제로 멱등성을 설계해야 한다.
- 실행 보장과 장애 복구가 필요할수록 메시지 큐를 검토할 이유가 커진다.

## 참고 자료

### 공식 자료

- [Spring Framework - Task Execution and Scheduling](https://docs.spring.io/spring-framework/reference/integration/scheduling.html)
- [Apache Kafka - Design: Message Delivery Semantics](https://kafka.apache.org/41/design/design/)

### 국내 기술 블로그

- [우아한형제들 - 우리 팀은 카프카를 어떻게 사용하고 있을까](https://techblog.woowahan.com/17386/)
