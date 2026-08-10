+++
title = 'Offset과 Keyset Pagination은 언제 선택해야 하는가'
slug = '3'
aliases = ['/posts/003/']
date = 2026-03-05T19:00:00+09:00
lastmod = 2026-08-10T10:30:01+09:00
draft = false
description = 'Offset이 깊은 페이지에서 느려지는 이유부터 Keyset Cursor 설계, Page·Slice·Window와 Count 비용, 데이터 변경 중 안정성까지 차례대로 알아봅니다.'
categories = ['데이터 접근 설계']
tags = ['페이징', 'Offset', 'Keyset', 'Spring Data']
+++

게시글이 100만 건 있는데 한 화면에서는 20건만 필요하다고 해보자.

```text
전체 게시글
→ 1,000,000건

현재 화면에 필요한 게시글
→ 20건
```

100만 건을 애플리케이션으로 모두 가져온 뒤 20개만 잘라 쓰면 네트워크와 메모리를 낭비한다. 데이터베이스에서 필요한 구간만 가져와야 한다. 이처럼 많은 데이터에서 현재 화면에 필요한 구간만 나누어 조회하는 것이 Pagination(페이징)이다.

가장 익숙한 페이징은 `LIMIT`과 `OFFSET`을 사용한다.

```sql
LIMIT 20 OFFSET 0;
LIMIT 20 OFFSET 20;
LIMIT 20 OFFSET 40;
```

구현이 단순하고 페이지 번호를 만들기 쉽다. 다만 데이터가 많아지고 뒤 페이지로 갈수록 처음에는 보이지 않던 비용이 커질 수 있다.

먼저 Offset이 뒤 페이지에서 느려지는 이유를 살펴본다. 이어서 Keyset이 이 문제를 어떻게 피하는지 알아보고, 화면 요구사항과 데이터 특성에 따라 두 방식 중 무엇을 선택하면 좋을지 정리한다.

## Offset은 앞에서부터 읽고 필요한 만큼 건너뛴다

다음 쿼리는 정렬된 게시글 중 100001번째 위치부터 20건을 가져오려는 요청이다.

```sql
SELECT id, title, created_at
  FROM post
 ORDER BY created_at DESC, id DESC
 LIMIT 20 OFFSET 100000;
```

`OFFSET 100000`이라고 쓰면 데이터베이스가 100001번째 행으로 바로 이동할 것처럼 느껴진다. 하지만 Offset에는 그 행을 찾을 수 있는 인덱스 값이 들어 있지 않다. 정렬된 결과에서 앞의 몇 건을 제외할지만 알려준다.

실행 계획에 따라 세부 동작은 달라지지만 핵심 비용은 다음처럼 이해할 수 있다.

```text
OFFSET 0
→ 처음부터 20건 읽고 반환

OFFSET 1,000
→ 앞의 1,000건을 지나친 뒤 20건 반환

OFFSET 100,000
→ 앞의 100,000건을 지나친 뒤 20건 반환
```

뒤 페이지로 갈수록 반환하지 않을 앞부분까지 처리하고 버리는 비용이 누적될 수 있다. 이것이 깊은 Offset이 느려지는 핵심 이유다.

다음 실행 계획은 20건을 반환하기 위해 그보다 훨씬 많은 행을 읽을 것으로 예상하는지 보여준다.

![Offset Pagination의 EXPLAIN 결과](/images/posts/pagination-optimization/legacy-01.png "Offset Pagination 실행계획")

### 인덱스가 있어도 건너뛰는 비용은 남는다

`ORDER BY created_at DESC, id DESC`와 맞는 인덱스가 있으면 MySQL은 테이블 전체를 다시 정렬하지 않고 인덱스 순서를 따라갈 수 있다. 그렇다고 앞의 10만 개를 건너뛰는 비용까지 사라지는 것은 아니다.

```text
정렬에 맞는 인덱스가 없음
→ 정렬 비용 + 앞부분을 건너뛰는 비용

정렬에 맞는 인덱스가 있음
→ 정렬 비용은 줄일 수 있음
→ 앞부분을 건너뛰는 비용은 남음
```

