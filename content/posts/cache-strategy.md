+++
title = '캐시 전략은 읽기 성능과 데이터 정합성을 어떻게 맞추는가'
slug = '7'
aliases = ['/posts/007/']
date = 2026-04-10T19:00:00+09:00
lastmod = 2026-08-10T17:12:37+09:00
draft = false
description = '캐시를 도입한 뒤 생기는 두 상태를 출발점으로 읽기와 쓰기 전략, 무효화, 장애 우회와 동시 만료 대응을 차례로 정리합니다.'
categories = ['데이터 접근 설계']
tags = ['캐시', 'Redis', 'Cache Aside', '정합성']
+++

캐시가 없을 때 데이터의 기준은 데이터베이스 하나다.

```text
요청 → DB → 응답
```

캐시를 추가하면 반복 조회를 빠르게 처리할 수 있다. 대신 애플리케이션이 관리해야 할 값이 두 개가 된다.

```text
DB        Cache
원본 값    원본에서 복사한 값
```

DB가 바뀐 직후에도 Cache에는 이전 값이 남을 수 있다. 따라서 캐시의 어려움은 값을 저장하는 데 있지 않다. **DB가 변경된 뒤 Cache와 어떻게 맞출지 정하는 데 있다.**

이 글에서는 이 문제를 다음 순서로 살펴본다.

```text
Cache에서 먼저 읽기
→ DB 변경 뒤 Cache 무효화
→ 반복되는 Miss 처리 책임 옮기기
→ DB와 Cache에 쓰는 시점 정하기
→ Cache 장애 시 우회 경로 만들기
→ 인기 Key의 동시 Miss 막기
→ 캐시할 데이터 고르기
```

`Cache Aside`, `Read Through`, `Write Through` 같은 이름은 이 흐름에서 **누가 읽기와 쓰기를 책임지는지** 정리한 패턴이다. 이름부터 외우기보다 앞 단계에서 어떤 문제가 생겼고, 다음 단계가 그 문제를 어떻게 다루는지 따라가 보자.

## 1. 캐시를 넣으면 무엇이 달라지는가

발전소 요약 정보를 조회하는 요청을 생각해보자. 먼저 Cache에서 값을 찾는다.

```text
요청
↓
Cache 조회

값 있음
→ 바로 반환

값 없음
→ DB 조회
→ Cache 저장
→ 반환
```

캐시에 값이 있는 경우를 `Cache Hit`, 없는 경우를 `Cache Miss`라고 한다. 이름보다 두 경로의 비용 차이가 중요하다.

```text
Hit
요청 → Cache → 응답

Miss
요청 → Cache → DB → Cache 저장 → 응답
```

Miss가 발생한 첫 요청은 Cache와 DB를 모두 거치고 Cache 저장까지 수행한다. 캐시를 넣었다고 첫 요청부터 무조건 빨라지는 것은 아니다. 같은 Key가 다시 조회되어 Hit가 발생해야 앞에서 지불한 적재 비용을 회수할 수 있다.

따라서 캐시는 단순히 빠른 저장소가 아니다. **반복되는 요청에서 이전 결과를 재사용해 원본 조회를 줄이는 장치다.** 이제 이 흐름을 애플리케이션 코드로 직접 구현하면 어떤 모습인지 살펴보자.

## 2. 애플리케이션이 읽기와 무효화를 관리한다

### 가장 단순한 읽기 흐름

아래 그림에서 Miss가 발생했을 때 요청이 Cache에서 끝나지 않고 DB까지 내려간 뒤, 조회 결과가 다시 Cache에 저장되는 순서를 먼저 보자.

![Cache Miss일 때 원본 저장소를 조회하고 캐시에 적재하는 흐름](/images/posts/cache-strategy/legacy-01.png "Cache Aside")

이를 Java 코드로 직접 작성하면 다음과 같다.

```java
public PlantSummary getSummary(long plantId) {
    String key = "plant:summary:" + plantId;

    PlantSummary cached =
        cache.get(key, PlantSummary.class);

    if (cached != null) {
        return cached;
    }

    PlantSummary loaded =
        repository.findSummary(plantId);

    cache.put(
        key,
        loaded,
        Duration.ofMinutes(10)
    );

    return loaded;
}
```

Service가 다음 순서를 직접 결정하고 있다.

```text
1. Cache 조회
2. Hit면 바로 반환
3. Miss면 DB 조회
4. 조회 결과를 Cache에 저장
5. 반환
```

