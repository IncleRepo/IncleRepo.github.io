+++
title = 'Lost Update는 왜 발생하고 어떤 동시성 제어를 선택해야 하는가'
date = 2026-03-16T19:00:00+09:00
lastmod = 2026-08-06T17:54:00+09:00
draft = false
description = '동일한 데이터를 동시에 읽고 수정할 때 발생하는 Lost Update를 재현하고 원자적 SQL, 낙관적 Lock과 비관적 Lock을 비교합니다.'
categories = ['데이터베이스']
tags = ['동시성', 'Lost Update', '낙관적 락', '비관적 락']
+++

Race Condition은 여러 작업이 공유 상태에 접근할 때 실행 순서에 따라 결과가 달라지는 상태다. 데이터베이스에서는 여러 트랜잭션이 같은 행을 읽고 수정할 때 자주 나타난다.

대표적인 문제가 Lost Update다. 두 트랜잭션이 모두 정상 Commit했는데 한쪽의 변경이 다른 쪽에 덮여 사라진다. `@Transactional`을 붙였다는 사실만으로는 이 문제를 막을 수 없다.

`@Transactional`은 각 요청 안의 작업을 하나로 묶어 줄 뿐, 서로 다른 두 트랜잭션을 자동으로 한 줄로 세우지는 않는다. 두 요청이 같은 값을 읽고 각각 계산한 뒤 저장하면 각 트랜잭션은 자기 작업에 실패가 없었다고 판단할 수 있다.

## Lost Update가 생기는 순서

재고가 10개인 상품을 두 요청이 동시에 1개씩 차감한다고 해보자.

```text
Transaction A: stock 10 조회
Transaction B: stock 10 조회
Transaction A: 9로 변경 후 Commit
Transaction B: 9로 변경 후 Commit
최종 재고: 9
```

두 번 판매했으므로 8이 되어야 하지만 마지막 `UPDATE`가 앞선 결과를 덮었다. JPA 코드도 같은 읽기-수정-쓰기 구조라면 동일한 문제가 생긴다.

```java
@Transactional
public void decrease(long productId) {
    Product product = productRepository.findById(productId)
        .orElseThrow();
    product.decrease();
}
```

각 요청은 서로 다른 영속성 컨텍스트와 트랜잭션에서 Entity를 읽는다. 둘 다 자기 관점에서는 정상적인 `stock = 9`를 만들기 때문에 충돌을 감지할 정보가 없다.

Hibernate가 실행하는 SQL을 단순화하면 두 요청 모두 다음과 같은 결과를 만든다.

```sql
UPDATE product
   SET stock = 9
 WHERE id = 1;
```

두 번째 SQL은 “10에서 1을 빼라”가 아니라 “값을 9로 덮어써라”는 명령이다. 데이터베이스는 앞선 요청도 9를 저장했다는 사실을 충돌로 보지 않는다. 두 Update 모두 성공하므로 한 번의 차감 결과가 사라진다.

이 문제를 곧바로 Lock으로 풀기 전에 읽기와 쓰기를 나눌 필요가 있는지부터 확인해야 한다. 현재 값에 일정량을 반영하는 단순한 변경이라면 데이터베이스가 한 문장 안에서 계산하게 만드는 편이 더 짧다.

## 가장 먼저 단일 SQL로 표현할 수 있는지 본다

현재 값에서 일정량을 더하거나 빼는 작업은 애플리케이션에서 먼저 읽지 않아도 된다.

```sql
UPDATE product
   SET stock = stock - 1
 WHERE id = ?
   AND stock > 0;
```

한 SQL 안에서 조건 검사와 변경이 이루어진다. 영향받은 행 수가 0이면 재고 부족으로 판단할 수 있다.

이 SQL은 값을 애플리케이션으로 가져와 계산하지 않는다. 데이터베이스가 현재 값을 기준으로 조건 확인과 차감을 한 문장 안에서 처리한다.

```text
기존 방식: SELECT → Java에서 계산 → UPDATE
원자적 SQL: UPDATE 안에서 조건 확인과 계산
```

