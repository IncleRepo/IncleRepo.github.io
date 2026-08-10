+++
title = 'JPA를 유지하면서 JDBC를 선택해야 하는 순간'
slug = '10'
aliases = ['/posts/010/']
date = 2026-04-27T00:59:00+09:00
lastmod = 2026-08-06T17:54:00+09:00
draft = false
description = 'JPA의 영속성 컨텍스트와 Dirty Checking 비용을 이해하고 대량 변경·삽입·조회에서 JDBC를 선택하는 기준을 정리합니다.'
categories = ['데이터 접근 설계']
tags = ['JPA', 'JDBC', 'Hibernate', '배치']
+++

JPA와 JDBC를 비교할 때 “어느 쪽이 더 빠른가?”부터 물으면 결론이 흔들리기 쉽다. 두 기술이 맡는 책임이 다르기 때문이다.

JPA는 Entity의 상태와 연관 관계를 관리하고 변경을 SQL로 변환한다. JDBC는 개발자가 SQL과 Parameter Binding, 결과 변환을 직접 통제한다. 일반적인 업무 변경에는 JPA의 생산성이 유리하지만, 수십만 행을 한 번에 처리하는 작업에서는 그 관리 비용이 불필요할 수 있다.

두 기술을 대체 관계로만 볼 필요는 없다. 한 애플리케이션에서도 일반적인 등록·수정은 JPA로 처리하고, 대량 통계 적재처럼 SQL의 모양과 전송 단위가 중요한 구간만 JDBC로 처리할 수 있다. 중요한 것은 기술을 하나로 통일하는 일이 아니라, 해당 작업에서 필요한 책임이 무엇인지 구분하는 일이다.

## Dirty Checking은 공짜가 아니다

트랜잭션 안에서 조회한 Entity는 영속성 컨텍스트가 관리한다. Java 객체의 값을 바꾸면 Hibernate는 Flush 시점에 변경을 감지하고 SQL을 만든다.

```java
@Transactional
public void rename(long memberId, String name) {
    Member member = memberRepository.findById(memberId)
        .orElseThrow();

    member.changeName(name);
}
```

이 방식이 편리한 이유는 Entity 조회, 상태 추적, SQL 생성과 쓰기 지연을 Hibernate가 담당하기 때문이다. 그만큼 영속성 컨텍스트에는 Entity와 Snapshot이 쌓이고, Flush 때 변경 여부를 확인하는 작업도 필요하다.

이 과정을 조금 더 풀어보면 다음과 같다.

```text
1. SELECT로 Member 조회
2. 영속성 컨텍스트에 Member와 최초 상태의 Snapshot 보관
3. Java 객체의 name 변경
4. 트랜잭션을 Commit하기 전에 Flush
5. 현재 상태와 Snapshot 비교
6. 달라진 Entity에 대한 UPDATE 생성
```

Snapshot은 조회 직후의 값을 기억해 두는 사본이다. Hibernate는 개발자가 `update()`를 직접 호출하지 않아도 현재 값과 이 사본을 비교해 변경 사실을 알아낸다. 이것이 Dirty Checking이다.

Entity 한두 개를 다룰 때는 거의 의식할 필요가 없다. 하지만 10만 건을 조회해 하나씩 바꾸면 10만 개의 Entity와 Snapshot이 영속성 컨텍스트에 머물 수 있고, Flush 때 비교할 대상도 그만큼 늘어난다. SQL뿐 아니라 Java Heap과 상태 관리 비용까지 함께 커지는 것이다.

몇 건의 업무 Entity를 변경하는 상황에서는 이 비용보다 코드의 명확성과 도메인 규칙을 Entity에 모을 수 있다는 이점이 크다. 반면 대량 데이터를 전부 Entity로 적재해 한 건씩 수정하면 메모리 사용량과 SQL 실행 횟수가 커질 수 있다.

대상이 커질수록 먼저 물어볼 것은 JPA를 버릴지 여부가 아니다. 같은 변경을 데이터베이스가 한 문장으로 처리할 수 있는지부터 확인하면 불필요한 Entity 조회와 상태 추적을 피할 수 있다.

## 대량 변경은 한 번의 SQL로 표현할 수 있는가

모든 행에 같은 규칙을 적용할 수 있다면 Entity를 하나씩 가져올 필요가 없다.

```java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("""
    update Member m
       set m.status = :status
     where m.lastLoginAt < :threshold
    """)
int updateDormantMembers(MemberStatus status, LocalDateTime threshold);
```

이 쿼리는 데이터베이스가 대상 행을 한 번에 변경한다. 대신 Bulk DML은 이미 영속성 컨텍스트에 올라온 Entity의 상태를 자동으로 고쳐 주지 않는다. 같은 트랜잭션에서 해당 Entity를 계속 사용하면 메모리 상태와 데이터베이스가 달라질 수 있으므로 먼저 Flush할지, 실행 뒤 Clear할지를 검토해야 한다.

