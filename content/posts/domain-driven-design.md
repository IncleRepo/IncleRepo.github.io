+++
title = 'DDD, 도대체 도메인이 뭔데?'
slug = '15'
date = 2026-08-26T22:10:00+09:00
lastmod = 2026-08-28T23:00:00+09:00
draft = false
references_required = true
description = '주문 취소 사례를 따라가며 업무 이해가 도메인 모델, 바운디드 컨텍스트, Entity·Value Object·Aggregate와 코드로 이어지는 과정을 살펴봅니다.'
categories = ['애플리케이션 아키텍처']
tags = ['DDD', '도메인 모델', '바운디드 컨텍스트', '애그리거트', '도메인 이벤트']
+++

[레이어드 아키텍처](/posts/14/)가 코드를 책임과 의존 방향에 따라 나누는 구조라면, DDD는 다른 관점에서 업무를 이해하고 그 규칙을 모델과 코드로 표현하는 방법을 다룬다.

더 먼저 풀어야 할 문제가 있다. **우리가 같은 업무를 정말 같은 의미로 이해하고 있는가?** DDD는 이 질문에서 출발한다.

## 같은 주문 취소를 다르게 이해할 때

정의부터 외우기 전에 DDD가 필요한 상황부터 살펴보자. 주문팀, 결제팀, 물류팀과 고객센터가 따로 있는 쇼핑몰 회사를 가정해 보자.

어느 날 주문 취소 기능을 개선해 달라는 요청이 들어온다.

> **기획팀:** 배송 전에는 주문을 취소할 수 있게 해주세요.
>
> **주문팀:** 배송 전이 정확히 어느 상태까지인가요?
>
> **물류팀:** 송장이 발급되면 출고로 봅니다.
>
> **고객센터:** 송장이 나와도 실제 출고 전에는 취소해 주고 있는데요?
>
> **결제팀:** 주문 취소와 결제 환불은 다른 처리입니다.
>
> **개발자:** 정확한 기준은 어디에 정리되어 있나요?
>
> **기획팀:** 이전 담당자분이 잘 알고 계셨는데요.
>
> **개발자:** 그분은 퇴사하셨는데요.

잠시 정적이 흐른다.

그래도 일정은 잡혀 있다. 각 팀은 전달받은 내용을 바탕으로 기능을 만든다. 얼마 뒤 주문은 취소됐는데 환불이 되지 않고, 환불은 됐는데 쿠폰이 돌아오지 않는 일이 생긴다. 고객센터에서는 취소할 수 있다고 안내했지만 API에서는 거절하기도 한다.

코드보다 먼저 어긋난 것은 업무에 대한 이해였다. 같은 `주문 취소`를 주문팀은 상태 변경으로, 결제팀은 환불로, 물류팀은 출고 중단으로 받아들였다. 관련 규칙도 기획서와 코드, 오래된 대화와 담당자의 기억 속에 흩어져 있었다.

DDD, 즉 도메인 주도 설계는 복잡한 업무를 도메인 전문가와 개발자가 함께 이해하고, 그 이해를 모델과 코드로 이어 가는 접근이다. 사람의 설명과 코드가 같은 업무를 가리키게 만드는 것이 출발점이다.

이 글은 업무에서 발견한 문제가 도메인 모델과 경계, 코드와 컨텍스트 사이의 협력으로 이어지는 흐름을 따라간다.

![업무 이해에서 모델과 경계를 거쳐 실행과 협력으로 이어지는 DDD의 흐름](/images/posts/domain-driven-design/ddd-concept-map.svg "DDD를 따라가는 네 단계")

## 업무를 이해해 모델을 만든다

**도메인**은 소프트웨어로 해결하려는 현실의 업무 영역이다. 이 글에서는 온라인 쇼핑몰 운영을 하나의 도메인으로 보고, 그 안의 주문, 결제와 배송 업무를 차례로 살펴본다. 주문 취소 기능을 만든다면 화면이나 테이블보다 먼저 주문은 어떤 정보를 가지고 있는지, 상태는 언제 바뀌는지, 어느 시점부터 취소할 수 없는지 확인해야 한다.

업무에서 중요한 대상과 상태, 행동과 규칙을 추려 표현한 것이 **도메인 모델**이다. 현재 해결할 문제에 필요한 부분을 골라 현실의 업무를 단순화한 표현이다. 이 글에서는 `배송이 시작된 주문은 취소할 수 없다`처럼 반드시 지켜야 하는 판단과 규칙을 **업무 규칙** 또는 **도메인 로직**이라고 부르겠다.

주문 업무를 살펴보면 다음과 같은 내용이 보인다.

1. 주문에는 여러 상품이 담긴다.
2. 주문은 현재 상태를 가진다.
3. 배송이 시작된 주문은 취소할 수 없다.

주문과 주문 상품은 상태가 바뀌는 **대상**, 주문 상태는 **값**, 취소는 **행동**이다. 여기에 배송 시작 이후에는 허용하지 않는다는 **규칙**이 붙는다. 이 내용은 이후 `Order`, `OrderLine`, `OrderStatus`와 `Order.cancel()` 같은 코드로 표현하고 구현하면서 다듬는다.

### 도메인 전문가는 업무의 예외를 알고 있다

개발자가 코드만 보고 모든 업무 규칙을 알아내기는 어렵다. 실제 규칙과 예외를 깊이 이해하는 **도메인 전문가**와 함께 살펴봐야 한다. 주문 상태를 관리하는 운영자, 환불 규정을 정하는 기획자와 반복되는 문의를 처리하는 고객센터 직원 모두 자신이 맡은 업무의 전문가가 될 수 있다.

