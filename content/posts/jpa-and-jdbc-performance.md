+++
title = 'JPA를 유지하면서 JDBC를 선택해야 하는 순간'
slug = '10'
aliases = ['/posts/010/']
date = 2026-04-27T20:59:00+09:00
lastmod = 2026-08-10T21:12:37+09:00
draft = false
description = 'JPA와 JDBC의 계층 관계부터 Bulk DML과 Batch의 차이, JDBC를 직접 사용할 기준과 공정한 성능 비교 방법까지 정리합니다.'
categories = ['데이터 접근 설계']
tags = ['JPA', 'JDBC', 'Hibernate', '배치']
+++

회원 10만 명의 상태를 바꿔야 한다고 가정해 보자. JPA로 처리하면 느릴 것 같으니 JDBC로 바꾸면 될까?

이 질문에 답하려면 먼저 둘의 관계를 알아야 한다. JPA와 JDBC는 데이터베이스에 접근하는 서로 독립적인 두 통로가 아니다. 일반적인 Spring 애플리케이션에서 JPA로 작성한 작업도 마지막에는 JDBC를 거쳐 데이터베이스에 전달된다.

## JPA와 JDBC는 같은 계층의 기술이 아니다

JDBC는 Java 애플리케이션이 데이터베이스에 연결하고 SQL을 실행하기 위한 표준 API다. 개발자가 JDBC를 직접 사용하면 실행할 SQL과 Parameter, 결과 변환을 직접 정한다.

JPA는 Entity를 어떤 방식으로 관리할지 정의한 표준 명세다. JPA 자체가 SQL을 실행하는 구현체는 아니며, 실제 동작은 Hibernate 같은 JPA 구현체가 담당한다. Spring Data JPA는 그 위에서 Repository 작성을 편리하게 만들어 주는 도구다.

각 기술의 관계를 정리하면 다음과 같다.

| 계층 | 맡는 역할 |
|---|---|
| Spring Data JPA | Repository 인터페이스와 공통 데이터 접근 기능 제공 |
| JPA | Entity와 영속성 컨텍스트를 다루는 표준 API 정의 |
| Hibernate | JPA를 구현하고 Entity 변경을 SQL로 변환 |
| JDBC | SQL과 Parameter를 JDBC Driver에 전달 |
| JDBC Driver | 데이터베이스와 실제 통신 수행 |

따라서 Spring Data JPA로 `save()`를 호출해도 내부에서는 Hibernate가 SQL을 만든 뒤 JDBC로 실행한다. 반면 `JdbcTemplate`을 사용하면 Hibernate의 Entity 관리를 거치지 않고 애플리케이션 코드가 SQL을 직접 정해 JDBC로 실행한다.

이 글에서 **JPA 방식**은 JPA와 Hibernate의 Entity 관리를 이용하는 접근을, **JDBC 직접 사용**은 `JdbcTemplate`처럼 애플리케이션이 SQL과 결과 변환을 직접 제어하는 접근을 뜻한다. 두 방식 모두 밑에서는 JDBC Driver를 사용한다. 차이는 JDBC를 사용하는지 여부가 아니라 **JDBC 위에 Hibernate의 Entity 관리를 둘 것인지**에 있다.

따라서 먼저 확인할 것은 어느 기술이 더 빠른지가 아니다. 이번 작업에서 Entity의 상태 관리가 필요한지, 변경을 몇 개의 SQL로 표현할 수 있는지부터 살펴봐야 한다. 그 답에 따라 일반적인 Entity 변경, JPA Bulk DML, Hibernate의 JDBC Batch와 JDBC 직접 사용 중 자연스러운 선택지가 달라진다.

## JPA가 편리한 이유부터 본다

회원 한 명의 이름을 변경하는 일반적인 코드는 다음과 같다.

```java
@Transactional
public void rename(long memberId, String name) {
    Member member = memberRepository.findById(memberId)
        .orElseThrow();

    member.changeName(name);
}
```

개발자는 `UPDATE` SQL을 작성하지 않았다. 조회한 객체의 값을 바꾸었을 뿐인데 트랜잭션을 마칠 때 데이터베이스에도 변경이 반영된다. JPA 구현체인 Hibernate가 다음 작업을 대신하기 때문이다.

