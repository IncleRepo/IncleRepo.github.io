+++
title = 'Controller, Service, Repository로 이해하는 레이어드 아키텍처'
slug = '14'
date = 2026-08-21T20:49:00+09:00
lastmod = 2026-08-21T22:10:00+09:00
draft = false
references_required = true
description = 'Spring에서 익숙한 Controller, Service, Repository 구조를 따라가며 레이어드 아키텍처의 책임과 의존 방향, 장점과 한계를 살펴봅니다.'
categories = ['애플리케이션 아키텍처']
tags = ['레이어드 아키텍처', 'Controller', 'Service', 'Repository']
+++

Spring을 처음 배우면 대부분 다음과 같은 프로젝트 구조를 만나게 된다.

```text
user
├── controller
├── service
├── repository
├── entity
└── dto
```

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

Controller는 요청을 받고, Service는 필요한 업무를 처리하며, Repository는 데이터를 조회하거나 저장한다. 너무 익숙해서 단순한 Spring 관례처럼 느껴지지만, 사실 이 구조에도 아키텍처가 담겨 있다.

코드를 역할에 따라 나누고 서로 참조하는 방향을 정했기 때문이다. 우리가 자주 사용하던 이 구조가 **레이어드 아키텍처**(Layered Architecture)의 대표적인 모습이다.

## 폴더가 아니라 책임을 나눈 구조다

먼저 한 가지 오해를 피해야 한다. `controller`, `service`, `repository`라는 폴더를 만들었다고 레이어드 아키텍처가 완성되는 것은 아니다.

핵심은 폴더 이름보다 각 부분이 맡은 책임에 있다.

- Controller는 외부 요청을 애플리케이션이 이해할 형태로 받고 처리 결과를 반환한다.
- Service는 요청을 처리하는 흐름과 업무 규칙을 담당한다.
- Repository는 데이터 저장과 조회에 필요한 접근을 제공한다.

예를 들어 주문 서비스에서 `결제가 끝난 주문만 취소할 수 있다`, `회원 등급에 따라 할인 금액을 계산한다`, `재고가 있어야 주문할 수 있다`와 같은 조건은 서비스가 지켜야 할 업무 규칙이다. 이런 서비스 고유의 규칙을 흔히 **비즈니스 로직**이라고 부른다.

패키지는 이러한 책임 분리를 코드에 표현하는 한 가지 수단이다. 그리고 DTO는 Controller나 Service 같은 계층이 아니다. **DTO**(Data Transfer Object)는 외부와 데이터를 주고받거나 계층 사이에서 데이터를 전달하기 위한 객체다.

## 회원 조회 요청을 따라가 보자

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

Controller는 URL에서 회원 ID를 받아 Service에 전달한다. 회원을 어디에서 조회하고 어떤 형태로 조립할지는 Service에 맡긴다.

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

Service는 Repository를 통해 회원을 조회하고 API 응답에 사용할 DTO로 변환한다. Repository는 실제 데이터 접근을 Spring Data JPA에 맡긴다.

```java
public interface UserRepository extends JpaRepository<User, Long> {
}
```

회원 조회라는 하나의 작업도 요청 처리, 업무 흐름과 데이터 접근을 서로 다른 코드가 맡는다. 각 역할에 일반적으로 사용하는 계층 이름을 붙이면 다음과 같다.

- 외부 요청과 응답을 다루는 부분은 **Presentation Layer**다. REST API에서는 Controller가 대표적이다.
- 애플리케이션 흐름과 업무 처리를 담당하는 부분은 **Application Layer** 또는 **Business Layer**라고 부른다.
- 데이터 저장과 조회를 담당하는 부분은 **Persistence Layer** 또는 **Data Access Layer**라고 부른다. Repository나 DAO가 여기에 해당한다.

자료에 따라 Business Layer와 Application Layer를 구분하고 별도의 Domain Layer를 두기도 한다. 여기서는 이해하기 쉽도록 Service가 맡는 부분을 Business/Application Layer로 묶어서 보자. Database도 별도 계층으로 그리거나 Persistence Layer 바깥의 저장소로 보기도 한다.

![User Interface부터 Data까지 여러 계층으로 나눈 레이어드 아키텍처](/images/posts/layered-architecture/layered-architecture.png "계층을 더 세분화한 레이어드 아키텍처의 한 가지 예")

