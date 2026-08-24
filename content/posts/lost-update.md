+++
title = 'Lost Update는 왜 발생하고 어떤 동시성 제어를 선택해야 하는가'
slug = '4'
aliases = ['/posts/004/']
date = 2026-03-16T21:00:00+09:00
lastmod = 2026-08-11T20:31:31+09:00
draft = false
description = '동시에 같은 데이터를 수정할 때 발생하는 Lost Update부터 단일 UPDATE, 낙관적·비관적 Lock과 Unique Constraint의 선택 기준까지 차례대로 알아봅니다.'
categories = ['데이터베이스']
tags = ['동시성', 'Lost Update', '낙관적 락', '비관적 락']
+++

재고가 10개인 상품에 구매 요청 두 개가 거의 동시에 들어왔다고 해보자. 두 요청이 각각 재고를 1개씩 차감하면 최종 재고는 8개가 되어야 한다.

두 서비스 메서드에 모두 `@Transactional`이 붙어 있다면 한 요청이 끝날 때까지 다른 요청이 기다리지 않을까?

그렇지 않다. `@Transactional`은 한 요청 안에서 실행되는 작업을 하나의 Transaction으로 묶어 Commit하거나 Rollback할 수 있게 한다. 서로 다른 요청에서 시작된 두 Transaction을 자동으로 한 줄로 세우지는 않는다.

여러 요청이 같은 데이터를 거의 동시에 읽고 수정하면 실행 순서에 따라 예상하지 못한 결과가 생길 수 있다. 이런 문제를 동시에 실행되는 순서에 따라 결과가 달라지는 Race Condition이라고 한다. 그중 한쪽의 변경 결과가 다른 변경에 덮여 사라지는 문제가 Lost Update다.

## 두 요청이 모두 성공해도 변경 하나가 사라질 수 있다

두 요청이 같은 재고 10을 읽은 뒤 각자 1을 빼면 다음 순서로 실행될 수 있다.

```text
Transaction A: stock 10 조회
Transaction B: stock 10 조회
Transaction A: 9로 변경 후 Commit
Transaction B: 9로 변경 후 Commit
최종 재고: 9
```

Transaction A도 Commit에 성공했고 Transaction B도 Commit에 성공했다. 데이터베이스 오류도, 애플리케이션 예외도 발생하지 않았다. 하지만 두 번 판매한 재고는 8이 아니라 9가 됐다.

Lost Update가 위험한 이유는 한 요청이 실패해서 결과가 틀리는 것이 아니다. **두 요청이 모두 정상 처리됐다고 생각하는데 한쪽의 변경 결과가 사라진다.** JPA 코드도 값을 읽고, Java에서 수정한 뒤, 다시 저장하는 구조라면 같은 문제가 생길 수 있다.

```java
@Transactional
public void decrease(long productId) {
    Product product = productRepository.findById(productId)
        .orElseThrow();

    product.decrease();
}
```

`save()`를 호출하지 않았는데도 SQL이 실행되는 이유는 변경 감지에 있다. JPA는 Transaction 안에서 조회한 Entity의 값이 바뀌면 Flush 시점에 변경 내용을 찾아 `UPDATE`를 실행할 수 있다.

각 요청은 서로 다른 영속성 컨텍스트와 Transaction에서 Product를 읽는다. 두 요청 모두 자기 관점에서는 재고 10을 읽어 9로 바꿨으므로 충돌을 알아낼 정보가 없다.

Hibernate가 실행하는 SQL을 단순화하면 두 요청 모두 다음과 같은 결과를 만든다.

```sql
UPDATE product
   SET stock = 9
 WHERE id = 1;
```

애플리케이션이 수행한 작업과 데이터베이스가 받은 명령을 나누어 보면 원인이 선명해진다.

```text
애플리케이션이 생각한 작업

현재 stock 조회
→ Java에서 1 차감
→ 변경된 값 저장

데이터베이스가 받은 명령

A: stock에 9 저장
B: stock에 9 저장
```

