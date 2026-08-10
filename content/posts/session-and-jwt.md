+++
title = '세션과 JWT는 어떤 문제를 다르게 해결하는가'
slug = '11'
aliases = ['/posts/011/']
date = 2026-04-29T19:40:00+09:00
lastmod = 2026-08-10T15:47:49+09:00
draft = false
description = '로그인 이후의 요청을 사용자와 연결하는 문제에서 출발해 세션과 JWT의 구조, 확장, 폐기와 브라우저 보안을 비교합니다.'
categories = ['애플리케이션 보안']
tags = ['인증', '세션', 'JWT', 'HTTP']
+++

로그인 API가 성공했다고 가정해 보자. 브라우저는 이어서 게시글 목록을 요청한다.

```text
1. POST /login
   ID와 Password 전달
   → 로그인 성공

2. GET /posts
```

두 번째 `GET /posts` 요청만 받은 서버는 어떻게 이 요청이 방금 로그인한 김학인의 요청이라는 사실을 알 수 있을까?

HTTP에서는 이전 요청의 결과가 다음 요청에 자동으로 따라오지 않는다. 각각의 요청은 독립적으로 해석된다. 이런 성질을 HTTP의 Stateless, 즉 무상태성이라고 한다.

따라서 로그인 요청 자체를 기억하는 것이 아니라, 이후 요청을 특정 사용자와 연결할 방법이 필요하다. 클라이언트는 매 요청에 로그인 상태를 증명할 정보를 보내고, 서버는 다음 내용을 확인해야 한다.

- 이 요청을 보낸 사용자가 누구인지
- 현재 로그인된 상태인지
- 어떤 권한을 가진 사용자인지

여기서 인증과 인가도 나뉜다.

```text
인증(Authentication)
→ “당신이 김학인인가?”

인가(Authorization)
→ “인증된 김학인이 관리자 페이지를 볼 수 있는가?”
```

세션과 JWT는 주로 누구인지 확인된 상태를 다음 요청까지 이어주는 방법이다. 그 사용자가 특정 기능을 사용할 수 있는지는 그다음에 별도로 판단한다. 이 두 번째 판단이 인가다.

두 방식의 차이는 옛날 방식과 최신 방식의 차이가 아니다. 핵심은 인증 상태를 어디에 두고, 요청마다 어떻게 검증하며, 갱신하고 폐기할 것인가에 있다.

<!--more-->

## 세션은 서버의 로그인 상태를 식별자로 찾는다

먼저 실제 로그인 흐름을 따라가 보자.

```text
로그인 성공
↓
서버가 임의의 Session ID 생성

서버
SESSION=7f3a...
→ userId=15
→ role=MEMBER

브라우저
SESSION=7f3a...
```

다음 요청에서 브라우저는 전달받은 Session ID를 다시 보낸다.

```http
GET /posts HTTP/1.1
Cookie: SESSION=7f3a...
```

```text
서버가 7f3a... 세션 조회
↓
userId=15, role=MEMBER 확인
↓
현재 요청을 15번 사용자 요청으로 처리
```

이처럼 실제 인증 상태는 서버에 저장하고, 클라이언트에는 그 상태를 찾기 위한 식별자만 두는 방법을 서버 세션 방식이라고 한다.

아래 그림에서는 브라우저가 사용자 정보 전체를 들고 다니는 것이 아니라 Session ID만 반복해서 보낸다는 점을 보면 된다.

![Cookie를 이용해 Session ID를 주고받는 흐름](/images/posts/session-and-jwt/legacy-01.png "Cookie와 Session의 기본 흐름")

Session ID는 `userId=15`처럼 의미를 해석할 수 있는 사용자 정보가 아니다. 서버에 저장된 상태를 찾기 위한 임의의 열쇠에 가깝다.

```text
Session ID
≠ 사용자 정보

Session ID
= 서버의 Session을 찾기 위한 임의 식별자
```