1. 조회한 `Member`를 관리한다.
2. 처음 조회했을 때의 값을 기억한다.
3. 트랜잭션을 마치기 전에 현재 값과 처음 값을 비교한다.
4. 달라진 값이 있으면 `UPDATE` SQL을 만든다.

JPA API로 조회한 Entity는 단순히 메서드에서 반환된 Java 객체로 끝나지 않는다. 같은 트랜잭션 안에서는 Hibernate가 그 상태를 계속 추적한다. 이렇게 Entity를 관리하는 공간을 **영속성 컨텍스트**라고 한다.

처음 조회했을 때의 값을 기억해 둔 사본은 **Snapshot**이라고 한다. Hibernate가 현재 값과 Snapshot을 비교해 변경된 Entity를 찾아내는 기능이 **Dirty Checking**이다.

Dirty Checking 덕분에 비즈니스 코드는 SQL 작성이나 Parameter Binding보다 “회원 이름을 변경한다”는 의도에 집중할 수 있다. 다만 편의 기능은 아무 일도 하지 않고 얻는 것이 아니다. Hibernate는 Entity와 Snapshot을 메모리에 보관하고, Flush 시점에 두 상태를 비교해야 한다.

회원 몇 명을 변경할 때는 이 비용보다 코드의 명확성과 상태 관리가 주는 이점이 훨씬 크다. 반대로 회원 10만 명을 모두 조회한 뒤 변경하면 상황이 달라진다.

| 처리 규모 | 영속성 컨텍스트에서 일어나는 일 |
|---|---|
| 회원 10명 변경 | Entity와 Snapshot 10개를 관리하고 변경 여부를 확인 |
| 회원 100,000명 변경 | Entity와 Snapshot 100,000개를 보관하고 Flush 때 대량 비교 |

여기에 행마다 `UPDATE`가 실행된다면 SQL 실행 횟수도 함께 늘어난다. Dirty Checking 자체가 나쁜 것이 아니라, 작업 규모에 비해 필요 이상의 Entity를 관리할 때 비용이 커지는 것이다.

그렇다면 대량 처리에서는 Hibernate의 Entity 관리를 버리고 `JdbcTemplate`으로 내려가야 할까? 아직 JPA 경로에서 사용할 수 있는 더 간단한 선택지가 남아 있다. 먼저 같은 변경을 SQL 한 문장으로 표현할 수 있는지 확인해야 한다.

## 같은 변경은 한 번의 SQL로 처리한다

마지막 로그인 시각이 기준보다 오래된 모든 회원을 휴면 상태로 바꾼다고 해보자.

```sql
UPDATE member
   SET status = 'DORMANT'
 WHERE last_login_at < '2025-01-01';
```

이 작업은 회원을 하나씩 읽지 않아도 데이터베이스가 한 번의 SQL로 처리할 수 있다. 이처럼 하나의 SQL로 여러 행을 변경하는 작업을 **Bulk DML**이라고 한다. 여기서 DML은 데이터를 삽입·수정·삭제하는 SQL을 뜻한다.

Bulk DML은 특정 Java 데이터 접근 기술의 기능이 아니라 데이터베이스가 수행하는 SQL 작업이다. Spring Data JPA에서도 JPQL로 Bulk DML을 작성할 수 있으며, Hibernate가 이를 SQL로 변환한 뒤 JDBC를 통해 실행한다.

```java
@Modifying(flushAutomatically = true, clearAutomatically = true)
@Query("""
    update Member m
       set m.status = :status
     where m.lastLoginAt < :threshold
    """)
int updateDormantMembers(MemberStatus status, LocalDateTime threshold);
```

따라서 “대량 처리이므로 `JdbcTemplate`을 사용한다”고 바로 결론 내리면 선택지를 하나 건너뛰게 된다. 모든 행에 같은 규칙을 적용할 수 있다면 먼저 현재 JPA 기반 구조에서 Bulk DML로 충분한지 살펴보는 편이 단순하다.

### Bulk DML 이후에는 Entity 상태를 확인한다

Bulk DML은 데이터베이스의 행을 바로 바꾸지만, 영속성 컨텍스트가 이미 관리 중인 Entity까지 찾아가 값을 고쳐주지는 않는다.

예를 들어 메모리에는 `ACTIVE` 상태인 회원이 있고, Bulk Update가 데이터베이스의 값을 `DORMANT`로 바꾸었다고 해보자.