두 번째 SQL은 “현재 값에서 1을 빼라”가 아니라 “값을 9로 덮어써라”는 명령이다. 데이터베이스는 A와 B가 모두 9를 저장하는 일을 오류로 판단하지 않는다. 두 `UPDATE`가 정상 실행되면서 A의 차감 결과만 사라진다.

여기서 Lost Update를 발견했다고 곧바로 Lock부터 선택할 필요는 없다. 지금 작업이 현재 숫자에서 1을 빼는 것뿐이라면, 애플리케이션에서 값을 읽고 계산하는 단계 자체를 없앨 수 있다.

## 먼저 한 번의 UPDATE로 끝낼 수 있는지 확인한다

현재 값에서 일정량을 더하거나 빼는 작업은 값을 애플리케이션으로 가져오지 않아도 된다. 재고가 남았는지 확인하는 조건과 차감을 하나의 `UPDATE`에 함께 넣을 수 있다.

```sql
UPDATE product
   SET stock = stock - 1
 WHERE id = ?
   AND stock > 0;
```

```text
기존

SELECT stock
→ Java에서 stock - 1
→ UPDATE

개선

UPDATE
→ stock > 0 확인
→ 현재 stock에서 1 차감
```

이제 조건 확인과 계산이 한 SQL 안에서 이루어진다. 데이터베이스는 같은 행을 변경하는 `UPDATE`를 처리할 때 필요한 Lock을 사용하므로, 첫 번째 요청이 10을 9로 만들면 다음 요청은 그 결과인 9에서 다시 1을 뺀다.

```text
A: 현재 stock 10에서 1 차감 → 9
B: A의 변경 이후 stock 9에서 1 차감 → 8
```

이처럼 애플리케이션에서 값을 읽고 계산하는 단계를 거치지 않고 데이터베이스의 한 연산으로 처리하는 방식을 원자적 UPDATE라고 한다.

```java
int updated = productRepository.decreaseStock(productId);

if (updated == 0) {
    throw new OutOfStockException(productId);
}
```

`WHERE stock > 0`을 만족하는 상품만 변경되므로 영향받은 행 수로 성공 여부를 판단할 수 있다.

```text
stock > 0
→ 1행 변경
→ 차감 성공

stock = 0
→ 조건을 만족하는 행 없음
→ 0행 변경
→ 재고 부족
```

다만 상품 ID가 존재하지 않아도 영향받은 행 수는 0이다. 상품이 존재한다는 사실을 앞 단계에서 보장하지 않는다면 “상품 없음”과 “재고 부족”을 어떻게 구분할지도 정해야 한다.

단순한 숫자 증감과 재고 차감에는 이 방법이 짧고 Lock을 잡는 시간도 작다. 하지만 JPQL 일괄 변경이나 직접 작성한 변경 쿼리는 영속성 컨텍스트에 이미 들어 있는 Entity를 자동으로 고쳐주지 않는다. 같은 Transaction에서 Product를 먼저 조회했다면 데이터베이스의 최신 값과 Entity의 값이 달라질 수 있으므로 Clear하거나 다시 조회해야 한다.

모든 변경을 한 SQL로 자연스럽게 표현할 수 있는 것은 아니다. 상품 상태, 쿠폰 조건과 재고를 함께 확인하고 가격을 계산한 뒤 여러 필드를 바꿔야 한다면 애플리케이션에서 값을 읽고 판단하는 과정이 필요하다.

```text
상품 상태 확인
→ 쿠폰 조건 확인
→ 재고 확인
→ 가격 계산
→ 여러 필드 변경
```

이 경우에는 값을 읽은 뒤 저장하기 전까지 다른 요청이 같은 데이터를 수정했는지 확인할 방법이 필요하다.

## 낙관적 Lock은 저장할 때 충돌을 발견한다

값을 읽을 때는 다른 요청을 기다리게 하지 않고, 저장할 때 “내가 읽은 뒤 누군가 수정했는지”만 확인할 수는 없을까?

