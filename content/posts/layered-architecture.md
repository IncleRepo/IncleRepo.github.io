+++
title = 'Controller, Service, Repository로 이해하는 레이어드 아키텍처'
slug = '14'
date = 2026-08-21T20:49:00+09:00
lastmod = 2026-08-25T21:00:00+09:00
draft = false
references_required = true
description = 'Spring에서 익숙한 Controller, Service, Repository 구조를 따라가며 레이어드 아키텍처의 책임과 의존 방향, 장점과 한계를 살펴봅니다.'
categories = ['애플리케이션 아키텍처']
tags = ['레이어드 아키텍처', 'Controller', 'Service', 'Repository']
+++

Spring 애플리케이션에서는 Controller, Service와 Repository로 역할을 나눈 구조를 자주 만난다.

![Controller, DTO, Repository, Service로 나눈 패키지 구조](/images/posts/layered-architecture/project-directory.png "Spring에서 자주 만나는 패키지 구조")

왜 굳이 세 역할로 나눌까? 서로 다른 기술적 책임을 분리하고 코드의 참조 방향을 제한하기 위해서다.

전형적인 레이어드 구조에서는 실행 흐름이 위에서 아래로 이어진다.

```text
실행 흐름

HTTP 요청
↓
Controller
↓
Service
↓
Repository
↓
DB
```

소스코드의 의존 방향도 같은 방향을 따른다.

```text
코드 의존

Controller → Service → Repository
```

Layer는 요청이 지나가는 순서이면서, 더 본질적으로는 책임과 의존성을 나누는 경계다. 이 글에서는 실행 흐름과 코드 의존 방향이 같은 전형적인 **레이어드 아키텍처**(Layered Architecture)를 다룬다.

## 레이어드 아키텍처란 무엇인가

레이어드 아키텍처는 관련된 책임을 Layer로 묶고, Layer 사이에 허용할 의존 관계를 정하는 구조다.

![User Interface부터 Data까지 여러 계층으로 나눈 레이어드 아키텍처](/images/posts/layered-architecture/layered-architecture.png "계층을 더 세분화한 레이어드 아키텍처의 한 가지 예")

위 그림은 Layer를 더 세분화한 예다. 모든 이름을 외울 필요는 없다. 레이어드 아키텍처는 반드시 세 계층으로 구성되는 것이 아니며, 프로젝트에 따라 Application과 Domain 등을 나누거나 Infrastructure를 별도로 표현할 수 있다.

이 글에서는 단순한 Spring 구조에 맞춰 다음 세 영역을 중심으로 본다.

- **Presentation Layer:** 외부 인터페이스와 애플리케이션의 경계
- **Application/Business Layer:** 하나의 애플리케이션 작업을 조정하는 영역
- **Persistence Layer:** 데이터 접근을 담당하는 영역

중요한 것은 Layer의 개수가 아니라 **책임의 경계와 의존 규칙**이다. 이제 회원 조회 요청으로 각 영역의 역할을 살펴보자.

## 회원 조회 요청에서 각 계층은 무엇을 맡는가

`GET /users/1` 요청으로 1번 회원을 조회한다고 해보자. 요청은 먼저 Controller에 도착한다.

```java
@RestController
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @GetMapping("/users/{id}")
    public UserResponse getUser(@PathVariable long id) {
        return userService.getUser(id);
    }
}
```

