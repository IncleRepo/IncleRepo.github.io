+++
title = '캐시는 DB 부하를 어떻게 줄이고 Redis는 무엇을 맡는가'
slug = '6'
aliases = ['/posts/006/']
date = 2026-04-03T21:10:00+09:00
lastmod = 2026-08-11T20:31:31+09:00
draft = false
description = '반복 조회가 만드는 비용부터 Redis에 데이터를 저장하고, 안전하게 변경하며, 장애 뒤 복구하는 방법까지 단계별로 살펴봅니다.'
categories = ['데이터 접근 설계']
tags = ['Redis', '캐시', 'Spring Cache', 'TTL']
+++

상품 상세 조회 요청이 들어왔다고 생각해 보자.

애플리케이션은 데이터베이스에서 상품과 판매자 정보를 조회한다. 여러 테이블의 결과를 합치고 필요한 값을 계산한 뒤, API 응답에 맞는 DTO로 변환한다.

```text
상품 상세 요청
→ DB 조회
→ 여러 테이블의 결과를 합침
→ 필요한 값 계산
→ DTO 변환
→ 응답
```

잠시 뒤 같은 상품을 조회하는 요청이 다시 들어왔다. 그사이 상품 정보는 바뀌지 않았다. 그런데도 앞에서 했던 작업을 처음부터 다시 해야 할까?

조회 결과를 잠시 저장해 두면 다음 요청에는 이미 만들어진 결과를 재사용할 수 있다.

```text
첫 요청
→ DB 조회
→ 결과 생성
→ Cache 저장
→ 응답

다음 요청
→ Cache 조회
→ 저장된 결과 반환
```

이처럼 자주 사용하는 데이터를 더 빠르게 꺼낼 수 있는 곳에 보관하는 방법이 캐시다.

여기까지는 단순하다. 하지만 실제 서비스에 캐시를 넣으려면 질문이 계속 생긴다.

```text
캐시는 어떤 비용을 줄일까?
→ 애플리케이션 메모리의 Map을 쓰면 안 될까?
→ Redis에는 무엇을 어떤 모양으로 저장해야 할까?
→ 저장한 값은 언제까지 믿을 수 있을까?
→ 여러 요청이 동시에 값을 바꿔도 안전할까?
→ Redis가 재시작되면 데이터는 어떻게 될까?
→ Spring 코드에서는 어떻게 사용할까?
```

이 글에서는 이 질문을 앞에서부터 하나씩 따라간다.

## 1. 같은 조회를 다시 하지 않는 방법

### 캐시가 줄이는 비용

캐시가 없으면 요청마다 원본 저장소인 DB까지 가야 한다.

```text
요청
→ DB Connection 대여
→ SQL 실행
→ Join과 집계
→ 조회 결과 전송
→ Java 객체와 DTO로 변환
→ 응답
```

여기에는 생각보다 여러 비용이 들어간다.

- Connection Pool에서 DB와 통신할 연결인 DB Connection을 빌린다.
- DB가 SQL을 해석하고 데이터를 찾는다.
- 여러 테이블을 Join하거나 값을 집계한다.
- DB가 조회 결과를 애플리케이션으로 전송한다.
- 애플리케이션이 결과를 Java 객체와 DTO로 바꾼다.

캐시에 이미 완성된 결과가 있다면 이 과정을 대부분 건너뛸 수 있다.

```text
요청
→ Redis 조회
→ 이미 만들어진 결과 반환
```

따라서 캐시의 장점은 단순히 “메모리가 디스크보다 빠르다”로 끝나지 않는다. 같은 SQL과 Join을 반복하지 않고, DB Connection을 점유하는 시간과 결과를 조립하는 작업도 줄인다.

**캐시는 DB 자체를 빠르게 만드는 기술이라기보다 같은 DB 작업을 다시 하지 않도록 결과를 재사용하는 방법이다.**

### 값이 있을 때와 없을 때

캐시를 조회하면 두 가지 상황이 생긴다.

먼저 원하는 값이 이미 있는 경우다.

```text
Redis에 값이 있음
→ 저장된 값 반환
```

이처럼 캐시에 값이 있어 바로 사용하는 경우를 Cache Hit라고 한다.

반대로 원하는 값이 없을 수도 있다.

```text
Redis에 값이 없음
→ DB 조회
→ 조회 결과를 Redis에 저장
→ 응답
```

이처럼 캐시에 값이 없어 원본 저장소를 조회하는 경우를 Cache Miss라고 한다.

