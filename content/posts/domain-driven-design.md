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

세 문장을 코드로 바로 옮기기 전에 무엇을 표현해야 하는지 나누어 보자. 주문과 주문 상품은 계속 상태가 바뀌는 **대상**이고, 주문 상태는 그 대상이 가진 **값**이다. 취소는 주문에 일어나는 **행동**이며, 배송 시작 이후에는 허용하지 않는다는 **규칙**이 붙는다.

이 내용은 이후 `Order`, `OrderLine`, `OrderStatus`와 `Order.cancel()` 같은 코드로 표현할 수 있다. 어떤 대상을 중요하게 보고 어떤 규칙을 담을지는 실제 업무를 아는 사람과 개발자가 함께 정한다.

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

먼저 업무 담당자와 개발자가 `취소가 정확히 무엇인지` 합의해야 한다. `cancel()`은 그 합의가 코드에 드러난 결과다. 반대로 코드에 `changeStatus(4)`만 남겨 두면 개발자는 숫자 `4`의 의미를 다시 찾아야 한다. 함께 정한 언어가 문서에만 머물지 않고 코드까지 이어져야 하는 이유다.

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

대화에서 확인한 사건을 시간 순서로 놓으면 다음과 같다.

1. 주문 취소가 요청되었다.
2. 주문이 취소되었다.
3. 환불이 요청되었다.
4. 출고 중단이 요청되었다.
5. 쿠폰과 재고가 복구되었다.

이미 상품을 집기 시작했다면 두 번째 사건으로 넘어가지 못한다. 같은 `배송 전`이라는 말도 팀마다 가리키는 시점이 달랐고, 그 차이가 취소 가능 여부를 바꾸고 있었다.

사건은 이미 일어난 결과를 나타낸다. `주문이 취소되었다` 앞에는 주문 상태를 확인해 취소 여부를 판단하는 과정이 있다. 이 과정을 거꾸로 따라가면 상태를 가진 대상은 **주문**, 주문이 제공해야 할 행동은 **취소**라는 사실이 보인다.

여기에 `상품을 집기 시작한 주문은 취소할 수 없다`는 규칙을 붙이면 업무에서 말하던 취소가 `Order.cancel()`이라는 모델의 행동으로 이어진다. 이벤트 스토밍은 이처럼 여러 사람이 알고 있던 사건과 규칙을 하나의 흐름으로 펼쳐, 모델에 담을 후보를 찾는 방법이다.

![EventStorming을 활용해 업무의 사건과 경계를 함께 탐색하는 모습](/images/posts/domain-driven-design/event-storming.jpg "EventStorming을 활용해 업무의 사건과 경계를 함께 탐색하는 모습. 출처: EventStorming 공식 사이트")

대화 중에 새로운 사건이나 예외가 나오면 모델에 반영하고, 구현하면서 새로 알게 된 내용도 다시 대화로 가져온다. 이 과정을 거치면 주문팀 안에서는 같은 말로 같은 모델을 가리킬 수 있다.

하지만 회사 전체가 모든 단어를 같은 뜻으로 사용하기는 어렵다. 상품팀과 주문팀이 말하는 `상품`부터 필요한 정보가 다르다. 이제 하나의 모델이 어디까지 같은 의미로 통하는지 경계를 정해야 한다.

## 하나의 모델이 통하는 경계를 찾는다

쇼핑몰이라는 큰 업무를 한 번에 모델링하기는 어렵다. 그래서 주문, 결제와 배송처럼 비교적 작은 문제 영역으로 나누어 살펴본다. 이렇게 나눈 각각의 업무 영역을 **서브도메인**이라고 한다.

업무를 나눈 뒤에는 소프트웨어 안에서 각 모델이 어디까지 같은 의미로 통할지도 정해야 한다. 이 경계가 **바운디드 컨텍스트**다.

| 구분 | 경계를 나누는 기준 |
|---|---|
| 서브도메인 | 현실의 큰 업무를 나눈 문제 영역 |
| 바운디드 컨텍스트 | 모델과 언어가 같은 의미로 통하는 범위 |