Controller는 외부 요청을 애플리케이션 호출로 연결하고 결과를 응답으로 반환한다. REST API에서 Controller는 **Presentation Layer**의 대표적인 구성 요소다.

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;

    public UserResponse getUser(long id) {
        User user = userRepository.findById(id)
            .orElseThrow();

        return UserResponse.from(user);
    }
}
```

`UserService`는 회원 조회라는 애플리케이션 작업을 조정한다. 이 예제에서는 Repository에서 회원을 조회하고 응답 형태로 변환한다. DTO 변환을 어느 계층에서 맡을지는 프로젝트 설계에 따라 달라질 수 있다.

Service는 보통 하나의 작업을 완성하기 위해 조회, 검증, 상태 변경과 저장의 흐름을 조정한다. 다만 `어떤 주문을 취소할 수 있는가`와 같은 업무 규칙 자체를 반드시 Service 안에 구현해야 하는 것은 아니다.

마지막으로 Service가 호출하는 Repository를 보자.

```java
public interface UserRepository extends JpaRepository<User, Long> {
}
```

Repository는 데이터 접근을 담당하는 **Persistence Layer**에 해당한다. 위 코드에서는 실제 저장과 조회를 Spring Data JPA에 맡긴다.

`UserResponse`는 Layer가 아니라 외부에 데이터를 전달하는 DTO다.

자료마다 Application, Business와 Domain Layer를 나누는 방식은 다르다. 이 글에서는 단순한 Spring 구조를 설명하기 위해 Service가 위치하는 영역을 Application/Business 영역으로 묶어 본다.

각 계층의 역할을 나누면 이들이 서로 어느 방향으로 참조하는지도 중요해진다.

## 계층 사이에는 방향이 있다

앞의 회원 조회 흐름에서 Controller는 Service를 알고, Service는 Repository를 안다. 반대로 Repository는 Controller를 참조하지 않는다.

의존 방향을 제한한다는 것은 각 Layer가 알아도 되는 세부사항의 범위를 정하는 일이다. Repository는 REST Controller의 DTO를 알 필요가 없고, Service는 HTTP 상태 코드를 알 필요가 없다.

이처럼 위쪽 계층이 아래쪽 계층을 참조하도록 방향을 정하는 것이 레이어드 아키텍처의 일반적인 모습이다. 바로 아래 계층만 호출하도록 엄격하게 제한할 수도 있고, 필요한 경우 한 계층을 건너뛰어 더 아래 계층을 호출하도록 허용할 수도 있다.

이 방향이 변경의 영향을 막아 주는 것은 아니다. Repository의 반환형이 바뀌면 Service도 수정될 수 있다. 대신 **누가 누구를 참조하는지 예측하기 쉬워진다.**

역할과 참조 방향은 정했지만 실제 코드를 어떤 폴더에 배치할지는 아직 남아 있다. 이때 레이어와 패키징을 같은 개념으로 혼동하기 쉽다.

## 레이어와 패키징은 다른 문제다

레이어드 아키텍처를 사용하면 반드시 최상위 폴더를 `controller`, `service`, `repository`로 나눠야 할까?

다음은 기술 역할별로 코드를 모은 **Package by Layer** 구조다.

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

구조가 단순할 때는 역할별 코드를 한눈에 보기 쉽다. 반면 주문 기능 하나를 수정하려면 여러 패키지를 오가야 한다.

반대로 업무 기능을 먼저 나누고 그 안에 Controller, Service와 Repository를 둘 수도 있다.

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

기능이나 업무 영역을 기준으로 최상위 패키지를 나누는 방식을 **Package by Feature/Domain** 형태로 구성할 수 있다. 패키지 안에서는 여전히 Controller가 Service를 호출하고 Service가 Repository를 호출한다.

즉 **레이어링은 책임과 의존의 문제이고, 패키징은 코드를 배치하는 문제다.**

## 이 구조가 널리 사용되는 이유

Controller, Service와 Repository라는 이름만으로도 코드의 역할과 요청 흐름을 어느 정도 예상할 수 있다. 이러한 예측 가능성은 다음과 같은 장점으로 이어진다.

### 서로 다른 관심사를 나누기 쉽다

Controller 하나에서 HTTP 요청 처리, 업무 조건 검사, SQL 실행과 JSON 응답 생성까지 맡으면 서로 다른 이유로 바뀌는 코드가 뒤섞인다.

이를 Controller, Service와 Repository로 나누면 각 코드를 이해할 때 살펴볼 범위가 줄어든다. 이를 **관심사의 분리**(Separation of Concerns)라고 부른다.

### 학습과 협업 비용이 낮다

Spring 예제와 많은 웹 애플리케이션이 비슷한 구조를 사용한다. 팀원이 새로 합류해도 익숙한 책임과 호출 흐름을 바탕으로 코드를 찾기 쉽다. 업무 흐름이 단순한 서비스에서는 이 예측 가능성만으로도 충분히 큰 장점이 된다.

다만 Layer를 나누는 것만으로 애플리케이션의 모든 설계 문제가 해결되지는 않는다.

## 업무 규칙이 많아지면 Service가 비대해질 수 있다

다음 코드는 결제된 주문을 완료 상태로 바꾸는 과정을 단순화한 예다.

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

지금은 주문 조회, 상태 검증과 변경만 조정한다. 여기에 가격 계산, 할인 판단과 재고 확인이 더해지면 `OrderService`는 여러 업무 규칙을 절차대로 실행하는 코드로 커질 수 있다.

레이어드 아키텍처가 알려주는 것은 **코드를 어느 책임의 Layer에 둘 것인가**다. 주문이라는 업무 개념을 어떤 객체와 규칙으로 표현할지까지 정해 주지는 않는다. 별도의 도메인 모델링 기준 없이 Layer만 적용하면 업무 규칙이 Service에 모이기 쉽다.

계층 구조를 유지하면서도 업무 규칙을 도메인 객체로 나눌 수 있다. 앞에서는 Service가 너무 많은 책임을 가지는 경우를 봤다. 반대로 계층을 형식적으로 유지하느라 Service가 거의 아무 일도 하지 않는 경우도 있다.

## 형식적인 계층 통과가 반복되면 Architecture Sinkhole이 된다

레이어드 아키텍처에서는 상위 계층이 바로 아래 계층을 거치도록 제한할 수 있다. 이를 **닫힌 계층**이라고 한다. Controller가 Repository를 직접 호출하지 않고 반드시 Service를 거치는 구조가 대표적이다.

앞에서 본 회원 조회 요청을 다시 살펴보자. Service에는 Repository 호출을 전달하는 코드만 있다.

```java
public UserResponse getUser(long id) {
    return userRepository.findResponseById(id)
        .orElseThrow();
}
```

이 메서드는 업무 규칙, 검증이나 데이터 변환 없이 요청을 다음 계층으로 넘긴다. 이처럼 대부분의 요청이 의미 있는 처리 없이 여러 계층을 통과하는 구조를 **Architecture Sinkhole Anti-Pattern**이라고 부른다.

메서드 하나가 얇다는 이유만으로 Sinkhole이라고 판단하지 않는다. Service가 트랜잭션, 인가나 여러 작업의 조정을 맡는다면 분명한 책임이 있다. 핵심은 **대부분의 요청이 의미 있는 처리 없이 Layer만 통과하는가**다.

> **심화 참고**
>
> 《Fundamentals of Software Architecture》는 단순 전달 요청이 약 20%라면 자연스러울 수 있지만, 약 80%라면 현재 구조를 점검할 신호라고 설명한다. 절대적인 기준이 아니라 애플리케이션 전체 흐름을 돌아보기 위한 경험칙이다.

Sinkhole을 줄이기 위해 일부 Layer를 열어 Controller가 Repository를 직접 호출하도록 허용할 수도 있다. 어느 쪽이 항상 옳은 것은 아니다.

| 선택 | 얻는 점 | 비용 |
| --- | --- | --- |
| 닫힌 Layer 유지 | 요청 흐름과 Application 경계가 일정함 | 단순 전달 코드가 생길 수 있음 |
| 일부 Layer 개방 | 의미 없는 전달을 줄일 수 있음 | Controller가 Persistence에 직접 의존함 |

프로젝트에서 Layer를 건너뛰도록 허용한다면 일관된 기준이 필요하다. Sinkhole의 해법은 Service를 무조건 없애는 것이 아니라, **구조의 일관성과 불필요한 통과 코드 사이에서 선택하는 것**이다.

## 계층을 나눴다고 기술과 분리된 것은 아니다

Controller와 Repository를 분리했더라도 Service가 외부 기술을 직접 알고 있을 수 있다.

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;
}
```