Cache Miss가 발생하면 Redis와 DB를 모두 방문한다. 반복 조회가 거의 없다면 Redis 왕복만 하나 더 추가될 수 있다. 캐시를 넣었다고 무조건 빨라지는 것은 아니다.

캐시를 검토할 때는 다음 세 가지를 먼저 확인해야 한다.

- 같은 조회나 계산이 충분히 반복되는가?
- 원본 조회와 결과 생성 비용이 큰가?
- Cache Hit 비율을 충분히 확보할 수 있는가?

### DB Buffer Pool과 Redis는 무엇이 다를까?

여기서 한 가지 의문이 생긴다.

> DB도 자주 사용하는 데이터를 메모리에 올려 두는데 Redis가 또 필요한가?

MySQL의 InnoDB는 Buffer Pool에 자주 읽는 데이터 Page와 Index를 보관한다. 덕분에 같은 데이터를 다시 읽을 때 디스크 접근을 줄일 수 있다.

하지만 Buffer Pool이 애플리케이션의 최종 응답을 보관하는 것은 아니다. 필요한 행이 메모리에 있더라도 SQL의 Join과 집계, 결과 전송과 DTO 변환은 다시 수행할 수 있다.

Redis에는 이 과정을 거쳐 완성한 조회 결과나 계산 결과를 저장할 수 있다.

```text
DB Buffer Pool
→ DB가 Page와 Index를 더 빠르게 읽도록 도움

Redis Cache
→ 애플리케이션이 사용할 완성된 결과를 재사용
```

두 캐시는 서로 경쟁하는 기능이 아니다. 서로 다른 단계에서 반복 작업을 줄인다.

이제 결과를 어디에 저장할지 정해야 한다. 가장 단순한 생각은 애플리케이션 메모리에 `Map`을 하나 만드는 것이다.

## 2. 여러 서버가 함께 사용하는 캐시

### Java Map에 저장하면 안 될까?

애플리케이션이 한 대이고 캐시할 데이터가 많지 않다면 `ConcurrentHashMap` 같은 Local Cache를 사용할 수 있다. 네트워크를 거치지 않아 빠르고 구조도 단순하다.

문제는 애플리케이션 서버가 여러 대일 때 생긴다.

```text
Application 1 → Map Cache 1
Application 2 → Map Cache 2
```

두 Map은 서로 다른 Java 애플리케이션의 메모리에 있다. 첫 번째 서버가 값을 갱신해도 두 번째 서버의 값은 그대로 남는다. 사용자가 어느 서버로 요청을 보냈는지에 따라 서로 다른 결과를 받을 수 있다.

여러 서버가 같은 캐시를 봐야 한다면 캐시를 애플리케이션 밖으로 꺼낼 수 있다.

```text
Application 1 ─┐
               ├→ Redis
Application 2 ─┘
```

Redis는 Java 객체 안의 `Map`이 아니다. 애플리케이션이 네트워크로 명령을 보내고 응답을 받는 별도의 서버다.

다음 그림에서는 Redis가 애플리케이션 내부 객체가 아니라 독립적으로 실행되는 저장소라는 점만 보면 된다.

![메모리 기반 데이터 저장소 Redis](/images/posts/redis-cache-basics/legacy-01.png "Redis")

다음 그림에서도 Client가 요청을 보내면 Redis Server가 명령을 처리하고 결과를 돌려주는 흐름에 집중하면 된다.

![Client 요청을 처리하는 Redis Server의 구조](/images/posts/redis-cache-basics/legacy-02.png "Redis Server의 요청 처리 흐름")

### 공유 캐시에도 비용이 생긴다

Redis를 사용하면 여러 애플리케이션 인스턴스가 같은 캐시를 공유할 수 있다. 하지만 Local Cache에는 없던 비용도 함께 생긴다.

```text
장점
→ 여러 서버가 같은 캐시를 공유

비용
→ 네트워크 왕복
→ Java 객체를 전송 가능한 형태로 바꾸는 작업
→ Redis 장애라는 새로운 실패 지점
```

Redis가 잠시 응답하지 않을 때 어떻게 처리할지도 정해야 한다.

- 요청을 실패시킬 것인가?
- DB를 직접 조회하도록 우회할 것인가?
- 일부 기능을 잠시 제한할 것인가?

DB로 우회하는 방법도 항상 안전하지는 않다. Redis가 처리하던 요청이 한꺼번에 DB로 몰리면 Redis 장애가 DB 장애로 이어질 수 있다.