이렇게 애플리케이션이 Cache와 원본 저장소 사이의 흐름을 직접 관리하는 방식을 `Cache Aside`라고 한다.

장점과 단점은 같은 책임에서 나온다.

| 장점 | 단점 |
|---|---|
| 실제로 조회된 값만 Cache에 저장할 수 있다. | Service가 Cache 조회와 Miss 처리를 알아야 한다. |
| Cache 장애 시 DB를 직접 읽는 우회 경로를 만들기 쉽다. | 여러 Service에 비슷한 조회 코드가 반복될 수 있다. |
| 애플리케이션이 흐름을 세밀하게 제어할 수 있다. | DB 변경 뒤 Cache를 지우는 책임도 애플리케이션에 생긴다. |

Cache Aside는 단순하고 제어하기 쉽다. 그 대가로 Cache 관리 책임이 애플리케이션 코드 안으로 들어온다.

### 조회는 해결됐지만 값이 바뀌면 문제가 생긴다

발전소 이름이 변경됐다고 해보자.

```text
변경 전

DB    = 발전소 A
Cache = 발전소 A

DB 수정 후

DB    = 발전소 B
Cache = 발전소 A
```

이제 같은 발전소를 조회해도 DB와 Cache가 서로 다른 값을 가리킨다. 원본이 바뀌었을 때 이전 Cache를 지우거나 더 이상 사용하지 않게 만드는 작업을 `Cache Invalidation`, 즉 캐시 무효화라고 한다.

DB를 수정한 뒤 Cache에도 새 값을 바로 쓰면 되지 않을까? 가능하지만 변경된 필드와 Cache에 저장한 조회 결과가 정확히 대응하는지 계속 관리해야 한다. 반면 Cache를 삭제해두면 다음 조회가 DB의 최신 값을 다시 읽어 Cache를 채운다.

```text
DB 변경 성공
→ Cache 삭제
→ 다음 조회에서 최신 값 재적재
```

흐름이 단순해 실무에서 자주 선택하지만, 이것만으로 DB와 Cache가 항상 같아지는 것은 아니다.

### DB를 먼저 바꿔도 실패 구간이 남는다

일반적으로 DB 변경을 Commit한 뒤 Cache를 삭제한다. 그런데 두 작업 사이에서 애플리케이션이 종료되면 어떻게 될까?

```text
1. DB 수정 Commit
2. Cache 삭제 예정
3. Cache를 삭제하기 전에 Application 장애

결과

DB    = 새 값
Cache = 이전 값
```

DB 변경은 끝났지만 Cache에는 이전 값이 남는다. Cache 삭제를 재시도하거나 이벤트로 보강하지 않는다면, 이후 요청은 오래된 값을 읽을 수 있다.

### Cache를 먼저 지워도 Race가 생길 수 있다

그렇다면 Cache를 먼저 삭제하면 안전할까? 이번에는 두 요청이 엇갈릴 수 있다.

```text
1. 요청 A가 Cache 삭제
2. 요청 B가 조회 시작
3. 아직 DB에는 이전 값이 있으므로 요청 B가 이전 값을 조회
4. 요청 B가 이전 값을 Cache에 다시 저장
5. 요청 A가 DB 수정 Commit

결과

DB    = 새 값
Cache = 이전 값
```

Cache 삭제와 DB Commit 사이에 다른 조회가 들어와 이전 값을 다시 적재한 것이다. 동시에 실행되는 두 작업의 순서에 따라 결과가 달라지는 이런 상황을 Race라고 한다.

결국 순서만 바꾼다고 DB와 Cache를 하나의 트랜잭션처럼 만들 수는 없다. 두 저장소에 걸친 작업 중 하나만 성공할 수 있다는 사실을 전제로 복구 방법을 설계해야 한다.

### TTL은 동기화 기능이 아니라 마지막 안전장치다

Cache Key에 10분의 `TTL`을 설정했다고 해보자. TTL은 Key가 살아 있을 수 있는 시간을 제한한다.

```text
Cache 삭제 실패
↓
이전 값이 Cache에 남음
↓
TTL 만료
↓
다음 조회가 DB의 최신 값을 다시 적재
```

이 흐름은 DB와 Cache가 10분마다 자동으로 동기화된다는 뜻이 아니다. 삭제가 실패했다면 사용자는 TTL이 끝날 때까지 이전 값을 볼 수 있다.

따라서 TTL은 **오래된 Cache가 영원히 남는 것을 막는 마지막 안전장치다.** 즉시 정합성을 보장하는 기능은 아니다.

여기서 기술보다 먼저 답해야 할 질문이 있다.

