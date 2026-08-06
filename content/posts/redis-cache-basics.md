+++
title = '캐시는 DB 부하를 어떻게 줄이고 Redis는 무엇을 맡는가'
date = 2026-04-03T19:10:00+09:00
lastmod = 2026-08-06T17:40:00+09:00
draft = false
description = '캐시가 필요한 이유에서 출발해 Redis의 자료형, TTL, 영속성, Pipeline과 Spring Cache 연동을 정리합니다.'
categories = ['데이터 접근 설계']
tags = ['Redis', '캐시', 'Spring Cache', 'TTL']
+++

데이터베이스에도 Buffer Pool이 있는데 왜 Redis 같은 별도 캐시를 둘까? 데이터베이스의 메모리 캐시는 Page와 Index 접근을 빠르게 만들지만, 애플리케이션이 원하는 최종 응답을 그대로 보관하지는 않는다.

Redis에는 직렬화한 조회 결과나 계산 결과를 Key로 저장할 수 있다. 같은 요청이 반복되면 Join과 집계, 객체 변환을 다시 수행하지 않고 완성된 결과를 바로 반환할 수 있다.

Redis는 애플리케이션 안에 있는 Java `Map`이 아니라 별도 프로세스로 실행되는 서버다. 애플리케이션은 네트워크로 명령을 보내고 결과를 받는다. 여러 애플리케이션 인스턴스가 같은 Redis를 바라볼 수 있다는 장점이 있지만, 네트워크 지연과 Redis 장애도 새로운 실패 지점으로 들어온다.

## 캐시가 줄이는 비용

캐시가 없으면 요청마다 원본 저장소를 거친다.

```text
요청 → 애플리케이션 → DB 연결 대여 → SQL 실행 → 결과 변환 → 응답
```

캐시 Hit가 발생하면 흐름이 짧아진다.

```text
요청 → 애플리케이션 → Redis 조회 → 응답
```

여기서 얻는 이점은 메모리가 디스크보다 빠르다는 설명만으로 끝나지 않는다. 복잡한 SQL 실행, Lock 경합, Connection Pool 점유와 결과 조립을 함께 줄일 수 있다. 반대로 Cache Miss 때는 Redis와 DB를 모두 방문하므로 반복 조회가 적으면 오히려 왕복이 하나 늘어난다.

이 비용을 줄이려면 무엇을 Key 하나에 저장할지부터 정해야 한다. Redis는 단순한 Cache Map이 아니라 필요한 연산에 맞춰 자료형을 고를 수 있는 저장소다.

## Redis의 Key와 자료형

Redis는 단순한 문자열 저장소가 아니라 자료구조 서버다. 자주 사용하는 자료형은 다음과 같다.

- String: JSON, 숫자 Counter와 단일 값
- Hash: 한 객체의 여러 Field
- List: 입력 순서를 가진 값 목록
- Set: 중복 없는 값의 집합
- Sorted Set: Score로 정렬된 순위와 시간 기반 데이터

```text
SET plant:1:summary '{"status":"NORMAL"}' EX 600
HSET member:10 name "Kim" role "OPERATOR"
SADD post:1:viewers "member:10"
ZADD ranking 1200 "member:10"
```

자료형은 Java Collection 이름과 겉모양만 보고 고르면 안 된다. 필요한 연산을 기준으로 정한다. 중복 확인이 필요하면 Set, Score 범위와 순위가 필요하면 Sorted Set, 부분 Field 갱신이 필요하면 Hash가 자연스럽다.

Key는 Redis 안에서 데이터를 찾는 유일한 이름이므로 충돌과 운영 조회를 함께 고려해야 한다. 보통 `업무:대상:식별자:용도`처럼 일정한 구조를 사용한다.

```text
equipment:42:detail
member:10:session
post:100:viewers
```

`detail42`처럼 맥락이 없는 이름보다 어떤 업무의 어떤 데이터인지 바로 알 수 있고, Prefix 단위로 메모리 사용량과 만료 정책을 확인하기도 쉽다. 다만 운영 환경에서 `KEYS equipment:*`처럼 전체 공간을 한 번에 훑는 명령은 데이터가 많을수록 서버를 오래 점유할 수 있으므로 `SCAN`처럼 점진적으로 탐색하는 방식을 사용한다.

