+++
title = 'JDBC 커넥션은 왜 재사용하고 HikariCP는 어떻게 관리하는가'
date = 2026-04-23T20:00:00+09:00
lastmod = 2026-08-06T17:54:00+09:00
draft = false
description = 'JDBC 커넥션 생성 비용과 Connection Pool의 역할, HikariCP 설정을 처리량과 대기 시간 관점에서 설명합니다.'
categories = ['데이터 접근 설계']
tags = ['JDBC', 'HikariCP', 'Connection Pool', '데이터베이스']
+++

애플리케이션이 데이터베이스에 SQL을 보낸다는 말 뒤에는 네트워크 연결과 인증, 세션 생성 과정이 숨어 있다. 요청마다 이 과정을 새로 수행하면 쿼리가 빨라도 커넥션을 만드는 비용을 반복해서 지불하게 된다.

Connection Pool은 미리 만든 커넥션을 보관했다가 빌려주고, 사용이 끝나면 다시 회수한다. HikariCP는 Spring Boot에서 널리 사용되는 JDBC Connection Pool 구현체다.

## JDBC는 공통 인터페이스다

Java 애플리케이션은 JDBC API를 통해 데이터베이스와 통신한다. 실제 Protocol과 Socket 통신은 MySQL Connector/J 같은 JDBC Driver가 담당한다.

JDBC가 인터페이스라는 말은 애플리케이션이 MySQL의 통신 규칙을 직접 구현하지 않아도 된다는 뜻이다. Java 코드는 `Connection`, `PreparedStatement`, `ResultSet`이라는 공통 API를 사용하고, Driver가 이를 각 데이터베이스의 Protocol로 변환한다.

- `Connection`: 데이터베이스 세션과 연결된 통신 경로
- `PreparedStatement`: SQL과 Parameter를 전달하고 실행하는 객체
- `ResultSet`: 조회 결과 행을 순서대로 읽는 객체

그래서 같은 JDBC 코드를 사용하더라도 MySQL에는 Connector/J, PostgreSQL에는 PostgreSQL JDBC Driver가 필요하다.

데이터베이스도 네트워크 너머에 있는 하나의 서버다. 커넥션을 새로 만든다는 것은 Java 객체 하나를 생성하는 것보다 훨씬 많은 일을 포함할 수 있다.

```text
TCP 연결 수립
→ 필요하면 TLS 협상
→ 사용자 인증
→ 데이터베이스 세션 생성
→ 문자 집합과 세션 변수 같은 초기 상태 설정
```

짧은 `SELECT` 하나보다 연결 준비가 더 오래 걸릴 수도 있다. 요청마다 연결을 만들고 끊지 않고 이미 준비된 커넥션을 재사용하는 이유다.

```text
애플리케이션
→ JDBC API
→ JDBC Driver
→ TCP 연결
→ 데이터베이스 서버
```

일반적인 순서는 다음과 같다.

```java
try (Connection connection = dataSource.getConnection();
     PreparedStatement statement = connection.prepareStatement(sql)) {
    statement.setLong(1, memberId);
    return statement.executeQuery();
}
```

여기서 `close()`는 Pool을 사용할 때 실제 TCP 연결을 매번 끊는 동작이 아니다. Proxy Connection을 Pool에 반환해 다른 요청이 재사용할 수 있게 한다. 따라서 커넥션은 반드시 닫아야 한다. 반환하지 않으면 사용 가능한 커넥션이 줄어들어 결국 새 요청이 Pool에서 기다리게 된다.

Pool을 사용하지 않는다면 `DriverManager.getConnection()`으로 얻은 물리 커넥션의 `close()`는 실제 연결 종료로 이어진다. Pool을 사용하면 DataSource가 물리 커넥션을 감싼 객체를 빌려주기 때문에 같은 메서드의 의미가 “폐기”에서 “반납”으로 바뀐다. 호출자는 어느 경우든 사용이 끝나면 `close()`를 호출하면 되고, Pool 구현이 실제 연결을 유지할지 결정한다.

애플리케이션은 보통 HikariCP의 실제 커넥션을 직접 받지 않고 이를 감싼 Proxy를 받는다. Proxy의 `close()`는 아래처럼 동작한다.

```text
애플리케이션이 connection.close() 호출
→ 열려 있는 Statement와 상태 정리
→ autoCommit, readOnly 같은 변경 상태 복원
→ 실제 물리 커넥션을 HikariCP에 반환
→ 다음 요청이 같은 커넥션 재사용
```

이 때문에 `try-with-resources`가 중요하다. 정상 흐름뿐 아니라 예외가 발생한 경우에도 `close()`가 호출되어야 커넥션이 Pool로 돌아간다.

