+++
title = '캐시는 DB 부하를 어떻게 줄이고 Redis는 무엇을 맡는가'
slug = '6'
aliases = ['/posts/006/']
date = 2026-04-03T19:10:00+09:00
lastmod = 2026-08-10T11:56:59+09:00
draft = false
description = '반복 조회가 만드는 비용에서 출발해 Redis의 자료형과 Key, TTL, 원자성, Pipeline, 영속성과 Spring Cache까지 차례로 살펴봅니다.'
categories = ['데이터 접근 설계']
tags = ['Redis', '캐시', 'Spring Cache', 'TTL']
+++

상품 상세 조회 요청이 들어왔다고 생각해 보자. 애플리케이션은 데이터베이스에서 여러 테이블을 조회하고, 필요한 값을 계산한 뒤 DTO로 변환해 응답한다.

```text
상품 상세 요청
→ DB 조회
→ Join과 계산
→ DTO 변환
→ 응답
```

잠시 뒤 같은 상품을 조회하는 요청이 다시 들어왔다. 그사이 상품 정보가 바뀌지 않았는데도 앞선 작업을 처음부터 반복해야 할까?

조회 결과를 잠시 저장해 두면 다음 요청에는 이미 만들어진 결과를 재사용할 수 있다. 이처럼 자주 사용하는 데이터를 더 빠르게 꺼낼 수 있는 곳에 보관하는 방법이 캐시다.

```text
첫 요청  → DB 조회 → 결과 생성 → Cache 저장 → 응답
다음 요청 → Cache 조회 ───────────────→ 응답
```

캐시의 개념은 단순하지만 실제로 도입하려면 질문이 이어진다. 무엇을 저장할지, 얼마 동안 믿을지, 여러 요청이 동시에 접근해도 안전한지, 캐시 서버가 사라지면 어떻게 할지까지 정해야 한다. 이 글에서는 그 질문을 따라가며 Redis와 Spring Cache의 역할을 살펴본다.

## 같은 DB 작업을 반복하지 않는다

캐시가 없으면 요청마다 원본 저장소까지 가야 한다.

```text
요청
→ DB Connection 대여
→ SQL 실행
→ Join과 집계
→ 결과 전송
→ 객체 변환
→ 응답
```

캐시에 원하는 값이 있으면 흐름은 훨씬 짧아진다.

```text
요청
→ Redis 조회
→ 이미 만들어진 결과 반환
```

여기서 줄어드는 비용은 단순히 메모리가 디스크보다 빠르다는 데서 끝나지 않는다. SQL 실행, Lock 경합, Connection Pool 점유, 네트워크 전송과 결과 조립도 함께 줄어든다. 핵심은 명확하다. **캐시는 같은 DB 작업을 다시 하지 않도록 결과를 재사용하는 방법이다.**

그렇다고 캐시를 넣으면 모든 요청이 빨라지는 것은 아니다. 캐시에 원하는 값이 없으면 Redis를 확인한 뒤 다시 DB로 가야 한다.

```text
요청
→ Redis 조회
→ 값 없음
→ DB 조회
→ Redis 저장
→ 응답
```

캐시에 값이 있어 바로 사용하는 경우를 Cache Hit, 값이 없어 원본 저장소를 조회하는 경우를 Cache Miss라고 한다. 반복 조회가 거의 없다면 Cache Miss가 대부분이므로 Redis 왕복과 관리 비용만 추가될 수 있다. 따라서 도입하기 전에 다음 질문부터 확인해야 한다.

- 같은 요청이나 계산이 충분히 반복되는가?
- 원본 조회와 결과 생성 비용이 큰가?
- Cache Hit 비율이 충분히 높을 것으로 예상되는가?

### DB Buffer Pool이 있어도 별도 캐시를 두는 이유

데이터베이스에도 Buffer Pool과 같은 메모리 캐시가 있다. 이 캐시는 자주 읽는 데이터 Page와 Index를 메모리에 두어 디스크 접근을 줄인다. 하지만 애플리케이션이 원하는 최종 응답을 그대로 보관하는 것은 아니다.

