+++
title = 'N+1 문제를 Fetch Join만으로 해결할 수 없는 이유'
slug = '1'
aliases = ['/posts/001/']
date = 2026-02-20T19:00:00+09:00
lastmod = 2026-08-07T17:11:25+09:00
draft = false
description = 'JPA 목록 조회 뒤 Getter에서 추가 쿼리가 실행되는 이유부터 Fetch Join, Batch Fetching, EntityGraph와 DTO Projection의 선택 기준까지 차례대로 알아봅니다.'
categories = ['JPA']
tags = ['JPA', 'Hibernate', 'N+1', 'Fetch Join']
+++

JPA로 목록을 한 번 조회했는데 실행된 SQL을 확인해보면 쿼리가 수십 번 나가는 경우가 있다. 대표적인 원인이 N+1 문제다.

처음 N+1을 접하면 이상하게 느껴질 수 있다. Java 코드에는 추가 쿼리가 보이지 않기 때문이다. 평범한 Getter 호출이 연관된 Entity를 조회하는 SQL로 이어질 수도 있다.

이 글에서는 Getter를 호출했을 뿐인데 왜 SQL이 실행되는지부터 시작한다. 그 원리를 이해한 뒤 EAGER와 Fetch Join의 차이, 일대다 관계에서 생기는 페이징 문제, Batch Fetching과 EntityGraph, DTO Projection을 언제 선택하면 좋은지 차례대로 살펴본다.

## Getter를 호출했는데 왜 SQL이 실행될까

Member가 Team 하나에 소속된다고 해보자. Member Entity는 Team을 다음과 같이 참조한다.

```java
@Entity
public class Member {

    @ManyToOne(fetch = FetchType.LAZY)
    private Team team;
}
```

먼저 Member 20명을 조회한 뒤 각 Member가 속한 Team의 이름을 출력한다.

```java
List<Member> members = memberRepository.findTop20ByOrderByIdAsc();

for (Member member : members) {
    System.out.println(member.getTeam().getName());
}
```

목록을 가져오는 첫 번째 쿼리는 Member만 조회한다.

```sql
SELECT *
  FROM member
 ORDER BY id
 LIMIT 20;
```

문제는 반복문 안의 `member.getTeam().getName()`이다. Java 코드에서는 단순한 Getter 호출처럼 보이지만, 이 순간 Team을 조회하는 SQL이 실행될 수 있다.

앞의 연관 관계에는 `FetchType.LAZY`가 설정되어 있다. `LAZY`는 Member를 조회할 때 Team의 실제 데이터를 바로 가져오지 않고, Team이 정말 필요해지는 시점까지 조회를 미룰 수 있다는 의미다.

그러면 아직 조회하지 않은 Team 자리는 무엇으로 채울까? Hibernate는 나중에 Team이 필요할 때 대신 조회할 수 있는 대리 객체를 둘 수 있다. 이 대리 객체를 **Proxy**라고 한다.

Proxy는 Team의 식별자는 알고 있지만 이름과 같은 실제 데이터는 아직 갖고 있지 않을 수 있다. `getName()`처럼 실제 값이 필요한 메서드를 호출하면 다음 과정이 일어난다.

```text
member.getTeam()
→ Team Proxy 반환

getName()
→ 실제 Team 데이터가 필요한지 확인

아직 조회하지 않았다면
→ team_id로 SELECT 실행

조회한 데이터로 Proxy를 채움
→ name 반환
```

Proxy가 필요한 실제 데이터를 데이터베이스에서 조회해 채우는 과정을 **Proxy 초기화**라고 한다. 결국 Member 목록을 가져오는 쿼리 한 번 뒤에, Team을 사용하는 과정에서 추가 쿼리가 반복될 수 있다.

```text
Member 목록 조회 1번
+ Team 조회 N번
= N+1
```

이것이 N+1 문제다. 연관 관계가 존재한다는 이유만으로 무조건 발생하는 것은 아니다. 처음 쿼리에서 Team을 함께 가져오지 않았고, 이후 `getName()`처럼 Team의 실제 값을 사용할 때 발생한다.

다만 Member 20명을 조회했다고 Team 쿼리가 반드시 정확히 20번 실행되는 것은 아니다. 여러 Member가 같은 Team을 참조한다면 Hibernate가 앞에서 이미 조회한 Team을 다시 사용할 수 있기 때문이다.

