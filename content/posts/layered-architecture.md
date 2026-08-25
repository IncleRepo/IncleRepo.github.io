+++
title = 'Controller, Service, Repository로 이해하는 레이어드 아키텍처'
slug = '14'
date = 2026-08-21T20:49:00+09:00
lastmod = 2026-08-25T23:20:00+09:00
draft = false
references_required = true
description = 'Spring에서 익숙한 Controller, Service, Repository 구조를 따라가며 레이어드 아키텍처의 책임과 의존 방향, 장점과 한계를 살펴봅니다.'
categories = ['애플리케이션 아키텍처']
tags = ['레이어드 아키텍처', 'Controller', 'Service', 'Repository']
+++

Spring 애플리케이션에서는 Controller, Service, Repository로 역할을 나눈 구조를 자주 만난다.

![Controller, DTO, Repository, Service로 나눈 패키지 구조](/images/posts/layered-architecture/project-directory.png "Spring에서 자주 만나는 패키지 구조")

왜 굳이 세 부분으로 나눌까? 각기 다른 책임을 분리하고 코드의 참조 방향을 제한하기 위해서다.

주문 취소 요청 하나가 처리되는 과정을 단순화하면 다음과 같다.

```text
HTTP 요청 → Controller → Service → Repository → DB
```

실행할 때는 Controller가 Service를 호출하고 Service가 Repository를 호출한다. 코드에서도 Controller는 Service를, Service는 Repository를 참조한다.

이 익숙한 흐름은 어떤 기준으로 나뉘며, 각 부분은 무엇을 맡아야 할까? 이 글에서는 전형적인 **레이어드 아키텍처**(Layered Architecture)의 역할과 의존 방향을 살펴보고, 이 구조가 잘 맞는 상황과 한계까지 정리한다.

## 레이어드 아키텍처란 무엇인가

레이어드 아키텍처는 성격이 비슷한 책임을 하나의 Layer로 묶고, Layer 사이에 허용할 의존 관계를 정하는 구조다.

![User Interface부터 Data까지 여러 계층으로 나눈 레이어드 아키텍처](/images/posts/layered-architecture/layered-architecture.png "계층을 더 세분화한 레이어드 아키텍처의 한 가지 예")

위 그림은 Layer를 비교적 세분화한 사례다. 프로젝트에 따라 Application과 Domain을 나누기도 하고 Infrastructure를 별도로 표현하기도 한다. 레이어드 아키텍처가 반드시 세 계층으로 구성되는 것은 아니다.

이 글에서는 단순한 Spring 구조에 맞춰 다음 세 영역을 중심으로 본다.

- **Presentation Layer:** 외부 인터페이스와 애플리케이션의 경계
- **Application/Business Layer:** 하나의 애플리케이션 작업을 조정
- **Persistence Layer:** 데이터 접근을 담당

계층의 수보다 중요한 것은 **책임의 경계와 의존 규칙**이다. 주문 취소 요청을 따라가며 각 영역의 역할을 살펴보자.

## 주문 취소 요청에서 각 계층은 무엇을 맡는가

1번 주문을 취소하는 요청은 먼저 Controller에 도착한다.

```java
@RestController
@RequiredArgsConstructor
public class OrderController {

    private final OrderService orderService;

    @PostMapping("/orders/{id}/cancel")
    public void cancelOrder(@PathVariable long id) {
        orderService.cancel(id);
    }
}
```

