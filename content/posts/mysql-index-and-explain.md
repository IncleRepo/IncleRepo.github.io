+++
title = 'MySQL 인덱스와 실행 계획을 함께 읽어야 하는 이유'
slug = '2'
aliases = ['/posts/002/']
date = 2026-03-03T19:00:00+09:00
lastmod = 2026-08-10T17:12:37+09:00
draft = false
description = 'MySQL 인덱스가 범위를 줄이는 원리부터 복합 인덱스 설계와 EXPLAIN, EXPLAIN ANALYZE를 이용한 검증 방법까지 차례대로 알아봅니다.'
categories = ['데이터베이스']
tags = ['MySQL', '인덱스', 'EXPLAIN', 'B+Tree']
+++

회원이 100명인 테이블에서 이메일 하나를 찾는 것은 큰 문제가 아니다. 인덱스가 없어도 행을 처음부터 확인하면 금방 원하는 회원을 찾을 수 있다.

하지만 회원이 100만 명이라면 상황이 달라진다. 원하는 회원 한 명을 찾을 때마다 수십만 행을 확인하는 방식은 부담이 된다. 데이터베이스는 원하는 행을 더 빨리 찾기 위해 정렬된 별도의 탐색 구조를 둘 수 있다. 이것이 **인덱스**다.

그렇다고 인덱스를 만들기만 하면 모든 쿼리가 자동으로 빨라지는 것은 아니다. 쿼리의 조건과 정렬 방식이 인덱스의 정렬 구조와 맞아야 하고, MySQL이 실제 실행 경로로 그 인덱스를 선택해야 한다.

이 글에서는 인덱스가 왜 빠른지부터 시작한다. 인덱스에서 실제 행을 찾는 과정과 복합 인덱스의 컬럼 순서를 이해한 뒤, `EXPLAIN`과 `EXPLAIN ANALYZE`로 예상한 동작이 실제로 선택됐는지 확인해본다.

## 테이블이 커지면 왜 조회가 느려질까

다음 쿼리를 실행한다고 해보자.

```sql
SELECT *
  FROM member
 WHERE email = 'user@example.com';
```

`email`을 빠르게 찾을 구조가 없다면 MySQL은 테이블의 행을 앞에서부터 확인할 수 있다. 조건에 맞지 않는 행을 하나씩 건너뛰며 원하는 값을 찾는 방식이다. 이렇게 테이블 전체를 읽는 접근을 **Full Scan**이라고 한다.

행이 적거나 결과가 테이블 대부분을 차지한다면 Full Scan도 합리적인 선택일 수 있다. 문제는 100만 행 중 한두 건만 필요할 때도 거의 모든 행을 확인하는 경우다.

인덱스는 `email` 값을 정렬된 형태로 따로 보관한다. `user@example.com`이 있을 만한 위치로 바로 이동할 수 있으므로, 관계없는 이메일을 전부 비교하지 않고 탐색 범위를 줄일 수 있다.

실무에서는 흔히 “이 쿼리 인덱스 타나요?”라고 묻는다. MySQL이 해당 인덱스를 실제 조회 경로로 선택했는지 묻는 말이다. 하지만 인덱스를 사용했는지만 확인해서는 부족하다. 그 인덱스로 읽을 범위를 얼마나 줄였는지까지 봐야 한다.

그렇다면 인덱스는 어떤 구조이길래 정렬된 값에서 원하는 위치를 빠르게 찾을 수 있을까?

## B+Tree는 읽어야 할 Page를 줄인다

데이터베이스는 저장 장치에서 필요한 값 하나만 딱 떼어 읽지 않는다. 일정 크기로 묶인 **Page** 단위로 데이터를 읽는다. 이렇게 Page를 읽고 쓰는 작업을 Page I/O라고 한다. 따라서 비교 연산 몇 번을 줄였는가보다 원하는 값을 찾기 위해 몇 개의 Page를 읽었는지가 성능에 큰 영향을 준다.

한 Node에 Key가 두 개뿐이면 데이터가 늘어날수록 Tree가 깊어질 수 있다. 반면 한 Page에 수십·수백 개의 Key와 다음 Page를 가리키는 Pointer를 담으면, Page 하나를 읽고도 많은 범위를 제외할 수 있다.

```text
한 Node에 Key가 적음
→ 갈라지는 경로가 적음
→ Tree가 깊어질 수 있음

한 Page에 Key와 Pointer가 많음
→ 한 번 읽고 많은 범위를 제외
→ Tree 높이와 Page 접근 횟수를 줄일 수 있음
```

다음 그림은 같은 수의 값을 저장해도 한 Node에서 여러 갈래로 나뉘는 Tree가 더 낮은 높이를 유지할 수 있음을 보여준다.

![B-Tree와 Binary Tree의 탐색 구조 비교](/images/posts/mysql-index-and-explain/legacy-01.jpg "B-Tree와 Binary Tree")