JPA에서는 Entity에 `@Version` 필드를 두어 이 문제를 해결할 수 있다.

```java
@Entity
public class Product {

    @Id
    private Long id;

    private int stock;

    @Version
    private long version;
}
```

![Entity에 Version 필드를 선언한 낙관적 Lock 예시](/images/posts/lost-update/legacy-01.png "JPA @Version")

Product를 조회할 때 재고와 함께 Version도 읽는다. 저장할 때는 메모리에 있는 Version과 데이터베이스의 Version이 여전히 같은지 확인한다. Hibernate가 만드는 SQL의 핵심은 다음과 같다.

```sql
UPDATE product
   SET stock = ?, version = version + 1
 WHERE id = ?
   AND version = ?;
```

![Version을 조건에 포함한 UPDATE SQL](/images/posts/lost-update/legacy-02.png "Version으로 충돌을 감지하는 UPDATE")

두 요청이 재고 10과 Version 3을 함께 읽었다고 해보자.

```text
A가 읽음
stock = 10, version = 3

B도 읽음
stock = 10, version = 3

A가 먼저 저장
WHERE version = 3
→ UPDATE 성공
→ version = 4

B가 저장
WHERE version = 3
→ 현재 DB의 version은 4
→ 대상 행 없음
→ 충돌 발견
```

Version의 목적은 최신 번호를 관리하는 데 있지 않다. 핵심은 한 문장으로 정리할 수 있다. **내가 읽었던 상태가 저장하는 순간에도 그대로인지 확인한다.** Version이 달라져 `UPDATE`할 행을 찾지 못하면 JPA Provider는 `OptimisticLockException`으로 충돌을 알려준다.

```text
낙관적 Lock

A 실행
B 실행
→ 둘 다 기다리지 않고 진행

저장 시점
→ Version 비교
→ 먼저 저장한 요청만 성공
→ 나중 요청은 충돌로 실패
```

낙관적 Lock은 동시 실행을 미리 막는 기술이 아니라, 실제 충돌이 발생했을 때 나중에 발견하는 기술이다. Lock 대기는 적지만 실패한 요청이 수행한 계산은 버려야 한다.

### 충돌을 발견한 요청은 어떻게 처리할까

충돌을 발견하는 것만으로 업무가 끝나지는 않는다. 실패한 요청을 어떻게 처리할지 정책이 필요하다.

```text
사용자에게 충돌을 알리고 다시 시도하도록 요청

또는

최신 값 재조회
→ 업무 규칙 다시 계산
→ 제한된 횟수만 재시도
```

`OptimisticLockException`이 발생한 Transaction은 Rollback 대상으로 표시된다. 따라서 같은 Transaction 안에서 그대로 다시 저장하지 않고, 전체 업무를 새로운 Transaction에서 처음부터 실행해야 한다. 재시도 횟수와 간격도 제한하지 않으면 충돌이 심할 때 데이터베이스 부하를 더 키울 수 있다.

![낙관적 Lock 충돌을 처리하는 Service 예시](/images/posts/lost-update/legacy-03.png "낙관적 Lock 처리 예시")

충돌이 예외적인 상황이라면 이 방식이 효율적이다. 반대로 같은 행을 자주 다투고, 읽은 최신 상태를 기준으로 뒤의 판단을 반드시 이어가야 한다면 실패 후 재시도보다 처음부터 변경 순서를 세우는 편이 나을 수 있다.

## 비관적 Lock은 처음부터 변경 순서를 세운다

낙관적 Lock은 두 요청을 함께 실행한 뒤 충돌한 요청을 실패시킨다. 충돌이 자주 발생한다면 재시도가 반복되어 오히려 비용이 커질 수 있다.

```text
낙관적 Lock
→ 일단 함께 실행
→ 저장할 때 충돌 확인
→ 실패한 요청은 재시도

비관적 Lock
→ 충돌할 가능성이 크다고 판단
→ 처음부터 한 요청씩 변경
```