하나의 주문 컨텍스트가 주문 서브도메인 전체를 맡을 수도 있고, 업무가 커지면 주문 접수와 주문 이력을 서로 다른 컨텍스트로 나눌 수도 있다. 서브도메인은 현실의 문제를 나누고, 바운디드 컨텍스트는 그 문제를 표현할 모델의 범위를 정한다. 둘의 대응 관계는 업무와 모델의 경계에 따라 달라진다.

상품팀과 주문팀이 사용하는 `상품`을 예로 들어 보자. 상품팀에는 현재 판매 가격, 상품 설명과 선택 가능한 옵션이 중요하다. 주문팀에는 고객이 주문한 당시의 이름, 가격과 수량이 필요하다. 상품팀이 오늘 가격을 바꾸더라도 어제 결제한 주문 금액은 그대로 남아야 한다.

따라서 상품 컨텍스트의 `Product`와 주문 컨텍스트의 주문 상품은 이름이 비슷해도 서로 다른 모델로 둘 수 있다. 같은 현실의 상품을 다루더라도 풀려는 문제가 다르면 필요한 정보와 규칙도 달라지기 때문이다.

주문 취소도 마찬가지다. 주문 컨텍스트는 주문 상태를 바꾸고, 결제 컨텍스트는 환불 가능 여부와 금액을 계산하며, 배송 컨텍스트는 출고를 멈출 수 있는지 판단한다. 각 컨텍스트는 자신이 맡은 규칙을 판단하고 다른 컨텍스트에는 필요한 요청이나 사건만 전달한다.

이 경계는 `order`, `payment` 같은 패키지나 Gradle 모듈로 드러낼 수 있고, 필요하다면 독립 서비스로 분리할 수도 있다. 하나의 모놀리식 애플리케이션 안에도 여러 바운디드 컨텍스트를 둘 수 있다. 패키지, 모듈과 서비스는 경계를 코드로 표현하는 방법이며, 경계를 찾을 때는 구현 단위보다 모델의 의미가 달라지는 지점을 먼저 살핀다.

바운디드 컨텍스트 하나에도 서로 다른 규칙을 맡는 객체 묶음이 여러 개 들어갈 수 있다. 뒤에서 살펴볼 **Aggregate**는 컨텍스트 안에서 상태 변경의 일관성을 관리하는 더 작은 경계다.

![주문 컨텍스트 안의 여러 애그리거트와 상품 컨텍스트의 관계](/images/posts/domain-driven-design/context-and-aggregate.svg "바운디드 컨텍스트와 애그리거트 경계의 차이")

그림의 주문 컨텍스트에는 Order Aggregate와 Coupon Aggregate가 있고, 각자 맡은 규칙을 지킨다. 주문 컨텍스트는 상품 컨텍스트의 `Product` 대신, 두 컨텍스트가 합의한 형식으로 주문에 필요한 정보만 받는다.

지금까지처럼 큰 업무와 모델의 경계를 살펴보는 관점을 DDD의 **전략적 설계**라고 부른다. 이제 카메라를 컨텍스트 안쪽으로 옮겨 Entity, Value Object와 Aggregate로 대상과 값, 변경 규칙을 표현해 보자. 이 관점이 **전술적 설계**다.

전략적 설계와 전술적 설계의 관계를 정리하면 다음과 같다.

![전략적 설계는 업무와 모델의 큰 경계를 정하고, 전술적 설계는 경계 안의 객체와 규칙을 코드로 표현한다](/images/posts/domain-driven-design/strategic-tactical-design.svg "DDD의 전략적 설계와 전술적 설계")

## 경계 안의 모델을 코드로 표현한다

앞에서 주문 업무를 살펴보며 주문과 주문 상품이라는 대상, 주문 상태라는 값, 배송 이후에는 취소할 수 없다는 규칙을 찾았다. 이제 이 내용을 주문 컨텍스트의 객체로 옮겨 보자.

처음에는 다음처럼 상태를 직접 바꾸는 코드를 작성할 수 있다.

```java
order.setStatus(OrderStatus.CANCELED);
```

