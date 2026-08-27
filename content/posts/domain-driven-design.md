+++
title = 'DDD, 도대체 도메인이 뭔데?'
slug = '15'
date = 2026-08-26T22:10:00+09:00
lastmod = 2026-08-27T21:20:00+09:00
draft = false
references_required = true
description = '주문 취소 사례를 따라가며 업무 이해가 도메인 모델, 바운디드 컨텍스트, Entity·Value Object·Aggregate와 코드로 이어지는 과정을 살펴봅니다.'
categories = ['애플리케이션 아키텍처']
tags = ['DDD', '도메인 모델', '바운디드 컨텍스트', '애그리거트', '도메인 이벤트']
+++

세상에는 개발 방법론이 참 많다. 기술 하나를 익혀 기능을 만드는 것도 벅찬데, TDD와 DDD처럼 이름까지 비슷한 개념이 계속 등장한다.

오늘 살펴볼 DDD는 복잡한 업무를 여러 사람이 같은 기준으로 이해하고, 그 이해를 코드에 담는 접근이다. 우리말로는 도메인 주도 설계라고 한다.

이름은 어렵지만 출발점은 현실적이다. 같은 업무를 서로 다르게 이해하면 각자가 만든 코드도 조금씩 어긋난다.

## 같은 주문 취소를 다르게 이해할 때

정의부터 외우기 전에 DDD가 필요한 상황부터 살펴보자. 주문팀, 결제팀, 물류팀과 고객센터가 따로 있는 쇼핑몰 회사를 가정해 보겠다.

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

DDD는 이런 혼선을 줄이기 위해 업무를 이해하고, 그 이해를 모델과 코드에 담아 가는 접근이다. 사람의 설명과 코드가 같은 업무를 가리키게 만드는 것이 출발점이다.

이 글에서는 다음 흐름을 따라간다. 용어를 따로 외우기보다 앞에서 발견한 문제가 왜 다음 개념을 필요로 하는지 보면 된다.

![업무 이해에서 모델과 경계를 거쳐 실행과 협력으로 이어지는 DDD의 흐름](/images/posts/domain-driven-design/ddd-concept-map.svg "DDD를 따라가는 네 단계")

## 업무를 이해해 모델을 만든다

**도메인**은 소프트웨어로 해결하려는 현실의 업무 영역이다. 쇼핑몰이라면 주문, 결제, 배송과 재고가 도메인에 해당한다.

이 업무를 이해하려면 주문이 어떤 상태를 가지는지, 언제 취소할 수 있는지, 취소하면 어떤 처리가 이어지는지 알아야 한다. 여기에서 중요한 개념과 행동, 규칙을 추려 표현한 것이 **도메인 모델**이다.

주문 업무를 살펴보면 다음과 같은 내용이 보인다.

1. 주문에는 여러 상품이 담긴다.
2. 주문은 현재 상태를 가진다.
3. 배송이 시작된 주문은 취소할 수 없다.

이 내용을 정리하면 `Order`, `OrderLine`, `OrderStatus` 같은 개념이 드러난다. 배송이 시작된 주문은 취소할 수 없다는 규칙은 `cancel()`이라는 행동에 담을 수 있다.

먼저 업무를 설명하는 모델을 만들고, 그다음 Java 클래스와 메서드로 옮긴다. 뒤에서 살펴볼 Entity, Value Object와 Aggregate는 이 모델을 코드로 표현할 때 사용하는 설계 개념이다.

### 도메인 전문가는 업무의 예외를 알고 있다

개발자가 코드만 보고 모든 업무 규칙을 알아내기는 어렵다. 실제 규칙과 예외를 잘 아는 사람에게 확인해야 한다. 이런 사람을 **도메인 전문가**라고 부른다.

도메인 전문가는 직함이 아니라 특정 업무를 깊이 이해하는 사람을 가리킨다. 주문 상태를 관리하는 운영자, 환불 규정을 정하는 기획자와 반복되는 문의를 처리하는 고객센터 직원 모두 자신이 맡은 업무의 전문가가 될 수 있다.

개발자는 이들과 함께 주문 취소의 흐름을 펼쳐 본다.

1. 고객이 주문 취소를 요청한다.
2. 주문이 취소 가능한 상태인지 확인한다.
3. 주문을 취소한다.
4. 결제에는 환불을, 물류에는 출고 중단을 요청한다.
5. 사용한 쿠폰과 재고를 복구한다.

