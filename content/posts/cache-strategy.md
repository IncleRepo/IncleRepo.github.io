+++
title = '캐시 전략은 읽기 성능과 데이터 정합성을 어떻게 맞추는가'
date = 2026-04-10T19:00:00+09:00
lastmod = 2026-08-07T16:49:40+09:00
draft = false
description = 'Cache Aside, Read Through, Write Through와 Write Behind의 흐름을 비교하고 무효화와 동시 요청 문제를 정리합니다.'
categories = ['데이터 접근 설계']
tags = ['캐시', 'Redis', 'Cache Aside', '정합성']
+++

캐시를 도입하면 읽기가 빨라질 수 있다. 대신 원본 데이터와 캐시라는 두 상태가 생긴다. 이제 애플리케이션은 “어디서 읽을까?”뿐 아니라 “변경됐을 때 무엇을 먼저 고칠까?”도 결정해야 한다.

캐시 전략은 제품 이름이 아니라 읽기와 쓰기의 책임을 어디에 둘지 정하는 방법이다. 같은 Redis를 사용해도 Cache Aside와 Write Through는 장애 상황에서 전혀 다르게 동작한다.

먼저 두 용어를 짚고 가자. 요청한 값이 캐시에 있으면 Cache Hit, 없으면 Cache Miss다. Hit에서는 원본 저장소를 방문하지 않아 이득을 얻고, Miss에서는 캐시와 원본을 모두 조회한 뒤 다음 요청을 위해 값을 채운다. 따라서 캐시는 조회마다 항상 빠른 장치가 아니라 **반복되는 요청에서 이전 결과를 재사용하는 장치**다.

## Cache Aside는 애플리케이션이 흐름을 관리한다

![Cache Miss일 때 원본 저장소를 조회하고 캐시에 적재하는 흐름](/images/posts/cache-strategy/legacy-01.png "Cache Aside")

가장 흔한 읽기 패턴은 Cache Aside다.

```text
1. 캐시 조회
2. Hit면 바로 반환
3. Miss면 DB 조회
4. 조회 결과를 캐시에 저장
5. 반환
```

```java
public PlantSummary getSummary(long plantId) {
    String key = "plant:summary:" + plantId;
    PlantSummary cached = cache.get(key, PlantSummary.class);

    if (cached != null) {
        return cached;
    }

    PlantSummary loaded = repository.findSummary(plantId);
    cache.put(key, loaded, Duration.ofMinutes(10));
    return loaded;
}
```

필요한 데이터만 캐시에 올라가고 Cache 장애가 나도 DB를 직접 읽는 우회 경로를 만들기 쉽다. 반면 Miss 처리와 무효화 책임이 애플리케이션 코드에 생긴다.

처음 요청과 두 번째 요청을 나누어 보면 이 흐름이 더 분명하다.

```text
첫 요청
캐시 Miss → DB 조회 → 캐시 저장 → 응답

두 번째 요청
캐시 Hit → DB를 거치지 않고 응답
```

첫 요청은 오히려 Redis와 DB를 모두 방문하므로 더 많은 일을 한다. 같은 Key가 다시 조회되어야 앞에서 지불한 적재 비용을 회수할 수 있다.

데이터가 변경될 때는 보통 DB를 먼저 Commit한 뒤 캐시를 제거한다.

```text
DB 변경 성공 → 캐시 삭제
```

캐시에 새 값을 바로 쓰는 것보다 삭제가 단순한 이유는 다음 조회가 DB에서 최신 값을 다시 채울 수 있기 때문이다. 하지만 DB Commit 뒤 캐시 삭제 전에 프로세스가 종료되면 오래된 값이 남을 수 있다. TTL은 이런 실패가 영구화되지 않게 하는 마지막 안전장치이지 즉시 정합성을 보장하는 장치는 아니다.

순서를 반대로 잡는다고 문제가 사라지는 것도 아니다.

```text
캐시를 먼저 삭제
→ 삭제 직후 다른 요청이 DB의 이전 값을 조회
→ 이전 값을 캐시에 다시 저장
→ 원래 요청이 DB 변경 Commit
```

