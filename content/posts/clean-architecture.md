+++
title = '업무 규칙을 안쪽에 두는 클린 아키텍처'
slug = '17'
date = 2026-09-14T16:20:00+09:00
lastmod = 2026-09-14T16:28:00+09:00
draft = true
references_required = true
description = '주문 취소 예제로 Entity, Use Case, Adapter의 역할을 나누고, 클린 아키텍처의 의존성 규칙이 코드에서 어떻게 드러나는지 살펴봅니다.'
categories = ['애플리케이션 아키텍처']
tags = ['클린 아키텍처', '의존성 역전', 'Use Case', '소프트웨어 설계']
+++

주문 취소 API의 주소를 바꾸거나 저장 방식을 수정할 때, 주문을 취소할 수 있는 조건까지 함께 손댈 필요는 없다. 업무 규칙과 외부 기술이 서로의 세부사항을 적게 알수록 이런 변경을 나누어 처리하기 쉽다.

**클린 아키텍처는 업무 규칙을 중심에 두고, 바깥의 구현이 안쪽의 규칙과 계약에 의존하도록 구성한다.** 로버트 C. 마틴의 [The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)를 바탕으로, 주문 취소 기능 하나가 어떤 책임으로 나뉘고 서로 어떻게 연결되는지 살펴보자.

## 주문 취소 안에도 서로 다른 책임이 있다

주문을 취소하는 Service를 생각해 보자. 주문을 조회하고, 취소할 수 있는 상태인지 확인한 뒤, 상태를 변경해 저장한다. Controller는 요청을 받아 이 작업을 호출하고 처리 결과를 응답한다.

이 과정에 참여하는 코드는 서로 다른 이유로 바뀐다.

| 변경할 내용 | 관련된 책임 |
|---|---|
| 취소 API의 URL이나 JSON 필드를 바꾼다 | 외부 요청과 응답 처리 |
| 고객의 취소 요청에 본인 확인 절차를 추가한다 | 특정 작업의 처리 절차 |
| 취소를 허용하는 주문 상태를 바꾼다 | 주문 자체의 업무 규칙 |
| 주문을 읽고 저장하는 SQL을 바꾼다 | 데이터 접근 구현 |

클린 아키텍처에서는 이 책임들을 업무 규칙에 가까운 것부터 배치한다. 주문 자체의 규칙은 중심에, 그 규칙을 사용하는 작업은 바로 바깥에 둔다. HTTP나 DB를 다루는 코드는 이들을 둘러싼다.