공격자가 Session ID를 알아내면 해당 사용자인 것처럼 요청할 수 있다. 따라서 Session ID는 추측하기 어렵게 만들어야 한다.

로그인에 성공했을 때는 기존 Session ID를 새 값으로 바꾸는 것도 중요하다. 공격자가 미리 알아낸 Session ID를 피해자가 로그인한 뒤에도 계속 사용하는 공격을 줄이기 위해서다. 이런 공격을 Session Fixation, 즉 세션 고정 공격이라고 한다.

### Session과 Cookie는 서로 경쟁하는 기술이 아니다

세션 방식에서 Session ID를 반복해서 보내기 위해 Cookie를 자주 사용한다. 그렇다고 Session과 Cookie가 같은 개념은 아니다.

```text
Session
→ 서버가 관리하는 로그인 상태

Cookie
→ 브라우저가 값을 저장하고
   조건에 맞는 요청에 자동으로 실어 보내는 기능
```

Cookie 안에 Session 자체를 넣는다고 이해하면 안 된다. 일반적인 서버 세션 방식에서 Cookie에는 `SESSION=7f3a...` 같은 식별자가 들어가고, 실제 `userId`와 권한은 서버가 이 식별자로 찾아온다.

Session ID를 Cookie로 전달한다면 대표적으로 다음 속성을 확인한다.

- `Secure`: HTTPS 요청에서만 Cookie 전송
- `HttpOnly`: JavaScript가 `document.cookie`로 값을 직접 읽지 못하게 제한
- `SameSite`: 다른 Site에서 시작된 요청에 Cookie를 자동 전송할 범위 제한

이 속성들은 모든 공격을 없애는 기능이 아니다. 인증 정보가 노출되거나 원치 않는 요청에 자동으로 포함될 가능성을 줄이는 장치다.

{{< callout type="warning" title="HttpOnly가 XSS 자체를 막는 것은 아니다" >}}
`HttpOnly`는 JavaScript가 Cookie 값을 직접 읽어 탈취하는 경로를 줄인다. 그러나 사이트에서 공격자의 Script가 실행되는 XSS 취약점 자체를 제거하지는 않는다. 공격자는 사용자의 브라우저에서 인증된 요청을 보내는 등 다른 행동을 시도할 수 있다.
{{< /callout >}}

이제 한 서버의 메모리에서 세션을 관리하는 흐름은 이해했다. 그런데 서버가 여러 대라면 다음 요청은 로그인한 서버와 다른 곳으로 갈 수 있다.

## 서버가 여러 대라면 세션을 어디에서 찾을까

```text
로그인 요청
→ Server A
→ SESSION=123 생성

다음 요청
→ Load Balancer
→ Server B
```

Server B의 메모리에는 `SESSION=123`이 없을 수 있다. 서버가 여러 대가 되면 세션을 어느 서버에서 찾을지가 새로운 문제가 된다.

### 같은 사용자를 항상 같은 서버로 보낸다

Load Balancer가 같은 사용자의 요청을 항상 같은 서버로 보내면 각 서버가 자신의 메모리 세션을 계속 사용할 수 있다. 이 방식을 Sticky Session이라고 한다.

아래 그림에서는 요청이 여러 서버로 무작위 분산되는 대신, 한 사용자의 요청이 계속 같은 서버로 향한다는 점을 보면 된다.

![한 사용자의 요청을 같은 서버로 보내는 Sticky Session](/images/posts/session-and-jwt/legacy-02.png "Sticky Session")

구성이 비교적 단순하지만 특정 서버에 사용자와 부하가 몰릴 수 있다. 해당 서버가 내려가면 그 서버의 메모리에만 있던 세션을 잃을 수도 있다.

### 여러 서버가 세션을 복제한다

Server A의 세션을 Server B와 Server C에도 복제하면 어느 서버로 요청이 가도 같은 상태를 찾을 수 있다. 이를 Session Replication이라고 한다.

하지만 서버 수와 세션 변경량이 늘어날수록 복제해야 할 데이터와 통신량도 증가한다. 모든 서버가 모든 세션을 가지는지, 일부 서버끼리 복제하는지에 따라서도 장애 대응과 운영 방식이 달라진다.

