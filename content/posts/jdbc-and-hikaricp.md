+++
title = 'JDBC 커넥션은 왜 재사용하고 HikariCP는 어떻게 관리하는가'
slug = '9'
aliases = ['/posts/009/']
date = 2026-04-23T20:00:00+09:00
lastmod = 2026-08-10T17:12:37+09:00
draft = false
description = 'SQL이 JDBC를 거쳐 데이터베이스에 도달하는 과정부터 Connection Pool의 대여·반납·대기·교체와 HikariCP 병목 진단까지 설명합니다.'
categories = ['데이터 접근 설계']
tags = ['JDBC', 'HikariCP', 'Connection Pool', '데이터베이스']
+++

Spring Data JPA를 사용하면 다음과 같은 코드만으로 데이터를 조회할 수 있다.

```java
memberRepository.findById(memberId);
```

코드에서는 메서드 하나를 호출했지만, 데이터베이스는 다른 프로세스에서 실행되고 있다. 그렇다면 JPA가 만든 SQL은 어떻게 MySQL 서버까지 전달될까?

```text
Java Application
→ JPA 또는 JDBC 코드
→ JDBC API
→ JDBC Driver
→ Network
→ Database
```

SQL을 보내려면 이 경로를 따라갈 수 있는 데이터베이스 연결이 먼저 필요하다. 이 연결이 JDBC `Connection`이다. 그런데 Connection을 만드는 일에는 네트워크 연결과 인증이 포함될 수 있어 Java 객체 하나를 만드는 것보다 비싸다.

Connection Pool은 이 비용을 매 요청마다 반복하지 않도록 이미 만든 Connection을 재사용한다. 하지만 재사용할 수 있는 수는 한정되어 있으므로, Pool을 이해하려면 생성뿐 아니라 **대여·사용·반납·대기·교체**까지 전체 생명주기를 함께 봐야 한다.

## Java의 SQL은 JDBC를 거쳐 데이터베이스로 간다

MySQL과 PostgreSQL은 통신 규칙과 Driver가 다르다. 그렇다고 Java 애플리케이션이 데이터베이스마다 완전히 다른 방식으로 SQL을 실행할 필요는 없다.

Java는 `Connection`, `PreparedStatement`, `ResultSet`과 같은 공통 API를 제공한다. 실제 데이터베이스 Protocol과 Socket 통신은 MySQL Connector/J나 PostgreSQL JDBC Driver 같은 구현체가 맡는다. 이 공통 규격이 JDBC다.

![애플리케이션에서 JDBC Driver를 거쳐 데이터베이스로 연결되는 구조](/images/posts/jdbc-and-hikaricp/legacy-01.png "JDBC 연결 구조")

![JDBC API와 Driver 구현의 관계](/images/posts/jdbc-and-hikaricp/legacy-02.png "JDBC의 인터페이스 기반 구조")

이를 역할로 나누면 다음과 같다.

```text
애플리케이션
→ 공통 JDBC API 사용

데이터베이스별 차이
→ JDBC Driver가 처리
```

JPA도 이 구조를 건너뛰지 않는다. Hibernate가 Entity 작업을 SQL로 바꾸더라도 실제 데이터베이스 통신에는 결국 JDBC Driver와 Connection이 필요하다.

### Connection부터 ResultSet까지 한 흐름으로 보기

세 객체를 따로 외우기보다 조회 한 번의 흐름으로 보면 역할이 분명해진다.

```text
Connection
→ 데이터베이스와 통신할 연결 확보

PreparedStatement
→ 그 연결을 통해 SQL과 Parameter 전달

ResultSet
→ 데이터베이스가 돌려준 조회 결과를 읽음
```

JDBC 코드로 표현하면 다음과 같다.

```java
String sql = "SELECT id, name FROM member WHERE id = ?";

try (Connection connection = dataSource.getConnection();
     PreparedStatement statement = connection.prepareStatement(sql)) {

    statement.setLong(1, memberId);

    try (ResultSet resultSet = statement.executeQuery()) {
        if (resultSet.next()) {
            long id = resultSet.getLong("id");
            String name = resultSet.getString("name");
        }
    }
}
```

`PreparedStatement`는 Connection이 있어야 만들 수 있고, 실행 결과는 `ResultSet`으로 읽는다. 따라서 SQL 실행의 출발점은 데이터베이스와 이어진 Connection을 확보하는 일이다.