결과적으로 DB는 새 값인데 캐시에는 이전 값이 다시 들어갈 수 있다. 캐시와 DB는 하나의 트랜잭션으로 자연스럽게 묶이지 않으므로 “삭제를 먼저 할까, 나중에 할까?”만으로 완전한 정합성을 만들기 어렵다. 데이터가 얼마나 오래 어긋나도 되는지, 실패를 어떻게 감지하고 복구할지까지 함께 정해야 한다.

Cache Aside는 이 읽기 흐름을 애플리케이션 코드가 직접 드러낸다. 같은 책임을 캐시 계층 뒤로 숨기면 호출 코드는 단순해지지만, 실제 Miss 처리와 저장소 동작은 Provider의 계약에 의존하게 된다.

## Read Through는 읽기 책임을 캐시 계층에 숨긴다

Read Through에서는 애플리케이션이 캐시에 값을 요청하고, Miss가 발생하면 캐시 계층이나 Provider가 Loader를 호출해 원본에서 값을 가져온다.

```text
애플리케이션 → 캐시 계층 → Miss → Loader → DB
```

Spring의 `@Cacheable`도 호출 결과를 캐시하는 관점에서는 비슷한 사용 경험을 준다.

```java
@Cacheable(cacheNames = "plantSummary", key = "#plantId")
public PlantSummary getSummary(long plantId) {
    return repository.findSummary(plantId);
}
```

다만 Spring Cache는 캐시 저장소 자체가 아니라 추상화다. 실제 저장과 TTL, 분산 환경 동작은 Caffeine, Redis 같은 `CacheManager` 구현에 따라 달라진다.

Cache Aside 예제에서는 Service 코드가 `cache.get()`과 `repository.findSummary()`를 순서대로 호출했다. Read Through에서는 호출자가 캐시만 바라보고, Miss 때 원본을 읽는 규칙을 캐시 계층이 알고 있다는 점이 다르다.

```text
Cache Aside
Service가 캐시 조회와 DB 조회 순서를 직접 제어

Read Through
Service는 캐시 계층만 호출
캐시 계층이 Miss 때 Loader를 실행
```

`@Cacheable`은 메서드 실행 자체를 Loader처럼 사용할 수 있어 호출 코드에서는 Read Through와 비슷하게 보인다. 다만 캐시 Provider가 데이터베이스를 직접 아는 전통적인 Read Through와 완전히 같은 구현 구조라는 뜻은 아니다. 어느 계층이 Miss 처리 책임을 갖는지 구분하는 데 초점을 맞추면 된다.

지금까지는 값을 어떻게 읽을지 살펴봤다. 원본이 변경될 때 캐시까지 언제 반영할지는 별도의 쓰기 전략이며, 이 선택은 응답 시간과 데이터 유실 가능성을 바꾼다.

## Write Through와 Write Behind

![캐시와 데이터베이스에 함께 기록하는 Write Through](/images/posts/cache-strategy/legacy-02.png "Write Through")

![캐시에 먼저 기록하고 나중에 데이터베이스로 반영하는 Write Behind](/images/posts/cache-strategy/legacy-03.png "Write Behind")

Write Through는 쓰기 요청이 캐시 계층을 통과하면서 원본 저장소까지 동기적으로 반영되는 구조다.

```text
애플리케이션 → 캐시 계층 → DB 반영 → 캐시 반영 → 완료
```

읽을 때 최신 값이 캐시에 있을 가능성이 높지만 쓰기 경로가 느려지고 캐시 계층이 원본 저장 책임까지 알아야 한다. “Redis를 사용하면 자동으로 Write Through가 된다”는 뜻은 아니다. 이 흐름을 지원하는 제품이나 애플리케이션 구조가 필요하다.

Write Behind는 캐시에 먼저 쓰고 DB 반영을 뒤로 미룬다.

```text
애플리케이션 → 캐시 반영 → 완료
                         └→ 나중에 DB 반영
```

쓰기 응답은 빨라질 수 있지만 반영 전 장애에서 데이터를 잃을 수 있다. 재고, 결제처럼 원본 정합성이 중요한 업무에는 별도 내구성 장치 없이 적용하기 어렵다.

두 전략을 주문 상태 변경에 대입하면 차이가 더 분명하다.

```text
Write Through
주문 상태 변경 요청
→ DB와 캐시 반영 완료
→ 사용자에게 성공 응답

Write Behind
주문 상태 변경 요청
→ 캐시에 먼저 반영
→ 사용자에게 성공 응답
→ 나중에 DB 반영
```