Hibernate가 현재 조회하고 변경하는 Entity를 관리하는 공간을 **영속성 컨텍스트**라고 한다. 영속성 컨텍스트는 조회한 Entity를 식별자 기준으로 보관하며, 이 저장 공간을 **1차 캐시**라고 한다. 같은 Team을 다시 요구하면 1차 캐시에 있는 객체를 사용할 수 있으므로 추가 쿼리 수가 줄어든다.

따라서 N은 결과 행 수와 항상 정확히 같다는 뜻이 아니다. 결과가 커지고 서로 다른 연관 Entity를 사용할수록 추가 쿼리도 함께 늘어날 수 있다는 의미에 가깝다.

여기까지 보면 문제의 원인은 Team을 필요할 때마다 뒤늦게 조회한다는 데 있다. 그렇다면 가장 먼저 떠오르는 방법은 “처음부터 Team을 가져오면 되지 않을까?”이다.

## EAGER로 바꾸면 해결될까

JPA에는 연관 데이터를 바로 사용할 수 있도록 가져오는 `FetchType.EAGER`가 있다. `LAZY`가 나중에 필요할 때 조회하는 설정이라면 EAGER는 처음부터 준비하는 설정처럼 보인다.

하지만 EAGER는 연관된 Team을 즉시 사용할 수 있도록 가져오라는 뜻이지, 반드시 하나의 `JOIN` SQL로 가져오라는 뜻은 아니다. 처음 Member를 조회한 뒤 Hibernate가 Team을 채우기 위한 `SELECT`를 추가로 실행할 수도 있다. 이 과정에서도 N+1이 발생할 수 있다.

EAGER와 Fetch Join의 차이를 간단히 정리하면 다음과 같다.

- `EAGER`: 연관 데이터를 즉시 사용할 수 있게 준비하라는 Entity 매핑 설정
- Fetch Join: 이번 쿼리에서 연관 데이터를 `JOIN`으로 함께 가져오라는 조회 방법

모든 연관 관계를 EAGER로 바꾸면 Team이 필요하지 않은 화면에서도 Team을 가져올 수 있다. 어떤 SQL이 만들어질지 쿼리마다 예측하기도 어려워진다. 그래서 기본 연관 관계는 LAZY로 두고, 필요한 쿼리에서 가져올 범위를 명시하는 편이 일반적으로 관리하기 쉽다.

이처럼 한 번의 조회에서 어떤 연관 데이터를 함께 가져올지 정하는 방식을 **Fetch Plan**이라고 한다. EAGER에 맡기는 대신 이번 쿼리의 Fetch Plan을 직접 표현하는 가장 명확한 방법이 Fetch Join이다.

## Fetch Join으로 Team을 함께 가져오기

현재 목록에서 Team이 항상 필요하다면 Member와 Team을 한 번에 조회할 수 있다.

```java
@Query("""
    select m
      from Member m
      join fetch m.team
     order by m.id
    """)
List<Member> findMembersWithTeam(Pageable pageable);
```

실행되는 SQL의 모양은 다음과 같다.

```sql
SELECT m.*, t.*
  FROM member m
  JOIN team t ON t.id = m.team_id
 ORDER BY m.id
 LIMIT 20;
```

Member 하나가 Team 하나를 참조하는 다대일 관계에서는 Member 3명을 조회하면 Join 결과도 기본적으로 3행이다.

```text
Member 1 - Team A
Member 2 - Team B
Member 3 - Team C

Member 3명
→ Join 결과도 3행
```

각 Member 행 옆에 Team Column이 붙을 뿐, Team 때문에 같은 Member가 여러 행으로 늘어나지는 않는다. Member가 Team 하나를 참조하는 다대일 관계나 한 객체가 다른 객체 하나를 참조하는 일대일 관계는 이런 방식으로 조회해도 기준 Entity의 행 수가 그대로 유지된다.

따라서 다대일·일대일 관계의 Fetch Join은 목록 페이징과 비교적 잘 어울린다. 필요한 관계가 여러 개라면 함께 가져오는 것도 일반적으로 가능하다. 다만 Join 대상이 많으면 조회하는 Column 수와 전송하는 데이터 양이 커질 수 있으므로 실제 실행 계획은 확인해야 한다.