자료형이 값의 연산 방식을 정한다면 TTL은 그 값이 얼마 동안 유효한지를 정한다. 캐시에서는 저장 방법만큼 오래된 값을 언제 버릴지도 중요하다.

## TTL은 캐시 수명을 제한한다

TTL을 두면 Key가 일정 시간이 지난 뒤 자동으로 만료된다.

```text
SET equipment:42:detail "..." EX 300
TTL equipment:42:detail
```

TTL은 세 가지 역할을 한다.

- 오래된 값이 무한히 남지 않게 한다.
- 사용되지 않는 Key를 정리해 메모리 사용을 제한한다.
- 캐시 삭제 실패가 영구적인 불일치로 이어지는 것을 막는다.

TTL이 짧으면 원본 조회와 재적재가 자주 발생하고, 길면 오래된 값을 보는 시간이 늘어난다. 업무가 허용하는 데이터 신선도와 원본 조회 비용을 함께 보고 정해야 한다.

만료와 Eviction도 구분해야 한다. 만료는 개발자가 정한 TTL이 끝나 Key가 제거되는 일이다. Eviction은 Redis가 `maxmemory`에 도달했을 때 설정된 정책에 따라 Key를 쫓아내는 일이다. TTL이 아직 남아 있어도 메모리가 부족하면 Eviction 대상이 될 수 있다. 따라서 예상보다 Hit Ratio가 떨어진다면 TTL뿐 아니라 메모리 사용량과 Eviction 횟수도 확인해야 한다.

여기까지는 명령 하나의 결과를 기준으로 설명했다. 여러 Client가 동시에 접근하고 여러 명령을 조합하기 시작하면 “Redis는 단일 스레드”라는 말만으로는 안전성을 판단할 수 없다.

## Redis는 단일 스레드라는 말의 범위

Redis를 “단일 스레드라서 Lock이 필요 없다”고만 설명하면 현재 구조를 지나치게 단순화하게 된다. 핵심 명령 실행은 한 번에 처리되어 `INCR` 같은 단일 명령은 원자적으로 동작한다. 하지만 네트워크 I/O와 백그라운드 저장 같은 주변 작업에는 별도 스레드와 프로세스가 사용될 수 있다.

또한 명령 하나가 원자적이라고 여러 명령의 조합까지 원자적인 것은 아니다.

```text
GET stock
애플리케이션에서 계산
SET stock
```

이 사이에는 다른 Client의 변경이 들어올 수 있다. 조건부 변경에는 `WATCH`와 Transaction, Lua Script 또는 목적에 맞는 단일 명령을 사용해야 한다.

여러 명령을 다룰 때는 두 문제를 먼저 구분해야 한다. 네트워크 왕복이 많은 것인지, 명령 사이에 다른 작업이 끼어들면 안 되는 것인지에 따라 Pipeline과 원자적 실행의 선택이 달라진다.

## Pipeline은 원자성이 아니라 왕복을 줄인다

여러 명령을 차례로 보내고 매번 응답을 기다리면 네트워크 RTT가 반복된다. Pipeline은 명령을 모아 보내고 응답도 한 번에 받아 왕복 횟수를 줄인다.

```text
명령 A ┐
명령 B ├→ 한 번에 전송 → 결과 A, B, C
명령 C ┘
```

Pipeline은 명령 묶음의 원자적 실행을 보장하지 않는다. 다른 Client 명령과 섞이지 않게 실행해야 한다면 Redis Transaction이나 Script가 필요하다. 지나치게 큰 Pipeline은 서버가 응답을 보관하는 메모리를 늘리므로 적당한 Batch로 나눠야 한다.

목록 전체를 교체하기 위해 `DEL`, `RPUSH`, `EXPIRE`를 순서대로 실행한다고 해보자. 같은 갱신 메시지가 중복 전달되어 두 Consumer가 동시에 실행하면 다음처럼 명령 사이에 다른 작업이 끼어들 수 있다.