목록만 보면 단순하지만 실제로 구현하려면 답해야 할 질문이 많다. 환불 요청이 실패하면 주문 취소도 되돌려야 할까? 이미 출고된 주문은 취소가 아니라 반품으로 처리해야 할까?

이 질문에 하나씩 답하면서 주문의 상태와 취소 조건이 구체화된다. 도메인 모델에 담아야 할 규칙도 이 과정에서 발견한다.

업무 규칙을 함께 정하려면 먼저 사용하는 말의 의미부터 맞아야 한다. 여기서 유비쿼터스 언어가 필요해진다.

## 같은 말을 같은 의미로 사용한다

주문팀의 `취소`, 결제팀의 `환불`, 물류팀의 `출고 중단`은 서로 연결되어 있지만 같은 행동은 아니다. 이 차이를 합의하지 않으면 회의에서는 통하는 것처럼 보여도 코드에서는 다시 어긋난다.

도메인 전문가와 개발자가 함께 정한 공통 언어를 **유비쿼터스 언어**라고 한다. 우리말로는 보편 언어라고도 부른다.

회의에서 주문 취소라고 부른 개념은 화면과 문서, 테스트와 코드에서도 주문 취소로 표현한다. 그래야 대화에서 합의한 규칙을 코드에서도 같은 의미로 찾을 수 있다.

```java
order.cancel();
refund.request();
shipment.stopDispatch();
```

유비쿼터스 언어의 범위는 메서드 이름보다 넓다. 먼저 업무 담당자와 개발자가 `취소가 정확히 무엇인지` 합의한다. `cancel()`이라는 이름은 그 합의가 코드에 드러난 결과다.

반대로 코드에 `changeStatus(4)`만 남아 있다면 숫자 `4`의 의미를 다시 사람의 기억에서 찾아야 한다. 용어집만 작성하고 코드에서는 다른 말을 사용하는 것도 같은 문제를 남긴다.

### 대화 속 사건을 시간 순서로 펼쳐 본다

업무 흐름과 용어를 찾는 방법 중 하나가 **이벤트 스토밍**이다. 업무에서 일어난 사건을 시간 순서대로 펼쳐 놓고 대화하면서 빠진 과정과 서로 다른 해석을 찾는다.

짧은 회의 장면을 생각해 보자.

> **진행자:** 주문이 만들어진 다음에는 무슨 일이 일어나나요?
>
> **기획자:** 재고를 확보합니다.
>
> **개발자:** 실제 수량을 바로 차감하나요?
>
> **운영자:** 결제 전에는 예약 재고로 잡습니다.
>
> **개발자:** 그러면 `재고가 예약되었다`는 사건이 따로 있겠네요.
>
> **기획자:** 네. 결제가 실패하면 예약 재고를 풀어야 합니다.

대화에서 확인한 사건을 시간 순서로 놓으면 다음과 같다.

1. 주문이 생성되었다.
2. 재고가 예약되었다.
3. 결제가 완료되었다.

결제에 실패했다면 세 번째 사건 대신 `예약 재고가 해제되었다`가 이어진다.

사건을 따라가면 그 일을 일으키는 대상과 지켜야 할 규칙도 함께 드러난다. `주문이 취소되었다`라는 사건에서는 `Order`라는 대상을 찾을 수 있다.

여기에 `배송을 시작한 주문은 취소할 수 없다`라는 규칙을 더하면 주문과 배송이 각각 무엇을 책임져야 하는지도 생각하게 된다.

이렇게 대화에서 발견한 개념과 규칙이 `Order.cancel()`과 `CancelOrderService` 같은 코드로 이어진다. 앞으로 살펴볼 Entity와 Aggregate는 이 내용을 객체와 경계로 표현하는 방법이다.

![EventStorming을 활용해 업무의 사건과 경계를 함께 탐색하는 모습](/images/posts/domain-driven-design/event-storming.jpg "EventStorming을 활용해 업무의 사건과 경계를 함께 탐색하는 모습. 출처: EventStorming 공식 사이트")

이벤트 스토밍의 핵심은 포스트잇 자체가 아니라 각자 알고 있던 업무를 한곳에 펼쳐 놓는 데 있다. 대화 중에 새로운 사건이나 예외가 나오면 모델에도 반영한다.