그렇다면 필요한 연관 관계를 모두 Fetch Join하면 N+1을 해결할 수 있을까? Team이 여러 Member를 가지는 일대다 관계에서는 SQL 결과의 모양부터 달라진다.

## 일대다 Fetch Join에서는 같은 Team이 반복된다

Team 하나가 Member 세 명을 가진다고 해보자. Team 목록을 조회하면서 Members까지 Join하면 데이터베이스 결과는 다음처럼 펼쳐진다.

```text
Team A - Member 1
Team A - Member 2
Team A - Member 3

Team 1개
→ Join 결과는 3행
```

Java에서는 Team 하나가 여러 Member를 목록으로 가진다. 하지만 데이터베이스의 Join 결과에서는 Member마다 하나의 행이 만들어지므로 Team A의 데이터가 Member 수만큼 반복된다.

앞에서 본 다대일 관계와 지금의 일대다 관계를 SQL 결과 모양으로 비교하면 다음과 같다.

```text
Member → Team과 같은 다대일·일대일
→ Member 수와 Join 결과 행 수가 기본적으로 같음

Team → Members와 같은 일대다
→ Member 수만큼 같은 Team 데이터가 반복될 수 있음
```

일대다 관계를 Fetch Join하는 것이 무조건 나쁜 것은 아니다. 결과 규모가 작고 Member 목록 전체가 꼭 필요하다면 유용하다. 하지만 같은 Team 데이터가 반복되므로 Member가 많을수록 데이터베이스가 읽고 애플리케이션으로 전송하는 행 수도 크게 증가한다.

여기서 페이징까지 필요하면 더 어려운 문제가 생긴다. `LIMIT 20`을 적용했을 때 Team 20개를 잘라야 할까, 아니면 Join 결과 20행을 잘라야 할까?

## 일대다 Fetch Join과 페이징은 왜 충돌할까

Team A에 Member가 30명 있고 Team B에 Member가 1명 있다고 해보자. Join 결과의 처음 20행은 전부 Team A일 수 있다.

```text
Team A - Member 1
Team A - Member 2
...
Team A - Member 20
```

데이터베이스에서 이 결과에 `LIMIT 20`을 적용하면 Team 20개를 가져온 것이 아니다. Team A에 속한 Member 30명 중 20명만 잘라온 것이다. 그러면 실제로는 Member가 30명인 Team A에 20명만 들어 있는 불완전한 목록이 만들어진다.

Hibernate는 이런 불완전한 목록을 만들지 않기 위해 일대다 Fetch Join과 페이징을 함께 사용했을 때 데이터베이스의 `LIMIT`과 `OFFSET`을 적용하지 않을 수 있다. 전체 Join 결과를 가져온 뒤 애플리케이션 메모리에서 필요한 Team만 자르는 방식이다.

![일대다 Fetch Join에 Pageable을 적용했을 때 발생하는 Hibernate 경고](/images/posts/jpa-n-plus-one/legacy-01.png "일대다 Fetch Join과 메모리 페이징 경고")

![일대다 Fetch Join SQL에서 LIMIT과 OFFSET이 사라진 실행 로그](/images/posts/jpa-n-plus-one/legacy-02.png "LIMIT·OFFSET이 적용되지 않은 SQL")

개발 데이터가 작을 때는 정상처럼 보일 수 있다. 하지만 운영 데이터가 커지면 필요하지 않은 행까지 모두 데이터베이스에서 읽고 애플리케이션 메모리에 올리므로 응답 지연과 메모리 사용량 증가로 이어질 수 있다.

Hibernate 설정을 통해 이런 쿼리를 경고로 넘기지 않고 예외로 처리하면 개발 단계에서 발견하기 쉽다.

```yaml
spring:
  jpa:
    properties:
      hibernate.query.fail_on_pagination_over_collection_fetch: true
```

해결 방법 중 하나는 Team ID를 먼저 페이징한 뒤, 두 번째 쿼리에서 해당 Team과 Members를 가져오는 것이다.

```text
1. 조건과 정렬을 적용해 Team ID 20개 조회
2. WHERE team.id IN (...)으로 Team과 Members 조회
3. 첫 번째 쿼리의 Team ID 순서대로 결과 정렬
```

데이터베이스를 두 번 호출하지만 Team 20개의 범위를 먼저 확정할 수 있다. 전체 Join 결과를 메모리에 올리는 것보다 훨씬 안전할 수 있다.