이 반환 주기를 이해하면 Pool 고갈도 단순히 “커넥션 수가 적다”는 문제가 아니라는 점이 보인다. 빌려간 커넥션이 모두 돌아오지 않거나 오래 점유되면 다음 요청은 대기할 수밖에 없다.

## Pool이 가득 차면 무슨 일이 생기는가

HikariCP의 `maximumPoolSize`는 유휴 커넥션과 사용 중인 커넥션을 합친 최대 크기다. 모든 커넥션이 사용 중이면 `getConnection()` 호출은 사용 가능한 커넥션이 돌아올 때까지 기다린다. `connectionTimeout`이 지나면 `SQLException`이 발생한다.

여기서 유휴 커넥션은 연결이 끊긴 커넥션이 아니라, 데이터베이스와 연결된 채로 현재 빌려간 요청이 없는 커넥션이다. Active는 요청이 빌려 사용 중인 수, Idle은 바로 빌려줄 수 있는 수, Pending은 커넥션을 기다리는 요청 수라고 이해하면 된다.

Pool이 아직 최대 크기에 도달하지 않았다면 HikariCP가 새 물리 커넥션을 만들 수 있다. 이미 최대 크기라면 기다리는 것 외에는 방법이 없다. 따라서 `getConnection()`이 느리다는 말은 SQL 실행 전부터 요청이 줄을 서고 있다는 뜻일 수 있다.

```text
요청 스레드 → 커넥션 대여 시도
              ├─ 유휴 커넥션 있음 → 즉시 반환
              └─ 없음 → connectionTimeout까지 대기
                         ├─ 커넥션 반환됨 → 사용
                         └─ 시간 초과 → 예외
```

Pool 크기를 키우면 대기가 무조건 줄어들 것 같지만 데이터베이스가 동시에 처리할 수 있는 CPU, I/O와 Lock 자원은 한정되어 있다. 너무 많은 커넥션은 Context Switching과 경합을 늘려 오히려 전체 지연을 키울 수 있다.

예를 들어 애플리케이션 스레드가 200개라고 해서 커넥션도 200개가 필요한 것은 아니다. 대부분의 요청이 JSON 변환이나 외부 API 호출을 하는 동안에는 DB 커넥션을 사용하지 않는다. 반대로 긴 트랜잭션 10개가 커넥션을 오래 잡고 있다면 Pool이 10개뿐인 경우 다른 요청은 전부 기다린다. 중요한 값은 웹 스레드 수가 아니라 **동시에 DB 작업을 수행하는 요청 수와 커넥션 점유 시간**이다.

따라서 Pool의 성능은 크기 하나로 설명할 수 없다. 커넥션을 얼마나 빌려주고, 얼마나 기다리며, 언제 교체할지를 하나의 수명 주기로 조정해야 한다.

## 주요 설정은 커넥션의 대여와 교체 주기를 결정한다

세 설정은 각각 다른 시점을 제어한다. `maximumPoolSize`는 동시에 빌려줄 수 있는 커넥션 수를 제한하고, `connectionTimeout`은 빌릴 커넥션이 없을 때 얼마나 기다릴지 정한다. `maxLifetime`은 반환된 커넥션을 언제 새 연결로 교체할지 결정한다. 따라서 한 값만 따로 키우기보다 커넥션을 빌리고, 사용하고, 돌려받아 교체하는 전체 주기로 봐야 한다.

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      connection-timeout: 30000
      max-lifetime: 1740000