예를 들어 Buffer Pool 덕분에 필요한 행을 빠르게 읽더라도 SQL의 Join과 집계, 결과 전송과 DTO 변환은 다시 수행할 수 있다. 반면 Redis에는 직렬화한 조회 결과나 계산 결과를 저장할 수 있다. 두 캐시는 경쟁 관계가 아니라 서로 다른 단계의 반복 비용을 줄인다.

이제 결과를 어디에 보관할지 정해야 한다. 가장 단순한 방법은 애플리케이션 메모리의 `Map`이다. 그런데 서버가 여러 대라면 이야기가 달라진다.

## Java Map 대신 Redis를 두는 이유

애플리케이션이 한 대이고 데이터가 작다면 `ConcurrentHashMap` 같은 Local Cache로도 충분할 수 있다. 네트워크를 거치지 않아 빠르고 구성도 단순하다.

하지만 애플리케이션 인스턴스가 여러 대면 각 프로세스는 서로 다른 메모리를 사용한다.

```text
Application 1 → Map Cache 1
Application 2 → Map Cache 2
```

첫 번째 서버가 캐시를 갱신해도 두 번째 서버의 값은 그대로 남을 수 있다. 모든 서버가 같은 캐시를 공유해야 한다면 별도 저장소를 둘 수 있다.

```text
Application 1 ─┐
               ├→ Redis
Application 2 ─┘
```

Redis는 Java 객체 안의 `Map`이 아니라 네트워크로 접근하는 별도의 서버다. 여러 인스턴스가 같은 값을 볼 수 있지만 그만큼 비용도 생긴다.

- 장점: 여러 애플리케이션 인스턴스가 같은 캐시를 공유
- 비용: 네트워크 왕복과 직렬화·역직렬화
- 위험: Redis 장애라는 새로운 실패 지점

따라서 Redis가 잠시 응답하지 않을 때 요청을 실패시킬지, DB 조회로 우회할지, 일부 기능을 제한할지도 정해야 한다. DB 우회가 항상 정답은 아니다. 많은 요청이 한꺼번에 DB로 몰리면 Redis 장애가 DB 장애로 번질 수 있기 때문이다.

다음 그림에서는 Redis가 애플리케이션 내부 객체가 아니라 Client의 요청을 받는 별도 서버라는 점을 보면 된다.

![메모리 기반 데이터 저장소 Redis](/images/posts/redis-cache-basics/legacy-01.png "Redis")

![Client 요청을 처리하는 Redis Server의 구조](/images/posts/redis-cache-basics/legacy-02.png "Redis Server의 요청 처리 흐름")

공유 저장소를 선택했다면 이제 Redis에 무엇을 어떤 모양으로 넣을지 정해야 한다.

## 필요한 연산에 맞춰 자료형을 고른다

Java에서 `List`로 표현한 데이터를 Redis에서도 List에 넣으면 될까? 반드시 그렇지는 않다. 선택 기준부터 분명히 해야 한다. **Redis에서 그 값으로 어떤 연산을 자주 할 것인가?** 애플리케이션 객체의 모양보다 이 질문이 먼저다.

| 필요한 작업 | 우선 검토할 자료형 | 핵심 연산 |
| --- | --- | --- |
| JSON이나 단일 값 저장 | String | `SET`, `GET` |
| 숫자를 자주 증가 | String | `INCR`, `INCRBY` |
| 객체의 일부 Field만 변경 | Hash | `HSET`, `HGET` |
| 입력 순서를 유지한 목록 | List | `LPUSH`, `RPUSH`, `LPOP` |
| 중복 없는 회원 관리 | Set | `SADD`, `SISMEMBER` |
| 점수 기준 순위와 범위 조회 | Sorted Set | `ZADD`, `ZRANGE` |