개발자는 이들과 함께 주문 취소의 흐름을 펼쳐 본다.

1. 고객이 주문 취소를 요청한다.
2. 주문이 취소 가능한 상태인지 확인한다.
3. 주문을 취소한다.
4. 결제에는 환불을, 물류에는 출고 중단을 요청한다.
5. 사용한 쿠폰과 재고를 복구한다.

목록만 보면 단순하지만 실제로 구현하려면 답해야 할 질문이 많다. 환불 요청이 실패하면 주문 취소도 되돌려야 할까? 이미 출고된 주문은 취소가 아니라 반품으로 처리해야 할까? 이 질문에 하나씩 답하면서 주문의 상태와 취소 조건이 구체화되고, 도메인 모델에 담아야 할 규칙도 발견한다.

업무 규칙을 함께 정하려면 먼저 사용하는 말의 의미부터 맞아야 한다. 여기서 유비쿼터스 언어가 필요해진다.

### 같은 말을 같은 의미로 사용한다

주문팀의 `취소`, 결제팀의 `환불`, 물류팀의 `출고 중단`은 서로 연결되어 있지만 같은 행동은 아니다. 이 차이를 합의하지 않으면 회의에서는 통하는 것처럼 보여도 코드에서는 다시 어긋난다. 도메인 전문가와 개발자가 함께 정한 공통 언어를 **유비쿼터스 언어**라고 하며, 우리말로는 보편 언어라고도 부른다.

회의에서 주문 취소라고 부른 개념은 화면과 문서, 테스트와 코드에서도 같은 의미로 이어져야 한다.

```java
order.cancel();
refund.request();
shipment.stopDispatch();
```

`cancel()`은 `취소가 정확히 무엇인지` 합의한 결과가 코드에 드러난 이름이다. `changeStatus(4)`만 남겨 두면 숫자 `4`의 의미를 다시 찾아야 한다. 함께 정한 언어는 문서와 대화뿐 아니라 코드에서도 이어져야 한다.

이처럼 용어를 맞추려면 각자가 알고 있는 업무 흐름을 한곳에 펼쳐 놓고 이야기할 자리가 필요하다.

### 대화 속 사건을 시간 순서로 펼쳐 본다

업무 흐름과 용어를 찾는 방법 중 하나가 **이벤트 스토밍**이다. 업무에서 일어난 사건을 시간 순서대로 펼쳐 놓고 대화하면서 빠진 과정과 서로 다른 해석을 찾는다. 앞의 주문 취소 회의를 조금 더 이어 가보자.

> **진행자:** 고객이 취소를 요청하면 가장 먼저 무엇을 확인하나요?
>
> **주문팀:** 주문이 취소 가능한 상태인지 확인합니다.
>
> **물류팀:** 출고가 시작됐다면 취소할 수 없습니다.
>
> **고객센터:** 여기서 말하는 출고 시작은 송장 발급인가요, 실제 상품 이동인가요?
>
> **물류팀:** 창고에서 상품을 집기 시작한 시점입니다.
>
> **결제팀:** 주문이 취소된 뒤에는 저희가 환불을 처리합니다.

대화에서 확인한 흐름을 가장 단순하게 놓으면 다음과 같다.

![고객의 주문 취소 요청이 규칙 판단과 주문 취소 사건을 거쳐 환불과 출고 중단 요청으로 이어지는 흐름](/images/posts/domain-driven-design/order-cancel-conversation-flow.svg "주문 취소 대화에서 발견한 Command와 Event")

**Command**는 어떤 행동을 해 달라는 요청이고, **Event**는 이미 일어난 사실이다. 이벤트 스토밍에서는 Event와 이를 일으킨 Command, 사람과 뒤따르는 정책을 한 흐름에 놓을 수 있다.

이미 상품을 집기 시작했다면 `주문이 취소되었다`라는 Event에 도달하지 못한다. 흐름을 거꾸로 따라가면 상태를 가진 대상은 **주문**, 주문이 제공할 행동은 **취소**이고, 여기에 `상품을 집기 시작한 주문은 취소할 수 없다`는 규칙이 붙는다. 업무에서 말하던 취소가 `Order.cancel()`로 이어지는 과정이다.

이벤트 스토밍은 여러 사람이 알고 있던 사건과 규칙을 하나의 흐름으로 펼쳐 모델의 재료를 찾는 방법이다.

![사건과 흐름을 연결해 표현한 EventStorming 이미지](/images/posts/domain-driven-design/event-storming.jpg "출처: EventStorming 공식 사이트")

대화와 구현에서 발견한 사건과 예외를 모델에 반영하면 주문팀 안에서는 같은 말로 같은 모델을 가리킬 수 있다. 하지만 상품팀과 주문팀이 말하는 `상품`처럼 업무가 달라지면 필요한 정보도 달라진다. 이제 하나의 모델이 어디까지 같은 의미로 통하는지 경계를 정해야 한다.

## 하나의 모델이 통하는 경계를 찾는다

쇼핑몰이라는 큰 업무를 한 번에 모델링하기는 어렵다. 그래서 주문, 결제와 배송처럼 비교적 작은 문제 영역으로 나누어 살펴본다. 이렇게 나눈 각각의 업무 영역을 **서브도메인**이라고 한다.

업무를 나눈 뒤에는 소프트웨어 안에서 각 모델이 어디까지 같은 의미로 통할지도 정해야 한다. 이 경계가 **바운디드 컨텍스트**다.