## Connection 생성은 Java 객체 생성보다 비싸다

코드에서 `Connection`은 하나의 Java 객체로 보인다. 하지만 이 객체는 단순히 메모리에 만들어진 값이 아니라 데이터베이스 Session과 연결된 통신 경로를 나타낸다.

일반 객체는 다음처럼 애플리케이션 안에서 생성할 수 있다.

```java
Member member = new Member();
```

반면 새로운 물리 Connection을 만드는 과정에는 환경과 설정에 따라 다음 작업이 포함될 수 있다.

```text
TCP 연결 수립
→ 필요한 경우 TLS 협상
→ 사용자 인증
→ 데이터베이스 Session 생성
→ 문자 집합과 Session 상태 초기화
```

이 과정에는 애플리케이션과 데이터베이스 사이의 네트워크 왕복이 필요하다. 짧은 `SELECT` 하나는 몇 ms 안에 끝나는데, 매 요청마다 연결과 인증부터 다시 시작한다면 SQL보다 연결 준비에 더 큰 비용을 낼 수도 있다.

### 요청마다 만들고 끊는다면

Connection Pool이 없다면 요청은 다음 과정을 반복한다.

```text
요청 1
→ 물리 Connection 생성
→ SQL 실행
→ Connection 종료

요청 2
→ 물리 Connection 생성
→ SQL 실행
→ Connection 종료

요청 3
→ 물리 Connection 생성
→ ...
```

이미 만든 Connection을 다음 요청이 다시 쓸 수 있다면 이 비용을 줄일 수 있다.

```text
애플리케이션 시작
→ Connection 여러 개 준비

요청 1
→ Connection 대여
→ SQL 실행
→ 반납

요청 2
→ 반납된 Connection 재사용
```

이처럼 물리 Connection을 보관하고 빌려주며 다시 회수하는 구성 요소가 Connection Pool이다. Connection Pool은 개념이고, HikariCP는 이 개념을 구현한 JDBC Connection Pool이다. Spring Boot에서 JDBC 또는 JPA Starter를 사용하면 일반적으로 HikariCP가 함께 구성된다.

## Pool에서는 close가 종료가 아니라 반납이다

Connection을 재사용한다면 다음 코드가 이상하게 보일 수 있다.

```java
try (Connection connection = dataSource.getConnection()) {
    // 데이터베이스 작업
}
```

`try` 블록을 벗어나면 `connection.close()`가 호출된다. Pool을 쓴다면서 왜 매번 Connection을 닫는 걸까?

같은 `close()`라도 Connection을 얻은 방식에 따라 뒤에서 일어나는 일이 다르다.

```text
Pool 없이 직접 연결

DriverManager.getConnection()
→ 물리 Connection 사용
→ close()
→ 실제 연결 종료
```

Pool을 사용하면 애플리케이션은 보통 `DataSource`를 통해 Connection을 얻는다.

```text
Pool 사용

DataSource.getConnection()
→ Pool이 감싼 Connection 전달
→ close()
→ 실제 TCP 연결 종료 X
→ Pool에 반납
```

애플리케이션이 `close()`를 호출해도 실제 연결을 끊지 않고 Pool로 돌려보내려면, Pool은 원본 Connection을 감싼 객체를 제공할 수 있다. 이런 객체를 Proxy Connection이라고 한다.

호출자는 일반 Connection과 같은 API를 사용하지만 `close()`의 동작은 HikariCP가 가로챈다. 그래서 애플리케이션은 Pool의 내부 구현을 몰라도 Connection을 빌리고 반납할 수 있다.

### 반납 전에 상태도 정리한다

하나의 물리 Connection을 여러 요청이 차례로 사용하려면 이전 요청의 상태가 다음 요청에 남아서는 안 된다.

```text
요청 A
→ connection.setReadOnly(true)
→ 사용 후 반납

요청 B
→ 같은 물리 Connection 재사용
```

A가 바꾼 `readOnly` 상태가 그대로 남아 있다면 B의 작업에 영향을 줄 수 있다. HikariCP의 Proxy Connection은 반납 과정에서 열려 있는 Statement를 정리하고, 추적한 `autoCommit`, `readOnly`, Transaction 격리 수준 같은 변경 상태를 필요한 경우 원래 값으로 되돌린다.