조회할 컬럼이 인덱스에 모두 들어 있지 않다면 실제 테이블 행까지 찾아가는 비용도 추가될 수 있다. 따라서 “인덱스가 있다”는 사실만으로 깊은 Offset 문제가 해결됐다고 볼 수 없다. `EXPLAIN`에서 예상한 읽기 범위와 실제 실행 시간을 함께 확인해야 한다.

### Offset이 잘 맞는 화면도 많다

Offset이 느릴 수 있다는 것은 Offset을 사용하면 안 된다는 뜻이 아니다. 다음 요구사항에는 여전히 단순하고 좋은 선택이다.

- 1, 2, 3과 같은 페이지 번호를 제공하기 쉽다.
- 사용자가 특정 페이지로 바로 이동할 수 있다.
- Spring Data의 `Pageable`과 자연스럽게 연결된다.
- 데이터가 크지 않은 관리 화면에서는 구현 단순성이 큰 장점이다.

문제는 데이터가 많은 목록에서 사용자가 뒤 페이지까지 자주 이동할 때 생긴다. 페이지 번호보다 다음 데이터를 계속 보는 것이 중요하다면, 앞의 결과를 매번 건너뛰는 대신 마지막으로 본 위치에서 조회를 이어갈 수 있다.

## Keyset은 마지막으로 본 위치에서 이어 읽는다

두 방식의 차이를 먼저 한 문장으로 비교해보자.

```text
Offset
→ "앞에서 몇 건을 건너뛰어라"

Keyset
→ "마지막으로 본 값 뒤에서 이어서 가져와라"
```

게시글을 ID 내림차순으로 정렬했고 첫 화면에서 다음 다섯 건을 봤다고 해보자.

```text
id 105
id 104
id 103
id 102
id 101
```

마지막으로 본 값은 101이다. 다음 요청은 앞에서 다섯 건을 다시 세지 않고 `id < 101`인 범위에서 이어 읽을 수 있다.

```sql
SELECT id, title
  FROM post
 WHERE id < 101
 ORDER BY id DESC
 LIMIT 20;
```

이처럼 마지막으로 본 정렬 값을 조건에 넣어 다음 구간을 찾는 방식이 **Keyset Pagination**이다.

실제 게시글은 생성 시각과 ID를 함께 정렬하는 경우가 많다.

```sql
SELECT id, title, created_at
  FROM post
 ORDER BY created_at DESC, id DESC
 LIMIT 21;
```

첫 조회의 마지막 게시글이 다음 값이라면 이 값들이 다음 조회의 출발점이 된다.

```text
created_at = 2026-03-05 10:00:00
id         = 101
```

다음 요청에는 이 두 값을 다시 보내야 한다. 그러면 데이터베이스는 마지막 게시글보다 뒤에 있는 범위부터 조회를 이어갈 수 있다. 이때 다음 조회의 출발점이 되는 마지막 위치를 **Cursor**라고 한다.

```sql
SELECT id, title, created_at
  FROM post
 WHERE created_at < :lastCreatedAt
    OR (
        created_at = :lastCreatedAt
        AND id < :lastId
    )
 ORDER BY created_at DESC, id DESC
 LIMIT 21;
```

정렬과 맞는 인덱스가 있다면 마지막 위치 이후의 범위부터 이어 읽을 수 있다.

```text
ORDER BY
→ created_at DESC, id DESC

Cursor
→ created_at, id

인덱스
→ (created_at DESC, id DESC)
```

`ORDER BY`의 컬럼 순서, Cursor의 값 순서와 인덱스의 컬럼 순서가 서로 맞아야 한다. 이 조건이 맞으면 깊은 페이지에서도 이전 페이지 수만큼 앞부분을 반복해서 건너뛰는 비용을 줄일 수 있다. 다만 인덱스를 만들었다고 항상 같은 성능이 보장되는 것은 아니므로 실행 계획과 실제 시간을 확인해야 한다.