Set은 `SADD` 자체가 중복을 처리하므로 애플리케이션에서 먼저 중복을 검사할 필요가 없다. Sorted Set은 각 값에 Score를 두어 순위나 점수 범위를 조회할 수 있다. Hash는 전체 JSON을 다시 저장하지 않고 필요한 Field만 읽거나 바꾸기 좋다.

```text
SET plant:1:summary '{"status":"NORMAL"}' EX 600
HSET member:10 name "Kim" role "OPERATOR"
SADD post:1:viewers "member:10"
ZADD ranking 1200 "member:10"
```

다음 그림에서는 Redis가 문자열만 저장하는 단순 Key-Value 저장소가 아니라 여러 자료형과 연산을 제공한다는 점을 보면 된다.

![Redis가 제공하는 주요 자료형](/images/posts/redis-cache-basics/legacy-03.png "Redis 자료형")

자료형이 값의 연산 방식을 정한다면 Key는 그 값을 찾는 주소 역할을 한다.

## Key 이름이 데이터의 위치를 설명한다

Redis에는 관계형 데이터베이스의 테이블과 컬럼처럼 데이터를 분류해 주는 고정된 구조가 없다. 따라서 Key 이름 자체가 데이터의 소속과 용도를 설명해야 한다.

보통 `업무:대상:식별자:용도`처럼 구분자를 사용해 일정한 구조를 만든다.

```text
equipment:42:detail
member:10:session
post:100:viewers
```

`detail42`처럼 맥락이 없는 이름보다 어떤 업무의 어떤 데이터인지 바로 알 수 있다. Prefix 단위로 메모리 사용량을 조사하거나 만료 정책을 확인하기도 쉽고, 다른 서비스와 Key가 충돌할 가능성도 줄어든다.

### 운영에서 Key를 찾을 때 주의할 점

`KEYS equipment:*`는 패턴에 맞는 모든 Key를 한 번에 찾는다. 시간 복잡도는 전체 Key 수에 비례하며, 큰 Key 공간에서는 다른 요청을 오래 기다리게 할 수 있다. Redis 공식 문서도 일반 애플리케이션 코드에서 `KEYS`를 사용하지 말라고 경고한다.

운영 중 점진적으로 Key를 탐색하려면 `SCAN`을 검토한다. `SCAN`은 Cursor를 이용해 일부씩 반환하므로 한 번에 서버를 오래 점유하는 일을 줄인다. 다만 탐색 중 데이터가 바뀔 수 있고 중복된 Key가 반환될 수도 있으므로, 정확한 시점의 전체 목록을 한 번에 보장하는 명령으로 생각해서는 안 된다.

Key와 자료형을 정해도 한 가지 문제가 남는다. DB 값이 바뀌었는데 Redis에는 예전 값이 계속 남아 있으면 어떻게 해야 할까?

## TTL은 캐시가 살아 있을 시간을 정한다

캐시를 한 번 저장한 뒤 영원히 지우지 않으면 오래된 값이 계속 응답될 수 있다. TTL(Time To Live)은 Key가 살아 있을 시간을 정한다.

```text
저장
→ 일정 시간 동안 사용
→ TTL 종료
→ 만료
```

```text
SET equipment:42:detail "..." EX 300
TTL equipment:42:detail
```

TTL에는 세 가지 역할이 있다.

- 오래된 값이 무한히 남지 않게 한다.
- 더 이상 사용하지 않는 Key가 계속 쌓이는 일을 줄인다.
- 원본 변경 후 캐시 삭제가 누락되어도 불일치가 영구적으로 이어지는 것을 막는다.

TTL이 짧으면 최신 값에 가까워지지만 Cache Miss와 DB 조회가 늘어난다. TTL이 길면 Cache Hit가 늘지만 오래된 값을 볼 가능성도 커진다. 따라서 Redis 성능만 보고 정답처럼 고를 수 없다. 업무에서 얼마 동안 오래된 값을 허용할 수 있는지와 원본 조회 비용을 함께 봐야 한다.