Controller는 주문 ID를 받아 애플리케이션의 주문 취소 작업을 호출한다. REST API에서 **Presentation Layer**를 대표하는 구성 요소다.

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;

    @Transactional
    public void cancel(long id) {
        Order order = orderRepository.findById(id)
            .orElseThrow();

        if (order.getStatus() == OrderStatus.SHIPPED
                || order.getStatus() == OrderStatus.DELIVERED) {
            throw new IllegalStateException("배송을 시작한 주문은 취소할 수 없습니다.");
        }

        order.cancel();
    }
}
```

`OrderService`는 Repository에서 주문을 조회하고, 취소할 수 있는 상태인지 확인한 뒤 주문 상태를 변경한다. 주문 취소라는 하나의 애플리케이션 작업을 완성하는 흐름을 조정하며 **Application/Business Layer**에 해당한다.

마지막으로 Service가 호출하는 Repository를 보자.

```java
public interface OrderRepository extends JpaRepository<Order, Long> {
}
```

Repository는 데이터 접근을 담당하는 **Persistence Layer**에 해당한다. 위 예제에서는 주문 조회와 변경된 상태의 저장을 Spring Data JPA에 맡긴다.

각 계층의 역할을 나누면 이들이 서로 어느 방향으로 참조하는지도 중요해진다.

## 계층 사이에는 방향이 있다

앞의 주문 취소 흐름에서 Controller는 Service를 참조하고, Service는 Repository를 참조한다. Repository가 Controller를 거꾸로 참조하지는 않는다.

의존 방향을 제한하면 각 Layer가 알아야 할 세부사항의 범위도 정해진다. `OrderRepository`는 주문 취소가 어떤 URL에서 시작됐는지 알 필요가 없고, `OrderService`도 HTTP 상태 코드를 직접 다루지 않고 주문 취소 작업에 집중한다.

이처럼 위쪽 계층이 아래쪽 계층을 참조하는 것이 레이어드 아키텍처의 일반적인 모습이다. 프로젝트에 따라 바로 아래 계층만 호출하도록 제한하거나, 필요한 경우 한 계층을 건너뛸 수 있도록 허용한다.

![Controller Layer에서 하위 Layer로 향하는 의존 방향과 한 계층을 건너뛰는 참조](/images/posts/layered-architecture/layer-dependency-direction.png "한 DDD 리팩터링 사례의 Layer 의존 방향. 출처: Özkan, Babur와 van den Brand (2023), Figure 3, CC BY 4.0")

의존 방향을 정해도 변경의 영향까지 사라지지는 않는다. Repository의 반환형이 바뀌면 Service도 수정될 수 있다. 그래도 **누가 누구를 참조하는지 예측하기 쉬워진다.**

## 이 구조가 널리 사용되는 이유

Controller, Service, Repository라는 이름만으로도 코드의 역할과 요청 흐름을 어느 정도 예상할 수 있다. 이 예측 가능성은 다음과 같은 장점으로 이어진다.

### 서로 다른 관심사를 나누기 쉽다

Controller 하나에서 HTTP 요청 처리, 업무 조건 검사, SQL 실행과 JSON 응답 생성까지 맡으면 서로 다른 이유로 바뀌는 코드가 뒤섞인다.

이를 Controller, Service, Repository로 나누면 각 코드를 이해할 때 살펴볼 범위가 줄어든다. 서로 다른 관심사를 각 영역으로 나누는 **관심사의 분리**(Separation of Concerns)다.

### 학습과 협업 비용이 낮다

Spring 예제와 많은 웹 애플리케이션이 비슷한 구조를 사용한다. 새로 합류한 팀원도 익숙한 책임과 호출 흐름을 따라 코드를 찾기 쉽다. 업무 흐름이 단순한 서비스라면 이 정도의 예측 가능성만으로도 충분히 유용하다.

## 레이어드 구조를 패키징하는 두 가지 방법

같은 책임과 호출 방향을 유지하면서도 실제 코드는 프로젝트 성격에 맞게 다르게 배치할 수 있다. 먼저 기술 역할별로 코드를 모아 보자.

```text
controller
├── UserController
├── OrderController
└── PaymentController

service
├── UserService
├── OrderService
└── PaymentService

repository
├── UserRepository
├── OrderRepository
└── PaymentRepository
```

이런 **Package by Layer** 구조는 Controller, Service와 Repository를 한곳에서 찾기 쉽다. 구조가 단순하고 기능 수가 적을 때 이해하기 편하지만, 주문 기능 하나를 수정할 때 여러 패키지를 오가게 될 수 있다.

반대로 업무 기능을 먼저 나누고 그 안에 각 Layer를 둘 수도 있다.

```text
user
├── controller
├── service
├── repository
├── entity
└── dto