상태를 직접 바꾸면 취소할 수 있는 조건이 코드에 드러나지 않는다. 이 메서드를 호출하는 모든 곳에서 취소 규칙을 따로 확인해야 한다.

취소 규칙을 `Order` 안에 담으려면 주문 모델을 이루는 요소부터 구분해야 한다.

주문처럼 시간이 지나도 같은 대상으로 추적해야 하는 것이 있고, 금액과 주소처럼 값 자체로 의미를 표현하는 것도 있다. 각각 Entity와 Value Object에 해당한다.

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

    public Money totalPrice() {
        return totalPrice;
    }
}
```

외부에서는 `cancel()`을 호출해 주문 취소를 요청한다. 메서드 안에는 취소 조건과 상태 변경이 함께 있어, 코드만 읽어도 주문이 어떤 규칙에 따라 취소되는지 알 수 있다.

Spring에서 자주 접하는 JPA `@Entity`와 DDD의 Entity는 구분할 필요가 있다.

| 구분 | 의미 |
|---|---|
| DDD Entity | 식별성을 중심으로 같은 대상을 추적하는 도메인 모델 개념 |
| JPA `@Entity` | 객체를 관계형 데이터베이스에 매핑하는 ORM 기술 |

하나의 `Order` 클래스가 두 역할을 함께 맡을 수 있다. JPA `@Entity`는 이 객체를 데이터베이스에 저장하는 방법을 나타낸다. DDD Entity는 업무에서 같은 주문을 어떻게 식별하고 추적할지 설명한다.

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

다음 그림에서는 `OrderLine`과 `ShippingAddress`를 바꾸는 요청이 모두 Root인 `Order`를 거친다. 변경 경로를 한곳에 모아 수량 검증과 총금액 계산 같은 주문 규칙을 지킨다.

![Order가 Aggregate Root가 되어 주문 상품과 배송지 변경을 통제하는 구조](/images/posts/domain-driven-design/order-aggregate.svg "주문 Aggregate의 구조")

> **Aggregate의 경계는 한 트랜잭션에서 함께 지켜야 할 규칙을 기준으로 찾는다.**
>
> 주문 상품의 수량을 바꿀 때 총금액도 함께 맞춰야 한다면 두 상태는 같은 경계 안에서 관리한다.

한 작업이 끝났을 때 경계 안의 상태가 업무 규칙을 만족해야 한다. Aggregate가 지키는 **일관성**이란 이런 상태를 말한다. Entity와 Value Object가 객체의 성격을 설명한다면, Aggregate는 여러 객체의 변경을 어디까지 함께 관리할지 정한다. Aggregate Root는 Entity이며, 경계 안에는 Root를 통해 관리되는 다른 Entity가 함께 들어갈 수 있다.

Aggregate가 너무 크면 작은 변경에도 많은 데이터를 불러오거나 잠가야 할 수 있다. 반대로 너무 작게 나누면 한 번에 지켜야 할 규칙이 여러 Aggregate에 흩어진다. 한 Aggregate의 변경은 하나의 트랜잭션에서 완료하는 것을 기본으로 두고, 여러 Aggregate의 변경이 필요하다면 각각의 트랜잭션과 이벤트를 통한 후속 처리로 나눌 수 있는지 검토한다.

Aggregate를 만들면 한 객체 묶음 안에서 지켜야 할 규칙을 표현할 수 있다. 하지만 기능 하나를 완성하려면 한 객체에 담기 어려운 규칙도 계산하고, 필요한 모델을 불러와 실행한 뒤 저장해야 한다.

## 도메인 모델로 기능을 완성한다

기능을 완성하는 과정에서는 Domain Service, Application Service와 Repository가 서로 다른 일을 맡는다. 이름은 비슷해도 Domain Service는 업무 규칙을 표현하고, Application Service는 그 규칙을 사용해 하나의 작업을 진행한다.

### Domain Service는 한 객체가 맡기 어려운 규칙을 다룬다

주문 취소 조건은 `Order` 안에 자연스럽게 들어간다. 금액 계산도 `Money`가 맡을 수 있다. 먼저 규칙을 책임질 Entity나 Value Object가 있는지 살펴보는 이유다.

할인처럼 여러 모델의 정보를 함께 판단해야 하는 규칙은 한 객체에 넣기 어려울 수 있다. 이 규칙 자체가 독립적인 업무 개념이라면 **Domain Service**로 표현할 수 있다.

회원 등급과 주문을 같은 판매 정책 컨텍스트에서 다룬다고 가정해 보자. 두 정보를 함께 살펴 할인액을 계산하는 정책은 다음과 같다.

```java
public class DiscountPolicy {