| 구분 | 경계를 나누는 기준 |
|---|---|
| 서브도메인 | 현실의 큰 업무를 나눈 문제 영역 |
| 바운디드 컨텍스트 | 모델과 언어가 같은 의미로 통하는 범위 |

하나의 주문 컨텍스트가 주문 서브도메인 전체를 맡을 수도 있고, 업무가 커지면 주문 접수와 주문 이력을 서로 다른 컨텍스트로 나눌 수도 있다. 서브도메인은 현실의 문제를 나누고, 바운디드 컨텍스트는 그 문제를 표현할 모델의 범위를 정한다. 둘의 대응 관계는 업무와 모델의 경계에 따라 달라진다.

상품팀과 주문팀이 사용하는 `상품`을 예로 들어 보자. 상품팀에는 현재 판매 가격, 상품 설명과 선택 가능한 옵션이 중요하다. 주문팀에는 고객이 주문한 당시의 이름, 가격과 수량이 필요하다. 상품팀이 오늘 가격을 바꾸더라도 어제 결제한 주문 금액은 그대로 남아야 한다.

따라서 상품 컨텍스트의 `Product`와 주문 컨텍스트의 주문 상품은 서로 다른 모델로 둔다. 풀려는 문제가 다르면 필요한 정보와 규칙도 달라지기 때문이다.

주문 취소도 마찬가지다. 이 글에서는 설명을 위해 주문, 결제와 배송을 서로 다른 컨텍스트로 나눈다. 주문 컨텍스트는 주문 상태를 바꾸고, 결제 컨텍스트는 환불 가능 여부와 금액을 계산하며, 배송 컨텍스트는 출고를 멈출 수 있는지 판단한다. 실제 경계는 조직 이름이나 업무 명사보다 모델의 의미와 변경 패턴을 살펴 정한다.

바운디드 컨텍스트는 패키지나 Gradle 모듈로 드러낼 수도 있고, 독립 서비스로 분리할 수도 있다. 하나의 모놀리식 애플리케이션 안에도 여러 바운디드 컨텍스트가 존재할 수 있다.

바운디드 컨텍스트 하나에도 서로 다른 규칙을 맡는 객체 묶음이 여러 개 들어갈 수 있다. 뒤에서 살펴볼 **Aggregate**는 컨텍스트 안에서 상태 변경의 일관성을 관리하는 더 작은 경계다.

![주문 컨텍스트 안의 여러 애그리거트와 상품 컨텍스트의 관계](/images/posts/domain-driven-design/context-and-aggregate.svg "바운디드 컨텍스트와 애그리거트 경계의 차이")

그림의 주문 컨텍스트에는 Order Aggregate와 Coupon Aggregate가 있고, 각자 맡은 규칙을 지킨다. 주문 컨텍스트는 상품 컨텍스트의 `Product` 대신, 두 컨텍스트가 합의한 형식으로 주문에 필요한 정보만 받는다.

지금까지는 멀리서 업무와 모델의 큰 경계를 봤다. 서브도메인과 바운디드 컨텍스트를 나누고 컨텍스트 사이의 관계를 정하는 관점을 **전략적 설계**라고 한다.

이제 카메라를 컨텍스트 안쪽으로 옮긴다. Entity, Value Object와 Aggregate로 모델을 만들고 Domain Service, Repository와 Domain Event 같은 패턴으로 규칙과 저장, 중요한 변화를 표현하는 관점이 **전술적 설계**다.

유비쿼터스 언어는 두 설계를 관통한다. 업무 대화에서 합의한 용어가 모델과 코드에서도 같은 뜻으로 쓰여야 한다. 아래 그림에서는 DDD가 서로 다른 크기의 설계 문제를 다룬다는 점만 보면 된다.

![전략적 설계는 업무와 모델의 큰 경계를 정하고, 전술적 설계는 경계 안의 모델을 구체화한다](/images/posts/domain-driven-design/strategic-tactical-overview.svg "DDD의 전략적 설계와 전술적 설계")

여기까지는 코드보다 큰 경계를 찾았다.

| 질문 | 지금까지 찾은 개념 |
|---|---|
| 어떤 업무를 해결하는가? | Domain |
| 어떤 말을 같은 의미로 사용하는가? | Ubiquitous Language |
| 현실의 업무를 어떻게 나누는가? | Subdomain |
| 모델의 의미가 어디까지 같은가? | Bounded Context |

이제 주문 컨텍스트 안으로 들어가 대상과 값, 변경의 일관성을 살펴보자.

## 도메인 모델을 객체로 표현한다

주문 취소 규칙을 코드에 담으려면 무엇을 계속 추적하고, 어떤 값을 하나로 다루며, 어디까지 함께 변경할지 정해야 한다.

### Entity는 식별자로 같은 대상을 추적한다

속성이 바뀌어도 같은 대상으로 계속 추적해야 하는 객체를 **Entity**라고 한다.

주문의 배송지와 상태는 바뀔 수 있다. 그래도 주문 번호가 같다면 어제 조회한 주문과 오늘 조회한 주문은 같은 주문이다. Entity는 이처럼 식별자를 기준으로 같은 대상을 이어서 추적한다.

이하 코드는 모델링 흐름에 집중하기 위해 생성자와 일부 세부 구현을 생략한다.