![Entities를 중심으로 Use Cases, Interface Adapters, Frameworks and Drivers가 둘러싼 클린 아키텍처 원문 그림](https://blog.cleancoder.com/uncle-bob/images/2012-08-13-the-clean-architecture/CleanArchitecture.jpg "Robert C. Martin의 클린 아키텍처 구성도")

출처: [Robert C. Martin — The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

- **Entities:** 주문의 취소 조건처럼 핵심 업무 규칙을 담는다.
- **Use Cases:** 주문을 찾아 취소하고 저장하는 작업을 진행한다.
- **Interface Adapters:** HTTP 요청이나 저장 데이터를 내부에서 사용할 형태로 연결한다.
- **Frameworks & Drivers:** DB, 웹 프레임워크와 이를 연결하는 설정이 놓인다.

가운데에 놓인 주문의 규칙부터 코드로 옮겨 보자. 아래 Java 예제는 이 글의 주문 취소 상황에 맞춰 작성했다.

## Entity가 판단하고 Use Case가 작업을 진행한다

### 주문의 취소 조건은 Entity에 둔다

예제에서는 결제 완료 상태의 주문만 취소할 수 있다고 가정한다. `Order`는 자신의 상태를 확인하고, 조건을 만족하면 취소 상태로 바꾼다.

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

고객 화면과 관리자 기능에서 모두 `Order.cancel()`을 호출하면 같은 취소 조건을 확인한다. 화면의 처리 절차가 달라져도 함께 지켜야 하는 규칙을 `Order`에 모은 것이다.

이처럼 핵심 업무 규칙을 담는 부분이 **Entity**다. 마틴은 여러 애플리케이션에서도 사용할 수 있는 일반적인 업무 규칙을 이 영역에 둔다. 이 글의 `Order` 역시 저장 매핑보다 업무 규칙을 표현하는 객체로 보면 된다.

### 주문을 찾아 취소하는 흐름은 Use Case에 둔다

고객이 주문 ID를 보내면 그에 해당하는 주문을 찾아 `cancel()`을 호출하고, 바뀐 상태를 저장해야 한다. 이 흐름을 맡을 Service에는 주문을 읽고 저장하는 기능이 필요하다.

그 기능을 `OrderGateway` 인터페이스로 정의하자. `OrderData`에는 저장소와 주고받을 주문 ID와 상태를 담는다.

```java
public record OrderData(long id, OrderStatus status) {}

public interface OrderGateway {
    java.util.Optional<OrderData> findById(long orderId);
    void save(OrderData data);
}
```

`CancelOrderService`는 이 계약으로 주문 데이터를 읽고, `Order`를 만들어 취소한 뒤 결과를 저장한다.

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

주문 취소라는 작업이 **Use Case**이고, `CancelOrderService`는 이를 구현한다. 이 작업에 본인 주문인지 확인하는 절차가 필요하다면 Service의 흐름에 추가한다. 취소할 수 있는 주문 상태는 계속 `Order.cancel()`이 판단한다.

예제는 역할 구분에 집중해 조회·취소·저장만 담았다. 실제 DB에 적용할 때는 이 작업의 트랜잭션 범위와 다른 요청에 의한 동시 변경도 함께 처리해야 한다.

## Adapter가 HTTP와 저장 기술을 연결한다

여기까지 작성한 코드는 주문 ID로 취소 작업을 수행한다. 웹에서 이 기능을 사용하려면 HTTP 요청에서 ID를 꺼내 전달하는 코드가 필요하다. Controller가 그 연결을 맡는다.

```java
@RestController
public class OrderController {

    private final CancelOrderService cancelOrderService;

    public OrderController(CancelOrderService cancelOrderService) {
        this.cancelOrderService = cancelOrderService;
    }

    @PostMapping("/orders/{orderId}/cancel")
    public ResponseEntity<CancelOrderResult> cancel(
            @PathVariable("orderId") long orderId) {
        return ResponseEntity.ok(cancelOrderService.cancel(orderId));
    }
}
```

URL과 HTTP 상태 코드는 Controller에서 정한다. 예제에서는 `CancelOrderResult`를 그대로 응답 본문에 담았고, API에 별도의 필드나 표시 형식이 필요하면 이곳에서 변환할 수 있다.

저장소도 같은 방식으로 연결한다. `JpaOrderAdapter`가 `OrderGateway`를 구현해 JPA로 읽은 주문을 `OrderData`로 바꾸고, 저장 요청을 받으면 그 값을 JPA의 저장 형식으로 옮긴다.

이 두 연결 코드가 **Interface Adapters**에 해당한다. HTTP나 JPA에 맞추는 처리가 이곳에 모이므로 `CancelOrderService`는 주문 취소의 순서에 집중할 수 있다.

Adapter가 사용하는 Spring MVC, Hibernate와 DB는 바깥쪽의 **Frameworks & Drivers**에 해당한다. 어떤 Adapter를 Service에 주입할지 정하는 Bean 설정도 이쪽에 둔다. 업무 코드는 필요한 기능을 계약으로 정의하고, 바깥의 설정이 그 계약에 맞는 구현을 연결하는 구성이다.

## 의존성은 안쪽으로 향한다

이제 요청부터 저장까지 연결했다. 여기서 `CancelOrderService`가 `JpaOrderAdapter`를 직접 생성한다면, JPA 구현을 바꿀 때 Service도 수정해야 한다. 이를 피하려고 Service에는 `OrderGateway`만 전달했다.

이 관계를 모든 경계에 적용한 것이 **의존성 규칙**(Dependency Rule)이다. 바깥 코드는 안쪽의 타입을 사용할 수 있고, 안쪽 코드는 바깥에서 정의한 타입을 참조하지 않는다. `Order`가 `CancelOrderService`를 참조하지 않고, Service가 Controller나 JPA Adapter를 참조하지 않는 이유다.

주문을 저장할 때의 호출 방향과 코드 의존 방향을 따로 적으면 차이가 보인다.

| 구분 | 관계 |
|---|---|
| 실행할 때 | `CancelOrderService` → `JpaOrderAdapter` |
| Service의 코드 | `CancelOrderService` → `OrderGateway` |
| Adapter의 코드 | `JpaOrderAdapter` → `OrderGateway` 구현 |

`OrderGateway`는 Use Case 쪽에 둔다. Service는 이를 호출하고 JPA Adapter는 구현하므로, 두 클래스의 소스코드가 모두 안쪽을 향한다. 실행할 때는 주입된 Adapter가 동작해 실제 저장을 수행한다. **호출은 바깥으로 나가면서도 코드의 의존성은 안쪽을 향하는 것**이다.

### 결과를 전달할 때도 같은 원리를 쓴다

저장뿐 아니라 처리 결과를 화면에 전달할 때도 이 방식이 쓰인다. 원문 그림의 오른쪽 아래에 등장하는 **Presenter**는 결과를 화면에 표시하기 좋은 형태로 바꾸는 Adapter다. Use Case가 Presenter를 호출하도록 구성한다면, 안쪽에 결과 전달용 인터페이스를 둔다.

주문 취소에 적용한다면 다음과 같은 계약을 만들 수 있다.

```java
public interface CancelOrderOutputPort {
    void present(CancelOrderResult result);
}
```

Use Case가 `outputPort.present(result)`를 호출하면 이 인터페이스를 구현한 Presenter가 결과를 받아 표시 형식을 만든다. Use Case는 화면의 구성이나 표시 문구를 직접 다루지 않는다.

앞의 예제처럼 결과를 반환하고 Controller에서 응답을 만들어도 표현 처리는 바깥에 남는다. 원문의 Presenter 예시는 결과를 반환하는 대신 외부 객체에 전달할 때도 같은 의존 방향을 유지하는 방법을 보여준다.

## 경계를 넘는 데이터도 의존성을 만든다

이 계약에는 작업의 이름뿐 아니라 주고받는 데이터 형식도 포함된다. 예를 들어 `OrderGateway`가 JPA 전용 타입을 반환하면, Service는 그 값을 읽기 위해 JPA를 알아야 한다. 그래서 예제에는 주문 ID와 상태만 담은 `OrderData`를 두었다.

`OrderData`와 `OrderGateway`는 Use Case 쪽에, `OrderStatus`는 도메인 쪽에 둔다. JPA Adapter가 조회 결과를 이 형식에 맞춰 전달하므로 Service는 DB 매핑을 다루지 않고도 `Order`를 복원할 수 있다.

응답으로 나가는 `CancelOrderResult`도 같은 기준으로 정했다. Controller에는 취소 결과를 표시할 주문 ID와 상태 문자열만 전달한다. 주문을 변경하는 기능은 `Order`에 남는다.

상태를 `CANCELED` 대신 `주문 취소 완료`로 표시하려면 Controller나 Presenter에서 변환하면 된다. 취소 규칙이나 저장 코드를 고칠 필요는 없다. 마틴의 2011년 글 [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2011/11/22/Clean-Architecture.html)에서도 이런 요청·결과 데이터를 통해 UI와 업무 코드를 분리한다.

## 분리한 효과는 변경과 테스트로 확인한다

요청과 저장 형식을 바깥에서 처리하도록 나누면 업무 규칙을 따로 실행해 볼 수 있다. `Order` 객체를 만들고 `cancel()`을 호출하는 것만으로 결제 완료 주문은 취소되는지, 배송된 주문은 거절되는지 확인할 수 있다.

Use Case에는 DB 대신 `OrderGateway`의 테스트용 구현을 연결한다. 정해진 `OrderData`를 반환하고 저장 요청을 기록하게 만들면 다음을 검사할 수 있다.

- 결제 완료 주문을 취소하면 `CANCELED` 상태로 저장하는가?
- 배송된 주문의 취소가 거절되면 저장을 호출하지 않는가?
- 존재하지 않는 주문이라면 예외로 알리는가?

이 테스트는 웹 서버나 DB 없이 실행한다. HTTP 경로나 JPA 조회 코드를 바꾸어도 취소 규칙과 작업 절차가 같다면 테스트를 그대로 유지할 수 있다. 실제 SQL과 트랜잭션 동작은 별도의 통합 테스트에서 확인한다.

이런 분리에는 `OrderData` 같은 전달 타입과 변환 코드를 관리하는 비용이 따른다. 작은 조회 기능에서는 추가된 코드를 따라가는 일이 더 번거로울 수 있다. 반대로 저장 구현을 고칠 때마다 업무 테스트도 수정하고 있다면, 경계를 나눠 얻을 이점이 구체적으로 드러난 셈이다.

## 헥사고날과 함께 이해하기

앞에서 사용한 인터페이스와 Adapter는 [헥사고날 아키텍처](/posts/16/)에서도 만났던 구성이다. 같은 코드를 두 관점에 대응시키면 다음과 같다.

| 예제의 구성요소 | 헥사고날에서 보면 | 클린에서 보면 |
|---|---|---|
| `Order` | 애플리케이션 내부 | Entity |
| `CancelOrderService` | 애플리케이션 내부 | Use Case |
| `OrderGateway` | 출력 Port | 안쪽에서 정의한 저장 계약 |
| `OrderController` | 입력 Adapter | Interface Adapters |
| `JpaOrderAdapter` | 출력 Adapter | Interface Adapters |
| 웹 프레임워크·DB | 외부 기술 | Frameworks & Drivers |

헥사고날은 애플리케이션과 외부의 연결을 Port와 Adapter로 설명한다. 클린 아키텍처는 Entity와 Use Case의 책임도 구분하고, 각 경계의 의존 방향을 정한다. 마틴이 헥사고날을 비롯한 여러 접근을 종합한 만큼 원칙이 겹치며, 위 예제처럼 한 코드에 함께 적용할 수 있다.

## 정리

- Entity는 핵심 업무 규칙을, Use Case는 그 규칙을 사용해 작업을 진행하는 흐름을 맡는다.
- Adapter는 내부의 계약과 외부 기술을 연결한다. 바깥으로 호출해야 할 때도 계약을 안쪽에 두면 소스코드 의존성은 안쪽을 향한다.
- 경계를 나눈 효과는 수정 범위와 독립적인 테스트로 확인한다. 원문의 네 영역을 출발점으로 삼되, 실제 경계의 수는 프로젝트에 맞게 정한다.

## 참고 자료

- [Robert C. Martin — The Clean Architecture (2012)](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html): 네 영역과 의존성 규칙, 경계를 넘는 호출과 데이터의 기준.
- [Robert C. Martin — Clean Architecture (2011)](https://blog.cleancoder.com/uncle-bob/2011/11/22/Clean-Architecture.html): 요청·결과 데이터로 UI와 업무 코드를 분리하는 설명.
- [Alistair Cockburn — Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/): 애플리케이션과 외부를 Port와 Adapter로 연결하는 원문.