```text
영속성 컨텍스트의 Member.status = ACTIVE
데이터베이스의 member.status    = DORMANT
```

같은 트랜잭션에서 이 Entity를 다시 사용하면 오래된 값을 읽을 수 있다. 이를 막기 위해 `flushAutomatically`와 `clearAutomatically`를 함께 검토한다.

- `flushAutomatically`: Bulk DML을 실행하기 전에 아직 데이터베이스로 보내지 않은 Entity 변경을 먼저 반영한다.
- `clearAutomatically`: Bulk DML 실행 후 영속성 컨텍스트를 비워, 이후 조회가 데이터베이스의 최신 값을 읽게 한다.

두 옵션은 비슷한 정리 기능이 아니다. Flush는 **보류 중인 변경을 먼저 보존**하고, Clear는 **이미 오래된 상태가 된 관리 객체를 제거**한다. 영속성 컨텍스트를 비우면 모든 Entity가 분리되므로, 이후에도 같은 객체를 계속 사용해야 하는 흐름인지 함께 확인해야 한다.

또한 JPQL Bulk DML은 Entity를 하나씩 거치지 않는다. 따라서 Entity Callback이나 일반적인 낙관적 락 검사를 자동으로 적용할 것이라고 기대해서는 안 된다. 이런 동작이 업무 규칙에 중요하다면 Bulk DML보다 Entity 단위 변경이 더 적절할 수 있다.

결국 Bulk DML은 “행이 많다”는 이유로 선택하는 기능이 아니다. **여러 행의 변경을 동일한 조건과 값으로 표현할 수 있을 때** 가장 잘 맞는다. 행마다 다른 계산이 필요하다면 이제 Batch를 검토할 차례다.

## 행마다 값이 다르면 Batch를 검토한다

회원마다 서로 다른 포인트를 반영해야 한다고 해보자.

```sql
UPDATE member SET point = 120 WHERE id = 1;
UPDATE member SET point = 340 WHERE id = 2;
UPDATE member SET point = 275 WHERE id = 3;
```

각 행의 값이 다르므로 앞선 예처럼 하나의 단순한 `UPDATE`로 합치기 어렵다. 그렇다고 요청할 때마다 애플리케이션과 데이터베이스를 왕복하면 네트워크 비용이 반복된다. 여러 SQL 실행을 일정한 묶음으로 전달해 왕복을 줄이는 방식이 **JDBC Batch**다.

여기서 Bulk와 Batch를 구분해야 한다.

| 구분 | SQL 모양 | 줄이는 비용 |
|---|---|---|
| Bulk | 한 SQL이 여러 행을 처리 | SQL 실행 횟수와 Entity 적재 |
| Batch | 여러 SQL 실행을 묶어서 전달 | 애플리케이션과 DB 사이의 반복 왕복 |

Batch Size가 100이라고 해서 100개의 `UPDATE`가 하나의 SQL로 합쳐지는 것은 아니다. JDBC Driver가 여러 실행 요청을 모아 전달하며, 실제 패킷 구성이나 SQL 재작성 방식은 Driver와 설정에 따라 달라진다. Batch의 핵심은 “SQL 한 문장”이 아니라 **반복 전송을 줄일 기회**를 만드는 데 있다.

### Hibernate도 내부에서 JDBC Batch를 사용할 수 있다

JDBC Batch는 JDBC API가 제공하는 기능이다. 다만 개발자가 `JdbcTemplate`으로 직접 호출해야만 사용할 수 있는 것은 아니다. JPA 구현체인 Hibernate도 자신이 만든 여러 `INSERT`와 `UPDATE`를 JDBC Batch API에 모아 전달할 수 있다.

두 방식의 차이는 Batch 기능 자체보다 누가 SQL을 만들고 JDBC Batch를 호출하는지에 있다.

- **JPA API를 사용하는 방식:** 애플리케이션은 `EntityManager.persist()`나 Entity 변경만 수행한다. Hibernate가 SQL을 만들고 JDBC Batch를 사용하며 Entity 상태도 관리한다.
- **JDBC를 직접 사용하는 방식:** 개발자가 SQL을 작성해 `JdbcTemplate.batchUpdate()`에 전달한다. JdbcTemplate이 JDBC Batch를 사용하며 Entity 상태는 관리하지 않는다.