order
├── controller
├── service
└── repository
```

이런 구성은 기능을 기준으로 코드를 묶는 **Package by Feature**다. 예시처럼 `user`, `order` 같은 업무 영역을 최상위 패키지로 사용하면 **Package by Domain**이라고 부르기도 한다. 어느 이름을 사용하든 기능별로 패키지를 나눈 뒤 내부 요청 흐름은 Controller, Service, Repository 순서로 구성할 수 있다.

두 방식 가운데 하나가 항상 더 좋은 것은 아니다. 코드가 단순할 때는 역할별 구성이 편하고, 기능이 늘어나 서로 관련된 코드를 함께 찾는 일이 많아지면 기능별 구성이 유리할 수 있다.

## 업무 규칙이 Service에 모이면 비대해질 수 있다

앞에서 살펴본 `OrderService`로 돌아가 보자. 주문을 조회하고 취소 가능 여부를 검사한 뒤 상태를 바꾸는 정도라면 역할이 분명하고 코드도 짧다.

이제 결제된 주문을 완료 상태로 바꾸는 메서드가 같은 Service에 추가됐다고 해보자.

```java
@Transactional
public void completeOrder(long orderId) {
    Order order = orderRepository.findById(orderId)
        .orElseThrow();

    if (order.getStatus() != OrderStatus.PAID) {
        throw new IllegalStateException();
    }

    order.setStatus(OrderStatus.COMPLETED);
}
```

이 메서드도 주문 조회, 상태 검증과 변경만 조정하므로 그 자체로는 복잡하지 않다. 그러나 주문 취소와 완료에 이어 가격 계산, 할인 판단과 재고 확인까지 같은 클래스에 쌓이면 `OrderService`가 여러 업무 규칙을 모두 처리하며 점점 비대해질 수 있다.

레이어드 아키텍처는 **코드를 어느 책임의 Layer에 둘 것인지** 알려준다. 그러나 주문이라는 업무 개념을 어떤 객체와 규칙으로 표현할지는 정해 주지 않는다. 도메인 모델링에 별도의 기준이 없다면 업무 규칙이 Service에 모이기 쉽다.

이 문제는 계층 구조를 유지한 채 업무 규칙을 도메인 객체로 나누어 풀 수 있다. 그런데 Service가 커지는 것만 문제가 되는 것은 아니다. 계층 형식에 맞추느라 Service가 거의 아무 일도 하지 않는 반대 상황도 생긴다.

## 형식적인 계층 통과가 반복되면 Architecture Sinkhole이 된다

레이어드 아키텍처에서는 상위 계층이 바로 아래 계층을 거치도록 제한할 수 있다. 이를 **닫힌 계층**이라고 한다. Controller가 Repository를 직접 호출하지 않고 반드시 Service를 거치는 구조가 대표적이다.

반면 회원 조회처럼 단순한 요청에서는 Service가 Repository 호출만 전달할 수도 있다.

```java
public UserResponse getUser(long id) {
    return userRepository.findResponseById(id)
        .orElseThrow();
}
```

이 메서드는 업무 규칙을 적용하거나 데이터를 변환하지 않고 요청을 다음 계층으로 넘긴다. 이런 요청이 대부분을 차지해 여러 계층을 의미 없이 통과하는 구조를 **Architecture Sinkhole Anti-Pattern**이라고 부른다.

이런 메서드가 한두 개 생기는 것은 자연스럽다. Service가 트랜잭션이나 인가, 여러 작업의 조정을 맡는다면 얇아 보여도 역할이 있다. 살펴봐야 할 것은 **대부분의 요청이 별다른 처리 없이 Layer만 통과하는가**다.

> **심화 참고**
>
> 《Fundamentals of Software Architecture》는 단순 전달 요청이 약 20%라면 자연스러울 수 있지만, 약 80%라면 현재 구조를 점검할 신호라고 설명한다. 절대적인 기준이 아니라 애플리케이션 전체 흐름을 돌아보기 위한 경험칙이다.

Sinkhole을 줄이기 위해 일부 Layer를 열고 Controller가 Repository를 직접 호출하도록 허용할 수도 있다. 두 선택의 차이를 비교하면 다음과 같다.

| 선택 | 얻는 점 | 비용 |
| --- | --- | --- |
| 닫힌 Layer 유지 | 요청 흐름과 Application 경계가 일정함 | 단순 전달 코드가 생길 수 있음 |
| 일부 Layer 개방 | 의미 없는 전달을 줄일 수 있음 | Controller가 Persistence에 직접 의존함 |

Layer를 건너뛰도록 허용한다면 프로젝트 안에서 일관된 기준을 세워야 한다. Service를 무조건 없애기보다 **구조의 일관성과 불필요한 통과 코드 사이에서 적절한 지점을 찾는 것**이 중요하다.

각 Layer가 실제 책임을 맡고 있어도 외부 기술과의 결합은 그대로 남을 수 있다.

## 계층을 나눴다고 기술과 분리된 것은 아니다

Controller와 Repository를 나누었어도 Service가 외부 기술을 직접 알 수 있다.

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;
}
```

위 Service는 주문 작업을 조정하면서 Kafka의 구체적인 API를 직접 사용한다. `OrderRepository`가 `JpaRepository`를 그대로 노출하거나 외부 API Client를 직접 사용한다면 JPA와 외부 연동 기술도 Service 코드에 드러난다.

**Layer를 나누는 것과 핵심 로직을 외부 기술에서 분리하는 것은 별개의 문제다.** 레이어드 아키텍처만으로 이런 기술 의존성이 사라지지는 않는다. 외부 기술과 핵심 로직 사이에 더 분명한 경계가 필요하다면 헥사고날 아키텍처와 같은 다른 구조로 시야를 넓힐 수 있다.