- DB와 Cache가 1초만 달라도 안 되는 데이터인가?
- 수십 초 정도의 이전 값을 보여줘도 되는 데이터인가?
- 삭제 실패를 어떻게 발견하고 재시도할 것인가?
- 재시도도 실패했을 때 TTL로 제한할 최대 시간은 얼마인가?

캐시 전략은 읽기 성능만 선택하는 문제가 아니다. **원본과 Cache가 어긋난 상태를 얼마나 오래 허용할지 결정하는 문제이기도 하다.**

Cache Aside의 동작은 이해했지만, 여러 Service에서 같은 Miss 처리 코드가 반복될 수 있다. 다음에는 이 책임을 호출 코드 밖으로 옮기는 방법을 살펴보자.

## 3. 반복되는 Miss 처리를 캐시 계층에 맡긴다

Cache Aside를 여러 곳에 적용하면 다음 세 줄이 계속 나타날 수 있다.

```java
cache.get(...);
repository.find(...);
cache.put(...);
```

Cache 조회와 원본 조회 순서는 같지만 각 Service가 이를 반복해서 작성한다. 그렇다면 Miss 처리 자체를 캐시 계층에 맡길 수는 없을까?

### Read Through는 책임의 위치를 바꾼다

호출자는 Cache 계층에 값만 요청한다. 값이 없다면 Cache 계층이 Loader를 실행해 원본을 조회하고 결과를 저장한다.

```text
Service
→ Cache 계층 호출

Cache 계층
→ Hit면 값 반환
→ Miss면 Loader 실행
→ DB 조회
→ Cache 적재
→ 값 반환
```

Cache Miss 처리와 원본 조회를 Cache 계층에 맡기는 이 방식을 `Read Through`라고 한다.

Cache Aside와 Read Through는 모두 Cache를 먼저 본다. 두 방식의 차이는 Cache 사용 여부가 아니라 **Miss 처리를 누가 책임지느냐에 있다.**

```text
Cache Aside

Service
→ Cache 확인
→ Miss 판단
→ DB 조회
→ Cache 적재


Read Through

Service
→ Cache 계층 호출

Cache 계층
→ Miss 확인
→ Loader 실행
→ DB 조회
→ Cache 적재
```

호출 코드는 단순해지지만 Cache 계층이나 Provider가 Loader의 동작을 알아야 한다. 책임이 사라진 것이 아니라 다른 곳으로 이동한 것이다.

### Spring `@Cacheable`은 Cache 처리 코드를 감춘다

Spring에서는 다음처럼 메서드 결과를 캐시할 수 있다.

```java
@Cacheable(
    cacheNames = "plantSummary",
    key = "#plantId"
)
public PlantSummary getSummary(long plantId) {
    return repository.findSummary(plantId);
}
```

호출 흐름은 다음과 같다.

```text
메서드 호출
↓
Cache 확인

Hit
→ 메서드를 실행하지 않고 값 반환

Miss
→ 메서드 실행
→ 반환값을 Cache에 저장
→ 값 반환
```

Miss 처리가 호출 코드 밖으로 숨겨진다는 점에서는 Read Through와 비슷하다. 그러나 전통적인 Read Through처럼 Cache Provider 자체가 DB를 알고 읽어오는 구현과 완전히 같다는 뜻은 아니다. `@Cacheable`에서는 원본을 읽는 메서드가 Loader와 비슷한 역할을 한다.

또한 Spring Cache는 Redis 같은 저장소가 아니다. **공통된 Cache 동작을 제공하는 추상화다.** 실제 저장 방식과 TTL, 분산 환경의 동작은 Redis나 Caffeine 같은 구현체와 설정에 따라 달라진다.

지금까지는 Cache에 값이 없을 때 어디서 읽을지 살펴봤다. 이번에는 원본 값 자체가 변경될 때 DB와 Cache 중 무엇을 언제 바꿀지 살펴보자.

## 4. 쓰기 응답 시점과 원본 반영 시점을 정한다

읽기에서는 Hit와 Miss가 기준이었다. 쓰기에서는 다음 두 질문이 기준이 된다.

```text
누가 DB와 Cache에 쓰는 흐름을 관리하는가?

사용자에게 성공을 응답할 때 DB 반영이 끝나 있어야 하는가?
```

### 응답 전에 원본까지 반영하는 Write Through

아래 그림에서는 쓰기 요청이 Cache 계층을 지나 원본 저장소까지 반영된 뒤 완료되는 흐름을 확인할 수 있다.