### 모든 서버가 공용 저장소를 조회한다

세션을 애플리케이션 서버 밖의 Redis나 데이터베이스에 저장할 수도 있다.

```text
Server A ─┐
Server B ─┼→ Redis와 같은 공용 Session Store
Server C ─┘
```

모든 서버가 같은 저장소에서 Session ID를 조회하므로 애플리케이션 서버를 비교적 자유롭게 늘릴 수 있다. Spring 애플리케이션에서는 Spring Session으로 기존 `HttpSession`의 저장소를 Redis나 JDBC 기반 저장소로 교체할 수 있다.

대신 인증 경로에 네트워크 조회가 추가되고, 공용 저장소의 지연과 장애도 관리해야 한다. 세션이 사라진 것이 아니라 애플리케이션 서버의 메모리 밖으로 이동한 것이다.

아래 그림에서는 세 가지 방식이 모두 여러 서버에서 세션을 사용하기 위한 선택지이며, 각각 메모리·네트워크·운영 비용이 다르다는 점을 보면 된다.

![세션 복제와 공용 저장소를 이용한 Session Clustering](/images/posts/session-and-jwt/legacy-03.png "Session Clustering")

| 방식 | 해결 방법 | 함께 생기는 비용 |
|---|---|---|
| Sticky Session | 같은 사용자를 같은 서버로 전달 | 부하 편중, 서버 장애 시 메모리 세션 유실 가능성 |
| Session Replication | 서버끼리 세션 상태 복제 | 복제 통신과 메모리 사용량 증가 |
| Shared Session Store | 모든 서버가 공용 저장소 조회 | 네트워크 조회와 저장소 가용성 관리 |

따라서 “세션은 서버가 여러 대면 사용할 수 없다”는 설명은 맞지 않는다. 세션도 충분히 확장할 수 있으며, 실제 질문은 상태를 어디에 두고 요청마다 어떤 비용을 지불할 것인가다.

세션 방식에서는 매 요청마다 Session ID로 서버의 인증 상태를 찾아왔다. 그렇다면 사용자 ID와 권한, 만료 시각 같은 정보를 요청이 직접 들고 오고, 각 서버가 그 정보가 위조되지 않았는지만 확인할 수는 없을까?

## JWT는 사용자 정보를 Token에 담아 전달한다

흔히 사용하는 서명된 JWT는 점으로 구분된 세 부분으로 보인다.

```text
xxxxx.yyyyy.zzzzz
```

각 부분의 역할은 다음과 같다.

```text
Header
→ Token과 서명 방식에 대한 정보

Payload
→ 전달할 사용자 관련 정보

Signature
→ Header와 Payload가 발급 뒤 바뀌지 않았는지 확인할 서명
```

아래 그림에서는 Payload와 Signature가 서로 다른 역할을 한다는 점을 먼저 보면 된다. Payload는 정보를 담고, Signature는 발급 뒤 내용이 바뀌었는지 확인하는 데 사용한다.

![Header, Payload, Signature로 구성된 JWT](/images/posts/session-and-jwt/legacy-04.png "JWT의 세 부분")

이런 구조로 JSON 정보를 전달하는 표준이 JWT다. 이 글에서 다루는 `Header.Payload.Signature` 구조는 JWT에 JWS 서명을 적용한, 인증에서 흔히 볼 수 있는 형태다. 지금은 JWS라는 이름보다 Payload의 정보와 Signature의 역할이 다르다는 점만 이해하면 충분하다.

### Payload에는 사용자와 만료 정보가 들어간다

Payload를 JSON으로 펼치면 다음과 같은 값을 볼 수 있다.

```json
{
  "sub": "15",
  "role": "MEMBER",
  "iat": 1785931200,
  "exp": 1785932100,
  "iss": "https://auth.example.com",
  "aud": "https://api.example.com"
}
```

