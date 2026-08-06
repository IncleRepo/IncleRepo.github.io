+++
title = 'N+1 문제를 Fetch Join만으로 해결할 수 없는 이유'
date = 2026-02-20T19:00:00+09:00
lastmod = 2026-08-06T17:54:00+09:00
draft = false
description = 'JPA 연관 관계에서 N+1이 발생하는 원리를 확인하고 Fetch Join, Batch Fetching, EntityGraph와 DTO Projection의 선택 기준을 정리합니다.'
categories = ['JPA']
tags = ['JPA', 'Hibernate', 'N+1', 'Fetch Join']
+++

JPA의 N+1 문제는 처음 실행한 Query의 결과를 사용하는 과정에서 연관 Entity를 가져오기 위한 Query가 N번 더 발생하는 현상이다. 연관 관계가 있다는 이유만으로 생기는 것이 아니라, 어떤 Fetch Plan으로 조회했고 이후 어떤 연관 속성에 접근했는지가 함께 맞을 때 나타난다.

처음 JPA를 배울 때 N+1이 낯선 이유는 Java 코드에는 추가 Query가 보이지 않기 때문이다. `member.getTeam().getName()`은 평범한 Getter 호출처럼 보이지만, 지연 로딩된 연관 관계라면 이 순간 Hibernate가 데이터베이스를 조회할 수 있다. 객체 접근과 SQL 실행이 같은 줄에서 일어날 수 있다는 점부터 이해해야 한다.

Fetch Join은 강력한 해결책이지만 모든 목록에 그대로 적용할 수는 없다. 특히 Collection Fetch Join과 Pagination을 함께 사용하면 데이터베이스가 아닌 메모리에서 페이지를 자르거나 Join 결과가 폭발할 수 있다.

## N+1은 언제 발생하는가

Member가 Team을 지연 로딩한다고 해보자.

```java
@Entity
public class Member {

    @ManyToOne(fetch = FetchType.LAZY)
    private Team team;
}
```

먼저 Member 20명을 조회한다.

```java
List<Member> members = memberRepository.findTop20ByOrderByIdAsc();

for (Member member : members) {
    System.out.println(member.getTeam().getName());
}
```

첫 Query는 Member만 가져온다.

```sql
SELECT * FROM member ORDER BY id LIMIT 20;
```

반복문에서 `team.name`에 접근하면 초기화되지 않은 Team을 조회한다. 서로 다른 Team이 20개라면 추가 Query도 최대 20번 발생할 수 있다.

`LAZY` 관계에는 실제 Team 대신 Team을 나중에 조회할 수 있는 Proxy가 들어갈 수 있다. Proxy는 식별자를 알고 있지만 이름과 같은 실제 데이터는 아직 갖고 있지 않다. 이름을 요구받는 순간 다음 순서로 초기화된다.

```text
member.getTeam() 호출
→ Team Proxy 반환
→ getName() 호출
→ Proxy가 아직 초기화되지 않았는지 확인
→ team_id로 SELECT 실행
→ 조회 결과를 Proxy에 채운 뒤 name 반환
```

```text
Member 목록 조회 1번
+ Team 조회 N번
= N+1
```

같은 영속성 컨텍스트에서 여러 Member가 같은 Team을 참조하면 1차 Cache 때문에 실제 추가 Query 수는 줄어들 수 있다. 그래서 N은 “결과 행 수와 항상 정확히 같다”보다 “결과 규모에 따라 추가 조회가 늘어날 수 있다”는 의미로 이해하는 편이 정확하다.

개발 데이터에서 Member가 모두 같은 Team을 참조하면 첫 번째 Team 조회 뒤 나머지는 1차 Cache에서 찾기 때문에 Query가 두 번만 보일 수 있다. 반면 운영 데이터에서 Member마다 다른 Team을 참조하면 추가 Query가 크게 늘어난다. 테스트 데이터의 관계 분포까지 실제와 비슷하게 만들어야 하는 이유다.

이 현상을 발견하면 가장 손쉬운 해결책처럼 보이는 것이 Mapping을 EAGER로 바꾸는 것이다. 하지만 연관 데이터를 일찍 가져오라는 설정과 한 번의 Join으로 가져오라는 Query 계획은 같은 말이 아니다.