Fetch Join은 쿼리를 한 번으로 줄일 수 있는 강력한 방법이다. 그러나 일대다 관계와 페이징을 함께 사용하거나 여러 일대다 관계를 동시에 Join하는 경우, 너무 많은 Column과 행을 한꺼번에 읽는 경우에는 다른 선택지가 필요하다.

## Fetch Join이 맞지 않을 때 선택할 수 있는 방법

N+1을 줄이는 방법은 크게 세 방향으로 나눌 수 있다.

```text
방법 1. 이번 쿼리에서 필요한 연관 Entity를 함께 가져온다.
→ Fetch Join 또는 EntityGraph

방법 2. LAZY는 유지하되 필요한 연관 Entity 여러 개를 묶어서 조회한다.
→ Batch Fetching

방법 3. Entity 전체가 필요하지 않다면 필요한 Column만 DTO로 조회한다.
→ DTO Projection
```

SQL을 무조건 한 번으로 만드는 것이 목표는 아니다. 현재 조회가 Entity를 필요로 하는지, 연관 관계의 모양과 페이징 여부가 어떤지에 따라 적절한 방법이 달라진다.

### Batch Fetching: 하나씩 조회하지 않고 묶어서 가져오기

LAZY를 유지하면 Team이 필요한 시점에 추가 쿼리가 발생한다. 하지만 Team을 하나씩 조회하는 대신 여러 식별자를 `IN` 조건으로 묶어 가져올 수 있다. 이 방식을 **Batch Fetching**이라고 한다.

```yaml
spring:
  jpa:
    properties:
      hibernate.default_batch_fetch_size: 100
```

```sql
SELECT *
  FROM team
 WHERE id IN (?, ?, ?, ...);
```

Member 100명이 서로 다른 Team을 참조하고 Batch 크기가 20이라면 Team을 100번 따로 조회하는 대신 최대 20개 ID씩 묶어 약 5번 조회할 수 있다.

```text
Member 조회 1번
+ Team 20개씩 조회 5번
= 총 6번
```

Fetch Join처럼 SQL 한 번으로 끝나지는 않는다. 대신 하나의 거대한 Join 결과를 만들지 않으면서 쿼리 수를 크게 줄일 수 있다. 일대다 페이징이나 여러 연관 관계를 함께 다뤄야 할 때 현실적인 절충안이 된다.

Batch 크기를 지나치게 키우면 `IN` 절과 한 번에 읽는 데이터가 커진다. 무조건 큰 값을 사용하지 말고 실제 목록 크기와 데이터베이스의 실행 계획을 확인해야 한다.

### EntityGraph: 이 Repository Method에서는 Team도 같이 가져오기

Fetch Join을 사용하려면 JPQL에 `join fetch`를 직접 작성해야 한다. 단순한 Repository Method에서 Team도 함께 가져오고 싶다면 조회할 연관 관계를 선언하는 방법도 있다.

```java
@EntityGraph(attributePaths = "team")
List<Member> findTop20ByOrderByIdAsc();
```

이처럼 Repository Method가 가져올 연관 범위를 선언하는 기능이 **EntityGraph**다. JPQL 문자열을 직접 작성하지 않고도 “이 메서드에서는 Team까지 필요하다”는 의도를 표현할 수 있다.

EntityGraph도 마법처럼 SQL 결과의 구조를 바꾸는 별개의 해결책은 아니다. 다대일·일대일 관계를 포함하면 비교적 단순하지만 일대다 관계를 포함하면 Fetch Join과 마찬가지로 같은 Team 데이터가 반복될 수 있다. 따라서 일대다 관계와 페이징을 함께 사용할 때는 결과 행 증가를 똑같이 확인해야 한다.

### DTO Projection: 필요한 Column만 바로 조회하기

화면에서 Member ID, Member 이름, Team 이름 세 개만 필요하다고 해보자. 이 값을 보여주기 위해 Member와 Team Entity 전체를 조회해야 할까?

필요한 Column만 골라 DTO로 바로 조회할 수 있다. 이를 **DTO Projection**이라고 한다.

```java
public record MemberSummary(
    long memberId,
    String memberName,
    String teamName
) {}
```

