+++
title = 'DDD, 도대체 도메인이 뭔데?'
slug = '15'
date = 2026-08-26T22:10:00+09:00
lastmod = 2026-08-26T22:10:00+09:00
draft = false
references_required = true
description = 'DDD가 필요한 이유부터 도메인과 경계, Entity와 Value Object, Aggregate, Repository, Event와 CQRS까지 하나의 흐름으로 살펴봅니다.'
categories = ['애플리케이션 아키텍처']
tags = ['DDD', '도메인 모델', '바운디드 컨텍스트', '애그리거트', '도메인 이벤트']
+++

세상에는 정말 많은 개발 이론이 있다. 기술 하나를 익혀 기능 하나 돌아가게 만드는 것도 벅찬데, TDD니 DDD니 이름이 비슷한 무언가가 계속 나온다.

이런 방법론이 괜히 생긴 것은 아니다. 개발은 혼자 하는 일이 아니고, 함께 일하는 사람이 많아질수록 같은 업무를 서로 다르게 이해한다. 오늘 알아볼 DDD도 이 차이를 줄이려는 고민에서 출발한다. 한국말로는 도메인 주도 설계다. 이름부터 벌써 묵직하다.

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

> 여러 사람이 복잡한 업무를 같은 말과 기준으로 이해하려면 어떻게 해야 할까?

DDD는 이 질문에서 출발한다.

## 도메인은 해결하려는 업무 영역이다

도메인은 **소프트웨어로 해결하려는 현실의 문제 영역이자 업무 영역**이다.

쇼핑몰의 주문 처리, 은행의 대출 심사, 공장의 철강 가격 계산이 모두 도메인이 될 수 있다. 데이터베이스나 통신 방식, 라이브러리 버전은 그 업무를 구현하기 위한 기술적인 선택이다.

도메인이 크고 복잡하면 서로 다른 업무를 기준으로 더 작은 영역으로 나눌 수 있다. 이를 **서브도메인**이라고 한다. 쇼핑몰이라면 다음과 같이 구분할 수 있다.

- **상품:** 판매할 상품과 가격 관리
- **주문:** 고객이 선택한 상품과 주문 상태 관리
- **결제:** 결제 승인과 환불 처리
- **배송:** 출고와 배송 상태 관리

이는 메뉴나 폴더를 나눈 것과 다르다. 각 서브도메인에는 고유한 개념과 규칙이 있다. 같은 주문 취소를 두고 팀마다 다른 일을 떠올린 것도 각자 바라보는 업무 영역이 달랐기 때문이다.

### 서브도메인은 중요도에 따라 나뉜다

회사가 모든 업무에 같은 비용을 쓸 수는 없다. DDD에서는 어디에 힘을 집중할지 판단하기 위해 서브도메인을 다음과 같이 구분한다.

- **핵심 서브도메인:** 회사의 경쟁력을 만드는 가장 중요한 업무
- **지원 서브도메인:** 핵심 업무를 돕는 회사 고유의 업무
- **일반 서브도메인:** 인증이나 이메일 발송처럼 여러 회사에 공통으로 필요한 업무

무엇이 핵심인지는 기술 난이도가 아니라 회사가 무엇으로 경쟁하는지에 따라 달라진다. 일반 쇼핑몰에서는 결제가 외부 PG 연동에 가까울 수 있지만, 결제 회사에서는 결제 자체가 핵심 서브도메인이다.

## 업무를 도메인 모델로 정리한다

도메인을 찾았다고 화이트보드에 `쇼핑몰 운영`이라고 크게 적는 것만으로 프로그램을 만들 수는 없다. 중요한 개념과 관계, 규칙을 골라 표현해야 한다. 이것이 **도메인 모델**이다.

도메인 모델은 지도와 비슷하다. 지도가 현실의 모든 모습을 복사하지 않듯이, 도메인 모델도 업무의 모든 세부사항을 담지 않는다. 해결하려는 문제에 필요한 부분만 남긴다.

주문 취소를 모델링한다면 다음과 같은 질문에 답할 수 있어야 한다.

