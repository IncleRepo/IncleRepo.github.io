+++
title = 'MVCC는 락 없이 일관된 읽기를 어떻게 제공하는가'
slug = '5'
aliases = ['/posts/005/']
date = 2026-03-24T21:00:00+09:00
lastmod = 2026-08-10T21:12:37+09:00
draft = false
description = '트랜잭션 격리 수준에서 출발해 InnoDB의 MVCC, Undo Log, Read View와 일반 조회·잠금 조회의 차이를 차례로 살펴봅니다.'
categories = ['데이터베이스']
tags = ['트랜잭션', 'MVCC', 'MySQL', 'InnoDB']
+++

사용자 A가 계좌 잔액을 읽는 동안 사용자 B가 같은 계좌의 잔액을 바꾼다고 생각해 보자. B가 수정 중인 값을 A에게 바로 보여줘야 할까, Commit한 뒤에 보여줘야 할까? A가 한 번 읽은 뒤라면 같은 트랜잭션이 끝날 때까지 처음 값을 유지해야 할까?

모든 트랜잭션을 한 번에 하나씩만 실행하면 이런 고민은 줄어든다. 하지만 요청이 100개 들어오면 99개는 앞의 요청이 끝나기를 기다려야 한다. 실제 데이터베이스는 가능한 한 여러 트랜잭션을 동시에 처리하면서도, 각 트랜잭션이 이해할 수 있는 기준에 맞춰 데이터를 보여줘야 한다.

따라서 다른 트랜잭션의 변경을 내 트랜잭션에서 어느 정도까지 보이게 할지 정하는 기준이 필요하다. 이를 트랜잭션 격리 수준(Isolation Level)이라고 한다. 먼저 격리 수준이 막으려는 문제가 무엇인지 살펴본 뒤, InnoDB가 Lock만으로 모든 읽기를 막지 않고도 일관된 결과를 만드는 방법까지 이어서 알아보자.

## 읽기 결과가 달라지는 세 가지 경우

격리 수준 표부터 외우기보다 동시에 실행된 트랜잭션 사이에서 무엇이 달라지는지 먼저 보는 편이 이해하기 쉽다.

### Commit되지 않은 값을 읽는 경우

```text
Transaction A: balance를 100에서 0으로 변경
               아직 Commit하지 않음

Transaction B: balance 조회
               → 0

Transaction A: Rollback
               → balance는 다시 100
```

B가 읽은 0은 A의 Rollback으로 사라졌다. 결과적으로 B는 데이터베이스에 확정된 적 없는 중간 값을 사용한 셈이다. 이렇게 다른 트랜잭션이 아직 Commit하지 않은 값을 읽는 현상을 Dirty Read라고 한다.

### 같은 행의 값이 달라지는 경우

```text
Transaction A: balance 조회
               → 100

Transaction B: balance를 120으로 변경한 뒤 Commit

Transaction A: 같은 행을 다시 조회
               → 120
```

A는 같은 트랜잭션 안에서 같은 행을 두 번 읽었지만 서로 다른 값을 받았다. 한 번 읽은 결과를 같은 조건으로 다시 얻을 수 없다는 의미에서 Non-repeatable Read라고 한다.

### 조건에 맞는 행의 수가 달라지는 경우

```text
Transaction A: status = 'READY'인 행 조회
               → 3행

Transaction B: READY 상태의 행 1개를 INSERT한 뒤 Commit

Transaction A: 같은 조건으로 다시 조회
               → 4행
```

이번에는 기존 행의 값이 바뀐 것이 아니다. 같은 조건에 맞는 행 하나가 새로 나타나 결과 집합이 달라졌다. 이런 현상을 Phantom Read라고 한다.

세 현상은 다음 질문으로 구분할 수 있다.

```text
Dirty Read
→ Commit되지 않은 값을 보았는가?

Non-repeatable Read
→ 같은 행을 다시 읽었더니 값이 달라졌는가?

Phantom Read
→ 같은 조건으로 다시 읽었더니 행의 집합이 달라졌는가?
```

이제 각 격리 수준이 세 현상을 어디까지 허용하는지 비교할 수 있다.

## 네 가지 격리 수준은 허용하는 현상이 다르다

다음 그림에서는 격리 수준이 높아질수록 어떤 읽기 현상을 더 막는지 확인하면 된다.

![트랜잭션 격리 수준별 Dirty Read, Non-repeatable Read, Phantom Read 비교](/images/posts/transaction-isolation-and-mvcc/legacy-03.svg "격리 수준별 읽기 현상 비교")

