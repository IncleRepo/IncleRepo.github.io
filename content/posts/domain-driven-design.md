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

세상에는 정말 많은 개발 이론이 있다. 기술 하나를 익혀 기능 하나 돌아가게 만드는 것도 벅찬데, TDD니 DDD니 이름이 비슷한 무언가가 계속 나온다.

이런 방법론이 괜히 생긴 것은 아니다. 개발은 혼자 하는 일이 아니고, 업무가 복잡해질수록 같은 기능을 서로 다르게 이해하기 쉽다. 오늘 알아볼 DDD도 이 간격을 줄이려는 고민에서 출발한다. 한국말로는 도메인 주도 설계다. 이름부터 벌써 묵직하다.

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

DDD는 이런 혼선을 줄이기 위해 업무를 배우고, 그 이해를 모델에 담아 계속 다듬어 가는 접근이다. 사람의 설명과 코드가 같은 업무를 가리키게 만들려면 먼저 우리가 해결하려는 일이 무엇인지 알아야 한다.

이 글에서는 다음 흐름을 따라간다. 용어를 따로 외우기보다 앞에서 발견한 문제가 왜 다음 개념을 필요로 하는지 보면 된다.

![업무 이해에서 모델과 경계를 거쳐 실행과 협력으로 이어지는 DDD의 흐름](/images/posts/domain-driven-design/ddd-concept-map.svg "DDD를 따라가는 네 단계")

## 업무를 이해해 모델을 만든다

**도메인**은 소프트웨어로 해결하려는 현실의 업무 영역이다. 쇼핑몰이라면 주문, 결제, 배송과 재고가 도메인에 해당한다. Java의 `domain` 패키지나 데이터베이스를 뜻하는 말이 아니다.

도메인을 이해하면서 중요한 개념과 상태, 행동과 규칙을 추려 표현한 것이 **도메인 모델**이다.

주문 업무를 살펴보면 다음과 같은 내용이 보인다.

1. 주문에는 여러 상품이 담긴다.
2. 주문은 현재 상태를 가진다.
3. 배송이 시작된 주문은 취소할 수 없다.

여기에서 `Order`, `OrderLine`, `OrderStatus` 같은 개념을 찾고, 취소 규칙은 `cancel()`이라는 행동으로 표현할 수 있다.

도메인 모델은 처음부터 Java 클래스나 DB 테이블을 의미하지 않는다. 먼저 업무를 설명하는 개념적 모델이 있고, 이를 Java와 객체지향 코드로 구현할 수 있다.

Entity, Value Object와 Aggregate도 Java나 JPA가 만든 기능이 아니다. 도메인을 모델링할 때 사용하는 설계 개념이며, Java 클래스와 객체는 이를 구현하는 한 방법이다.

### 도메인 전문가는 업무의 예외를 알고 있다

모델은 개발자 혼자 상상해서 만들 수 없다. 실제 업무의 규칙과 예외를 잘 아는 사람과 함께 확인해야 한다. 이런 사람을 **도메인 전문가**라고 부른다.

거창한 직책만 가리키는 말은 아니다. 주문 상태를 관리하는 운영자, 환불 규정을 정하는 기획자와 반복되는 문의를 처리하는 고객센터 직원도 각자의 영역에서는 도메인 전문가다.

개발자는 이들과 함께 주문 취소의 흐름을 펼쳐 본다.

1. 고객이 주문 취소를 요청한다.
2. 주문이 취소 가능한 상태인지 확인한다.
3. 주문을 취소한다.
4. 결제에는 환불을, 물류에는 출고 중단을 요청한다.
5. 사용한 쿠폰과 재고를 복구한다.

순서는 단순해 보이지만 질문은 계속 생긴다. 환불 요청이 실패하면 주문 취소도 되돌려야 할까? 이미 출고됐다면 취소가 아니라 반품으로 바뀌어야 할까? 하나씩 답하다 보면 모델에 담아야 할 상태와 규칙이 보인다.

하지만 여러 사람이 함께 모델을 만들려면 같은 말을 같은 뜻으로 사용해야 한다. 여기서 유비쿼터스 언어가 필요해진다.

## 같은 말을 같은 의미로 사용한다