- 어떤 상태의 주문을 취소할 수 있는가?
- 결제가 끝난 주문을 취소하면 환불은 언제 요청하는가?
- 배송이 시작됐다는 기준은 무엇인가?
- 취소한 주문에 사용한 쿠폰과 재고는 어떻게 복구하는가?

처음에는 글이나 그림으로 업무를 이해하고, 개발에 들어가면 같은 개념과 규칙을 객체와 동작으로 옮긴다. 형식보다 중요한 것은 모델과 코드가 같은 이야기를 하는 것이다.

```java
order.cancel();
```

문서에는 `주문을 취소한다`고 적혀 있는데 코드에는 의미를 알 수 없는 `changeStatus(4)`만 남아 있다면, 힘들게 정리한 업무 지식이 코드에서 다시 사라진다.

### 팀이 사용하는 언어를 맞춘다

도메인 모델은 개발자 혼자 상상해서 만들 수 없다. 실제 업무의 규칙과 예외를 아는 **도메인 전문가**와 함께 만들어야 한다.

도메인 전문가는 특별한 직책이나 단 한 명을 뜻하지 않는다. 주문 담당자는 주문 상태를, 결제 담당자는 환불을, 고객센터 직원은 자주 발생하는 예외를 잘 안다. 개발자는 이들과 대화하며 애매한 표현과 서로 다른 해석을 찾아 모델과 코드에 반영한다.

이때 활용할 수 있는 방법 중 하나가 **이벤트 스토밍**이다. 업무에서 일어난 사건을 시간 순서대로 늘어놓고 서로의 이해를 맞춰가는 방식이다.

> **진행자:** 고객이 취소 버튼을 누르면 가장 먼저 무슨 일이 생기나요?
>
> **주문팀:** 주문이 취소됩니다.
>
> **결제팀:** 결제가 끝났다면 환불 요청도 필요합니다.
>
> **물류팀:** 출고가 시작됐다면 취소할 수 없습니다.
>
> **고객센터:** 환불 전에도 주문이 취소됐다고 안내하는데요?
>
> **개발자:** 그러면 주문 취소, 환불 요청과 환불 완료를 구분해야겠네요.

대화를 시작하자 같은 `취소` 안에 여러 사건이 섞여 있었다는 사실이 드러난다. 사건의 순서를 바꾸고 빠진 내용을 채우는 동안 업무의 흐름과 용어도 선명해진다.

![EventStorming을 활용해 업무의 사건과 경계를 함께 탐색하는 모습](/images/posts/domain-driven-design/event-storming.jpg "EventStorming을 활용해 업무의 사건과 경계를 함께 탐색하는 모습")