원본을 변경할 때 해당 캐시를 바로 삭제하고, TTL은 삭제 누락에 대비한 안전망으로 두는 방식도 사용할 수 있다.

```text
DB 변경
→ 관련 Cache 삭제

삭제가 누락되더라도
→ TTL이 끝나면 다시 조회
```

### 만료와 Eviction은 원인이 다르다

다음 두 상황은 결과만 보면 Key가 사라진다는 점이 같지만 원인은 다르다.

```text
TTL 5분 설정
→ 5분 경과
→ Key 만료
```

```text
TTL이 아직 3분 남음
→ Redis가 maxmemory에 도달
→ 정책에 따라 Key 제거 또는 쓰기 거부
```

첫 번째가 만료(Expiration), 두 번째 상황에서 정책에 따라 Key를 제거하는 것이 Eviction이다. `allkeys-lru` 같은 정책은 Key를 제거하지만, `noeviction`은 기존 Key를 지우는 대신 새로운 데이터를 쓰는 명령에 오류를 반환한다.

예상보다 Cache Hit 비율이 낮다면 TTL만 볼 것이 아니라 메모리 사용량, `maxmemory-policy`, Eviction 횟수와 쓰기 거부도 함께 확인해야 한다.

### 인기 있는 Key가 동시에 만료되면

조회가 몰리는 Key 하나가 만료되는 순간 수많은 요청이 동시에 Cache Miss를 만날 수 있다. 모든 요청이 같은 데이터를 DB에서 읽고 다시 계산하면 캐시가 막아 주던 부하가 한꺼번에 원본 저장소로 향한다. 이를 Cache Stampede라고 부른다.

상황에 따라 한 요청만 값을 다시 만들게 조정하거나, TTL에 작은 무작위 차이를 두거나, 만료 전에 일부 요청이 값을 미리 갱신하도록 설계할 수 있다. TTL을 정하는 데서 끝내지 말고 다음 질문까지 확인해야 한다. **동시에 Miss가 발생했을 때 원본이 그 부하를 견딜 수 있는가?**

지금까지는 값을 저장하고 만료시키는 방법을 살펴봤다. 이제 여러 요청이 같은 값을 동시에 바꿀 때도 안전한지 확인할 차례다.

## Redis가 단일 스레드면 동시성 문제도 없을까?

흔히 “Redis는 단일 스레드”라고 하지만, 이는 주로 명령 실행 관점의 설명이다. 현대 Redis는 느린 디스크 작업과 일부 I/O 등에 별도의 스레드나 프로세스를 사용한다. 다만 일반적인 명령은 주 실행 흐름에서 차례로 처리되므로 한 명령의 중간에 다른 Client 명령이 끼어들지 않는다.

예를 들어 다음 명령 하나는 값을 읽고 증가한 뒤 다시 저장하는 과정을 Redis 내부에서 처리한다.

```text
INCR counter
```

다른 Client가 `INCR`의 절반만 보거나 중간 상태에 끼어들 수는 없다. 이처럼 하나의 연산이 중간 상태 없이 처리되는 성질을 원자성(Atomicity)이라고 한다.

하지만 단일 명령이 원자적이라고 여러 명령의 조합까지 원자적인 것은 아니다.

```text
GET stock
→ Java에서 계산
→ SET stock
```

Client A와 B가 같은 재고 10을 읽고 각각 1을 뺀 뒤 모두 9를 저장하면, 두 번의 차감 요청 중 하나가 사라진다. 목적에 맞는 단일 명령이 있다면 먼저 사용하고, 여러 명령이 필요하다면 `WATCH`와 Transaction 또는 Lua Script를 검토해야 한다.