Entity의 생성 규칙이나 Callback처럼 JPA의 상태 관리가 필요하다면 애플리케이션 코드는 JPA를 유지하면서 Hibernate가 JDBC Batch를 사용하도록 설정할 수 있다.

먼저 `saveAll()`의 역할부터 분리해서 봐야 한다. Spring Data JPA의 `saveAll()`은 전달받은 Entity마다 `save()`를 호출해 `persist()` 또는 `merge()`가 실행되도록 돕는 편의 메서드다. `saveAll()`이 JDBC Batch를 시작하거나 활성화하는 것은 아니다.

```java
@Transactional
public void saveMembers(List<Member> members) {
    memberRepository.saveAll(members);
}
```

이 코드가 JDBC Batch로 실행될 수 있는 이유는 메서드 이름이 `saveAll()`이어서가 아니다. 여러 저장 작업이 같은 트랜잭션과 영속성 컨텍스트에 쌓이고, Hibernate의 Batch 설정과 나머지 조건이 맞으면 Flush할 때 Hibernate가 JDBC Batch를 사용하기 때문이다.

따라서 다음 코드도 바깥의 `@Transactional`로 같은 트랜잭션에 묶여 있다면 같은 조건에서 Batch가 가능하다.

```java
@Transactional
public void saveMembers(List<Member> members) {
    for (Member member : members) {
        memberRepository.save(member);
    }
}
```

반대로 각 `save()`가 서로 다른 트랜잭션에서 즉시 Flush된다면 SQL을 Batch로 모을 수 없다. 핵심은 `saveAll()` 사용 여부가 아니라 **여러 호환 가능한 SQL이 같은 Flush 구간에 모이는가**이다.

#### Hibernate Batch를 활성화하는 설정과 조건