위 그림처럼 계층을 더 세분화할 수도 있다. 중요한 것은 3계층인지 4계층인지 외우는 일이 아니다. **서로 다른 책임을 나누고 각 책임 사이의 관계를 정했다는 점**이 핵심이다.

## 계층 사이에는 방향이 있다

앞의 회원 조회 코드를 다시 보면 참조 방향이 눈에 들어온다.

```text
Controller
   ↓
Service
   ↓
Repository
   ↓
DB
```

Controller는 Service를 알고, Service는 Repository를 안다. 반대로 Repository는 Controller가 존재하는지 몰라도 데이터를 조회할 수 있다.

Repository를 작성할 때 외부 요청이 REST API인지 GraphQL인지까지 알 필요는 없다. 데이터 저장과 조회에만 집중하면 된다. Service 역시 요청을 보낸 클라이언트가 웹인지 모바일인지 몰라도 회원 조회 흐름을 처리할 수 있다.

이처럼 위쪽 계층이 아래쪽 계층을 참조하도록 방향을 정하는 것이 레이어드 아키텍처의 일반적인 모습이다. 바로 아래 계층만 호출하도록 엄격하게 제한할 수도 있고, 필요한 경우 한 계층을 건너뛰어 더 아래 계층을 호출하도록 허용할 수도 있다.

다만 참조 방향이 한쪽이라고 해서 코드 변경의 영향까지 한쪽으로만 전달되는 것은 아니다. Repository의 메서드 반환형을 바꾸면 이를 사용하는 Service도 수정해야 할 수 있다. 의존 방향의 목적은 변경을 없애는 것이 아니라, **누가 누구를 참조하는지 예측할 수 있게 만들어 구조를 단순하게 유지하는 데 있다.**

## 이 구조가 널리 사용되는 이유

이 구조가 널리 사용되는 데에는 분명한 이유가 있다.

### 역할을 이해하기 쉽다

요청을 처리하는 코드는 Controller, 업무를 처리하는 코드는 Service, 데이터를 다루는 코드는 Repository에서 찾을 수 있다. 처음 보는 프로젝트에서도 클래스가 맡은 역할을 이름만으로 어느 정도 예상할 수 있다.

### 한 클래스에 서로 다른 코드가 섞이는 일을 줄인다

Controller 하나에서 HTTP 요청 값을 읽고, 업무 조건을 검사하고, SQL을 실행한 뒤 JSON 응답까지 만든다고 생각해보자. 기능이 조금만 늘어도 서로 다른 이유로 바뀌는 코드가 한곳에 뒤섞인다.

이를 Controller, Service와 Repository로 나누면 각 코드를 이해할 때 살펴볼 범위가 줄어든다. 서로 다른 종류의 관심사를 분리한다는 의미에서 이를 **관심사의 분리**(Separation of Concerns)라고 부른다.

### 학습과 협업 비용이 낮다

Spring 예제와 많은 웹 애플리케이션이 비슷한 구조를 사용한다. 팀원이 새로 합류해도 익숙한 책임과 호출 흐름을 바탕으로 코드를 찾기 쉽다. 작은 CRUD 서비스에서는 이 단순함만으로도 충분히 큰 장점이 된다.

하지만 프로젝트가 커지면 `Controller`, `Service`, `Repository`로 나눴다는 사실만으로 해결되지 않는 문제도 드러난다.

## Service에 업무 규칙이 계속 쌓일 수 있다

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

Controller와 Repository에서 코드를 분리했어도 Service 안에서는 `조회 → if문 → setter → 저장`이라는 절차가 계속 길어진다. 레이어드 아키텍처는 기술 역할에 따라 코드를 어디에 둘지는 알려주지만, **복잡한 업무 규칙을 어떤 객체와 모델로 표현할지까지 정해 주지는 않는다.**

따라서 Service가 비대해진다고 해서 레이어드 아키텍처가 잘못된 것은 아니다. 계층 분리와 도메인 모델링은 서로 다른 문제다.

## 계층을 나눠도 Service가 JPA와 HTTP 구조를 알 수 있다