예를 들어 영속성 컨텍스트에는 `ACTIVE` 상태인 Member가 있는데 Bulk Update로 데이터베이스 값을 `DORMANT`로 바꿨다고 해보자. Bulk Update는 SQL을 데이터베이스에 바로 실행할 뿐, 메모리에 있는 Member 객체까지 찾아가 수정하지 않는다.

```text
영속성 컨텍스트의 Member.status = ACTIVE
데이터베이스의 member.status    = DORMANT
```

이 상태에서 같은 Entity를 다시 사용하면 오래된 값을 읽게 된다. `flushAutomatically`는 Bulk Query 전에 보류 중인 변경을 먼저 데이터베이스에 반영하고, `clearAutomatically`는 실행 뒤 영속성 컨텍스트를 비워 다음 조회가 데이터베이스를 향하게 한다. 두 옵션의 목적은 서로 다르다.

업무 규칙이 행마다 다르거나 Entity Callback과 연관 관계 변경이 중요하다면 Bulk Update가 항상 정답은 아니다. 이때는 일정 묶음마다 Flush와 Clear를 수행하는 JPA Batch를 사용할 수 있다.

```java
for (int index = 0; index < members.size(); index++) {
    entityManager.persist(members.get(index));

    if ((index + 1) % batchSize == 0) {
        entityManager.flush();
        entityManager.clear();
    }
}
```

여기서 한 SQL로 여러 행을 바꾸는 방식과 여러 SQL을 묶어 보내는 방식이 함께 등장한다. 둘은 모두 대량 처리에 쓰이지만 줄이는 비용이 다르므로 Batch와 Bulk를 먼저 구분해야 한다.

## Batch와 Bulk는 다른 말이다

두 용어는 비슷하게 들리지만 실행 방식이 다르다.

- Batch: 여러 SQL을 모아 네트워크 왕복을 줄인다. SQL은 여러 건이다.
- Bulk: 하나의 SQL이 여러 행을 변경한다.

SQL 모양으로 비교하면 더 분명하다.

```sql
-- Bulk: SQL 한 문장이 여러 행을 변경
UPDATE member
   SET status = 'DORMANT'
 WHERE last_login_at < '2025-01-01';

-- Batch: 값이 서로 다른 SQL 여러 개를 묶어서 전송
UPDATE member SET point = 120 WHERE id = 1;
UPDATE member SET point = 340 WHERE id = 2;
UPDATE member SET point = 275 WHERE id = 3;
```

Batch는 데이터베이스가 실행할 문장 수 자체를 한 개로 만드는 기술이 아니다. 애플리케이션과 데이터베이스 사이를 세 번 왕복할 요청을 한 묶음으로 보내는 데 의미가 있다. 반면 Bulk는 데이터베이스가 한 문장으로 여러 행을 처리한다.

`saveAll()`을 호출했다고 JDBC Batch가 자동으로 보장되는 것도 아니다. Hibernate Batch 설정, 식별자 생성 전략과 SQL 모양이 함께 맞아야 한다. 특히 IDENTITY 방식은 INSERT 시점에 식별자를 얻어야 하므로 투명한 INSERT Batch에 제약이 생길 수 있다.

이 구분을 실제 업무에 적용하려면 변경 대상의 개수보다 각 행에 들어갈 값이 같은지 다른지를 봐야 한다. 값의 모양이 Bulk로 묶을지, 여러 SQL을 Batch로 전송할지를 결정한다.

## 변경 값의 모양이 실행 방식을 결정한다

대량 변경을 최적화할 때는 기술 이름보다 **변경할 값이 몇 종류인가**부터 보는 편이 이해하기 쉽다.

회원 1,000명의 등급을 BASIC, PREMIUM, VIP 세 값 중 하나로 바꾼다고 해보자. 회원마다 Entity를 조회해 Dirty Checking으로 수정하면 최대 1,000번의 Update가 필요하다. 하지만 같은 등급이 될 ID를 먼저 묶으면 다음과 같이 등급별 Bulk Update 세 번으로 표현할 수 있다.

```text
BASIC 대상 ID 목록   → UPDATE ... WHERE id IN (...)
PREMIUM 대상 ID 목록 → UPDATE ... WHERE id IN (...)
VIP 대상 ID 목록     → UPDATE ... WHERE id IN (...)
```

반대로 회원마다 서로 다른 포인트를 저장해야 한다면 한 값으로 묶는 Bulk Update가 어렵다. 이때는 PreparedStatement에 각 Parameter를 넣어 여러 Update를 쌓고 `executeBatch()`로 한 번에 전달할 수 있다. SQL은 여러 건이지만 매번 네트워크 왕복하지 않는다는 점이 핵심이다.

```text
같은 값으로 많은 행 변경 → 조건별 Bulk Update
행마다 다른 값으로 변경 → JDBC Execute Batch
```