주문팀의 `취소`, 결제팀의 `환불`, 물류팀의 `출고 중단`은 서로 연결되어 있지만 같은 행동은 아니다. 이 차이를 합의하지 않으면 회의에서는 통하는 것처럼 보여도 코드에서는 다시 어긋난다.

도메인 전문가와 개발자가 같은 개념을 같은 뜻으로 사용하는 공통 언어를 **유비쿼터스 언어**라고 한다. 우리말로는 보편 언어라고도 부른다. 함께 정한 언어는 회의와 문서뿐 아니라 화면, 테스트와 코드에도 그대로 사용한다.

```java
order.cancel();
refund.request();
shipment.stopDispatch();
```

좋은 이름을 짓는 것과 유비쿼터스 언어는 닮았지만 범위가 다르다. 좋은 이름은 코드를 읽기 쉽게 만든다. 유비쿼터스 언어는 그보다 먼저 `취소가 정확히 무엇인지`를 업무 담당자와 개발자가 함께 합의하게 한다. `cancel()`이라는 이름은 그 합의가 코드에 드러난 결과다.

반대로 코드에 `changeStatus(4)`만 남아 있다면 숫자 `4`의 의미를 다시 사람의 기억에서 찾아야 한다. 용어집만 작성하고 코드에서는 다른 말을 사용하는 것도 같은 문제를 남긴다.

### 대화 속 사건을 시간 순서로 펼쳐 본다

업무 흐름과 용어를 함께 찾을 때는 **이벤트 스토밍** 같은 협업 방법을 활용할 수 있다. 업무에서 이미 일어난 사건을 시간 순서대로 펼쳐 놓고, 질문을 주고받으며 빠진 과정과 서로 다른 해석을 찾는다.

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

대화를 밖으로 꺼내 놓으면 흐름이 조금 더 구체적으로 보인다.

1. 주문이 생성되었다.
2. 재고가 예약되었다.
3. 결제가 완료되었다.

결제에 실패했다면 세 번째 사건 대신 `예약 재고가 해제되었다`가 이어진다.

이 대화에서는 사건뿐 아니라 사건을 일으키는 대상과 그 과정에서 지켜야 할 규칙도 찾을 수 있다.

이 과정에서 대화는 차츰 모델과 코드로 구체화된다.

1. **업무 대화:** 배송이 시작되기 전까지만 주문을 취소할 수 있다.
2. **사건과 규칙:** 주문이 취소되었다. 배송을 시작한 주문은 취소할 수 없다.
3. **모델과 경계 후보:** `Order`와 `Shipment`의 책임을 나눈다.
4. **코드:** `Order.cancel()`과 `CancelOrderService`로 표현한다.

대화에서 얻은 재료를 바탕으로 이후 `Order`의 성격과 경계를 정한다. Entity와 Aggregate를 살펴보면서 이 내용이 실제 코드로 바뀌는 과정을 이어 가자.

![EventStorming을 활용해 업무의 사건과 경계를 함께 탐색하는 모습](/images/posts/domain-driven-design/event-storming.jpg "EventStorming을 활용해 업무의 사건과 경계를 함께 탐색하는 모습. 출처: EventStorming 공식 사이트")

포스트잇의 색상보다 중요한 것은 사람마다 머릿속에 있던 업무를 한곳에 꺼내 놓는 과정이다. 질문하다가 새로운 사건과 예외를 발견했다면 모델도 함께 바뀐다. DDD는 분석을 끝내고 완벽한 모델을 확정한 뒤 개발하는 방식이 아니다. 업무 이해, 모델링과 구현을 반복하며 모델을 발전시킨다.

이제 주문팀 안에서는 같은 언어로 대화할 수 있게 되었다. 그렇다고 회사 전체가 모든 단어를 같은 뜻으로 사용해야 할까? 상품팀과 주문팀이 말하는 `상품`부터 이미 의미가 다르다.

## 하나의 모델이 통하는 경계를 찾는다

회사의 큰 업무도 한 덩어리로 이해하기는 어렵다. 복잡한 도메인을 주문, 결제와 배송처럼 작은 문제 영역으로 나눈 것이 **서브도메인**이다.

그런데 현실의 업무를 나누는 것과 소프트웨어 모델의 경계를 정하는 것은 같은 일이 아니다.

| 구분 | 던지는 질문 | 다루는 대상 |
|---|---|---|
| 서브도메인 | 현실의 업무를 어떻게 나눌 것인가? | 문제 영역 |
| 바운디드 컨텍스트 | 그 업무를 표현하는 모델의 경계를 어디까지 둘 것인가? | 해결 영역 |