InnoDB의 인덱스는 B-Tree 계열 구조를 사용한다. 실제 구현은 Leaf Page에 정렬된 Key를 모으고, Leaf끼리 다음 범위를 이어서 읽을 수 있는 B+Tree 형태로 이해하면 쉽다.

다음 그림처럼 탐색할 때는 Root에서 Branch를 거쳐 원하는 Leaf로 내려가고, 범위를 읽을 때는 서로 연결된 Leaf Page를 순서대로 따라간다.

![B+Tree의 Root, Branch와 Leaf Node 구조](/images/posts/mysql-index-and-explain/legacy-03.png "B+Tree 구조")

단건 조회는 Tree를 따라 원하는 Leaf Page까지 내려간다.

```sql
SELECT *
  FROM member
 WHERE id = 100;
```

```text
Root Page에서 id = 100이 속한 범위 선택
→ Branch Page에서 범위를 다시 축소
→ Leaf Page에서 id = 100 탐색
```

범위 조회는 시작 Key를 찾은 뒤 정렬된 Leaf Page를 옆으로 이어서 읽을 수 있다.

```sql
SELECT *
  FROM member
 WHERE id BETWEEN 100 AND 200;
```

```text
id = 100이 있는 Leaf Page 탐색
→ 연결된 Leaf Page를 순서대로 읽음
→ id = 200에서 종료
```

B+Tree가 단건 조회뿐 아니라 범위 조회와 정렬에도 유리한 이유다. 단순히 `O(log N)`이라는 시간 복잡도만 외우기보다, 정렬된 Key로 시작 위치를 찾고 필요한 Page만 읽는 구조라고 이해하는 편이 실용적이다.

여기까지는 인덱스 안에서 원하는 Key를 찾는 과정이었다. 하지만 쿼리가 최종적으로 필요한 것은 `member`의 실제 행이다. 이제 인덱스에서 찾은 Key가 실제 데이터로 어떻게 이어지는지 살펴보자.

## 인덱스에서 실제 회원 행까지 찾아가는 방법

먼저 Primary Key로 회원을 조회해보자.

```sql
SELECT *
  FROM member
 WHERE id = 42;
```

InnoDB에서는 일반적으로 Primary Key 인덱스의 Leaf Page에 실제 회원 행이 함께 저장된다.

```text
Primary Key 인덱스에서 id = 42 탐색
→ Leaf Page에서 실제 member 행 확인
```

Primary Key가 있는 InnoDB 테이블은 실제 행 데이터를 Primary Key 순서로 저장한다. 이처럼 Leaf Page에 실제 행을 담는 인덱스가 **Clustered Index**다. 테이블마다 하나만 존재하며 일반적으로 Primary Key가 이 역할을 맡는다.

이번에는 이메일에 별도 인덱스가 있고 다음 쿼리를 실행한다고 해보자.

```sql
SELECT name
  FROM member
 WHERE email = 'user@example.com';
```

이메일처럼 Primary Key가 아닌 컬럼에 만든 인덱스는 **Secondary Index**다. Leaf Page에는 원본 행 전체가 아니라 인덱스 컬럼 값과 해당 행의 Primary Key가 저장된다.

```text
email 인덱스에서 user@example.com 탐색
→ member_id = 42 확인
→ Primary Key 인덱스에서 id = 42를 다시 탐색
→ 실제 member 행의 name 반환
```

다음 그림은 Secondary Index에서 Primary Key를 찾은 뒤 Clustered Index를 다시 탐색해 실제 행에 도달하는 과정을 보여준다.

![Clustered Index와 Secondary Index의 탐색 경로](/images/posts/mysql-index-and-explain/legacy-02.png "Clustered Index와 Secondary Index")

Secondary Index만으로 `name`을 반환할 수 없다면 이렇게 Tree를 두 번 탐색할 수 있다. 조건에 맞는 회원이 많으면 Primary Key로 실제 행을 다시 찾는 과정도 반복된다. 많은 행을 가져오는 쿼리에서 MySQL이 Secondary Index보다 Full Scan이 더 저렴하다고 판단할 수 있는 이유 중 하나다.

Primary Key 값은 Secondary Index의 Leaf마다 함께 저장된다. 따라서 Primary Key가 길면 모든 Secondary Index의 크기도 커질 수 있다. Primary Key 설계가 다른 인덱스의 저장 공간과 Page 효율에도 영향을 주는 이유다.

인덱스 하나의 탐색 경로를 이해했다면 다음 질문은 여러 조건을 사용하는 쿼리다. `status`, `created_at`, `id`를 함께 조건과 정렬에 사용한다면 세 인덱스를 따로 만들면 될까?

## 복합 인덱스는 여러 컬럼을 하나의 순서로 정렬한다

`(status, created_at, id)` 인덱스를 만들면 세 컬럼이 각각 따로 정렬된다고 생각하기 쉽다. 실제로는 세 값을 하나의 묶음으로 보고 왼쪽부터 사전식으로 정렬한 **하나의 인덱스**다.