```java
public class Order {

    private final OrderId id;
    private OrderStatus status;
    private Money totalPrice;
    private ShippingAddress shippingAddress;

    public void cancel() {
        if (!status.isCancelable()) {
            throw new CannotCancelOrderException();
        }

        status = OrderStatus.CANCELED;
    }

    public void changeShippingAddress(
        ShippingAddress newAddress
    ) {
        shippingAddress = newAddress;
    }

    public OrderId id() {
        return id;
    }

    public Money totalPrice() {
        return totalPrice;
    }
}
```

외부에서는 `cancel()`을 호출해 주문 취소를 요청한다. 메서드 안에는 주문 취소라는 업무 의도와 규칙, 상태 변경이 한 행동으로 드러난다.

주문은 주문 번호로 계속 추적한다. 금액과 배송지는 식별자보다 어떤 값을 담고 있는지가 중요하다.

### Value Object는 값으로 의미를 표현한다

식별자보다 값 자체와 그 의미가 중요한 객체를 **Value Object**라고 한다. 금액, 주소와 기간이 대표적이다. 주문 금액을 `BigDecimal` 하나로만 사용하면 그 숫자의 단위와 허용 범위를 매번 주변 코드에서 확인해야 한다. 프로젝트가 원화만 다룬다고 가정하고 `Money`로 묶으면 금액이라는 의미와 규칙을 한곳에 둘 수 있다.

```java
public record Money(BigDecimal amount) {

    public static final Money ZERO =
        new Money(BigDecimal.ZERO);

    public Money {
        Objects.requireNonNull(amount);
        amount = amount.stripTrailingZeros();

        if (amount.signum() < 0) {
            throw new IllegalArgumentException(
                "금액은 음수일 수 없습니다."
            );
        }
    }

    public Money add(Money other) {
        return new Money(amount.add(other.amount));
    }

    public Money multiply(BigDecimal multiplier) {
        return new Money(amount.multiply(multiplier));
    }
}
```

두 `Money`의 금액이 같다면 같은 값으로 볼 수 있다. Value Object는 자신이 가진 값 전체로 의미가 결정되기 때문이다. 값이 중간에 바뀌면 같은 객체가 다른 의미를 갖게 되므로 불변 객체로 만드는 경우가 많다.

배송지를 바꿀 때도 기존 주소를 수정하기보다 새로운 주소 값을 만들어 전달할 수 있다.

```java
ShippingAddress newAddress =
    new ShippingAddress("서울", "강남구");

order.changeShippingAddress(newAddress);
```

새 주소를 만드는 일과 주문의 배송지를 바꾸는 일은 다르다. `ShippingAddress`는 새 값으로 만들고, `Order`가 그 값을 받아 자신의 상태를 변경한다.

이제 주문과 주문 상품, 배송지를 각각 표현할 수 있다. 다음으로 정할 것은 변경의 입구다. 어느 객체를 통해 상태를 바꾸고, 어디까지 한 번에 올바른 상태를 보장해야 할까?

### Aggregate는 Root가 일관성을 지키는 경계다

주문 상품의 수량을 바꾸면 주문 총금액도 다시 계산해야 한다. 이미 취소한 주문이라면 수량 자체를 바꿀 수 없어야 한다. 외부 코드가 `OrderLine`을 직접 꺼내 수량만 바꾸면 이 규칙을 빠뜨릴 수 있다.

그래서 주문과 주문 상품처럼 함께 규칙을 지켜야 하는 객체를 하나의 경계로 묶고, 변경 요청은 대표 객체를 거치게 한다. 이 경계가 **Aggregate**이며, 주문 Aggregate에서는 `Order`가 대표 객체인 **Aggregate Root**가 된다. 외부에서는 `OrderLine`을 직접 바꾸지 않고 Root인 `Order`에 변경을 요청하고, `Order`는 내부 객체의 상태까지 살피며 주문 규칙을 지킨다.

```java
public void changeQuantity(OrderLineId lineId, int quantity) {
    if (status == OrderStatus.CANCELED) {
        throw new IllegalStateException(
            "취소된 주문은 수량을 바꿀 수 없습니다."
        );
    }

    OrderLine line = findOrderLine(lineId);
    line.changeQuantity(quantity);
    totalPrice = calculateTotalPrice();
}
```

다음 그림에서는 `OrderLine`과 `ShippingAddress`를 바꾸는 요청이 모두 Root인 `Order`를 거친다. `Order`가 내부 변경을 통제한다는 점을 중심으로 보면 된다.

![Order가 Aggregate Root가 되어 주문 상품과 배송지 변경을 통제하는 구조](/images/posts/domain-driven-design/order-aggregate.svg "주문 Aggregate의 구조")

Aggregate의 경계는 한 트랜잭션에서 함께 지켜야 할 규칙을 기준으로 찾는다. 주문 상품의 수량과 총금액처럼 함께 맞아야 하는 상태를 같은 경계에서 관리한다. 한 작업이 끝났을 때 이 경계 안의 상태가 업무 규칙을 만족하는 것이 **일관성**이다.

주문 Aggregate에서는 `Order`가 Root이자 Entity이고, `OrderLine`은 Root를 통해 관리되는 Entity, `ShippingAddress`는 Value Object다. 모든 Aggregate Root는 Entity지만 모든 Entity가 Root인 것은 아니다.

Aggregate가 너무 크면 작은 변경에도 많은 데이터를 불러오거나 잠가야 할 수 있다. 반대로 너무 작게 나누면 강한 일관성이 필요한 규칙이 여러 Aggregate에 흩어진다. Aggregate는 트랜잭션 경계를 결정하는 중요한 기준이며, 한 Aggregate의 변경을 한 트랜잭션에서 완료하는 것을 기본으로 삼을 수 있다. 여러 Aggregate가 협력한다면 하나의 트랜잭션으로 묶을지, 별도 트랜잭션과 Event로 이어갈지는 필요한 일관성과 실패 처리 방식에 따라 정한다.