한 번에 너무 많이 묶으면 Packet 크기와 메모리 사용량이 커지고 Connection과 Transaction을 오래 점유한다. 따라서 일정한 Chunk로 잘라 처리량과 복구 단위를 제한한다. 결국 “JDBC가 JPA보다 빠르다”보다 **업무 변경을 몇 번의 SQL과 데이터베이스 왕복으로 표현했는가**가 더 중요한 비교 기준이다. 현재 처리량이 충분하다면 병렬화까지 미리 도입할 필요도 없다.

이 판단을 따라가면 JDBC가 필요한 지점도 자연스럽게 좁혀진다. Entity의 수명 주기보다 SQL 모양과 전송 단위를 직접 제어하는 일이 핵심일 때 JDBC의 장점이 커진다.

## Entity 관리가 필요 없으면 JDBC가 자연스럽다

JDBC는 SQL 결과를 Entity 수명 주기와 연결할 필요가 없을 때 강점이 생긴다.

```java
String sql = """
    insert into daily_statistics(stat_date, plant_id, amount)
    values (?, ?, ?)
    """;

jdbcTemplate.batchUpdate(sql, rows, 500, (ps, row) -> {
    ps.setObject(1, row.date());
    ps.setLong(2, row.plantId());
    ps.setBigDecimal(3, row.amount());
});
```

다음과 같은 작업에는 JDBC를 검토할 가치가 크다.

- 대량 INSERT·UPDATE를 일정한 SQL로 반복하는 작업
- 통계, 리포트처럼 Entity가 아니라 조회 전용 결과가 필요한 작업
- 데이터베이스 전용 함수, CTE, Window Function을 적극적으로 사용하는 작업
- 메모리에 Entity를 쌓지 않고 행 단위로 읽어 파일을 생성하는 작업

그러나 Native SQL을 쓴다는 사실만으로 빨라지는 것은 아니다. 같은 실행 계획과 같은 왕복 횟수라면 차이가 작을 수 있다. 실제 병목이 객체 매핑인지, 데이터베이스 실행인지, 네트워크 왕복인지 먼저 측정해야 한다.

예를 들어 실행 시간의 대부분이 인덱스가 없는 Query에서 발생한다면 JPA를 JDBC로 바꿔도 데이터베이스는 같은 Full Scan을 수행한다. 반대로 Query는 단순하지만 수십만 Entity와 Snapshot을 만드는 데 메모리를 많이 쓴다면 JDBC의 행 단위 처리나 Batch가 효과를 낼 수 있다. 도구를 바꾸기 전에 어느 구간의 비용을 줄이려는지 먼저 정해야 한다.

결국 선택을 뒷받침하려면 “JDBC가 빠르더라”는 결과만으로는 부족하다. 어떤 비용이 줄었는지 다시 확인할 수 있도록 비교 조건과 관측값을 함께 남겨야 한다.

## 성능 비교를 어떻게 해야 할까

JPA와 JDBC의 짧은 실행 시간 숫자만 나열하면 다른 환경에서 재현하기 어렵다. 최소한 다음 조건을 함께 기록해야 한다.

- Java, Hibernate, JDBC Driver와 데이터베이스 버전
- 행 수와 컬럼 수, 인덱스 및 데이터 분포
- 로컬 DB인지 원격 DB인지
- Connection Pool과 Batch 크기
- Warm-up과 반복 횟수
- 생성된 SQL, 실행 계획과 실제 처리 행 수

JPA를 없애기 위한 Benchmark가 아니라 병목 구간에 알맞은 도구를 고르기 위한 Benchmark여야 한다.

한 번 실행한 시간만 비교하는 것도 피해야 한다. 첫 실행에는 JVM 최적화, Connection 생성과 데이터베이스 Buffer Cache가 충분히 준비되지 않은 비용이 섞일 수 있다. 여러 번 Warm-up한 뒤 반복 측정하고, 평균뿐 아니라 편차도 함께 보는 편이 안전하다. SQL 로그를 과도하게 출력하면 로그 I/O가 측정 결과에 끼어들 수 있다는 점도 주의한다.

## 정리

- Dirty Checking은 Entity 변경을 편리하게 만들지만 상태 추적과 Flush 비용을 동반한다.
- 같은 변경을 한 SQL로 표현할 수 있다면 Bulk DML이 효율적일 수 있다.
- Bulk DML 뒤에는 영속성 컨텍스트의 오래된 상태를 주의해야 한다.
- Batch는 여러 SQL의 왕복을 줄이고 Bulk는 한 SQL로 여러 행을 처리한다.
- JPA를 기본으로 사용하되 Entity 관리가 필요 없는 대량 작업에는 JDBC를 선택적으로 사용할 수 있다.

## 참고 자료

### 공식 자료

- [Hibernate ORM User Guide - Persistence Context and Flushing](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)
- [Spring Data JPA - Modifying Queries](https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html)

### 국내 기술 블로그

- [카카오페이 기술 블로그 - Spring Batch 애플리케이션 성능 향상을 위한 주요 팁](https://tech.kakaopay.com/post/spring-batch-performance/)