가장 먼저 Hibernate의 JDBC Batch Size를 0보다 큰 값으로 설정해야 한다.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true
```

`hibernate.jdbc.batch_size`가 한 번에 모을 SQL 실행 수를 정하는 핵심 설정이다. `order_inserts`와 `order_updates`는 필수 조건은 아니지만, 같은 형태의 SQL이 연속하도록 실행 순서를 정리해 Batch가 중간에 끊기는 일을 줄일 수 있다. 정렬 자체에도 비용이 있으므로 데이터 규모를 측정해 적용한다.

설정을 켠 뒤에도 다음 조건이 맞아야 Hibernate가 SQL을 실제 Batch로 묶는다.

1. **같은 트랜잭션과 Flush 구간에 여러 작업이 모여야 한다.** Commit이나 `flush()`가 실행되기 전에 여러 Insert·Update가 대기해야 한다.
2. **SQL 형태가 서로 호환되어야 한다.** 같은 PreparedStatement로 실행할 수 있는 SQL끼리 Batch를 구성한다. 서로 다른 Entity의 Insert가 번갈아 나오거나 Update하는 Column 구성이 다르면 Batch가 나뉠 수 있다.
3. **ID 생성 전략이 Batch를 방해하지 않아야 한다.** `GenerationType.IDENTITY`는 Insert 직후 생성된 ID를 받아야 하므로 Hibernate가 Insert를 뒤로 모으기 어렵다.
4. **JDBC Driver가 Batch를 지원해야 한다.** Driver와 설정에 따라 실제 전송이나 SQL 재작성 방식이 달라질 수 있다.

즉 `saveAll()`은 여러 Entity를 한 메서드로 저장하고 같은 트랜잭션에서 처리하기 쉽게 만들어 주지만, Batch의 발생 조건 그 자체는 아니다. 실제 조건은 **Hibernate 설정, 트랜잭션과 Flush 범위, SQL 형태, ID 전략과 JDBC Driver**다.

대량 작업에서는 Batch 활성화와 별개로 영속성 컨텍스트의 크기도 제어해야 한다. 다음 예제의 메서드 이름은 Spring Data의 `saveAll()`과 구분하기 위해 `saveMembersInBatches()`로 작성했다.

```java
@Transactional
public void saveMembersInBatches(List<CreateMemberCommand> commands, int batchSize) {
    for (int index = 0; index < commands.size(); index++) {
        CreateMemberCommand command = commands.get(index);
        entityManager.persist(Member.create(command.name(), command.grade()));

        if ((index + 1) % batchSize == 0) {
            entityManager.flush();
            entityManager.clear();
        }
    }

    entityManager.flush();
    entityManager.clear();
}
```

위 코드는 여전히 JPA API로 Entity를 저장한다. `flush()`가 실행되면 Hibernate가 Entity 변경을 SQL로 만들고, 앞의 조건이 맞으면 그 SQL들을 JDBC Batch API에 모아 전달한다. `clear()`는 처리가 끝난 Entity를 영속성 컨텍스트에서 분리한다. 이를 반복하면 수십만 개의 Entity가 한꺼번에 메모리에 쌓이는 상황을 피할 수 있다.

여기서 `flush()`와 `clear()`는 JDBC Batch 기능을 켜는 설정이 아니다. `flush()`는 대기 중인 SQL을 실행하면서 하나의 Batch 처리 경계를 만들고, `clear()`는 이미 처리한 Entity를 메모리에서 분리한다. 수천 건 정도라면 Commit 때 한 번 Flush해도 Batch가 동작할 수 있지만, 수십만 건을 다룰 때는 영속성 컨텍스트가 너무 커지지 않도록 일정한 간격으로 두 메서드를 호출한다.

```text
EntityManager.persist()
→ Hibernate가 INSERT 생성
→ Hibernate가 JDBC Batch에 INSERT를 모음
→ JDBC Driver가 데이터베이스로 전달
```

Batch가 동작하더라도 Hibernate는 각 Entity에 필요한 SQL을 논리적으로 하나씩 만든다. 따라서 SQL 로그에 `INSERT`가 여러 번 보인다는 사실만으로 Batch가 실패했다고 단정할 수 없다. SQL 로그는 생성된 SQL을 보여줄 뿐, JDBC Driver가 이를 어떤 Batch로 전송했는지까지 충분히 보여 주지 못할 수 있다. Hibernate의 Batch 실행 정보와 JDBC Driver·데이터베이스의 관측 지표를 함께 확인해야 한다.

### Chunk Size와 Batch Size는 목적이 다르다

전체 10만 건을 하나의 트랜잭션으로 처리하면 Connection을 오래 점유하고, 실패했을 때 다시 처리할 범위도 커진다. 그래서 작업 전체를 일정한 단위로 나누어 Commit할 수 있다.

예를 들어 10만 건을 5천 건씩 나누어 처리하면 각 5천 건이 하나의 복구 단위가 된다. 이런 **작업의 처리·Commit 단위**를 Chunk라고 부른다. 반면 Batch Size는 한 번에 묶어 전달할 SQL 실행 수를 뜻한다.

```text
전체 100,000건

Chunk Size 5,000건  → 처리 후 Commit
  ├─ Batch Size 500건 → 전송
  ├─ Batch Size 500건 → 전송
  └─ ...
```

두 값을 같게 둘 수도 있지만 같은 개념은 아니다. Chunk Size는 트랜잭션 길이와 실패 복구 범위를, Batch Size는 전송 효율과 메모리 사용량을 조절한다.

## 변경 값의 모양이 선택지를 좁힌다

대량 변경 방식은 데이터 건수보다 **SQL 모양**을 먼저 보면 이해하기 쉽다. 여기서 SQL 모양은 몇 개의 SQL이 필요한지, 각 행에 적용할 값이 같은지를 의미한다.

### 모든 행에 같은 값을 적용할 때

회원 1,000명을 모두 휴면 상태로 바꾼다면 조건을 이용한 Bulk Update가 자연스럽다.

```sql
UPDATE member
   SET status = 'DORMANT'
 WHERE last_login_at < :threshold;