이처럼 충돌이 날 가능성을 높게 보고 조회하는 순간부터 변경 순서를 세우는 방식이 비관적 Lock이다.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select p from Product p where p.id = :id")
Optional<Product> findByIdForUpdate(long id);
```

![PESSIMISTIC_WRITE를 선언한 Repository 예시](/images/posts/lost-update/legacy-04.png "JPA 비관적 Lock")

일반 `SELECT`는 값을 읽기만 한다. 반면 `SELECT ... FOR UPDATE`는 “이 행을 곧 변경할 것이므로 다른 Transaction이 먼저 변경하지 못하게 잠가 달라”는 뜻의 조회다. 이렇게 조회하면서 Lock도 함께 거는 방식을 Locking Read라고 한다.

![FOR UPDATE가 포함된 비관적 Lock 실행 SQL](/images/posts/lost-update/legacy-05.png "비관적 Lock 실행 SQL")

Transaction A가 Lock을 얻으면 같은 행을 변경하거나 다시 `FOR UPDATE`로 조회하려는 B는 A가 Commit하거나 Rollback할 때까지 기다린다.

```text
Transaction A: SELECT ... FOR UPDATE → stock 10, Lock 획득
Transaction B: SELECT ... FOR UPDATE → 대기
Transaction A: stock 9 저장 후 Commit, Lock 해제
Transaction B: stock 9 조회 후 실행 재개 → stock 8 저장
```

일반 조회까지 모두 멈추는 것은 아니다. `READ COMMITTED`와 `REPEATABLE READ`에서 InnoDB의 일반 `SELECT`는 MVCC 기반 Consistent Read로 동작하므로 다른 Transaction의 행 Lock을 반드시 기다리지는 않는다. 주로 같은 행을 변경하려는 `UPDATE`, `DELETE`와 다른 Locking Read가 기다리게 된다.

다만 이 설명을 모든 격리 수준에 그대로 적용할 수는 없다. `SERIALIZABLE`에서 Autocommit이 꺼져 있으면 InnoDB는 일반 `SELECT`를 `SELECT ... FOR SHARE`로 바꿔 실행한다.

비관적 Lock은 재시도를 줄이고 최신 상태를 확보한 뒤 판단하기 쉽다는 장점이 있다. 대신 다른 요청이 기다리는 시간이 생기고, Lock Timeout과 Deadlock을 처리해야 한다.

### 여러 행을 다른 순서로 잠그면 서로 기다릴 수 있다

두 Transaction이 여러 행을 반대 순서로 잠그면 다음 상황이 생길 수 있다.

```text
Transaction A
1번 행 Lock 획득
→ 2번 행 Lock 필요

Transaction B
2번 행 Lock 획득
→ 1번 행 Lock 필요