업무를 이해하고 모델을 만든 뒤에도 대화는 계속된다. 구현하면서 새로 알게 된 내용이 생기면 모델에 다시 반영한다.

이제 주문팀 안에서는 같은 언어로 대화할 수 있게 되었다. 그런데 회사 전체가 모든 단어를 같은 뜻으로 사용하기는 어렵다. 상품팀과 주문팀이 말하는 `상품`부터 의미가 다르다.

## 하나의 모델이 통하는 경계를 찾는다

쇼핑몰이라는 큰 업무를 한 번에 모델링하기는 어렵다. 그래서 주문, 결제와 배송처럼 비교적 작은 문제 영역으로 나누어 살펴본다. 이렇게 나눈 각각의 업무 영역을 **서브도메인**이라고 한다.

업무를 서브도메인으로 나누었다면, 이제 소프트웨어 안에서 각 모델이 어디까지 같은 의미로 통할지 정해야 한다. 이 경계를 **바운디드 컨텍스트**라고 한다.

| 구분 | 던지는 질문 | 다루는 대상 |
|---|---|---|
| 서브도메인 | 현실의 업무를 어떻게 나눌 것인가? | 문제 영역 |
| 바운디드 컨텍스트 | 그 업무를 표현하는 모델의 경계를 어디까지 둘 것인가? | 해결 영역 |

하나의 주문 컨텍스트가 주문 서브도메인 전체를 맡을 수도 있다. 업무가 커지면 주문 접수와 주문 이력을 서로 다른 컨텍스트로 나눌 수도 있다.

서브도메인은 현실의 문제를 나누고, 바운디드 컨텍스트는 그 문제를 표현할 모델의 범위를 정한다. 둘이 항상 일대일로 대응하는 것은 아니다.

상품팀과 주문팀이 사용하는 `상품`을 예로 들어 보자.

상품팀에는 현재 판매 가격, 상품 설명과 선택 가능한 옵션이 중요하다. 반면 주문팀에는 고객이 주문한 당시의 이름, 가격과 수량이 중요하다. 상품팀이 오늘 가격을 바꾸더라도 어제 결제한 주문 금액은 그대로 남아야 한다.

따라서 상품 컨텍스트의 `Product`와 주문 컨텍스트의 주문 상품은 이름이 비슷해도 서로 다른 모델로 둘 수 있다. 각 모델은 자신의 바운디드 컨텍스트 안에서 필요한 의미와 규칙을 가진다.

주문 취소도 마찬가지다. 주문 컨텍스트는 주문 상태를 바꾼다. 결제 컨텍스트는 환불 가능 여부와 금액을 계산하고, 배송 컨텍스트는 출고를 멈출 수 있는지 판단한다.

각 컨텍스트는 자신이 맡은 규칙을 판단하고, 다른 컨텍스트에는 필요한 요청이나 사건만 전달한다.

바운디드 컨텍스트는 **모델과 언어의 의미가 통하는 범위**를 정한다. 코드에서는 `order`, `payment` 같은 패키지나 Gradle 모듈로 경계를 드러낼 수 있다. 시스템의 필요에 따라 독립 서비스로 분리할 수도 있다.

다음 그림은 큰 컨텍스트 경계와 그 안에 있는 작은 Aggregate 경계를 나누어 보여준다.

![주문 컨텍스트 안의 여러 애그리거트와 상품 컨텍스트의 관계](/images/posts/domain-driven-design/context-and-aggregate.svg "바운디드 컨텍스트와 애그리거트 경계의 차이")

하나의 바운디드 컨텍스트 안에도 여러 Aggregate가 들어갈 수 있다. 그림의 주문 컨텍스트에는 Order Aggregate와 Coupon Aggregate가 있고, 각자 맡은 규칙을 지킨다.

상품 컨텍스트의 Product는 주문 모델 안으로 직접 들어오지 않는다. 두 컨텍스트가 합의한 형식으로 주문에 필요한 정보만 제공한다.

서브도메인은 회사에서 차지하는 중요도에 따라 핵심, 지원과 일반 서브도메인으로 구분하기도 한다.

| 구분 | 의미 | 쇼핑몰 예시 |
|---|---|---|
| 핵심 | 회사의 경쟁력을 만드는 업무 | 독자적인 추천·가격 정책 |
| 지원 | 핵심 업무를 돕는 회사 고유 업무 | 주문·정산 운영 |
| 일반 | 여러 회사가 비슷하게 해결하는 업무 | 이메일 발송 |