```sql
CREATE INDEX idx_post_status_created_id
    ON post(status, created_at DESC, id DESC);
```

정렬된 값의 모양은 다음과 같다.

```text
(DRAFT,     2026-08-01, 10)
(DRAFT,     2026-08-02, 14)
(PUBLISHED, 2026-07-30,  7)
(PUBLISHED, 2026-08-01,  9)
```

먼저 `status`로 묶이고, 같은 status 안에서 `created_at`이 정렬된다. 날짜까지 같다면 마지막으로 `id` 순서를 사용한다.

다음 쿼리는 이 정렬 구조와 잘 맞을 수 있다.

```sql
SELECT id, title, created_at
  FROM post
 WHERE status = 'PUBLISHED'
   AND created_at < ?
 ORDER BY created_at DESC, id DESC
 LIMIT 20;
```

`status = 'PUBLISHED'`는 status를 하나의 값으로 고정한다. 인덱스에서 PUBLISHED 영역으로 바로 좁힌 뒤, 그 안에 정렬된 `created_at`을 이어서 활용할 수 있다.

반면 `created_at < ?`는 특정 값 하나가 아니라 일정 구간을 읽는 범위 조건이다. 복합 인덱스에서 범위 조건이 시작되면 뒤의 `id`까지 탐색 범위를 줄이는 데 사용하는 방식에는 제약이 생길 수 있다. `id`가 정렬이나 추가 조건 판단에 도움을 줄 수는 있어도, 앞의 `status`처럼 하나의 연속된 구간을 항상 더 좁혀 주는 것은 아니다.

반대로 다음처럼 선두 컬럼인 `status`를 사용하지 않으면 `created_at`만으로 시작 위치를 결정하기 어렵다.

```sql
SELECT id, title, created_at
  FROM post
 WHERE created_at < ?
 ORDER BY created_at DESC
 LIMIT 20;
```

앞에서 본 사전식 정렬을 다시 생각해보면 DRAFT 영역과 PUBLISHED 영역마다 `created_at` 순서가 따로 존재한다. status를 모르면 날짜 하나만 보고 인덱스의 연속된 한 구간으로 바로 이동하기 어렵다. 이것이 복합 인덱스의 선두 컬럼이 중요한 이유다.

다음 실습 화면에는 테이블 컬럼과 실제로 생성한 복합 인덱스의 순서가 함께 표시되어 있다.

![복합 인덱스 실습에 사용한 테이블과 인덱스 구성](/images/posts/mysql-index-and-explain/legacy-04.png "복합 인덱스 실습 구성")

“선택도가 높은 컬럼을 무조건 앞에 둔다”는 규칙만으로는 충분하지 않다. PUBLISHED 값이 많아 선택도가 낮더라도 `status = 'PUBLISHED'`로 먼저 영역을 고정해야 뒤의 날짜 정렬을 효율적으로 사용할 수 있다. `=` 조건으로 어느 컬럼까지 값을 고정하는지, 범위 조건이 어디서 시작되는지, `ORDER BY`와 `GROUP BY`가 어떤 순서인지, 실제 쿼리가 얼마나 자주 실행되는지를 함께 봐야 한다.

복합 인덱스의 순서가 쿼리와 맞으면 탐색 범위를 줄일 수 있다. 여기에 반환할 값까지 인덱스 안에 들어 있다면 실제 행을 다시 찾는 두 번째 Tree 탐색도 생략할 수 있다.

## Covering Index는 인덱스만으로 조회를 끝낸다

이메일 인덱스로 회원을 찾았지만 반환할 `name`이 인덱스에 없다면 Primary Key 42로 Clustered Index를 다시 찾아가야 했다.

```text
email 인덱스에서 회원 탐색
→ member_id = 42 확인
→ name이 인덱스에 없음
→ Primary Key 인덱스에서 실제 행 조회
```

반면 쿼리에 필요한 모든 값이 현재 인덱스 안에 있다면 Secondary Index에서 결과를 바로 만들 수 있다.

```sql
SELECT id, created_at
  FROM post
 WHERE status = 'PUBLISHED'
 ORDER BY created_at DESC, id DESC
 LIMIT 20;
```

앞의 `(status, created_at, id)` 인덱스에는 조건, 정렬과 반환에 필요한 값이 모두 있다.

```text
Secondary Index에서 PUBLISHED 영역 탐색
→ created_at과 id가 이미 Leaf에 있음
→ Clustered Index를 다시 찾지 않고 결과 반환
```

이 경우 MySQL은 인덱스만 읽고 결과를 반환한다. 쿼리에 필요한 컬럼을 모두 담아 실제 행 조회를 생략하게 해 주는 인덱스를 **Covering Index**라고 한다. 고정된 인덱스 종류를 가리키는 말은 아니다. 같은 인덱스라도 쿼리가 어떤 컬럼을 요구하느냐에 따라 Covering Index가 될 수도 있고 아닐 수도 있다.

