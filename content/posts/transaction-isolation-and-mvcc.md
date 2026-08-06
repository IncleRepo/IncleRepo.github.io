+++
title = 'MVCC는 락 없이 일관된 읽기를 어떻게 제공하는가'
date = 2026-03-24T19:00:00+09:00
lastmod = 2026-08-06T17:40:00+09:00
draft = false
description = '트랜잭션 이상 현상과 격리 수준을 살펴보고 InnoDB의 Read View와 Undo Log가 일관된 읽기를 제공하는 방식을 설명합니다.'
categories = ['데이터베이스']
tags = ['트랜잭션', 'MVCC', 'MySQL', 'InnoDB']
+++

여러 트랜잭션이 동시에 같은 데이터를 읽고 쓰면 처리량은 높아지지만 서로의 중간 상태가 보일 수 있다. 모든 트랜잭션을 완전히 직렬로 실행하면 문제는 줄어들지만 동시성도 잃는다.

트랜잭션은 여러 데이터베이스 작업을 하나의 논리적인 작업으로 묶는 경계다. 계좌 이체라면 출금과 입금이 모두 성공하거나 모두 취소돼야 한다. 그런데 서로 다른 사용자의 트랜잭션을 항상 한 줄로 세워 실행하면 안전한 대신 기다리는 요청이 많아진다. 데이터베이스는 가능한 한 동시에 실행하면서도 각 트랜잭션이 이해할 수 있는 읽기 결과를 제공해야 한다.

트랜잭션 격리 수준은 이 사이에서 어떤 현상을 허용할지 정한다. InnoDB의 MVCC는 읽기마다 공유 Lock을 잡는 대신 행의 여러 Version 중 현재 트랜잭션이 볼 수 있는 Version을 선택해 일관된 읽기를 제공한다.

## 격리 수준을 이해하기 위한 세 가지 현상

### Dirty Read

다른 트랜잭션이 아직 Commit하지 않은 값을 읽는 현상이다. 값을 쓴 트랜잭션이 Rollback하면 읽은 데이터는 실제로 존재하지 않았던 상태가 된다.

```text
Transaction A: balance 100 → 0으로 변경, 아직 Commit하지 않음
Transaction B: balance 조회 → 0
Transaction A: Rollback → balance는 다시 100
```

B는 최종적으로 존재하지 않게 된 0을 읽었다.

### Non-repeatable Read

한 트랜잭션 안에서 같은 행을 두 번 읽었는데 다른 트랜잭션의 Commit 때문에 값이 달라지는 현상이다.

```text
Transaction A: balance 조회 → 100
Transaction B: balance 120으로 변경 후 Commit
Transaction A: 같은 행 조회 → 120
```

A의 첫 번째 읽기와 두 번째 읽기가 반복되지 않는다는 의미다.

### Phantom Read

같은 조건으로 범위를 두 번 조회했는데 다른 트랜잭션이 행을 추가하거나 삭제해 결과 집합이 달라지는 현상이다.

```text
Transaction A: WHERE status = 'READY' 조회 → 3행
Transaction B: READY 상태의 행 1개 INSERT 후 Commit
Transaction A: 같은 조건 조회 → 4행
```

기존 행의 값이 바뀐 것이 아니라 조건에 맞는 행이 유령처럼 새로 나타난 것처럼 보여 Phantom이라는 이름이 붙었다.

이 현상들은 “동시 요청이 있으면 무조건 생긴다”가 아니라 격리 수준과 읽기 종류에 따라 허용 여부가 달라진다.

세 현상을 기준으로 삼으면 각 격리 수준이 무엇을 막고 무엇을 허용하는지 비교할 수 있다. 먼저 SQL 표준이 정의한 최소 경계를 보고, 그다음 InnoDB가 이를 어떻게 구현하는지 내려가 보자.

## SQL 표준의 네 격리 수준

| 격리 수준 | Dirty Read | Non-repeatable Read | Phantom Read |
| --- | --- | --- | --- |
| Read Uncommitted | 허용 가능 | 허용 가능 | 허용 가능 |
| Read Committed | 방지 | 허용 가능 | 허용 가능 |
| Repeatable Read | 방지 | 방지 | 허용 가능 |
| Serializable | 방지 | 방지 | 방지 |

이 표는 표준이 허용하는 최소 경계를 설명한다. 실제 데이터베이스 구현은 더 강하게 동작할 수 있다. MySQL InnoDB의 기본 격리 수준은 Repeatable Read이며 Consistent Read와 Next-Key Lock을 조합한다.

격리 수준이 높아질수록 무조건 좋은 것은 아니다. 더 강한 일관성을 제공하려면 동시 실행을 제한하거나 더 많은 Lock과 관리 비용이 필요할 수 있다. 애플리케이션은 데이터베이스의 기본값을 맹목적으로 바꾸기보다, 업무에서 허용할 수 없는 현상이 무엇인지 먼저 정해야 한다.