어느 서브도메인이 핵심인지는 구현 난이도가 아니라 회사의 경쟁력이 어디에서 나오는지에 따라 달라진다.

경쟁력을 만드는 업무에는 시간과 인력을 들여 깊이 모델링한다. 여러 회사가 비슷하게 처리하는 일반 업무는 검증된 제품을 사용하거나 단순하게 구현할 수 있다.

지금까지는 큰 업무를 나누고 모델의 경계를 찾았다. 이를 DDD의 **전략적 설계**라고 부른다.

경계를 정했다면 그 안의 상태와 규칙을 코드로 표현해야 한다. Entity, Value Object와 Aggregate 같은 도구를 사용하는 이 과정을 **전술적 설계**라고 한다.

## 경계 안의 모델을 코드로 표현한다

주문 컨텍스트의 모델에는 주문 번호, 주문 상태와 주문 상품이 필요하다. `배송이 시작된 주문은 취소할 수 없다`는 규칙도 코드에서 지켜야 한다.

처음에는 다음처럼 상태를 직접 바꾸는 코드를 작성할 수 있다.

```java
order.setStatus(OrderStatus.CANCELED);
```

상태를 직접 바꾸면 취소할 수 있는 조건이 코드에 드러나지 않는다. 이 메서드를 호출하는 모든 곳에서 취소 규칙을 따로 확인해야 한다.

이 규칙을 모델 안에 담으려면 먼저 업무의 대상과 값을 구분하고, 어느 객체가 변경을 책임질지 정해야 한다.

### Entity는 식별자로 이어진다

속성이 바뀌어도 같은 대상으로 계속 추적해야 하는 객체를 **Entity**라고 한다.

주문의 배송지와 상태는 바뀔 수 있다. 그래도 주문 번호가 같다면 어제 조회한 주문과 오늘 조회한 주문은 같은 주문이다. Entity는 이처럼 식별자를 기준으로 같은 대상을 이어서 추적한다.

```java
public class Order {

    private final OrderId id;
    private OrderStatus status;

    public void cancel() {
        if (!status.isCancelable()) {
            throw new CannotCancelOrderException();
        }

        status = OrderStatus.CANCELED;
    }
}
```

외부에서는 상태를 직접 바꾸는 대신 `cancel()`을 호출한다. 이제 `Order`가 주문을 취소하는 방법과 취소할 수 있는 조건을 함께 보여준다.

Spring에서 자주 접하는 JPA `@Entity`와 DDD의 Entity는 구분할 필요가 있다.

| 구분 | 의미 |
|---|---|
| DDD Entity | 식별성을 중심으로 같은 대상을 추적하는 도메인 모델 개념 |
| JPA `@Entity` | 객체를 관계형 데이터베이스에 매핑하는 ORM 기술 |

하나의 `Order` 클래스가 두 역할을 함께 맡을 수 있다. JPA `@Entity`는 이 객체를 데이터베이스에 저장하는 방법을 나타낸다. DDD Entity는 업무에서 같은 주문을 어떻게 식별하고 추적할지 설명한다.

### Value Object는 값으로 의미를 표현한다

식별자보다 값 자체와 그 의미가 중요한 객체를 **Value Object**라고 한다. 금액, 주소와 기간이 대표적이다.

주문 금액을 `BigDecimal` 하나로만 사용하면 그 숫자의 단위와 허용 범위를 매번 주변 코드에서 확인해야 한다. 프로젝트가 원화만 다룬다고 가정하고 `Money`로 묶으면 금액이라는 의미와 규칙을 한곳에 둘 수 있다.

```java
public record Money(BigDecimal amount) {

    public Money {
        Objects.requireNonNull(amount);

        if (amount.signum() < 0) {
            throw new IllegalArgumentException(
                "금액은 음수일 수 없습니다."
            );
        }
    }

    public Money add(Money other) {
        return new Money(amount.add(other.amount));
    }
}
```

두 `Money`의 금액이 같다면 같은 값으로 볼 수 있다. Value Object는 자신이 가진 값 전체로 의미가 결정되기 때문이다.

값이 중간에 바뀌면 같은 객체가 다른 의미를 갖게 된다. 이런 혼란을 막기 위해 Value Object는 불변 객체로 만드는 경우가 많다.