Aggregate의 이름은 기능 이름과도 구분한다. 회원가입과 주문 취소는 실행해야 할 **Use Case**이고, `Member`와 `Order`는 상태와 생명주기를 가진 Aggregate 후보다.

## 한 객체에 담기 어려운 업무 규칙을 다룬다

주문 취소 조건은 `Order` 안에 자연스럽게 들어간다. 금액 계산도 `Money`가 맡을 수 있다. 업무 규칙은 먼저 그 규칙을 가장 잘 아는 Entity나 Value Object에 둔다.

할인처럼 한 객체에 자연스럽게 귀속시키기 어려운 업무 규칙도 있다. 이 규칙 자체가 독립적인 도메인 개념이라면 **Domain Service**로 표현할 수 있다. Domain Service를 둘지는 업무 규칙을 어느 객체가 책임지는 것이 자연스러운지를 기준으로 판단한다.

회원 등급과 주문을 같은 판매 정책 컨텍스트에서 다룬다고 가정해 보자. 두 정보를 함께 살펴 할인액을 계산하는 정책은 다음과 같다.

```java
public class DiscountPolicy {

    public Money calculate(
        MemberGrade grade,
        Order order
    ) {
        if (grade.isVip()) {
            return order.totalPrice()
                .multiply(new BigDecimal("0.10"));
        }

        return Money.ZERO;
    }
}
```

`DiscountPolicy`는 회원 등급과 주문 금액을 바탕으로 할인액을 계산한다. 어느 등급과 주문을 불러오고 계산 결과를 어디에 저장할지는 결정하지 않는다.

여기까지 컨텍스트 안의 모델을 구성하는 네 가지 개념을 살펴봤다.

| 개념 | 모델에서 맡는 일 |
|---|---|
| Entity | 식별자로 같은 대상을 계속 추적 |
| Value Object | 값 자체와 의미를 표현 |
| Aggregate | Root가 상태와 업무 규칙의 일관성을 관리 |
| Domain Service | 한 객체에 자연스럽게 귀속하기 어려운 업무 규칙을 표현 |

이제 앞에서 만든 도메인 모델을 실제 주문 취소 기능에서 어떻게 사용하는지 살펴보자.

## 도메인 모델로 주문 취소를 실행한다

### Application Service는 작업의 흐름을 조정한다

앞에서 `Order.cancel()`에 주문 취소 규칙을 담았다. 주문 취소 기능을 완성하려면 취소할 주문을 불러오고, `cancel()`을 호출한 뒤, 바뀐 주문을 다시 저장해야 한다. 이처럼 도메인 모델을 사용해 하나의 기능을 처음부터 끝까지 진행하는 흐름을 **Application Service**가 맡는다.

```java
@Service
@RequiredArgsConstructor
public class CancelOrderService {

    private final OrderRepository orderRepository;

    @Transactional
    public void cancel(OrderId orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(OrderNotFoundException::new);

        order.cancel();

        orderRepository.save(order);
    }
}
```

코드의 흐름은 다음 세 단계다.

```text
주문 조회 → 취소 규칙 실행 → 변경 결과 저장
```

`CancelOrderService`는 이 순서를 조정하고 트랜잭션을 묶는다. 취소할 수 있는 주문인지는 `Order.cancel()`이 판단한다. HTTP API, 관리자 기능과 배치 중 어디에서 취소를 요청하더라도 같은 규칙을 지켜야 하기 때문이다.

앞의 할인 예시도 역할은 같다. Application Service가 주문과 회원 등급을 준비하면 `DiscountPolicy`가 할인액을 계산한다. Application Service는 작업의 순서를 맡고, 도메인 모델은 업무적으로 가능한지를 판단한다.

### Repository는 Aggregate를 저장하고 복원한다

Application Service가 작업을 진행하려면 Aggregate를 불러오고 변경 결과를 저장할 통로가 필요하다. 이 역할을 **Repository**가 맡는다. DDD에서는 Repository를 Aggregate 단위의 저장과 복원을 담당하는 컬렉션 같은 추상화로 본다.

```java
public interface OrderRepository {

    Optional<Order> findById(OrderId orderId);

    void save(Order order);
}
```

`orders`와 `order_lines`가 서로 다른 테이블에 있더라도 수량 변경에는 `Order` Aggregate를 복원한다. Root를 통해 변경해야 수량과 총금액을 함께 관리할 수 있기 때문이다.

Aggregate를 저장하는 경로를 정했다면, 값을 읽기만 하는 기능도 같은 경로를 따라야 하는지 살펴볼 수 있다.

### 조회는 필요한 형태로 가져온다

지금까지 살펴본 주문 취소는 상태를 바꾸는 기능이다. 업무 규칙을 지켜야 하므로 Application Service가 `Order` Aggregate를 불러와 변경한다.

주문 목록처럼 값을 읽기만 하는 기능은 목적이 다르다. Aggregate 전체를 복원하지 않고 조회 전용 DAO나 Projection으로 화면에 필요한 값만 가져올 수 있다.

| 작업 | 처리 흐름 |
|---|---|
| 변경 | `Command → Application Service → Aggregate → Repository` |
| 조회 | `Query → DAO / Projection → Response` |