    public Money calculate(Member member, Order order) {
        if (member.isVip()) {
            return order.totalPrice()
                .multiply(new BigDecimal("0.10"));
        }

        return Money.ZERO;
    }
}
```

`DiscountPolicy`는 회원 등급과 주문 금액을 바탕으로 할인액을 계산한다. 다만 어느 회원과 주문을 불러오고, 계산 결과를 어디에 저장할지까지 결정하지는 않는다. 그 전체 흐름은 Application Service가 맡는다.

### Application Service는 작업의 흐름을 조정한다

주문 취소 사례로 돌아가 보자. `Order`는 취소 가능 여부를 판단할 수 있지만, 기능을 실행하려면 요청받은 주문을 불러오고 `cancel()`을 호출한 뒤 변경 결과를 저장해야 한다.

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

`CancelOrderService`는 주문 취소라는 사용 사례를 처음부터 끝까지 진행한다. 반면 배송을 시작한 주문을 취소할 수 있는지는 `Order.cancel()`이 판단한다. HTTP API뿐 아니라 관리자 기능과 배치에서도 똑같이 지켜야 하는 주문의 규칙이기 때문이다.

Spring 애플리케이션의 각 영역에 방금 살펴본 역할을 놓으면 다음과 같다.

| 영역 | 맡는 일 |
|---|---|
| Presentation | 외부 요청과 응답을 애플리케이션에 연결 |
| Application Service | 도메인 모델을 준비하고 호출해 하나의 사용 사례와 트랜잭션을 완성 |
| Domain Model | 업무적으로 무엇이 가능한지 판단 |
| Infrastructure | DB, 메시지와 외부 API를 구체적인 기술로 연결 |

할인 적용 기능에서도 같은 구분을 사용할 수 있다.

| 구분 | 할인 적용에서 맡는 일 |
|---|---|
| Application Service | `Order`와 `Member` 조회 → 할인 계산 요청 → 결과 저장 |
| Domain Service | 회원 등급과 주문 금액을 바탕으로 할인액 계산 |

### Repository는 Aggregate를 저장하고 복원한다

Application Service가 작업을 진행하려면 Aggregate를 불러오고 변경 결과를 저장할 통로가 필요하다. 이 역할을 **Repository**가 맡는다.

```java
public interface OrderRepository {

    Optional<Order> findById(OrderId orderId);

    void save(Order order);
}
```

수량 변경처럼 주문 전체의 규칙을 확인해야 하는 작업에서는 `orders`와 `order_lines`가 서로 다른 테이블에 있더라도 `Order` Aggregate를 복원한다. 변경도 Root를 거쳐야 수량과 총금액을 함께 관리할 수 있다.

```java
// 내부 객체만 바꾸면 주문 규칙을 우회할 수 있다.
OrderLine line = orderLineRepository.findById(lineId)
    .orElseThrow();
line.changeQuantity(quantity);

// Root를 통해 바꾸면 수량과 총금액을 함께 관리한다.
Order order = orderRepository.findById(orderId)
    .orElseThrow(OrderNotFoundException::new);

