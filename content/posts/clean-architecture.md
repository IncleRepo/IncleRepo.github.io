+++
title = '업무 규칙을 안쪽에 두는 클린 아키텍처'
slug = '17'
date = 2026-09-14T16:20:00+09:00
lastmod = 2026-09-14T16:40:00+09:00
draft = true
references_required = true
description = '익숙한 주문 취소 Service에서 출발해 업무 규칙과 외부 구현을 분리하며, 클린 아키텍처의 구성요소와 의존성 규칙을 살펴봅니다.'
categories = ['애플리케이션 아키텍처']
tags = ['클린 아키텍처', '의존성 역전', 'Use Case', '소프트웨어 설계']
+++

Service에서 주문을 조회하고, 취소 조건을 확인한 뒤 저장하는 코드는 익숙하다. 여기에 HTTP 응답 처리와 JPA 사용법까지 들어가면, 취소 조건을 바꿀 때도 저장 방식을 바꿀 때도 같은 Service를 수정해야 한다.

**클린 아키텍처**(Clean Architecture)는 업무 규칙을 중심에 두고, HTTP나 DB를 다루는 코드를 바깥으로 분리한다. 로버트 C. 마틴의 [The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)를 바탕으로, 익숙한 주문 취소 코드를 나누어 가며 이 구조를 살펴보자.

## 주문 취소 Service에 무엇이 들어 있을까

결제 완료 상태의 주문만 취소할 수 있는 서비스를 가정하자. 아래는 JPA로 주문을 찾아 상태를 바꾸고, HTTP 응답까지 만드는 예제다.

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

트랜잭션 안에서 조회한 JPA Entity의 상태 변경은 Dirty Checking으로 DB에 반영된다. 이 짧은 Service 안에서 다음 세 가지 일을 처리하고 있다.

- 결제 완료 주문만 취소할 수 있다는 **업무 규칙**.
- 주문을 찾아 취소하는 **작업의 순서**.
- JPA로 조회하고 HTTP 응답을 만드는 **기술별 처리**.

이렇게 묶여 있으면 저장 구현만 바꿔도 취소 규칙이 있는 파일을 수정하게 된다. 콘솔 기반 관리자 도구에서 재사용하려 해도 반환값으로 HTTP 응답 객체를 받아야 한다.

먼저 주문 자체의 규칙을 분리하고, 그다음 저장과 HTTP 처리를 옮겨 보자. 이후 예제에서는 역할과 의존성을 살펴보기 위해 트랜잭션 설정·동시성 제어·HTTP 예외 처리를 생략한다.

## 주문의 취소 규칙을 Entity에 담는다

취소 조건을 확인하고 상태를 바꾸는 코드를 `Order.cancel()`로 옮긴다. 이제 주문 객체가 자신의 취소 가능 여부를 판단한다.

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

고객 화면과 관리자 도구에서 모두 `Order.cancel()`을 호출하면 같은 취소 조건을 확인한다. `Order`는 자신의 상태만으로 판단하므로 요청이 들어온 화면이나 저장할 DB를 알 필요가 없다.

클린 아키텍처에서 이런 핵심 업무 규칙을 담는 부분이 **Entity**다. 마틴은 여러 애플리케이션에서도 사용할 수 있는 일반적인 업무 규칙을 이 영역에 둔다. 예제의 `Order`도 JPA 매핑 없이 주문의 상태와 규칙을 담고 있다.

## Use Case에는 작업의 흐름을 남긴다

취소 조건을 `Order.cancel()`로 옮기면 Service에는 주문을 찾아 취소하고 저장하는 흐름이 남는다. 여기서 JPA 사용법도 분리하려면, Service가 저장소에 요청할 작업을 먼저 정해야 한다.

`OrderGateway`는 그 기능을 정의한 인터페이스다. `OrderData`에는 주문을 복원하고 저장하는 데 필요한 ID와 상태를 담는다.

```java
public record OrderData(long id, OrderStatus status) {}

public interface OrderGateway {
    java.util.Optional<OrderData> findById(long orderId);
    void save(OrderData data);
}
```

Service가 `JpaOrderRepository` 대신 이 계약을 사용하도록 바꾸고, 반환값도 HTTP 응답에서 단순한 처리 결과로 바꾼다.

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

주문 취소처럼 사용자가 애플리케이션을 통해 수행하는 작업이 **Use Case**다. `CancelOrderService`는 주문을 읽어 `Order`로 복원하고, 취소한 뒤 저장하는 순서로 이 작업을 구현한다.

고객의 취소 요청이라면 이 흐름에 소유자 확인을 추가할 수 있고, 관리자용 작업이라면 별도의 권한과 절차를 적용할 수 있다. 작업별 절차는 달라도 주문 자체의 취소 조건은 같은 `Order.cancel()`로 확인한다.

## HTTP와 JPA 코드는 Adapter로 연결한다

앞의 Service는 `OrderGateway`에 저장을 요청한다. 실제 DB와 연결하려면 이 인터페이스를 구현한 객체가 필요하다.