## 레이어드 아키텍처는 언제 잘 맞을까?

레이어드 아키텍처는 책임을 나누고 의존 방향을 정하는 데 집중한다.
업무 규칙을 어디에 둘지, 언제 계층을 건너뛸지, 외부 기술과 어떻게 분리할지는 별도로 정해야 한다.
다만 이러한 추가 설계가 크게 필요하지 않은 다음 상황에서는 Controller, Service, Repository만으로도 충분할 수 있다.

- 업무 규칙과 기능 흐름이 단순함
- 외부 기술이 바뀔 가능성과 영향이 크지 않음
- 팀이 전체 구조와 호출 흐름을 쉽게 파악할 수 있음
- 더 복잡한 구조가 주는 이점보다 학습과 유지 비용이 큼

반면 Service가 계속 비대해지고 형식적인 Layer 통과가 늘어나거나, 기술 변경이 업무 코드까지 반복해서 영향을 준다면 다른 설계 방식을 검토할 이유가 생긴다.

프로젝트 크기만으로 아키텍처를 결정할 수는 없다. **업무 복잡도, 변경 빈도, 의존성의 복잡도, 팀이 감당해야 하는 구조적 비용**을 함께 봐야 한다. 더 복잡한 구조를 도입해 얻는 이점이 그 비용보다 클 때 비로소 바꿀 이유가 생긴다.

## 정리

레이어드 아키텍처는 서로 다른 책임을 계층으로 나누고 코드가 어느 방향으로 참조되는지 예측하기 쉽게 만든다. 구조가 단순하고 익숙하며, 코드를 계층별로 묶든 기능별로 묶든 적용할 수 있다.

이와 별도로 다음 문제에는 다른 설계 기준이 필요하다.

- 복잡한 업무 규칙을 어떤 객체와 모델로 표현할 것인가?
- 모든 요청이 반드시 모든 Layer를 지나야 하는가?
- 핵심 업무 코드와 JPA, Kafka 같은 외부 기술을 어떻게 분리할 것인가?

단순함이 주는 이점이 현재 시스템에 충분하다면 레이어드 아키텍처는 좋은 선택이다. 업무 규칙과 기술 의존성이 복잡해질수록 **Layer를 나누는 것만으로 충분한가**라는 다음 질문이 생긴다.

Service 안에 쌓인 업무 규칙을 객체가 직접 맡게 하려면 어떻게 해야 할까? Service가 JPA나 Kafka를 모르게 만들 수는 없을까? 경계와 의존 방향을 애플리케이션 전체에서 더 엄격하게 관리하면 어떤 구조가 될까? 이런 질문은 DDD, 헥사고날 아키텍처, 클린 아키텍처를 공부하는 출발점이 된다.

## 참고 자료

### 공식 자료

- [Microsoft Azure Architecture Center - N-tier Architecture Style](https://learn.microsoft.com/ko-kr/azure/architecture/guide/architecture-styles/n-tier)
- [Spring Data JPA - Core Concepts](https://docs.spring.io/spring-data/jpa/reference/repositories/core-concepts.html)
- [O'Reilly - Fundamentals of Software Architecture](https://www.oreilly.com/library/view/fundamentals-of-software/9781492043447/)

### 추가 자료

- [Martin Fowler - Presentation Domain Data Layering](https://martinfowler.com/bliki/PresentationDomainDataLayering.html)
- [Herberto Graca - Layered Architecture](https://herbertograca.com/2017/08/03/layered-architecture/)
- [Klarc - Layered Architecture with Sinkhole](https://klarciel.net/wiki/architecture/architecture-layered/)
- [매일메일 - 레이어드 아키텍처란 무엇인가요?](https://www.maeil-mail.kr/question/310)
- [andrew.log - 그 서비스, 진짜 일하고 있나요?](https://velog.io/@sh1623/%EA%B7%B8-%EC%84%9C%EB%B9%84%EC%8A%A4-%EC%A7%84%EC%A7%9C-%EC%9D%BC%ED%95%98%EA%B3%A0-%EC%9E%88%EB%82%98%EC%9A%94-%EC%8B%B1%ED%81%AC%ED%99%80-%EC%95%88%ED%8B%B0%ED%8C%A8%ED%84%B4)
- [Özkan, Babur와 van den Brand - Refactoring with domain-driven design in an industrial context: An action research report](https://doi.org/10.1007/s10664-023-10310-1) (Figure 3, [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/))