배송지를 바꿀 때도 기존 주소를 수정하기보다 새로운 주소 값을 만들어 전달할 수 있다.

```java
ShippingAddress newAddress =
    new ShippingAddress("서울", "강남구");

order.changeShippingAddress(newAddress);
```

첫 줄에서 새로운 `ShippingAddress`를 만들고, 두 번째 줄에서 `Order`에 전달한다. 주소라는 값은 새로 만들었지만, 주문의 배송지를 바꾸는 작업은 여전히 `Order`가 맡는다.

이제 주문과 주문 상품, 배송지를 각각 표현할 수 있다. 다음으로 정할 것은 변경의 입구다. 어느 객체를 통해 상태를 바꾸고, 어디까지 한 번에 올바른 상태를 보장해야 할까?

### Aggregate는 Root가 일관성을 지키는 경계다

주문 상품의 수량을 바꾸면 주문 총금액도 다시 계산해야 한다. 이미 취소한 주문이라면 수량 자체를 바꿀 수 없어야 한다.

외부 코드가 `OrderLine`을 직접 꺼내 수량만 바꾸면 이 규칙을 빠뜨릴 수 있다. 그래서 주문과 주문 상품처럼 함께 규칙을 지켜야 하는 객체를 하나의 경계로 묶고, 변경 요청은 대표 객체를 거치게 한다.

이 경계가 **Aggregate**다. 주문 Aggregate에서는 `Order`가 대표 객체인 **Aggregate Root**가 된다.

외부에서는 `OrderLine`을 직접 바꾸지 않고 Root인 `Order`에 변경을 요청한다. `Order`는 내부 객체의 상태까지 살피며 주문 규칙을 지킨다.

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

Aggregate가 지키는 일관성은 한 작업이 끝났을 때 경계 안의 상태가 업무 규칙을 만족한다는 의미다. 주문 상품의 수량을 바꾼 뒤에는 총금액도 그 수량과 맞아야 한다.

Entity와 Value Object는 객체의 성격을 설명한다. Aggregate는 여러 객체의 변경을 어디까지 함께 관리할지 정한다. 서로 답하는 질문이 다르다.

모든 Aggregate Root는 Entity지만, 모든 Entity가 Aggregate Root인 것은 아니다.

Aggregate는 기능 이름이 아니라 상태와 생명주기를 가진 대상을 중심으로 찾는다. `회원가입`과 `주문 취소`는 한 번의 요청을 처리하는 사용 사례이고, 그 과정에서 상태가 바뀌는 `Member`와 `Order`가 Aggregate 후보가 된다.

Aggregate가 너무 크면 작은 변경에도 많은 데이터를 불러오거나 잠가야 할 수 있다. 반대로 너무 작게 나누면 한 번에 지켜야 할 규칙이 여러 Aggregate에 흩어진다.

따라서 테이블이나 화면 구성이 아니라 **어떤 규칙을 한 트랜잭션에서 함께 지켜야 하는가**를 기준으로 경계를 조정한다.

이제 `Order`는 취소 가능 여부를 스스로 판단할 수 있다. 남은 일은 HTTP 요청에 맞는 주문을 불러오고, `cancel()`을 호출한 뒤 결과를 저장하는 것이다.

## Application Service는 도메인 모델로 기능을 완성한다

Spring 애플리케이션에서는 흔히 Controller가 요청을 받고, Service가 업무를 처리하며, Repository가 데이터에 접근한다. DDD에서는 이 가운데 Service와 도메인 객체가 맡을 일을 구분한다.

| 영역 | 맡는 일 |
|---|---|
| Presentation | 외부 요청과 응답을 애플리케이션에 연결 |
| Application Service | 도메인 객체를 준비하고 호출해 하나의 사용 사례와 트랜잭션을 완성 |
| Domain Model | 업무적으로 무엇이 가능한지 판단 |
| Infrastructure | DB, 메시지와 외부 API를 구체적인 기술로 연결 |

주문 취소 요청은 다음 순서로 진행할 수 있다.

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

`CancelOrderService`는 주문을 불러오고 `cancel()`을 호출한 뒤 저장한다. 주문 취소라는 사용 사례를 처음부터 끝까지 진행하는 역할이다.