앞의 인덱스는 `SELECT id, created_at`에 필요한 값을 모두 가지고 있다. 반면 인덱스에 없는 `content`까지 조회하면 실제 행을 다시 찾아야 한다. 다음 두 화면을 비교하면 `SELECT *`일 때는 보이지 않던 `Using index`가 필요한 컬럼만 조회했을 때 `Extra`에 표시되는 것을 확인할 수 있다.

![SELECT *로 Covering Index가 깨진 실행계획](/images/posts/mysql-index-and-explain/legacy-15.png "SELECT *와 Covering Index")

![필요한 컬럼만 조회한 실행계획](/images/posts/mysql-index-and-explain/legacy-16.png "필요한 컬럼만 조회한 경우")

인덱스만으로 조회를 끝내기 위해 모든 응답 컬럼을 인덱스에 넣는 것도 좋은 방법은 아니다. 인덱스가 넓어질수록 저장 공간과 쓰기 비용이 커지고 Page 하나에 담을 수 있는 Key도 줄어든다. 자주 실행되고 성능이 중요한 쿼리에 한해 적용해야 한다.

여기까지는 인덱스 구조를 보고 “이렇게 동작할 것이다”라고 예상한 내용이다. 하지만 실제 쿼리에서 어떤 인덱스를 사용할지는 MySQL이 결정한다. 이제 설계한 경로가 실제로 선택됐는지 검증해야 한다.

## 어떤 인덱스를 사용할지는 MySQL이 결정한다

MySQL은 하나의 쿼리를 실행할 수 있는 여러 방법을 비교하고, 비용이 가장 낮을 것으로 예상되는 경로를 선택한다. 이 판단을 담당하는 구성 요소가 **Optimizer**다.

Optimizer는 사용 가능한 인덱스, 테이블 통계와 데이터 분포, 조건으로 예상되는 결과 건수, 정렬과 Join 비용 등을 비교한다. 인덱스가 존재해도 많은 행을 반환해야 한다면 Full Scan을 선택할 수 있고, 후보가 여러 개라면 개발자가 예상한 것과 다른 인덱스를 고를 수도 있다.

따라서 “인덱스를 만들었다”와 “그 인덱스를 효율적으로 사용했다”는 같은 말이 아니다. MySQL이 예상한 실행 경로를 확인하는 기본 도구가 `EXPLAIN`이다.

## EXPLAIN은 여섯 가지 질문으로 읽는다

다음처럼 조회 쿼리 앞에 `EXPLAIN`을 붙이면 예상 실행 계획을 확인할 수 있다.

```sql
EXPLAIN
SELECT id, title
  FROM post
 WHERE status = 'PUBLISHED'
 ORDER BY created_at DESC
 LIMIT 20;
```

처음 결과 표를 보면 `type`, `possible_keys`, `key`, `rows`, `filtered`, `Extra`가 암호처럼 보일 수 있다. 각 컬럼을 따로 외우기보다 다음 질문을 순서대로 던지면 읽기 쉽다.

```text
1. 사용할 수 있었던 후보 인덱스는 무엇인가?        → possible_keys
2. MySQL이 실제로 고른 인덱스는 무엇인가?         → key
3. 테이블 전체를 읽는가, 범위를 줄여 읽는가?       → type
4. 이 단계에서 대략 몇 행을 읽을 것으로 보는가?    → rows
5. 읽은 행 중 조건을 통과할 비율은 어느 정도인가?  → filtered
6. 별도 정렬이나 임시 결과 같은 추가 작업이 있는가? → Extra
```

먼저 기본 실행 계획에서 후보 인덱스와 실제 선택된 `key`, 접근 방식인 `type`을 찾아보자.

![EXPLAIN의 기본 실행계획](/images/posts/mysql-index-and-explain/legacy-05.png "EXPLAIN 기본 결과")

`type`은 테이블에 접근하는 방식을 나타낸다. `ALL`은 테이블 전체를 읽는 Full Scan이고, `range`는 인덱스의 일정 범위를 읽는다. `ref`는 인덱스 앞부분을 특정 값과 비교해 범위를 좁히며, `const`는 Primary Key나 Unique Index의 상수 조건처럼 한 행 수준으로 매우 강하게 제한되는 접근이다.

다음 화면에서 `ref` 컬럼의 `const`는 인덱스 값을 컬럼이 아닌 상수와 비교했다는 뜻이다. 앞에서 접근 방식으로 살펴본 `type = const`와는 다른 정보이므로 구분해서 읽어야 한다.

![상수 조건에서 ref가 const로 표시된 실행계획](/images/posts/mysql-index-and-explain/legacy-06.png "ref가 const인 경우")