![캐시와 데이터베이스에 함께 기록하는 Write Through](/images/posts/cache-strategy/legacy-02.png "Write Through")

```text
쓰기 요청
↓
Cache 계층
↓
DB 반영
↓
Cache 반영
↓
성공 응답
```

응답 전에 Cache와 원본 저장소까지 함께 반영하는 방식을 `Write Through`라고 한다. 구현에 따라 내부 작업 순서는 달라질 수 있지만, 성공을 응답하기 전에 원본 반영까지 기다린다는 약속이 핵심이다.

| 얻는 점 | 감수할 점 |
|---|---|
| 쓰기 직후 Cache에 최신 값이 있을 가능성이 높다. | DB 반영을 기다리므로 쓰기 응답 시간이 늘어난다. |
| 다음 읽기에서 Hit가 발생하기 쉽다. | Cache 계층이 원본 저장 책임까지 알아야 한다. |
| 읽기 시 오래된 값이 나올 구간을 줄일 수 있다. | DB나 Cache 중 한쪽의 장애가 쓰기 요청에 전파될 수 있다. |

Redis를 사용한다고 자동으로 Write Through가 되는 것은 아니다. Redis는 값을 저장하는 기술이고, Write Through는 애플리케이션이나 Cache Provider가 쓰기 흐름을 어떻게 구성했는지 설명하는 전략이다.

### DB 반영을 나중으로 미루는 Write Behind

이번에는 아래 그림에서 사용자 응답이 DB 반영보다 먼저 나가는 지점을 보자.

![캐시에 먼저 기록하고 나중에 데이터베이스로 반영하는 Write Behind](/images/posts/cache-strategy/legacy-03.png "Write Behind")

```text
쓰기 요청
↓
Cache 반영
↓
성공 응답
↓
나중에 DB 반영
```

Cache에 먼저 기록하고 DB 반영을 나중으로 미루는 방식을 `Write Behind` 또는 `Write Back`이라고 한다.

Write Through와 나란히 놓으면 가장 큰 차이가 보인다.

```text
Write Through

요청 → DB 반영 → Cache 반영 → 성공 응답


Write Behind

요청 → Cache 반영 → 성공 응답 → 나중에 DB 반영
```

Write Behind의 핵심은 단순히 비동기 처리를 쓴다는 데 있지 않다. **사용자에게 성공했다고 응답하는 시점에도 DB 반영이 끝나지 않았을 수 있다.**

주문 상태 변경에 대입해보자.

```text
Cache에 COMPLETED 기록
↓
사용자에게 성공 응답
↓
DB에는 나중에 반영할 예정

그사이 장애
↓
DB에는 여전히 PROCESSING
```

사용자는 성공 응답을 받았지만 원본 DB에는 변경이 남지 않을 수 있다. 따라서 Write Behind를 검토할 때는 다음 질문도 함께 답해야 한다.

- 지연된 쓰기를 어디에 보관할 것인가?
- DB 반영에 실패하면 어떻게 재시도할 것인가?
- 같은 작업이 두 번 전달돼도 중복 반영되지 않게 할 수 있는가?
- 장애 후 아직 반영되지 않은 작업을 어떻게 찾아 복구할 것인가?

재고나 결제처럼 원본 정합성이 중요한 데이터에 별도의 내구성 장치 없이 적용하기는 어렵다. 다만 업무의 보장 수준과 복구 설계에 따라 선택 가능성은 달라지므로, 특정 도메인에서는 절대 사용할 수 없다고 단정할 문제도 아니다.

### 자주 읽히지 않는 값은 Cache를 우회할 수 있다

![쓰기 요청이 캐시를 우회해 데이터베이스로 향하는 Write Around](/images/posts/cache-strategy/legacy-04.png "Write Around")

`Write Around`는 쓰기 요청을 Cache에 넣지 않고 DB에만 반영한다.

```text
쓰기
→ DB만 반영
→ Cache에는 저장하지 않음

다음 조회
→ Cache Miss
→ DB 조회
→ Cache 적재
```

자주 읽지 않는 값까지 Cache에 채우는 일을 줄일 수 있다. 대신 변경 직후 첫 조회는 Miss를 만나므로 읽기 비용이 늘어난다. 다른 전략과 같은 비중으로 외우기보다, **쓰기 시점에는 Cache를 채우지 않을 수도 있다**는 정도로 이해하면 충분하다.

### 이름보다 두 가지 책임을 비교한다

전략 이름이 비슷해 헷갈릴 때는 다음 두 가지만 보면 된다.

