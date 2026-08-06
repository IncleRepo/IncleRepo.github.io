+++
title = 'Offset과 Keyset Pagination은 언제 선택해야 하는가'
date = 2026-03-05T19:00:00+09:00
lastmod = 2026-08-06T17:40:00+09:00
draft = false
description = 'Offset Pagination이 뒤 페이지에서 느려지는 이유와 Keyset Pagination의 안정적인 Cursor 조건, Page와 Slice 선택 기준을 설명합니다.'
categories = ['데이터 접근 설계']
tags = ['페이징', 'Offset', 'Keyset', 'Spring Data']
+++

화면에 데이터를 나눠 보여주는 가장 익숙한 방법은 `LIMIT`과 `OFFSET`이다. 구현이 간단하고 원하는 페이지로 바로 이동할 수 있지만 Offset이 커질수록 앞의 행을 처리하고 버리는 비용이 커질 수 있다.

Pagination이 필요한 이유부터 보면 두 방식의 차이가 분명해진다. 데이터가 100만 건인데 한 화면에 20건만 필요하다면 애플리케이션으로 모두 가져온 뒤 자르는 것은 네트워크와 메모리를 낭비한다. 데이터베이스가 정렬된 결과 중 필요한 구간만 반환하도록 해야 한다.

Keyset Pagination은 마지막으로 본 정렬 값을 다음 조회 조건으로 사용한다. 깊은 페이지에서도 Index 범위 탐색을 이어갈 수 있지만 임의의 페이지 번호로 이동하기 어렵다. 두 방식은 속도만이 아니라 사용자 화면의 탐색 방식까지 함께 보고 선택해야 한다.

## Offset Pagination은 어떻게 동작하는가

```sql
SELECT id, title, created_at
  FROM post
 ORDER BY created_at DESC, id DESC
 LIMIT 20 OFFSET 100000;
```

이 쿼리는 100001번째 행으로 순간 이동하는 명령이 아니다. 실행 계획에 따라 Index를 사용하더라도 앞선 행을 따라가며 Offset만큼 제외한 뒤 20개를 반환해야 한다. 정렬에 적합한 Index가 없으면 정렬 비용까지 더해질 수 있다.

`OFFSET 100000`은 결과의 시작 번호를 알고 있을 뿐, 100001번째 행의 인덱스 Key를 알려 주지 않는다. 데이터베이스는 정렬 순서상 앞에 있는 행을 확인하면서 10만 개를 세고 버린 뒤에야 다음 20개를 반환할 수 있다.

```text
OFFSET 0      → 앞에서 20개 읽고 반환
OFFSET 1000   → 1,000개 건너뛴 뒤 20개 반환
OFFSET 100000 → 100,000개 건너뛴 뒤 20개 반환
```

Index가 있으면 정렬된 인덱스를 따라갈 수 있어 전체 Table 정렬은 피할 수 있다. 그러나 앞의 인덱스 항목을 건너뛰는 작업까지 없어지는 것은 아니다. 조회 Column이 인덱스에 모두 없으면 건너뛸 행마다 원본 행 접근이 추가될 수도 있다.

Offset 방식의 장점도 분명하다.

- 1, 2, 3과 같은 페이지 번호를 제공하기 쉽다.
- 특정 페이지로 바로 이동할 수 있다.
- Spring Data의 `Pageable`과 자연스럽게 연결된다.

관리 화면이나 전체 데이터가 크지 않은 목록이라면 이 단순함이 충분한 가치가 있다.

문제는 같은 탐색 방식을 데이터가 계속 쌓이는 깊은 목록에도 그대로 적용할 때다. 앞선 행을 매번 건너뛰는 비용을 피하려면 페이지 번호가 아니라 마지막으로 읽은 위치에서 조회를 이어가야 한다.

## Keyset Pagination은 마지막 위치에서 이어간다

첫 조회는 정렬의 처음부터 일정 개수를 가져온다.

```sql
SELECT id, title, created_at
  FROM post
 ORDER BY created_at DESC, id DESC
 LIMIT 21;
```

다음 조회에서는 직전 페이지의 마지막 값이 Cursor가 된다.

```sql
SELECT id, title, created_at
  FROM post
 WHERE created_at < :lastCreatedAt
    OR (created_at = :lastCreatedAt AND id < :lastId)
 ORDER BY created_at DESC, id DESC
 LIMIT 21;
```

정렬과 맞는 `(created_at DESC, id DESC)` Index가 있다면 데이터베이스는 마지막 위치 이후의 범위를 이어서 읽을 수 있다. Offset처럼 앞의 10만 행을 매번 제외할 필요가 없다.

첫 페이지의 마지막 행이 `(2026-03-05 10:00:00, 101)`이라면 Cursor는 “21번째 위치”라는 숫자가 아니라 이 두 값을 기억한다. 다음 Query는 인덱스에서 해당 값보다 뒤에 오는 범위를 찾아 바로 이어 읽는다. 그래서 페이지가 깊어져도 이전 페이지 수만큼 건너뛰는 비용이 누적되지 않는다.