Join에서는 앞 테이블에서 읽은 컬럼을 다음 테이블의 인덱스와 비교할 수 있다. 이때 `ref`에 어떤 컬럼이 연결되는지를 확인하면 Join 조건이 실제 탐색에 사용됐는지 이해하기 쉽다.

![Join 조건의 컬럼이 ref에 표시된 실행계획](/images/posts/mysql-index-and-explain/legacy-07.png "Join에서 ref를 읽는 방법")

`rows`는 Optimizer가 이 단계에서 읽을 것으로 예상한 행 수다. 실제 측정값이 아니라 테이블 통계에 기반한 추정치다. `filtered`는 읽은 행 중 나머지 조건을 통과할 것으로 예상한 비율이다.

예를 들어 `rows = 100000`이고 `filtered = 10`이라면 약 10만 행을 읽은 뒤 그중 10% 정도가 다음 단계로 넘어갈 것으로 예상한다. 정확한 계산값이라기보다 어디에서 많은 행을 읽고 버리는지 찾는 단서다.

다음 화면처럼 `rows`와 `filtered`는 따로 보지 않고 “몇 행을 읽어서 그중 얼마나 남기는가”로 묶어서 읽는다.

![rows와 filtered를 함께 확인하는 실행계획](/images/posts/mysql-index-and-explain/legacy-08.png "rows와 filtered")

인덱스를 추가하거나 순서를 바꾼 뒤에는 이전 실행 계획과 나란히 비교해야 한다. 다음 화면에서는 변경 후 실제 `key`와 `type`, 예상 `rows`가 어떻게 달라졌는지 확인할 수 있다.

![인덱스 구성 이후 달라진 실행계획](/images/posts/mysql-index-and-explain/legacy-09.png "인덱스 적용 후 실행계획")

이제 EXPLAIN을 읽는 기본 순서를 다시 묶어보자.

```text
1. key에 실제 선택한 인덱스가 있는가?
2. type이 ALL인가, range·ref처럼 범위를 줄였는가?
3. rows가 최종 결과 건수에 비해 지나치게 크지 않은가?
4. filtered에서 읽은 행을 너무 많이 버리고 있지 않은가?
5. Extra에 별도 정렬이나 임시 결과가 있는가?
```

이 순서는 `key`에 인덱스 이름이 보이면 검증을 끝내는 실수를 막아준다.

## key에 인덱스 이름이 있으면 성공일까

```text
key에 인덱스 이름이 있다
≠ 쿼리가 충분히 효율적이다
```

최종 결과는 20건인데 `rows = 800000`이라면 인덱스를 사용하면서도 80만 개에 가까운 인덱스 항목을 읽고 있을 수 있다. 조건과 맞지 않는 넓은 범위를 읽은 뒤 대부분을 버린다면 인덱스를 사용했다는 사실만으로 충분하지 않다.

복합 인덱스도 이름 전체가 `key`에 표시됐다고 모든 컬럼이 탐색 범위를 줄이는 데 사용됐다고 볼 수 없다. `(ad_id, status, created_at)` 인덱스를 선택했지만 실제 사용한 Key 부분이 다음과 같을 수 있다.

```text
key            = idx_ad_id_status_created_at
used_key_parts = ad_id, status
```

`used_key_parts`는 복합 인덱스 중 실제 탐색 범위를 만드는 데 사용된 부분을 보여준다. 조회 기간이 매우 넓어 `created_at` 조건을 거의 모든 행이 만족한다면 Optimizer는 날짜까지 범위를 좁히는 것보다 `ad_id`, `status`로 읽은 뒤 조건을 판단하는 편이 저렴하다고 예상할 수 있다.

반복해서 검증해도 `created_at`이 탐색에 기여하지 않는다면 더 짧은 `(ad_id, status)` 인덱스도 후보가 된다. 다만 한 번의 실행 계획만 보고 바로 제거하지 말고 다른 쿼리의 정렬과 범위 조건까지 확인해야 한다.

`EXPLAIN`은 어디까지나 MySQL의 예상이다. 통계가 실제 데이터 분포를 완벽하게 반영하지 못하면 예상 행 수와 실제 행 수가 다를 수 있다. 이 차이를 확인할 때 `EXPLAIN ANALYZE`를 사용한다.

## EXPLAIN ANALYZE는 실제 실행 결과를 보여준다

두 명령의 차이는 예상과 실제로 정리할 수 있다.

```text
EXPLAIN
→ MySQL이 어떻게 실행할 것으로 예상하는지 보여줌

EXPLAIN ANALYZE
→ 쿼리를 실제로 실행하고 실제 행 수와 시간까지 보여줌
```

```sql
EXPLAIN ANALYZE
SELECT id, title
  FROM post
 WHERE status = 'PUBLISHED'
 ORDER BY created_at DESC
 LIMIT 20;
```