Redis Transaction은 `MULTI`와 `EXEC` 사이에 모은 명령을 다른 Client의 명령이 끼어들지 않게 차례로 실행한다. `WATCH`한 Key가 `EXEC` 전에 변경되면 Transaction을 취소해 다시 시도할 수도 있다. 다만 관계형 데이터베이스의 Transaction처럼 실행 중 오류가 난 명령까지 모두 되돌리는 Rollback을 제공하는 것은 아니다.

여러 명령을 다룬다고 무조건 Transaction이나 Script가 필요한 것은 아니다. 먼저 느린 이유가 네트워크 왕복인지, 명령 사이의 간섭인지 구분해야 한다.

## Pipeline은 네트워크 왕복을 줄인다

명령을 하나씩 보내고 매번 응답을 기다리면 명령 자체가 빨라도 네트워크 대기 시간이 반복된다.

```text
명령 A 전송 → 응답 대기
명령 B 전송 → 응답 대기
명령 C 전송 → 응답 대기
```

요청을 보내고 응답을 받는 한 번의 네트워크 왕복 시간을 RTT(Round Trip Time)라고 한다. Pipeline은 여러 명령을 먼저 모아 보내고 응답도 묶어서 받아 RTT의 반복을 줄인다.

```text
A ┐
B ├→ 한 번에 전송
C ┘
  ↓
응답을 묶어서 수신
```

Pipeline은 전송 방식을 최적화할 뿐 명령 묶음의 원자성을 보장하지 않는다.

| 해결하려는 문제 | 검토할 방법 |
| --- | --- |
| 명령마다 네트워크 응답을 기다려 느림 | Pipeline |
| 명령 사이에 다른 Client 작업이 들어오면 안 됨 | 단일 명령, Transaction, Lua Script |

너무 큰 Pipeline은 서버가 응답을 모아 두는 메모리를 늘린다. 대량 명령은 한 번에 모두 보내기보다 적당한 Batch로 나눠야 한다.

## 여러 명령을 하나의 업무 단위로 묶어야 할 때

친구 목록 전체를 교체하기 위해 `DEL`, `RPUSH`, `EXPIRE`를 순서대로 실행한다고 생각해 보자. 같은 갱신 메시지가 중복 전달되어 두 Consumer가 동시에 실행하면 다음 순서가 가능하다.

```text
Consumer A: DEL friends:1
Consumer B: DEL friends:1

Consumer A: RPUSH friends:1 10 20 30
Consumer B: RPUSH friends:1 10 20 30

결과: 10 20 30 10 20 30
```

각 Redis 명령은 정상적으로 실행되었다. 그러나 여러 명령을 합친 “친구 목록 교체”라는 업무 단위는 안전하지 않았다.

세 명령을 Pipeline으로 보내도 이 문제는 해결되지 않는다. Pipeline의 목적은 왕복 감소이지 명령 조합의 원자성 보장이 아니기 때문이다. 세 동작 사이에 다른 요청이 끼어들면 안 된다면 하나의 Lua Script로 묶을 수 있다.

```text
DEL
RPUSH
EXPIRE
→ 하나의 Script로 실행
```

Redis는 Script가 실행되는 동안 다른 Client의 명령을 처리하지 않아 Script 전체의 원자성을 보장한다. 그만큼 오래 걸리는 Script는 다른 요청을 막으므로 짧게 유지해야 한다.

### Redis Cluster에서는 추가로 확인할 점

Script가 사용할 Key는 `KEYS` 인자로 명시해야 한다. Redis Cluster에서 여러 Key를 하나의 명령, Transaction 또는 Script로 다루려면 모두 같은 Hash Slot에 있어야 한다. 관련 Key에 같은 Hash Tag를 넣어 Slot을 맞출 수 있다.

```text
member:{10}:profile
member:{10}:session
```

단일 Key의 친구 목록을 교체하는 앞선 예시는 이 제약을 맞추기 쉽다. 여러 Key를 함께 다룰 때는 애플리케이션 로직뿐 아니라 Cluster의 데이터 배치까지 고려해야 한다.