```text
애플리케이션이 connection.close() 호출
→ 열려 있는 Statement 정리
→ 변경된 Connection 상태 복원
→ 물리 Connection을 HikariCP에 반환
→ 다음 요청이 재사용
```

Pool은 Connection을 보관만 하는 창고가 아니다. 다음 요청이 안전하게 재사용할 수 있도록 대여와 반납 경계를 관리한다.

### try-with-resources가 반납을 보장한다

Pool 환경에서 `close()`는 Connection 폐기가 아니라 반납이므로 오히려 반드시 호출해야 한다.

```text
Connection 대여
→ SQL 실행 중 Exception
→ close() 누락
→ Pool에 반환되지 않음
```

이 상황이 반복되면 Pool 크기가 10이어도 실제로 빌릴 수 있는 Connection은 9개, 8개로 줄어든다. 빌린 Connection이 정상적으로 Pool로 돌아오지 않는 상황을 흔히 Connection Leak이라고 부른다.

```text
Pool Size = 10

10개 요청이 Connection 대여
→ 그중 2개가 반환되지 않음
→ 이후 실제로 사용할 수 있는 Connection은 8개
```

`try-with-resources`를 사용하면 정상 종료뿐 아니라 예외가 발생한 경우에도 자동으로 `close()`가 호출된다. `Connection`, `PreparedStatement`, `ResultSet`처럼 닫아야 하는 JDBC 자원에 이 문법을 사용하는 이유다.

다만 Connection 객체가 많이 쌓였다고 해서 항상 애플리케이션이 `close()`를 빠뜨린 것은 아니다. 뒤에서 살펴볼 Driver 내부 정리 지연처럼 Pool Leak과 비슷한 증상을 만드는 다른 원인도 있다.

## 모든 Connection이 사용 중이면 요청은 기다린다

Connection을 재사용해도 Pool이 가진 수보다 많은 요청이 동시에 DB 작업을 할 수는 없다.

```text
Pool Size = 10

Active = 10
Idle = 0

새 요청 도착
```

Pool이 이미 최대 크기에 도달했다면 Connection을 무한히 추가하지 않는다. 새 요청은 누군가 사용 중인 Connection을 반납할 때까지 기다린다.

![HikariCP가 Connection을 탐색하고 대여하는 흐름](/images/posts/jdbc-and-hikaricp/legacy-03.png "HikariCP Connection 획득 흐름")

```text
Connection 대여 시도
├─ 빌려줄 Connection 있음
│  └─ 즉시 사용
└─ 빌려줄 Connection 없음
   └─ connectionTimeout까지 대기
      ├─ Connection 반환됨 → 사용
      └─ 시간 초과 → SQLException
```

이 상황에서 Pool 지표의 이름도 자연스럽게 이해할 수 있다.

| 지표 | 의미 |
| --- | --- |
| Active | 현재 요청이 빌려서 사용 중인 Connection |
| Idle | DB와 연결돼 있고 지금 바로 빌려줄 수 있는 Connection |
| Pending | Connection을 얻지 못해 기다리는 요청 |

`Pending`이 늘어난다는 것은 SQL을 실행하기도 전에 요청이 Pool 앞에서 줄을 서고 있다는 뜻이다.

### getConnection이 느리면 SQL보다 앞을 봐야 한다

DB 요청이 느리다고 해서 항상 SQL 실행이 느린 것은 아니다.

```text
요청 시작

↓ 300ms

Connection 획득

↓ 5ms

SQL 실행 완료
```

이 요청에서 SQL 실행 시간은 5ms다. 지연의 대부분은 `dataSource.getConnection()`이 사용할 Connection을 기다리는 동안 발생했다.

따라서 데이터베이스 구간을 측정할 때는 적어도 다음 시간을 구분해야 한다.

```text
Connection 획득 대기
→ SQL 실행
→ 결과 처리
→ Connection 반환
```

슬로 쿼리만 확인하면 Pool 앞의 대기를 놓칠 수 있다. 반대로 Pending이 생겼다는 사실만으로 Pool이 작다고 결론 내리면 Connection을 오래 점유하게 만든 원인을 놓칠 수 있다.

## Pool 크기는 동시 DB 작업의 상한이다

`maximumPoolSize`는 HikariCP가 보유할 수 있는 Active와 Idle Connection을 합친 최대 개수다. 애플리케이션 관점에서는 **동시에 DB 작업에 사용할 수 있는 Connection 수의 상한**으로 이해할 수 있다.