실행 계획의 각 단계가 실제로 몇 번 수행됐고, 몇 행을 반환했으며, 시간이 얼마나 걸렸는지 확인할 수 있다. MySQL은 이런 실행 단계를 Iterator 형태로 표현하지만 처음에는 Iterator 이름을 외우기보다 예상 행 수와 실제 행 수, 반복 횟수와 소요 시간의 차이를 보는 것이 중요하다.

`EXPLAIN ANALYZE`는 쿼리를 실제로 실행한다. 운영 환경의 무거운 조회나 변경을 포함하는 작업에는 부하와 영향을 고려해야 한다. 가능한 경우 운영과 비슷한 데이터 분포를 가진 안전한 환경에서 먼저 확인한다.

트리 형태의 실행 계획이 익숙하지 않다면 MySQL Workbench의 Visual Explain으로 Join 순서와 테이블 접근 경로를 먼저 살펴볼 수 있다. 하지만 화면에서 인덱스 접근이 보기 좋게 표시됐다는 이유만으로 검증이 끝난 것은 아니다. `key`, `used_key_parts`, 예상·실제 행 수와 실행 시간을 함께 읽어야 한다.

인덱스로 읽은 범위를 확인한 뒤에는 `Extra`에 표시되는 추가 작업도 해석해야 한다. `Using filesort`와 `Using temporary`는 이름만 보면 오류처럼 느껴지지만, 쿼리가 실패했다는 뜻은 아니다.

## Using filesort와 Using temporary가 보이면 실제 비용을 확인한다

EXPLAIN의 `Extra`에 이런 문구가 나타났다고 쿼리가 잘못된 것은 아니다. 추가 작업이 생겼다는 신호이므로 처리하는 행 수와 실행 빈도, 실제 시간을 함께 확인해야 한다.

### Using filesort

`Using filesort`는 인덱스 순서만으로 결과 정렬을 끝내지 못해 별도의 정렬 단계가 필요했다는 뜻이다. 이름에 `file`이 들어 있지만 항상 디스크 파일에 정렬한다는 의미는 아니다. 데이터 크기와 메모리 상황에 따라 메모리 또는 디스크를 사용할 수 있다.

결과 20건을 만들기 전에 수백만 행을 반복해서 정렬한다면 병목이 될 수 있다. 반대로 이미 충분히 좁혀진 수십 행을 정렬한다면 인덱스를 추가하는 비용보다 현재 정렬을 유지하는 편이 나을 수 있다.

### Using temporary

`Using temporary`는 `GROUP BY`, `DISTINCT`나 복잡한 정렬을 처리하기 위해 중간 결과를 별도로 만들었다는 뜻이다. `status` 값이 세 종류뿐인 작은 테이블을 그룹화한다면 비용이 작을 수 있다. 반대로 수백만 행을 Join한 뒤 큰 중간 결과를 만들고 다시 정렬한다면 비용이 커진다.

두 표시 모두 같은 질문으로 판단할 수 있다.

```text
얼마나 많은 행을 처리하는가?
얼마나 자주 실행되는가?
실제 실행 시간이 긴가?
메모리와 디스크 사용량이 큰가?
```

따라서 표시를 없애는 것 자체를 목표로 삼지 않는다. 쿼리 조건과 인덱스 정렬이 어긋난 이유를 찾고, 변경 전후의 실행 계획과 시간을 비교해야 한다.

## 인덱스가 있어도 기대대로 사용하지 못하는 이유

앞에서 B+Tree는 정렬된 값을 이용해 시작 위치와 읽을 범위를 빠르게 정한다고 했다. 쿼리의 조건이 이 정렬 값을 그대로 활용하지 못하면 인덱스가 있어도 넓은 범위를 읽거나 Full Scan을 선택할 수 있다.

### 인덱스 컬럼에 함수를 적용한 경우

```sql
SELECT *
  FROM post
 WHERE DATE(created_at) = '2026-08-07';
```

인덱스는 원래 `created_at` 값을 정렬해 둔다. 그런데 쿼리는 `DATE(created_at)`로 변환한 값을 기준으로 비교한다. 이 경우 기존 정렬을 그대로 이용해 시작 위치를 찾기 어려울 수 있다.

범위로 바꾸면 원래 값을 그대로 비교할 수 있다.

```sql
SELECT *
  FROM post
 WHERE created_at >= '2026-08-07 00:00:00'
   AND created_at <  '2026-08-08 00:00:00';
```

다음 실행 계획을 통해 함수를 적용하기 전과 후의 `type`, `key`, `rows`를 비교할 수 있다.

![인덱스 컬럼에 함수를 적용한 실행계획](/images/posts/mysql-index-and-explain/legacy-10.png "인덱스 컬럼에 함수를 적용한 경우")

### 비교하는 타입이 달라 암묵적 형변환이 발생한 경우

문자열 컬럼을 숫자 값과 비교하면 MySQL이 한쪽 타입을 내부적으로 변환할 수 있다.

```sql
SELECT *
  FROM member
 WHERE phone_number = 1012345678;
```