위 Service는 주문 작업을 조정하면서 Kafka의 구체적인 API도 직접 사용한다. 같은 방식으로 `JpaRepository`나 외부 API Client가 핵심 업무 코드까지 드러날 수 있다.

즉 **Layer 분리와 핵심 로직의 기술 독립성은 같은 문제가 아니다.** 레이어드 아키텍처는 기술 의존성을 자동으로 제거하지 않는다. 외부 기술과 핵심 로직 사이의 경계를 더 명확하게 만드는 방법은 이후 헥사고날 아키텍처에서 살펴볼 수 있다.

## 레이어드 아키텍처는 언제 잘 맞을까?

레이어드 아키텍처의 단순함은 분명한 장점이다. 다음 조건에서는 Controller, Service와 Repository만으로도 충분할 수 있다.

- 업무 규칙과 기능 흐름이 단순함
- 외부 기술이 바뀔 가능성과 영향이 크지 않음
- 팀이 전체 구조와 호출 흐름을 쉽게 파악할 수 있음
- 더 복잡한 구조가 주는 이점보다 학습과 유지 비용이 큼

반대로 Service 비대화, 형식적인 Layer 통과와 기술 변경의 영향이 반복된다면 다른 설계 기준을 보완할 이유가 생긴다.

프로젝트 크기만으로 아키텍처를 결정할 수는 없다. **업무 복잡도, 변경 빈도, 의존성의 복잡도와 팀이 감당할 구조 비용**을 함께 봐야 한다. 결국 더 많은 구조가 주는 이점이 그 복잡성보다 큰지가 선택 기준이다.