표만으로는 Lock 없이 같은 값을 반복해서 읽을 수 있는 이유가 보이지 않는다. InnoDB의 Consistent Read를 이해하려면 현재 행에서 과거 Version을 복원하는 MVCC 구조를 살펴봐야 한다.

## MVCC가 이전 값을 만드는 방법

InnoDB가 행을 변경하면 이전 값을 복원할 수 있는 정보를 Undo Log에 남긴다. 트랜잭션은 Read View를 기준으로 어떤 Transaction ID의 변경을 볼 수 있는지 판단한다.

```text
현재 행: balance = 120, trx_id = 30
    ↓ Undo Log
이전 행: balance = 100, trx_id = 20
```

현재 행이 아직 보여서는 안 되는 Version이라면 Undo Log를 따라가 과거 Version을 재구성한다. 그래서 일반 `SELECT`는 다른 트랜잭션이 쓰고 있는 행을 읽을 때 반드시 그 쓰기가 끝나기를 기다리지 않아도 된다.

여기서 Read View는 “어떤 트랜잭션의 변경까지 볼 수 있는가”를 기록한 가시성 기준이고, Undo Log는 과거 값을 복원할 재료다. 둘의 역할을 나누면 MVCC 흐름을 이해하기 쉽다.

```text
Read View로 현재 행을 볼 수 있는지 판단
→ 볼 수 있으면 현재 값 반환
→ 아직 볼 수 없으면 Undo Log에서 이전 Version 탐색
→ Read View에 보이는 Version을 찾을 때까지 반복
```

MVCC가 행 전체의 복사본을 무한히 저장한다는 뜻은 아니다. 변경 전 값을 복원할 정보를 Version Chain으로 연결하고, 더 이상 어떤 트랜잭션도 필요로 하지 않는 과거 정보는 정리한다.

Undo Log는 무한히 남지 않는다. 이전 Version을 필요로 하는 오래된 트랜잭션이 사라지면 Purge 대상이 된다. 장시간 열린 트랜잭션은 오래된 Version 정리를 막고 Undo 공간을 늘릴 수 있으므로 읽기 전용 작업도 필요 이상으로 오래 유지하면 안 된다.

과거 Version을 만들 수 있다는 것만으로 어떤 Version을 선택하는지는 설명되지 않는다. 그 선택 시점을 달리하는 것이 Repeatable Read와 Read Committed의 중요한 차이다.

## Repeatable Read와 Read Committed의 Snapshot

InnoDB Repeatable Read에서 일반적인 Consistent Read는 첫 번째 읽기가 만든 Snapshot을 같은 트랜잭션 동안 재사용한다.

```text
Transaction A: SELECT balance → 100
Transaction B: UPDATE balance = 120 → COMMIT
Transaction A: SELECT balance → 100
```

Read Committed에서는 Consistent Read마다 새 Snapshot을 만든다. 따라서 두 번째 `SELECT`는 120을 볼 수 있다.

두 격리 수준의 차이는 “Commit된 데이터만 읽는가”만으로 설명되지 않는다. 둘 다 다른 트랜잭션이 Commit하지 않은 값은 읽지 않지만 Snapshot을 만드는 주기가 다르다.

```text
Read Committed  : SELECT 문마다 새로운 Read View
Repeatable Read : 트랜잭션의 첫 Consistent Read에서 만든 Read View 재사용
```

여기서 말하는 것은 `SELECT ... FOR UPDATE`가 아닌 일반 `SELECT`다. Locking Read와 `UPDATE`, `DELETE`는 최신 상태를 기준으로 Lock을 획득하므로 같은 Repeatable Read 트랜잭션 안에서도 일반 Consistent Read와 서로 다른 시점을 볼 수 있다.

Snapshot은 일관된 조회를 제공하지만 이후의 변경 순서를 예약하지는 않는다. 읽은 값을 기준으로 곧바로 수정해야 한다면 과거 Version을 보는 것만으로는 부족하고 현재 행에 대한 Lock이 필요할 수 있다.

## 읽기 Lock은 언제 필요한가

조회한 값을 바탕으로 곧바로 변경할 때 단순 Snapshot만으로는 다른 트랜잭션의 변경을 막을 수 없다.

```sql
SELECT stock
  FROM product
 WHERE id = 1
 FOR UPDATE;
```

`FOR UPDATE`는 조회한 Index Record에 배타적 Lock을 잡고 트랜잭션이 끝날 때까지 다른 변경을 기다리게 한다. 범위 조건에서는 Record Lock뿐 아니라 Gap 또는 Next-Key Lock이 생길 수 있다.