배송을 시작한 주문을 취소할 수 있는지는 `Order.cancel()`이 판단한다. 이 조건은 HTTP API뿐 아니라 관리자 기능과 배치에서도 똑같이 지켜야 하는 주문의 규칙이기 때문이다.

이처럼 Application Service는 기능의 진행 순서를 조정하고, 도메인 모델은 업무상 허용되는 행동을 판단한다.

### Repository는 Aggregate를 저장하고 복원한다

주문 취소를 실행하려면 저장된 `Order`를 다시 불러와야 한다. Aggregate를 저장하고 복원하는 역할을 **Repository**가 맡는다.

```java
public interface OrderRepository {

    Optional<Order> findById(OrderId orderId);

    void save(Order order);
}
```

Repository는 테이블이 아니라 Aggregate 단위로 저장하고 조회한다. 예를 들어 `orders`와 `order_lines`가 서로 다른 테이블에 저장되더라도, 주문 상품은 `Order`와 함께 불러와 Root를 통해 변경한다.

`OrderLineRepository`를 따로 만들어 주문 상품만 수정하면 수량 변경 뒤 총금액을 다시 계산하는 규칙을 우회할 수 있다.

도메인에는 필요한 저장 기능을 `OrderRepository`라는 인터페이스로 정의한다.

Infrastructure 영역은 이 인터페이스를 실제 저장 기술로 구현한다. Spring Data `JpaRepository`나 `EntityManager`로 MySQL에 접근하는 코드가 여기에 들어갈 수 있다.

도메인 객체와 JPA Entity는 한 클래스로 구현할 수도 있다. 저장을 위한 매핑 코드가 업무 규칙을 표현하는 데 방해가 될 만큼 복잡해졌다면 두 모델을 분리하는 방법을 검토한다.

주문 취소처럼 상태를 바꾸고 규칙을 실행할 때는 Order Aggregate를 불러온다. 반면 주문 목록 화면에는 주문 번호, 회원명과 상품명만 있으면 된다. 이런 조회는 JOIN이나 Projection으로 필요한 값만 가져올 수 있다.

Aggregate는 특히 **변경할 때 일관성을 지키는 경계**다. 조회까지 항상 Aggregate 전체를 불러올 필요는 없다.

### 특정 객체 하나가 맡기 어려운 규칙

여러 Aggregate를 함께 살펴야 판단할 수 있는 업무 규칙도 있다. 회원 등급과 주문 금액을 함께 사용하는 할인 정책이 그런 예다.

이처럼 특정 Aggregate나 Value Object 하나에 두기 어려운 규칙은 **Domain Service**로 표현할 수 있다. 여기서는 회원과 주문을 받아 할인 금액을 계산하는 `DiscountPolicy`를 만든다.

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

할인을 적용하는 전체 흐름은 Application Service가 진행한다.

- **Application Service:** `Order`와 `Member`를 조회하고 `DiscountPolicy`를 호출한 뒤, 할인 결과를 반영해 저장한다.
- **Domain Service:** `Order`와 `Member`의 상태를 바탕으로 할인 금액을 계산한다.

Application Service는 기능의 진행 순서를 조정하고, Domain Service는 여러 도메인 객체가 필요한 판단이나 계산을 맡는다.

`DiscountPolicy`는 Order와 Member 바깥에서 두 Aggregate를 함께 살피는 도메인 모델이다. 덕분에 할인 규칙을 어느 한쪽에 억지로 넣지 않고도 업무 개념으로 표현할 수 있다.

주문 컨텍스트 안에서는 `Order`가 취소 규칙을 판단하고, Application Service가 작업을 진행한다. 이제 환불과 출고 중단을 처리하기 위해 결제와 배송 컨텍스트에 결과를 전달해야 한다.

## 다른 컨텍스트와 명시적으로 협력한다

경계를 나누어도 주문, 결제와 배송은 계속 협력한다. 다만 상대 컨텍스트의 내부 모델을 직접 가져오는 대신, 서로 주고받을 정보와 형식을 계약으로 정한다.

다음 코드는 주문 컨텍스트의 `OrderLine`이 상품 컨텍스트의 `Product`를 직접 참조한다.

```java
import catalog.domain.Product;

public class OrderLine {
    private Product product;
}
```

