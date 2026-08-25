+++
title = 'Controller, Service, Repository로 이해하는 레이어드 아키텍처'
slug = '14'
date = 2026-08-21T20:49:00+09:00
lastmod = 2026-08-25T20:20:00+09:00
draft = false
references_required = true
description = 'Spring에서 익숙한 Controller, Service, Repository 구조를 따라가며 레이어드 아키텍처의 책임과 의존 방향, 장점과 한계를 살펴봅니다.'
categories = ['애플리케이션 아키텍처']
tags = ['레이어드 아키텍처', 'Controller', 'Service', 'Repository']
+++

Spring을 처음 배우면 대부분 역할에 따라 패키지를 나눈 프로젝트 구조를 만나게 된다.

![Controller, DTO, Repository, Service로 나눈 패키지 구조](/images/posts/layered-architecture/project-directory.png "Spring에서 자주 만나는 패키지 구조")

요청은 대체로 다음 순서로 처리된다.

```text
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

Controller는 요청, Service는 업무 처리, Repository는 데이터 접근을 맡는다. 단순한 Spring 관례처럼 보이지만, 역할과 참조 방향을 나눈 **레이어드 아키텍처**(Layered Architecture)의 대표적인 모습이다.

## 레이어드 아키텍처란 무엇인가

레이어드 아키텍처는 코드를 책임에 따라 계층으로 나누고 계층 사이의 관계를 정하는 구조다.

![User Interface부터 Data까지 여러 계층으로 나눈 레이어드 아키텍처](/images/posts/layered-architecture/layered-architecture.png "계층을 더 세분화한 레이어드 아키텍처의 한 가지 예")

위 그림은 레이어드 아키텍처를 비교적 세분화한 예다. 그림 안의 계층을 역할에 따라 묶으면 다음과 같다.

- **User Interface와 Presentation**은 사용자와의 상호작용을 맡는다.
- **Application과 Domain Model**은 작업의 흐름과 업무 규칙을 처리한다.
- **Persistence와 Data**는 데이터 접근과 실제 저장을 담당한다.

오른쪽의 Infrastructure는 Framework와 Logging처럼 여러 계층의 구현을 지원하는 기술을 모아 표현한 부분이다.

프로젝트마다 계층의 수와 이름은 다르다. Spring 애플리케이션에서는 Controller, Service와 Repository로 단순하게 구성하는 경우가 많다. 이제 회원 조회 요청 하나가 이 세 계층을 어떻게 지나는지 살펴보자.

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

Controller는 URL에서 회원 ID를 꺼내 Service에 전달하고 처리 결과를 HTTP 응답으로 반환한다. 이처럼 외부 요청과 응답을 다루는 부분을 **Presentation Layer**라고 하며, REST API에서는 Controller가 대표적이다.

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

Service는 회원 조회라는 작업의 흐름을 맡는다. Repository에서 회원을 찾고 그 결과를 API 응답에 사용할 형태로 변환한다. 이런 애플리케이션의 작업 흐름을 다루는 부분을 **Application Layer** 또는 **Business Layer**라고 부른다.

지금 예제의 흐름은 단순하지만 실제 Service에는 업무 규칙이 함께 들어갈 수 있다. `결제가 끝난 주문만 취소할 수 있다`, `회원 등급에 따라 할인 금액을 계산한다`, `재고가 있어야 주문할 수 있다`와 같은 조건이 대표적이다. 이런 업무의 조건과 계산 규칙을 흔히 **비즈니스 로직**이라고 부른다.

마지막으로 Service가 호출하는 Repository를 보자.

```java
public interface UserRepository extends JpaRepository<User, Long> {
}
```

Repository는 회원을 어떻게 저장하고 조회할지에 필요한 접근 방법을 제공한다. 이 부분을 **Persistence Layer** 또는 **Data Access Layer**라고 하며 Repository와 DAO가 여기에 해당한다. 위 코드에서는 실제 데이터 접근을 Spring Data JPA에 맡긴다.

코드에 등장한 `UserResponse`는 계층이 아니다. **DTO**(Data Transfer Object)는 외부와 데이터를 주고받거나 계층 사이에서 데이터를 전달하기 위한 객체다.

자료에 따라 Application, Business와 Domain Layer를 구분하는 방식이나 Database의 위치는 달라질 수 있다. 이름보다 각 부분이 맡는 책임을 구분하는 일이 중요하다.

각 계층의 역할을 나누면 이들이 서로 어느 방향으로 참조하는지도 중요해진다.

## 계층 사이에는 방향이 있다

앞의 회원 조회 흐름에서 Controller는 Service를 알고, Service는 Repository를 안다. 반대로 Repository는 Controller가 존재하는지 몰라도 데이터를 조회할 수 있다.

Repository는 외부 요청이 REST API인지 GraphQL인지 몰라도 데이터를 조회할 수 있다. Service도 클라이언트가 웹인지 모바일인지 몰라도 회원 조회를 처리할 수 있다.

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

이를 **Package by Feature** 또는 **Package by Domain**이라고 부른다. 최상위 폴더가 달라졌을 뿐, 각 기능 안에서는 여전히 Controller가 Service를 호출하고 Service가 Repository를 호출한다. 따라서 기능별 패키징을 선택해도 레이어드 아키텍처를 사용할 수 있다.

레이어드 아키텍처는 책임과 호출 방향에 관한 구조이고, Package by Layer는 파일을 배치하는 방법이다. 기능별 패키징 안에서도 레이어드 아키텍처를 사용할 수 있다.

## 이 구조가 널리 사용되는 이유

Controller, Service와 Repository라는 이름만으로도 코드의 역할과 요청 흐름을 어느 정도 예상할 수 있다. 이러한 예측 가능성은 다음과 같은 장점으로 이어진다.

### 서로 다른 관심사를 나누기 쉽다

Controller 하나에서 HTTP 요청 처리, 업무 조건 검사, SQL 실행과 JSON 응답 생성까지 맡으면 서로 다른 이유로 바뀌는 코드가 뒤섞인다.

이를 Controller, Service와 Repository로 나누면 각 코드를 이해할 때 살펴볼 범위가 줄어든다. 이를 **관심사의 분리**(Separation of Concerns)라고 부른다.

### 학습과 협업 비용이 낮다

Spring 예제와 많은 웹 애플리케이션이 비슷한 구조를 사용한다. 팀원이 새로 합류해도 익숙한 책임과 호출 흐름을 바탕으로 코드를 찾기 쉽다. 작은 CRUD 서비스에서는 이 단순함만으로도 충분히 큰 장점이 된다.

프로젝트가 커지고 업무 규칙이 복잡해지면 이 세 계층만으로는 부족한 지점도 나타난다.

## 규모가 커지면 Service에 업무 규칙이 쌓일 수 있다

다음 코드는 결제된 주문을 완료 상태로 바꾸는 과정을 단순화한 예다.

```java
@Transactional
public void completeOrder(long orderId) {
    Order order = orderRepository.findById(orderId)
        .orElseThrow();

    if (!"PAID".equals(order.getStatus())) {
        throw new IllegalStateException();
    }

    order.setStatus("COMPLETED");
}
```

기능이 적을 때는 이 정도 코드도 이해하기 어렵지 않다. 하지만 주문 기능이 늘어나면 상태 확인, 가격 계산, 할인 판단, 재고 확인과 상태 변경이 모두 `OrderService`에 쌓일 수 있다.

```text
OrderService
→ 주문 조회
→ 상태 검증
→ 가격과 할인 계산
→ 재고 확인
→ 주문 상태 변경
→ 저장
```

Controller와 Repository에서 코드를 분리했어도 Service 안에서는 `조회 → if문 → setter → 저장`이라는 절차가 길어진다. 레이어드 아키텍처는 **복잡한 업무 규칙을 어떤 객체로 표현할지까지 정해 주지는 않기 때문이다.**

계층 구조를 유지하면서 업무 규칙을 도메인 객체로 나눌 수 있다. 반대로 업무 규칙이 거의 없는 요청까지 모든 계층을 거치게 하면 Service가 단순 전달만 맡는 경우도 생긴다.

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

### 단순 전달 요청이 차지하는 비율을 본다

메서드 하나가 얇다는 이유만으로 Sinkhole이라고 판단하지 않는다. Service가 트랜잭션, 인가나 여러 작업의 조정을 수행한다면 분명한 책임이 있다.

Repository 호출과 결과만 전달하는 요청이 **전체에서 얼마나 반복되는지** 확인해야 한다. 《소프트웨어 아키텍처 101》에서는 다음과 같은 경험적 기준을 제시한다.

| 단순 전달 요청의 비율 | 해석 |
| --- | --- |
| 약 20% | 레이어드 구조에서 자연스럽게 생길 수 있는 수준 |
| 약 80% | 현재 문제에 계층 구조가 잘 맞는지 점검할 신호 |

이 숫자는 합격선을 정하는 공식이 아니라 구조를 점검하기 위한 기준이다.

### 단순 조회에는 계층을 열 수도 있다

단순 전달 요청이 일부라면 계층의 일관성을 위해 Service를 유지할 수 있다. 대부분의 요청이 같은 형태라면 일부 계층을 건너뛰도록 허용하는 방법도 검토할 수 있다.

예를 들어 단순 조회는 Controller가 Repository를 직접 호출하도록 정할 수 있다.

```java
@GetMapping("/users/{id}")
public UserResponse getUser(@PathVariable long id) {
    return userRepository.findResponseById(id)
        .orElseThrow();
}
```

```text
Controller → Repository → DB
```

이 방식은 전달 메서드를 줄이는 대신 Controller가 Repository와 조회 결과에 직접 의존한다. 이후 트랜잭션이나 업무 규칙이 필요해지면 호출 경계를 다시 정리해야 할 수 있다.

따라서 `단순 조회만 허용`처럼 계층을 건너뛸 범위를 팀 규칙으로 정한다. 개발자가 메서드마다 임의로 경로를 선택하게 두면 요청 흐름을 예측하기 어려워진다.

레이어드 아키텍처는 책임이 한 계층에 몰리는 상황과 책임 없이 계층만 통과하는 상황을 모두 경계해야 한다. 그렇다면 어떤 프로젝트에서 이 단순한 구조의 장점이 한계보다 클까?

## 레이어드 아키텍처는 언제 잘 맞을까?

다음과 같은 조건에서는 Controller, Service와 Repository만으로도 충분하다.

- 작은 CRUD 애플리케이션
- 업무 규칙과 기능 흐름이 단순한 서비스
- 개발 인원이 적고 모두가 전체 구조를 이해할 수 있는 프로젝트
- 복잡한 구조를 도입했을 때 얻는 이점보다 학습과 유지 비용이 더 큰 경우

반대로 다음과 같은 신호가 반복된다면 계층을 나눈 다음의 설계 과제를 고민할 시점이다.

- Service에 업무 규칙이 계속 쌓이면 도메인 책임과 모델을 다시 살펴본다.
- 형식적인 전달 코드가 많아지면 모든 요청이 같은 계층을 거쳐야 하는지 검토한다.

계층을 더 추가하기보다 실제로 복잡해진 책임과 경계를 보완해야 한다.

## 정리

- 레이어드 아키텍처는 코드를 책임에 따라 나누며, Spring에서는 Controller가 요청과 응답, Service가 업무 흐름, Repository가 데이터 접근을 맡는 구조가 대표적이다.
- 레이어드 아키텍처는 책임과 의존 방향에 관한 구조이고, Package by Layer는 파일을 기술 역할별로 배치하는 방법이다. 기능별 패키지 안에서도 레이어드 아키텍처를 사용할 수 있다.
- 대부분의 요청이 의미 있는 처리 없이 계층을 통과하면 Architecture Sinkhole이 된다. 단순 조회에서 Service를 생략하더라도 허용 범위와 의존 규칙을 팀에서 함께 정해야 한다.

더 엄격하게 책임과 기술 의존성을 분리하는 방법은 클린 아키텍처와 헥사고날 아키텍처에서 다시 살펴본다.

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