그림은 다음처럼 읽을 수 있다.

- Read Uncommitted는 다른 트랜잭션이 Commit하기 전의 값까지 볼 수 있다.
- Read Committed는 Commit된 값만 읽지만, 같은 트랜잭션의 두 조회 결과가 달라질 수 있다.
- Repeatable Read는 같은 트랜잭션의 일반 조회가 같은 읽기 기준을 유지한다.
- Serializable은 트랜잭션을 순서대로 실행한 것에 가까운 결과를 얻도록 동시 실행을 가장 강하게 제한한다.

다만 이 그림은 SQL 표준이 정의한 경계다. 데이터베이스마다 이를 구현하는 방법과 실제 보장 범위는 다를 수 있다. MySQL InnoDB의 기본 격리 수준은 Repeatable Read이며, 일반 조회에는 MVCC를 이용하고 잠금이 필요한 범위 조회에는 Next-Key Lock을 활용해 표준보다 강한 방식으로 Phantom 문제를 다룬다.

격리 수준은 높을수록 무조건 좋은 것이 아니다. 더 많은 이상 현상을 막는 대신 동시에 처리할 수 있는 범위가 줄거나 Lock 대기와 관리 비용이 늘 수 있다. 가장 높은 수준을 고르는 것이 목표가 아니라, 업무에서 어떤 현상을 허용하면 안 되는지 먼저 정해야 한다.

여기까지 보면 한 가지 의문이 남는다. Repeatable Read에서는 B가 100을 120으로 바꾸고 Commit했는데도 A가 어떻게 다시 100을 읽을 수 있을까? 처음 읽은 순간부터 해당 행을 계속 잠그고 있는 것일까?

## InnoDB는 필요한 과거 값을 골라 보여준다

InnoDB의 일반 `SELECT`는 행을 처음 읽은 뒤 계속 잠그는 방식으로 반복 가능한 읽기를 만들지 않는다. 현재 값뿐 아니라 과거 상태를 복원할 수 있게 해두고, 각 트랜잭션에 보여도 되는 버전을 골라 반환한다.

이처럼 하나의 행에 대해 여러 시점의 상태를 다루면서 동시 실행을 제어하는 방식을 Multi-Version Concurrency Control, 줄여서 MVCC라고 한다.

```text
Multi-Version
→ 하나의 행에 대해 여러 시점의 상태를 다룰 수 있음

Concurrency Control
→ 여러 트랜잭션을 동시에 처리하면서 읽기 결과를 조정함
```

MVCC가 동작하려면 두 가지 질문에 답해야 한다.

```text
과거 값은 어디에서 가져오는가?
→ Undo Log

여러 버전 중 어떤 값을 보여주는가?
→ Read View
```

역할이 다른 두 요소를 나눠서 살펴보자.

## Undo Log는 과거 값을 복원할 재료를 남긴다

InnoDB가 행을 수정할 때는 이전 상태를 다시 구성하는 데 필요한 정보를 Undo Log에 남긴다.

```text
현재 행
balance = 120
trx_id = 30

        ↓ 이전 상태 복원

이전 행
balance = 100
trx_id = 20
```

현재 행에는 가장 최근 값이 있다. InnoDB는 이 값뿐 아니라 값을 변경한 트랜잭션의 식별자와 이전 상태를 찾을 정보도 함께 관리한다. 현재 행에서 이전 상태로, 다시 그 이전 상태로 이어지는 연결을 흔히 Version Chain이라고 부른다.

다음 그림에서는 현재 레코드에서 Undo Log를 따라 이전 버전으로 이동하는 흐름에 집중하면 된다.

![Undo Log의 이전 버전이 연결되는 Record History](/images/posts/transaction-isolation-and-mvcc/legacy-02.png "InnoDB Record History")

그렇다고 UPDATE할 때마다 행 전체의 복사본을 영원히 하나씩 보관한다는 뜻은 아니다. InnoDB는 이전 값을 복원하는 데 필요한 정보를 관리하며, 더 이상 어떤 트랜잭션도 과거 버전을 필요로 하지 않으면 정리할 수 있다.

Undo Log가 과거 값을 복원할 수 있게 해주더라도 아무 버전이나 보여줄 수는 없다. 이제 현재 트랜잭션이 볼 수 있는 버전을 판단할 기준이 필요하다.