이 구조에서는 상품팀이 `Product`의 필수 속성을 바꿀 때 주문 코드도 영향을 받는다. 현재 판매 정보를 다루는 `Product`가 바뀌면서, 주문 당시의 상품명과 가격을 보존해야 하는 주문 모델까지 함께 흔들릴 수 있다.

상품 컨텍스트는 API 응답이나 DTO, 이벤트처럼 합의한 형식으로 필요한 정보만 제공한다. 주문 컨텍스트는 전달받은 `ProductInfo`를 자신의 모델인 `OrderedProduct`로 바꿔 사용한다.

### Command와 Domain Event는 방향이 다르다

컨텍스트 사이의 메시지는 요청과 사실로 나눠 볼 수 있다. 다른 컨텍스트에 어떤 작업을 요구하는 메시지가 Command이고, 이미 일어난 일을 알리는 메시지가 Domain Event다.

| 구분 | 예 | 의미 |
|---|---|---|
| Command | `CancelOrder` | 주문을 취소해 달라는 요청 |
| Domain Event | `OrderCanceled` | 주문이 이미 취소되었다는 사실 |

주문이 취소되었다는 사실을 코드로 표현하면 다음과 같다.

```java
public record OrderCanceled(
    OrderId orderId,
    Money refundAmount,
    Instant occurredAt
) {
}
```

결제 컨텍스트는 `OrderCanceled`를 받아 환불을 시작한다. 쿠폰 컨텍스트는 같은 이벤트를 받아 사용한 쿠폰을 복구할 수 있다.

주문 Aggregate는 주문이 취소되었다는 사실만 알린다. 환불과 쿠폰 복구를 어떻게 처리할지는 각 컨텍스트가 결정한다.

Domain Event를 전달하는 방법은 시스템 구조에 따라 달라진다. 같은 애플리케이션 안에서는 Spring Event를 사용할 수 있다. 컨텍스트가 서로 다른 서비스에 있다면 Kafka 같은 메시징 시스템을 선택할 수도 있다.

메시징 시스템을 선택하면 이벤트 유실과 중복 처리, DB 트랜잭션과의 정합성도 함께 설계해야 한다.

### 외부 모델은 내 언어로 번역한다

결제 컨텍스트가 `PaymentResponse`에 `paymentCode`와 `cancelAmount`를 담아 반환한다고 해보자. 이 객체를 주문 도메인까지 그대로 전달하면 주문 코드도 결제 컨텍스트의 이름과 구조를 알아야 한다.

경계에 Translator나 Adapter를 두면 `PaymentResponse`를 주문 컨텍스트의 `RefundResult`로 바꿀 수 있다. 결제 응답 형식이 달라져도 변환 코드만 수정하면 주문 모델은 자신의 언어를 유지한다.

이처럼 외부 모델을 내 컨텍스트의 언어로 번역하는 경계를 **부패 방지 계층**이라고 한다. 영어로는 Anti-Corruption Layer, 줄여서 ACL이라고 부른다.

여러 컨텍스트가 어떤 관계로 연결되는지 정리한 그림은 **컨텍스트 맵**이라고 한다. 먼저 어느 컨텍스트가 정보를 제공하고 누가 사용하는지, 어떤 계약으로 연결되는지 표시하면 전체 협력 관계를 파악하기 쉽다.

## DDD, 헥사고날, CQRS와 MSA는 무엇이 다를까?

DDD를 공부하면 헥사고날 아키텍처, CQRS, 이벤트 기반 아키텍처와 MSA도 자주 만난다. 함께 사용할 수 있는 개념이지만 각각 풀려는 문제가 다르다.

| 개념 | 중심 질문 |
|---|---|
| DDD | 복잡한 업무를 어떻게 이해하고 모델링할까? |
| 헥사고날·클린 아키텍처 | 도메인과 외부 기술의 의존성을 어떻게 통제할까? |
| CQRS | 상태 변경 모델과 조회 모델을 분리할까? |
| 이벤트 기반 아키텍처 | 사건을 중심으로 시스템이 어떻게 협력할까? |
| MSA | 시스템을 어떤 배포와 운영 단위로 나눌까? |

DDD는 모놀리식 애플리케이션과 레이어드 아키텍처에도 적용할 수 있다. 업무를 이해하고 모델링하는 접근이므로 MSA나 CQRS의 도입 여부와는 별개다.

어떤 구조를 선택하든 DDD의 중심에는 다음 과정이 있다.