즉 Local Cache와 Redis 중 하나가 언제나 더 좋은 것은 아니다. 여러 서버가 값을 공유해야 하는지와 새로운 네트워크·장애 비용을 함께 보고 선택한다.

Redis를 사용하기로 했다면 다음 질문은 자연스럽다. Redis에는 값을 어떤 모양으로 저장해야 할까?

## 3. Redis에 데이터를 저장하는 방법

### Java 객체의 모양보다 필요한 연산을 먼저 본다

Java에서 `List`로 사용하는 데이터는 Redis에서도 List에 넣어야 할까?

반드시 그렇지는 않다. Redis 자료형은 Java 객체와 이름이 비슷하지만, 선택 기준은 객체의 모양보다 Redis에서 필요한 연산이다.

예를 들어 중복 없는 회원 목록이 필요하다고 생각해 보자.

Java 코드가 `List<Member>`를 사용하더라도 Redis에서는 Set이 더 자연스러울 수 있다. Set의 `SADD` 명령이 중복을 처리해 주기 때문이다.

필요한 작업을 기준으로 보면 자료형을 선택하기 쉽다.

```text
JSON 형태의 조회 결과 하나를 저장
→ String

숫자를 자주 증가
→ String과 INCR

객체의 일부 Field만 변경
→ Hash

입력 순서를 유지한 목록
→ List

중복 없는 값 관리
→ Set

점수를 기준으로 순위 조회
→ Sorted Set
```

| 필요한 작업 | 자료형 | 대표 명령 |
| --- | --- | --- |
| 단일 값이나 JSON 저장 | String | `SET`, `GET` |
| 숫자 증가 | String | `INCR`, `INCRBY` |
| Field 단위 조회와 변경 | Hash | `HSET`, `HGET` |
| 순서가 있는 값 추가와 제거 | List | `LPUSH`, `RPUSH`, `LPOP` |
| 중복 없는 값 관리 | Set | `SADD`, `SISMEMBER` |
| 점수 기준 정렬과 범위 조회 | Sorted Set | `ZADD`, `ZRANGE` |

Set은 값을 추가할 때 중복 여부를 함께 처리한다. Sorted Set은 각 값에 Score를 두기 때문에 순위나 특정 점수 범위를 조회하기 좋다. Hash는 전체 JSON을 다시 저장하지 않고 필요한 Field만 읽거나 바꿀 수 있다.

```text
SET plant:1:summary '{"status":"NORMAL"}' EX 600
HSET member:10 name "Kim" role "OPERATOR"
SADD post:1:viewers "member:10"
ZADD ranking 1200 "member:10"
```

다음 그림에서는 Redis가 단순히 문자열만 저장하지 않고 여러 자료형과 연산을 제공한다는 점을 보면 된다.

![Redis가 제공하는 주요 자료형](/images/posts/redis-cache-basics/legacy-03.png "Redis 자료형")

### Key 이름은 데이터의 주소다

자료형을 정했다면 값을 찾을 Key가 필요하다.

관계형 데이터베이스에는 테이블과 컬럼이 있다. 반면 Redis에는 데이터를 분류해 주는 고정된 구조가 없다. 따라서 Key 이름 자체가 데이터의 소속과 용도를 설명해야 한다.

보통 콜론으로 영역을 나눠 일정한 규칙을 사용한다.

```text
업무:대상:식별자:용도
```

```text
equipment:42:detail
member:10:session
post:100:viewers
```

`detail42`처럼 맥락이 없는 이름보다 어떤 데이터인지 바로 알 수 있다. 같은 시작 문자열을 가진 Key끼리 찾거나, 업무별 메모리 사용량과 만료 정책을 확인하기도 쉽다.

### 운영에서 Key를 찾을 때 주의할 점

개발 환경에서는 다음 명령으로 설비 관련 Key를 쉽게 찾을 수 있다.

```text
KEYS equipment:*
```

하지만 `KEYS`는 Redis 안의 전체 Key를 확인한 뒤 패턴에 맞는 값을 찾는다. Key가 많으면 그동안 다른 요청을 오래 기다리게 할 수 있다. Redis 공식 문서도 일반적인 운영 코드에서 `KEYS`를 사용하지 말라고 안내한다.

운영 중 Key를 조금씩 확인하려면 `SCAN`을 검토한다.

```text
SCAN 0 MATCH equipment:* COUNT 100
```

`SCAN`은 Cursor를 사용해 Key를 일부씩 반환한다. 한 번에 서버를 오래 점유하는 문제는 줄지만, 조회 중 데이터가 변경되면 같은 Key가 중복으로 나올 수도 있다. 따라서 결과를 처리하는 코드도 중복에 안전해야 한다.