```text
Pool 10
→ 동시에 Connection을 사용할 수 있는 요청 최대 10개

Pool 100
→ 동시에 Connection을 사용할 수 있는 요청 최대 100개
```

숫자만 보면 100이 더 빨라 보인다. 그러나 데이터베이스의 CPU, Disk I/O, Buffer와 Lock 처리 능력은 무한하지 않다.

```text
Connection 증가
→ DB 동시 작업 증가
→ CPU·I/O·Lock 경합 증가
→ Context Switching과 대기 증가
→ 전체 응답 시간이 오히려 길어질 수 있음
```

Connection Pool은 요청을 빠르게 만드는 무한 확장 장치가 아니다. 생성 비용을 줄여주는 동시에 데이터베이스로 들어가는 동시 작업 수를 제한하는 장치이기도 하다.

![Pool 크기에 따른 TPS와 응답 시간 실험 결과](/images/posts/jdbc-and-hikaricp/legacy-04.png "maximumPoolSize 실험 결과")

### 웹 Thread 수와 같게 맞추는 값이 아니다

Tomcat이 요청 Thread를 200개 사용한다고 해서 `maximumPoolSize`도 200이어야 하는 것은 아니다.

```text
요청 Thread
→ JSON 파싱
→ DB 작업 10ms
→ 외부 API 호출 300ms
→ JSON 응답 생성
```

이 요청은 전체 처리 시간 내내 DB Connection을 필요로 하지 않는다. Connection은 DB 작업을 수행하는 동안에만 필요할 수 있다.

Thread Pool과 Connection Pool도 역할이 다르다.

```text
Thread Pool
→ 실행할 Thread를 재사용

Connection Pool
→ DB Connection을 재사용
```

중요한 것은 전체 Request Thread 수가 아니라 **동시에 DB Connection을 필요로 하는 요청 수와 Connection 점유 시간**이다. 같은 요청량이라도 SQL이 느리거나 Transaction이 길어지면 Connection이 늦게 돌아오므로 더 쉽게 Pool이 가득 찬다.

### 초기값을 추정하는 공식

HikariCP의 Pool 크기 가이드는 PostgreSQL 프로젝트에서 제시한 다음 공식을 초기 추정 방법으로 소개한다.

```text
connections = (core_count × 2) + effective_spindle_count
```

`core_count`는 Hyper-Threading으로 늘어난 논리 Thread를 제외한 DB 서버의 CPU Core 수다. `effective_spindle_count`는 I/O 대기 중 다른 작업을 처리할 여지가 얼마나 있는지를 전통적인 회전식 Disk 수로 나타낸 값이다. 활성 데이터가 메모리에 모두 올라가 있다면 0에 가까워지고, Cache Hit Ratio가 낮아질수록 실제 Disk 수에 가까워진다.

예를 들어 DB 서버가 4 Core이고 유효한 Disk가 1개라면 시작점은 다음과 같다.

```text
(4 × 2) + 1 = 9
```

이 값은 그대로 복사할 정답이 아니다. HikariCP 문서도 예상 부하를 재현해 공식 주변의 여러 크기를 시험하라고 설명한다. 특히 SSD·NVMe와 Cloud Storage 환경에서는 `effective_spindle_count`를 단순히 계산하기 어렵다. 공식은 웹 Thread 수에 맞춰 Pool을 크게 잡는 대신 작은 값에서 측정을 시작하기 위한 기준에 가깝다.

또한 이 공식이 추정하는 것은 DB 서버가 감당할 수 있는 전체 Active Connection의 출발점이다. 애플리케이션 인스턴스가 여러 개라면 이 숫자를 각 Pool에 그대로 적용하는 것이 아니라 전체 Connection 예산을 나누어야 한다.

### 여러 애플리케이션의 합계도 봐야 한다

애플리케이션 인스턴스마다 Pool이 하나씩 생긴다면 데이터베이스가 상대하는 Connection은 각 Pool 설정의 합계가 된다.

```text
App 1: maximumPoolSize = 20
App 2: maximumPoolSize = 20
App 3: maximumPoolSize = 20

DB가 받을 수 있는 최대 Connection
→ 애플리케이션 Pool만 합쳐도 60개
```

여기에 배치 서버, 관리 도구와 다른 서비스의 Connection도 더해진다. Pool 크기는 애플리케이션 하나만 보고 정할 수 없으며, 데이터베이스의 처리 능력과 전체 연결 예산 안에서 측정해야 한다.