다만 “마지막 위치”를 정확히 표현하지 못하면 빠른 조회를 얻고도 행을 빠뜨릴 수 있다. Keyset Pagination의 핵심은 Cursor가 정렬 순서를 완전하게 재현하는 데 있다.

## Cursor는 정렬 순서를 빠짐없이 표현해야 한다

`created_at`만 Cursor로 사용하면 같은 시각에 만들어진 여러 행 사이의 순서가 불완전하다. 다음 요청에서 일부 행을 건너뛰거나 중복으로 볼 수 있다.

```text
2026-03-05 10:00:00, id 103
2026-03-05 10:00:00, id 102
2026-03-05 10:00:00, id 101
```

정렬 값이 유일하지 않다면 고유한 Tie-breaker를 마지막에 둬야 한다. Cursor는 정렬에 사용한 모든 값을 담아야 하고, 비교 조건의 방향도 `ORDER BY`와 일치해야 한다.

예를 들어 첫 페이지의 마지막 행이 id 102인데 Cursor에 시각만 저장했다고 하자. 다음 조건을 `created_at < 10:00:00`으로 만들면 같은 시각의 id 101은 조회 대상에서 빠진다. 반대로 `created_at <= 10:00:00`으로 만들면 이미 본 id 103과 102가 다시 나올 수 있다. `created_at`이 같을 때 `id`까지 비교해야 경계가 정확해진다.

복합 Cursor는 외부에 그대로 노출하기보다 Base64URL 같은 형태로 직렬화할 수 있다. 이것은 값을 숨기는 암호화가 아니라 API 형식을 안정적으로 유지하는 방법이다. Client가 임의로 조작하면 안 되는 값이라면 서명도 검토해야 한다.

Cursor에 포함할 식별자는 정렬뿐 아니라 외부 노출 방식도 함께 고려한다. Auto Increment ID는 단순하고 효율적이지만 값의 증가 방향이 드러난다. 다음 식별자를 쉽게 짐작하게 만들고 싶지 않으면서 생성 순서도 유지해야 한다면 시간 정보와 Random Bit를 함께 가진 UUID v7을 후보로 볼 수 있다.

다만 UUID는 인가를 대신하지 않는다. 값을 추측하기 어렵게 만드는 것과 해당 사용자가 리소스를 조회할 권한이 있는지는 다른 문제다. 모든 Cursor에 UUID v7이 필요한 것도 아니다. 이미 안정적인 정렬 Column과 고유 ID가 있다면 두 값을 함께 Cursor로 사용해도 된다. **정렬 기준, Index와 외부에 노출할 식별자의 성질을 함께 보고 Cursor를 정한다.**

Cursor 조건을 정했다면 이제 이 방식을 애플리케이션의 응답 모델과 연결해야 한다. Spring Data의 `Page`, `Slice`, `Window`는 비슷해 보이지만 전체 개수와 다음 위치를 다루는 방식이 서로 다르다.

## Page, Slice와 Window의 차이

Spring Data에서 `Page<T>`는 전체 행 수와 전체 페이지 수를 제공하기 위해 Count Query가 필요할 수 있다.

```java
Page<Post> findByStatus(PostStatus status, Pageable pageable);
```

전체 개수가 필요 없다면 `Slice<T>`가 더 단순하다.

```java
Slice<Post> findByStatus(PostStatus status, Pageable pageable);
```

Slice는 요청한 크기보다 한 건 더 조회해 다음 데이터가 있는지 판단한다.

```text
pageSize = 20
실제 조회 = 21
21번째 행 존재 → hasNext = true
응답에는 앞의 20개만 포함
```

다만 `Slice`에 `Pageable`을 넘기는 것만으로 Keyset 방식이 되는 것은 아니다. Count Query를 생략한 Offset Pagination일 수 있다. Spring Data의 Keyset 기반 `Window<T>`나 직접 작성한 Cursor 조건을 사용해야 조회 위치 자체가 Keyset으로 바뀐다.

`Page`, `Slice`와 Keyset을 한 축으로만 비교하면 헷갈리기 쉽다. 서로 답하는 질문이 다르다.

- `Page`와 `Slice`: 전체 개수와 다음 페이지 여부를 응답에 어떻게 담을 것인가
- Offset과 Keyset: 데이터베이스에서 다음 구간을 어떤 조건으로 찾을 것인가

따라서 `Slice`이면서 Offset 방식일 수도 있고, 별도의 Cursor 응답을 사용하면서 Keyset 방식일 수도 있다. `Window<T>`는 Spring Data가 ScrollPosition과 다음 위치를 다루도록 제공하는 추상화다.