이제 Redis에 무엇을 어떻게 저장할지는 정했다. 다음 문제는 저장한 값을 언제까지 믿을 수 있느냐다.

## 4. 캐시 값을 언제까지 믿을 것인가

### TTL이 필요한 이유

DB의 상품 이름이 바뀌었는데 Redis에는 예전 이름이 남아 있다고 생각해 보자.

```text
DB
→ 새 상품 이름

Redis
→ 예전 상품 이름
```

Redis의 값을 지우지 않으면 사용자는 계속 예전 이름을 볼 수 있다. 이 문제를 줄이기 위해 캐시 값이 살아 있을 시간을 정한다. 이를 TTL(Time To Live)이라고 한다.

```text
저장
→ 정해진 시간 동안 사용
→ TTL 종료
→ 만료
```

```text
SET equipment:42:detail "..." EX 300
TTL equipment:42:detail
```

`EX 300`은 이 Key를 300초 동안 유지한다는 뜻이다. `TTL` 명령으로 남은 시간을 확인할 수 있다.

TTL은 다음 문제를 줄인다.

- 오래된 값이 무한히 남는 문제
- 더 이상 사용하지 않는 Key가 계속 쌓이는 문제
- 원본 변경 후 캐시 삭제가 누락되어 불일치가 영구적으로 이어지는 문제

### TTL은 짧을수록 좋은가?

TTL이 짧으면 오래된 값을 볼 가능성이 줄어든다. 하지만 값이 자주 만료되므로 Cache Miss와 DB 조회는 늘어난다.

```text
TTL이 짧음
→ 최신 값에 가까움
→ Cache Miss 증가
→ DB 조회 증가
```

TTL이 길면 Cache Hit는 늘지만 예전 값을 더 오래 볼 수 있다.

```text
TTL이 김
→ Cache Hit 증가
→ DB 부하 감소
→ 오래된 값을 볼 가능성 증가
```

따라서 “TTL은 5분이 좋다”처럼 모든 서비스에 적용할 정답은 없다.

```text
업무에서 오래된 값을 얼마 동안 허용할 수 있는가?
원본을 다시 조회하는 비용은 얼마나 큰가?
```

두 질문을 함께 보고 정한다.

원본이 변경될 때 캐시를 바로 삭제하고 TTL도 함께 사용하는 방법이 흔하다.

```text
원본 변경
→ 관련 Cache 삭제

삭제가 누락되더라도
→ TTL이 끝나면 다시 조회
```

원본 변경 시 삭제는 오래된 값을 빠르게 없애고, TTL은 삭제 누락이 영구적인 불일치가 되지 않도록 돕는다.

### 만료와 Eviction은 다르다

다음 두 상황을 비교해 보자.

```text
TTL을 5분으로 설정
→ 5분이 지남
→ Key 만료
```

```text
TTL이 아직 3분 남음
→ Redis가 사용할 수 있는 메모리를 모두 사용
→ 설정된 정책에 따라 Key 제거 또는 쓰기 거부
```

첫 번째처럼 TTL이 끝나 Key가 사라지는 일을 만료(Expiration)라고 한다.

두 번째처럼 Redis가 `maxmemory`에 도달했을 때 정책에 따라 Key를 제거하는 일을 Eviction이라고 한다. 다만 `noeviction` 정책은 Key를 제거하지 않고 새로운 데이터를 쓰는 명령에 오류를 반환한다.

예상보다 Cache Hit 비율이 낮다면 TTL만 확인해서는 안 된다. 메모리가 부족해 Key가 계속 Eviction되고 있는지도 함께 봐야 한다.

```text
확인할 항목
→ 메모리 사용량
→ maxmemory-policy
→ Eviction 횟수
→ 메모리 부족으로 거부된 쓰기 명령
```

저장 방식과 수명까지 정했다. 이제 여러 요청이 동시에 같은 값을 다룰 때 어떤 일이 생기는지 살펴보자.

## 5. 여러 요청을 빠르고 안전하게 처리하는 방법

### Redis가 단일 스레드면 동시성 문제도 없을까?

흔히 “Redis는 단일 스레드”라고 말한다. 이 표현은 Redis 프로세스 전체가 오직 Thread 하나만 사용한다는 뜻이 아니다.

현대 Redis는 느린 디스크 작업과 일부 I/O 같은 주변 작업에 별도의 Thread나 Process를 사용할 수 있다. 다만 일반적인 명령 실행은 주 실행 흐름에서 하나씩 차례로 처리한다.

