+++
title = 'MySQL 인덱스와 실행 계획을 함께 읽어야 하는 이유'
date = 2026-03-03T19:00:00+09:00
lastmod = 2026-08-07T16:49:40+09:00
draft = false
description = 'InnoDB의 B+Tree, Clustered·Secondary Index와 복합 인덱스 순서를 이해하고 EXPLAIN과 EXPLAIN ANALYZE로 검증하는 방법을 정리합니다.'
categories = ['데이터베이스']
tags = ['MySQL', '인덱스', 'EXPLAIN', 'B+Tree']
+++

인덱스는 특정 Column에 붙이는 빠른 검색 옵션이 아니다. 어떤 값을 어떤 순서로 저장하고, 그 끝에서 원본 행을 어떻게 찾을지 정하는 별도의 자료구조다.

인덱스가 없을 때 데이터베이스는 조건에 맞는 행을 찾기 위해 Table의 행을 처음부터 끝까지 확인할 수밖에 없다. 100만 행 중 한 건을 찾더라도 운이 나쁘면 거의 100만 건을 비교한다. 인덱스는 값을 정렬된 형태로 별도 보관해 확인하지 않아도 되는 범위를 빠르게 제외한다.

인덱스를 만들었다는 사실만으로 Query가 빨라지지도 않는다. `WHERE`, `JOIN`, `ORDER BY`, 반환 Column과 데이터 분포에 따라 Optimizer가 다른 경로를 선택한다. 설계와 실행 계획을 함께 봐야 하는 이유다.

## InnoDB는 왜 B+Tree를 사용하는가

![B-Tree와 Binary Tree의 탐색 구조 비교](/images/posts/mysql-index-and-explain/legacy-01.jpg "B-Tree와 Binary Tree")

![B+Tree의 Root, Branch와 Leaf Node 구조](/images/posts/mysql-index-and-explain/legacy-03.png "B+Tree 구조")

균형 잡힌 Tree는 데이터가 늘어도 탐색 깊이가 급격히 커지지 않는다. B+Tree는 한 Node에 여러 Key를 저장해 높이를 낮추고, Leaf Node를 정렬된 순서로 연결한다.

일반적인 이진 탐색 Tree는 한 Node에서 왼쪽과 오른쪽 두 갈래로 나뉜다. 하지만 데이터베이스는 저장 장치에서 Page 하나를 읽는 비용이 크다. 한 Page에 많은 Key와 자식 Pointer를 담아 한 번에 수십·수백 갈래를 판단할 수 있다면 Tree 높이와 Page 접근 횟수를 줄일 수 있다. 이것이 B-Tree 계열이 데이터베이스 인덱스에 잘 맞는 이유다.

B+Tree에서는 실제 탐색에 필요한 Key가 내부 Node에 있고, Leaf Node가 정렬된 상태로 서로 연결된다. `id = 100`은 Tree를 따라 Leaf로 내려가 찾고, `id between 100 and 200`은 시작 Leaf를 찾은 뒤 옆 Leaf를 순서대로 읽을 수 있다.

이 구조는 두 종류의 조회에 잘 맞는다.

- `id = 100`처럼 한 Key를 찾는 조회
- `created_at between ...`처럼 연속 범위를 순서대로 읽는 조회

데이터베이스는 Page 단위로 저장 장치를 읽는다. 한 Page에 여러 Key와 Pointer를 담아 Tree 높이를 낮추면 필요한 Page 접근 횟수를 줄일 수 있다. 따라서 메모리 안의 이진 탐색 시간 복잡도만으로 데이터베이스 인덱스를 설명하기보다 Page I/O와 정렬된 범위 탐색을 함께 보는 편이 정확하다.

이 B+Tree가 실제 행을 어디에 저장하는지에 따라 한 번의 탐색으로 끝날지, 다른 Tree를 다시 찾아갈지가 달라진다. InnoDB의 Clustered Index와 Secondary Index를 구분해야 하는 이유다.

## Clustered Index와 Secondary Index