이 차이에서 별도로 확인할 비용이 Count Query다. 전체 페이지 수가 필요한 `Page`를 선택했다면 목록 조회가 빨라도 Count가 병목이 될 수 있다.

## Count Query는 언제 비싼가

단순한 작은 테이블의 `COUNT(*)`는 문제가 아닐 수 있다. 하지만 복잡한 Filter와 Join이 붙고 대상 행이 매우 많으면 목록 쿼리와 별도로 큰 범위를 다시 처리해야 한다.

전체 개수가 정말 필요한지 먼저 묻는 편이 좋다.

- “다음 페이지 있음”만 필요하면 Slice
- 정확한 전체 페이지가 필요하면 Page와 Count Query
- 대략적인 개수면 충분하면 별도 집계나 근삿값
- 자주 바뀌지 않는 집계라면 Cache 또는 집계 테이블

Count를 없애기 위해 더 복잡하고 부정확한 구조를 바로 도입할 필요는 없다. 실제 실행 시간과 사용자 요구를 함께 확인한다.

목록 Query가 20건만 찾으면 끝나는 반면 Count Query는 조건을 만족하는 전체 범위를 확인해야 할 수 있다. 예를 들어 검색 결과 50만 건 중 첫 20건만 보여주는 화면이라면 목록은 적절한 Index로 빨리 끝나도 정확한 전체 개수를 구하기 위해 훨씬 많은 행을 검사할 수 있다. `Page`가 느린 원인이 목록 SQL이 아니라 자동 생성된 Count SQL인 경우도 있으므로 두 실행 계획을 따로 확인해야 한다.

조회 비용만으로 방식을 고르면 한 가지를 놓치기 쉽다. 사용자가 페이지를 넘기는 동안 데이터가 추가되거나 변경될 때 같은 행을 다시 보거나 건너뛰는지도 함께 살펴봐야 한다.

## 데이터 변경 중 페이지 안정성

Offset Pagination은 사용자가 다음 페이지로 이동하기 전에 앞쪽에 새 행이 들어오면 기존 행이 밀려 중복되거나 누락될 수 있다. Keyset은 마지막 정렬 값 이후를 조회하므로 앞쪽 삽입의 영향을 덜 받는다.

```text
첫 페이지: [10, 9, 8]
그사이 맨 앞에 11 삽입
Offset 두 번째 페이지: OFFSET 3 → [8, 7, 6]
```

첫 페이지에서 이미 본 8이 다시 나타난다. Keyset 방식으로 “8보다 작은 값”을 조회했다면 새로 들어온 11과 관계없이 `[7, 6, 5]`를 이어서 받을 수 있다.

그렇다고 완전한 Snapshot을 제공하는 것은 아니다. 페이지 사이에 기존 행의 정렬 값이 바뀌거나 삭제되면 결과는 달라질 수 있다. 모든 페이지를 하나의 시점으로 고정해야 한다면 별도의 Snapshot 기준 시각이나 검색 결과 ID 집합 같은 더 강한 설계가 필요하다.

결국 선택 기준은 Query 성능 하나가 아니다. 화면이 페이지 번호를 요구하는지, 다음 데이터만 이어 읽는지, 조회 중 변경을 어느 정도 허용하는지를 함께 놓고 판단해야 한다.

## 무엇을 선택할까

Offset Pagination이 잘 맞는 경우는 다음과 같다.

- 페이지 번호와 임의 이동이 중요한 관리 화면
- 데이터가 크지 않거나 사용자가 깊은 페이지까지 가지 않는 목록
- 단순한 구현과 전체 개수가 중요한 경우

Keyset Pagination이 잘 맞는 경우는 다음과 같다.

- 무한 스크롤과 다음 페이지 중심의 탐색
- 시간순 Feed처럼 데이터가 계속 추가되는 목록
- 데이터가 크고 깊은 Offset 조회가 실제 병목인 경우

## 정리

- Offset은 구현이 단순하지만 깊어질수록 앞선 행을 처리하는 비용이 커질 수 있다.
- Keyset은 마지막 정렬 값 이후를 조회해 Index 범위 탐색을 이어간다.
- 안정적인 Cursor에는 고유한 Tie-breaker와 정렬에 사용한 모든 값이 필요하다.
- `Slice`는 Count를 생략하지만 자동으로 Keyset Pagination이 되는 것은 아니다.
- 페이지 번호가 필요한지, 다음 이동만 필요한지가 기술 선택을 결정한다.

## 참고 자료

### 공식 자료

- [Spring Data JPA - Query Methods and Paging](https://docs.spring.io/spring-data/jpa/reference/4.0/repositories/query-methods-details.html)
- [Spring Data JPA - Scrolling Large Query Results](https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html#jpa.query-methods.scroll)
- [MySQL - SELECT and LIMIT](https://dev.mysql.com/doc/refman/8.4/en/select.html)