다음 실행 계획에서는 Cursor 조건을 이용해 필요한 범위부터 읽기 시작하는지와 예상 행 수를 확인할 수 있다.

![Keyset Pagination의 EXPLAIN 결과](/images/posts/pagination-optimization/legacy-02.png "Keyset Pagination 실행계획")

두 방식의 실행 시간을 비교할 때는 첫 페이지의 결과만 봐서는 안 된다. 다음 화면처럼 페이지가 깊어질수록 Offset과 Keyset의 실행 시간이 어떻게 달라지는지 비교해야 한다.

![Offset과 Keyset Pagination의 실행 시간 비교](/images/posts/pagination-optimization/legacy-03.png "Offset과 Keyset 실행 시간 비교")

Keyset의 장점을 얻으려면 마지막 위치를 정확하게 표현해야 한다. Cursor가 정렬 순서의 일부만 기억하면 빠르게 조회하더라도 게시글을 빠뜨리거나 중복해서 보여줄 수 있다.

## Cursor에는 정렬에 사용한 값을 모두 담아야 한다

먼저 `created_at`만으로 게시글을 정렬했다고 해보자.

```sql
ORDER BY created_at DESC
```

같은 시각에 세 게시글이 생성될 수 있다.

```text
10:00:00 / id 103
10:00:00 / id 102
10:00:00 / id 101
```

첫 화면이 id 102에서 끝났는데 Cursor에 `created_at = 10:00:00`만 저장했다고 하자. 다음 조건을 사용하면 문제가 생긴다.

```sql
WHERE created_at < '10:00:00'
```

같은 10시 정각에 생성된 id 101은 조건에 포함되지 않아 건너뛰게 된다. 반대로 `created_at <= '10:00:00'`으로 조회하면 이미 본 id 103과 102가 다시 나타날 수 있다.

`created_at`만으로는 같은 시각에 생성된 세 행의 순서를 구분할 수 없기 때문이다. 이럴 때는 값이 겹치지 않는 `id`를 정렬 조건에 추가해 마지막 순서까지 확정한다. 이렇게 같은 정렬 값을 가진 행의 순서를 결정하는 추가 기준을 **Tie-breaker**라고 한다.

```sql
ORDER BY created_at DESC, id DESC
```

Cursor도 정렬에 사용한 두 값을 함께 가져야 한다.

```text
(created_at, id)
```

결국 Cursor만으로도 원래 정렬 순서를 그대로 재현할 수 있어야 한다.

```text
ORDER BY에 사용한 값
↓
Cursor에도 모두 포함
↓
비교 방향도 ORDER BY 방향과 일치
```

내림차순 정렬에서는 마지막 값보다 작은 범위를 이어 읽고, 오름차순 정렬에서는 반대 방향을 사용한다. 정렬 컬럼이 여러 개라면 앞 컬럼이 같을 때 다음 컬럼을 비교하는 조건까지 정확히 표현해야 한다.

정렬 컬럼에 `NULL`이 들어갈 수 있다면 비교 조건이 더 복잡해진다. SQL에서 `NULL`은 일반적인 크기 비교가 되지 않고 데이터베이스마다 `NULL`을 정렬하는 방식도 다를 수 있기 때문이다. 가능하면 Keyset에 사용하는 정렬 컬럼은 `NOT NULL`로 두고, `NULL`을 허용해야 한다면 정렬 순서와 다음 조회 조건을 명시적으로 설계해야 한다.

조회 위치를 정했다면 이제 애플리케이션이 결과를 어떤 형태로 반환할지 선택해야 한다. 여기서 Spring Data의 `Page`, `Slice`, `Window`를 Offset이나 Keyset과 같은 선택지로 생각하면 개념이 뒤섞이기 쉽다.

## 조회 위치를 찾는 방식과 결과를 반환하는 형식은 다르다

Offset과 Keyset은 데이터베이스에서 조회를 시작할 위치를 찾는 방법이다. 반면 Page, Slice와 Window는 조회한 결과와 다음 위치를 애플리케이션에서 표현하는 방법이다.