`phone_number`가 문자열이라면 문자열 리터럴로 비교하는 편이 의도가 명확하다.

```sql
SELECT *
  FROM member
 WHERE phone_number = '1012345678';
```

형변환 방향과 데이터 타입에 따라 정렬된 인덱스 값을 그대로 탐색하기 어려워질 수 있다. 다음 화면은 같은 값처럼 보여도 비교 타입에 따라 접근 경로가 달라질 수 있음을 보여준다.

![암묵적 형변환이 포함된 실행계획](/images/posts/mysql-index-and-explain/legacy-11.png "암묵적 형변환")

### 복합 인덱스의 선두 컬럼을 건너뛴 경우

`(status, created_at, id)`는 status부터 사전식으로 정렬되어 있다. `created_at`만 조건에 사용하면 DRAFT, PUBLISHED 같은 각 status 영역을 가로질러 날짜를 찾아야 하므로 하나의 연속된 시작 범위를 정하기 어렵다.

```sql
SELECT id, created_at
  FROM post
 WHERE created_at >= '2026-08-01';
```

이 쿼리가 중요하다면 실제 조회 패턴에 맞는 별도 인덱스가 필요한지 검토해야 한다. 기존 복합 인덱스에 컬럼이 포함되어 있다는 이유만으로 효율적인 탐색을 보장할 수 없다.

### 결과가 테이블의 많은 부분을 차지하는 경우

인덱스로 범위를 찾을 수 있어도 결과가 테이블 대부분이라면 Secondary Index와 Clustered Index를 반복해서 오가는 비용이 커질 수 있다. 이 경우 Optimizer는 순차적으로 테이블을 읽는 Full Scan이 더 저렴하다고 판단할 수 있다.

Full Scan이 선택됐다는 이유만으로 실패라고 단정하지 않는다. 최종 반환 건수와 실제 읽는 행 수, 테이블 크기와 실행 시간을 함께 확인한다.

### 앞쪽이 열려 있는 LIKE 검색

```sql
SELECT *
  FROM post
 WHERE title LIKE '%mysql';
```

앞에서 B+Tree는 정렬된 값으로 시작 위치를 찾는다고 했다. `mysql%`은 첫 글자가 `m`인 영역부터 시작할 수 있지만, `%mysql`은 앞부분이 무엇인지 알 수 없어 특정 시작 위치를 정하기 어렵다.

다음 첫 화면에서는 앞쪽 와일드카드가 있는 검색의 접근 방식을 확인하고, 두 번째 화면에서는 와일드카드 위치에 따라 `type`과 `rows`가 어떻게 달라지는지 비교하면 된다.

![앞쪽 와일드카드가 포함된 LIKE 검색](/images/posts/mysql-index-and-explain/legacy-13.png "앞쪽 와일드카드 LIKE")

![앞쪽 와일드카드 유무에 따른 실행계획 비교](/images/posts/mysql-index-and-explain/legacy-14.png "LIKE 검색의 실행계획 비교")

포함 검색이 핵심 요구사항이라면 일반 B+Tree 인덱스만으로 해결하려 하기보다 MySQL의 `FULLTEXT` 인덱스나 검색 엔진 같은 다른 방법도 검토해야 한다.

### 정렬 방향과 컬럼 순서가 맞지 않는 경우

인덱스가 `(status, created_at DESC, id DESC)` 순서인데 쿼리의 조건과 정렬이 다른 컬럼 순서로 섞여 있다면 인덱스 순서만으로 정렬을 끝내지 못할 수 있다. 이때 `Using filesort`가 나타날 수 있다.

정렬을 위해 새 인덱스를 추가하기 전에 조건으로 얼마나 범위를 좁히는지, 정렬할 행이 실제로 얼마나 많은지부터 확인한다.

### OR 조건에서는 여러 인덱스를 합쳐 사용할 수 있다

`OR`은 하나의 실행 계획에서 여러 조건을 처리한다. 각 조건이 서로 다른 인덱스와 잘 맞으면 MySQL이 여러 인덱스 결과를 합치는 `index_merge`를 선택할 수 있다.

다음 화면에서 `key`에 여러 인덱스가 표시되는지, 각 결과를 어떤 방식으로 합치는지 확인할 수 있다.

![OR 조건에서 index_merge가 선택된 실행계획](/images/posts/mysql-index-and-explain/legacy-12.png "OR 조건과 index_merge")

`index_merge`가 선택됐다는 사실만으로 빠르거나 느리다고 단정할 수 없다. 각 인덱스가 읽는 행 수와 결과를 합치는 비용을 함께 봐야 한다.

## OR과 UNION ALL은 실행 계획을 비교해 선택한다

`OR` 조건을 독립된 쿼리로 나누고 `UNION ALL`로 합치면 각 쿼리가 자신에게 맞는 인덱스를 선택할 가능성이 생긴다.