```

### maximumPoolSize

데이터베이스에 동시에 열 수 있는 실제 커넥션의 상한이다. 웹 스레드 수와 같게 맞추는 값이 아니다. 한 요청이 커넥션을 점유하는 시간, 데이터베이스의 처리 능력, 같은 DB를 사용하는 다른 애플리케이션까지 함께 측정해야 한다.

### connectionTimeout

Pool에 남는 커넥션이 없을 때 호출자가 기다릴 최대 시간이다. 지나치게 길면 장애가 요청 스레드 적체로 번지고, 지나치게 짧으면 순간적인 부하에도 실패가 늘어난다.

### maxLifetime

Pool 안의 커넥션을 교체하기 전 최대 수명이다. 사용 중인 커넥션을 강제로 끊지는 않고 반환된 뒤 제거한다. HikariCP는 데이터베이스나 네트워크 장비가 연결을 끊는 제한 시간보다 몇 초 짧게 설정할 것을 권장한다.

`maxLifetime`을 짧게 잡는다고 항상 더 안전한 것은 아니다. 커넥션을 더 자주 교체하면 새 연결을 만드는 횟수와 폐기할 연결 수도 함께 늘어난다.

`maxLifetime`과 `connectionTimeout`은 이름에 시간 단위가 들어가지만 전혀 다른 상황을 다룬다. `connectionTimeout`은 요청이 Pool에서 커넥션을 **빌리기 위해 기다리는 시간**이고, `maxLifetime`은 물리 커넥션 하나를 **Pool이 재사용할 수 있는 전체 수명**이다.

한 운영 사례에서는 데이터베이스 장애 전환 뒤 새 대상에 빠르게 연결하기 위해 `maxLifetime`을 50초로 설정했다. HikariCP 기본값인 30분보다 훨씬 자주 커넥션이 교체되는 조건이었다. 그런데 MySQL Connector/J는 폐기된 Connection의 네트워크 자원을 `AbandonedConnectionCleanupThread`라는 단일 스레드에서 하나씩 정리하고 있었다.

```text
짧은 maxLifetime
→ Connection 생성·폐기 빈도 증가
→ Driver의 단일 Cleanup Thread가 폐기 속도를 따라가지 못함
→ PhantomReference 대기열과 Connection 참조 증가
→ Heap 점유 증가와 OOM
```

겉으로는 Connection Leak처럼 보였지만 애플리케이션이 `close()`를 빠뜨린 문제가 아니었다. Heap Dump에서 `connectionFinalizerPhantomRefs`와 `AbandonedConnectionCleanupThread`의 참조 경로를 확인한 뒤에야 Driver 내부 정리 병목임을 알 수 있었다. 팀은 짧은 수명이라는 운영 목적은 유지하면서 지원되는 Connector/J 버전으로 올리고 Cleanup Thread 비활성화 옵션을 적용했다.

이 사례는 특정 옵션을 그대로 사용하라는 뜻이 아니다. Pool 설정은 HikariCP 안에서 끝나지 않고 JDBC Driver의 정리 방식, 네트워크 장비의 Idle Timeout과 데이터베이스 장애 전환 정책까지 하나의 연결 수명 주기로 이어진다는 점을 보여준다.

### minimumIdle과 idleTimeout

`minimumIdle`은 유지하려는 최소 유휴 커넥션 수다. HikariCP는 별도 요구가 없다면 `minimumIdle`을 지정하지 않고 고정 크기 Pool처럼 사용하는 구성을 권장한다. `idleTimeout`은 `minimumIdle`이 `maximumPoolSize`보다 작을 때만 유휴 커넥션 정리에 영향을 준다.

설정의 의미를 알아도 증상만 보고 값을 키우면 원인을 가릴 수 있다. Pending이 늘어난 이유가 작은 Pool인지, 느린 Query인지, 반환 누락인지부터 분리해야 한다.

## 고갈의 원인은 Pool 크기만이 아니다

커넥션 대기 시간이 늘었다면 먼저 다음을 확인한다.

- 반환하지 않은 Connection, Statement와 ResultSet이 있는가
- 느린 SQL이나 Lock 대기로 커넥션을 오래 점유하는가
- 외부 API 호출을 DB Transaction 안에서 기다리는가
- 요청량이 데이터베이스의 처리량을 장기간 넘어섰는가
- 여러 애플리케이션의 Pool 합계가 DB 허용량을 넘는가

`leakDetectionThreshold`는 커넥션을 일정 시간 이상 반환하지 않은 호출 위치를 찾는 보조 수단이다. 오래 걸리는 정상 쿼리도 Leak처럼 보일 수 있으므로 상시 정답이라기보다 진단 목적으로 사용한다.

Pool 지표에 뚜렷한 대기나 Timeout이 없어도 Driver 내부 참조가 쌓일 수 있다. 메모리 사용량이 함께 증가한다면 Heap Dump에서 커넥션 객체의 수와 참조 경로까지 확인해야 한다.

Pool 지표에서는 최소한 Active, Idle, Pending, Timeout을 함께 본다. Pending이 늘면서 Active가 계속 최대치라면 커넥션 점유 시간이 길거나 처리량이 부족한 것이다. 단순히 Pool을 키우기 전에 SQL과 트랜잭션 경계를 먼저 확인해야 한다.

## 정리

- JDBC Driver가 Java API와 데이터베이스 Protocol 사이를 연결한다.
- HikariCP는 생성 비용이 큰 커넥션을 재사용한다.
- `close()`는 Pool 환경에서 커넥션 반환을 의미하므로 반드시 호출해야 한다.
- Pool이 클수록 항상 빠른 것은 아니며 데이터베이스 처리 능력이 상한을 결정한다.
- 대기 증가를 발견하면 Pool 크기보다 커넥션 점유 시간과 Leak부터 확인한다.

## 참고 자료

### 공식 자료

- [HikariCP 공식 문서](https://github.com/brettwooldridge/HikariCP)
- [HikariCP - About Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)

### 국내 기술 블로그

- [컬리 기술 블로그 - 99%가 모른다는 DB Connection 누수 문제](https://helloworld.kurly.com/blog/connection-leak/)