![Clustered Index와 Secondary Index의 탐색 경로](/images/posts/mysql-index-and-explain/legacy-02.png "Clustered Index와 Secondary Index")

InnoDB Table에는 행 데이터를 담는 Clustered Index가 하나 있다. 일반적으로 Primary Key가 Clustered Index가 된다.

```text
Primary Key 탐색 → Leaf Page에 행 데이터 존재
```

Primary Key 외의 인덱스는 Secondary Index다. Secondary Index의 Leaf에는 지정한 Key와 해당 행의 Primary Key가 들어 있다.

```text
Secondary Index에서 Key 탐색
→ Primary Key 획득
→ Clustered Index에서 행 조회
```

필요한 Column이 Secondary Index에 모두 있지 않으면 이 두 번째 탐색이 발생한다. Primary Key가 길면 모든 Secondary Index에 포함되는 값도 길어져 저장 공간이 늘어날 수 있다.

예를 들어 `member(email)`에 Secondary Index가 있고 다음 Query를 실행한다고 하자.

```sql
SELECT name
  FROM member
 WHERE email = 'user@example.com';
```

Secondary Index의 Leaf에 `email`과 Primary Key `member_id`만 있다면 다음 두 번의 Tree 탐색이 필요하다.

```text
email 인덱스에서 user@example.com 탐색
→ member_id = 42 확인
→ Clustered Index에서 42 탐색
→ 행 데이터의 name 반환
```

조건에 맞는 행이 많을수록 두 번째 탐색도 반복된다. Optimizer가 많은 행을 가져올 때 Secondary Index 대신 Full Scan이 더 싸다고 판단할 수 있는 이유 중 하나다.

Secondary Index를 설계할 때는 Column을 포함하는 것만으로 충분하지 않다. B+Tree가 앞에서부터 정렬된다는 성질 때문에 복합 인덱스의 Column 순서가 실제 탐색 범위를 결정한다.

## 복합 인덱스는 순서가 핵심이다

![복합 인덱스 실습에 사용한 테이블과 인덱스 구성](/images/posts/mysql-index-and-explain/legacy-04.png "복합 인덱스 실습 구성")

다음 인덱스는 `(status, created_at, id)` 순서로 정렬된다.

```sql
CREATE INDEX idx_post_status_created_id
    ON post(status, created_at DESC, id DESC);
```

이 인덱스는 다음 Query와 잘 맞을 수 있다.

```sql
SELECT id, title, created_at
  FROM post
 WHERE status = 'PUBLISHED'
   AND created_at < ?
 ORDER BY created_at DESC, id DESC
 LIMIT 20;
```

선두 Column인 `status`를 Equality로 좁힌 뒤 뒤의 `created_at`, `id` 순서를 활용할 수 있다. 반대로 `created_at`만 조건에 두면 선두 `status`를 건너뛰므로 같은 방식의 탐색을 기대하기 어렵다.

복합 인덱스는 각 Column이 따로 정렬된 인덱스 세 개가 아니다. 전체 값이 사전식 순서로 정렬된 하나의 인덱스다.

```text
(DRAFT,     2026-08-01, 10)
(DRAFT,     2026-08-02, 14)
(PUBLISHED, 2026-07-30,  7)
(PUBLISHED, 2026-08-01,  9)
```

먼저 `status`로 묶이고, 같은 status 안에서만 `created_at`이 정렬된다. 따라서 status를 모른 채 created_at만 찾으려 하면 인덱스 전체 곳곳을 확인해야 한다. 이것이 복합 인덱스의 선두 Column이 중요한 이유다.

Equality 조건이 이어지는 동안에는 다음 Column의 정렬을 활용하기 쉽다. Range 조건이 시작되면 그 뒤 Column은 탐색 범위를 좁히는 데 제한이 생길 수 있다. 예를 들어 `(status, created_at, id)`에서 status는 Equality, created_at은 Range라면 id는 정렬이나 추가 Filter에 쓰일 수 있어도 같은 수준의 범위 탐색을 보장하지 않는다.