```text
Consumer A: DEL friends:1
Consumer B: DEL friends:1
Consumer A: RPUSH friends:1 10 20 30
Consumer B: RPUSH friends:1 10 20 30
결과: 10 20 30 10 20 30
```

Pipeline으로 세 명령을 함께 보내도 네트워크 왕복만 줄어들 뿐 다른 Client의 명령이 끼어들지 않는다고 보장하지 않는다. 이때 `DEL → RPUSH → EXPIRE`를 Lua Script 하나로 만들면 Redis는 Script 전체를 하나의 명령처럼 실행한다.

Redis Cluster에서는 Script가 접근할 Key를 `KEYS`로 전달하고 관련 Key가 같은 Hash Slot에 있도록 설계해야 한다. 단일 Key의 목록 교체라면 이 조건을 맞추기 쉽다. 중요한 것은 Lua Script 자체가 아니라 **여러 명령 사이의 간섭이 문제인지, 단순히 왕복 횟수가 문제인지 먼저 구분하는 것**이다.

명령을 안전하게 실행하는 방법을 정한 뒤에는 Redis 자체가 사라졌을 때 무엇을 복구해야 하는지도 결정해야 한다. 같은 Redis라도 다시 채울 수 있는 캐시와 원본 상태를 맡은 저장소는 내구성 요구가 다르다.

## Redis를 복구 가능한 캐시로 볼지 먼저 정한다

Redis는 메모리 데이터만 사용하는 구성도 가능하고 디스크에 남길 수도 있다.

- RDB: 특정 시점의 Dataset Snapshot을 생성
- AOF: 쓰기 명령을 Log에 기록하고 재시작 때 재생
- RDB + AOF: 두 방식을 함께 사용
- 영속성 없음: 원본에서 다시 만들 수 있는 순수 캐시

RDB는 파일이 작고 복구가 빠르지만 Snapshot 사이의 변경을 잃을 수 있다. AOF는 더 촘촘하게 변경을 남길 수 있지만 파일과 Rewrite 비용이 생긴다. Redis가 없어져도 DB에서 다시 채울 수 있는 캐시라면 영속성을 끄는 선택도 가능하다. 반대로 Redis가 업무 원본이나 작업 Queue 역할까지 맡는다면 캐시와 같은 기준으로 판단하면 안 된다.

예를 들어 5분마다 RDB Snapshot을 만든다면 장애 시 마지막 Snapshot 이후 변경은 남지 않을 수 있다. AOF는 실행한 쓰기 명령을 이어 기록하고 재시작할 때 이를 다시 실행해 상태를 복원한다. 어느 쪽이 무조건 우월한 것이 아니라 허용할 데이터 손실 범위와 복구 시간, 디스크 비용이 다르다.

따라서 RDB와 AOF부터 고르기 전에 Redis의 데이터가 사라졌을 때 원본에서 다시 만들 수 있는지부터 확인해야 한다. 다시 만들 수 있다면 복구 속도와 비용을 기준으로 선택하면 되고, 다시 만들 수 없다면 Redis를 단순 캐시가 아닌 원본 저장소의 일부로 보고 내구성과 복구 절차를 설계해야 한다.

모든 값을 같은 내구성으로 관리할 필요도 없다. 빠르게 변하는 실시간 상태와 오래 보관할 결과의 수명이 다르다면 Redis와 관계형 데이터베이스가 서로 다른 책임을 맡을 수 있다.

## 실시간 상태와 장기 보관 데이터를 나눈다

실시간 방송의 반응 수처럼 짧은 시간에 증가 요청이 몰리는 값은 역할을 나눌 수 있다. 방송 중에는 Redis Counter를 `INCRBY`로 갱신하고, 방송이 끝나면 최종 값을 MySQL로 옮긴 뒤 Redis Key를 삭제한다. Redis에는 처리량이 중요한 일시적 상태를, MySQL에는 장기 보관할 결과를 맡기는 구조다.