1. 누가 Cache Miss 또는 쓰기 흐름을 관리하는가?
2. 언제 원본 DB에 반영되는가?

| 전략 | Miss 또는 쓰기를 관리하는 주체 | 원본 DB 반영 시점 | 주의할 점 |
|---|---|---|---|
| Cache Aside | 애플리케이션 | 애플리케이션이 DB를 직접 변경 | 무효화 누락과 반복 코드 |
| Read Through | Cache 계층과 Loader | 읽기 시 Loader가 원본 조회 | Provider의 Loader 계약에 의존 |
| Write Through | Cache 계층 또는 애플리케이션 | 성공 응답 전에 반영 | 쓰기 지연과 장애 전파 |
| Write Behind | Cache 계층 또는 별도 작업 | 성공 응답 뒤에 지연 반영 | 유실·중복 반영과 복구 |
| Write Around | 애플리케이션 | DB에 직접 반영 | 변경 직후 첫 조회에서 Miss |

실제 시스템은 하나의 전략 이름으로만 구성되지 않는다.

```text
조회
→ Cache Aside

갱신
→ DB 먼저 반영 + Cache 삭제

일부 인기 데이터
→ 만료 전에 미리 Cache 갱신
```

전략은 시스템 전체에 붙이는 상표가 아니라, 경로별 책임을 설명하는 도구에 가깝다. 이제 이 경로 중 Cache가 비거나 고장 났을 때 요청을 어떻게 이어갈지 살펴보자.

## 5. Cache가 비거나 장애가 나도 요청을 이어간다

Cache는 성능을 높이기 위해 추가한 계층이다. Cache 하나가 비었다는 이유로 사용자 요청 전체가 실패한다면 성능 최적화가 새로운 단일 장애 지점이 될 수 있다.

가게 정보를 많은 요청에 제공하는 시스템을 예로 들어보자. 서비스에 맞게 가공한 1차 Redis, 원본에 가까운 2차 Redis와 주 저장소가 있다면 조회는 다음처럼 내려갈 수 있다.

```text
1차 Redis Hit
→ 바로 응답

1차 Redis Miss
→ 2차 Redis 조회
→ 사용자에게 먼저 응답
→ 1차 Redis는 별도로 복구

2차 Redis도 조회 실패
→ 주 저장소 조회
```

여기서 중요한 것은 Redis를 두 개 두었다는 사실이 아니다. **각 단계가 실패했을 때 다음으로 어디에서 읽을지 정한 구조다.**

### 사용자 응답과 Cache 복구를 분리할 수 있다

1차 Cache Miss를 발견한 요청이 1차 Cache를 다시 채우는 작업까지 모두 기다려야 하는 것은 아니다.

```text
1차 Cache Miss
↓
2차 저장소에서 값 조회
↓
사용자에게 먼저 응답

동시에
↓
1차 Cache 복구
```

하위 저장소에서 얻은 값으로 먼저 응답하고 Cache 적재는 별도 작업으로 보낼 수 있다. 이렇게 하면 Cache 복구가 늦어져도 현재 사용자의 응답 시간에는 영향을 덜 준다.

반대로 데이터 갱신은 주 저장소에서 시작해 하위 Cache로 순차 전달할 수도 있다. 만료 후 첫 조회가 모든 Cache를 다시 채우게 하면 인기 Key가 비는 순간 사용자 응답이 느려질 수 있기 때문이다.

이 구조 자체를 그대로 따라야 한다는 의미는 아니다. 가져올 것은 다음 질문이다.

- 각 Cache에 어떤 형태의 데이터를 얼마나 오래 보관하는가?
- Miss나 장애가 발생하면 다음으로 어디에서 읽는가?
- 사용자 응답과 Cache 복구를 분리할 수 있는가?
- 복구가 계속 실패하면 어떻게 감지하는가?

이제까지는 Miss가 한 번 발생하는 상황을 생각했다. 같은 인기 Key에 수많은 요청이 들어오는 순간 동시에 Miss가 발생하면 문제의 크기가 달라진다.

## 6. 인기 Key의 동시 Miss로부터 원본을 보호한다

초당 1,000번 조회되는 상품 정보가 Cache에 있다고 해보자.

```text
평상시

Request 1    ┐
Request 2    ├→ Cache Hit → 응답
...          │
Request 1000 ┘
```

모든 요청이 Cache에서 끝나므로 DB는 이 조회를 직접 처리하지 않는다. 그런데 TTL이 끝나 Key가 사라지면 어떻게 될까?