### 저장 계약을 JPA로 구현한다

`JpaOrderAdapter`가 `OrderGateway`를 구현하고, 실제 조회와 저장은 Spring Data JPA Repository에 맡긴다.

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

조회 결과는 `OrderData`에 담아 반환하고, 저장할 때는 다시 `OrderJpaEntity`로 바꾼다. 예제의 `OrderJpaEntity`에는 ID와 상태만 있으며, `from(data)`가 이 두 값을 옮긴다.

JPA 매핑이나 Repository 사용법이 바뀌면 이 Adapter에서 맞춘다. `CancelOrderService`는 계속 `OrderData`를 주고받으며 `OrderGateway`에 조회와 저장을 요청한다.

### HTTP 요청은 Controller에서 받는다

저장소를 연결했으니 처음 Service에 있던 HTTP 처리도 Controller로 옮기자. Controller는 URL에서 주문 ID를 받아 취소 작업을 호출하고, 성공하면 HTTP 응답을 반환한다.

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

처음과 같은 204 응답을 이제 Controller에서 만든다. 취소 결과를 JSON으로 내려주고 싶다면 `CancelOrderResult`를 받아 응답 본문을 구성하면 된다.

Controller와 JPA Adapter처럼 외부의 요청·데이터를 내부 호출에 맞춰 연결하는 코드가 **Interface Adapters**다. 이들이 사용하는 Spring MVC, Hibernate, DB와 연결 설정은 바깥쪽의 **Frameworks & Drivers**에 놓인다.

Spring에서는 Bean 설정에서 `JpaOrderAdapter`를 만들어 `CancelOrderService`에 주입할 수 있다. 어떤 구현을 연결할지는 설정에서 정하고, Service는 생성자로 받은 `OrderGateway`를 사용한다.

## 완성된 구조에서 의존 방향을 살펴보자

한 Service에 있던 코드를 주문 규칙, 작업 흐름, HTTP·저장 구현으로 나눴다. 이를 원문의 그림에 대입하면 각 영역의 역할이 보인다.