예를 들어 다음 명령을 보자.

```text
INCR counter
```

`INCR`는 현재 값을 읽고 1을 더한 뒤 다시 저장하는 과정을 하나의 Redis 명령으로 처리한다. 이 명령이 실행되는 중간에 다른 Client가 끼어들 수는 없다.

이처럼 하나의 연산이 중간 상태 없이 처리되는 성질을 원자성(Atomicity)이라고 한다.

### 명령 하나와 여러 명령의 조합은 다르다

재고를 읽고 Java에서 계산한 뒤 다시 저장한다고 생각해 보자.

```text
GET stock
→ Java에서 1을 뺌
→ SET stock
```

이 코드는 세 단계로 나뉜다.

```text
Client A가 재고 10을 읽음
Client B도 재고 10을 읽음
Client A가 9를 저장
Client B도 9를 저장
```

차감 요청은 두 번 들어왔지만 최종 값은 8이 아니라 9가 된다. 각각의 `GET`과 `SET`은 정상적으로 실행됐지만, `GET → 계산 → SET` 전체는 하나의 Redis 명령이 아니기 때문이다.

**단일 Redis 명령의 원자성과 여러 명령을 조합한 업무의 원자성은 다르다.**

이런 경우에는 다음 순서로 검토할 수 있다.

1. `INCRBY`, `SET NX`처럼 목적에 맞는 단일 명령이 있는지 확인한다.
2. 읽은 값이 바뀌었는지 검사해야 한다면 `WATCH`와 Transaction을 검토한다.
3. 여러 명령을 하나의 작업처럼 실행해야 한다면 Lua Script를 검토한다.

Redis Transaction은 `MULTI`와 `EXEC` 사이에 모은 명령을 차례로 실행한다. 실행이 시작된 뒤에는 다른 Client의 명령이 중간에 끼어들지 않는다.

`WATCH`한 Key가 `EXEC` 전에 변경되면 Transaction을 취소할 수 있다. 애플리케이션은 최신 값을 다시 읽고 재시도한다.

다만 관계형 데이터베이스의 Transaction과 완전히 같지는 않다. Redis Transaction 안의 한 명령이 실행 중 오류를 내더라도 이미 실행된 다른 명령을 모두 Rollback하지는 않는다.

### Pipeline은 기다리는 횟수를 줄인다

이번에는 안전성 문제가 아니라 속도 문제를 생각해 보자.

Redis 명령을 하나씩 보내고 응답을 기다리면 다음 흐름이 반복된다.

```text
명령 A 전송
→ 응답 대기

명령 B 전송
→ 응답 대기

명령 C 전송
→ 응답 대기
```

요청을 보내고 응답을 받는 한 번의 네트워크 왕복 시간을 RTT(Round Trip Time)라고 한다. 명령 자체는 매우 빨라도 매번 RTT를 기다리면 전체 시간이 길어질 수 있다.

Pipeline은 여러 명령을 먼저 모아 보내고 응답도 묶어서 받는다.

```text
A ┐
B ├→ 한 번에 전송
C ┘
  ↓
응답을 묶어서 수신
```

Pipeline이 해결하는 문제는 네트워크 왕복이다. 명령 사이에 다른 Client가 끼어들지 못하게 만드는 기능은 아니다.

```text
명령마다 응답을 기다려서 느림
→ Pipeline

명령 A와 B 사이에 다른 작업이 들어오면 안 됨
→ 단일 명령, Transaction, Lua Script
```

너무 큰 Pipeline을 한 번에 보내면 Redis가 응답을 모아 두는 메모리도 커진다. 대량 작업은 적당한 Batch로 나누는 편이 안전하다.

### 여러 명령을 하나의 업무로 묶어야 하는 사례

친구 목록 전체를 새 값으로 교체한다고 생각해 보자.

```text
DEL friends:1
RPUSH friends:1 10 20 30
EXPIRE friends:1 600
```

먼저 기존 목록을 지우고, 새 친구를 넣은 뒤, TTL을 설정한다.

같은 갱신 메시지가 중복 전달되어 두 Consumer가 동시에 실행하면 다음 순서가 가능하다.

```text
Consumer A: DEL friends:1
Consumer B: DEL friends:1

Consumer A: RPUSH friends:1 10 20 30
Consumer B: RPUSH friends:1 10 20 30

결과: 10 20 30 10 20 30
```

각각의 Redis 명령은 정상적으로 실행되었다. 하지만 “기존 목록을 새 목록으로 한 번 교체한다”는 업무 전체는 안전하지 않았다.