주문 서브도메인을 하나의 주문 컨텍스트가 맡을 수 있다. 시스템이 커지면 주문 접수와 주문 이력을 서로 다른 컨텍스트가 나누어 맡을 수도 있다. 따라서 서브도메인과 바운디드 컨텍스트가 항상 일대일로 대응하지는 않는다.

이제 소프트웨어에서 모델의 경계를 정해 보자. 상품팀의 `상품`에는 현재 판매 가격, 설명과 선택 가능한 옵션이 중요하다. 주문팀의 `상품`에는 고객이 주문한 당시의 이름, 가격과 수량이 중요하다. 상품팀이 오늘 가격을 바꿨다고 어제 결제한 주문 금액까지 바뀌어서는 안 된다.

DDD에서는 하나의 모델과 언어가 같은 의미로 통하는 명시적인 범위를 **바운디드 컨텍스트**라고 한다. 같은 `상품`이라도 상품 컨텍스트의 `Product`와 주문 컨텍스트의 주문 상품은 서로 다른 모델일 수 있다.

주문 취소 역시 경계마다 책임이 다르다. 주문 컨텍스트는 주문 상태를 바꾸고, 결제 컨텍스트는 환불 가능 여부와 금액을 판단하며, 배송 컨텍스트는 출고를 멈출 수 있는지 판단한다. 한 컨텍스트가 모든 규칙을 소유하지 않고 각자의 모델로 판단한 뒤 필요한 요청이나 사건을 주고받는다.

바운디드 컨텍스트의 핵심은 패키지나 배포 단위가 아니라 **모델과 언어의 의미 경계**에 있다. 구현할 때는 `order`, `payment` 같은 패키지나 Gradle 모듈로 표현하고, 필요하면 독립 서비스로 분리할 수 있다. 물리적인 구조는 이 경계를 드러내는 수단이다.

다음 그림은 큰 컨텍스트 경계와 그 안에 있는 작은 Aggregate 경계를 나누어 보여준다.

![주문 컨텍스트 안의 여러 애그리거트와 상품 컨텍스트의 관계](/images/posts/domain-driven-design/context-and-aggregate.svg "바운디드 컨텍스트와 애그리거트 경계의 차이")

하나의 바운디드 컨텍스트 안에는 여러 Aggregate가 들어갈 수 있다. 같은 컨텍스트 안에서도 Aggregate마다 일관성 경계를 지키고, 컨텍스트 밖의 Entity는 내 모델처럼 직접 사용하지 않는다.

서브도메인은 회사에서 차지하는 중요도에 따라 핵심, 지원과 일반 서브도메인으로 구분하기도 한다.

| 구분 | 의미 | 쇼핑몰 예시 |
|---|---|---|
| 핵심 | 회사의 경쟁력을 만드는 업무 | 독자적인 추천·가격 정책 |
| 지원 | 핵심 업무를 돕는 회사 고유 업무 | 주문·정산 운영 |
| 일반 | 여러 회사가 비슷하게 해결하는 업무 | 이메일 발송 |

무엇이 핵심인지는 기술 난이도가 아니라 회사가 무엇으로 경쟁하는지에 따라 달라진다. 모든 영역에 같은 비용을 쓰지 않고, 복잡성과 경쟁력이 큰 곳에 깊은 모델링을 투자하기 위한 구분이다.

여기까지처럼 큰 업무와 모델의 경계를 찾는 일을 DDD의 **전략적 설계**라고 부른다. 이제 각 컨텍스트의 모델을 Entity, Value Object와 Aggregate로 표현하는 **전술적 설계**로 내려가 보자.

## 경계 안의 모델을 코드로 표현한다

주문 컨텍스트에는 주문 번호, 주문 상태와 주문 상품이 필요하다. `배송이 시작된 주문은 취소할 수 없다`는 규칙도 함께 표현해야 한다.

처음에는 다음처럼 상태를 직접 바꾸는 코드를 작성할 수 있다.

```java
order.setStatus(OrderStatus.CANCELED);
```