Redis 안에서 명령을 안전하게 실행하는 방법을 정했다면 다음 질문이 남는다. Redis 서버가 재시작되거나 데이터를 잃었을 때 무엇을 복구해야 할까?

## Redis 데이터가 사라져도 다시 만들 수 있는가?

영속성 방식을 고르기 전에 Redis가 맡은 데이터의 역할부터 확인해야 한다.

```text
Redis 데이터가 모두 사라짐
↓
DB에서 다시 만들 수 있음
→ Cache에 가까움

DB에서도 다시 만들 수 없음
→ Redis가 원본 데이터의 일부를 맡고 있음
```

순수 캐시라면 장애 후 DB에서 다시 채울 수 있다. 이때는 복구에 걸리는 시간과 그동안 DB에 몰릴 부하를 감당할 수 있는지가 중요하다. 반대로 Redis의 데이터가 유일한 원본이라면 캐시와 같은 기준으로 운영해서는 안 된다.

Redis가 없어졌을 때의 정책도 함께 정해야 한다. 요청을 실패시킬지, 제한적으로 DB를 조회할지, 캐시가 복구될 때까지 기능을 줄일지 선택해야 한다. 데이터 재생성 가능 여부와 장애 중 트래픽을 함께 봐야 한다.

## RDB와 AOF는 복구 방식이 다르다

Redis는 데이터를 메모리에만 둘 수도 있고 디스크에 남겨 재시작 후 복구할 수도 있다. 이를 영속성(Persistence)이라고 한다.

RDB는 특정 시점의 전체 상태를 Snapshot 파일로 저장한다.

```text
특정 시점의 Redis 상태
→ RDB Snapshot
```

AOF(Append Only File)는 Redis가 받은 쓰기 명령을 기록하고, 재시작할 때 다시 실행해 상태를 복원한다.

```text
SET, HSET, INCR 같은 쓰기 명령 기록
→ 재시작 때 재생
→ 상태 복원
```

선택지는 네 가지다.

- RDB: 일정 시점의 Snapshot을 저장
- AOF: 쓰기 명령을 이어서 기록
- RDB + AOF: 두 방식을 함께 사용
- 영속성 없음: 원본에서 다시 만들 수 있는 Cache로 사용

RDB는 파일이 비교적 작고 큰 Dataset의 재시작이 빠르지만, 마지막 Snapshot 이후의 변경을 잃을 수 있다. AOF는 `fsync` 정책에 따라 변경을 더 촘촘하게 남길 수 있지만 파일과 쓰기, Rewrite 비용이 생긴다. 어느 방식이 항상 우월한 것은 아니다.

```text
허용 가능한 데이터 손실 범위
복구 시간
디스크 사용량
쓰기 비용
```

이 기준으로 선택해야 한다. Redis 데이터를 DB에서 다시 만들 수 있다면 영속성을 사용하지 않는 구성도 가능하다. 다시 만들 수 없다면 RDB와 AOF를 고르는 데서 끝내지 말고 복제, Backup과 복구 절차까지 원본 저장소 수준으로 설계해야 한다.

모든 데이터가 같은 수명을 가지는 것도 아니다. 몇 초 동안 빠르게 변하다가 최종 결과만 오래 남기면 되는 값이라면 Redis와 MySQL의 책임을 나눌 수 있다.

## 빠르게 변하는 상태와 장기 보관할 결과를 나눈다

실시간 방송 중 시청자가 반응 버튼을 연속해서 누른다고 생각해 보자. 요청마다 MySQL의 같은 행을 갱신하면 짧은 시간에 쓰기가 집중된다. 방송 중에는 Redis의 Counter를 사용하고 종료 후 최종 값만 MySQL에 저장할 수 있다.

```text
방송 중
반응 요청 대량 발생
→ Redis INCRBY

방송 종료
최종 반응 수 확인
→ MySQL 저장
→ Redis Key 삭제
```