Write Behind에서는 사용자가 성공 응답을 받은 뒤에도 DB에는 아직 이전 값이 남아 있을 수 있다. 그사이 캐시나 전달 작업이 사라지면 성공했다고 응답한 변경을 복원할 수 없다. 따라서 단순히 “쓰기 성능이 빠르다”가 아니라 지연 반영을 보존하고 재시도할 장치가 있는지를 먼저 확인해야 한다.

Write Around는 쓰기 요청을 캐시에 넣지 않고 DB에만 반영한다. 자주 읽지 않는 데이터로 캐시가 오염되는 것을 줄일 수 있지만, 변경 직후 처음 조회할 때는 Cache Miss가 발생한다.

![쓰기 요청이 캐시를 우회해 데이터베이스로 향하는 Write Around](/images/posts/cache-strategy/legacy-04.png "Write Around")

이름이 비슷하므로 책임을 표로 다시 묶어보자.

| 전략 | Miss 또는 쓰기를 처리하는 주체 | 원본 반영 시점 | 주의할 점 |
|---|---|---|---|
| Cache Aside | 애플리케이션 | DB를 직접 변경 | 무효화 누락과 중복 코드 |
| Read Through | 캐시 계층과 Loader | 읽기 시 원본 조회 | Provider 동작에 대한 의존 |
| Write Through | 캐시 계층 | 응답 전에 동기 반영 | 쓰기 지연과 장애 전파 |
| Write Behind | 캐시 계층 또는 별도 작업 | 응답 뒤 지연 반영 | 유실·중복 반영과 복구 |

이처럼 각 전략에는 분명한 장단점이 있지만 실제 장애는 전략 이름대로만 일어나지 않는다. Cache가 비거나 일부 계층이 실패했을 때도 요청을 계속 처리할 수 있는 경로가 있는지가 더 직접적인 운영 문제다.

## 캐시가 비어도 요청을 처리할 수 있어야 한다

실제 시스템은 한 가지 전략만 고집하기보다 갱신 경로와 조회 경로를 따로 설계한다. 중요한 질문은 “어떤 전략 이름을 썼는가?”가 아니라 **캐시가 비거나 고장 났을 때 어느 저장소가 요청을 이어받는가?**다.

가게 정보를 초당 많은 요청에 제공하는 시스템을 생각해보자. 서비스용으로 가공한 1차 Redis, 원본에 가까운 2차 Redis와 주 저장소가 있다면 조회는 다음처럼 내려갈 수 있다.

```text
1차 Redis Hit  → 바로 응답
1차 Redis Miss → 2차 Redis 조회 → 먼저 응답 → 1차 Redis는 비동기 복구
2차 Redis 실패 → 주 저장소 조회
```

이때 Miss를 만난 사용자 요청이 1차 Cache 복구까지 기다릴 필요는 없다. 하위 저장소에서 얻은 값으로 먼저 응답하고 Cache 적재는 뒤로 보낼 수 있다. 반대로 데이터 갱신은 주 저장소에서 시작해 2차 Cache, 1차 Cache 순서로 전달한다. 만료될 때까지 기다렸다가 첫 조회가 모든 Cache를 채우게 하면 인기 Key가 만료되는 순간 응답이 느려질 수 있기 때문이다.

여기서 그대로 복사할 것은 저장소 개수가 아니다. 원본과 Cache의 생명주기를 구분하고, 각 단계가 실패했을 때 다음 조회 경로와 복구 시점을 미리 정하는 사고방식이다.

## 동시에 Miss가 발생하면 생기는 문제

![동시에 발생한 Cache Miss가 원본 저장소로 몰리는 Cache Stampede](/images/posts/cache-strategy/legacy-05.png "Cache Stampede와 Thundering Herd")

인기 Key가 만료되는 순간 여러 요청이 동시에 들어오면 모두 DB를 조회할 수 있다. 이를 Cache Stampede 또는 Thundering Herd라고 부른다.

예를 들어 초당 1,000번 조회되는 상품 정보가 정확히 같은 시각에 만료됐다고 해보자. 첫 요청 하나가 DB에서 값을 다시 채우는 동안 나머지 999개 요청도 모두 Miss를 확인할 수 있다. 평소에는 Redis가 막아주던 부하가 만료 순간에 한꺼번에 DB로 이동한다. 문제는 Miss 한 번이 아니라 **같은 값을 채우려는 요청이 동시에 몰리는 것**이다.