세 명령을 Pipeline으로 보내도 이 문제는 해결되지 않는다. Pipeline의 목적은 네트워크 왕복 감소이지 명령 조합의 원자성 보장이 아니기 때문이다.

세 동작 사이에 다른 요청이 끼어들면 안 된다면 하나의 Lua Script로 묶을 수 있다.

```text
DEL
RPUSH
EXPIRE
→ 하나의 Script로 묶음
→ Redis가 Script 전체를 하나의 작업처럼 실행
```

Lua 문법 자체보다 사용하는 이유가 중요하다. 여러 Redis 명령 사이에 다른 Client의 작업이 들어오지 못하게 해야 할 때 하나의 선택지가 된다.

Redis는 Script 실행 중 다른 Client 명령을 처리하지 않으므로 Script가 오래 걸리면 다른 요청도 기다린다. 따라서 Script는 짧고 단순하게 유지해야 한다.

### Redis Cluster에서만 추가로 볼 내용

이 부분은 Redis를 여러 Node로 나눈 Cluster를 사용할 때만 알아두면 된다.

Redis Cluster는 Key를 여러 Hash Slot에 나누어 저장한다. 여러 Key를 하나의 명령, Transaction 또는 Script로 처리하려면 관련 Key가 같은 Hash Slot에 있어야 한다.

같은 중괄호 부분을 가진 Key는 같은 Slot에 배치할 수 있다.

```text
member:{10}:profile
member:{10}:session
```

Script가 접근할 Key는 `KEYS` 인자로 명시해야 한다. 단일 Key의 친구 목록을 교체하는 앞선 예시는 이 조건을 맞추기 쉽다. 여러 Key를 함께 처리할 때만 Cluster의 Slot 배치까지 확인하면 된다.

지금까지는 Redis가 실행 중일 때 값을 다루는 방법을 살펴봤다. 이제 Redis 서버 자체가 재시작되면 어떻게 되는지 확인할 차례다.

## 6. Redis가 재시작될 때 무엇을 복구할 것인가

### 먼저 데이터가 사라져도 되는지 묻는다

RDB와 AOF 중 무엇을 사용할지 고르기 전에 더 중요한 질문이 있다.

> Redis 데이터가 모두 사라져도 DB에서 다시 만들 수 있는가?

```text
Redis 데이터가 사라짐
↓
DB에서 다시 만들 수 있음
→ Cache에 가까운 데이터

DB에서도 다시 만들 수 없음
→ Redis가 원본 데이터의 일부를 맡고 있음
```

순수한 조회 Cache라면 Redis가 비어도 DB에서 다시 채울 수 있다. 다만 한꺼번에 Cache를 다시 만들 때 DB가 그 부하를 견딜 수 있는지는 확인해야 한다.

반대로 Redis의 값이 다른 곳에는 없는 유일한 데이터라면 단순 Cache처럼 운영해서는 안 된다. Redis가 원본 저장소의 역할까지 맡고 있기 때문이다.

### RDB와 AOF는 복구 방식이 다르다

Redis 데이터를 재시작 후에도 복구할 수 있도록 디스크에 남기는 것을 영속성(Persistence)이라고 한다.

RDB는 특정 시점의 Redis 전체 상태를 Snapshot 파일로 저장한다.

```text
특정 시점의 Redis 상태
→ RDB Snapshot 저장
```

예를 들어 5분마다 Snapshot을 만든다면 장애가 발생했을 때 마지막 Snapshot 이후의 변경은 잃을 수 있다.

AOF(Append Only File)는 Redis가 처리한 쓰기 명령을 이어서 기록한다.

```text
SET, HSET, INCR 같은 쓰기 명령 기록
→ 재시작할 때 다시 실행
→ 상태 복원
```

AOF는 메모리에 쌓인 기록을 실제 디스크에 반영하는 `fsync` 주기에 따라 RDB보다 변경을 더 촘촘하게 남길 수 있다. 대신 파일 크기와 쓰기 비용이 늘어난다. 오래 쌓인 명령을 현재 상태를 만드는 데 필요한 기록만으로 정리하는 Rewrite 작업에도 비용이 든다.

Redis의 영속성 선택지는 다음과 같다.

- RDB: 특정 시점의 Snapshot을 저장
- AOF: 쓰기 명령을 이어서 기록
- RDB + AOF: 두 방식을 함께 사용
- 영속성 없음: 원본에서 다시 만들 수 있는 Cache로 사용

어느 방식이 항상 더 좋은 것은 아니다.

