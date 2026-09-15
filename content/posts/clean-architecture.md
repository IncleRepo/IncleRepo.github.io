+++
title = '업무 규칙을 안쪽에 두는 클린 아키텍처'
slug = '17'
date = 2026-09-14T16:20:00+09:00
lastmod = 2026-09-15T16:43:00+09:00
draft = false
references_required = true
description = '익숙한 주문 취소 Service에서 출발해 업무 규칙과 외부 구현을 분리하며, 클린 아키텍처의 구성요소와 의존성 규칙을 살펴봅니다.'
categories = ['애플리케이션 아키텍처']
tags = ['클린 아키텍처', '의존성 역전', 'Use Case', '소프트웨어 설계']
+++

주문 취소 조건은 그대로인데, API 응답이나 저장 방식을 바꾸면서 취소 처리를 담당하는 Service까지 고쳐야 할 때가 있다. 업무 규칙을 처리하는 코드에 HTTP나 JPA 사용법이 섞여 있으면 서로 다른 이유로 같은 코드를 수정하게 된다.

**클린 아키텍처**(Clean Architecture)는 이런 영향을 줄이기 위해 업무 규칙과 외부 기술을 분리한다. 로버트 C. 마틴의 [The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)를 바탕으로, 주문 취소 Service 안의 코드를 나누어 보며 각 부분의 역할과 연결 방식을 살펴보자.

## 주문 취소 Service에 무엇이 들어 있을까

결제 완료 상태의 주문만 취소할 수 있다고 해보자. 이 조건을 확인하는 코드와 JPA 조회, HTTP 응답 처리를 한 Service에 넣으면 다음과 같다.

```java
@Service
@RequiredArgsConstructor
public class CancelOrderService {

    private final JpaOrderRepository orders;

    @Transactional
    public ResponseEntity<Void> cancel(long orderId) {
        OrderJpaEntity order = orders.findById(orderId)
            .orElseThrow(() -> new IllegalArgumentException("주문이 없습니다."));

        if (order.getStatus() != OrderStatus.PAID) {
            throw new IllegalStateException("결제 완료 주문만 취소할 수 있습니다.");
        }

        order.setStatus(OrderStatus.CANCELED);
        return ResponseEntity.noContent().build();
    }
}
```

Service는 주문을 조회해 취소 상태로 바꾸고 HTTP 응답을 반환한다. 트랜잭션 안에서 변경한 JPA Entity의 상태는 Dirty Checking을 통해 DB에 반영된다. 이 과정을 역할별로 나누면 세 가지 일이 보인다.

- 결제 완료 주문만 취소할 수 있다는 **업무 규칙**.
- 주문을 찾아 취소하는 **작업의 순서**.
- JPA로 조회하고 HTTP 응답을 만드는 **기술별 처리**.

이 기능을 콘솔 기반 관리자 도구에서도 사용하려면 어떨까? 주문을 찾아 취소하는 과정은 그대로 쓸 수 있지만, 필요하지 않은 HTTP 응답 객체까지 반환받는다. 취소 조건만 따로 테스트하고 싶어도 JPA Repository가 반환할 객체부터 준비해야 한다.

취소 규칙을 HTTP와 JPA에서 분리하면 이런 준비 없이도 규칙을 사용하고 검증할 수 있다. 그 경계를 어디에 둘지 보여주는 것이 클린 아키텍처다.

## 업무 규칙을 중심으로 안과 밖을 나눈다

클린 아키텍처에서는 업무 규칙을 안쪽에, 외부 기술을 다루는 코드를 바깥쪽에 둔다. 주문 예제라면 취소 조건과 처리 절차를 안쪽에 남기고, HTTP 요청을 해석하거나 DB에 저장하는 코드는 바깥으로 옮기는 식이다. 원문은 이 구조를 네 영역으로 표현한다.