이 한 줄만으로는 누가 어떤 상태에서 취소할 수 있는지 알 수 없다. 호출하는 모든 코드가 취소 규칙을 따로 기억해야 한다. 이제 업무의 대상과 값을 구분하고, 변경할 때 반드시 지켜야 할 규칙의 경계를 찾아야 한다.

### Entity는 식별자로 이어진다

속성이 달라져도 같은 대상으로 계속 추적해야 하는 객체를 DDD에서는 **Entity**라고 한다.

주문의 배송지와 상태는 바뀔 수 있다. 그래도 주문 번호가 같다면 어제 조회한 주문과 오늘 조회한 주문은 같은 주문이다. Entity에서 중요한 것은 현재 속성의 조합이 아니라 `어떤 대상을 계속 추적하는가`이다.

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

외부에서 상태를 마음대로 바꾸지 않고 `cancel()`을 거치게 했다. 이제 주문 취소의 의미와 취소할 수 있는 조건을 `Order`가 함께 보여준다.

Spring 개발자라면 Entity라는 말을 보고 JPA의 `@Entity`부터 떠올릴 수 있다. 둘은 같은 말처럼 자주 쓰이지만 기준이 다르다.

| 구분 | 의미 |
|---|---|
| DDD Entity | 식별성을 중심으로 같은 대상을 추적하는 도메인 모델 개념 |
| JPA `@Entity` | 객체를 관계형 데이터베이스에 매핑하는 ORM 기술 |

`Order` 하나가 DDD Entity이면서 JPA Entity일 수는 있다. 하지만 `@Entity`가 붙었다는 이유만으로 업무 규칙과 식별성이 잘 표현된 DDD Entity가 되는 것은 아니다.

### Value Object는 값으로 의미를 표현한다

고유한 식별자보다 값 자체와 그 의미가 중요한 객체를 **Value Object**라고 한다. 금액, 주소와 기간이 대표적이다.

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

같은 금액을 가진 두 `Money`는 같은 값으로 볼 수 있다. 주문처럼 생성된 뒤 변하는 대상을 식별자로 추적하는 Entity와 달리, Value Object는 값 전체가 의미를 결정한다. 그래서 불변 객체로 만드는 경우가 많다.

Value Object는 Aggregate 안에서만 생성되는 부속품은 아니다.

```java
ShippingAddress newAddress =
    new ShippingAddress("서울", "강남구");

order.changeShippingAddress(newAddress);
```

첫 줄은 새로운 주소 값을 만든다. 두 번째 줄에서 그 값이 `Order`의 상태로 들어가면, 이후 주소 변경은 Order Aggregate가 허용한 규칙을 따라야 한다.

Entity와 Value Object를 만들었더라도 객체가 많아지면 또 다른 질문이 생긴다. 어느 객체를 통해 상태를 바꾸고, 어디까지 한 번에 올바른 상태를 보장해야 할까?

### Aggregate는 Root가 일관성을 지키는 경계다

주문에는 여러 주문 상품과 배송지가 들어 있다. 주문 상품의 수량을 바꾸면 총금액도 다시 계산해야 하고, 이미 취소한 주문이라면 수량 변경을 막아야 한다.

외부 코드가 `OrderLine`을 직접 꺼내 수량만 바꾸면 총금액 재계산을 놓칠 수 있다. 따라서 외부의 변경 요청은 대표 객체 하나를 거치게 한다.

**Aggregate**는 Root Entity가 내부 상태와 업무 불변식을 책임지는 일관성·생명주기 경계다. 주문 Aggregate에서는 `Order`가 **Aggregate Root**가 되어 외부의 변경 요청을 받는다.

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

다음 그림의 핵심은 소유 관계보다 변경 경로에 있다. 외부의 변경 요청은 Root인 `Order`를 거쳐야 하므로 주문 규칙을 빠뜨리지 않는다.

![Order가 Aggregate Root가 되어 주문 상품과 배송지 변경을 통제하는 구조](/images/posts/domain-driven-design/order-aggregate.svg "주문 Aggregate의 구조")

`함께 일관성을 지킨다`는 말이 모든 값을 매번 동시에 바꾼다는 뜻은 아니다. 회원의 이름과 주소를 서로 다른 날에 바꿔도 같은 Member Aggregate일 수 있다. Member Root는 이 상태들의 유효성과 생명주기를 자신의 경계 안에서 책임진다.