- `sub`: 이 Token이 누구를 나타내는지
- `role`: 애플리케이션이 추가한 권한 정보
- `iat`: 언제 발급했는지
- `exp`: 언제 만료되는지
- `nbf`: 언제부터 사용할 수 있는지
- `iss`: 누가 발급했는지
- `aud`: 어느 수신자를 대상으로 발급했는지

이처럼 Token 안에 담은 정보 하나하나를 Claim이라고 한다. `sub`, `exp`처럼 표준에서 이름과 의미를 정한 Claim이 있고, `role`처럼 서비스가 직접 추가하는 Claim도 있다. 모든 값을 넣어야 하는 것은 아니며 실제 검증에 필요한 정보만 선택한다.

### Base64URL 인코딩은 암호화가 아니다

Payload는 전송하기 쉬운 문자열 형태로 바뀌지만 내용이 숨겨지는 것은 아니다. 포장 방법을 바꾼 것이지 자물쇠를 채운 것이 아니기 때문이다.

```text
Payload JSON
↓
Base64URL Encoding
↓
문자열 형태 변경
```

Token을 가진 사람은 Payload를 다시 디코딩해 읽을 수 있다. 따라서 비밀번호나 외부에 노출되면 안 되는 민감한 정보를 일반적인 서명 JWT에 넣으면 안 된다. 내용의 기밀성까지 필요하다면 별도의 암호화 표준인 JWE를 검토해야 한다.

### 서명은 내용을 숨기는 대신 변조를 찾아낸다

정상적으로 발급된 Token에 다음 권한 정보가 들어 있다고 해보자.

```text
role = MEMBER
```

공격자는 Payload 문자열을 고쳐 `role = ADMIN`으로 만들 수 있다. 하지만 서명에 필요한 Key를 모르면 변경한 내용에 맞는 Signature를 새로 만들 수 없다. 서버가 신뢰하는 Key로 검증하면 내용과 서명이 맞지 않으므로 이 Token을 거부한다.

서명의 목적은 Payload를 암호화하는 것이 아니다. Header와 Payload가 발급 이후 바뀌지 않았는지, 서버가 신뢰하는 발급자가 만든 Token인지 확인하는 데 있다.

{{< callout type="warning" title="서명이 정상이어도 정보는 오래됐을 수 있다" >}}
서명은 발급 이후 내용이 바뀌지 않았음을 확인한다. Token에 들어 있는 권한이 현재 데이터베이스의 권한과 같다는 사실까지 보장하지는 않는다.
{{< /callout >}}

예를 들어 15시에 다음 Token을 발급했다고 해보자.

```text
JWT 발급 시점
role = MEMBER
```

10분 뒤 데이터베이스에서 사용자의 권한을 `ADMIN`으로 바꿔도 기존 Token의 `role=MEMBER`와 서명은 그대로 정상이다. 별도의 재발급이나 현재 권한 조회가 없다면 만료 전까지 이전 권한 정보가 사용된다.

세션도 권한을 Session에 복사해 두고 갱신하지 않으면 오래된 값을 사용할 수 있다. 다만 서버가 관리하는 상태이므로 Session을 수정하거나 폐기하는 제어 지점을 두기 쉽다. 어떤 방식을 사용하든 권한 변경을 얼마나 빨리 반영해야 하는지 먼저 정해야 한다.

### 서버는 JWT의 모양만 확인하지 않는다

API 요청은 보통 다음 흐름으로 처리한다.

```text
Authorization: Bearer eyJ...
↓
Token 추출
↓
허용한 알고리즘과 Key로 Signature 검증
↓
exp와 nbf 확인
↓
iss와 aud 등 서비스가 요구하는 Claim 확인
↓
사용자 ID와 권한 추출
↓
현재 요청의 인증 정보 구성
↓
인가 판단
```

Spring Security를 사용하면 이 검증 과정을 직접 처음부터 만들 필요는 없다. Spring Security는 서명과 만료 시각 등을 확인하고, Token에서 꺼낸 사용자 정보를 현재 요청의 인증 정보로 바꿔 준다. OAuth2 문서에서는 이렇게 Access Token을 검사하고 보호된 API를 제공하는 서버를 Resource Server라고 부른다.