```text
TTL 만료
↓
Request 1    → Miss → DB
Request 2    → Miss → DB
Request 3    → Miss → DB
...          → ...  → DB
Request 1000 → Miss → DB
```

아래 그림에서는 하나의 Cache Miss가 아니라 같은 값을 다시 만들려는 요청들이 한꺼번에 원본으로 향하는 부분을 확인해보자.

![동시에 발생한 Cache Miss가 원본 저장소로 몰리는 Cache Stampede](/images/posts/cache-strategy/legacy-05.png "Cache Stampede와 Thundering Herd")

하나의 인기 Cache가 만료되는 순간 많은 요청이 동시에 원본 저장소로 몰리는 현상을 `Cache Stampede`라고 한다. `Thundering Herd`라는 표현도 함께 쓰인다. 여기서는 두 용어의 세부 차이보다 **같은 인기 Key의 Miss가 동시에 몰려 DB 부하가 급증하는 상황이다.** 이 정도로 이해하면 된다.

문제는 Cache 하나가 사라졌다는 사실만이 아니다. 같은 값을 채우려는 1,000개의 요청이 각각 DB 조회와 계산을 반복한다는 점이다.

### 방법 1. 만료 시점을 분산한다

관련 Key가 모두 정확히 10분 뒤 만료되면 한 시점에 Miss가 몰릴 수 있다. 각 TTL에 작은 무작위 값을 더하면 만료 시점이 흩어진다.

```text
고정 TTL

Key A = 10분
Key B = 10분
Key C = 10분
→ 동시에 만료될 가능성


TTL 분산

Key A = 9분 42초
Key B = 10분 8초
Key C = 10분 21초
→ 만료 시점 분산
```

이처럼 만료 시점을 조금씩 다르게 만드는 방식을 `TTL Jitter`, 즉 TTL 분산이라고 한다. 구현이 비교적 단순하고 여러 Key가 동시에 만료되는 문제를 완화한다. 다만 하나의 인기 Key에 요청이 집중되는 상황까지 막지는 못한다.

### 방법 2. 한 요청만 원본을 조회하게 한다

같은 Key에서 Miss가 발생해도 한 요청만 DB로 보내고, 나머지 요청은 값이 채워질 때까지 기다리거나 다른 응답 경로를 사용하게 만들 수 있다.

```text
Request 1
→ Key 단위 Lock 획득
→ DB 조회
→ Cache 저장

Request 2 ~ 1000
→ 같은 Key의 갱신이 끝날 때까지 대기
→ 채워진 Cache 사용
```

여러 Application Instance가 같은 Cache를 사용한다면 Redis 기반 분산 Lock 등을 검토할 수 있다. 그러나 Lock 자체에도 획득 실패, 만료, 대기 시간과 장애 복구 문제가 생긴다.

```text
Stampede 발견
→ 무조건 분산 Lock
```

으로 접근하기보다 다음 순서가 낫다.

```text
TTL 분산만으로 충분한가?
↓
만료 전에 갱신할 수 있는가?
↓
잠시 이전 값을 보여줘도 되는가?
↓
그래도 한 요청만 갱신해야 하는가?
↓
Key 단위 Lock 검토
```

Lock은 실제로 동시 Miss가 병목이 되는 인기 Key에 제한해 사용하는 편이 낫다.

### 방법 3. 만료 전에 갱신한다

만료 시점을 기다리지 않고 백그라운드 작업이 주기적으로 인기 Key를 갱신할 수 있다.

```text
Cache 만료 전
→ Background 작업이 DB 조회
→ 새 값으로 Cache 교체
→ 사용자 요청은 계속 Hit
```

이를 `Background Refresh` 또는 사전 갱신으로 볼 수 있다. 만료 순간의 Miss를 줄이는 대신 어떤 Key를 언제 갱신할지 관리해야 하고, 조회되지 않는 값까지 계속 갱신하면 불필요한 비용이 생긴다.

### 방법 4. 갱신 중에는 이전 값을 잠시 제공한다

최신 값이 아니어도 잠시 사용할 수 있는 이전 Cache 값을 `Stale Value`라고 한다.

```text
논리적 만료 감지
↓
Request 1
→ 새 값 갱신 시작

나머지 요청
→ 이전 값을 잠시 응답

갱신 완료
→ 이후 요청부터 새 값 사용
```

사용자는 갱신을 기다리지 않지만 잠시 오래된 값을 볼 수 있다. HTTP 캐시의 `stale-while-revalidate`와 비슷한 생각이다. 최신성이 조금 늦어도 되는 데이터에서 응답 지연과 원본 부하를 함께 줄일 수 있다.