Redis는 처리량이 중요한 일시적인 상태를, MySQL은 장기 보관할 결과를 맡는다. 하지만 “실시간 데이터는 Redis, 원본은 MySQL”이라는 공식이 항상 성립하는 것은 아니다. 데이터 수명, 변경 빈도, 유실 허용 여부, 조회 방식과 복구 방법을 보고 책임을 나눠야 한다.

Redis에서 MySQL로 옮기는 과정도 하나의 실패 경계다. MySQL 저장은 성공했지만 Redis Key 삭제 전에 장애가 날 수 있고, 재시도하면서 같은 값을 두 번 반영할 수도 있다. 어느 작업까지 끝나야 완료로 볼지, 재시작할 때 무엇을 기준으로 이어갈지 정해야 한다.

### 실시간 처리와 분석 기록의 요구는 다르다

API 요청을 처리할 때 필요한 것은 “지금 반응 수가 몇 개인가”일 수 있다. 반면 분석에서는 “누가, 언제, 어떤 반응을 했는가”가 필요하다.

```text
실시간 처리
→ Redis Counter

분석
→ 별도 Event 또는 Log 저장
```

두 요구를 분리하면 실시간 경로는 단순한 Counter의 처리량을 유지하고, 분석에는 필요한 상세 기록을 남길 수 있다.

지금까지는 Redis 명령과 저장 구조를 직접 다뤘다. 일반적인 조회 캐시를 만들 때마다 Service에서 `GET`, `SET`, `DEL`을 직접 작성해야 할까? Spring Cache는 이 반복을 줄이는 추상화를 제공한다.

## Spring Cache는 캐시 경계를 선언한다

다음 메서드에 `@Cacheable`을 붙이면 메서드 호출 앞에서 캐시를 확인한다.

```java
@Cacheable(cacheNames = "equipment", key = "#equipmentId")
public EquipmentResponse getEquipment(long equipmentId) {
    return equipmentRepository.findResponse(equipmentId);
}
```

실행 흐름은 다음과 같다.

```text
메서드 호출
→ Cache 확인
   ├─ 값 있음 → 메서드를 실행하지 않고 저장된 값 반환
   └─ 값 없음 → 메서드 실행 → 반환값 저장 → 반환
```

Spring Cache는 이런 동작을 Annotation으로 선언할 수 있게 해주는 공통 API다. `@Cacheable` 자체가 Redis 기능은 아니다.

```text
Spring Cache
→ 공통 Annotation과 Cache API

CacheManager와 Cache Provider
→ 실제 구현 선택
→ Redis, Caffeine 등
```

`CacheManager` 설정에 따라 같은 Annotation을 Redis나 Local Cache 구현과 연결할 수 있다. 다만 Annotation만 붙였다고 TTL, 직렬화 방식, Key Prefix와 장애 정책까지 자동으로 정해지는 것은 아니다.

Annotation 기반 캐시를 사용하려면 Spring 설정에서 `@EnableCaching`도 활성화해야 한다. 이 설정이 빠지면 `@Cacheable`과 `@CacheEvict`를 붙여도 캐시 동작이 시작되지 않는다.

## 원본이 바뀌면 캐시도 무효화해야 한다

조회 결과를 캐시한 뒤 DB 값만 수정하면 두 저장소의 값이 달라진다.

```text
DB     → 새 값
Cache  → 예전 값
```

`@CacheEvict`는 데이터 변경 메서드가 성공한 뒤 관련 Cache Entry를 제거하는 데 사용할 수 있다.

```java
@CacheEvict(cacheNames = "equipment", key = "#equipmentId")
public void updateEquipment(long equipmentId, UpdateEquipmentCommand command) {
    equipmentRepository.update(equipmentId, command);
}
```

같은 원본을 바꾸는 경로가 여러 개라면 모든 경로에서 같은 Cache를 무효화해야 한다. TTL은 그중 하나가 누락되었을 때 불일치가 영구적으로 이어지지 않게 하는 안전망이 될 수 있다.

### Spring Cache를 사용할 때 하나 더: Proxy 호출