```text
OR
→ 하나의 실행 계획에서 여러 조건 처리
→ index_merge 같은 접근을 선택할 수 있음

UNION ALL
→ 조건을 쿼리별로 분리
→ 각 쿼리가 서로 다른 인덱스를 선택할 수 있음
→ 마지막에 결과를 합침
```

먼저 OR 실행 계획에서 각 조건의 결과를 병합하는 방식과 예상 행 수를 확인한다.

![OR 조건에서 결과를 병합하는 실행계획](/images/posts/mysql-index-and-explain/legacy-17.png "OR의 index_merge 실행계획")

다음으로 같은 조건을 `UNION ALL`로 나누었을 때 각 SELECT가 선택한 인덱스와 결과를 합치는 비용을 비교한다.

![UNION ALL로 나눈 쿼리의 실행계획](/images/posts/mysql-index-and-explain/legacy-18.png "UNION ALL 실행계획")

`OR`을 `UNION ALL`로 바꾸면 무조건 빨라진다는 규칙은 없다. `UNION ALL`은 중복을 제거하지 않으므로 두 조건을 모두 만족하는 행이 있으면 중복될 수 있다. 중복 허용 여부, 각 쿼리의 인덱스 효율, 결과를 합치는 비용과 실제 실행 시간을 함께 확인해야 한다.

## 느린 쿼리를 확인하는 순서

인덱스 튜닝은 정해진 규칙 하나를 적용하고 끝내는 작업이 아니다. 쿼리가 느린 이유를 예상하고 실행 계획과 측정 결과로 검증해야 한다.

```text
쿼리가 느리다
↓
WHERE·JOIN 조건, ORDER BY와 반환 컬럼 확인
↓
현재 인덱스의 컬럼과 순서 확인
↓
쿼리가 B+Tree에서 시작 위치와 범위를 정할 수 있는지 예상
↓
EXPLAIN 실행
↓
possible_keys / key / type / rows / filtered / Extra 확인
↓
필요하면 EXPLAIN ANALYZE로 실제 행 수와 시간 확인
↓
인덱스 또는 쿼리 변경
↓
변경 전후 실행 계획과 실제 시간 다시 비교
```

새 인덱스를 추가하기 전에는 기존 인덱스와 역할이 겹치지 않는지, 쓰기 비용과 저장 공간이 얼마나 늘어나는지도 확인해야 한다. 읽기 쿼리 하나만 빨라지고 전체 시스템의 쓰기와 유지 비용이 커진다면 좋은 설계라고 보기 어렵다.

## 정리

- 인덱스는 컬럼에 붙이는 단순한 속도 옵션이 아니라 정렬된 별도의 탐색 구조다.
- B+Tree는 한 Page에서 많은 범위를 제외하고 Leaf Page를 이어 읽어 단건과 범위 조회에 유리하다.
- InnoDB의 Clustered Index Leaf에는 실제 행이 있고 Secondary Index Leaf에는 Secondary Key와 Primary Key가 있다.
- Secondary Index에 필요한 값이 없으면 Primary Key로 Clustered Index를 다시 탐색할 수 있다.
- 복합 인덱스는 여러 컬럼이 각각 정렬된 것이 아니라 왼쪽부터 사전식으로 정렬된 하나의 인덱스다.
- 컬럼 순서는 선택도만으로 정하지 않고 `=` 조건, 범위 조건, 정렬과 실제 조회 패턴을 함께 봐야 한다.
- 쿼리에 필요한 값이 한 인덱스에 모두 있으면 실제 행을 다시 조회하지 않고 결과를 만들 수 있다.
- `key`에 인덱스 이름이 보여도 `type`, `rows`, `filtered`, `used_key_parts`와 `Extra`를 함께 확인해야 한다.
- `Using filesort`, `Using temporary`, Full Scan과 `index_merge`가 보이면 처리한 행 수와 실제 실행 시간을 확인한다.
- `EXPLAIN`은 예상 실행 계획이고 `EXPLAIN ANALYZE`는 실제 실행 결과를 보여준다.

**좋은 인덱스 설계는 인덱스를 많이 만드는 것이 아니다. 쿼리가 어떤 범위를 어떻게 탐색할지 이해하고, 실행 계획과 실제 수행 시간으로 동작을 검증하는 것이다.**

## 참고 자료

### 공식 자료

- [MySQL - Clustered and Secondary Indexes](https://dev.mysql.com/doc/refman/8.4/en/innodb-index-types.html)
- [MySQL - Column Indexes](https://dev.mysql.com/doc/refman/8.4/en/column-indexes.html)
- [MySQL - EXPLAIN Statement](https://dev.mysql.com/doc/refman/8.4/en/explain.html)

### 국내 기술 블로그

- [LINE Engineering - MySQL Workbench의 VISUAL EXPLAIN으로 인덱스 동작 확인하기](https://engineering.linecorp.com/ko/blog/mysql-workbench-visual-explain-index)