order.changeQuantity(lineId, quantity);
orderRepository.save(order);
```

#### 저장 기술로 구현할 때

이 예제의 Application Service는 도메인에 정의한 `OrderRepository` 인터페이스에 의존한다. Infrastructure 영역은 Spring Data JPA나 `EntityManager`로 이 계약을 구현한다.

JPA Entity가 영속 상태라면 변경 사항은 Dirty Checking으로 반영될 수 있다. 예제의 `save()`는 `Aggregate를 불러오고 변경한 뒤 저장한다`는 흐름을 보여주기 위해 명시했다.

Aggregate를 저장하는 경로를 정했다면, 값을 읽기만 하는 기능도 같은 경로를 따라야 하는지 살펴볼 수 있다.

### 조회는 필요한 형태로 가져온다

지금까지 살펴본 주문 취소는 상태를 바꾸는 기능이다. 업무 규칙을 지켜야 하므로 Application Service가 `Order` Aggregate를 불러와 변경한다.

주문 목록처럼 값을 읽기만 하는 기능은 목적이 다르다. Aggregate 전체를 복원하지 않고 조회 전용 DAO나 Projection으로 화면에 필요한 값만 가져올 수 있다.

| 작업 | 처리 흐름 |
|---|---|
| 변경 | `Command → Application Service → Aggregate → Repository` |
| 조회 | `Query → DAO / Projection → Response` |

이처럼 변경과 조회의 책임을 나누는 구조를 **CQRS**라고 한다. Command Query Responsibility Segregation, 즉 명령과 조회 책임 분리를 뜻한다.

가장 단순한 형태는 하나의 애플리케이션과 데이터베이스를 유지한 채 코드의 경로만 나누는 방식이다. CQRS는 필요에 따라 함께 사용할 수 있는 선택지이며 DDD의 필수 요소는 아니다.

이제 주문 컨텍스트 안에서는 규칙을 판단하고, 기능을 실행하고, 상태를 저장할 수 있다. 주문 취소 뒤에 필요한 환불과 출고 중단은 결제와 배송 컨텍스트의 몫이다.

## 컨텍스트는 계약으로 협력한다

주문, 결제와 배송은 경계를 나눈 뒤에도 계속 협력한다. 각 컨텍스트는 내부 모델을 독립적으로 유지하고, 서로 주고받을 정보와 형식을 계약으로 정한다. 다음처럼 주문 컨텍스트가 상품 컨텍스트의 내부 모델을 직접 참조하면 두 코드가 함께 묶인다.

```java
import catalog.domain.Product;

public class OrderLine {
    private Product product;
}
```

상품 컨텍스트는 내부 `Product` 대신 외부에 공개할 계약을 제공한다. 주문 컨텍스트는 이 값을 주문 당시의 상품 정보로 바꿔 보관한다.

```java
// 상품 컨텍스트가 공개한 계약
public record ProductInfo(
    long productId,
    String name,
    BigDecimal price
) {
}

// 주문 컨텍스트의 모델
public record OrderedProduct(
    long productId,
    String name,
    Money orderPrice
) {
    public static OrderedProduct from(ProductInfo source) {
        return new OrderedProduct(
            source.productId(),
            source.name(),
            new Money(source.price())
        );
    }
}
```

상품의 현재 가격과 구조가 바뀌어도 이미 생성된 주문은 `OrderedProduct`에 저장한 이름과 주문 가격을 유지한다.

### Command, Domain Event와 Integration Event를 구분한다

상품 정보를 조회하듯 데이터를 주고받을 수도 있다. 주문 취소처럼 상태를 바꾸고 후속 처리를 이어 가야 할 때는 메시지를 사용할 수 있다. 메시지에서는 `해 달라는 요청`과 `이미 일어난 사실`을 구분한다. 전자가 **Command**, 후자가 **Domain Event**다.

- **Command — `CancelOrder`**: 주문을 취소해 달라는 요청

- **Domain Event — `OrderCanceled`**: 주문이 이미 취소되었다는 사실

- **Integration Event**: 주문 취소 사실을 다른 컨텍스트에 공개하는 메시지

주문이 취소되었다는 사실을 코드로 표현하면 다음과 같다.

```java
public record OrderCanceled(
    OrderId orderId,
    Instant occurredAt
) {
}
```

#### 같은 애플리케이션 안에서 전달한다면

Application Service는 주문 상태를 바꾼 뒤 `OrderCanceled`를 만들어 발행할 수 있다. 같은 애플리케이션 안에서 전달한다면 Spring의 `ApplicationEventPublisher`를 사용할 수 있다.

```java
order.cancel();