```text
데이터베이스에서 다음 구간을 찾는 방법
→ Offset / Keyset

애플리케이션에서 결과와 다음 위치를 표현하는 방법
→ Page / Slice / Cursor 응답 / Window
```

따라서 `Page vs Slice vs Keyset`처럼 한 줄에 놓고 비교하면 헷갈리기 쉽다. `Slice`를 반환하면서 내부에서는 Offset을 사용할 수 있고, 별도의 Cursor 응답을 반환하면서 Keyset으로 조회할 수도 있다.

| 개념 | 주로 다루는 것 | 핵심 정보 |
| --- | --- | --- |
| Offset | DB에서 다음 구간을 찾는 방법 | 앞에서 건너뛸 건수 |
| Keyset | DB에서 다음 구간을 찾는 방법 | 마지막 정렬 위치 |
| Page | 페이징 결과 표현 | 데이터와 전체 개수·페이지 수 |
| Slice | 페이징 결과 표현 | 데이터와 다음 페이지 여부 |
| Window | 스크롤 결과 표현 | 데이터와 다음 ScrollPosition |

### Page는 전체 개수와 페이지 수를 알려준다

`Page<T>`는 현재 데이터뿐 아니라 전체 데이터 수와 전체 페이지 수 같은 정보를 제공한다.

```text
현재 페이지의 데이터
전체 데이터 수
전체 페이지 수
현재 페이지 번호
다음·이전 페이지 여부
```

```java
Page<Post> findByStatus(PostStatus status, Pageable pageable);
```

전체 페이지 수를 계산하려면 전체 데이터 수를 알아야 한다. 그래서 목록을 조회하는 SQL과 별도로 Count 쿼리가 실행될 수 있다.

### Slice는 다음 데이터가 있는지만 알려준다

전체 개수와 페이지 수가 필요하지 않고 다음 데이터가 있는지만 알고 싶다면 `Slice<T>`를 사용할 수 있다.

```java
Slice<Post> findByStatus(PostStatus status, Pageable pageable);
```

일반적으로 요청한 크기보다 한 건 더 조회해 다음 데이터가 있는지 판단한다.

```text
pageSize = 20
실제로는 21건 조회

21번째 데이터 있음
→ hasNext = true

응답에는 앞의 20건만 반환
```

**Slice를 사용한다고 자동으로 Keyset Pagination이 되는 것은 아니다.** `Slice`에 `Pageable`을 넘기면 Count 쿼리를 생략한 Offset 방식일 수 있다. Slice는 응답에 무엇을 담는지에 관한 선택이고, 데이터베이스에서 위치를 찾는 방식은 별도로 정해야 한다.

### Window는 조회한 묶음과 다음 위치를 함께 다룬다

Spring Data는 큰 조회 결과를 일정한 묶음으로 이어 읽을 수 있는 Scroll API도 제공한다. `Window<T>`에는 이번에 조회한 데이터 묶음이 들어가고, `ScrollPosition`은 다음 묶음을 어디서부터 읽을지 나타낸다. 이 위치는 Offset 방식과 Keyset 방식으로 모두 표현할 수 있다.

다음 예제는 Keyset 방식으로 게시글 20건을 조회한 뒤, 첫 번째 Window의 마지막 위치에서 다음 조회를 이어가는 흐름이다.

```java
Window<Post> posts = postRepository
    .findFirst20ByStatusOrderByCreatedAtDescIdDesc(
        PostStatus.PUBLISHED,
        ScrollPosition.keyset()
    );

if (!posts.isEmpty() && !posts.isLast()) {
    ScrollPosition nextPosition = posts.positionAt(posts.size() - 1);

    posts = postRepository.findFirst20ByStatusOrderByCreatedAtDescIdDesc(
        PostStatus.PUBLISHED,
        nextPosition
    );
}
```

첫 조회의 `ScrollPosition.keyset()`은 아직 마지막 위치가 없으므로 정렬 결과의 처음부터 시작하라는 뜻이다. `positionAt(posts.size() - 1)`은 이번 Window의 마지막 게시글에서 다음 조회에 필요한 정렬 값을 꺼낸다. 두 번째 조회는 그 위치를 제외하고 바로 다음 게시글부터 시작한다.