```java
@Query("""
    select new com.example.member.MemberSummary(
        m.id,
        m.name,
        t.name
    )
      from Member m
      join m.team t
     order by m.id
    """)
List<MemberSummary> findSummaries(Pageable pageable);
```

쿼리가 `MemberSummary`에 필요한 세 Column을 처음부터 가져온다. 반환된 DTO에는 LAZY Proxy가 없으므로 이후 값을 읽어도 연관 Entity를 가져오는 추가 SQL이 발생하지 않는다. 조회하는 Column과 데이터 양도 줄일 수 있어 읽기 전용 API에 잘 맞는다.

반면 DTO는 Entity가 아니므로 변경 감지로 저장할 수 없고 Member의 도메인 로직을 실행하는 용도에도 맞지 않는다. 화면이나 API 응답별 DTO가 늘어날 수도 있다. Entity가 필요한 업무 로직과 값만 필요한 조회를 구분해서 사용해야 한다.

## 실무에서 놓치기 쉬운 경우

기본 원리는 웹 API의 목록 조회뿐 아니라 페이징과 Join을 사용하는 다른 코드에도 그대로 적용된다. 특히 Spring Batch와 `distinct`는 Fetch Join 문제를 해결했다고 착각하기 쉬운 지점이다.

### Spring Batch의 PagingItemReader도 페이징한다

일대다 Fetch Join과 페이징 문제는 Controller가 `Pageable`을 받을 때만 발생하지 않는다. Repository Method에 `Pageable`이 보이지 않아도 Spring Batch의 PagingItemReader가 일정 크기로 결과를 나누어 읽으면 페이징 쿼리가 된다.

```text
Batch가 Team 100개를 읽으려고 함
→ Team과 Members를 Fetch Join
→ DB 결과는 Member 수만큼 Team 행이 반복됨
→ Team 기준으로 LIMIT을 적용하기 어려움
→ 전체 Join 결과를 읽은 뒤 메모리에서 Team 100개 선택
```

Batch라고 해서 조회 원리가 달라지는 것은 아니다. 페이징이 Reader 설정 안에 숨어 있을 뿐이다. 해결할 때는 Team ID를 먼저 일정 크기로 조회하고, 두 번째 쿼리에서 `WHERE team.id IN (...)`으로 필요한 연관 데이터를 가져오는 방법을 검토할 수 있다.

### distinct가 Join 결과 행 자체를 없애는 것은 아니다

일대다 Fetch Join에 `distinct`를 붙이면 Hibernate가 같은 Team Entity를 하나로 합치는 데 도움을 줄 수 있다. 하지만 데이터베이스의 Join 단계에서 Member 수만큼 만들어진 행 자체가 처음부터 사라지는 것은 아니다.

```text
데이터베이스 Join 결과
Team A - Member 1
Team A - Member 2
Team A - Member 3

Hibernate가 반환하는 Team Entity
Team A 한 개
```

애플리케이션에서 중복 Team이 하나로 보이더라도 데이터베이스는 여러 행을 읽고 전송했을 수 있다. `distinct`를 붙였다는 이유만으로 조회 비용과 페이징 문제가 모두 해결됐다고 판단해서는 안 된다.

## 테스트에서는 왜 N+1이 잘 안 보일까

N+1과 일대다 Fetch Join 문제는 테스트 데이터의 개수뿐 아니라 관계가 어떻게 분포했는지에 따라 가려질 수 있다.

### 여러 Member가 같은 Team만 참조하는 경우

```text
Member 1 → Team A
Member 2 → Team A
Member 3 → Team A
```

첫 번째 Member에서 Team A를 조회하면 같은 영속성 컨텍스트의 1차 캐시에 보관된다. 이후 Member가 같은 Team A를 요구하면 이미 조회한 Entity를 사용할 수 있다. Member가 세 명이어도 Team 추가 쿼리는 한 번만 보일 수 있어 N+1의 위험이 작아 보인다.

운영 데이터에서 Member마다 서로 다른 Team을 참조한다면 결과는 달라진다.

```text
Member 1 → Team A
Member 2 → Team B
Member 3 → Team C
```

이 경우 Team A, B, C를 각각 조회하므로 목록 크기에 따라 추가 쿼리가 늘어난다.

### Team마다 Member가 한 명뿐인 경우

```text
Team A → Member 1
Team B → Member 2
Team C → Member 3
```