## Read View는 보여줄 수 있는 버전을 판단한다

Read View는 현재 트랜잭션의 입장에서 어떤 트랜잭션의 변경까지 볼 수 있는지 판단하는 기준이다. 사진처럼 데이터베이스 전체를 복제한 물리적인 사본이 아니라, 읽을 수 있는 데이터의 범위를 정하는 정보에 가깝다.

다음 그림에서는 Read View를 만든 트랜잭션마다 보이는 변경 범위가 달라질 수 있다는 점을 보면 된다.

![트랜잭션별 Read View가 판단하는 가시성 범위](/images/posts/transaction-isolation-and-mvcc/legacy-01.png "Read View")

조회 흐름을 단순화하면 다음과 같다.

```text
현재 행 확인
↓
Read View 기준으로 볼 수 있는 버전인가?
↓
YES → 현재 값 사용

NO
↓
Undo Log를 따라 이전 버전 확인
↓
볼 수 있는 버전을 찾을 때까지 반복
```

정리하면 Undo Log는 과거 값을 복원할 재료이고, Read View는 어떤 버전을 보여줄지 판단하는 기준이다. 일반 `SELECT`는 이 둘을 이용해 다른 트랜잭션의 쓰기가 끝날 때까지 항상 기다리지 않고도 현재 트랜잭션에 맞는 값을 읽을 수 있다.

InnoDB 문서에서는 이런 일반 조회를 Consistent Nonlocking Read라고 설명한다. 이 글에서는 이해를 돕기 위해 일반 `SELECT` 또는 일관된 읽기(Consistent Read)라고 부르겠다. 여기서 일관되다는 말은 언제나 최신이라는 뜻이 아니라, 같은 트랜잭션이 이해할 수 있는 읽기 기준에 맞는다는 뜻이다.

## 실무에서 하나 더: 트랜잭션을 오래 열어두면?

과거 버전은 더 이상 필요하지 않을 때 Purge 과정에서 정리할 수 있다. 하지만 오래된 Read View를 사용하는 트랜잭션이 남아 있다면 상황이 달라진다.

```text
오래된 Read View를 사용하는 트랜잭션 존재
↓
해당 트랜잭션이 과거 버전을 아직 필요로 할 수 있음
↓
InnoDB가 관련 Undo 정보를 바로 정리할 수 없음
↓
Undo 정보와 삭제 표시된 레코드가 오래 유지될 수 있음
```

따라서 읽기 전용 트랜잭션이라도 필요 이상으로 오래 열어두면 Undo 정리를 늦추고 저장 공간과 조회 비용에 영향을 줄 수 있다. “데이터를 바꾸지 않으니 오래 열어도 괜찮다”는 뜻은 아니다.

이제 과거 값을 어떻게 찾는지는 알았다. 다음 질문은 읽기 기준을 언제 만들고 언제까지 유지하는가다. 이 시점이 Read Committed와 Repeatable Read의 결과를 가른다.

## Read Committed와 Repeatable Read는 읽기 기준을 만드는 시점이 다르다

Read Committed도 다른 트랜잭션이 Commit하지 않은 값은 읽지 않는다. Repeatable Read도 마찬가지다. 그런데 왜 같은 트랜잭션에서 두 번 조회한 결과는 달라질 수 있을까?

차이는 일반 `SELECT`가 사용할 Read View를 만드는 주기에 있다.

```text
Read Committed
→ 일반 SELECT마다 새로운 Read View를 만듦

Repeatable Read
→ 첫 번째 일반 SELECT에서 만든 Read View를
   같은 트랜잭션의 다음 일반 SELECT에서도 재사용함
```

Read View가 정한 읽기 기준을 Snapshot이라고도 표현한다. 특정 시점의 데이터베이스 전체를 복사한다는 뜻이 아니라, 이 트랜잭션에서 어떤 변경까지 볼 것인지 정한 기준이다.

Repeatable Read에서 다음 순서로 실행했다고 하자.

```text
Transaction A: SELECT balance
               → 100

Transaction B: balance를 120으로 UPDATE
Transaction B: COMMIT

Transaction A: SELECT balance
               → 100
```

이 시점의 상태를 나누면 다음과 같다.

```text
데이터베이스의 최신 상태
→ 120

Transaction A의 Read View에서 보이는 상태
→ 100
```

A가 Java 객체에 저장해 둔 100을 재사용하는 것은 아니다. 두 번째 SQL도 데이터베이스에서 실행되며, InnoDB가 A의 Read View에 맞는 과거 버전을 찾아 100을 반환할 수 있다.