읽기와 쓰기 사이의 틈이 없어졌기 때문에 두 요청이 동시에 실행돼도 각 Update는 실행 시점의 현재 stock에서 1을 뺀다. 첫 번째가 10을 9로 만들면 두 번째는 9를 8로 만든다.

```java
int updated = productRepository.decreaseStock(productId);
if (updated == 0) {
    throw new OutOfStockException(productId);
}
```

단순 Counter와 재고 차감에는 이 방법이 가장 짧고 Lock 보유 시간도 작다. 다만 Entity Method와 Callback을 거치지 않고 영속성 컨텍스트도 자동으로 갱신되지 않으므로 같은 트랜잭션에서 이미 읽은 Entity가 있다면 Clear나 재조회가 필요할 수 있다.

하지만 모든 업무 규칙을 한 SQL로 자연스럽게 표현할 수 있는 것은 아니다. 여러 필드와 도메인 규칙을 Entity에서 변경해야 한다면 읽은 상태가 그사이 달라졌는지를 감지하는 방법이 필요하다.

## 낙관적 Lock은 충돌을 발견한다

낙관적 Lock은 동시에 수정하는 경우가 드물다고 가정한다. Entity에 Version을 두고 읽었던 Version이 그대로일 때만 Update한다.

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

Hibernate가 만드는 SQL의 핵심은 다음과 같다.

```sql
UPDATE product
   SET stock = ?, version = version + 1
 WHERE id = ?
   AND version = ?;
```

Transaction A가 Version 3을 4로 바꾸면, 같은 Version 3을 읽었던 Transaction B의 Update는 대상 행을 찾지 못한다. JPA Provider는 이를 `OptimisticLockException`으로 알려 준다.

실행 순서를 따라가면 다음과 같다.

```text
Transaction A: stock 10, version 3 조회
Transaction B: stock 10, version 3 조회
Transaction A: WHERE version = 3으로 UPDATE 성공 → version 4
Transaction B: WHERE version = 3으로 UPDATE → 영향받은 행 0개
Transaction B: OptimisticLockException
```

낙관적 Lock은 동시 실행을 미리 막지 않는다. 두 요청을 그대로 진행시키고 저장 시점에 Version이 달라졌는지 확인한다. 그래서 Lock 대기는 적지만, 실패한 요청이 수행한 계산은 버리고 최신 값을 다시 읽어 재시도해야 할 수 있다.

낙관적 Lock은 기다리는 대신 충돌을 실패로 바꾼다. 따라서 사용자가 다시 시도하게 할지, 서버가 제한된 횟수만 재시도할지 정책이 필요하다. 충돌이 잦으면 재시도 비용이 커질 수 있다.

충돌이 예외적인 상황이라면 이 방식이 효율적이다. 반대로 같은 행을 자주 다투고, 읽은 최신 상태를 기준으로 뒤의 판단을 반드시 이어가야 한다면 실패 후 재시도보다 처음부터 변경 순서를 세우는 편이 나을 수 있다.

## 비관적 Lock은 다른 변경을 기다리게 한다