![Entities를 중심으로 Use Cases, Interface Adapters, Frameworks and Drivers가 둘러싼 클린 아키텍처 원문 그림](https://blog.cleancoder.com/uncle-bob/images/2012-08-13-the-clean-architecture/CleanArchitecture.jpg "Robert C. Martin의 클린 아키텍처 구성도")

출처: [Robert C. Martin — The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

그림의 가운데부터 주문 취소 예제에 대입해 보자.

| 영역 | 주문 취소에서 맡을 일 |
|---|---|
| **Entities** | 주문의 상태와 취소 조건을 관리한다 |
| **Use Cases** | 주문을 찾아 취소하고 저장하는 작업을 진행한다 |
| **Interface Adapters** | HTTP 요청과 DB 데이터를 업무 코드에 맞게 변환한다 |
| **Frameworks & Drivers** | Spring MVC, Hibernate, DB 등 실제 기술과 실행 환경을 제공한다 |

네 영역을 연결할 때는 **소스코드의 의존성이 안쪽으로 향하도록 한다.** 이를 **의존성 규칙**(Dependency Rule)이라고 부른다. 예를 들어 취소 조건을 판단하는 코드에서 JPA를 참조하는 대신, JPA를 다루는 쪽이 업무 코드에서 정한 계약을 구현하도록 만든다.

먼저 Service 안에 있던 취소 규칙부터 옮겨 보자. 이후 예제는 역할을 나누고 연결하는 데 집중할 수 있도록 트랜잭션 설정·동시성 제어·HTTP 예외 처리를 생략했다.

## 주문의 취소 규칙을 Entity에 담는다

취소할 수 있는지 확인하고 상태를 바꾸는 일은 주문 객체에 맡길 수 있다. Service에 있던 조건문과 상태 변경 코드를 `Order.cancel()`로 옮겨 보자.

```java
public class Order {

    private final long id;
    private OrderStatus status;

    public Order(long id, OrderStatus status) {
        this.id = id;
        this.status = java.util.Objects.requireNonNull(status);
    }

    public void cancel() {
        if (status != OrderStatus.PAID) {
            throw new IllegalStateException("결제 완료 주문만 취소할 수 있습니다.");
        }
        status = OrderStatus.CANCELED;
    }

    public long id() { return id; }
    public OrderStatus status() { return status; }
}

public enum OrderStatus {
    PAID, SHIPPED, CANCELED
}
```

`cancel()`을 호출하면 `PAID`인 주문은 `CANCELED`로 바뀌고, 이미 `SHIPPED`인 주문은 예외를 던지며 기존 상태를 유지한다. 주문 객체가 자신의 상태로 판단하므로 HTTP 요청이나 DB 연결 없이도 이 규칙을 실행할 수 있다.

이처럼 상태와 핵심 업무 규칙을 담는 부분이 **Entity**다. 마틴은 여러 애플리케이션에서 공통으로 사용할 수 있는 업무 규칙을 이 영역에 둔다. 예제에서도 고객 화면과 관리자 도구의 처리 절차는 달라질 수 있지만, 취소 조건은 같은 `Order`를 통해 확인할 수 있다.

## Use Case가 주문 취소 작업을 진행한다

취소 규칙을 `Order`에 담았으니, 이제 이 객체로 주문 취소 기능을 완성할 차례다. 요청으로 받은 ID로 주문을 찾고, `cancel()`을 호출한 뒤 변경된 상태를 저장해야 한다.

주문 취소처럼 애플리케이션이 제공하는 하나의 작업을 **Use Case**라고 하며, 여기서는 `CancelOrderService`가 이를 구현한다. 고객 본인의 주문인지 확인하거나 관리자 권한을 검사하는 등, 해당 작업에 필요한 절차도 이 Service에서 조정할 수 있다.

### 조회와 저장은 계약으로 요청한다

이 Service에서도 JPA를 분리하려면 조회와 저장을 맡길 방법이 필요하다. Service 쪽에서 필요한 작업을 인터페이스로 정하고, 실제 JPA 코드는 그 인터페이스를 구현하도록 하자.

`OrderGateway`에는 ID로 주문 데이터를 조회하는 메서드와 변경된 데이터를 저장하는 메서드를 둔다.

```java
public record OrderData(long id, OrderStatus status) {}

public interface OrderGateway {
    java.util.Optional<OrderData> findById(long orderId);
    void save(OrderData data);
}
```

조회 결과를 JPA Entity로 돌려주면 Service도 그 타입에 의존한다. 그래서 여기서는 주문 ID와 상태만 담은 `OrderData`를 주고받는다. 이 인터페이스와 전달용 객체는 Use Case 쪽에 두고, 주문 상태를 나타내는 `OrderStatus`는 Entity 쪽에 둔다.

### Service에는 취소 절차를 구현한다

조회와 저장을 요청할 방법이 생겼으니, 이를 사용해 취소 절차를 구현해 보자. `CancelOrderService`는 `OrderGateway`로 주문을 조회하고 `Order`에 취소를 요청한 뒤, HTTP 형식과 무관한 처리 결과를 반환한다.

```java
public class CancelOrderService {

    private final OrderGateway orders;

    public CancelOrderService(OrderGateway orders) {
        this.orders = orders;
    }

    public CancelOrderResult cancel(long orderId) {
        OrderData data = orders.findById(orderId)
            .orElseThrow(() -> new IllegalArgumentException("주문이 없습니다."));

        Order order = new Order(data.id(), data.status());
        order.cancel();

        orders.save(new OrderData(order.id(), order.status()));
        return new CancelOrderResult(order.id(), order.status().name());
    }
}

public record CancelOrderResult(long orderId, String status) {}
```

조회한 데이터로 `Order`를 만들면 앞에서 정의한 `cancel()`을 사용할 수 있다. 취소에 성공한 경우에만 변경된 상태를 저장하며, 배송된 주문이라면 `cancel()`에서 예외가 발생해 저장 전에 처리가 끝난다.

여기서 두 역할이 나뉜다. **`CancelOrderService`가 취소 작업의 순서를 조정하고, `Order`가 취소 가능 여부를 판단한다.**

작업이 끝나면 주문 ID와 처리 후 상태를 `CancelOrderResult`에 담아 돌려준다. 호출한 쪽에서는 주문 객체를 직접 다루지 않고도 이 결과로 HTTP 응답이나 콘솔 출력을 만들 수 있다.

## HTTP와 JPA 코드는 Adapter로 연결한다

Service가 사용할 계약과 취소 절차를 만들었다. 이제 `OrderGateway`의 구현을 연결해 실제 DB에서 주문을 읽고 저장할 수 있게 하자.

### 저장 계약을 JPA로 구현한다

`JpaOrderAdapter`는 `OrderGateway`를 구현하면서, 조회와 저장 요청을 Spring Data JPA Repository로 전달한다.

```java
@RequiredArgsConstructor
public class JpaOrderAdapter implements OrderGateway {

    private final JpaOrderRepository repository;

    @Override
    public java.util.Optional<OrderData> findById(long orderId) {
        return repository.findById(orderId)
            .map(entity -> new OrderData(
                entity.getId(), entity.getStatus()
            ));
    }

    @Override
    public void save(OrderData data) {
        repository.save(OrderJpaEntity.from(data));
    }
}
```

조회할 때는 Repository가 반환한 JPA Entity에서 ID와 상태를 꺼내 `OrderData`로 바꾸고, 저장할 때는 반대로 `OrderJpaEntity`에 옮긴다. 예제에서는 이 변환이 보이도록 저장 모델을 ID와 상태만 가진 형태로 단순화했다.

JPA 매핑이나 Repository 사용법이 바뀌면 이 Adapter를 수정한다. `OrderGateway`의 계약을 유지할 수 있는 변경이라면, `CancelOrderService`의 조회·저장 코드는 그대로 사용할 수 있다.

### HTTP 요청은 Controller에서 받는다

저장소 연결이 끝났으니, 처음 Service에 있던 HTTP 처리도 옮겨 보자. Controller에서 URL의 주문 ID를 받아 취소 작업을 호출하고, 성공 응답까지 반환하도록 한다.

```java
@RestController
@RequiredArgsConstructor
public class OrderController {

    private final CancelOrderService cancelOrderService;

    @PostMapping("/orders/{orderId}/cancel")
    public ResponseEntity<Void> cancel(
            @PathVariable("orderId") long orderId) {
        cancelOrderService.cancel(orderId);
        return ResponseEntity.noContent().build();
    }
}
```

처음 예제와 같은 204 응답이지만, 이를 만드는 곳이 Service에서 Controller로 바뀌었다. 취소 결과를 JSON으로 내려주려면 Service가 반환한 `CancelOrderResult`로 응답 본문을 구성할 수도 있다.

Controller가 HTTP 요청을 Service 호출로 바꿔 주듯, JPA Adapter는 업무 코드의 조회·저장 요청을 JPA로 처리한다. 이렇게 업무 코드와 외부 기술 사이에서 호출과 데이터 형식을 맞춰 주는 부분이 **Interface Adapters**다.

그 바깥의 **Frameworks & Drivers**에는 Adapter가 사용하는 Spring MVC, Hibernate와 DB가 놓인다. 예제에서는 Spring의 Bean 설정으로 `JpaOrderAdapter`를 만들고 `CancelOrderService`에 주입해 실제 저장소를 연결할 수 있다.

## 호출은 DB로 향해도 의존성은 안쪽으로 향한다

이렇게 코드를 나누어도 요청은 Controller에서 시작해 Service로 전달되고, 저장할 때는 JPA Adapter를 거쳐 DB에 도달한다. 실행은 여전히 바깥쪽 기술까지 이어지는데, 의존성이 안쪽을 향한다는 말은 무엇일까? 각 클래스가 소스코드에서 어떤 타입을 참조하는지 보면 차이가 드러난다.

| 코드 | 소스코드에서 참조하는 대상 |
|---|---|
| `OrderController` | `CancelOrderService`를 호출한다 |
| `CancelOrderService` | `OrderGateway`를 호출한다 |
| `JpaOrderAdapter` | `OrderGateway`를 구현한다 |

저장 쪽을 보면 Service와 JPA Adapter가 모두 Use Case 영역의 `OrderGateway`를 참조한다. Service는 계약에 정의된 메서드를 호출하고, Adapter는 그 메서드를 구현한다. **업무 코드가 저장 구현을 따라가던 관계에서, 저장 구현이 업무 쪽 계약을 따르는 관계로 바뀐 것**이다.

실제로 동작할 때는 생성자로 주입된 JPA Adapter가 `orders.save()` 호출을 처리한다. 인터페이스와 의존성 주입 덕분에 Service에서 구현 클래스의 이름을 참조하지 않고도 저장을 요청할 수 있다.

이런 관계를 의존성 역전이라고 한다. 프로젝트에 맞춰 원문의 네 영역을 더 나누더라도 소스코드의 의존성은 같은 기준으로 정한다.

## 업무 규칙을 외부 기술 없이 테스트한다

이제 처음에 어려웠던 테스트로 돌아가 보자. 취소 조건을 확인하려고 JPA Repository의 반환 객체를 준비할 필요가 없어졌다. `Order`를 만들어 취소를 요청하면 배송된 주문이 거절되는지 바로 확인할 수 있다.

```java
@Test
void 배송된_주문은_취소할_수_없다() {
    Order order = new Order(1L, OrderStatus.SHIPPED);

    assertThrows(IllegalStateException.class, order::cancel);
    assertEquals(OrderStatus.SHIPPED, order.status());
}
```

Use Case를 테스트할 때는 `OrderGateway`의 가짜 구현을 연결하면 된다. 정해진 주문을 반환하고 저장 요청을 기록하도록 만들면, 취소 후 `CANCELED` 상태가 저장되는지와 취소가 거절됐을 때 저장을 건너뛰는지를 검사할 수 있다.

두 테스트 모두 웹 서버와 DB 없이 실행할 수 있다. HTTP 경로나 JPA 조회 구현을 바꾸더라도 주문의 규칙과 처리 절차가 같으면 테스트도 그대로 유지한다. 실제 쿼리와 트랜잭션의 동작은 Adapter를 포함한 통합 테스트로 따로 확인한다.

물론 분리한 인터페이스와 전달용 객체, 변환 코드도 관리해야 한다. 규칙이 거의 없는 단순 CRUD에서는 그 부담이 더 클 수 있으므로, 업무 규칙을 자주 검증해야 하거나 외부 구현 변경이 잦은 기능부터 적용할 범위를 살펴보는 편이 현실적이다.

## 더 살펴보기: 결과를 표현하는 Presenter

앞에서는 Service가 처리 결과를 반환하고 Controller가 HTTP 응답을 만들도록 했다. 마틴의 [2011년 글](https://blog.cleancoder.com/uncle-bob/2011/11/22/Clean-Architecture.html)에서도 이런 흐름을 설명하는데, 2012년 원문 그림에는 결과를 전달하는 또 다른 구성이 나온다.

Use Case가 **Presenter**에 결과를 전달하면, Presenter가 화면이나 출력에 맞는 형태로 바꿔 주는 방식이다. 콘솔 도구라면 취소된 주문의 ID를 받아 `주문 1 취소 완료`라는 문구를 출력할 수 있다.

Use Case에서 콘솔 구현을 직접 참조하지 않도록, 결과를 전달할 계약을 안쪽의 Output Port로 정의한다. 이를 구현하는 Presenter는 바깥쪽 Adapter가 된다.

```java
public interface CancelOrderOutputPort {
    void present(CancelOrderResult result);
}

public class ConsoleCancelOrderPresenter
        implements CancelOrderOutputPort {

    @Override
    public void present(CancelOrderResult result) {
        System.out.printf("주문 %d 취소 완료%n", result.orderId());
    }
}
```

이를 연결하려면 Use Case에 Output Port를 주입하고 `cancel()`의 반환형을 `void`로 바꾼다. 작업 마지막에는 결과를 반환하는 대신 `outputPort.present(result)`를 호출해 Presenter로 넘기면 된다.

결과를 전달하는 방법이 달라져도 취소 절차는 Use Case에, 출력 형식은 바깥에 남는다. 저장소를 연결할 때와 마찬가지로 Presenter도 안쪽에서 정한 계약에 맞춰 연결한 셈이다.

## 헥사고날과 함께 이해하기

앞에서 사용한 `OrderGateway`와 `JpaOrderAdapter`는 [헥사고날 아키텍처](/posts/16/)의 출력 Port와 Adapter에 해당한다. 나머지 구성요소도 함께 놓고 보면 두 아키텍처가 같은 코드에서 어떻게 드러나는지 알 수 있다.

| 예제의 구성요소 | 헥사고날에서 보면 | 클린에서 보면 |
|---|---|---|
| `Order` | 애플리케이션 내부 | Entity |
| `CancelOrderService` | 애플리케이션 내부 | Use Case |
| `OrderGateway` | 출력 Port | 안쪽에서 정의한 저장 계약 |
| `OrderController` | 입력 Adapter | Interface Adapters |
| `JpaOrderAdapter` | 출력 Adapter | Interface Adapters |
| 웹 프레임워크·DB | 외부 기술 | Frameworks & Drivers |

헥사고날 관점에서는 애플리케이션과 외부를 잇는 Port와 Adapter가 보인다. 클린 아키텍처 관점에서는 여기에 업무 규칙과 작업 절차를 맡는 Entity와 Use Case의 구분까지 드러난다. 마틴이 헥사고날을 비롯한 여러 접근을 종합했기 때문에 공통점이 많으며, 이 예제처럼 두 접근의 원칙을 함께 적용할 수 있다.

## 정리

하나의 Service에 모여 있던 코드를 나누어 취소 조건은 Entity에, 처리 절차는 Use Case에 담고 HTTP와 JPA 처리는 Adapter로 옮겼다. 외부 기능이 필요한 곳에는 안쪽에서 계약을 정하고, 바깥의 구현을 연결했다.

이렇게 나누는 목적은 **외부 기술의 사용법이 바뀌어도 업무 규칙을 독립적으로 유지하고 검증하는 것**이다. 실제 코드에 적용할 때도 업무 규칙 하나를 테스트하려면 무엇을 준비해야 하는지부터 살펴보면 좋다. HTTP와 DB의 세부 설정까지 필요하다면, 그중 어떤 부분을 계약과 Adapter로 분리할 수 있을지 검토해 볼 수 있다.

## 참고 자료

- [Robert C. Martin — The Clean Architecture (2012)](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Robert C. Martin — Clean Architecture (2011)](https://blog.cleancoder.com/uncle-bob/2011/11/22/Clean-Architecture.html)
- [Alistair Cockburn — Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/)
- [Spring Data JPA — Transactionality](https://docs.spring.io/spring-data/jpa/reference/jpa/transactions.html)
- [Spring Data JPA — Persisting Entities](https://docs.spring.io/spring-data/jpa/reference/jpa/entity-persistence.html)