## 주요 설정은 대여·대기·교체 시점을 조절한다

HikariCP의 주요 설정은 이름을 따로 외우기보다 Connection 생명주기의 어느 부분을 조절하는지 보면 구분하기 쉽다.

```text
Connection을 몇 개까지 빌려줄까?
→ maximumPoolSize

지금 빌릴 수 없다면 얼마나 기다릴까?
→ connectionTimeout

물리 Connection 하나를 얼마나 오래 재사용할까?
→ maxLifetime
```

세 값은 서로 다른 시점의 문제를 다룬다. 이름에 `Timeout`이나 `Lifetime`이 들어간다는 이유로 비슷한 시간 설정이라고 생각해서는 안 된다.

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      connection-timeout: 30000
      max-lifetime: 1740000
```

위 값은 설정 형태를 보여주기 위한 예시일 뿐이다. 적절한 값은 데이터베이스와 네트워크의 제한 시간, 요청량, Connection 점유 시간과 애플리케이션 인스턴스 수를 측정해 정해야 한다.

### maximumPoolSize는 대여 가능한 총량을 제한한다

Idle과 Active를 합쳐 Pool이 보유할 수 있는 최대 Connection 수다. Pool이 이 크기에 도달하고 Idle Connection도 없다면 새 `getConnection()` 호출은 기다린다.

값을 늘리면 Pool 앞의 대기가 줄어들 수 있지만, 그만큼 데이터베이스에 동시에 더 많은 작업이 들어간다. 따라서 Pending이 있다는 이유만으로 값을 늘리기보다 DB가 추가 동시 작업을 감당할 수 있는지 먼저 확인해야 한다.

### connectionTimeout은 요청이 기다릴 시간이다

Pool에서 즉시 빌릴 Connection이 없을 때 호출자가 기다릴 수 있는 최대 시간이다. 그 안에 Connection이 반환되지 않으면 `SQLException`이 발생한다.

```text
Request
→ Pool에서 Connection 대기
→ 얼마나 기다릴 것인가?
```

값이 너무 길면 Pool 고갈이 요청 Thread 적체로 번질 수 있다. 너무 짧으면 잠깐의 부하에도 실패가 늘어날 수 있다. 이 값은 Connection 생성 시간이나 SQL Timeout을 정하는 설정이 아니다.

### maxLifetime은 물리 Connection의 수명이다

Pool이 물리 Connection 하나를 재사용할 수 있는 최대 수명이다.

```text
물리 Connection 생성
→ 여러 번 대여와 반납
→ 수명 도달
→ 반환된 뒤 제거
→ 새 Connection으로 교체
```

`connectionTimeout`이 요청의 대기 시간을 다룬다면 `maxLifetime`은 Connection 자체의 교체 주기를 다룬다.

HikariCP는 사용 중인 Connection을 수명이 됐다는 이유로 강제로 끊지 않는다. Connection이 반환되면 Pool에서 제거한다. 따라서 `maxLifetime`이 30분이라고 해서 30분째 실행 중인 SQL을 즉시 중단한다는 뜻은 아니다.

HikariCP는 데이터베이스나 네트워크 인프라가 적용하는 Connection 제한 시간보다 `maxLifetime`을 몇 초 짧게 설정할 것을 권장한다. 이유는 다음과 같은 엇갈림을 줄이기 위해서다.

```text
HikariCP
→ Connection이 살아 있다고 생각