일대다 관계여도 각 Team에 Member가 한 명뿐이라면 Join 결과에서 Team이 반복되지 않는다. 일대다 Fetch Join과 페이징을 함께 사용해도 개발 데이터에서는 결과 행 증가가 잘 드러나지 않을 수 있다.

실제 운영에서는 Team 하나에 Member가 여러 명일 수 있다.

```text
Team A → Member 1, Member 2, Member 3
Team B → Member 4, Member 5
```

이 데이터로 조회해야 같은 Team 행이 반복되고, `distinct`와 메모리 페이징 문제도 재현된다. 따라서 테스트 데이터는 행 개수만 늘릴 것이 아니라 연관 관계의 분포도 운영 환경과 비슷하게 구성해야 한다.

## 어떤 방법을 선택할까

| 상황 | 먼저 검토할 방법 |
| --- | --- |
| 이번 쿼리에서 필요한 다대일·일대일 관계 | Fetch Join 또는 EntityGraph |
| 일대다 관계와 페이징을 함께 사용 | ID 선조회 후 Fetch 또는 Batch Fetching |
| 여러 일대다 관계를 동시에 조회 | 분리 조회 또는 Batch Fetching |
| 읽기 전용 API에서 일부 Column만 필요 | DTO Projection |
| 연관 데이터가 필요한 시점이 다양함 | Batch Fetching |

N+1을 발견했을 때 곧바로 EAGER나 Fetch Join을 적용하기보다 다음 질문부터 확인하는 편이 좋다.

- 이번 조회에서 Entity 전체가 필요한가?
- Member → Team처럼 하나를 참조하는 관계인가, Team → Members처럼 여러 개를 가지는 관계인가?
- 페이징이 필요한가?
- Join하면 결과 행이 얼마나 늘어나는가?
- 필요한 Column만 DTO로 직접 조회하는 편이 낫지는 않은가?

이 질문에 따라 SQL 한 번이 가장 좋은 선택일 수도 있고, 크기가 제한된 쿼리 두세 번이 더 안전할 수도 있다.

## 정리

- N+1은 처음 쿼리에서 가져오지 않은 연관 데이터를 나중에 사용하면서 추가 쿼리가 반복되는 문제다.
- Java에서는 단순한 Getter 호출처럼 보여도 LAZY Proxy가 실제 데이터를 가져오는 SQL을 실행할 수 있다.
- EAGER는 연관 데이터를 즉시 준비하라는 설정이지 하나의 Join 쿼리를 보장하지 않는다.
- 다대일·일대일 Fetch Join은 결과 행 수가 기본 Entity 수와 같아 페이징과 비교적 잘 어울린다.
- 일대다 Fetch Join은 자식 수만큼 같은 데이터가 반복되어 페이징과 함께 사용할 때 주의해야 한다.
- Fetch Join이 맞지 않으면 Batch Fetching, EntityGraph, DTO Projection과 ID 선조회 방식을 상황에 맞게 선택할 수 있다.
- 테스트 데이터는 개수뿐 아니라 실제 연관 관계의 분포까지 반영해야 문제를 발견할 수 있다.

N+1은 “JPA가 느려서 생기는 문제”가 아니다. 처음 쿼리에서 가져오지 않은 연관 데이터를 나중에 사용하면서 추가 쿼리가 반복되는 문제다.

**N+1 해결의 목적은 SQL 개수를 무조건 한 개로 만드는 것이 아니다. 쿼리 수뿐 아니라 데이터베이스가 읽는 행 수, 조회하는 Column과 데이터 양, 페이징의 안정성을 함께 줄이는 것이 목적이다.**

## 참고 자료

### 공식 자료

- [Hibernate ORM - Association Fetching](https://docs.jboss.org/hibernate/orm/7.0/userguide/html_single/Hibernate_User_Guide.html)
- [Hibernate Query Language - Association Fetching and Pagination](https://docs.jboss.org/hibernate/orm/7.0/querylanguage/html_single/Hibernate_Query_Language.html)
- [Jakarta Persistence - Entity Graphs and Fetch Joins](https://jakarta.ee/specifications/persistence/3.2/jakarta-persistence-spec-3.2)

### 국내 기술 블로그

- [우아한형제들 - 잊을만 하면 돌아오는 정산 신병들](https://techblog.woowahan.com/2711/)