비관적 Lock은 조회 시점부터 다른 트랜잭션의 변경을 막는다.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select p from Product p where p.id = :id")
Optional<Product> findByIdForUpdate(long id);
```

데이터베이스에서는 일반적으로 `SELECT ... FOR UPDATE`와 같은 Locking Read가 실행된다. Transaction A가 Lock을 잡으면 B는 A가 Commit하거나 Rollback할 때까지 기다린다.

```text
Transaction A: SELECT ... FOR UPDATE → stock 10, Lock 획득
Transaction B: SELECT ... FOR UPDATE → 대기
Transaction A: stock 9 저장 후 Commit, Lock 해제
Transaction B: stock 9 조회 후 실행 재개 → stock 8 저장
```

낙관적 Lock은 충돌을 나중에 발견하고, 비관적 Lock은 충돌할 작업의 순서를 미리 세운다. 어느 방식이든 비용이 없지는 않다. 낙관적 Lock은 재시도 비용이, 비관적 Lock은 대기와 Deadlock 가능성이 있다.

이 방식은 충돌이 잦고 반드시 최신 값을 읽은 뒤 여러 판단을 이어가야 할 때 유용하다. 하지만 Transaction이 길어지면 대기 시간이 늘고, 여러 행을 서로 다른 순서로 잠그면 Deadlock이 생길 수 있다. 외부 API 호출을 Lock을 가진 Transaction 안에서 기다리지 않아야 하는 이유다.

지금까지의 방법은 이미 존재하는 행의 변경을 조정했다. 새 행이 중복으로 만들어지는 문제라면 읽기 Lock보다 데이터베이스가 중복 자체를 거부하게 만드는 편이 규칙을 더 직접적으로 표현한다.

## Unique Constraint도 동시성 제어다

“한 사용자는 한 이벤트에 한 번만 참여할 수 있다”처럼 중복 자체가 금지된 규칙은 Lock보다 데이터베이스 제약이 더 직접적이다.

```sql
ALTER TABLE event_participant
    ADD CONSTRAINT uk_event_member UNIQUE (event_id, member_id);
```

애플리케이션에서 먼저 존재 여부를 조회해도 두 요청이 같은 순간에는 둘 다 없다고 판단할 수 있다. Unique Constraint는 마지막 저장 지점에서 경쟁을 판정한다. 사전 조회는 친절한 오류 메시지를 위한 보조 수단이고, 제약 조건이 최종 안전장치가 된다.

```text
Request A: 참여 기록 없음 확인
Request B: 참여 기록 없음 확인
Request A: INSERT 성공
Request B: INSERT 시 Unique Constraint 위반
```

사전 조회만으로는 두 요청 사이의 틈을 닫지 못한다. 데이터베이스 제약 조건이 있으면 동시에 저장을 시도해도 최종 상태에는 한 행만 남는다. 애플리케이션은 제약 위반을 업무 오류로 변환해 응답하면 된다.

따라서 동시성 제어를 하나의 기술로 통일할 이유는 없다. 업무 불변식이 단순 계산인지, 상태 충돌인지, 변경 순서인지, 중복 생성 금지인지에 따라 가장 작은 수단을 고르면 된다.

## 무엇을 선택할까

| 상황 | 먼저 검토할 방법 |
| --- | --- |
| 숫자 증감과 단순 조건 변경 | 조건을 포함한 단일 `UPDATE` |
| 충돌이 드물고 복잡한 Entity 변경 | 낙관적 Lock |
| 충돌이 잦고 현재 상태 선점이 필요 | 비관적 Lock |
| 중복 생성 금지 | Unique Constraint |

Lock을 거는 것이 목적이 아니라 업무 불변식을 가장 작은 경계에서 지키는 것이 목적이다. 가능하면 데이터베이스가 한 연산으로 판정하게 하고, 여러 단계가 필요할 때 Lock 전략을 선택한다.

## 정리

- `@Transactional`은 원자적 Commit 경계를 만들지만 Lost Update를 자동으로 감지하지 않는다.
- 단순 증감은 조건을 포함한 단일 `UPDATE`가 가장 먼저 검토할 선택지다.
- 낙관적 Lock은 Version 불일치로 충돌을 발견하고 재시도를 요구한다.
- 비관적 Lock은 다른 트랜잭션을 기다리게 하므로 짧은 Transaction이 중요하다.
- 중복 금지 규칙은 Unique Constraint가 최종 안전장치가 된다.

## 참고 자료

- [Jakarta Persistence - Locking and Concurrency](https://jakarta.ee/specifications/persistence/3.2/jakarta-persistence-spec-3.2)
- [Jakarta Persistence - @Version](https://jakarta.ee/specifications/persistence/3.2/apidocs/jakarta.persistence/jakarta/persistence/version)
- [MySQL - Locking Reads](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking-reads.html)