반면 Read Committed에서는 두 번째 일반 `SELECT`가 새로운 Read View를 만든다. B가 이미 Commit했다면 새로운 읽기 기준에는 B의 변경이 포함되므로 120을 볼 수 있다.

이 설명은 일반 `SELECT`에 해당한다. 지금 읽은 값을 기준으로 곧바로 수정해야 한다면 과거 버전을 보는 것만으로는 부족할 수 있다.

## 일반 SELECT와 SELECT FOR UPDATE의 목적은 다르다

재고를 단순히 화면에 보여주려면 현재 트랜잭션의 기준에 맞는 값만 일관되게 읽으면 된다.

```sql
SELECT stock
  FROM product
 WHERE id = 1;
```

그러나 읽은 재고를 바로 차감하려면 조회와 수정 사이에 다른 트랜잭션이 값을 바꾸는 일을 막아야 할 수 있다.

```sql
SELECT stock
  FROM product
 WHERE id = 1
   FOR UPDATE;
```

두 읽기의 목적을 비교하면 동작 차이가 보인다.

| 읽기 | 목적 | 동작 |
| --- | --- | --- |
| 일반 `SELECT` | 현재 트랜잭션의 기준에 맞는 값을 일관되게 조회 | Snapshot에 맞는 과거 버전을 볼 수 있으며 읽기 Lock을 설정하지 않음 |
| `SELECT ... FOR UPDATE` | 읽은 현재 상태를 기준으로 이어서 변경 | 가장 최근 상태를 기준으로 필요한 Lock을 획득하며 다른 트랜잭션이 기다릴 수 있음 |

Repeatable Read라고 해서 `SELECT ... FOR UPDATE`가 오래된 Snapshot만 읽는 것은 아니다. 일반 Consistent Read는 읽기의 일관성이 목적이고, Locking Read는 지금부터 변경할 현재 행을 확보하는 것이 목적이기 때문이다. `UPDATE`와 `DELETE`도 일반 Consistent Read와 달리 가장 최근 상태를 기준으로 잠금 대상을 찾는다.

이 차이 때문에 한 Repeatable Read 트랜잭션 안에서 일반 조회와 잠금 조회를 섞으면 서로 다른 시점의 상태가 보일 수 있다. MySQL 공식 문서도 이런 혼합은 해석하기 어려운 결과를 만들 수 있다고 주의를 준다. 어떤 값을 읽고 무엇을 변경하려는지 트랜잭션의 목적을 먼저 분명히 해야 한다.

MVCC와 Lock은 둘 중 하나만 선택하는 기술이 아니다.

```text
일관된 값을 조회하고 싶음
→ MVCC를 이용한 일반 SELECT

현재 값을 읽은 뒤 그 상태를 기준으로 변경해야 함
→ Locking Read가 필요할 수 있음
```

제목의 “락 없이”도 모든 Lock이 사라진다는 의미가 아니다. 일반적인 읽기에서 매번 공유 Lock을 잡지 않고도 일관된 값을 보여줄 수 있다는 뜻이다. `UPDATE`, `DELETE`, `SELECT ... FOR UPDATE` 같은 작업에는 여전히 Lock이 사용된다.

## 범위 조회는 빈 구간까지 잠글 수 있다

Primary Key나 Unique Index의 모든 열을 정확히 지정해 한 행을 찾는 경우에는 보통 해당 인덱스 레코드만 잠근다.

```sql
SELECT stock
  FROM product
 WHERE id = 1
   FOR UPDATE;
```

하지만 범위 조건에서는 현재 존재하는 행만 잠그는 것으로 충분하지 않을 수 있다.

```sql
SELECT id, price
  FROM product
 WHERE price BETWEEN 1000 AND 2000
   FOR UPDATE;
```

트랜잭션이 유지되는 동안 다른 트랜잭션이 이 가격 범위에 새 상품을 INSERT하면 같은 조건의 결과 집합이 달라진다. 이를 막기 위해 InnoDB는 검색 과정에서 만난 인덱스 레코드뿐 아니라 레코드 사이의 빈 구간도 잠글 수 있다.

```text
현재 인덱스 값
10           20

다른 트랜잭션이 15를 INSERT
→ 같은 범위 조회의 결과가 달라짐
```

이제 Lock 이름을 붙이면 다음과 같다.