### 조금 더 나아가면 만료 전에 일부 요청이 갱신할 수 있다

고정된 백그라운드 작업 대신, 만료가 가까워질수록 일부 요청이 갱신을 먼저 맡게 만들 수도 있다. `Probabilistic Early Recomputation`은 확률을 이용한 조기 재계산 방식이다.

```text
Cache 값 조회
+ 남은 TTL 확인
+ 이전 원본 계산 시간 확인
↓
만료가 가깝고 계산 비용이 클수록
일부 요청이 조기 갱신자로 선택될 가능성 증가
↓
선택된 요청만 원본을 다시 계산하고 Cache 갱신
↓
나머지 요청은 기존 값 사용
```

목적은 확률 공식을 외우는 데 있지 않다. **모든 요청이 정확한 만료 순간에 원본으로 몰리지 않도록 갱신 시점을 만료 전 구간에 분산한다.** 이것이 핵심이다.

실제 구현에서는 Cache 값, 이전 계산 시간과 남은 TTL을 함께 읽어야 할 수 있다. 여러 값을 하나의 Redis 연산 안에서 안전하게 확인하기 위한 방법으로 Lua Script를 사용할 수 있다. Redis는 Script를 실행하는 동안 해당 Script의 연산을 원자적으로 처리하므로 중간에 다른 명령이 끼어들지 않는다.

다만 이 글의 중심은 Lua 문법이 아니다. Script가 오래 실행되면 Redis의 다른 작업도 기다릴 수 있으므로 짧게 유지해야 하며, 자세한 구현은 별도 주제로 다루는 편이 낫다.

Stampede에는 TTL 분산, Lock, 사전 갱신, 이전 값 제공 등 여러 방식으로 대응할 수 있다. 방식마다 응답 지연과 구현 복잡도가 다르지만, 이런 보호 장치도 캐시가 충분한 이득을 준다는 전제에서만 의미가 있다.

## 7. 복잡성을 감수할 데이터를 고른다

여기까지의 해결책은 모두 Cache가 충분한 이득을 준다는 전제에서 의미가 있다. 그렇다면 모든 데이터를 Cache에 올릴 가치가 있을까?

다음 조건이 많을수록 Cache가 효과를 내기 쉽다.

- 같은 Key가 반복해서 조회된다.
- 원본 조회나 계산 비용이 크다.
- 쓰기보다 읽기가 훨씬 많다.
- 짧은 시간의 오래된 값을 허용할 수 있다.
- Key 개수와 Value 크기를 예측할 수 있다.

반대로 다음 데이터는 신중하게 봐야 한다.

- 요청마다 조회 조건이 달라 같은 Key가 거의 재사용되지 않는다.
- 값이 자주 변경돼 무효화가 계속 발생한다.
- DB와 Cache가 잠시라도 달라서는 안 된다.
- Value가 커서 Redis 메모리와 네트워크 비용이 크다.

### 느린 API라고 바로 Redis부터 넣지 않는다

API가 느리면 먼저 실제 병목을 확인해야 한다.

```text
API 응답이 느림
↓
SQL과 실행 계획 확인

Query가 불필요하게 느림
→ SQL과 Index부터 개선

같은 비싼 조회가 반복됨
→ Cache 검토
```

잘못된 Join이나 빠진 Index 때문에 느린 조회라면 Cache가 원인을 해결하지 않는다. 첫 Miss, 만료 후 재적재와 장애 우회에서는 결국 같은 느린 Query를 다시 실행해야 한다. 따라서 캐시를 도입하기 전에 DB Query와 Index부터 확인한다.

### Hit Ratio 하나만 보고 판단하지 않는다

`Hit Ratio`는 전체 Cache 조회 중 Hit가 차지하는 비율이다.

```text
Hit Ratio = Cache Hit 수 / 전체 Cache 조회 수
```

Hit Ratio가 높으면 원본 조회를 많이 줄였다는 신호가 될 수 있다. 하지만 Value가 지나치게 크거나 갱신 비용이 높다면 전체적으로 좋은 설계라고 단정할 수 없다.

다음 지표를 함께 본다.

| 지표 | 확인할 내용 |
|---|---|
| Hit Ratio | 요청이 실제로 Cache에서 얼마나 끝나는가? |
| Miss Load Time | Miss 뒤 원본 조회와 적재에 얼마나 걸리는가? |
| Eviction | 메모리 부족으로 유효한 Key가 얼마나 밀려나는가? |
| Redis 메모리 | Key와 Value가 예상한 범위 안에서 증가하는가? |
| DB 부하 | Cache 도입 전보다 원본 조회 부하가 실제로 줄었는가? |