즉 `Window`가 다음 데이터를 자동으로 조회하는 것은 아니다. 이번에 조회한 데이터와 각 데이터의 위치를 함께 제공하고, 애플리케이션은 마지막 위치를 다음 요청에 전달해 조회를 이어간다.

세부 지원 범위와 쿼리 작성 방식은 사용하는 Spring Data 모듈과 버전에 따라 다르다. 여기서 기억할 점은 Window가 전체 페이지 수를 보여주는 화면보다 현재 위치에서 다음 데이터를 계속 읽는 흐름에 어울린다는 것이다.

`Page`, `Slice`, `Window`의 차이를 이해하면 목록 조회 밖에서 발생하는 비용도 보인다. 지금까지는 목록 쿼리 자체만 봤지만, Page를 사용하면 Count 쿼리가 추가로 실행될 수 있다.

## Page를 사용하면 Count 쿼리까지 확인해야 한다

Page 조회 요청은 다음 두 SQL로 나뉠 수 있다.

```text
1. 현재 페이지 목록 20건 SELECT
2. 조건을 만족하는 전체 개수 COUNT
```

목록 쿼리는 적절한 인덱스를 이용해 20건을 찾으면 끝날 수 있다. 반면 Count 쿼리는 전체 개수를 계산하기 위해 조건을 만족하는 범위를 끝까지 확인해야 할 수 있다.

예를 들어 검색 결과가 50만 건인 화면에서 첫 20건만 보여준다고 해보자. 목록은 빠르게 끝나도 정확한 전체 개수를 구하는 Count는 훨씬 많은 행을 확인할 수 있다. `Page`가 느린 원인이 목록 SQL이 아니라 자동 생성된 Count SQL일 수도 있다.

다음 실행 계획은 Count 쿼리가 얼마나 많은 행을 읽을 것으로 예상하는지와 어떤 인덱스를 선택했는지 보여준다. 목록 쿼리와 Count 쿼리는 하는 일이 다르므로 실행 계획도 따로 확인해야 한다.

![COUNT Query 실행계획 확인](/images/posts/pagination-optimization/legacy-04.png "COUNT Query 실행계획")

Count가 항상 비싼 것은 아니다. 작은 테이블이나 단순한 조건에서는 충분히 빠를 수 있다. 반면 데이터가 크고 Join과 조회 조건이 복잡하며 결과 수가 많다면 비용이 커질 수 있다.

가장 먼저 물어볼 질문은 “전체 개수가 정말 필요한가?”이다.

```text
정확한 전체 개수와 페이지 수가 필요
→ Page + Count

다음 데이터 존재 여부만 필요
→ Slice

대략적인 개수면 충분
→ 통계 기반 근삿값 검토

자주 조회하는 집계
→ Cache / 집계 테이블 / Counter Table 검토
```

### Count 결과를 미리 저장할 수도 있다

정확한 집계를 빠르게 반복 조회해야 한다면 별도의 테이블에 개수를 미리 저장해둘 수도 있다. 흔히 **Counter Table**이라고 하는 방식이다. 첫 번째 화면은 원본 데이터가 바뀔 때 집계 값도 함께 갱신하는 흐름을, 두 번째 화면은 원본을 다시 세지 않고 저장된 집계 값을 조회하는 흐름을 보여준다.

![Counter Table을 갱신하는 예시](/images/posts/pagination-optimization/legacy-05.png "Counter Table 갱신")

![Counter Table에서 집계 값을 조회하는 예시](/images/posts/pagination-optimization/legacy-06.png "Counter Table 조회")

Counter Table은 Count 비용을 없애는 만능 해결책이 아니다. 원본 데이터와 집계 값의 정합성을 유지해야 하고, 동시 수정과 실패 복구도 설계해야 한다. 실제 Count가 병목인지 측정한 뒤 복잡성을 감수할 가치가 있을 때 선택한다.