1. 업무 담당자와 대화한다.
2. 용어와 사건을 정리한다.
3. 모델이 통하는 경계를 찾는다.
4. Aggregate 후보와 규칙을 코드로 표현한다.
5. 구현하며 새로 발견한 업무 지식을 모델에 반영한다.

정해진 패키지 구조를 복사하는 것보다 업무 담당자와 개발자가 모델을 함께 만들고, 구현하며 계속 수정하는 과정이 중요하다.

## DDD는 언제 도움이 될까?

다음과 같이 업무 자체를 이해하고 유지하기 어려워졌다면 DDD의 접근이 도움이 될 수 있다.

- 같은 용어를 팀마다 다른 의미로 사용한다.
- 상태와 예외가 많아 기능을 바꿀 때마다 조건을 빠뜨린다.
- 중요한 규칙이 여러 Service와 배치 코드에 흩어져 있다.
- 테이블 구조보다 업무 규칙의 변화가 코드를 더 자주 흔든다.
- 개발자만으로 규칙을 정하기 어려워 업무 담당자와 계속 대화해야 한다.

관리자용 CRUD나 외부 데이터를 그대로 전달하는 기능은 업무 규칙이 비교적 단순하다. 이런 영역에 풍부한 도메인 모델을 적용하면 코드와 학습 비용만 늘어날 수 있다.

먼저 단순한 Service와 Repository로 충분한지 살펴보는 편이 낫다.

한 시스템 안에서도 적용 수준은 달라질 수 있다. 복잡한 주문 정책에는 도메인 모델과 Aggregate를 사용하고, 단순한 목록 조회에는 Projection을 사용할 수 있다. 컨텍스트 사이의 결합을 줄여야 할 때는 Event를 검토한다.

핵심 서브도메인은 깊게 모델링하고, 일반 서브도메인은 단순하게 구현한다. DDD를 적용할 범위도 업무의 복잡성과 중요도에 맞춰 정한다.

## 정리

주문 취소 기능의 문제는 코드보다 앞에서 시작됐다. `취소`라는 말을 팀마다 다르게 이해했고, 주문과 결제, 배송 중 어디에서 무엇을 판단할지도 정리되지 않았다.

DDD는 도메인 전문가와 개발자가 업무를 함께 이해하는 데서 출발한다. 같은 언어를 사용하고, 그 언어와 모델이 통하는 경계를 정한다. 경계 안에서는 중요한 상태와 행동, 규칙을 코드로 표현한다.

1. 업무를 이해하고 같은 언어를 만든다.
2. 모델이 통하는 경계를 찾는다.
3. Entity와 Value Object로 업무의 대상과 값을 표현한다.
4. Aggregate Root를 통해 함께 지켜야 할 규칙을 보호한다.
5. Application Service가 모델을 사용해 기능을 완성한다.
6. 다른 컨텍스트와는 명시적인 계약으로 협력한다.

DDD의 목적은 Entity, Aggregate와 Event를 많이 만드는 데 있지 않다. 복잡한 업무를 정확하게 이해하고, 그 이해가 코드에서도 같은 의미로 유지되게 만드는 데 있다.

## 참고 자료

### 공식 자료

- [Microsoft Learn - DDD 지향 마이크로 서비스 디자인](https://learn.microsoft.com/ko-kr/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/ddd-oriented-microservice)
- [Microsoft Learn - Designing a microservice domain model](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/microservice-domain-model)
- [Microsoft Learn - Designing the infrastructure persistence layer](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-design)
- [Microsoft Learn - Domain events: Design and implementation](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/domain-events-design-implementation)
- [EventStorming - 공식 사이트](https://www.eventstorming.com/)

### 추가 자료

- [Martin Fowler - Domain-Driven Design](https://martinfowler.com/bliki/DomainDrivenDesign.html)
- [Martin Fowler - Ubiquitous Language](https://martinfowler.com/bliki/UbiquitousLanguage.html)
- [Martin Fowler - Bounded Context](https://martinfowler.com/bliki/BoundedContext.html)
- [카카오페이 기술 블로그 - 여신코어 DDD로 구축하기](https://tech.kakaopay.com/post/backend-domain-driven-design/)
- [LY Corporation Tech Blog - DDD를 Merchant 시스템 구축에 활용한 사례](https://techblog.lycorp.co.jp/ko/applying-ddd-to-merchant-system-development)