## EAGER로 바꾸면 해결될까

`FetchType.EAGER`는 연관 Entity를 즉시 사용할 수 있게 가져오라는 요구다. SQL Join 한 번으로 가져오라는 보장은 아니다. JPQL이 연관 관계를 Fetch Join하지 않으면 Hibernate가 추가 Select를 실행해 EAGER 관계를 채울 수 있고 이 과정에서도 N+1이 발생할 수 있다.

즉 `EAGER`와 Fetch Join은 서로 다른 층의 설정이다.

- `EAGER`: 이 연관 관계를 현재 조회 결과에서 사용할 수 있게 준비하라는 Mapping 규칙
- Fetch Join: 이번 Query의 SQL에서 연관 대상을 Join해 함께 가져오라는 조회 규칙

EAGER는 “언제 필요하다고 볼 것인가”를 정하지만, 반드시 “어떤 SQL 한 문장으로 가져올 것인가”까지 정하지는 않는다.

모든 연관 관계를 EAGER로 바꾸면 필요하지 않은 화면에서도 연관 데이터를 가져오게 된다. 기본 Mapping은 LAZY로 두고 Query마다 필요한 Fetch Plan을 명시하는 편이 예측하기 쉽다.

Query마다 필요한 관계를 명시하는 가장 직접적인 방법이 Fetch Join이다. 먼저 부모 행 수를 늘리지 않는 To-one 관계부터 보면 Fetch Join을 어디까지 안전하게 사용할 수 있는지 이해하기 쉽다.

## To-one Fetch Join

현재 Query에서 Team이 항상 필요하다면 Fetch Join으로 함께 조회할 수 있다.

```java
@Query("""
    select m
      from Member m
      join fetch m.team
     order by m.id
    """)
List<Member> findMembersWithTeam(Pageable pageable);
```

```sql
SELECT m.*, t.*
  FROM member m
  JOIN team t ON t.id = m.team_id
 ORDER BY m.id
 LIMIT 20;
```

Member 하나가 Team 하나만 참조하는 To-one Fetch Join은 부모 행 수를 Collection 크기만큼 늘리지 않는다. 여러 To-one 관계를 함께 Fetch하는 것도 일반적으로 안전하다. 물론 Join 대상이 많고 행이 넓어지면 실제 실행 계획과 전송량은 확인해야 한다.

Member 20명과 각 Member의 Team을 Join해도 결과는 기본적으로 Member 수와 같은 20행이다. 각 Member 행 옆에 Team Column이 붙을 뿐이기 때문이다. 이 특성 때문에 To-one Fetch Join은 목록 Pagination과 비교적 잘 어울린다.

같은 Fetch Join이라도 Collection에서는 결과의 모양이 달라진다. 부모 하나가 여러 자식과 결합하면서 행 수가 늘어나기 때문에 Pagination과 함께 사용할 때 별도의 문제가 생긴다.

## Collection Fetch Join과 Pagination

Team과 Member가 일대다이고 Team 목록을 조회하면서 Member Collection까지 Fetch하면 한 Team이 Member 수만큼 반복된다.

```text
Team A - Member 1
Team A - Member 2
Team A - Member 3
Team B - Member 4
```

여기에 `LIMIT 20`을 적용하면 Team 20개가 아니라 Join 결과 20행이 잘릴 수 있다. Hibernate는 Collection Fetch Join과 Pagination을 결합했을 때 전체 결과를 가져온 뒤 메모리에서 페이지를 적용할 수 있다. 데이터가 많으면 치명적이다.

예를 들어 Team A에 Member가 30명 있고 Team B에 1명이 있다고 하자. Join 결과의 처음 20행은 전부 Team A일 수 있다. 데이터베이스에서 `LIMIT 20`을 적용하면 부모 Team을 20개 가져온 것이 아니라 Team A의 Collection 일부만 가져오게 된다. JPA는 Collection을 불완전한 상태로 만들 수 없으므로 데이터베이스 Pagination을 그대로 적용하기 어렵다.

Hibernate 설정으로 이런 Query를 예외 처리하게 만들어 조기에 발견할 수 있다.