“선택도가 높은 Column을 무조건 앞에 둔다”는 규칙은 충분하지 않다. Equality 조건, Range가 시작되는 위치, 정렬과 Grouping, 실제 조회 패턴을 함께 봐야 한다. 선택도는 후보를 비교하는 자료이지 순서를 자동으로 결정하는 공식이 아니다.

이 순서가 Query의 조건과 맞으면 탐색 범위뿐 아니라 원본 행을 다시 찾는 비용도 줄일 수 있다. 필요한 Column까지 Index 안에 있다면 Secondary Index만으로 결과를 만들 수 있기 때문이다.

## Covering Index는 원본 행 조회를 줄인다

Query에 필요한 모든 Column이 Index에 들어 있으면 Clustered Index를 다시 찾지 않고 결과를 만들 수 있다.

```sql
SELECT id, created_at
  FROM post
 WHERE status = 'PUBLISHED'
 ORDER BY created_at DESC, id DESC
 LIMIT 20;
```

앞의 복합 인덱스에는 Secondary Key와 Primary Key `id`가 있으므로 Covering이 가능할 수 있다. 실행 계획의 `Extra`에 `Using index`가 표시되는지 확인할 수 있다.

그렇다고 응답 Column을 전부 Index에 넣으면 쓰기 비용과 저장 공간이 커진다. 빈도가 높고 성능상 중요한 Query에 제한해서 적용한다.

Covering Index는 별도의 인덱스 종류가 아니다. **현재 Query가 필요로 하는 값이 우연히 한 인덱스에 모두 들어 있는 상태**를 뜻한다. 같은 인덱스라도 `SELECT id, created_at`에는 Covering이 될 수 있고, 인덱스에 없는 `content`까지 조회하면 더는 Covering이 아니다.

여기까지는 설계 단계에서 예상한 동작이다. Optimizer가 실제로 같은 Index를 선택했고 몇 행을 읽는지는 실행 계획으로 확인해야 한다.

## EXPLAIN에서 먼저 볼 값

실행계획은 한 열만 따로 읽기보다 접근 방식, 선택한 인덱스, 비교 대상과 예상 행 수를 한 흐름으로 확인하는 편이 이해하기 쉽다. 다음 화면은 각 값을 실제 `EXPLAIN` 결과에서 확인한 예시다.

![EXPLAIN의 기본 실행계획](/images/posts/mysql-index-and-explain/legacy-05.png "EXPLAIN 기본 결과")

![상수 조건에서 ref가 const로 표시된 실행계획](/images/posts/mysql-index-and-explain/legacy-06.png "ref가 const인 경우")

![Join 조건의 컬럼이 ref에 표시된 실행계획](/images/posts/mysql-index-and-explain/legacy-07.png "Join에서 ref를 읽는 방법")

![rows와 filtered를 함께 확인하는 실행계획](/images/posts/mysql-index-and-explain/legacy-08.png "rows와 filtered")

![인덱스 구성 이후 달라진 실행계획](/images/posts/mysql-index-and-explain/legacy-09.png "인덱스 적용 후 실행계획")

```sql
EXPLAIN
SELECT id, title
  FROM post
 WHERE status = 'PUBLISHED'
 ORDER BY created_at DESC
 LIMIT 20;
```

주요 Column은 다음과 같다.

- `type`: Table에 접근하는 방식. `ALL`은 Full Scan, `range`는 범위 탐색, `ref`와 `const`는 더 제한적인 탐색을 의미한다.
- `possible_keys`: 후보가 될 수 있는 Index
- `key`: 실제 선택한 Index
- `rows`: Optimizer가 읽을 것으로 추정한 행 수
- `filtered`: 읽은 행 중 조건을 통과할 것으로 추정한 비율
- `Extra`: Covering, 정렬과 임시 결과 같은 추가 정보

`rows`는 실제 행 수가 아니라 통계에 기반한 추정치다. Query를 실제로 실행할 수 있는 환경에서는 `EXPLAIN ANALYZE`로 예상과 실제를 비교한다.

처음 실행 계획을 읽을 때는 다음 순서가 단순하다.