완화 방법은 여러 가지다.

- TTL에 작은 무작위 값을 더해 만료 시점을 분산
- 한 요청만 값을 적재하도록 Key 단위 Lock 사용
- 만료 전에 백그라운드에서 갱신
- 오래된 값을 잠시 제공하면서 한 요청만 재검증

분산 Lock은 비용과 실패 모드를 추가한다. 모든 Key에 바로 적용하기보다 실제로 동시 Miss가 병목이 되는 인기 Key에 제한해서 사용하는 편이 낫다.

Lock을 잡지 않고 만료 시점을 흩뜨리는 방법도 있다. Probabilistic Early Recomputation은 Cache가 완전히 만료될 때까지 기다리지 않고, 일부 요청이 확률적으로 갱신을 먼저 맡게 한다.

```text
Cache 값 + 남은 TTL + 이전 계산 시간 조회
→ 계산이 오래 걸렸고 만료가 가까울수록 조기 갱신 확률 증가
→ 선택된 요청만 원본을 다시 계산하고 Cache 갱신
→ 나머지 요청은 기존 값 사용
```

실제 값과 이전 계산 시간을 함께 보관하고 Redis Lua Script로 남은 TTL까지 원자적으로 읽으면 이 판단을 하나의 Cache 연산으로 묶을 수 있다. 만료 순간 모든 요청이 원본으로 떨어지는 대신 갱신 시점을 만료 전 구간에 분산하는 것이다. 특정 환경의 성능 테스트에서 응답 시간이 개선됐더라도 그 수치를 다른 서비스에 그대로 적용할 수는 없다.

여기서 기억할 점은 특정 확률식이 아니다. Stampede가 확인됐을 때 Lock만 떠올리지 말고 TTL 분산, 조기 갱신, 오래된 값 제공처럼 **사용자 요청을 기다리게 하는 정도와 구현 복잡도가 다른 대안**을 함께 비교해야 한다.

다만 이런 대책은 캐시에 올릴 가치가 있는 데이터라는 전제에서만 의미가 있다. Hit가 거의 없거나 즉시 정합성이 필요한 값까지 복잡하게 보호하면 캐시가 줄이는 비용보다 운영 비용이 더 커질 수 있다.

## 무엇을 캐시해야 하는가

다음 조건이 많을수록 캐시 효과가 크다.

- 같은 Key가 반복해서 조회된다.
- 원본 조회 비용이 충분히 크다.
- 데이터가 읽기보다 덜 자주 변경된다.
- 짧은 시간의 오래된 값을 허용할 수 있다.
- Key 수와 Value 크기를 예측할 수 있다.

반대로 매번 다른 조건으로 조회하거나 값이 계속 바뀌고 즉시 정합성이 필요하다면 Hit Ratio가 낮거나 무효화 비용이 더 커질 수 있다.

캐시를 도입하기 전에는 DB Query와 Index부터 확인하고, 도입한 뒤에는 Hit Ratio, Miss Load Time, Eviction, 메모리 사용량과 원본 DB 부하를 함께 관측해야 한다.

## 정리

- 캐시는 읽기 성능을 얻는 대신 원본과 캐시의 정합성 문제를 만든다.
- Cache Aside는 단순하고 유연하지만 Miss와 무효화를 애플리케이션이 책임진다.
- Spring Cache는 저장소가 아니라 여러 캐시 구현을 연결하는 추상화다.
- TTL은 즉시 정합성보다 오래된 값의 최대 수명을 제한하는 장치다.
- Stampede 방지는 실제 인기 Key와 부하를 확인한 뒤 필요한 범위에 적용한다.

## 참고 자료

### 공식 자료

- [Spring Framework - Cache Abstraction](https://docs.spring.io/spring-framework/reference/integration/cache.html)
- [Spring Framework - @Cacheable](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/cache/annotation/Cacheable.html)

### 국내 기술 블로그

- [우아한형제들 - 배달의민족 최전방 시스템! 가게노출 시스템을 소개합니다](https://techblog.woowahan.com/2667/)
- [LINE Engineering - Atomic 처리와 cache stampede 대책을 위해 Redis Lua script를 활용한 이야기](https://engineering.linecorp.com/ko/blog/atomic-cache-stampede-redis-lua-script/)