JWT는 Token에 담긴 정보와 서명을 이용해 기본적인 인증 상태를 확인할 수 있다. 따라서 매 요청마다 중앙 Session Store에서 로그인 상태를 찾는 의존성을 줄일 수 있다.

하지만 “JWT를 쓰면 데이터베이스 조회가 0번”이라는 뜻은 아니다. 현재 권한이나 강제로 폐기한 Token인지 확인해야 한다면 외부 저장소를 다시 조회할 수 있다.

이제 새로운 문제가 생긴다. 이미 정상적으로 발급한 JWT를 사용자가 로그아웃하면 그 Token은 즉시 무효가 될까?

## 정상적으로 발급한 Token은 만료 전까지 살아 있을 수 있다

서명과 `exp`만 확인하는 구조라면 로그아웃했다고 Token 자체가 자동으로 바뀌지는 않는다.

```text
14:00 Token 발급
exp = 17:00

15:00 사용자가 로그아웃

기존 Token
→ Signature 정상
→ exp도 지나지 않음
→ 별도 폐기 확인이 없다면 검증 통과 가능
```

이 문제를 다루는 방법마다 얻는 점과 비용이 다르다.

| 방법 | 얻는 것 | 함께 생기는 비용 |
|---|---|---|
| Access Token 수명을 짧게 설정 | 탈취된 Token이 사용될 수 있는 시간 축소 | 재발급 횟수 증가 |
| 폐기 목록(Blocklist)에 Token 기록 | 만료 전에도 특정 Token 거부 가능 | 매 요청 조회와 목록 관리 필요 |
| 사용자별 Token Version 관리 | 이전 Version의 Token을 한꺼번에 무효화 | 현재 Version을 확인할 서버 상태 필요 |

Blocklist는 아직 만료되지 않았지만 더 이상 허용하지 않을 Token을 적어 두는 목록이다. 요청이 올 때마다 이 목록에 있는지 확인하면 즉시 거부할 수 있지만, 중앙 저장소 조회가 다시 필요하다.

Token Version은 사용자에게 인증 정보의 버전 번호를 두는 방식이다. 예를 들어 데이터베이스의 현재 Version이 `3`인데 Token에는 `2`가 들어 있다면 이전에 발급한 Token으로 판단해 거부한다. 여러 Token을 한꺼번에 무효화하기 쉽지만 역시 현재 Version을 확인해야 한다.

Token을 짧게 만들면 유출 피해 시간은 줄어든다. 하지만 사용자가 몇 분마다 다시 로그인하게 만들 수는 없다. 이 지점에서 Access Token과 Refresh Token을 나누는 설계가 등장한다.

## Access Token과 Refresh Token은 수명과 역할을 나눈다

Access Token과 Refresh Token은 JWT가 반드시 요구하는 두 구성 요소가 아니다. Token 수명과 재인증 문제를 해결하기 위한 인증 설계다.

- Access Token: API 요청에 사용하는 비교적 짧은 수명의 인증 정보
- Refresh Token: 새 Access Token을 발급받기 위한 더 긴 수명의 인증 정보

두 Token이 반드시 JWT 형식일 필요도 없다. Access Token은 서명된 JWT일 수도 있고, 서버에서 조회해야 의미를 알 수 있는 임의 문자열일 수도 있다. Refresh Token도 서버 저장소에서 찾아 검증하는 임의 문자열로 만들 수 있다.

기본 흐름은 다음과 같다.

```text
1. 로그인 성공
   → Access Token과 Refresh Token 발급

2. API 요청
   → Access Token 사용

3. Access Token 만료
   → Refresh Token으로 재발급 요청

4. 서버가 Refresh Token 검증
   → 새 Access Token 발급
```