Entity와 Value Object는 도메인 객체의 성격을 설명한다. Aggregate는 이런 객체를 Root 중심으로 묶는 경계다. 따라서 세 용어는 같은 층의 객체 종류가 아니다.

모든 Aggregate Root는 Entity지만, 모든 Entity가 Aggregate Root인 것은 아니다.

또한 Aggregate는 기능 이름으로 정하지 않는다. `회원가입`, `로그인`과 `주문 취소`는 보통 한 번의 요청을 완성하는 사용 사례다. `Member`, `Session`과 `Order`처럼 상태와 생명주기를 가진 대상이 Aggregate 후보가 된다.

Aggregate를 크게 잡으면 매번 많은 데이터를 불러오고 잠가야 할 수 있다. 너무 작게 나누면 한 번에 지켜야 할 규칙이 여러 Aggregate에 흩어진다. 무엇을 함께 저장하고 싶은지가 아니라 **어떤 규칙을 한 트랜잭션에서 지켜야 하는가**를 기준으로 경계를 조정해야 한다.

이제 `Order`는 스스로 취소 가능 여부를 판단한다. HTTP 요청을 받아 Aggregate를 불러오고 주문 취소 작업을 끝까지 진행할 코드가 따로 필요하다.

## Application Service는 도메인 모델로 기능을 완성한다

Spring 입문에서는 흔히 Controller는 요청, Service는 비즈니스 로직, Repository는 DB를 맡는다고 배운다. DDD에서는 Service 안의 일을 조금 더 나눠서 본다.

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

`CancelOrderService`는 주문을 불러오고 `cancel()`을 호출한 뒤 저장한다. 이것은 사용 사례의 **진행 순서**다. 반면 배송을 시작한 주문을 취소할 수 있는지는 `Order.cancel()`이 판단한다. 이것이 **도메인 규칙**이다.

Application Service는 기능을 완료하는 순서를 조정하고, 도메인 모델은 업무상 가능한 행동을 판단한다. 취소 규칙을 `Order`에 두면 관리자 기능이나 배치에서도 같은 조건을 그대로 사용할 수 있다.

### Repository는 Aggregate를 저장하고 복원한다

Order Aggregate를 하나의 일관성 단위로 정했다면 저장소에서도 완전한 Order Aggregate로 불러와야 한다. 이 저장과 복원의 계약을 **Repository**가 맡는다.

```java
public interface OrderRepository {

    Optional<Order> findById(OrderId orderId);

    void save(Order order);
}
```

Repository는 테이블보다 Aggregate를 중심으로 본다. `orders`와 `order_lines` 테이블이 따로 있다는 이유만으로 `OrderRepository`와 `OrderLineRepository`를 각각 만들어 내부 주문 상품을 독립적으로 수정하면, Root가 지키던 규칙을 우회할 수 있다.

DDD의 Repository와 Spring Data의 `JpaRepository`는 역할이 다르다. 도메인에는 필요한 저장 기능을 `OrderRepository`라는 계약으로 두고, Infrastructure에서는 `JpaOrderRepository`, `EntityManager`와 MySQL로 그 계약을 구현한다.

도메인 객체와 JPA Entity를 한 클래스로 함께 쓰는 것도 가능하다. 규모와 복잡성이 커져 저장 기술 때문에 업무 규칙을 표현하기 어려워질 때 두 모델의 분리를 검토하면 된다.

조회도 언제나 Aggregate 전체를 복원할 필요는 없다. 주문 취소처럼 상태를 바꾸고 규칙을 실행할 때는 Order Aggregate가 필요하다. 주문 목록 화면에 주문 번호, 회원명과 상품명만 필요하다면 JOIN이나 Projection으로 조회 전용 결과를 만들 수 있다.

Aggregate는 특히 **변경의 일관성 경계**다. 조회는 화면과 사용 목적에 맞는 모델을 사용할 수 있다.

### 특정 객체 하나가 맡기 어려운 규칙

업무 규칙 중에는 Aggregate Root 하나가 맡기 어려운 것도 있다. 회원 등급과 주문 금액을 함께 봐야 하는 할인 정책이 그런 예다.

먼저 한 Aggregate나 Value Object가 이 규칙을 맡을 수 있는지 확인한다. 어느 한 객체에 넣기 어렵고 규칙 자체가 중요한 도메인 개념이라면 **Domain Service**로 표현할 수 있다.

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