여기까지는 “얼마나 빠르게 조회할 수 있는가”를 봤다. 하지만 사용자가 여러 페이지를 순서대로 보는 동안 데이터가 계속 추가되거나 수정되는 서비스라면 한 가지 문제가 더 생긴다.

**다음 페이지에서 같은 데이터를 또 보거나, 반대로 어떤 데이터를 건너뛰지는 않는가?**

## 조회하는 동안 데이터가 바뀌면 중복과 누락이 생길 수 있다

Offset 방식으로 최신 게시글을 세 건씩 조회한다고 해보자.

```text
첫 페이지
[10, 9, 8]
```

사용자가 다음 페이지를 누르기 전에 게시글 11이 맨 앞에 추가됐다.

```text
현재 전체 목록
[11, 10, 9, 8, 7, 6, 5]
```

두 번째 페이지에서 `OFFSET 3`을 적용하면 다음 결과가 나온다.

```text
두 번째 페이지
[8, 7, 6]
```

8은 첫 페이지에서 이미 본 데이터인데 다시 나타난다. 앞쪽에 11이 추가되면서 기존 데이터의 Offset 위치가 한 칸씩 밀렸기 때문이다. 반대 방향의 변경에서는 어떤 데이터를 건너뛸 수도 있다.

Keyset은 마지막으로 본 값 8을 기준으로 이어 읽는다.

```text
마지막으로 본 값
→ 8

다음 조회
→ id < 8

결과
→ [7, 6, 5]
```

앞쪽에 새 데이터가 추가돼도 마지막 위치보다 뒤의 범위를 조회하므로 Offset보다 영향을 덜 받는다. 최신 데이터가 계속 추가되는 피드나 무한 스크롤에서 Keyset이 잘 맞는 이유다.

### Keyset도 모든 페이지를 같은 시점으로 고정하지는 않는다

Keyset이 모든 변경에 안정적인 것은 아니다. 앞쪽에 새 행이 추가되는 영향은 줄일 수 있지만, 사용자가 페이지를 넘기는 사이 기존 행의 정렬 값이 바뀌면 Cursor 앞뒤로 이동할 수 있다. 행이 삭제돼도 이후 결과는 달라질 수 있다.

즉 Keyset은 여러 페이지를 자동으로 동일한 시점의 결과로 만들지 않는다. 첫 페이지를 조회한 순간의 데이터를 끝까지 그대로 보여줘야 한다면 기준 시각을 Cursor에 포함하거나 조회할 ID 목록을 미리 고정하는 등 별도의 설계가 필요하다. 이렇게 조회 대상을 특정 시점으로 고정한 결과를 **Snapshot**이라고 한다.

이제 성능뿐 아니라 화면 요구와 데이터 변경 특성까지 포함해 두 방식을 선택할 수 있다.

## Offset과 Keyset을 선택하는 세 가지 기준

### 1. 화면의 탐색 방식

- 페이지 번호와 임의 페이지 이동이 필요한가?
- 아니면 다음 데이터를 계속 이어 보는 것으로 충분한가?

### 2. 데이터 규모와 실제 쿼리 성능

- 사용자가 깊은 페이지까지 자주 이동하는가?
- 깊은 Offset이 실행 계획과 측정 결과에서 실제 병목으로 나타나는가?
- 정렬과 Cursor 조건에 맞는 인덱스를 만들 수 있는가?

### 3. 데이터의 변경 특성

- 사용자가 페이지를 넘기는 동안 새 데이터가 자주 추가되는가?
- 기존 데이터의 정렬 값이 자주 바뀌는가?
- 여러 페이지를 같은 시점으로 고정해야 하는가?

Offset이 잘 맞는 경우는 다음과 같다.

- 페이지 번호가 중요한 관리자 화면
- 임의 페이지로 바로 이동해야 하는 목록
- 데이터 규모가 크지 않음
- 깊은 페이지 조회가 드묾
- 구현 단순성과 정확한 전체 페이지 수가 중요함

Keyset이 잘 맞는 경우는 다음과 같다.