```text
1. 실제 선택한 key가 있는가
2. type이 ALL인지, range·ref처럼 범위를 줄였는가
3. rows가 결과 건수에 비해 지나치게 크지 않은가
4. filtered 이후 얼마나 많은 행이 버려질 것으로 보이는가
5. Extra에 별도 정렬이나 임시 결과가 있는가
```

예를 들어 결과는 20건인데 `rows`가 80만 건이라면 인덱스 이름이 표시됐다는 사실만으로 만족할 수 없다. 많은 인덱스 행을 읽은 뒤 대부분을 버리고 있을 수 있다.

```sql
EXPLAIN ANALYZE
SELECT ...;
```

`EXPLAIN ANALYZE`는 Query를 실제 실행하고 각 Iterator의 실제 시간, 반환 행 수와 반복 횟수를 보여준다. 변경 Query나 운영의 무거운 Query에는 실행 자체가 영향을 줄 수 있으므로 주의한다.

실행 계획이 익숙하지 않다면 Visual Explain으로 Nested Loop와 Table 접근 순서를 먼저 읽는 것도 도움이 된다. 하지만 화면에서 Index 접근이 초록색으로 보인다는 이유만으로 설계가 끝난 것은 아니다.

예를 들어 `(ad_id, status, created_at)` 복합 인덱스를 선택한 실행 계획에서도 실제 탐색에는 `ad_id`, `status`만 사용될 수 있다. 조회 기간 조건을 거의 모든 행이 만족한다면 Optimizer는 `created_at`까지 Index 범위를 좁히는 것보다 행을 읽고 조건을 판단하는 편이 싸다고 볼 수 있기 때문이다.

```text
key            = idx_ad_id_status_created_at
used_key_parts = ad_id, status
```

이 경우 실행 계획에서 인덱스 이름만 보면 세 Column이 모두 도움이 된 것처럼 보이지만 `used_key_parts`는 다른 사실을 보여준다. 반복 검증 결과 `created_at`이 계속 탐색에 기여하지 않는다면 더 짧은 `(ad_id, status)` 인덱스도 후보가 된다. 화면의 색보다 선택한 Index, 실제 사용한 Key 부분, 예상·실제 행 수와 실행 시간을 함께 확인해야 한다.

Index를 사용했는지 확인한 뒤에는 `Extra`에 나타나는 추가 작업도 해석해야 한다. 별도 정렬이나 임시 결과가 있다는 표시는 실패 판정이 아니라 실제 비용을 더 살펴보라는 신호다.

## Using filesort와 Using temporary

`Using filesort`는 정렬을 Index 순서만으로 끝내지 못해 별도 정렬 단계가 생겼다는 뜻이다. 이름에 file이 들어가지만 항상 Disk 정렬이라는 뜻은 아니다. 결과가 작다면 충분히 빠를 수 있고, 큰 범위를 반복해서 정렬한다면 병목이 될 수 있다.

`Using temporary`는 Grouping, DISTINCT나 복잡한 정렬을 처리하기 위해 내부 임시 결과를 사용했다는 신호다. 이것도 표시 자체가 실패는 아니다. 실제 처리 행 수, 메모리·Disk 사용과 실행 시간을 함께 확인해야 한다.

예를 들어 상태가 세 종류뿐인 작은 Table을 Grouping하며 임시 결과를 만드는 비용은 미미할 수 있다. 반대로 수백만 행을 Join한 뒤 큰 중간 결과를 임시 Table에 담고 정렬한다면 비용이 커진다. 표시의 존재보다 **임시 결과에 들어간 행 수와 실제 소요 시간**이 중요하다.

이 신호를 줄이려면 먼저 Query와 Index가 어긋난 이유를 찾아야 한다. 단순히 새 Index를 추가하기보다 조건식이 정렬 값을 훼손하거나 복합 인덱스의 선두를 건너뛰지는 않았는지 확인한다.

## 인덱스를 타지 못하는 흔한 이유

다음 예시는 쿼리 형태를 바꿨을 때 옵티마이저의 선택이 어떻게 달라지는지 보여준다. 인덱스가 있다는 사실보다 해당 조건이 B+Tree의 시작 위치와 범위를 결정할 수 있는지가 중요하다.