```

### 몇 종류의 값으로 나눌 수 있을 때

회원 1,000명의 등급이 `BASIC`, `PREMIUM`, `VIP` 중 하나라면 등급별 ID를 모아 Bulk Update를 세 번 실행할 수 있다.

```text
BASIC 대상 ID 목록   → Bulk Update
PREMIUM 대상 ID 목록 → Bulk Update
VIP 대상 ID 목록     → Bulk Update
```

행마다 Entity를 변경하는 것보다 SQL 수를 크게 줄일 수 있다. 다만 `IN` 목록이 지나치게 커지지 않도록 데이터베이스 제한과 실행 계획을 확인해야 한다.

### 모든 행에 다른 값을 적용할 때

회원마다 다른 포인트나 계산 결과를 저장해야 한다면 Bulk로 묶기 어렵다. 이때 Entity 규칙이 필요하면 JPA를 유지한 채 Hibernate의 JDBC Batch를, Entity 관리가 필요 없고 SQL을 직접 제어하는 편이 단순하면 `JdbcTemplate`의 JDBC Batch를 검토한다.

정리하면 건수만 보고 “1만 건은 Batch, 10만 건은 Bulk”처럼 결정할 수 없다. 같은 10만 건이라도 동일한 값이면 Bulk가 잘 맞고, 전부 다른 값이면 Batch가 더 자연스럽다.

## Entity 관리가 필요 없을 때 JDBC를 직접 사용한다

이제 Hibernate를 거치지 않고 JDBC를 직접 사용할 이유가 분명해진다. Spring에서는 JDBC API를 그대로 다루는 대신 반복적인 자원 관리를 줄여 주는 `JdbcTemplate`을 주로 사용할 수 있다. 핵심 질문은 다음과 같다.

> 이 작업에서 Entity를 조회해 상태와 생명 주기를 관리하는 일이 실제로 필요한가?

일별 통계 결과 10만 건을 적재한다고 해보자. 이미 계산을 마친 결과에 날짜, 설비 ID와 발전량만 담아 저장한다면 Lazy Loading이나 Dirty Checking, 연관 관계 변경이 필요하지 않다.

```java
String sql = """
    insert into daily_statistics(stat_date, equipment_id, amount)
    values (?, ?, ?)
    """;