Controller, Service, Repository를 서로 다른 패키지에 두었다고 해서 각 계층의 기술까지 자동으로 분리되는 것은 아니다. 다음 `OrderService`를 보자.

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final JpaOrderRepository orderRepository;

    public void cancel(long orderId) {
        OrderEntity order = orderRepository.findById(orderId)
            .orElseThrow();

        order.cancel();
    }
}
```

`OrderService`는 Repository가 주문을 찾아준다는 사실만 아는 것이 아니다. JPA에서 사용하는 `OrderEntity`와 `JpaOrderRepository`도 직접 알고 있다. 이 구조가 항상 잘못된 것은 아니지만, 저장 기술이나 Entity 구조가 바뀌면 Service도 함께 수정될 가능성이 커진다.

필요하다면 Service가 원하는 저장 기능을 별도의 인터페이스로 표현할 수 있다.

```java
public interface OrderRepository {

    Order findById(long orderId);
}
```

JPA를 사용하는 클래스는 이 인터페이스를 구현하고, Service는 `OrderRepository`와 `Order`만 사용한다. 그러면 JPA Entity의 필드나 조회 방식이 바뀌더라도 변경을 Repository 구현 안에서 처리할 여지가 생긴다.

다만 다음처럼 인터페이스가 JPA Entity를 그대로 반환한다면 이름만 바뀌었을 뿐 Service의 JPA 의존성은 남아 있다.

```java
public interface OrderRepository {

    OrderEntity findById(long orderId);
}
```

API 요청 DTO를 안쪽 계층까지 그대로 전달하는 경우도 같은 문제를 만든다.

```java
@PostMapping("/orders")
public void create(@RequestBody CreateOrderRequest request) {
    orderService.create(request);
}
```

`CreateOrderRequest`는 HTTP 요청을 받기 위한 객체다. Service와 Repository까지 이 객체를 사용하면 JSON 필드 이름이나 요청 형식이 바뀔 때 안쪽 코드도 함께 수정해야 한다. 필요하다면 Controller에서 Service가 사용할 입력으로 변환해 전달할 수 있다.

```java
orderService.create(
    new CreateOrderCommand(request.productCode(), request.quantity())
);
```

결국 패키지를 나누는 것만으로는 부족하다. Service가 HTTP 요청 형식과 JPA Entity를 직접 알아도 되는지, 변경 영향을 줄이기 위해 별도의 입력이나 저장 계약이 필요한지를 함께 결정해야 한다. 작은 CRUD에서는 구조를 단순하게 유지하는 편이 나을 수도 있다. 반대로 기술 변경의 영향이 업무 코드까지 자주 퍼진다면 의존성을 더 엄격하게 나누는 방법을 고민할 수 있다.

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

업무 기능을 먼저 나누고 그 안에서 계층을 구성할 수도 있다.

```text
user
├── controller
├── service
└── repository