![인덱스 컬럼에 함수를 적용한 실행계획](/images/posts/mysql-index-and-explain/legacy-10.png "인덱스 컬럼에 함수를 적용한 경우")

![암묵적 형변환이 포함된 실행계획](/images/posts/mysql-index-and-explain/legacy-11.png "암묵적 형변환")

![OR 조건에서 index_merge가 선택된 실행계획](/images/posts/mysql-index-and-explain/legacy-12.png "OR 조건과 index_merge")

![앞쪽 와일드카드가 포함된 LIKE 검색](/images/posts/mysql-index-and-explain/legacy-13.png "앞쪽 와일드카드 LIKE")

![앞쪽 와일드카드 유무에 따른 실행계획 비교](/images/posts/mysql-index-and-explain/legacy-14.png "LIKE 검색의 실행계획 비교")

![SELECT *로 Covering Index가 깨진 실행계획](/images/posts/mysql-index-and-explain/legacy-15.png "SELECT *와 Covering Index")

![필요한 컬럼만 조회한 실행계획](/images/posts/mysql-index-and-explain/legacy-16.png "필요한 컬럼만 조회한 경우")

### OR과 UNION ALL의 실행계획 비교

`OR`은 여러 인덱스 결과를 하나의 실행계획 안에서 병합할 수 있다. 같은 조건을 독립된 `SELECT`로 나누어 `UNION ALL`로 연결하면 각 쿼리가 자신에게 맞는 인덱스를 선택할 여지가 생긴다. 어느 쪽이 유리한지는 중복 가능성과 실제 실행계획을 함께 확인해야 한다.

![OR 조건에서 결과를 병합하는 실행계획](/images/posts/mysql-index-and-explain/legacy-17.png "OR의 index_merge 실행계획")

![UNION ALL로 나눈 쿼리의 실행계획](/images/posts/mysql-index-and-explain/legacy-18.png "UNION ALL 실행계획")

- 인덱스 Column에 함수를 적용해 원래 정렬 값을 바로 비교할 수 없음
- 문자열 Column과 숫자처럼 타입이 다른 값을 비교해 암시적 변환 발생
- 선두 Column을 사용하지 않는 복합 인덱스
- 너무 많은 행을 반환해 Full Scan이 더 싸다고 판단
- `LIKE '%keyword'`처럼 앞부분이 열려 있는 검색
- 정렬 방향과 조건이 인덱스 구성과 맞지 않음

`OR`을 `UNION ALL`로 바꾸면 무조건 빨라진다는 규칙도 없다. 각 조건이 서로 다른 Index를 효율적으로 사용하고 중복 가능성을 통제할 수 있을 때 후보가 된다. 최종 판단은 실행 계획과 실제 시간으로 한다.

## 정리

- InnoDB의 Clustered Index Leaf에는 행 데이터가 저장된다.
- Secondary Index는 Primary Key를 통해 원본 행을 다시 찾을 수 있다.
- 복합 인덱스 순서는 선택도 하나가 아니라 Equality, Range와 정렬을 함께 보고 결정한다.
- EXPLAIN의 `rows`는 추정치이며 실제 검증에는 `EXPLAIN ANALYZE`가 유용하다.
- `Using filesort`와 `Using temporary`는 확인할 신호이지 그 자체로 실패는 아니다.

## 참고 자료

### 공식 자료

- [MySQL - Clustered and Secondary Indexes](https://dev.mysql.com/doc/refman/8.4/en/innodb-index-types.html)
- [MySQL - Column Indexes](https://dev.mysql.com/doc/refman/8.4/en/column-indexes.html)
- [MySQL - EXPLAIN Statement](https://dev.mysql.com/doc/refman/8.4/en/explain.html)

### 국내 기술 블로그

- [LINE Engineering - MySQL Workbench의 VISUAL EXPLAIN으로 인덱스 동작 확인하기](https://engineering.linecorp.com/ko/blog/mysql-workbench-visual-explain-index)