```yaml
spring:
  jpa:
    properties:
      hibernate.query.fail_on_pagination_over_collection_fetch: true
```

해결 방법 중 하나는 ID를 먼저 페이지 조회한 뒤 두 번째 Query에서 필요한 연관 관계를 가져오는 것이다.

```text
1. 조건과 정렬로 Team ID 20개 조회
2. WHERE team.id IN (...)으로 Team과 Member Fetch
3. 첫 Query의 ID 순서대로 결과 정렬
```

두 번 왕복하지만 Join 결과 전체를 메모리에 올리는 것보다 훨씬 안전할 수 있다.

여기서 놓치기 쉬운 점이 하나 있다. Repository Method에 `Pageable`이 보이지 않아도 Batch의 Paging ItemReader가 일정 크기로 결과를 나누면 Pagination Query가 된다. 이 Reader에 Collection Fetch Join을 넣으면 화면 목록에서 발생하는 것과 같은 메모리 Pagination 문제가 생길 수 있다.

```text
Batch가 부모 100개를 읽으려고 함
→ 부모와 자식 Collection을 Fetch Join
→ DB 결과는 자식 수만큼 부모 행이 반복됨
→ 부모 기준으로 LIMIT을 적용할 수 없음
→ 전체 Join 결과를 읽은 뒤 메모리에서 100개 선택
```

해결할 때는 부모 ID를 먼저 일정 크기로 조회하고, 두 번째 Query에서 `WHERE parent.id IN (...)`으로 필요한 연관 데이터를 가져올 수 있다. Batch라고 해서 조회 원리가 달라지는 것이 아니라 Pagination이 코드 바깥의 Reader 설정에 숨어 있을 뿐이다.

테스트 데이터도 실제 Join 모양을 만들어야 한다. 부모마다 자식이 하나뿐이면 Join 뒤에도 부모 행이 한 번만 나타나므로 `distinct` 누락과 메모리 Pagination 문제가 가려진다. 일대다 조회 테스트에는 부모 하나에 여러 자식을 넣어 행이 실제로 늘어나는 조건을 재현해야 한다.

Collection을 한 Query에 모두 펼치는 것이 위험하다면 Query 수를 조금 허용하고 각 조회의 크기를 제한하는 선택지가 필요하다. Batch Fetching은 이 지점에서 N번의 개별 조회와 하나의 거대한 Join 사이를 절충한다.

## Batch Fetching

Batch Fetching은 초기화되지 않은 여러 Proxy나 Collection을 `IN` 조건으로 묶어 조회한다.

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

추가 Query가 20번에서 한두 번으로 줄어들 수 있다. Fetch Join처럼 한 번의 SQL로 끝나는 것은 아니지만, Collection Pagination이나 여러 연관 관계에서 Join 결과가 폭발하는 상황에는 현실적인 절충안이다.

Member 100명이 서로 다른 Team을 참조하고 Batch 크기가 20이라면 Team을 하나씩 100번 조회하는 대신 최대 20개 ID씩 묶어 약 5번 조회할 수 있다.

```text
Member 조회 1번
+ Team 20개씩 조회 5번
= 총 6번
```

Query 한 번보다 많지만 101번보다는 훨씬 작고, 거대한 Join 결과를 한 번에 만드는 문제도 피할 수 있다. Batch 크기를 지나치게 키우면 `IN` 절과 한 번에 읽는 데이터가 커지므로 실제 데이터 규모에 맞춰 조정한다.

Batch Fetching이 연관 초기화의 실행 방식을 조정한다면, EntityGraph는 현재 Repository Method에 필요한 연관 범위를 선언한다. 해결하려는 문제는 비슷하지만 Fetch Plan을 표현하는 위치가 다르다.

## EntityGraph

EntityGraph는 특정 Repository Method에서 가져올 연관 속성을 선언하는 Fetch Plan이다.

```java
@EntityGraph(attributePaths = "team")
List<Member> findTop20ByOrderByIdAsc();
```