A는 B가 2번 Lock을 풀기를 기다림
B는 A가 1번 Lock을 풀기를 기다림
```

서로 상대가 가진 Lock이 풀리기만 기다려 어느 쪽도 진행하지 못하는 상황이 Deadlock이다. 데이터베이스는 보통 한 Transaction을 Rollback해 교착 상태를 해소하므로 애플리케이션은 실패한 작업을 처리하거나 재시도해야 한다. 여러 행을 잠가야 한다면 가능한 한 항상 같은 순서로 접근하는 것이 Deadlock 가능성을 줄이는 데 도움이 된다.

### Lock을 가진 채 외부 응답을 기다리지 않는다

Transaction이 길어질수록 다른 요청의 대기 시간도 함께 늘어난다.

```text
DB Row Lock 획득
↓
외부 결제 API 호출
↓
5초 동안 응답 대기
↓
그동안 같은 행이 필요한 요청도 Lock 대기
```

외부 시스템의 지연이 데이터베이스 Lock 대기로 번질 수 있다. 비관적 Lock이 필요한 Transaction에는 원자적으로 처리해야 하는 데이터베이스 작업만 남기고, 네트워크 호출이나 파일 처리는 Lock 범위 밖으로 분리하는 것이 좋다.

충돌이 드물면 낙관적 Lock, 충돌이 잦으면 비관적 Lock이라고 기계적으로 결정할 수는 없다. 충돌 빈도뿐 아니라 재시도 비용, Lock을 유지하는 시간, 읽은 상태를 먼저 확보해야 하는지와 사용자가 기다릴 수 있는 시간을 함께 봐야 한다.

지금까지의 방법은 이미 존재하는 행의 변경을 조정했다. 새 행이 중복으로 만들어지는 문제라면 읽기 Lock보다 데이터베이스가 중복 자체를 거부하게 만드는 편이 규칙을 더 직접적으로 표현한다.

## 중복 INSERT는 Unique Constraint로 막는다

지금까지는 이미 존재하는 Product 한 행을 여러 요청이 동시에 수정하는 문제를 다뤘다. 이번에는 “한 사용자는 같은 이벤트에 한 번만 참여할 수 있다”처럼 같은 데이터가 두 번 생성되면 안 되는 경우를 생각해보자.

다음 코드는 저장 전에 참여 기록이 있는지 확인하므로 안전해 보인다.

```java
if (!participantRepository.existsByEventIdAndMemberId(eventId, memberId)) {
    participantRepository.save(new EventParticipant(eventId, memberId));
}
```

문제는 `exists()`와 `INSERT`가 하나의 연산이 아니라는 데 있다. A가 존재 여부를 확인한 뒤 저장하기 전에 B도 같은 조회를 실행할 수 있다.

```text
Request A: 참여 기록 없음 확인
Request B: 참여 기록 없음 확인
Request A: INSERT 시도
Request B: INSERT 시도
```

사전 조회만으로는 두 요청 사이의 틈을 닫을 수 없다. 이 문제에는 Lock보다 데이터베이스가 중복 값을 거부하도록 만드는 Unique Constraint(고유 제약 조건)가 더 직접적이다.

```sql
ALTER TABLE event_participant
    ADD CONSTRAINT uk_event_member UNIQUE (event_id, member_id);
```

```text
Request A: 참여 기록 없음 확인
Request B: 참여 기록 없음 확인
Request A: INSERT 성공
Request B: INSERT 시 Unique Constraint 위반
```

사전 조회는 이미 참여했다는 사실을 미리 알려 사용자 경험을 좋게 만들 수 있다. 하지만 실제 중복 데이터 저장을 막는 최종 안전장치는 데이터베이스의 Unique Constraint다. 애플리케이션은 마지막 경쟁에서 발생한 제약 위반을 “이미 참여한 이벤트입니다”와 같은 업무 오류로 변환해야 한다.

단일 UPDATE, 낙관적 Lock, 비관적 Lock과 Unique Constraint는 서로 대체하는 하나의 기술이 아니다. 어떤 업무 규칙을 보호하려는지에 따라 선택이 달라진다.

## Transaction, Lock과 Constraint의 역할은 다르다

세 가지 모두 데이터의 안전성과 관련 있지만 맡은 역할은 다르다.

| 수단 | 주된 역할 |
| --- | --- |
| Transaction | 하나의 작업 단위를 Commit하거나 Rollback |
| Lock과 Version 검사 | 같은 데이터를 동시에 다루는 순서를 조정하거나 충돌을 발견 |
| Constraint | 데이터베이스에 저장되면 안 되는 값을 거부 |

`@Transactional`이 있다고 해서 모든 동시성 문제가 해결되지 않고, Lock을 사용한다고 해서 중복 데이터의 규칙까지 가장 잘 표현하는 것도 아니다. Transaction 경계를 기본으로 두고 그 안에서 보호할 규칙에 맞는 수단을 추가해야 한다.

## 보호하려는 규칙에 따라 가장 작은 수단을 선택한다

어떤 상황에서도 반드시 지켜져야 하는 업무 규칙이 있다.

```text
재고는 음수가 되면 안 된다.