Access Token을 짧게 유지하면서도 사용자가 매번 ID와 Password를 입력하지 않게 할 수 있다. 대신 Refresh Token은 더 오래 살아 있고 새 Access Token을 만들 수 있으므로 탈취되었을 때 더 큰 피해로 이어질 수 있다. 보관, 만료, 폐기와 기기별 관리가 더 엄격해야 하는 이유다.

### Refresh Token도 사용할 때마다 새 값으로 바꿀 수 있다

재발급에 사용한 Refresh Token을 폐기하고 새 Refresh Token을 발급할 수 있다. 이렇게 사용할 때마다 Token을 새 값으로 교체하는 방식을 Rotation이라고 한다.

아래 그림에서는 Access Token뿐 아니라 Refresh Token도 `RT1 → RT2 → RT3`로 교체되고, 이미 폐기한 `RT1`을 다시 사용하면 거부된다는 점을 보면 된다.

![Access Token 재발급과 Refresh Token Rotation 흐름](/images/posts/session-and-jwt/legacy-05.png "Refresh Token Rotation")

```text
정상 사용자와 공격자가 RT1을 함께 보유
↓
정상 사용자가 RT1으로 재발급
↓
서버가 RT1 폐기 후 RT2 발급
↓
공격자가 RT1 재사용
↓
이미 사용된 Token임을 탐지
```

서버가 이전 Token과 새 Token의 관계를 보관하면 재사용이 발견된 Token 계열 전체를 폐기하는 정책도 만들 수 있다. 이 기능을 운영하려면 Refresh Token 상태가 서버에 남는다.

즉 JWT를 사용해도 강제 폐기, 기기별 로그인, Refresh Token Rotation과 재사용 탐지가 필요하면 서버 상태가 다시 생길 수 있다. “JWT를 사용하면 서버가 완전히 Stateless해진다”는 설명이 지나치게 단순한 이유다.

지금까지는 Token 안에 무엇을 담고 얼마나 오래 사용할지를 정했다. 그런데 브라우저는 이 인증 정보를 어디에 보관하고 어떻게 전송해야 할까?

## Token의 형식과 브라우저 저장 위치는 별개의 선택이다

다음과 같은 공식은 존재하지 않는다.

```text
Session → Cookie
JWT → localStorage
```

JWT 자체를 `HttpOnly` Cookie에 담을 수도 있다.

```http
Set-Cookie: access_token=eyJ...; Secure; HttpOnly; SameSite=Lax
```

반대로 Session ID도 Cookie가 아닌 Header로 전달하도록 설계할 수 있다. JWT와 Session ID는 인증 정보의 모양과 검증 방법에 관한 문제다. Cookie와 JavaScript Storage는 그 정보를 브라우저 어디에 보관하고 어떻게 보낼지에 관한 문제다.

### JavaScript가 읽을 수 있는 저장소

`localStorage`와 같은 저장소의 값은 같은 출처, 즉 같은 Origin에서 실행되는 JavaScript가 읽을 수 있다. 여기서 Origin은 `https://example.com`처럼 Protocol, Host와 Port가 같은 주소 범위를 뜻한다.

```text
공격자의 Script가 사이트에서 실행됨
↓
저장된 Token을 읽음
↓
외부로 전송해 다른 환경에서 재사용
```

따라서 XSS가 발생하면 Token 자체가 탈취될 가능성을 고려해야 한다. XSS는 공격자의 Script가 신뢰받는 사이트 안에서 실행되는 공격이다.

### HttpOnly Cookie

`HttpOnly` Cookie는 JavaScript가 값을 직접 읽는 것을 제한한다. 그러나 브라우저는 조건에 맞는 요청에 Cookie를 자동으로 첨부한다.

```text
공격 Site에서 사용자의 브라우저를 통해 요청 유도
↓
브라우저가 대상 Site의 Cookie를 자동 첨부할 수 있음
↓
사용자가 원하지 않은 작업 실행 가능
```