DB 또는 Network 장비
→ 자체 제한 시간으로 이미 연결 종료
```

외부 장비가 먼저 연결을 끊으면 다음 요청이 죽은 Connection을 만날 수 있다. Pool이 그보다 먼저 Connection을 교체하도록 수명을 조절하는 것이다. 구체적인 값은 데이터베이스, Driver, Proxy와 Network 장비의 설정에 따라 달라진다.

### 필요할 때 추가로 보는 설정

처음에는 `maximumPoolSize`, `connectionTimeout`, `maxLifetime`의 차이를 이해하는 것이 우선이다. Pool을 탄력적으로 운용할 필요가 있다면 `minimumIdle`과 `idleTimeout`도 살펴볼 수 있다.

- `minimumIdle`: HikariCP가 유지하려는 최소 Idle Connection 수
- `idleTimeout`: `minimumIdle`보다 남는 Idle Connection을 정리하기까지 기다리는 시간

`idleTimeout`은 `minimumIdle`이 `maximumPoolSize`보다 작을 때만 Pool 크기 축소에 영향을 준다. HikariCP는 별도 요구가 없다면 `minimumIdle`을 따로 지정하지 않고 사실상 고정 크기 Pool처럼 사용하는 구성을 권장한다.

## Pending이 많아도 Pool부터 키우지 않는다

Pending이 늘었다면 `maximumPoolSize`를 먼저 키우고 싶어진다. 그러나 Pending은 Connection 공급보다 수요가 많다는 증상일 뿐, 왜 부족해졌는지까지 알려주지는 않는다.

### Connection이 돌아오지 않는 경우

JDBC 자원의 `close()`가 누락되면 Connection이 Pool로 돌아오지 않는다. 이 경우 실제로 사용할 수 있는 Connection이 계속 줄어든다.

HikariCP의 `leakDetectionThreshold`는 Connection이 Pool 밖에 일정 시간 이상 머물면 대여한 코드 위치를 로그로 남기는 진단 설정이다. 하지만 정상적으로 오래 실행되는 쿼리나 Transaction도 Leak 후보로 기록될 수 있다. 상시 해결책이 아니라 반환 지연 위치를 찾는 보조 도구로 사용해야 한다.

### Connection을 너무 오래 점유하는 경우

Connection을 정상적으로 반환하더라도 점유 시간이 길면 같은 증상이 나타난다.

```text
Transaction 시작
→ Connection 획득
→ SELECT 또는 UPDATE 10ms
→ 외부 API 응답 3초 대기
→ 추가 SQL 10ms
→ Commit
→ Connection 반환
```

실제 DB 작업은 20ms지만 이미 Connection을 획득한 상태에서 외부 API를 기다렸다면 약 3초 동안 Pool 자리를 차지할 수 있다. 이때 문제는 SQL 자체보다 Transaction 경계다.

JDBC Transaction은 일반적으로 하나의 Connection에서 `commit()` 또는 `rollback()`까지 수행한다. Spring도 Transaction 안에서 데이터 접근 코드가 같은 Connection을 사용하도록 관리한다. 따라서 Transaction이 길어질수록 Connection 반환도 늦어진다.

다음 원인도 Connection 점유 시간을 늘린다.

- 느린 SQL과 대량 결과 처리
- 다른 Transaction이 가진 Lock 대기
- 불필요하게 넓은 `@Transactional` 범위
- DB 작업 사이에 포함된 외부 API나 파일 처리

### 데이터베이스의 처리량을 넘은 경우

Connection이 제때 반환되고 SQL도 개별적으로는 정상이어도, 요청량이 데이터베이스가 감당할 수 있는 처리량을 계속 넘으면 Pending이 생긴다. 이때 Pool을 키우면 대기 장소가 애플리케이션에서 데이터베이스 내부로 이동할 뿐 전체 처리량은 늘지 않을 수 있다.

그래서 Pool 고갈은 다음 순서로 확인하는 편이 안전하다.

```text
1. Connection 획득 시간이 느린가?
2. Active가 계속 최대치이고 Idle이 0인가?
3. Pending과 Timeout이 함께 증가하는가?
4. Connection이 정상적으로 반환되는가?
5. 느린 SQL, Lock 대기와 긴 Transaction이 있는가?
6. DB CPU와 I/O가 이미 포화 상태인가?
7. 모든 애플리케이션 Pool의 합계가 적절한가?
8. 그다음 maximumPoolSize 조정을 검토
```

Active, Idle, Pending과 Timeout은 증상을 보여준다. 원인은 SQL, Transaction, 애플리케이션 코드와 DB 자원까지 내려가 확인해야 한다.

## 실무 사례: maxLifetime을 너무 짧게 설정했더니

여기부터는 HikariCP의 기본 원리보다 한 단계 더 깊은 운영 사례다.

한 서비스에서는 데이터베이스 장애 전환 뒤 새 대상에 빠르게 연결하기 위해 `maxLifetime`을 50초로 설정했다. HikariCP 기본값인 30분보다 물리 Connection을 훨씬 자주 생성하고 폐기하는 조건이었다.

당시 사용한 MySQL Connector/J에서는 폐기된 Connection과 관련된 네트워크 자원을 `AbandonedConnectionCleanupThread`가 정리하고 있었다. 이 Thread는 정리 대상을 한 번에 하나씩 처리했다.

```text
짧은 maxLifetime
→ Connection 생성과 폐기 빈도 증가
→ Driver의 Cleanup Thread가 폐기 속도를 따라가지 못함
→ Connection 관련 참조 누적
→ Heap 사용량 증가
→ OOM
```

겉으로는 Connection 객체가 계속 늘어났기 때문에 애플리케이션이 `close()`를 빠뜨린 Pool Leak처럼 보일 수 있었다. 하지만 Pool 지표만으로 원인을 찾을 수 없었고, Heap Dump에서 `connectionFinalizerPhantomRefs`와 `AbandonedConnectionCleanupThread`로 이어지는 참조 경로를 확인한 뒤 Driver 내부 정리 병목임을 알 수 있었다.

`PhantomReference`의 JVM 동작을 모두 알아야 이 사례를 이해할 필요는 없다. 핵심은 폐기된 Connection 관련 자원을 Driver가 별도 Thread에서 정리하고 있었고, Connection을 버리는 속도가 정리 속도보다 빨라 대기 대상이 쌓였다는 점이다.

해당 팀은 장애 전환을 위한 짧은 수명이라는 목적을 유지하면서 Connector/J를 지원되는 버전으로 올리고 Cleanup Thread 비활성화 옵션을 적용했다. Connector/J 8.0.22부터는 `com.mysql.cj.disableAbandonedConnectionCleanup` 시스템 속성으로 이 Thread를 비활성화할 수 있다.

이 사례를 `maxLifetime`을 짧게 설정하면 항상 OOM이 발생한다는 규칙으로 받아들여서는 안 된다. 특정 Connector/J 버전과 운영 조건에서 발생한 문제다. 여기서 얻을 수 있는 일반적인 교훈은 다음과 같다.

```text
Connection 생성
→ Pool에서 재사용
→ HikariCP가 폐기
→ Driver가 네트워크 자원 정리
→ DB와 Network가 연결 종료 인식
```

Connection의 생명주기는 HikariCP 설정 하나에서 끝나지 않는다. JDBC Driver, 데이터베이스, Network 장비와 장애 전환 정책까지 이어진다. 메모리 증가와 Connection 객체 증가가 함께 보인다면 Pool 지표뿐 아니라 Heap Dump의 실제 참조 경로와 Driver 버전도 확인해야 한다.

## 정리

- Java 애플리케이션은 공통 JDBC API를 사용하고, JDBC Driver가 데이터베이스별 통신을 담당한다.
- Connection은 단순 Java 객체가 아니라 데이터베이스 Session과 연결된 통신 경로다.
- Connection Pool은 생성 비용이 큰 물리 Connection을 대여하고 반납받아 재사용한다.
- Pool 환경의 `close()`는 실제 연결 종료가 아니라 반납이므로 반드시 호출해야 한다.
- `maximumPoolSize`는 동시 DB 작업의 상한, `connectionTimeout`은 요청의 대기 시간, `maxLifetime`은 물리 Connection의 교체 주기다.
- Pending이 늘면 Pool 크기보다 반환 누락, Connection 점유 시간, 느린 SQL, Lock, Transaction 경계와 DB 처리량부터 확인한다.
- Pool 튜닝은 숫자 하나를 키우는 일이 아니라 Connection의 생성부터 폐기까지 전체 생명주기를 조절하는 일이다.

## 참고 자료

### 공식 자료

- [Oracle Java Tutorials - Processing SQL Statements with JDBC](https://docs.oracle.com/javase/tutorial/jdbc/basics/processingsqlstatements.html)
- [Oracle Java Tutorials - Connecting with DataSource Objects](https://docs.oracle.com/javase/tutorial/jdbc/basics/sqldatasources.html)
- [Spring Boot - SQL Databases](https://docs.spring.io/spring-boot/3.5/reference/data/sql.html)
- [HikariCP 공식 문서](https://github.com/brettwooldridge/HikariCP)
- [HikariCP - About Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
- [MySQL Connector/J 8.0.22 Release Notes](https://dev.mysql.com/doc/relnotes/connector-j/en/news-8-0-22.html)

### 국내 기술 블로그

- [컬리 기술 블로그 - 99%가 모른다는 DB Connection 누수 문제](https://helloworld.kurly.com/blog/connection-leak/)