분석에 필요한 세부 입력까지 Counter 하나에 넣을 필요도 없다. 요청 처리에는 단순한 숫자만 사용하고, 어느 시점에 반응이 발생했는지 같은 상세 기록은 별도 Log 흐름으로 보내 분석 저장소에 쌓을 수 있다. 실시간 처리 모델과 분석 모델을 분리하면 Counter의 단순함을 유지하면서도 나중에 필요한 정보를 잃지 않는다.

여기서 핵심은 모든 실시간 값을 Redis에 넣는 것이 아니다. 서비스 중 빠르게 바뀌는 상태와 장기 보관해야 하는 원본의 수명이 다를 때 저장소의 책임도 나눌 수 있다는 점이다. 이 경우 Redis에서 MySQL로 옮기는 시점과 실패했을 때 다시 시작할 기준까지 함께 정해야 한다.

이제 이런 저장소 선택을 Spring 애플리케이션 코드에 어떻게 드러낼지 남는다. Spring Cache는 Redis의 세부 API를 직접 호출하지 않고도 메서드 결과의 캐시 경계를 선언하게 해준다.

## Spring Cache와 연결하기

Spring Cache는 Cache Provider를 바꿔도 비슷한 Annotation 모델을 사용하게 해준다.

호출자는 Redis 명령을 직접 알 필요 없이 “이 메서드 결과를 어떤 이름과 Key로 캐시할지”를 선언한다. Spring AOP Proxy가 메서드 호출 앞에서 캐시를 확인하고 Hit면 메서드를 실행하지 않는다. Miss일 때만 실제 메서드를 실행하고 반환값을 저장한다.

```text
호출 → Cache Proxy가 Key 조회
          ├─ Hit  → 저장된 결과 반환, 메서드 실행 생략
          └─ Miss → 메서드 실행 → 반환값 저장 → 반환
```

```java
@Cacheable(cacheNames = "equipment", key = "#equipmentId")
public EquipmentResponse getEquipment(long equipmentId) {
    return equipmentRepository.findResponse(equipmentId);
}

@CacheEvict(cacheNames = "equipment", key = "#equipmentId")
public void updateEquipment(long equipmentId, UpdateEquipmentCommand command) {
    equipmentRepository.update(equipmentId, command);
}
```

실제 Cache가 Redis인지 Caffeine인지는 `CacheManager` 설정이 결정한다. Annotation만 붙였다고 적절한 TTL과 직렬화, Key Prefix, 장애 시 동작까지 자동으로 정해지는 것은 아니다.

`@CacheEvict`도 같은 Proxy 방식이므로 같은 클래스 안에서 자기 메서드를 직접 호출하면 기대한 AOP가 적용되지 않을 수 있다. 캐시 Annotation은 저장소 API를 줄여주지만, Proxy 호출 조건과 데이터 변경 경로 전체의 무효화 책임까지 없애주지는 않는다.

## 정리

- Redis Cache는 완성된 조회 결과와 계산 결과를 재사용해 DB 작업을 줄인다.
- Hit가 충분하지 않으면 Redis 왕복만 추가될 수 있다.
- 자료형은 저장 모양보다 필요한 연산을 기준으로 선택한다.
- 단일 Redis 명령의 원자성과 여러 명령 조합의 원자성은 다르다.
- RDB와 AOF는 Redis가 단순 캐시인지 원본 데이터까지 맡는지에 따라 선택한다.
- 실시간 상태와 장기 보관 데이터의 수명이 다르면 저장소의 책임도 나눌 수 있다.

## 참고 자료

### 공식 자료

- [Redis - Data Types](https://redis.io/docs/latest/develop/data-types/)
- [Redis - Pipelining](https://redis.io/docs/latest/develop/using-commands/pipelining/)
- [Redis - Persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
- [Spring Framework - Cache Abstraction](https://docs.spring.io/spring-framework/reference/integration/cache.html)

### 국내 기술 블로그

- [LINE Engineering - Atomic 처리와 cache stampede 대책을 위해 Redis Lua script를 활용한 이야기](https://engineering.linecorp.com/ko/blog/atomic-cache-stampede-redis-lua-script/)
- [LINE Engineering - 안정적인 love를 제공하는 방법](https://engineering.linecorp.com/ko/blog/how-to-provide-stable-loves/)