이제 전략 이름을 고르기 전에 실제 업무에서 던질 질문을 순서대로 정리해보자.

## 실제 설계에서 확인할 질문

```text
같은 데이터가 반복 조회되는가?
↓
NO
→ Cache 효과가 크지 않을 수 있음

YES
↓
원본 조회 비용이 큰가?
↓
NO
→ DB Query와 Index 개선만으로 충분한지 확인

YES
↓
Cache Miss를 누가 처리할까?
├→ Application이 직접 관리: Cache Aside
└→ Cache 계층에 위임: Read Through 검토

원본 데이터가 변경됨
↓
Cache를 어떻게 무효화하고 실패를 복구할까?
↓
허용 가능한 이전 값의 시간과 TTL은 얼마인가?

쓰기 성공 응답 전에 DB 반영이 끝나야 하는가?
├→ YES: Write Through 계열 검토
└→ NO: 지연 반영의 보존·재시도 장치가 있을 때 Write Behind 검토

Cache가 비거나 장애가 남
↓
하위 Cache나 원본으로 우회할 수 있는가?

인기 Key 만료 순간에 DB 부하가 몰림
↓
TTL 분산, 갱신 제어, 사전 갱신, 이전 값 제공 검토
```

이 순서는 절대적인 공식이 아니다. 캐시를 도입할 때 빠뜨리기 쉬운 책임과 실패 상황을 확인하는 질문 목록에 가깝다.

## 정리

- 캐시는 읽기를 빠르게 만들 수 있지만 그 순간부터 DB와 Cache라는 두 상태를 관리해야 한다.
- Cache Aside에서는 애플리케이션이 Miss와 Cache 적재를 직접 관리한다.
- Read Through에서는 Miss 확인과 Loader 실행 책임을 Cache 계층으로 옮긴다.
- Spring Cache는 Cache 저장소가 아니라 여러 구현체를 연결하는 추상화다.
- TTL은 DB와 Cache를 항상 같게 만드는 기능이 아니라, 이전 값이 남을 수 있는 최대 시간을 제한하는 장치다.
- Write Through는 성공 응답 전에 원본 반영을 끝내고, Write Behind는 성공 응답 뒤로 원본 반영을 미룰 수 있다.
- Cache Stampede의 핵심은 하나의 Key가 사라진 사실보다 같은 값을 채우려는 요청이 동시에 원본으로 향하는 데 있다.
- 캐시 도입 전에는 Query와 Index를 확인하고, 도입 후에는 Hit Ratio뿐 아니라 Miss 비용, 메모리와 DB 부하를 함께 관측한다.

좋은 캐시 설계는 `Cache Aside`나 `Write Through` 같은 전략 이름을 고르는 데서 끝나지 않는다. 원본과 Cache가 어긋날 수 있는 시간을 얼마나 허용할지, 중간 단계가 실패하면 어디에서 다시 읽을지, 인기 Key가 동시에 만료되면 어떻게 원본 저장소를 보호할지까지 함께 설계해야 한다.

## 참고 자료

### 공식 자료

- [Spring Framework - Cache Abstraction](https://docs.spring.io/spring-framework/reference/integration/cache.html)
- [Spring Framework - Understanding the Cache Abstraction](https://docs.spring.io/spring/reference/integration/cache/strategies.html)
- [Spring Framework - @Cacheable](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/cache/annotation/Cacheable.html)
- [Redis - Cache Aside](https://redis.io/docs/latest/develop/use-cases/cache-aside/)
- [Redis - EXPIRE](https://redis.io/docs/latest/commands/expire/)
- [Redis - Scripting with Lua](https://redis.io/docs/latest/develop/programmability/eval-intro/)
- [AWS - Database Caching Strategies Using Redis](https://docs.aws.amazon.com/whitepapers/latest/database-caching-strategies-using-redis/caching-patterns.html)

### 국내 기술 블로그

- [우아한형제들 - 배달의민족 최전방 시스템! 가게노출 시스템을 소개합니다](https://techblog.woowahan.com/2667/)
- [LINE Engineering - Atomic 처리와 Cache Stampede 대책을 위해 Redis Lua Script를 활용한 이야기](https://engineering.linecorp.com/ko/blog/atomic-cache-stampede-redis-lua-script/)