조회 전용 DAO와 Projection은 화면에 필요한 값을 효율적으로 읽는 방법이다. **CQRS**는 여기서 더 나아가 Command와 Query에 별도의 모델과 처리 경로를 둔다. 가장 단순한 형태에서는 하나의 애플리케이션과 데이터베이스를 유지하면서 읽기와 쓰기 모델만 구분할 수도 있다.

이제 주문 컨텍스트 안에서는 규칙을 판단하고, 기능을 실행하고, 상태를 저장할 수 있다. 주문 취소 뒤에 필요한 환불과 출고 중단은 결제와 배송 컨텍스트의 몫이다.

## 컨텍스트는 관계와 계약을 정해 협력한다

주문 컨텍스트가 주문을 취소해도 환불은 결제 컨텍스트가, 쿠폰 복구는 쿠폰 컨텍스트가 처리한다. 경계를 나눈 뒤에도 하나의 업무를 완성하려면 계속 협력해야 한다.

협력할 때는 **어느 컨텍스트가 모델을 소유하고, 경계에서 어떤 계약을 주고받을지**를 정한다. 이 절에서는 먼저 컨텍스트 사이의 관계를 고르고, 상품 정보를 주문 모델로 번역하는 흐름과 주문 취소 사실을 외부에 알리는 흐름을 차례로 살펴본다.

### 관계 패턴은 두 질문으로 나눠 본다

상품 컨텍스트가 상품 정보를 제공하고 주문 컨텍스트가 이를 사용한다면, 상품은 **업스트림**, 주문은 **다운스트림**이다. Context Map의 관계 패턴은 이 두 팀이 어떻게 협력할지 설명한다. 여기서는 이해하기 쉽게 두 가지 질문으로 나눠 보자.

**1. 두 팀은 모델을 얼마나 함께 사용할까?**

- **공유 커널(Shared Kernel) — 같이 쓰되, 작게 쓴다.** 두 팀에 정말 같은 의미를 가진 모델과 코드만 공동으로 관리한다. 이를 바꿀 때도 함께 합의한다.
- **순응자(Conformist) — 상대의 말에 맞춘다.** 다운스트림이 별도의 번역 없이 업스트림의 모델과 언어를 받아들인다. 빠르게 연결할 수 있지만 업스트림이 바꾸면 함께 따라가야 한다.
- **부패 방지 계층(Anti-Corruption Layer) — 경계에 통역사를 둔다.** 다운스트림이 외부 계약을 자신의 언어와 모델로 번역한다. 변환 코드를 관리하는 대신 자신의 모델을 독립적으로 유지할 수 있다.
- **각자의 길(Separate Ways) — 안 엮이는 것도 설계다.** 통합 비용이 얻는 이익보다 크다면 각자 필요한 기능을 구현한다. 일부 중복을 감수하는 대신 서로 독립적으로 일한다.

위 네 패턴은 두 팀이 모델과 변경의 영향을 얼마나 나눠 가질지 보여준다. 서로 연결하기로 했다면 다음으로 업스트림이 무엇을 공개할지 정한다.

**2. 업스트림은 어떤 공식 출입구와 형식을 제공할까?**

- **공개 호스트 서비스(Open Host Service) — 필요한 기능을 정식 출입구로 공개한다.** 여러 다운스트림이 같은 방식으로 이용할 수 있는 공통 서비스를 제공한다.
- **공표된 언어(Published Language) — 출입구에서 주고받을 말을 정한다.** 요청과 응답, 메시지에 담을 데이터 형식과 의미를 공개된 계약으로 만든다.

상품 조회 기능에 적용하면 다음처럼 구분할 수 있다.

| 구분 | 상품 컨텍스트의 예 | 정하는 것 |
|---|---|---|
| 공개 호스트 서비스 | `GET /products/{productId}` | 어디로 어떻게 요청할지 |
| 공표된 언어 | `ProductInfo { productId, name, currentPrice }` | 어떤 이름과 형식으로 데이터를 주고받을지 |

쉽게 말해 공개 호스트 서비스는 **공식 창구**, 공표된 언어는 그 창구에서 사용하는 **공통 양식과 용어**다. REST 대신 gRPC나 Kafka를 사용해도 이 구분은 같다.

두 질문의 답은 함께 조합된다. 이번 예에서는 상품 컨텍스트가 공개 호스트 서비스와 공표된 언어로 `ProductInfo`를 제공하고, 주문 컨텍스트는 부패 방지 계층을 두어 이를 `OrderedProduct`로 번역한다. 하나의 관계 안에 세 패턴이 함께 사용된 셈이다.

### 공개 계약을 내 모델로 번역한다

상품 컨텍스트는 현재 상품 정보를 `ProductInfo`라는 계약으로 제공한다. 주문 컨텍스트는 이 값을 주문 당시의 상품 정보인 `OrderedProduct`로 바꿔 저장한다.

```java
public record ProductInfo(
    long productId,
    String name,
    BigDecimal currentPrice
) {
}

public record OrderedProduct(
    long productId,
    String name,
    Money orderPrice
) {
}

public class ProductTranslator {

    public OrderedProduct translate(ProductInfo source) {
        return new OrderedProduct(
            source.productId(),
            source.name(),
            new Money(source.currentPrice())
        );
    }
}
```

`ProductTranslator`는 외부 계약을 주문 컨텍스트의 언어로 바꾸는 **부패 방지 계층(Anti-Corruption Layer, ACL)** 역할을 한다. 상품의 현재 가격이나 응답 구조가 바뀌면 번역 코드가 먼저 영향을 받고, 주문 모델은 `OrderedProduct`라는 자신의 의미를 유지한다.