eventPublisher.publishEvent(
    new OrderCanceled(order.id(), Instant.now())
);

@TransactionalEventListener(
    phase = TransactionPhase.AFTER_COMMIT
)
public void handle(OrderCanceled event) {
    notificationService.sendCancelNotice(event.orderId());
}
```

일반 `@EventListener`는 기본 설정에서 동기로 실행된다. 트랜잭션이 커밋된 뒤 후속 작업을 시작하려면 위와 같이 `@TransactionalEventListener`의 실행 시점을 `AFTER_COMMIT`으로 정할 수 있다.

#### 다른 컨텍스트에 전달한다면

다른 컨텍스트에도 알려야 한다면 `OrderCanceled`를 외부 계약인 `OrderCanceledIntegrationEvent`로 바꿔 Kafka 같은 메시지 브로커로 전달할 수 있다.

![Spring Application Event로 같은 애플리케이션 안에서 Domain Event를 전달하고, Integration Event는 Kafka를 거쳐 다른 컨텍스트로 전달하는 흐름](/images/posts/domain-driven-design/domain-event-flow.svg "Spring Application Event와 Integration Event의 전달 범위")

결제와 쿠폰 컨텍스트는 이 메시지를 받은 뒤 각자의 규칙에 따라 환불과 쿠폰 복구를 처리한다. 외부 메시지는 주문 트랜잭션이 성공한 뒤 전달되어야 하며, 실제 구현에서는 발행 실패와 재시도, 중복 처리, DB 트랜잭션과 메시지 사이의 정합성을 함께 설계한다.

컨텍스트는 메시지뿐 아니라 API로도 협력한다. 응답을 직접 주고받을 때는 외부 모델이 내부로 그대로 퍼지지 않도록 경계에서 변환할 수 있다.

### 외부 모델을 내 언어로 번역한다

결제 컨텍스트가 반환한 `PaymentResponse`를 주문 코드까지 그대로 전달하면 주문도 결제의 이름과 응답 구조를 알아야 한다. 경계에 Adapter를 두면 외부 응답을 주문 컨텍스트의 `RefundResult`로 바꿀 수 있다.

```java
@Component
@RequiredArgsConstructor
public class PaymentAdapter {

    private final PaymentApi paymentApi;