order
├── controller
├── service
└── repository
```

이 구조는 **Package by Feature** 또는 **Package by Domain**이라고 부른다. 최상위 폴더가 달라졌을 뿐, 각 기능 안에서는 여전히 Controller가 Service를 호출하고 Service가 Repository를 호출한다. 따라서 기능별 패키징을 선택해도 레이어드 아키텍처를 사용할 수 있다.

정리하면 레이어드 아키텍처는 책임과 의존 방향에 관한 구조이고, Package by Layer는 파일을 기술 역할별 폴더에 모으는 방법이다. 둘은 함께 자주 보이지만 같은 개념은 아니다.

## 모든 요청이 모든 계층을 지나야 할까?

계층을 엄격하게 지키려다 보면 Service가 Repository 호출을 그대로 전달하기만 하는 코드가 생길 수 있다.

```java
public UserResponse getUser(long id) {
    return userRepository.findResponseById(id)
        .orElseThrow();
}
```

Controller도 Service를 호출할 뿐이다.

```java
@GetMapping("/users/{id}")
public UserResponse getUser(@PathVariable long id) {
    return userService.getUser(id);
}
```

여기서 Service는 실제로 무엇을 하고 있을까?

계층은 존재하지만 요청이 의미 있는 처리 없이 다음 계층으로 그대로 통과하는 상황을 **Architecture Sinkhole Pattern**이라고 부르기도 한다. 핵심 문제는 메서드 호출 한 번의 CPU나 메모리 비용이 아니다. 형식을 유지하기 위한 전달 코드가 계속 늘어나면 실제 책임이 어디에 있는지 흐려지고 수정할 파일만 많아질 수 있다는 점이다.

그렇다고 얇은 Service를 발견하는 즉시 없애야 하는 것은 아니다. 트랜잭션이나 인가를 한곳에서 처리하고 호출 경계를 일정하게 유지하기 위해 얇은 Service를 허용하는 팀도 있다. 중요한 것은 모든 요청이 무조건 모든 계층을 지나야 한다고 생각하기보다, **그 계층이 맡은 책임이 실제로 있는지 확인하는 일**이다.

## 레이어드 아키텍처는 언제 잘 맞을까?

레이어드 아키텍처는 오래됐기 때문에 버려야 하는 구조가 아니다. 역할과 흐름이 단순하고 많은 개발자에게 익숙하다는 장점은 지금도 유효하다.

다음과 같은 조건에서는 Controller, Service와 Repository만으로도 충분히 이해하기 쉽고 생산적인 구조를 만들 수 있다.

- 작은 CRUD 애플리케이션
- 업무 규칙과 기능 흐름이 단순한 서비스
- 개발 인원이 적고 모두가 전체 구조를 이해할 수 있는 프로젝트
- 복잡한 구조를 도입했을 때 얻는 이점보다 학습과 유지 비용이 더 큰 경우

반대로 Service에 업무 규칙이 계속 쌓이고, JPA나 외부 API 같은 기술 변경이 핵심 코드까지 흔들며, 형식적인 전달 코드가 늘어난다면 계층 분리 다음의 문제를 고민할 시점이다.

아키텍처는 유명한 이름보다 프로젝트가 가진 문제와 복잡도에 맞춰 선택해야 한다. 단순한 문제에는 단순한 구조가 더 좋은 답일 수 있다.

## 계층을 나눈 다음에는 무엇을 고민해야 할까?

Controller, Service와 Repository로 요청 처리, 업무 흐름과 데이터 접근의 책임은 나눴다. 하지만 이 구조만으로 모든 설계 문제가 해결되지는 않는다.

- Service에 비즈니스 규칙이 계속 쌓인다면, 비즈니스 로직 자체는 어디에 두는 것이 좋을까?
- Service가 JPA Repository나 `KafkaTemplate` 같은 구체적인 기술을 직접 알아야 할까?
- 계층의 경계와 의존 방향을 코드에서 더 엄격하게 관리하려면 어떻게 해야 할까?

이 질문들은 레이어드 아키텍처가 틀렸다는 뜻이 아니다. **계층을 나누는 것만으로는 비즈니스 모델을 설계하거나 외부 기술로부터 핵심 로직을 보호하는 문제까지 해결할 수 없다는 뜻**이다.

## 정리

- Controller, Service와 Repository 구조는 서로 다른 책임을 계층으로 나눈 레이어드 아키텍처의 대표적인 모습이다.
- Controller는 외부 요청과 응답, Service는 업무 흐름, Repository는 데이터 접근을 담당한다.
- 패키지 이름보다 책임 분리와 의존 방향이 핵심이며, DTO는 별도의 계층이 아니라 데이터 전달 객체다.
- 레이어드 아키텍처는 역할을 이해하고 코드를 찾기 쉬워 작은 서비스와 단순한 업무에 효과적이다.
- 계층만 나눈다고 좋은 도메인 모델이나 기술과의 결합이 낮은 구조가 자동으로 만들어지지는 않는다.
- 모든 요청이 모든 계층을 형식적으로 통과해야 하는 것은 아니며, 각 계층이 실제 책임을 맡고 있는지 살펴봐야 한다.

Controller-Service-Repository 구조가 익숙한 이유는 코드를 기술 역할에 따라 나누기 쉽기 때문이다. 이 구조는 단순하고 이해하기 쉽다는 큰 장점이 있다. 다만 시스템이 복잡해질수록 “**계층을 나눈 뒤에는 무엇을 고민해야 할까**?”라는 질문이 생긴다.

## 참고 자료

### 공식 자료

- [Microsoft Azure Architecture Center - N-tier Architecture Style](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/n-tier)
- [Spring Data JPA - Core Concepts](https://docs.spring.io/spring-data/jpa/reference/repositories/core-concepts.html)

### 추가 자료

- [Martin Fowler - Presentation Domain Data Layering](https://martinfowler.com/bliki/PresentationDomainDataLayering.html)
- [Herberto Graca - Layered Architecture](https://herbertograca.com/2017/08/03/layered-architecture/)