이런 특성을 악용해 인증된 사용자가 원하지 않은 요청을 보내게 만드는 공격이 CSRF다. Cookie 기반 인증에서는 다음 방어 수단을 서비스 구조에 맞게 함께 검토한다.

- `SameSite`: Cross-site 요청에서 Cookie를 자동 전송할 범위 제어
- CSRF Token: 공격 Site가 알기 어려운 별도 값을 요청과 함께 검증
- Origin 검증: 요청이 서버가 허용한 사이트 주소에서 시작됐는지 확인

이 중 하나만 적용했다고 모든 CSRF가 자동으로 해결되는 것은 아니다. 프론트엔드와 API가 같은 사이트인지, 요청을 어떤 방식으로 보내는지와 브라우저 정책을 함께 확인해야 한다.

| 저장·전송 방식 | 주로 확인할 위험 | 함께 검토할 내용 |
|---|---|---|
| JavaScript가 읽을 수 있는 저장소 | XSS로 인증 정보 자체가 노출될 가능성 | XSS 예방, 짧은 수명, 노출 범위 축소 |
| `HttpOnly` Cookie | Cookie 자동 전송을 이용한 CSRF | `SameSite`, CSRF Token, Origin 검증 |

따라서 “localStorage와 Cookie 중 무엇이 더 안전한가”를 한 문장으로 결정하기 어렵다. 두 방식은 주로 노출되는 공격 경로가 다르며, 어느 쪽도 애플리케이션의 XSS와 CSRF 대응을 대신하지 않는다.

## 선택하기 전에 필요한 보장을 먼저 정한다

지금까지 살펴보면 세션과 JWT 모두 장점과 운영 비용이 있었다. 따라서 서버가 한 대인지 여러 대인지, 어느 기술이 더 최신인지부터 물으면 선택 기준이 흐려진다.

먼저 다음 질문에 답해야 한다.

- 인증 상태는 어디에 둘 것인가?
- 매 요청에서 무엇을 조회하고 검증할 것인가?
- 로그아웃이나 관리자의 강제 폐기를 얼마나 빨리 반영해야 하는가?
- 권한 변경을 얼마나 빨리 반영해야 하는가?
- 여러 API 서버가 중앙 인증 저장소를 거치지 않고 인증 정보를 직접 확인해야 하는가?
- 인증 정보의 수명과 재발급을 어떻게 운영할 것인가?
- 브라우저, 모바일 앱과 서버 간 통신 중 어떤 Client를 지원하는가?
- 인증 정보를 어디에 저장하고 어떤 방식으로 전송할 것인가?

권한 변경 사례로 차이를 확인해 보자.

```text
15:00
userId=15, role=MEMBER

15:05
데이터베이스의 role을 ADMIN으로 변경
```

서버 세션에서는 Session 내용을 갱신하거나 다음 요청에서 권한을 다시 조회하도록 설계할 수 있다. JWT에 `role=MEMBER`, `exp=15:20`이 들어 있다면 별도의 폐기나 재발급 전략이 없는 동안 기존 권한 정보가 유지될 수 있다.

이 차이도 절대적이지는 않다. 세션에 권한을 복사하고 갱신하지 않으면 오래된 값을 사용할 수 있고, JWT를 받더라도 매번 현재 권한을 조회하도록 만들 수 있다. 중요한 것은 기술 이름이 아니라 서비스가 요구하는 반영 시점을 어떤 구조로 보장할 것인가다.

### 서버 세션이 잘 맞을 수 있는 경우

- 로그인 상태를 서버에서 직접 관리하고 즉시 폐기해야 하는 경우
- 권한 변경을 다음 요청부터 빠르게 반영해야 하는 경우
- 브라우저 중심 서비스이며 공용 Session Store를 안정적으로 운영할 수 있는 경우

### 서명된 Token이 잘 맞을 수 있는 경우

- 여러 API 서버가 중앙 세션 조회 없이 Token을 직접 검증해야 하는 경우
- 서비스 사이에 정해진 형식의 사용자 정보를 전달해야 하는 경우
- Token의 만료, 재발급과 폐기 정책을 운영할 수 있는 경우