상품 정보를 받아오는 경계를 정했으니, 이제 주문 컨텍스트에서 일어난 변화를 밖으로 알리는 경계를 살펴보자.

### 요청과 사건도 컨텍스트의 계약이 된다

상품 정보를 가져오는 일은 한 컨텍스트가 다른 컨텍스트에 필요한 값을 요청하는 흐름이었다. 주문 취소처럼 다른 컨텍스트의 후속 작업을 일으키는 변화는 요청과 사건을 다음처럼 구분해 표현할 수 있다.

- **Command — `CancelOrder`**: 주문 컨텍스트에 취소를 요청한다. 현재 상태와 규칙에 따라 성공하거나 거절될 수 있다.
- **Domain Event — `OrderCanceled`**: 주문 컨텍스트 안에서 주문이 취소되었다는 사실을 표현한다.
- **Integration Event — `OrderCanceledIntegrationEvent`**: 확정된 주문 취소 사실 중 다른 컨텍스트에 필요한 정보를 공개하는 계약이다.

`CancelOrder`가 처리되면 주문 모델은 `OrderCanceled`를 기록한다. 주문 취소가 확정된 뒤에는 이를 외부 계약인 `OrderCanceledIntegrationEvent`로 바꿔 공개한다. 결제와 쿠폰 컨텍스트는 이 계약을 받아 각자의 규칙에 따라 환불과 쿠폰 복구를 처리한다.

이렇게 관계와 계약을 정한 뒤 실행 환경에 맞는 전달 기술을 고른다. 같은 애플리케이션에서는 인터페이스 호출이나 Spring Application Event를, 프로세스 사이에서는 HTTP·gRPC나 Kafka 같은 메시지 브로커를 사용할 수 있다.

### Context Map은 선택한 관계를 보여준다

지금까지 상품에서 주문으로 들어오는 정보와 주문에서 결제·쿠폰으로 나가는 사건의 계약을 정했다. 이 관계를 한 장에 모아 보면 어느 컨텍스트가 정보를 제공하고, 경계에서 어떤 번역이 일어나며, 변경이 어디까지 이어지는지 확인할 수 있다. 이를 **Context Map**이라고 한다.

앞에서 살펴본 관계를 다음처럼 표현할 수 있다.

![상품, 주문, 결제와 쿠폰 컨텍스트가 API 계약, ACL과 Integration Event로 협력하는 Context Map](/images/posts/domain-driven-design/context-map-example.svg "주문 기능을 중심으로 정리한 Context Map의 예")

상품 컨텍스트는 `ProductInfo`를 제공하고 주문 컨텍스트는 ACL에서 이를 번역한다. 주문 취소 결과는 Integration Event로 결제와 쿠폰 컨텍스트에 전달한다. **Context Map은 각 경계에서 선택한 관계와 계약을 기록한 지도다.**

## DDD는 언제 도움이 될까?

DDD는 다음처럼 업무 자체를 이해하고 유지하기 어려운 영역에서 특히 도움이 된다.

- 같은 용어를 팀마다 다른 의미로 사용한다.
- 상태와 예외가 많아 기능을 바꿀 때마다 조건을 빠뜨린다.
- 중요한 규칙이 여러 Service와 배치 코드에 흩어져 있다.
- 테이블 구조보다 업무 규칙의 변화가 코드를 더 자주 흔든다.
- 개발자만으로 규칙을 정하기 어려워 업무 담당자와 계속 대화해야 한다.

업무 규칙이 거의 없는 단순한 CRUD나 외부 데이터를 그대로 전달하는 기능은 Service와 Repository만으로도 충분할 수 있다. 모델링에 들이는 비용도 해결하려는 문제의 복잡도에 맞춰야 한다.

한 시스템 안에서도 적용 수준은 달라질 수 있다. 어디에 깊게 모델링할 시간과 비용을 쓸지 판단하기 위해 서브도메인을 회사의 경쟁력과 역할에 따라 핵심, 지원과 일반으로 구분하기도 한다.

아래는 독자적인 추천 기술로 경쟁하는 가상의 쇼핑몰을 가정한 예다. 서브도메인의 분류는 회사가 어디에서 경쟁력을 만드는지에 따라 달라진다.

| 구분 | 의미 | 쇼핑몰 예시 |
|---|---|---|
| 핵심 | 회사의 경쟁력을 만드는 업무 | 독자적인 추천·가격 정책 |
| 지원 | 핵심 업무를 돕는 회사 고유 업무 | 주문·정산 운영 |
| 일반 | 여러 회사가 비슷하게 해결하는 업무 | 이메일 발송 |

핵심 서브도메인에는 업무 담당자와 함께 깊이 모델링할 시간과 인력을 투자할 수 있다. 일반 서브도메인은 검증된 제품을 사용하거나 단순하게 구현하는 편이 나을 수 있다. DDD를 적용할 깊이는 업무의 복잡성과 중요도에 맞춰 정한다.

DDD를 적용할 영역을 골랐다면 작은 업무 흐름 하나를 출발점으로 삼는 편이 좋다.

## 실제 프로젝트에서는 작게 시작한다

지금까지 살펴본 흐름을 주문 취소 기능 하나에 적용하면 다음과 같다.