*출처: [EventStorming 공식 사이트](https://www.eventstorming.com/)*

이벤트 스토밍에는 널리 쓰이는 진행 방식과 색상 규칙이 있지만, 형식을 완벽하게 지키는 것이 목적은 아니다. 서로 다른 관점을 한곳에 꺼내 함께 이해하는 일이 먼저다.

이렇게 같은 모델을 다루는 사람들이 같은 의미로 사용하는 말을 **유비쿼터스 언어**라고 한다. 우리말로는 보편 언어라고도 부른다. 합의한 용어는 회의에서 끝나지 않고 문서와 화면, 테스트와 코드까지 이어져야 한다.

```java
order.cancel();
payment.requestRefund();
shipment.cancelDispatch();
```

업무를 더 깊이 이해해 의미가 달라지면 언어와 모델, 코드도 함께 바뀐다.

## 같은 말도 경계가 바뀌면 뜻이 달라진다

회사 전체가 모든 단어를 단 하나의 뜻으로 사용할 필요는 없다. 같은 단어도 업무 영역에 따라 필요한 의미가 달라질 수 있다.

상품팀과 주문팀은 모두 `상품`을 다룬다. 상품팀에는 설명, 옵션과 현재 판매 가격이 중요하다. 주문팀에는 고객이 주문한 당시의 상품명, 수량과 가격이 중요하다. 상품 가격이 나중에 바뀌어도 이미 결제한 주문 금액까지 바뀌어서는 안 된다.

하나의 거대한 `Product` 모델로 두 업무를 모두 해결하려 하면 서로 필요하지 않은 속성과 규칙이 계속 섞인다. DDD에서는 하나의 모델과 언어가 일관되게 통하는 범위를 **바운디드 컨텍스트**라고 한다.

![여러 모델이 각자의 경계 안에서 일관성을 유지하는 Bounded Context](/images/posts/domain-driven-design/bounded-context.png "여러 모델이 각자의 경계 안에서 일관성을 유지하는 Bounded Context")

*출처: [Martin Fowler – Bounded Context](https://martinfowler.com/bliki/BoundedContext.html)*

두 컨텍스트는 같은 현실을 바라보더라도 서로 다른 모델을 가질 수 있다. 주문 컨텍스트는 상품 컨텍스트의 객체를 그대로 들고 오는 대신 주문 시점에 필요한 값만 자신의 모델로 저장할 수 있다.

```java
public record OrderedProduct(
    Long productId,
    String productName,
    Money unitPrice
) {
}
```

이 `OrderedProduct`는 현재 판매 중인 상품 전체를 표현하지 않는다. 주문이 성립한 순간의 상품 정보를 주문의 관점에서 표현한다.

**서브도메인**과 **바운디드 컨텍스트**는 비슷해 보이지만 관점이 다르다. 서브도메인은 현실의 업무를 나눈 문제 영역이고, 바운디드 컨텍스트는 그 문제를 해결할 모델의 경계다. 둘이 일대일로 대응할 수도 있지만 항상 그런 것은 아니다.

여러 컨텍스트가 어떤 관계를 맺는지 그린 것이 **컨텍스트 맵**이다. 관계 이름을 모두 외우기보다 어느 모델이 어디까지 유효하고, 경계를 넘을 때 무엇을 주고받는지 분명히 하는 것이 중요하다.

## 도메인 모델을 코드로 옮기면

바운디드 컨텍스트 안에서 사용할 모델을 정했다면 이를 코드로 표현할 차례다. DDD에서는 Entity, Value Object와 Aggregate 같은 구성요소로 모델의 의미와 일관성을 드러낸다.

### 식별자로 이어지는 Entity

**Entity**는 속성이 바뀌어도 같은 대상으로 추적해야 하는 객체다. 주문의 배송지나 상태가 바뀌어도 주문 번호가 같다면 같은 주문이다. 따라서 Entity는 속성 전체보다 식별자가 중요하다.

Entity는 데이터만 보관하지 않고 자신의 상태와 관련된 규칙도 함께 지킬 수 있다.

```java
public class Order {

    private final OrderId id;
    private OrderStatus status;

    public void cancel() {
        if (status == OrderStatus.SHIPPED) {
            throw new IllegalStateException(
                "배송을 시작한 주문은 취소할 수 없습니다."
            );
        }

        status = OrderStatus.CANCELED;
    }
}
```

외부에서 상태를 아무 값으로 바꾸게 두는 대신 `cancel()`을 통해서만 취소하게 만들었다. 코드에서 업무의 의미가 드러나고, 취소할 수 없는 조건도 한곳에서 지킬 수 있다.

### 값으로 의미를 표현하는 Value Object

**Value Object**는 식별자가 아니라 값 자체로 구분하는 객체다. 금액, 주소와 기간이 대표적이다.

금액을 단순한 숫자로만 사용하면 그 숫자가 원화인지 달러인지, 음수를 허용하는지 알기 어렵다. `Money`라는 타입으로 묶으면 값의 의미와 규칙을 함께 표현할 수 있다.

```java
public record Money(BigDecimal amount) {

    public static final Money ZERO =
        new Money(BigDecimal.ZERO);

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

    public Money multiply(BigDecimal rate) {
        return new Money(amount.multiply(rate));
    }
}
```

Value Object는 불변 객체로 만드는 경우가 많다. 기존 값을 바꾸는 대신 새로운 값을 반환하면, 같은 객체를 참조하는 다른 코드에 예상하지 못한 변경이 퍼지는 일을 줄일 수 있다.

### Aggregate가 변경의 범위를 정한다

주문 하나에는 주문 상품, 배송지와 결제 정보처럼 여러 객체가 연결될 수 있다. 이들을 외부에서 자유롭게 수정하면 주문 전체의 규칙을 지키기 어렵다.

**Aggregate**는 한 번의 변경에서 함께 일관성을 지켜야 하는 객체를 묶은 단위다. 외부에서는 **Aggregate Root**를 통해서만 내부 상태를 변경한다. 주문 Aggregate라면 주문이 Root가 되고, 주문 상품의 수량도 주문을 거쳐 바꾼다.

```java
public void changeQuantity(OrderLineId orderLineId, int quantity) {
    OrderLine orderLine = findOrderLine(orderLineId);
    orderLine.changeQuantity(quantity);
    recalculateTotalPrice();
}
```

수량을 바꾸면 총금액도 다시 계산해야 한다는 규칙을 주문이 책임진다. 외부 코드가 `OrderLine`만 직접 수정하면 이 규칙을 놓칠 수 있다.

![Buyer Aggregate와 여러 Entity·Value Object를 포함한 Order Aggregate](/images/posts/domain-driven-design/aggregate.png "Buyer Aggregate와 여러 Entity·Value Object를 포함한 Order Aggregate")

*출처: [Microsoft Learn – Designing a microservice domain model](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/microservice-domain-model)*

Aggregate는 관련 객체를 가능한 한 많이 담는 상자가 아니다. 범위가 커질수록 한 트랜잭션에서 수정할 데이터가 많아지고 다른 작업과 충돌하기 쉬워진다. 반드시 함께 일관성을 지켜야 하는 범위만 포함하는 것이 좋다.

동시에 같은 Aggregate를 수정할 수 있다면 낙관적 잠금이나 비관적 잠금 같은 동시성 제어도 필요하다. DDD가 동시성 문제를 없애주는 것은 아니지만, 어느 범위를 보호해야 하는지는 분명하게 해준다.

## 모델을 저장하고 복잡한 규칙을 처리한다

도메인 객체만으로 애플리케이션을 완성할 수는 없다. Aggregate를 저장하고 다시 불러와야 하며, 하나의 객체에 자연스럽게 넣기 어려운 규칙도 처리해야 한다.

### Repository는 Aggregate 단위로 접근한다

**Repository**는 Aggregate를 저장하고 조회하는 방법을 제공한다. 도메인 코드는 데이터가 MySQL에 있는지, 파일에 있는지보다 필요한 주문을 가져오고 저장할 수 있는지에 관심이 있다.

```java
public interface OrderRepository {

    Optional<Order> findById(OrderId orderId);

    void save(Order order);
}
```

Repository는 보통 Aggregate Root 단위로 만든다. `orders`와 `order_lines` 테이블이 따로 있다고 해서 각각의 Repository를 만들어 변경하는 것이 아니라, 주문 Aggregate의 일관성을 지키는 Root를 통해 접근한다.

실제 JPA 구현은 인프라스트럭처 영역에 둘 수 있다.

```java
@Repository
@RequiredArgsConstructor
class JpaOrderRepository implements OrderRepository {

    private final SpringDataOrderRepository repository;
    private final OrderMapper mapper;

    @Override
    public Optional<Order> findById(OrderId orderId) {
        return repository.findById(orderId.value())
            .map(mapper::toDomain);
    }

    @Override
    public void save(Order order) {
        repository.save(mapper.toEntity(order));
    }
}
```

Repository는 DDD를 하기 위한 필수 인증서가 아니다. ORM을 직접 사용하는 편이 단순한 프로젝트도 있다. 다만 저장 기술의 세부사항이 도메인 모델을 흔들기 시작한다면 유용한 경계가 된다.

### 어느 객체에도 어울리지 않는 규칙은 Domain Service가 맡는다

할인처럼 여러 객체의 정보가 필요하거나 특정 Entity 하나의 책임으로 보기 어려운 규칙도 있다. 이런 도메인 로직은 **Domain Service**로 표현할 수 있다.

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

먼저 규칙과 가장 가까운 Entity나 Value Object에 둘 수 있는지 살펴봐야 한다. Entity가 처리할 수 있는 규칙까지 전부 Domain Service로 옮기면 객체는 데이터만 담고 서비스가 모든 일을 하게 된다.

## 도메인 밖의 코드는 무엇을 맡을까

DDD가 애플리케이션의 모든 코드를 도메인 모델로 만들라는 뜻은 아니다. 사용자의 요청을 받고 작업의 흐름을 조정하며 데이터베이스와 외부 API를 연결하는 코드도 필요하다.

일반적으로 다음과 같이 역할을 나누어 볼 수 있다.

- **표현 영역:** HTTP 요청을 받고 응답을 반환
- **응용 영역:** 하나의 사용 사례와 트랜잭션의 흐름을 조정
- **도메인 영역:** 업무의 개념과 규칙을 표현
- **인프라스트럭처 영역:** 데이터베이스, 메시징과 외부 API 같은 기술을 구현

주문 취소 요청은 다음과 같이 흐를 수 있다.

```text
Controller
    ↓
CancelOrderService
    ↓
Order.cancel()
    ↓
OrderRepository
```

응용 서비스는 주문을 불러오고 취소한 뒤 저장하는 순서를 조정한다. 실제로 주문을 취소할 수 있는지는 도메인 모델이 판단한다.

```java
@Service
@RequiredArgsConstructor
public class CancelOrderService {

    private final OrderRepository orderRepository;

    @Transactional
    public void cancel(OrderId orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow();

        order.cancel();
        orderRepository.save(order);
    }
}
```

응용 서비스가 `배송을 시작한 주문은 취소할 수 없다`는 조건까지 직접 판단하면 같은 규칙이 여러 사용 사례에 흩어질 수 있다. 응용 서비스는 흐름을, 도메인 모델은 업무 판단을 맡는다는 구분이 중요하다.

Repository 계약을 안쪽에 두고 JPA 구현을 바깥쪽에 두면 도메인이 구체적인 저장 기술을 직접 의존하지 않는다. 이것이 **DIP**, 즉 의존성 역전 원칙을 활용하는 방식이다.

![Domain의 Repository 계약을 Infrastructure 구현과 연결하는 구조](/images/posts/domain-driven-design/repository-infrastructure.png "Domain의 Repository 계약을 Infrastructure 구현과 연결하는 구조")

*출처: [Microsoft Learn – Designing the infrastructure persistence layer](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-design)*

DDD가 반드시 네 개의 계층이나 특정 폴더 구조를 요구하는 것은 아니다. 중요한 것은 업무 규칙이 HTTP나 데이터베이스 같은 기술적인 세부사항에 파묻히지 않도록 경계를 정하는 것이다.

## 경계를 넘을 때는 계약이 필요하다

주문과 결제가 서로 다른 바운디드 컨텍스트라면 한쪽의 Entity를 다른 쪽에서 그대로 가져다 쓰지 않는 편이 좋다. 내부 모델을 공유하면 주문 모델의 변경이 결제 코드까지 번질 수 있기 때문이다.

두 컨텍스트는 필요한 데이터와 동작을 공개된 API나 메시지 계약으로 주고받는다. 상대의 모델이 자신의 모델과 다르다면 중간에서 변환한다.

```java
public RefundResult requestRefund(Order order) {
    PaymentResponse response =
        paymentClient.refund(order.paymentId(), order.totalPrice());

    return refundTranslator.toDomain(response);
}
```

`PaymentResponse`는 결제 시스템의 언어다. 이를 주문 컨텍스트가 사용하는 `RefundResult`로 바꾸면 외부 모델이 내부까지 퍼지는 것을 막을 수 있다. 이런 변환 경계를 **부패 방지 계층**이라고 부르기도 한다.

### 이벤트는 이미 일어난 일을 알린다

모든 협력을 동기 API로 처리할 필요는 없다. 어떤 일이 발생했다는 사실을 이벤트로 알릴 수도 있다.

**Domain Event**는 도메인에서 의미 있는 사건을 과거형으로 표현한 것이다.

```java
public record OrderCanceled(
    OrderId orderId,
    Instant occurredAt
) {
}
```

주문 Aggregate가 결제와 쿠폰 시스템을 모두 직접 호출하는 대신 `OrderCanceled`를 발행하면, 필요한 쪽에서 환불이나 쿠폰 복구를 처리할 수 있다. 주문은 자신이 모르는 후속 작업과 덜 결합된다.

이벤트가 곧 비동기를 뜻하는 것은 아니다. 같은 애플리케이션 안에서 동기적으로 처리할 수도 있다. 다른 시스템으로 비동기 전달할 때는 보통 외부에 공개할 **Integration Event**로 변환하며, 이벤트 유실과 중복 처리, DB 트랜잭션과의 불일치까지 함께 설계해야 한다.

### 조회와 변경의 모델을 나눌 수도 있다

도메인 모델은 업무 규칙을 지키며 상태를 변경하는 데 초점을 맞춘다. 반면 조회 화면은 여러 Aggregate의 데이터를 한 번에 조합하거나 통계에 맞는 형태를 요구할 수 있다.

이때 변경 모델과 조회 모델을 분리하는 방식이 **CQRS**다.

```java
public interface OrderQueryRepository {

    Optional<OrderDetailResponse> findDetail(OrderId orderId);

    Page<OrderSummaryResponse> findAll(OrderSearchCondition condition);
}
```

주문 변경은 Aggregate를 통해 규칙을 지키고, 조회는 화면에 필요한 전용 결과를 바로 가져올 수 있다. 별도 데이터베이스나 메시지 브로커가 반드시 필요한 것은 아니다. 같은 애플리케이션과 DB 안에서도 읽기 모델과 쓰기 모델을 코드 수준에서 나눌 수 있다.

CQRS는 단순한 목록 조회까지 무조건 분리하라는 규칙도 아니다. 읽기와 쓰기의 요구가 크게 다르고 하나의 모델로 양쪽을 만족시키기 어려울 때 얻는 이점이 있다. 단순한 시스템에 적용하면 모델과 동기화 지점만 늘어날 수 있다.

## DDD를 전부 적용해야 할까?

DDD의 목적은 Entity, Aggregate와 Event를 가능한 한 많이 사용하는 것이 아니다. 복잡한 업무를 코드와 대화 속에서 계속 이해할 수 있게 만드는 데 있다.

업무 규칙과 예외가 많고, 여러 사람이 같은 용어를 다르게 이해하며, 변경이 여러 영역에 걸쳐 반복된다면 DDD의 모델링이 도움이 된다. 반대로 단순한 관리자 CRUD나 데이터 전달 기능이라면 평범한 구조가 더 적절할 수 있다.

모든 서브도메인에 같은 비용을 쓸 필요도 없다. 회사의 경쟁력을 만드는 핵심 서브도메인에는 충분한 모델링 시간을 쓰고, 일반 서브도메인은 검증된 외부 제품이나 단순한 구현을 선택할 수 있다.

DDD는 MSA도, 헥사고날 아키텍처도, 특정 패키지 구조도 아니다. 다만 도메인의 경계를 분명히 만들다 보면 이러한 아키텍처를 선택하거나 모듈의 경계를 정할 근거가 생긴다.

## 정리

DDD의 흐름을 한 번에 이어 보면 다음과 같다.

```text
도메인을 이해한다
    ↓
서브도메인과 바운디드 컨텍스트로 경계를 나눈다
    ↓
유비쿼터스 언어로 모델을 함께 만든다
    ↓
Entity, Value Object와 Aggregate로 규칙을 표현한다
    ↓
Repository와 Service로 사용 사례를 실행한다
    ↓
API와 이벤트로 다른 경계와 협력한다
```

결국 DDD는 업무를 잘 아는 사람과 개발자가 같은 문제를 같은 언어로 이해하고, 그 이해를 모델과 코드에 계속 반영하는 과정이다. 패턴은 이를 돕는 수단이고, 출발점은 언제나 도메인이다.

## 참고 자료

- [Martin Fowler – Domain Driven Design](https://martinfowler.com/bliki/DomainDrivenDesign.html)
- [Martin Fowler – Bounded Context](https://martinfowler.com/bliki/BoundedContext.html)
- [Martin Fowler – CQRS](https://martinfowler.com/bliki/CQRS.html)
- [Microsoft Learn – Use domain analysis to model microservices](https://learn.microsoft.com/en-us/azure/architecture/microservices/model/domain-analysis)
- [Microsoft Learn – Use tactical DDD to design microservices](https://learn.microsoft.com/en-us/azure/architecture/microservices/model/tactical-domain-driven-design)
- [EventStorming 공식 사이트](https://www.eventstorming.com/)