규모가 커졌다는 이유만으로 세션을 버릴 필요는 없다. 세션도 공용 저장소로 여러 서버에서 사용할 수 있고, JWT도 폐기와 Refresh Token 관리 때문에 서버 상태를 가질 수 있다. 보안 역시 형식 하나를 선택했다고 자동으로 좋아지지 않는다.

결국 인증 기능은 다음 순서로 설계할 수 있다.

```text
로그인 상태를 다음 요청에서 어떻게 확인할까?
↓
인증 상태를 서버가 직접 제어할 필요가 큰가?
↓
즉시 로그아웃과 강제 폐기가 중요한가?
↓
여러 API 서버가 중앙 조회 없이 인증 정보를 검증해야 하는가?
↓
Token을 사용한다면 수명·폐기·권한 반영·재발급은 어떻게 처리할까?
↓
브라우저 Client라면 어디에 저장하고 어떻게 전송할까?
↓
XSS와 CSRF를 어떻게 방어할까?
```

이 흐름은 정답을 자동으로 골라 주는 의사결정 트리가 아니다. 인증을 설계할 때 빠뜨리지 말아야 할 질문의 순서다.

## 정리

- HTTP는 로그인 성공 사실을 다음 요청에 자동으로 연결하지 않으므로 이후 요청마다 사용자를 식별할 인증 정보가 필요하다.
- 세션은 클라이언트가 전달한 Session ID로 서버가 관리하는 로그인 상태를 찾는 방식이다.
- 서버가 여러 대여도 Sticky Session, Session Replication이나 공용 Session Store를 이용해 세션을 운영할 수 있다.
- JWT는 사용자와 만료 정보를 Token에 담고, 서명으로 발급 뒤 내용이 바뀌지 않았는지 확인할 수 있다.
- 일반적인 서명 JWT의 Payload는 암호화된 정보가 아니며, 서명이 정상이어도 Token의 정보가 현재 데이터와 같다는 보장은 없다.
- Access Token과 Refresh Token의 분리는 JWT의 필수 구조가 아니라 수명과 재인증을 다루는 설계다.
- JWT도 강제 폐기, 기기 관리, Rotation과 재사용 탐지가 필요하면 서버 상태를 가질 수 있다.
- 인증 정보의 형식과 브라우저 저장 위치는 별개의 문제이며, XSS와 CSRF를 서비스 구조에 맞게 함께 방어해야 한다.

세션과 JWT의 핵심 차이는 “서버에 상태가 있느냐 없느냐” 한 문장으로 끝나지 않는다. 세션은 클라이언트가 보낸 식별자로 서버의 인증 상태를 찾는 방식이다. JWT는 인증에 필요한 사용자 정보 일부를 Token에 담아 각 서버가 검증하는 방식이다. 실제 선택에서는 상태 위치뿐 아니라 수명, 강제 폐기, 권한 변경, 재발급, 브라우저 저장 방식과 보안 요구까지 함께 설계해야 한다.

## 참고 자료

### 공식 자료

- [IETF RFC 9110 - HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
- [IETF RFC 7519 - JSON Web Token](https://www.rfc-editor.org/rfc/rfc7519.html)
- [IETF RFC 7515 - JSON Web Signature](https://www.rfc-editor.org/rfc/rfc7515.html)
- [IETF RFC 7516 - JSON Web Encryption](https://www.rfc-editor.org/rfc/rfc7516.html)
- [IETF RFC 6749 - OAuth 2.0 Authorization Framework](https://www.rfc-editor.org/rfc/rfc6749.html)
- [IETF RFC 9700 - Best Current Practice for OAuth 2.0 Security](https://www.rfc-editor.org/rfc/rfc9700.html)
- [OWASP - Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [OWASP - HTML5 Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html)
- [OWASP - Cross-Site Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [Spring Session Reference](https://docs.spring.io/spring-session/reference/index.html)
- [Spring Security - OAuth 2.0 Resource Server JWT](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html)