```text
허용 가능한 데이터 손실 범위
복구에 걸리는 시간
디스크 사용량
쓰기 비용
```

이 기준으로 선택한다.

Redis 데이터가 DB에서 다시 만들어지는 단순 Cache라면 영속성을 사용하지 않는 선택도 가능하다. 다시 만들 수 없다면 RDB나 AOF를 켜는 데서 끝내지 말고 복제, Backup과 실제 복구 절차까지 설계해야 한다.

### 빠르게 변하는 값과 오래 보관할 값을 나눈다

모든 데이터가 같은 수명을 가지는 것은 아니다.

실시간 방송에서 시청자가 반응 버튼을 연속해서 누른다고 생각해 보자. 방송 중에는 같은 숫자를 매우 자주 늘려야 하지만, 방송이 끝나면 최종 반응 수만 장기간 보관해도 충분할 수 있다.

```text
방송 중
반응 요청이 많이 들어옴
→ Redis INCRBY로 Counter 증가

방송 종료
최종 반응 수 확인
→ MySQL 저장
→ Redis Key 삭제
```

이 구조에서 Redis는 빠르게 변하는 일시적인 상태를 맡고, MySQL은 장기 보관할 최종 결과를 맡는다.

하지만 “실시간 데이터는 Redis, 원본 데이터는 MySQL”이라는 공식이 항상 맞는 것은 아니다.

```text
데이터가 얼마나 자주 바뀌는가?
얼마나 오래 보관해야 하는가?
유실을 허용할 수 있는가?
어떤 방식으로 조회하는가?
장애 후 어떻게 복구하는가?
```

이 질문을 기준으로 저장소의 책임을 나눈다.

Redis 값을 MySQL로 옮기는 과정도 실패할 수 있다. MySQL 저장은 성공했는데 Redis Key를 지우기 전에 애플리케이션이 종료될 수 있다. 재시도하면서 같은 값을 두 번 반영할 수도 있다.

따라서 다음 기준까지 정해야 한다.

- 어느 단계까지 끝나야 작업이 완료된 것인가?
- 실패하면 어느 단계부터 다시 시작할 것인가?
- 같은 작업이 다시 실행되어도 결과가 중복되지 않는가?

### 실시간 처리와 분석에 필요한 데이터는 다르다

API 요청을 처리할 때는 현재 반응 수만 필요할 수 있다.

```text
지금 반응 수가 몇 개인가?
→ Redis Counter
```

반면 분석에는 더 자세한 정보가 필요하다.

```text
누가 반응했는가?
언제 반응했는가?
어떤 반응이었는가?
→ 별도 Event 또는 Log
```

두 요구를 분리하면 실시간 요청 경로에서는 단순한 Counter를 빠르게 처리하고, 분석에는 필요한 상세 기록을 별도로 남길 수 있다.

지금까지 Redis를 직접 다루는 기준을 살펴봤다. 마지막으로 이런 조회 Cache를 Spring 코드에서 어떻게 표현하는지 확인해 보자.

## 7. Spring에서 캐시를 사용하는 방법

### `@Cacheable`은 메서드 결과를 재사용한다

일반적인 조회 Cache를 만들 때마다 Service 코드에서 Redis의 `GET`과 `SET`을 직접 작성해야 할까?

Spring Cache를 사용하면 메서드 결과의 캐시 여부를 Annotation으로 선언할 수 있다.

```java
@Cacheable(cacheNames = "equipment", key = "#equipmentId")
public EquipmentResponse getEquipment(long equipmentId) {
    return equipmentRepository.findResponse(equipmentId);
}
```

먼저 실행 흐름부터 살펴보자.

```text
getEquipment(42) 호출
→ equipment Cache에서 Key 42 확인

값이 있음
→ 메서드를 실행하지 않고 저장된 값 반환

값이 없음
→ 실제 메서드 실행
→ 반환값을 Cache에 저장
→ 반환
```

이 동작을 제공하는 공통 규칙이 Spring Cache다.

### Spring Cache와 Redis는 같은 것이 아니다

`@Cacheable`을 붙였다고 무조건 Redis를 사용하는 것은 아니다.

```text
Spring Cache
→ 캐시를 사용하는 공통 Annotation과 API

CacheManager와 Cache Provider
→ 실제 값을 어디에 저장할지 결정
→ Redis, Caffeine 등
```

Cache Provider는 실제 캐시 기능을 제공하는 구현체다. `CacheManager` 설정이 애플리케이션에서 사용할 Provider와 Cache를 연결한다.