Fetch Join Query 문자열을 직접 작성하지 않고도 필요한 관계를 명시할 수 있다. To-one 관계를 가져오는 데도 사용할 수 있으며 “중복이 생기므로 To-one에는 피한다”는 규칙은 맞지 않는다. Collection을 포함하면 Fetch Join과 마찬가지로 결과 행 증가와 Pagination 문제를 확인해야 한다.

하지만 조회 화면이 Entity와 연관 관계 전체를 요구하지 않을 수도 있다. 필요한 Column이 명확한 읽기 전용 응답이라면 Entity의 Fetch Plan을 조정하는 대신 처음부터 조회 모델을 만들 수 있다.

## DTO Projection

화면에 Member 이름과 Team 이름만 필요하다면 Entity Graph 전체를 만들지 않아도 된다.

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

필요한 Column만 조회하고 Entity의 지연 로딩을 피할 수 있어 조회 전용 API에 잘 맞는다. 반면 조회 결과를 변경해 영속화할 수 없고, 화면별 Projection이 늘어날 수 있다. Entity가 필요한 업무 로직과 조회 모델을 구분해서 사용한다.

DTO Projection에서는 `MemberSummary`에 필요한 세 Column을 Query가 처음부터 가져온다. 반환된 객체에는 지연 로딩 Proxy가 없으므로 이후 Getter를 호출해도 추가 SQL이 생기지 않는다. 대신 Member의 도메인 메서드를 실행하거나 변경 감지로 저장하려는 용도에는 맞지 않는다.

Fetch Join에 `distinct`를 붙이는 것만으로 모든 문제가 해결되는 것도 아니다. SQL Join 단계에서 생성되는 행 수는 이미 Collection 크기만큼 늘어난다. Hibernate가 같은 부모 Entity를 하나로 합칠 수는 있어도 데이터베이스가 읽고 전송한 행 자체가 사라지는 것은 아니다. 중복의 원인이 무엇인지 확인하지 않고 `distinct`로 가리는 방식은 피해야 한다.

결국 N+1을 해결하는 도구는 하나가 아니다. 현재 Query가 Entity를 필요로 하는지, 연관 관계가 To-one인지 Collection인지, Pagination이 있는지를 기준으로 앞의 선택지를 정리할 수 있다.

## 어떤 방법을 선택할까

| 상황 | 우선 검토할 방법 |
| --- | --- |
| 현재 Query에서 필요한 To-one 관계 | Fetch Join 또는 EntityGraph |
| Collection과 Pagination | ID 선조회 후 Fetch 또는 Batch Fetching |
| 여러 Collection을 동시에 조회 | 분리 조회 또는 Batch Fetching |
| 읽기 전용 API의 일부 Column | DTO Projection |
| 필요한 시점이 다양하고 Join 폭발 우려 | Batch Fetching |

N+1 해결의 목적은 SQL 개수를 무조건 한 개로 만드는 것이 아니다. Query 수, 반환 행 수, 데이터 폭과 Pagination 안정성을 함께 줄이는 것이 목적이다.

## 정리

- N+1은 첫 조회 뒤 연관 관계 초기화 Query가 결과 규모에 따라 늘어나는 문제다.
- EAGER는 Join을 보장하지 않으므로 N+1의 일반적인 해결책이 아니다.
- To-one Fetch Join은 Pagination과 함께 사용해도 Collection Fetch Join과 성격이 다르다.
- Collection Fetch Join과 Pagination은 메모리 Pagination을 만들 수 있다.
- Batch Fetching, EntityGraph와 DTO Projection은 각각 다른 조회 조건에 맞는 선택지다.

## 참고 자료

### 공식 자료

- [Hibernate ORM - Association Fetching](https://docs.jboss.org/hibernate/orm/7.0/userguide/html_single/Hibernate_User_Guide.html)
- [Hibernate Query Language - Association Fetching and Pagination](https://docs.jboss.org/hibernate/orm/7.0/querylanguage/html_single/Hibernate_Query_Language.html)
- [Jakarta Persistence - Entity Graphs and Fetch Joins](https://jakarta.ee/specifications/persistence/3.2/jakarta-persistence-spec-3.2)

### 국내 기술 블로그

- [우아한형제들 - 잊을만 하면 돌아오는 정산 신병들](https://techblog.woowahan.com/2711/)