    public RefundResult refund(
        OrderId orderId,
        Money amount
    ) {
        PaymentResponse response = paymentApi.refund(
            orderId.value(),
            amount.amount()
        );

        return new RefundResult(
            response.paymentCode(),
            new Money(response.cancelAmount())
        );
    }
}
```

결제 응답 형식이 바뀌면 `PaymentAdapter`의 변환 코드가 영향을 받는다. 주문 모델은 `RefundResult`라는 자신의 언어를 유지한다. 이처럼 외부 모델을 내 컨텍스트의 언어로 번역하는 경계를 **부패 방지 계층**이라고 한다. 영어로는 Anti-Corruption Layer, 줄여서 ACL이라고 부른다.

컨텍스트가 많아지면 개별 계약뿐 아니라 전체 관계도 함께 볼 필요가 있다.

### Context Map은 컨텍스트 사이의 관계를 보여준다

여러 컨텍스트가 어떤 관계로 연결되고 어떤 계약을 주고받는지 정리한 그림을 **Context Map**이라고 한다. 상품 컨텍스트가 주문에 상품 정보를 제공하고, 주문 컨텍스트가 결제와 쿠폰에 취소 결과를 전달하는 관계를 다음처럼 표현할 수 있다.

![상품, 주문, 결제와 쿠폰 컨텍스트가 API 계약, ACL과 Integration Event로 협력하는 Context Map](/images/posts/domain-driven-design/context-map-example.svg "주문 기능을 중심으로 정리한 Context Map의 예")

Context Map을 보면 어느 컨텍스트가 정보를 제공하고, 경계에서 어떤 변환이 필요하며, 변경의 영향이 어디까지 이어지는지 확인할 수 있다.

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

복잡한 주문 정책에는 도메인 모델과 Aggregate를 사용하고, 단순한 목록 조회에는 Projection을 사용할 수 있다. 컨텍스트 사이의 결합을 줄여야 할 때는 Event를 검토한다.

어디에 DDD가 필요한지 골랐다면 처음부터 완성된 모델을 만들려고 하기보다, 작은 업무 흐름 하나에서 시작하는 편이 좋다.

## 실제 프로젝트에서는 작게 시작한다

지금까지 살펴본 흐름을 주문 취소 기능 하나에 적용하면 다음과 같다.

1. 업무 담당자와 주문 취소의 실제 흐름을 대화한다.
2. 취소, 환불과 출고 중단처럼 서로 다른 용어와 사건을 정리한다.
3. 주문, 결제와 배송 모델이 통하는 경계를 찾는다.
4. 주문 Aggregate와 취소 규칙을 코드로 표현한다.
5. 컨텍스트 사이에 필요한 요청과 이벤트의 계약을 정한다.
6. 구현하며 발견한 예외와 규칙을 다시 모델에 반영한다.

이 과정은 한 번에 끝나지 않는다. 구현에서 발견한 내용을 다시 대화로 가져오면서 업무에 대한 이해와 모델을 함께 다듬는다. 처음에는 복잡한 업무 하나를 골라 이 순서를 반복해 보는 것만으로도 충분하다.

## 정리

주문 취소 기능의 문제는 코드보다 앞에서 시작됐다. `취소`라는 말을 팀마다 다르게 이해했고, 주문과 결제, 배송 중 어디에서 무엇을 판단할지도 정리되지 않았다.

DDD는 도메인 전문가와 개발자가 업무를 함께 이해하는 데서 출발한다. 같은 언어와 모델이 통하는 경계를 정하고, 경계 안에서는 중요한 상태와 행동, 규칙을 코드로 표현한다. 다른 컨텍스트와 협력할 때는 서로 합의한 계약을 사용한다.

개념을 다시 정의하기보다, 각 개념이 어떤 질문에 답하는지 정리해 보자.

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

실제 요구사항을 만났을 때 표에 적은 질문을 던지는 것부터 시작해 보자.

## 참고 자료

### 공식 자료

- [Eric Evans - Domain-Driven Design Reference](https://www.domainlanguage.com/wp-content/uploads/2016/05/DDD_Reference_2015-03.pdf)
- [Microsoft Learn - DDD 지향 마이크로 서비스 디자인](https://learn.microsoft.com/ko-kr/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/ddd-oriented-microservice)
- [Microsoft Learn - Designing a microservice domain model](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/microservice-domain-model)
- [Microsoft Learn - Designing the infrastructure persistence layer](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-design)
- [Microsoft Learn - Domain events: Design and implementation](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/domain-events-design-implementation)
- [Microsoft Learn - CQRS pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [Spring Framework - Application Events](https://docs.spring.io/spring-framework/reference/core/beans/context-introduction.html#context-functionality-events)
- [Spring Framework - Transaction-bound Events](https://docs.spring.io/spring-framework/reference/data-access/transaction/event.html)
- [Hibernate ORM User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)
- [EventStorming - 공식 사이트](https://www.eventstorming.com/)

### 추가 자료

- [Martin Fowler - Domain-Driven Design](https://martinfowler.com/bliki/DomainDrivenDesign.html)
- [Martin Fowler - Ubiquitous Language](https://martinfowler.com/bliki/UbiquitousLanguage.html)
- [Martin Fowler - Bounded Context](https://martinfowler.com/bliki/BoundedContext.html)
- [Martin Fowler - DDD Aggregate](https://martinfowler.com/bliki/DDD_Aggregate.html)
- [카카오페이 기술 블로그 - 여신코어 DDD로 구축하기](https://tech.kakaopay.com/post/backend-domain-driven-design/)
- [LY Corporation Tech Blog - DDD를 Merchant 시스템 구축에 활용한 사례](https://techblog.lycorp.co.jp/ko/applying-ddd-to-merchant-system-development)