- Record Lock은 실제 인덱스 레코드를 잠근다.
- Gap Lock은 인덱스 레코드 사이 또는 양 끝의 빈 구간에 새 값이 들어오지 못하게 한다.
- Next-Key Lock은 인덱스 레코드와 그 앞의 Gap을 함께 잠근다.

다음 그림에서는 실제 레코드뿐 아니라 레코드 사이 빈 구간도 Lock 대상이 될 수 있다는 점을 보면 된다.

![Record Lock, Gap Lock과 Next-Key Lock의 범위](/images/posts/transaction-isolation-and-mvcc/legacy-04.png "InnoDB의 Record·Gap·Next-Key Lock")

실제로 잡히는 Lock의 범위는 격리 수준, 사용한 인덱스와 검색 조건에 따라 달라진다. InnoDB Repeatable Read에서 Unique Index를 완전한 유일 조건으로 조회하면 발견한 인덱스 레코드만 잠그지만, 범위 검색이나 유일하지 않은 조건은 스캔한 범위에 Gap Lock 또는 Next-Key Lock을 만들 수 있다. Read Committed에서는 일반 검색과 인덱스 스캔의 Gap Lock을 대부분 사용하지 않으며 외래 키와 중복 키 검사 같은 경우에는 예외가 있다.

Lock은 정합성을 지키는 대신 대기와 Deadlock 가능성을 만든다. 따라서 모든 조회를 습관적으로 잠그기보다 현재 상태를 확보하고 이어서 변경해야 하는 구간에 사용해야 한다.

지금까지 살펴본 Read View와 Lock은 모두 트랜잭션이 유지되는 동안 의미가 있었다. 그렇다면 Spring 애플리케이션에서는 이 경계를 어디에서 정할까?

## @Transactional은 데이터베이스 작업의 경계를 정한다

Spring의 `@Transactional`은 어느 작업부터 어느 작업까지 하나의 데이터베이스 트랜잭션으로 묶을지 선언한다.

```java
@Service
@RequiredArgsConstructor
public class TransferService {

    private final AccountRepository accountRepository;

    @Transactional
    public void transfer(long fromId, long toId, long amount) {
        Account from = accountRepository.findByIdForUpdate(fromId);
        Account to = accountRepository.findByIdForUpdate(toId);

        from.withdraw(amount);
        to.deposit(amount);
    }
}
```

위 메서드에서는 출금과 입금이 하나의 Commit 또는 Rollback으로 묶인다. `@Transactional`과 데이터베이스 기능의 역할은 다음처럼 구분할 수 있다.

| 구분 | 역할 |
| --- | --- |
| `@Transactional` | 트랜잭션의 시작과 끝, 전파 방식, 격리 수준과 Rollback 규칙을 선언 |
| Isolation Level | 다른 트랜잭션의 변경을 어느 범위까지 읽을지 결정 |
| MVCC | 일반 조회에 보여줄 버전을 선택해 일관된 읽기를 제공 |
| Lock | 현재 상태를 보호해야 하는 읽기와 쓰기의 충돌을 조정 |

따라서 `@Transactional`을 붙였다고 Lost Update가 자동으로 사라지지는 않는다.

```text
Transaction A: stock 10 조회
Transaction B: stock 10 조회

Transaction A: 1을 빼서 stock 9 저장
Transaction B: 1을 빼서 stock 9 저장

두 트랜잭션 모두 Commit
→ 두 번 차감했지만 최종 재고는 9
```

두 작업은 각각 원자적으로 끝났지만 B가 A의 결과를 덮어썼다. 이 문제에는 원자적 UPDATE, 낙관적 Lock 또는 비관적 Lock처럼 상황에 맞는 별도의 동시성 제어가 필요하다.

## 실무에서 하나 더: readOnly는 무엇을 보장하는가

```java
@Transactional(readOnly = true)
public ProductDetail getProduct(long productId) {
    return productRepository.findDetail(productId);
}
```

`readOnly = true`는 이 트랜잭션이 읽기 전용이라는 의도를 전달하고, 실행 환경이 지원한다면 최적화할 수 있게 하는 힌트다. 하지만 모든 구성에서 쓰기 SQL을 물리적으로 차단하는 안전장치는 아니다. Spring 공식 문서도 실제 트랜잭션 시스템이 이 힌트를 이해하지 못하면 무시할 수 있고, 쓰기 시도가 반드시 실패하는 것은 아니라고 설명한다.