- 무한 스크롤과 피드
- 최신 데이터가 계속 추가되는 목록
- 데이터가 크고 깊은 Offset이 실제 병목임
- 이전 페이지 번호보다 다음 데이터가 중요함
- 안정적인 정렬 기준과 이에 맞는 인덱스를 구성할 수 있음

Keyset이 무조건 더 빠른 것은 아니다. 정렬과 맞는 인덱스, 올바른 Cursor 조건, 페이지 번호 직접 이동이 필요하지 않은 화면이라는 조건이 맞아야 장점을 얻는다. 데이터가 작고 임의 이동이 중요한 화면에서는 Offset의 단순성이 더 큰 가치가 될 수 있다.

선택 흐름을 간단히 정리하면 다음과 같다.

```text
페이지 번호와 임의 이동이 꼭 필요한가?
↓
YES → Offset 우선 검토

NO
↓
다음 데이터만 이어 보면 되는가?
↓
YES → Keyset 검토

데이터가 큰가?
깊은 Offset이 실제로 느린가?
정렬 기준을 안정적으로 만들 수 있는가?
↓
실행 계획과 실제 시간으로 최종 확인
```

## Cursor 값을 하나의 문자열로 전달할 수 있다

복합 Cursor의 `created_at`과 `id`를 쿼리 파라미터 두 개로 받을 수도 있다. API에서 Cursor를 하나의 값으로 전달하고 싶다면 두 값을 JSON으로 묶은 뒤 Base64URL 문자열로 바꿀 수 있다.

```text
원본 Cursor
{"createdAt":"2026-03-05T10:00:00","id":101}

API Cursor
→ Base64URL 문자열
```

Base64는 암호화가 아니다. 누구나 원래 값으로 되돌릴 수 있다. 클라이언트가 Cursor 값을 임의로 바꾸지 못하게 해야 한다면 서버 비밀키로 서명하는 방법을 함께 검토한다. Cursor를 알아내기 어렵게 만드는 것과 해당 데이터를 조회할 권한을 검사하는 것도 별개의 문제이므로 인가는 따로 적용해야 한다.

## 정리

- 페이징은 많은 데이터 중 현재 화면에 필요한 구간만 데이터베이스에서 조회하는 방법이다.
- Offset은 특정 행으로 순간 이동하지 않고 앞에서 지정한 건수만큼 지나친 뒤 결과를 반환한다.
- 정렬에 맞는 인덱스가 있어도 깊은 Offset에서 앞부분을 건너뛰는 비용은 남을 수 있다.
- Keyset은 페이지 번호 대신 마지막 정렬 위치를 기억하고 그 뒤의 범위를 이어 읽는다.
- 안정적인 Cursor는 `ORDER BY`에 사용한 모든 값과 같은 방향의 비교 조건을 가져야 한다.
- Offset·Keyset은 데이터베이스에서 위치를 찾는 방식이고 Page·Slice·Window는 결과와 위치를 표현하는 방식이다.
- Page는 목록 쿼리와 별도로 Count 쿼리를 실행할 수 있으므로 두 실행 계획을 따로 확인해야 한다.
- Keyset은 앞쪽에 데이터가 추가될 때 Offset보다 중복과 누락이 적지만, 여러 페이지를 자동으로 같은 시점의 결과로 고정하지는 않는다.

**Offset과 Keyset 중 하나가 항상 더 좋은 것은 아니다. 페이지 번호가 필요한지, 깊은 페이지 조회가 실제 병목인지, 데이터를 조회하는 동안 얼마나 자주 변경되는지를 함께 보고 선택해야 한다.**

## 참고 자료

### 공식 자료

- [Spring Data Commons - Paging, Iterating Large Results, Sorting & Limiting](https://docs.spring.io/spring-data/commons/reference/repositories/query-methods-details.html#repositories.paging-and-sorting)
- [Spring Data JPA - Scrolling Large Query Results](https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html#jpa.query-methods.scroll)
- [MySQL - LIMIT Query Optimization](https://dev.mysql.com/doc/refman/8.4/en/limit-optimization.html)