## 정리

레이어드 아키텍처는 서로 다른 기술적 책임을 Layer로 나누고, 코드가 참조하는 방향을 예측하기 쉽게 만든다. 구조가 단순하고 익숙하다는 점도 분명한 장점이다. 또한 레이어링은 책임과 의존의 문제이므로, Package by Layer뿐 아니라 기능별 패키징과도 함께 사용할 수 있다.

다만 레이어드 아키텍처만으로 다음 문제의 답까지 정해지지는 않는다.

- 복잡한 업무 규칙을 어떤 객체와 모델로 표현할 것인가?
- 모든 요청이 반드시 같은 Layer를 지나야 하는가?
- 핵심 업무 코드와 JPA, Kafka 같은 외부 기술을 어떻게 분리할 것인가?

레이어드 아키텍처는 단순해서 부족한 구조가 아니다. 단순함이 주는 이점이 현재 시스템에 충분하다면 좋은 선택이다. 다만 업무 규칙과 기술 의존성이 복잡해질수록 **Layer를 나누는 것만으로 충분한가**라는 다음 질문이 생긴다.

Service 안에 쌓인 업무 규칙을 객체 자체로 옮기려면 어떻게 해야 할까? Service가 JPA나 Kafka를 직접 알지 않게 만들 수 있을까? 이러한 경계와 의존 방향을 애플리케이션 전체에서 더 엄격하게 관리하면 어떤 구조가 될까? 이 질문은 각각 DDD, 헥사고날 아키텍처와 클린 아키텍처를 공부하는 출발점이 된다.

## 참고 자료

### 공식 자료

- [Microsoft Azure Architecture Center - N-tier Architecture Style](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/n-tier)
- [Spring Data JPA - Core Concepts](https://docs.spring.io/spring-data/jpa/reference/repositories/core-concepts.html)
- [O'Reilly - Fundamentals of Software Architecture](https://www.oreilly.com/library/view/fundamentals-of-software/9781492043447/)

### 추가 자료

- [Martin Fowler - Presentation Domain Data Layering](https://martinfowler.com/bliki/PresentationDomainDataLayering.html)
- [Herberto Graca - Layered Architecture](https://herbertograca.com/2017/08/03/layered-architecture/)
- [Klarc - Layered Architecture with Sinkhole](https://klarciel.net/wiki/architecture/architecture-layered/)
- [매일메일 - 레이어드 아키텍처란 무엇인가요?](https://www.maeil-mail.kr/question/310)
- [andrew.log - 그 서비스, 진짜 일하고 있나요?](https://velog.io/@sh1623/%EA%B7%B8-%EC%84%9C%EB%B9%84%EC%8A%A4-%EC%A7%84%EC%A7%9C-%EC%9D%BC%ED%95%98%EA%B3%A0-%EC%9E%88%EB%82%98%EC%9A%94-%EC%8B%B1%ED%81%AC%ED%99%80-%EC%95%88%ED%8B%B0%ED%8C%A8%ED%84%B4)