Application Service는 기능의 진행 순서를 조정하고, Domain Service는 업무 판단이나 계산을 맡는다. 어느 객체가 규칙을 가장 잘 설명하는지 살펴본 뒤 자리를 정한다.

`DiscountPolicy`는 도메인 모델의 일부지만 Order나 Member Aggregate의 구성원은 아니다. Domain Service도 도메인 개념이므로 Aggregate 경계 바깥에 존재할 수 있다.

한 컨텍스트 안에서 모델을 만들고 사용하는 흐름은 갖춰졌다. 주문을 취소한 뒤 환불과 출고 중단을 처리하려면 다른 컨텍스트와 협력해야 한다.

## 다른 컨텍스트와 명시적으로 협력한다

바운디드 컨텍스트를 나눠도 주문, 결제와 배송은 계속 협력해야 한다. 이때 무엇을 주고받는지 명시적인 계약으로 드러낸다.

다음처럼 주문 컨텍스트가 상품 컨텍스트의 Entity를 자신의 내부 모델처럼 사용하면 경계가 흐려진다.

```java
import catalog.domain.Product;

public class OrderLine {
    private Product product;
}
```

상품팀이 `Product`의 가격 정책이나 필수 속성을 바꾸면 주문 코드까지 함께 흔들릴 수 있다. 주문 시점의 상품명과 가격을 보존해야 하는 주문 모델이 상품 컨텍스트의 현재 판매 모델에 끌려가기도 한다.

대신 API 응답이나 명시적인 DTO, Event로 필요한 정보만 받는다. 상품 컨텍스트가 제공한 `ProductInfo`는 주문 컨텍스트의 `OrderedProduct`로 바꿔 사용한다.

### Command와 Domain Event는 방향이 다르다

다른 Context에 `무엇을 해달라`고 요청할 수 있고, 이미 일어난 사실을 알릴 수도 있다.

| 구분 | 예 | 의미 |
|---|---|---|
| Command | `CancelOrder` | 주문을 취소해 달라는 요청 |
| Domain Event | `OrderCanceled` | 주문이 이미 취소되었다는 사실 |

**Domain Event**는 도메인에서 이미 일어난 의미 있는 사건이다.

```java
public record OrderCanceled(
    OrderId orderId,
    Money refundAmount,
    Instant occurredAt
) {
}
```

결제 컨텍스트는 이 사건을 받아 환불을 시작하고, 쿠폰 컨텍스트는 사용한 쿠폰을 복구할 수 있다. 주문 Aggregate는 자신이 알 필요 없는 후속 처리와 직접 결합되지 않는다.

Domain Event는 Kafka나 비동기 처리를 전제로 하지 않는다. 같은 애플리케이션 안에서 Spring Event로 동기 전달할 수도 있다. Kafka는 사건을 전달하는 기술 중 하나다. 메시지 큐를 사용한다면 이벤트 유실과 중복, DB 트랜잭션과의 불일치도 별도로 설계해야 한다.

### 외부 모델은 내 언어로 번역한다

결제 컨텍스트의 `PaymentResponse`, `paymentCode`와 `cancelAmount`를 주문 도메인까지 그대로 전달하면 주문이 결제의 언어에 종속된다. Translator나 Adapter를 두어 이 응답을 주문 컨텍스트의 `RefundResult`로 바꿀 수 있다.

이처럼 외부 모델을 내 컨텍스트의 언어로 번역해 내부 모델을 보호하는 경계를 **부패 방지 계층**이라고 한다. 영어로는 Anti-Corruption Layer, 줄여서 ACL이라고 부른다.

컨텍스트 사이의 관계를 한눈에 정리한 그림을 **컨텍스트 맵**이라고 한다. 모든 관계 패턴을 외우기보다 어느 컨텍스트가 정보를 제공하고 누가 사용하는지, 어떤 계약으로 연결되는지부터 정리하면 된다.

## DDD, 헥사고날, CQRS와 MSA는 무엇이 다를까?

DDD를 공부하다 보면 헥사고날 아키텍처, CQRS, 이벤트 기반 아키텍처와 MSA가 함께 등장한다. 서로 잘 어울릴 수 있지만 해결하려는 질문은 다르다.