Spring Cache의 기본 동작은 Spring AOP Proxy를 통한다.

```text
외부 Bean 호출
→ Cache Proxy
→ 실제 메서드
```

반면 같은 객체 안에서 `this.getEquipment()`처럼 자기 메서드를 호출하면 Proxy를 다시 거치지 않는다. 이 경우 대상 메서드에 `@Cacheable`이 있어도 캐시 기능이 적용되지 않을 수 있다.

```text
같은 객체 내부 호출
→ this.getEquipment()
→ Cache Proxy를 거치지 않음
```

따라서 캐시 적용 메서드는 외부 Bean을 통해 호출되는 `public` 메서드에 두고, Self Invocation이 생기지 않도록 책임을 나누는 편이 안전하다.

## 정리

캐시는 메모리가 빠르다는 이유만으로 도입하는 기능이 아니다. 반복되는 SQL과 Join, 계산, 객체 변환을 다시 수행하지 않고 결과를 재사용해 DB와 애플리케이션의 반복 작업을 줄인다.

실제 도입 여부는 다음 질문을 따라 판단할 수 있다.

```text
같은 조회가 충분히 반복되는가?
→ 원본 조회 비용이 큰가?
→ Cache Hit 비율을 확보할 수 있는가?
→ Cache 검토

여러 애플리케이션 인스턴스가 같은 Cache를 공유해야 하는가?
→ Redis 같은 외부 Cache 검토

Redis에서 어떤 연산이 필요한가?
→ 자료형과 Key 설계

얼마 동안 오래된 값을 허용할 수 있는가?
→ TTL과 무효화 정책 결정

여러 Redis 명령을 사용해야 하는가?
→ 네트워크 왕복이 문제면 Pipeline
→ 명령 사이 간섭이 문제면 단일 명령, Transaction, Lua Script 검토

Redis 데이터가 사라져도 다시 만들 수 있는가?
→ 만들 수 있으면 Cache에 맞는 영속성 수준 결정
→ 만들 수 없으면 원본 저장소의 역할까지 맡는지 다시 설계
```

Redis를 도입하는 목적은 DB보다 빠른 저장소 하나를 추가하는 것이 아니다. 반복되는 작업을 어디까지 재사용할지, 값의 수명과 정합성을 어떻게 관리할지, Redis가 장애 나도 복구할 수 있는 데이터인지까지 포함해 저장소의 책임을 설계하는 것이다.

## 참고 자료

### 공식 자료

- [Redis - Data Types](https://redis.io/docs/latest/develop/data-types/)
- [Redis - KEYS](https://redis.io/docs/latest/commands/keys/)
- [Redis - SCAN](https://redis.io/docs/latest/commands/scan/)
- [Redis - Key Eviction](https://redis.io/docs/latest/develop/reference/eviction/)
- [Redis - Transactions](https://redis.io/docs/latest/develop/using-commands/transactions/)
- [Redis - Diagnosing Latency Issues](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/)
- [Redis - Pipelining](https://redis.io/docs/latest/develop/using-commands/pipelining/)
- [Redis - Scripting with Lua](https://redis.io/docs/latest/develop/programmability/eval-intro/)
- [Redis - Scale with Redis Cluster](https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/)
- [Redis - Persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
- [Spring Framework - Cache Abstraction](https://docs.spring.io/spring-framework/reference/integration/cache.html)
- [Spring Framework - Annotation-based Caching](https://docs.spring.io/spring-framework/reference/integration/cache/annotations.html)

### 국내 기술 블로그

- [LINE Engineering - Atomic 처리와 Cache Stampede 대책을 위해 Redis Lua Script를 활용한 이야기](https://engineering.linecorp.com/ko/blog/atomic-cache-stampede-redis-lua-script/)
- [LINE Engineering - 안정적인 Love를 제공하는 방법](https://engineering.linecorp.com/ko/blog/how-to-provide-stable-loves/)
