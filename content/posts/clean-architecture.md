+++
title = '업무 규칙을 안쪽에 두는 클린 아키텍처'
slug = '17'
date = 2026-09-14T16:20:00+09:00
lastmod = 2026-09-14T16:20:00+09:00
draft = true
references_required = true
description = '주문 취소 예제로 Entity, Use Case, Adapter의 역할을 나누고, 클린 아키텍처의 의존성 규칙이 코드에서 어떻게 드러나는지 살펴봅니다.'
categories = ['애플리케이션 아키텍처']
tags = ['클린 아키텍처', '의존성 역전', 'Use Case', '소프트웨어 설계']
+++

[헥사고날 아키텍처](/posts/16/)에서는 애플리케이션과 외부 기술을 Port와 Adapter로 연결했다. 클린 아키텍처에도 익숙한 구성이 등장한다. 여기에 Entity와 Use Case라는 구분이 더해지면, 업무 코드 안에서 각각 무엇을 맡는지도 살펴보게 된다.

**클린 아키텍처는 업무 규칙을 중심에 두고, 소스코드의 의존성이 안쪽을 향하도록 구성한다.** 이 글은 로버트 C. 마틴의 2012년 글 [The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)를 기준으로, 주문 취소 기능을 직접 나누어 보며 그 의미를 살펴본다.

## 주문 취소 안에도 서로 다른 책임이 있다

주문을 취소하는 Service를 생각해 보자. 주문을 조회하고, 취소할 수 있는 상태인지 확인한 뒤, 상태를 변경해 저장한다. Controller는 요청을 받아 이 작업을 호출하고 처리 결과를 응답한다.

하나의 요청이지만, 바뀌는 이유는 서로 다르다.

| 변경할 내용 | 관련된 책임 |
|---|---|
| 취소 API의 URL이나 JSON 필드를 바꾼다 | 외부 요청과 응답 처리 |
| 고객의 취소 요청에 본인 확인 절차를 추가한다 | 특정 작업의 처리 절차 |
| 취소를 허용하는 주문 상태를 바꾼다 | 주문 자체의 업무 규칙 |
| 주문을 읽고 저장하는 SQL을 바꾼다 | 데이터 접근 구현 |

취소 API의 URL이 바뀌어도 주문의 취소 조건은 그대로일 수 있다. 이 둘이 서로의 코드를 얼마나 알고 있느냐에 따라 수정 범위가 달라진다.

클린 아키텍처의 네 영역을 이 예제에 대응시키면 다음과 같다. 그림의 가운데부터 **Entities, Use Cases, Interface Adapters, Frameworks & Drivers**가 놓인다.