1. 업무 담당자와 주문 취소의 실제 흐름을 대화한다.
2. 취소, 환불과 출고 중단처럼 서로 다른 용어와 사건을 정리한다.
3. 주문, 결제와 배송 모델이 통하는 경계를 찾는다.
4. 주문 Aggregate와 취소 규칙을 코드로 표현한다.
5. 컨텍스트 사이에 필요한 요청과 이벤트의 계약을 정한다.
6. 구현하며 발견한 예외와 규칙을 다시 모델에 반영한다.

주문 취소처럼 경계가 분명한 Use Case 하나를 구현하고, 발견한 예외를 대화와 모델에 다시 반영하며 범위를 넓혀 간다. 이 반복 속에서 모델이 구체화된다.

## 정리

주문 취소 기능의 문제는 코드보다 앞에서 시작됐다. `취소`라는 말을 팀마다 다르게 이해했고, 주문과 결제, 배송 중 어디에서 무엇을 판단할지도 정리되지 않았다.

DDD는 도메인 전문가와 개발자가 업무를 함께 이해하는 데서 출발한다. 같은 언어와 모델이 통하는 경계를 정하고, 경계 안에서는 중요한 상태와 행동, 규칙을 코드로 표현한다. 다른 컨텍스트와 협력할 때는 서로 합의한 계약을 사용한다.

마지막으로 각 개념이 어떤 질문에 답하는지 정리해 보자.

### 업무와 경계를 살펴볼 때

| 개념 | 답하려는 질문 |
|---|---|
| Domain | 어떤 현실의 업무 문제를 해결하는가? |
| Subdomain | 큰 업무를 어떤 작은 문제 영역으로 나눌까? |
| Domain Model | 그 업무에서 어떤 대상·상태·행동·규칙이 중요한가? |
| Ubiquitous Language | 같은 말을 같은 뜻으로 사용하고 있는가? |
| Bounded Context | 이 모델과 언어는 어디까지 같은 의미인가? |
| Context Map | 컨텍스트들은 어떤 관계와 계약으로 협력하는가? |

### 경계 안의 모델을 만들 때

| 개념 | 답하려는 질문 |
|---|---|
| Entity | 식별자로 계속 추적해야 하는 대상인가? |
| Value Object | 값 자체로 의미가 결정되는가? |
| Aggregate | 어떤 상태와 규칙을 Root가 일관되게 책임질까? |
| Domain Service | 한 객체에 귀속시키기 어려운 업무 규칙인가? |

### 기능을 실행하고 다른 컨텍스트와 협력할 때

| 개념 | 답하려는 질문 |
|---|---|
| Application Service | 도메인 모델을 이용해 한 작업을 어떤 순서로 진행할까? |
| Repository | Aggregate를 어떻게 저장하고 복원할까? |
| Domain Event | 도메인에서 어떤 의미 있는 일이 이미 일어났는가? |
| Integration Event | 그 사실을 다른 컨텍스트에 어떤 계약으로 알릴까? |
| Anti-Corruption Layer | 외부 모델을 내 컨텍스트의 언어로 어떻게 바꿀까? |

DDD는 복잡한 업무를 함께 이해하고, 그 이해가 모델과 코드에서도 같은 의미로 이어지게 만든다. 위 개념들은 그 과정을 표현하는 수단이다.

실제 요구사항을 만났을 때 표에 적은 질문을 던지는 것부터 시작해 보자. 그 답을 찾는 과정이 업무 이해를 모델과 코드로 이어 가는 출발점이다.

## 참고 자료

### 공식 자료

- [Eric Evans - Domain-Driven Design Reference](https://www.domainlanguage.com/wp-content/uploads/2016/05/DDD_Reference_2015-03.pdf)
- [Microsoft Learn - Use domain analysis to model microservices](https://learn.microsoft.com/en-us/azure/architecture/microservices/model/domain-analysis)
- [Microsoft Learn - Anti-Corruption Layer pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/anti-corruption-layer)
- [Microsoft Learn - Use Tactical DDD to Design Microservices](https://learn.microsoft.com/en-us/azure/architecture/microservices/model/tactical-ddd)
- [Microsoft Learn - DDD 지향 마이크로 서비스 디자인](https://learn.microsoft.com/ko-kr/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/ddd-oriented-microservice)
- [Microsoft Learn - Designing a microservice domain model](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/microservice-domain-model)
- [Microsoft Learn - Domain events: Design and implementation](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/domain-events-design-implementation)
- [Microsoft Learn - CQRS pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [EventStorming - 공식 사이트](https://www.eventstorming.com/)

### 추가 자료

- [Martin Fowler - Domain-Driven Design](https://martinfowler.com/bliki/DomainDrivenDesign.html)
- [Martin Fowler - Ubiquitous Language](https://martinfowler.com/bliki/UbiquitousLanguage.html)
- [Martin Fowler - Bounded Context](https://martinfowler.com/bliki/BoundedContext.html)
- [Martin Fowler - DDD Aggregate](https://martinfowler.com/bliki/DDD_Aggregate.html)
- [ioob.dev - DDD 3편: Context Mapping](https://ioob.dev/posts/ddd-3-context-mapping/)
- [ASSU BLOG - DDD 바운디드 컨텍스트 연동](https://assu10.github.io/dev/2024/08/24/ddd-bounded-context-linkage/)
- [카카오페이 기술 블로그 - 여신코어 DDD로 구축하기](https://tech.kakaopay.com/post/backend-domain-driven-design/)
- [LY Corporation Tech Blog - DDD를 Merchant 시스템 구축에 활용한 사례](https://techblog.lycorp.co.jp/ko/applying-ddd-to-merchant-system-development)