한 회원은 같은 이벤트에 한 번만 참여할 수 있다.

한 주문의 결제 금액은 중복으로 차감되면 안 된다.
```

이처럼 어떤 실행 순서에서도 반드시 유지해야 하는 규칙을 불변식(Invariant)이라고 한다. 동시성 제어의 목적은 특정 Lock 기술을 사용하는 것이 아니라 이 불변식을 안전하게 지키는 데 있다.

| 상황 | 먼저 검토할 방법 | 함께 확인할 비용과 주의점 |
| --- | --- | --- |
| 숫자 증감과 단순 조건 변경 | 조건을 포함한 단일 `UPDATE` | 복잡한 규칙 표현과 영속성 컨텍스트 동기화 |
| 충돌이 드물고 복잡한 Entity 변경 | 낙관적 Lock | 충돌한 업무의 재시도 |
| 충돌이 잦고 현재 상태 선점이 필요 | 비관적 Lock | Lock 대기, Timeout과 Deadlock |
| 중복 생성 금지 | Unique Constraint | 제약 위반을 업무 오류로 변환 |

선택할 때는 다음 순서로 생각해볼 수 있다. 절대적인 공식이 아니라 불필요하게 무거운 Lock부터 선택하지 않기 위한 출발점이다.

```text
동시성 문제가 있다
↓
한 SQL이나 Constraint로 규칙을 표현할 수 있는가?
↓
YES
→ 단일 UPDATE 또는 Constraint 우선 검토

NO
↓
같은 데이터에서 충돌이 드문가?
↓
YES
→ 낙관적 Lock 검토

NO
↓
현재 상태를 먼저 확보한 뒤 판단해야 하는가?
↓
YES
→ 비관적 Lock 검토

문제가 중복 생성 자체인가?
↓
Unique Constraint 검토
```

Lock을 거는 것이 목적이 아니라, 어떤 상황에서도 지켜져야 하는 업무 규칙을 가장 작은 범위에서 안전하게 지키는 것이 목적이다.

## 정리

- `@Transactional`은 한 요청의 작업을 하나의 Transaction으로 묶지만, 서로 다른 요청을 자동으로 순서대로 실행하지는 않는다.
- Lost Update는 두 요청이 모두 정상 Commit했는데도 한 요청의 변경 결과가 다른 요청에 덮여 사라지는 문제다.
- 단순한 숫자 증감은 조건을 포함한 단일 `UPDATE`로 읽기와 쓰기 사이의 틈을 없앨 수 있는지 먼저 확인한다.
- 낙관적 Lock은 Version이 달라졌는지 저장할 때 확인하며, 충돌한 작업은 새로운 Transaction에서 재시도해야 한다.
- 비관적 Lock은 현재 상태를 먼저 확보하고 변경 순서를 세우지만 Lock 대기, Timeout과 Deadlock 비용이 있다.
- 중복 생성을 막는 규칙은 애플리케이션의 사전 조회보다 Unique Constraint가 최종 안전장치가 된다.

## 참고 자료

- [Jakarta Persistence - Locking and Concurrency](https://jakarta.ee/specifications/persistence/3.2/jakarta-persistence-spec-3.2)
- [Jakarta Persistence - @Version](https://jakarta.ee/specifications/persistence/3.2/apidocs/jakarta.persistence/jakarta/persistence/version)
- [Jakarta Persistence - OptimisticLockException](https://jakarta.ee/specifications/persistence/3.2/apidocs/jakarta.persistence/jakarta/persistence/optimisticlockexception)
- [MySQL - Locking Reads](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking-reads.html)
- [MySQL - Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html)
- [MySQL - Consistent Nonlocking Reads](https://dev.mysql.com/doc/refman/8.4/en/innodb-consistent-read.html)
- [MySQL - Deadlocks in InnoDB](https://dev.mysql.com/doc/refman/8.4/en/innodb-deadlocks.html)