Annotation 기반 캐시를 사용하려면 Spring 설정에 `@EnableCaching`도 필요하다.

```java
@Configuration
@EnableCaching
public class CacheConfig {
}
```

Annotation만 붙였다고 TTL, Key Prefix, 직렬화 방식과 Redis 장애 시 동작까지 자동으로 정해지는 것은 아니다. 이런 세부 정책은 Provider와 `CacheManager` 설정에서 별도로 결정한다.

### 원본이 바뀌면 캐시도 지워야 한다

조회 결과를 Cache에 저장한 뒤 DB의 값만 수정하면 다음 상태가 된다.

```text
DB
→ 새 값

Cache
→ 예전 값
```

`@CacheEvict`는 데이터 변경 메서드가 성공한 뒤 관련 Cache Entry를 제거하는 데 사용할 수 있다.

```java
@CacheEvict(cacheNames = "equipment", key = "#equipmentId")
public void updateEquipment(long equipmentId, UpdateEquipmentCommand command) {
    equipmentRepository.update(equipmentId, command);
}
```

`beforeInvocation`의 기본값은 `false`다. 따라서 위 예제의 Cache 제거는 메서드가 예외 없이 끝난 뒤 실행된다. 다만 **메서드가 정상적으로 끝난 시점과 DB Transaction이 Commit된 시점이 항상 같다는 뜻은 아니다.** Cache 제거를 Commit 성공과 반드시 묶어야 한다면 `@TransactionalEventListener`처럼 Commit 단계에 동작을 연결하는 방법을 검토해야 한다. 이 Annotation은 기본적으로 `AFTER_COMMIT` 단계에 Listener를 실행한다.

다음 조회에서는 Cache에 값이 없으므로 DB의 새 값을 읽어 다시 저장한다.

같은 원본을 변경하는 경로가 여러 개라면 모든 경로에서 관련 Cache를 무효화해야 한다. 하나라도 빠지면 예전 값이 남을 수 있다. TTL은 이런 삭제 누락이 영구적인 불일치로 이어지지 않도록 돕는 안전망이 된다.

### 같은 클래스 내부 호출은 Proxy를 거치지 않는다

이 부분은 Spring AOP의 동작과 관련된 보충 내용이다.

Spring Cache의 기본 방식에서는 Spring이 실제 객체 앞에 Proxy를 둔다. 외부 Bean이 메서드를 호출하면 Proxy가 먼저 Cache를 확인한다.

```text
외부 Bean
→ Cache Proxy
→ 실제 메서드
```

하지만 같은 객체 안에서 자기 메서드를 직접 호출하면 Proxy를 다시 거치지 않는다.

```text
this.getEquipment(42)
→ 같은 객체 내부 호출
→ Cache Proxy를 거치지 않음
```

이 경우 `getEquipment()`에 `@Cacheable`이 있어도 캐시 기능이 적용되지 않을 수 있다. 캐시 Annotation을 적용한 메서드는 외부 Bean을 통해 호출되는 `public` 메서드로 두고, 같은 클래스의 Self Invocation을 피하는 편이 안전하다.

## 정리

Redis Cache를 사용하는 목적은 단순히 메모리가 빠르기 때문만은 아니다.

```text
반복되는 SQL
Join과 집계
DB Connection 점유
결과 전송
객체와 DTO 변환
```

캐시의 가치는 이미 만든 결과를 재사용해 DB와 애플리케이션의 반복 작업을 줄이는 데 있다.

실제 도입 여부는 다음 순서로 판단할 수 있다.

```text
같은 조회가 충분히 반복되는가?
→ 원본 조회 비용이 큰가?
→ Cache Hit 비율을 확보할 수 있는가?
→ Cache 검토

여러 애플리케이션 인스턴스가 Cache를 공유해야 하는가?
→ Redis 같은 외부 Cache 검토

Redis에서 어떤 연산이 필요한가?
→ 자료형과 Key 선택

오래된 값을 얼마 동안 허용할 수 있는가?
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
- [Spring Framework - Transaction-bound Events](https://docs.spring.io/spring-framework/reference/data-access/transaction/event.html)

### 국내 기술 블로그

- [LINE Engineering - Atomic 처리와 Cache Stampede 대책을 위해 Redis Lua Script를 활용한 이야기](https://engineering.linecorp.com/ko/blog/atomic-cache-stampede-redis-lua-script/)
- [LINE Engineering - 안정적인 Love를 제공하는 방법](https://engineering.linecorp.com/ko/blog/how-to-provide-stable-loves/)