jdbcTemplate.batchUpdate(sql, rows, 500, (statement, row) -> {
    statement.setObject(1, row.date());
    statement.setLong(2, row.equipmentId());
    statement.setBigDecimal(3, row.amount());
});
```

이런 작업은 Entity를 만드는 것보다 실행할 SQL과 묶음 크기를 드러내는 편이 목적에 가깝다. JDBC를 직접 사용하는 방식이 자연스러운 대표 사례는 다음과 같다.

- 계산이 끝난 결과의 대량 Insert·Update
- 통계와 보고서처럼 조회 전용 결과를 만드는 복잡한 SQL
- CTE와 Window Function 등 데이터베이스 기능을 적극 활용하는 조회
- Entity를 쌓지 않고 일정 단위로 읽어 파일을 생성하는 Export

반대로 회원 정보 수정, 주문 상태 전이, 연관 관계 변경처럼 Entity의 상태와 업무 규칙이 중요한 작업은 JPA의 장점이 크다.

| Entity 관리의 가치가 큰 작업 | SQL 제어의 가치가 큰 작업 |
|---|---|
| 회원 정보 변경 | 대량 통계 적재 |
| 주문 상태 전이 | 보고서 조회 |
| 연관 관계 변경 | CTE·Window Function 기반 집계 |
| Entity Callback이 필요한 처리 | 대용량 파일 Export |

조회 전용이라는 이유만으로 반드시 JDBC를 직접 사용할 필요는 없다. JPA의 DTO Projection, QueryDSL, Native Query로도 Entity 전체를 만들지 않고 필요한 결과만 조회할 수 있다. **Entity 관리가 필요 없고, SQL과 전송 단위를 직접 제어할 가치까지 충분할 때** `JdbcTemplate` 같은 직접 접근을 검토하면 된다.

Native SQL을 작성한다는 사실도 JDBC 직접 사용을 뜻하지 않는다. JPA의 Native Query도 결국 Hibernate가 JDBC를 통해 실행한다. SQL 문법을 직접 작성하는지와 Hibernate의 Entity 관리·Mapping을 거치는지는 서로 다른 선택이다.

## JPA 기반 접근과 JDBC 직접 접근을 함께 사용할 때

JPA 기반 접근과 JDBC 직접 접근은 한 애플리케이션 안에서 역할에 따라 함께 사용할 수 있다. 예를 들면 다음과 같이 나눌 수 있다.

- `MemberService`: 일반적인 회원 조회와 상태 변경은 JPA
- `DailyStatisticsJob`: 계산된 통계의 대량 적재는 `JdbcTemplate`
- `ReportRepository`: 조회 성격에 따라 QueryDSL 또는 `JdbcTemplate`

다만 같은 데이터와 트랜잭션에서 두 기술을 섞으면 실행 순서와 상태를 직접 확인해야 한다.

첫째, JPA의 변경 SQL은 Flush 전까지 보류될 수 있다. 바로 뒤의 JDBC 쿼리가 그 변경을 읽어야 한다면 먼저 `entityManager.flush()`가 필요한지 확인한다.

둘째, `JdbcTemplate`이 JDBC로 데이터베이스의 값을 바꾸어도 영속성 컨텍스트의 Entity는 자동으로 갱신되지 않는다. 같은 Entity를 계속 사용한다면 `clear()`나 `refresh()` 또는 트랜잭션 경계 분리를 검토한다.

셋째, 같은 Spring Transaction Manager와 DataSource 아래에서 `EntityManager`와 `JdbcTemplate`을 사용해야 하나의 트랜잭션에 참여할 수 있다. 서로 다른 DataSource라면 같은 `@Transactional`만으로 원자성이 자동 보장된다고 생각해서는 안 된다.

JPA 기반 접근과 JDBC 직접 접근을 함께 사용하면 작업에 맞는 도구를 선택할 수 있지만, 데이터 접근 방식과 테스트 범위도 늘어난다. 성능 차이가 작거나 JPA만으로 충분히 명확하다면 한 가지 방식을 유지하는 편이 단순할 수 있다.

## 성능 비교는 줄어든 비용을 확인하는 과정이다

JPA와 Hibernate를 거친 코드가 5초, `JdbcTemplate`으로 JDBC를 직접 사용한 코드가 2초 걸렸다는 결과만으로 “JDBC가 2.5배 빠르다”고 일반화할 수는 없다. JPA 방식도 내부에서 JDBC를 사용하기 때문이다. 무엇이 줄어들어 빨라졌는지 확인해야 다른 환경에서도 같은 판단을 재현할 수 있다.

예를 들어 두 방식이 모두 Index 없이 같은 조건을 조회한다면 데이터베이스는 똑같이 Full Scan을 수행할 수 있다. 이 경우 Hibernate를 거치던 코드를 `JdbcTemplate`으로 바꾸어도 가장 큰 병목은 남는다.

반대로 50만 건을 Entity로 만들고 Snapshot까지 보관하던 코드를 `JdbcTemplate`으로 일정 단위 처리하도록 바꾸었다면 Heap 사용량과 객체 생성 비용이 크게 줄 수 있다. 이때의 개선 원인은 단순히 “JDBC라서”가 아니라 **Hibernate의 불필요한 Entity 상태 관리를 제거했기 때문**이다.

공정한 비교를 위해서는 최소한 다음 조건을 같게 맞춘다.

- 동일한 Java, Hibernate, JDBC Driver와 데이터베이스 버전
- 동일한 Schema, Index, 데이터 건수와 분포
- 동일한 로컬·원격 환경과 Connection Pool 설정
- 동일한 Transaction 범위와 Batch Size
- 동일한 결과 건수와 업무 규칙

한 번의 실행 시간만 기록하지 않는다. 첫 실행에는 JVM 최적화, Connection 준비와 데이터베이스 Buffer Cache 상태가 영향을 줄 수 있다. 충분히 Warm-up한 뒤 여러 번 측정하고 평균뿐 아니라 편차도 함께 본다.

SQL 로그 역시 필요한 구간에서만 사용한다. 수십만 건의 SQL을 모두 출력하면 로그 I/O가 측정 결과를 왜곡할 수 있다.

측정할 항목은 실행 시간 하나보다 넓다.

- 실제 생성된 SQL 수와 Batch 동작 여부
- 데이터베이스 왕복 횟수
- 실행 계획과 데이터베이스 처리 시간
- Heap 사용량과 GC 횟수
- Connection 점유 시간과 Transaction 길이
- 전체 처리량과 실패 시 재처리 범위

성능 비교는 특정 기술의 우열을 증명하는 일이 아니다. 병목이 데이터베이스의 실행 계획인지, 네트워크 왕복인지, Entity와 Snapshot 관리인지 찾아내는 과정이다.

## 실제 선택 순서

앞의 내용을 하나의 판단 순서로 정리하면 다음과 같다.

1. **일반적인 업무 변경인가?** Entity의 상태와 업무 규칙이 중요하고 규모가 크지 않다면 JPA의 일반적인 변경 방식을 사용한다.

2. **같은 변경을 한 SQL로 표현할 수 있는가?** 가능하다면 JPA Bulk DML을 먼저 검토하고 영속성 컨텍스트 정합성을 처리한다.

3. **행마다 값이나 규칙이 다른가?** Entity의 생명 주기와 Callback이 필요하면 JPA를 유지하고 Hibernate의 JDBC Batch를 검토한다.

4. **Entity 관리 없이 SQL과 전송 단위를 직접 제어하는 편이 명확한가?** 그렇다면 `JdbcTemplate`을 이용한 JDBC Batch나 JDBC 기반 조회가 자연스러운 선택일 수 있다.

5. **정말 이 구간이 병목인가?** 실행 계획, SQL 수, 왕복 횟수, Heap과 처리 시간을 측정한 뒤 최종 선택한다.

## 정리

- JPA의 Dirty Checking은 Entity 변경을 편리하게 만들지만 Entity와 Snapshot 관리 비용을 동반한다.
- 대량 처리라고 곧바로 JDBC를 직접 사용하지 않는다. 같은 규칙은 먼저 현재 JPA 구조에서 Bulk DML로 표현할 수 있는지 확인한다.
- Bulk는 한 SQL로 여러 행을 처리하고, Batch는 여러 SQL 실행을 묶어 반복 왕복을 줄인다.
- JPA 코드가 만든 SQL도 Hibernate를 통해 JDBC Batch로 전달할 수 있지만, `saveAll()` 호출만으로 이를 보장하지는 않는다.
- Entity 상태 관리가 필요하면 JPA와 Hibernate를 유지하고, 그 관리가 필요 없으며 SQL을 직접 제어할 가치가 클 때 `JdbcTemplate`을 선택적으로 사용한다.
- 기술을 바꾸기 전에 실행 계획, SQL 수, 왕복 횟수와 메모리 사용량을 측정해 실제 병목을 찾는다.

JPA와 JDBC 중 하나를 애플리케이션 전체의 정답으로 고를 필요는 없다. JPA 방식도 내부에서 JDBC를 사용하므로 실제 선택은 **Hibernate의 Entity 관리를 이용할지, SQL을 직접 제어할지**에 가깝다. Entity 관리의 가치가 큰 업무에는 JPA와 Hibernate의 장점을 사용하고, 그 관리보다 SQL과 전송 단위의 제어가 중요한 구간에는 Bulk DML, Hibernate의 JDBC Batch와 `JdbcTemplate`을 단계적으로 검토하면 된다. 중요한 것은 어느 기술이 더 빠르다고 외우는 일이 아니라, 현재 작업에서 어떤 비용을 줄이려는지 설명할 수 있는 선택을 하는 것이다.

## 참고 자료

### 공식 자료

- [Jakarta Persistence 3.2 Specification](https://jakarta.ee/specifications/persistence/3.2/jakarta-persistence-spec-3.2.pdf)
- [Hibernate ORM User Guide - Batch Processing](https://docs.hibernate.org/orm/7.1/userguide/html_single/#batch)
- [Spring Data JPA - SimpleJpaRepository](https://github.com/spring-projects/spring-data-jpa/blob/main/spring-data-jpa/src/main/java/org/springframework/data/jpa/repository/support/SimpleJpaRepository.java)
- [Spring Data JPA - Transactionality](https://docs.spring.io/spring-data/jpa/reference/jpa/transactions.html)
- [Spring Framework - Transaction Propagation](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html)
- [Spring Boot - SQL Databases](https://docs.spring.io/spring-boot/reference/data/sql.html)
- [Spring Data JPA - Modifying Queries](https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html)
- [Spring Framework - JDBC Batch Operations](https://docs.spring.io/spring-framework/reference/data-access/jdbc/advanced.html)
- [Java SE - PreparedStatement](https://docs.oracle.com/en/java/javase/26/docs/api/java.sql/java/sql/PreparedStatement.html)

### 국내 기술 블로그

- [카카오페이 기술 블로그 - Spring Batch 애플리케이션 성능 향상을 위한 주요 팁](https://tech.kakaopay.com/post/spring-batch-performance/)