Hibernate의 Flush 처리, JDBC Connection 설정, 읽기 전용 DataSource Routing처럼 실제 효과는 Transaction Manager와 ORM, Driver, 데이터베이스 구성에 따라 달라진다. 그러므로 `readOnly = true`의 효과를 단정하기보다 사용하는 조합에서 실제 SQL과 Connection 동작을 확인해야 한다.

그렇다고 조회 트랜잭션이 불필요하다는 뜻도 아니다. 같은 Snapshot에서 여러 조회를 묶어야 하거나 지연 로딩이 필요한 경우처럼 명확한 이유가 있다면 읽기 전용 트랜잭션은 유용하다. 중요한 것은 조회 메서드마다 관성적으로 붙이는 대신 필요한 경계를 의식하는 것이다.

## 트랜잭션 안에 외부 대기까지 넣지 않는다

Class 전체에 `@Transactional`을 선언하면 데이터베이스 작업과 무관한 외부 API 대기까지 트랜잭션에 포함되기 쉽다.

```java
@Transactional
public void placeOrder(PlaceOrderCommand command) {
    Order order = orderRepository.save(Order.create(command));

    paymentClient.approve(order.getId(), order.getTotalAmount());
    // 외부 결제 API가 5초 지연되면 트랜잭션도 그만큼 오래 유지될 수 있다.

    order.markPaid();
}
```

이 구조에서는 결제 서버를 기다리는 동안 Connection과 트랜잭션이 유지되고, 앞에서 획득한 Lock도 Commit 또는 Rollback까지 오래 남을 수 있다. 외부 호출의 성공과 데이터베이스 변경을 어떻게 조정할지 별도로 설계하고, 데이터베이스에서 원자적으로 처리해야 하는 최소 메서드 범위에 트랜잭션 경계를 두는 편이 좋다.

## 정리

전체 흐름을 다시 연결하면 다음과 같다.

```text
여러 트랜잭션이 동시에 실행됨
↓
서로의 변경을 어디까지 보여줄지 정해야 함
↓
Isolation Level
↓
일반 SELECT에서 일관된 읽기가 필요함
↓
MVCC
↓
Read View로 볼 수 있는 버전을 판단함
↓
필요하면 Undo Log에서 과거 버전을 복원함
↓
Read Committed와 Repeatable Read는
Read View를 만드는 시점이 다름
↓
현재 값을 읽고 바로 변경해야 한다면
Snapshot만으로 부족할 수 있음
↓
Locking Read
↓
Spring에서는 @Transactional로
이 동작이 유지될 트랜잭션 경계를 정함
```

핵심만 다시 나누면 다음과 같다.

- Undo Log는 과거 값을 복원할 재료다.
- Read View는 어떤 버전을 볼 수 있는지 판단하는 기준이다.
- Read Committed는 일반 조회마다 새로운 읽기 기준을 만든다.
- Repeatable Read는 첫 일반 조회에서 만든 읽기 기준을 같은 트랜잭션에서 재사용한다.
- 일반 `SELECT`는 일관되게 읽는 것이 목적이다.
- `SELECT ... FOR UPDATE`는 현재 상태를 확보해 이어서 변경하는 것이 목적이다.
- `@Transactional`은 트랜잭션의 경계를 정하지만 동시성 문제를 자동으로 해결하지는 않는다.

**MVCC는 Lock을 전혀 사용하지 않는 기술이 아니다. 데이터베이스는 가능한 일반 조회를 서로 기다리지 않게 하면서도 각 트랜잭션에 맞는 과거 버전을 보여준다. 다만 현재 상태를 기준으로 변경해야 하는 작업에는 여전히 Lock과 별도의 동시성 제어가 필요할 수 있다.**

## 참고 자료

### 공식 자료

- [MySQL - Consistent Nonlocking Reads](https://dev.mysql.com/doc/refman/8.4/en/innodb-consistent-read.html)
- [MySQL - Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html)
- [MySQL - InnoDB Multi-Versioning](https://dev.mysql.com/doc/refman/8.4/en/innodb-multi-versioning.html)
- [MySQL - InnoDB Locking](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)
- [MySQL - Locking Reads](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking-reads.html)
- [Spring Framework - Using @Transactional](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)
- [Spring Framework API - @Transactional](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/annotation/Transactional.html)

### 국내 기술 블로그

- [카카오페이 기술 블로그 - JPA Transactional 잘 알고 쓰고 계신가요?](https://tech.kakaopay.com/post/jpa-transactional-bri/)