| 개념 | 중심 질문 |
|---|---|
| DDD | 복잡한 업무를 어떻게 이해하고 모델링할까? |
| 헥사고날·클린 아키텍처 | 도메인과 외부 기술의 의존성을 어떻게 통제할까? |
| CQRS | 상태 변경 모델과 조회 모델을 분리할까? |
| 이벤트 기반 아키텍처 | 사건을 중심으로 시스템이 어떻게 협력할까? |
| MSA | 시스템을 어떤 배포와 운영 단위로 나눌까? |

DDD는 모놀리식 애플리케이션과 평범한 레이어드 구조에도 적용할 수 있다. MSA나 CQRS를 함께 도입해야 성립하는 방식이 아니다.

실무에서는 특정 회사의 패키지 구조를 복사하기보다 다음 과정을 반복한다.

1. 업무 담당자와 대화한다.
2. 용어와 사건을 정리한다.
3. 모델이 통하는 경계를 찾는다.
4. Aggregate 후보와 규칙을 코드로 표현한다.
5. 구현하며 새로 발견한 업무 지식을 모델에 반영한다.

국내 적용 사례를 살펴봐도 공통점은 특정 구현 패턴이 아니라 과정에 있다. 업무 담당자와 개발자가 모델과 경계를 함께 정하고 그 결과를 코드로 옮긴다.

## DDD는 언제 도움이 될까?

DDD는 업무가 복잡할수록 가치가 커진다.

- 하나의 용어를 팀마다 다르게 이해함
- 상태와 예외 규칙이 많아 변경할 때마다 빠뜨리는 조건이 생김
- 중요한 규칙이 여러 Service와 배치 코드에 흩어짐
- 데이터 구조보다 업무 변화가 시스템을 더 자주 흔듦
- 개발자만으로 정확한 규칙을 정하기 어려워 업무 담당자와 지속적인 협업이 필요함

반대로 단순한 관리자 CRUD나 외부 데이터를 그대로 전달하는 기능이라면 풍부한 도메인 모델이 주는 이점보다 코드와 학습 비용이 더 클 수 있다. 이때는 평범한 Service와 Repository로도 충분하다.

한 시스템 안에서도 복잡한 주문 정책에는 풍부한 도메인 모델과 Aggregate를 사용하고, 단순 조회에는 Projection을 사용할 수 있다. 다른 컨텍스트와 느슨하게 협력해야 할 때만 Event를 검토해도 된다. 핵심 서브도메인은 깊게 모델링하고 일반 서브도메인은 단순하게 구현하는 식으로 투자 수준을 달리한다.

DDD를 도입한다는 말은 시스템 전체에 같은 패턴을 강제한다는 뜻이 아니다.

## 정리

주문 취소 기능의 문제는 처음부터 코드에만 있지 않았다. `취소`라는 말을 각 팀이 다르게 이해했고, 어느 업무가 무엇을 판단해야 하는지도 흐렸다.

DDD는 이 업무를 함께 이해하면서 출발한다. 도메인 전문가와 개발자가 같은 언어를 만들고, 그 언어와 모델이 통하는 경계를 찾는다. 경계 안에서는 중요한 상태와 행동, 규칙을 도메인 모델로 표현한다.

1. 업무를 이해하고 같은 언어를 만든다.
2. 모델이 통하는 경계를 찾는다.
3. Entity와 Value Object로 업무의 대상과 값을 표현한다.
4. Aggregate Root를 통해 함께 지켜야 할 규칙을 보호한다.
5. Application Service가 모델을 사용해 기능을 완성한다.
6. 다른 컨텍스트와는 명시적인 계약으로 협력한다.

Entity는 식별성이 중요한 도메인 객체이고, Value Object는 값 자체와 의미가 중요한 도메인 객체다. Aggregate는 이들과 나란한 객체 종류가 아니라 Root가 내부 상태의 일관성을 책임지는 경계다.

Application Service는 도메인 객체를 준비하고 호출해 사용 사례를 진행한다. Domain Service는 Aggregate 하나가 맡기 어려운 독립적인 업무 규칙을 표현한다. Repository는 테이블보다 Aggregate를 저장하고 복원하는 일에 초점을 맞춘다.

결국 DDD는 Entity, Aggregate와 Event를 많이 사용하는 방법이 아니다. 복잡한 업무를 더 정확하게 이해하고 코드에 남기기 위해 필요한 모델링 도구를 필요한 곳에 사용하는 접근이다.

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