![Entities를 중심으로 Use Cases, Interface Adapters, Frameworks and Drivers가 둘러싼 클린 아키텍처 원문 그림](https://blog.cleancoder.com/uncle-bob/images/2012-08-13-the-clean-architecture/CleanArchitecture.jpg "Robert C. Martin의 클린 아키텍처 구성도")

출처: [Robert C. Martin — The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

- **Entities:** 여러 작업에서 사용하는 핵심 업무 규칙.
- **Use Cases:** 특정 애플리케이션의 작업과 그 처리 흐름.
- **Interface Adapters:** 내부와 외부 사이의 데이터 변환과 연결.
- **Frameworks & Drivers:** DB, 웹 프레임워크 등의 기술과 연결 코드.

먼저 가운데 두 영역을 코드로 구분해 보자. 아래 코드는 원문의 개념을 주문 취소에 적용해 작성한 Java 예제다.

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

고객 화면에서 취소하든 관리자 기능에서 취소하든 `Order.cancel()`을 사용하면 같은 조건을 확인한다. `Order`가 관리하는 것은 주문의 상태와 규칙이므로, URL이나 SQL은 이 코드에 등장하지 않는다.

이처럼 핵심 업무 규칙을 담는 부분이 **Entity**다. 원문에서는 여러 애플리케이션에서도 사용할 수 있는 일반적인 업무 규칙을 이 영역에 둔다. 여기서의 Entity는 JPA의 `@Entity`보다 넓은 개념이다.

### 주문을 찾아 취소하는 흐름은 Use Case에 둔다

`Order`는 자기 상태를 바꿀 수 있지만, 취소할 주문을 저장소에서 찾는 작업은 아직 남아 있다. 예제의 저장 계약부터 정의하자. 저장소와 주고받을 `OrderData`에는 주문을 복원하는 데 필요한 값만 담는다.

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

`CancelOrderService`가 진행하는 주문 취소 작업이 **Use Case**다. 규칙의 판단은 `Order`에 맡기고, 조회부터 저장까지의 순서를 조정한다. 고객의 요청이라면 본인 주문인지 확인하는 절차를, 관리자 요청이라면 별도의 권한 확인 절차를 이 작업 흐름에 더할 수 있다.

지금은 역할을 보기 위해 조회·취소·저장만 남겼다. 실제 DB에 적용할 때는 이 작업의 트랜잭션 범위와 다른 요청에 의한 동시 변경도 함께 처리해야 한다.

## Adapter가 HTTP와 저장 기술을 연결한다

이제 `CancelOrderService`에 HTTP 요청을 연결해 보자. Controller는 URL에서 주문 ID를 받아 작업을 호출하고, 결과를 HTTP 응답으로 바꾼다.

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

`@PostMapping`과 `ResponseEntity`는 Controller에만 있다. 취소 결과의 HTTP 상태 코드를 바꿀 때도 이곳을 수정한다. 예제에서는 `CancelOrderResult`를 응답 본문에도 사용하며, API에 별도의 필드나 표현이 필요해지면 Controller 쪽에서 변환한다.

저장 쪽에서는 `JpaOrderAdapter`가 `OrderGateway`를 구현한다. JPA로 읽은 주문의 ID와 상태를 `OrderData`에 담아 반환하고, 저장 요청을 받으면 그 값을 JPA의 저장 형식으로 옮긴다.

**Interface Adapters**는 이런 변환과 연결이 모이는 영역이다. Controller는 HTTP 입력을 내부 호출로 바꾸고, 저장 Adapter는 내부의 저장 계약을 실제 데이터 접근으로 연결한다.

그 바깥의 **Frameworks & Drivers**에는 웹 프레임워크, DB 같은 기술과 이를 연결하는 설정이 놓인다. 예제에서는 Spring MVC, Hibernate와 DB가 해당한다. 구체적으로 어떤 Adapter를 사용할지는 바깥의 Bean 설정에서 정해 `CancelOrderService`에 연결한다.

## 의존성은 안쪽으로 향한다

구성요소를 나누었으니 서로 어떤 코드를 참조하는지도 확인해 보자. 원문의 **의존성 규칙**(Dependency Rule)은 안쪽 코드가 바깥쪽 코드의 이름과 데이터 형식에 의존하지 않도록 요구한다.

예제에서 `Order`는 `CancelOrderService`를 모른다. `CancelOrderService`도 Controller나 JPA Adapter를 모르며, 저장을 요청할 때는 `OrderGateway`를 참조한다. 반대로 Controller와 JPA Adapter는 안쪽에서 정의한 타입을 사용한다.

주문을 저장할 때의 호출 방향과 코드 의존 방향을 따로 적으면 차이가 보인다.

| 구분 | 관계 |
|---|---|
| 실행할 때 | `CancelOrderService` → `JpaOrderAdapter` |
| Service의 코드 | `CancelOrderService` → `OrderGateway` |
| Adapter의 코드 | `JpaOrderAdapter` → `OrderGateway` 구현 |

`OrderGateway`를 Use Case 쪽에 두면 두 클래스가 모두 안쪽의 계약에 의존한다. 실행할 때는 주입받은 Adapter의 메서드가 동작하므로, Use Case가 구현 클래스의 이름을 몰라도 DB에 저장할 수 있다. 앞 글에서 살펴본 의존성 역전이 이 경계에서도 쓰인다.

### 결과를 전달할 때도 같은 원리를 쓴다

원문 그림의 오른쪽 아래에는 Use Case가 **Presenter**를 호출하는 예가 있다. Presenter는 처리 결과를 화면에 표시하기 좋은 형태로 바꾸는 Adapter다. Use Case가 이 외부 구현을 직접 참조하는 대신, 안쪽에 결과 전달용 인터페이스를 둔다.

주문 취소에 적용한다면 다음과 같은 계약을 만들 수 있다.

```java
public interface CancelOrderOutputPort {
    void present(CancelOrderResult result);
}
```

이 구성을 선택하면 Use Case는 결과를 `outputPort.present(result)`에 전달하고, 바깥의 Presenter가 구현한 메서드가 실행된다. 저장 Adapter를 호출할 때처럼 계약의 소유자는 안쪽이다.

앞의 Java 예제는 결과를 반환하고 Controller에서 응답을 만드는 방식이다. Presenter를 따로 두는 방식과 비교할 때는 **누가 결과의 표현을 맡는지, Use Case가 그 표현 기술을 직접 알고 있는지**를 확인하면 된다.

## 경계를 넘는 데이터도 의존성을 만든다

인터페이스를 사용해도 그 반환형이 JPA 전용 타입이면 호출하는 쪽은 JPA를 알아야 한다. 메서드의 이름뿐 아니라 인자와 반환값도 의존 방향을 결정한다.

예제에서 `OrderGateway`가 반환하는 값은 `OrderData`다. 이 타입과 `OrderGateway`는 Use Case 쪽에 두고, `OrderStatus`는 도메인 쪽에 둔다. 따라서 안쪽 코드는 JPA 매핑 클래스나 HTTP 요청 객체를 가져올 필요가 없다.

또한 Controller가 받는 `CancelOrderResult`에는 주문 ID와 상태 문자열만 있다. Controller에 주문을 변경하는 `cancel()` 메서드까지 넘겨주지 않고, 작업 결과를 표현하는 데 필요한 값만 전달한 것이다.

이 구성에서는 상태 표시를 `CANCELED`에서 `주문 취소 완료`로 바꾸고 싶을 때 Controller나 Presenter에서 처리할 수 있다. 주문의 취소 조건이나 저장 계약은 그 표시 문구를 몰라도 된다. 마틴은 2011년의 [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2011/11/22/Clean-Architecture.html)에서도 Controller가 요청을 단순한 데이터로 풀어 작업에 전달하고, 결과 데이터로 화면을 구성하는 흐름을 설명한다.

## 헥사고날과 비교하면 무엇이 보일까

앞의 코드에는 Port 역할의 인터페이스와 Adapter가 있다. 그래서 같은 구성을 헥사고날의 관점으로도 설명할 수 있다.

| 예제의 구성요소 | 헥사고날에서 보면 | 클린에서 보면 |
|---|---|---|
| `Order` | 애플리케이션 내부 | Entity |
| `CancelOrderService` | 애플리케이션 내부 | Use Case |
| `OrderGateway` | 출력 Port | 안쪽에서 정의한 저장 계약 |
| `OrderController` | 입력 Adapter | Interface Adapters |
| `JpaOrderAdapter` | 출력 Adapter | Interface Adapters |
| 웹 프레임워크·DB | 외부 기술 | Frameworks & Drivers |

헥사고날은 애플리케이션과 외부가 만나는 Port와 Adapter에 초점을 둔다. 내부에는 프로젝트에 맞는 구조를 사용할 수 있다. 클린 아키텍처의 그림은 업무 코드도 Entity와 Use Case로 구분하고, 이들 사이까지 포함해 안쪽으로 향하는 의존성을 설명한다.

마틴은 헥사고날을 비롯한 여러 아키텍처의 아이디어를 종합했다고 밝힌다. 두 구조의 공통점이 많은 이유다. 헥사고날의 Port와 Adapter를 사용하면서 Entity와 Use Case의 책임도 나누면 두 접근의 원칙을 함께 적용할 수 있다.

## 분리한 효과는 변경과 테스트로 확인한다

이제 예제의 `Order`를 테스트하려면 Java 객체를 만들고 `cancel()`을 호출하면 된다. 결제 완료 주문은 취소되는지, 배송된 주문은 거절되는지 확인하는 데 웹 서버나 DB가 참여하지 않는다.

Use Case도 `OrderGateway`의 테스트용 구현을 연결하면 같은 방식으로 확인할 수 있다. 예를 들어 정해진 `OrderData`를 반환하고 저장 요청을 기록하는 가짜 저장소를 두면 다음을 검사할 수 있다.

- 결제 완료 주문을 취소하면 `CANCELED` 상태로 저장하는가?
- 배송된 주문의 취소가 거절되면 저장을 호출하지 않는가?
- 존재하지 않는 주문이라면 예외로 알리는가?

HTTP 경로를 수정하거나 JPA 조회 코드를 바꾼 뒤에도 이 테스트가 그대로 유지된다면, 업무 코드와 외부 구현을 분리한 효과를 확인할 수 있다. 실제 SQL과 트랜잭션 동작은 별도의 통합 테스트에서 확인한다.

그만큼 `OrderData` 같은 전달 타입과 변환 코드도 관리해야 한다. 적용 범위를 정할 때는 줄어드는 변경 비용과 새로 생기는 코드를 함께 보는 편이 좋다. 외부 구현을 고칠 때 업무 테스트까지 반복해서 수정한다면, 그 부분부터 경계를 점검할 이유가 있다.

## 정리

- Entity는 핵심 업무 규칙을, Use Case는 그 규칙을 사용해 작업을 진행하는 흐름을 맡는다.
- Adapter는 내부의 계약과 외부 기술을 연결한다. 바깥으로 호출해야 할 때도 계약을 안쪽에 두면 소스코드 의존성은 안쪽을 향한다.
- 경계를 나눈 효과는 수정 범위와 독립적인 테스트로 확인한다. 원문의 네 영역을 출발점으로 삼되, 실제 경계의 수는 프로젝트에 맞게 정한다.

## 참고 자료

- [Robert C. Martin — The Clean Architecture (2012)](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html): 네 영역과 의존성 규칙, 경계를 넘는 호출과 데이터의 기준.
- [Robert C. Martin — Clean Architecture (2011)](https://blog.cleancoder.com/uncle-bob/2011/11/22/Clean-Architecture.html): 요청·결과 데이터로 UI와 업무 코드를 분리하는 설명.
- [Alistair Cockburn — Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/): 애플리케이션과 외부를 Port와 Adapter로 연결하는 원문.