일반 `SELECT`는 과거 Version을 읽어 쓰기 작업을 기다리지 않을 수 있지만, `FOR UPDATE`는 곧 변경할 현재 행을 선점하려는 읽기다. 따라서 오래된 Snapshot이 아니라 최신 상태를 확인하고 Lock을 획득한다. 같은 `SELECT` 문처럼 보여도 목적과 동작이 다른 것이다.

- Record Lock: 실제로 존재하는 인덱스 레코드를 잠금
- Gap Lock: 인덱스 레코드 사이의 빈 구간을 잠금
- Next-Key Lock: Record Lock과 앞쪽 Gap Lock을 결합

범위 조건에서 Gap까지 잠그는 이유는 다른 트랜잭션이 조건 범위 안에 새 행을 삽입해 결과 집합을 바꾸는 일을 막기 위해서다.

Lock은 정합성을 보호하지만 대기와 Deadlock 가능성을 만든다. 단순 조회에는 MVCC Consistent Read를 사용하고, 읽은 현재 상태를 기준으로 반드시 이어서 변경해야 할 때만 Locking Read를 선택하는 편이 좋다.

이 모든 동작은 트랜잭션이 어디서 시작하고 끝나는지에 따라 유지 시간과 영향 범위가 달라진다. Spring 애플리케이션에서는 그 경계를 주로 `@Transactional`로 선언한다.

## @Transactional은 데이터베이스 작업의 경계를 정한다

Spring의 `@Transactional`은 트랜잭션 경계와 전파, Rollback 규칙을 선언한다. 반면 데이터베이스 격리 수준과 MVCC는 그 경계 안에서 동시 읽기·쓰기를 처리하는 방식이다. 둘은 함께 동작하지만 같은 개념은 아니다.

`@Transactional`을 붙였다고 Lost Update가 자동으로 사라지지는 않는다. 같은 값을 읽은 두 트랜잭션이 각각 계산한 결과를 덮어쓰면 둘 다 정상 Commit할 수 있다. 이 문제에는 원자적 SQL, 낙관적 Lock이나 비관적 Lock처럼 별도의 동시성 제어가 필요하다.

`@Transactional(readOnly = true)`도 단순한 설명용 표시는 아니다. Spring과 Hibernate는 Flush Mode를 조정할 수 있고, DataSource와 Driver 설정에 따라 데이터베이스에 Read-only Transaction 설정과 Commit을 전달할 수 있다. 한 운영 환경에서는 MySQL General Log를 확인했을 때 `SELECT` 외에도 Auto Commit과 Session Transaction 설정, Commit 요청이 추가되는 것을 관찰했다.

그렇다고 모든 조회 트랜잭션을 제거해야 한다는 뜻은 아니다. 같은 Snapshot에서 여러 조회를 묶어야 하거나 지연 로딩이 필요한 경우, 읽기 전용 DataSource Routing에 사용하는 경우에는 명확한 역할이 있다. 반대로 단건 조회마다 관성적으로 붙였다면 실제 SQL과 Connection 점유 시간을 측정할 가치가 있다.

Class 전체에 넓게 선언하면 필요하지 않은 Method와 외부 API 대기까지 트랜잭션에 들어오기 쉽다. 원자적으로 묶어야 하는 데이터베이스 작업을 먼저 찾고 필요한 Method 범위에 경계를 두는 편이 의도를 읽기 쉽다. `@Transactional`은 동시성을 자동으로 해결하는 어노테이션이 아니라 **어떤 데이터베이스 작업을 하나의 Commit 또는 Rollback으로 묶을지 정하는 도구**다.

## 정리

- 격리 수준은 동시 실행에서 어떤 읽기 현상을 허용할지 정한다.
- InnoDB MVCC는 Read View와 Undo Log로 과거 행 Version을 재구성한다.
- Repeatable Read는 같은 트랜잭션의 Consistent Read가 같은 Snapshot을 재사용한다.
- 일반 `SELECT`와 `SELECT ... FOR UPDATE`는 읽는 시점과 Lock 동작이 다르다.
- 오래 열린 트랜잭션은 Undo Log 정리를 지연시킬 수 있다.
- `@Transactional`은 데이터베이스 작업의 경계를 정하며 동시성 문제를 자동으로 해결하지는 않는다.

## 참고 자료

### 공식 자료

- [MySQL - Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html)
- [MySQL - InnoDB Multi-Versioning](https://dev.mysql.com/doc/refman/8.4/en/innodb-multi-versioning.html)
- [MySQL - Locking Reads](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking-reads.html)

### 국내 기술 블로그

- [카카오페이 기술 블로그 - JPA Transactional 잘 알고 쓰고 계신가요?](https://tech.kakaopay.com/post/jpa-transactional-bri/)