![Entities를 중심으로 Use Cases, Interface Adapters, Frameworks and Drivers가 둘러싼 클린 아키텍처 원문 그림](https://blog.cleancoder.com/uncle-bob/images/2012-08-13-the-clean-architecture/CleanArchitecture.jpg "Robert C. Martin의 클린 아키텍처 구성도")

출처: [Robert C. Martin — The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

그림 가운데의 Entities에는 `Order`, 그 바깥의 Use Cases에는 `CancelOrderService`가 있다. Controller와 JPA Adapter가 그 둘을 외부 기술에 연결한다.

이 영역들을 연결하는 기준은 **소스코드의 의존성이 안쪽으로 향한다**는 것이다. 마틴은 이를 **의존성 규칙**(Dependency Rule)이라고 부른다. 바깥 코드는 안쪽의 타입을 사용하고, 안쪽 코드는 바깥에 정의된 타입을 참조하지 않는다.

예제에서도 `Order`는 Service를 모르고, Service는 Controller와 JPA Adapter를 모른다. 그런데 실제 저장은 JPA Adapter가 해야 한다. Service가 구현 클래스를 모르면서도 호출할 수 있는 이유는 둘 사이의 `OrderGateway`에 있다.

| 구분 | 참조하는 대상 |
|---|---|
| `CancelOrderService` | `OrderGateway`를 호출한다 |
| `JpaOrderAdapter` | `OrderGateway`를 구현한다 |

`OrderGateway`를 Use Case 쪽에 두면 두 클래스가 모두 안쪽의 계약을 참조한다. 실행할 때는 생성자로 주입된 JPA Adapter의 메서드가 동작하므로, **호출은 바깥으로 나가도 소스코드 의존성은 안쪽을 향한다.**

원문의 네 영역은 프로젝트에 맞게 더 나눌 수 있다. 경계가 늘어나더라도 의존 방향은 같은 기준으로 정한다.

## 경계를 넘을 때는 필요한 데이터만 전달한다

계약을 안쪽에 두었다면 그 인자와 반환형도 확인해야 한다. `OrderGateway`가 JPA 전용 타입을 반환하면 Service 역시 그 타입에 의존하기 때문이다. 예제에서는 이를 분리하려고 `OrderData`를 두었다.

`OrderGateway`와 `OrderData`는 Use Case 쪽에, `OrderStatus`는 도메인 쪽에 둔다. JPA Adapter가 조회 결과를 이 형식으로 바꾸면 Service는 DB 매핑과 관계없이 주문을 복원할 수 있다.

반환값인 `CancelOrderResult`에도 ID와 상태 문자열만 담았다. 주문을 변경하는 기능은 `Order`에 남기고, 외부에는 처리 결과를 전달한다. 마틴의 [2011년 글](https://blog.cleancoder.com/uncle-bob/2011/11/22/Clean-Architecture.html)에서도 요청을 단순한 데이터로 풀어 작업에 전달하고, 그 결과로 UI를 구성하는 흐름을 설명한다.

### 결과의 표시 형식은 Presenter가 맡을 수 있다

이 결과를 화면에 보여줄 때는 `CANCELED`를 `주문 취소 완료`라는 문구로 바꿀 수 있다. 이런 표시 형식을 만드는 Adapter가 **Presenter**다.

원문은 Use Case가 Presenter에 결과를 전달하는 구성도 보여준다. 이때도 안쪽에 계약을 두고 바깥에서 구현한다.

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

이 Presenter는 주문 ID를 받아 취소 완료 문구를 콘솔에 출력한다. 연결하려면 Use Case에 Output Port를 주입하고, `cancel()`의 반환형을 `void`로 바꾼 뒤 결과를 `outputPort.present(result)`에 전달한다.

Use Case는 출력할 문구나 콘솔 사용법을 몰라도 된다. 앞에서는 Controller가 작업을 호출한 뒤 HTTP 응답을 만들었다면, 이 구성에서는 Presenter가 결과를 전달받아 표현을 맡는다.

## 업무 규칙을 외부 기술 없이 테스트한다

업무 코드가 HTTP와 JPA에서 분리되면 테스트도 따로 실행할 수 있다. 배송된 주문의 취소를 거절하는지 확인하려면 `Order`를 만들고 `cancel()`을 호출하면 된다.

```java
@Test
void 배송된_주문은_취소할_수_없다() {
    Order order = new Order(1L, OrderStatus.SHIPPED);

    assertThrows(IllegalStateException.class, order::cancel);
    assertEquals(OrderStatus.SHIPPED, order.status());
}
```

Use Case에는 `OrderGateway`의 가짜 구현을 연결한다. 정해진 주문을 반환하고 저장 요청을 기록하게 만들면, 취소 후 `CANCELED` 상태를 저장하는지 확인할 수 있다. 취소가 거절됐을 때 저장을 호출하지 않는지도 같은 방식으로 검사한다.

이 테스트에는 웹 서버와 DB가 필요하지 않다. HTTP 경로나 JPA 조회 구현을 수정해도 주문의 규칙과 작업 순서가 그대로라면 테스트를 유지할 수 있다. 실제 쿼리와 트랜잭션은 Adapter를 포함한 통합 테스트에서 확인한다.

이런 분리를 위해 예제에는 `OrderData`와 변환 코드가 추가됐다. 업무가 단순하면 이 코드를 관리하는 부담이 더 클 수 있다. 외부 구현을 바꾸는 일이 얼마나 잦은지, 업무 테스트에 기술별 설정이 얼마나 필요한지를 보고 분리할 범위를 정한다.

## 헥사고날과 함께 이해하기

앞의 `OrderGateway`와 `JpaOrderAdapter`는 [헥사고날 아키텍처](/posts/16/)의 출력 Port와 Adapter로도 볼 수 있다. 같은 구성을 두 관점에서 정리하면 다음과 같다.

| 예제의 구성요소 | 헥사고날에서 보면 | 클린에서 보면 |
|---|---|---|
| `Order` | 애플리케이션 내부 | Entity |
| `CancelOrderService` | 애플리케이션 내부 | Use Case |
| `OrderGateway` | 출력 Port | 안쪽에서 정의한 저장 계약 |
| `OrderController` | 입력 Adapter | Interface Adapters |
| `JpaOrderAdapter` | 출력 Adapter | Interface Adapters |
| 웹 프레임워크·DB | 외부 기술 | Frameworks & Drivers |

헥사고날은 애플리케이션과 외부의 연결을, 클린은 업무 규칙을 포함한 각 영역의 책임과 의존 방향을 드러낸다. 마틴이 헥사고날을 비롯한 여러 접근을 종합한 만큼 공통점이 많고, 위 예제처럼 두 접근의 원칙을 함께 적용할 수 있다.

## 정리

처음 Service에 모여 있던 취소 조건은 Entity로, 작업의 순서는 Use Case로, HTTP와 JPA 처리는 Adapter로 나눴다. 이들을 연결할 때는 안쪽이 필요한 계약을 정하고 바깥쪽 구현이 그 계약을 따르게 했다.

실제 코드에서도 **업무 규칙이 어디에 모여 있는지, 그 코드가 외부의 어떤 타입에 의존하는지**를 살펴보면 된다. 외부 구현을 수정할 때 업무 코드가 함께 바뀌는지, 웹 서버나 DB 없이도 업무 규칙을 테스트할 수 있는지가 경계를 점검할 기준이 된다.

## 참고 자료

- [Robert C. Martin — The Clean Architecture (2012)](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Robert C. Martin — Clean Architecture (2011)](https://blog.cleancoder.com/uncle-bob/2011/11/22/Clean-Architecture.html)
- [Alistair Cockburn — Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/)
- [Spring Data JPA — Transactionality](https://docs.spring.io/spring-data/jpa/reference/jpa/transactions.html)
- [Spring Data JPA — Persisting Entities](https://docs.spring.io/spring-data/jpa/reference/jpa/entity-persistence.html)
