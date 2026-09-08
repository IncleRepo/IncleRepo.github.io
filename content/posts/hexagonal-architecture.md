+++
title = '애플리케이션을 중심에 두는 헥사고날 아키텍처'
slug = '16'
date = 2026-09-07T21:10:00+09:00
lastmod = 2026-09-07T21:10:00+09:00
draft = false
references_required = true
description = 'Controller, Service, Repository 구조에서 출발해 포트와 어댑터가 외부 기술의 의존성을 어떻게 애플리케이션 밖으로 밀어내는지 살펴봅니다.'
categories = ['애플리케이션 아키텍처']
tags = ['헥사고날 아키텍처', '포트와 어댑터', '의존성 역전', 'Application Service']
+++

[레이어드 아키텍처](/posts/14/)에서 Controller, Service와 Repository의 책임을 나누는 방법을 살펴봤다. 이어 [DDD](/posts/15/)에서는 업무 규칙을 도메인 모델에 담고, Application Service가 이를 이용해 하나의 작업을 완성하도록 역할을 구분했다.

그런데 Application Service가 JPA, Redis와 외부 API의 사용법까지 직접 알고 있다면 어떨까? 역할을 나눠도 외부 기술이 바뀔 때 애플리케이션 코드까지 함께 수정될 수 있다.

**헥사고날 아키텍처**(Hexagonal Architecture)는 이 결합을 줄이기 위해 애플리케이션과 외부 기술 사이에 경계를 만든다. 애플리케이션이 필요한 기능을 먼저 정의하면, 외부 기술을 다루는 코드가 그 계약에 맞추는 구조다.

## Service가 외부 기술의 사용법까지 알고 있다

로그인하려면 회원을 조회하고 비밀번호를 확인한 뒤, Token을 발급하고 로그인 Session을 저장해야 한다. 레이어드 구조에서는 이 일련의 흐름을 다음처럼 하나의 Service에 구현할 수 있다.

```java
@Service
@RequiredArgsConstructor
public class SignInService {

    private final JpaUserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final JwtTokenProvider tokenProvider;
    private final RedisTemplate<String, String> redisTemplate;

    public SignInResponse signIn(SignInRequest request) {
        User user = userRepository.findByLoginId(request.loginId())
            .orElseThrow(UserNotFoundException::new);

        if (!passwordEncoder.matches(request.password(), user.getPassword())) {
            throw new InvalidPasswordException();
        }

        String accessToken = tokenProvider.createAccessToken(user.getId());
        redisTemplate.opsForValue().set(
            "login:" + user.getId(),
            accessToken
        );

        return new SignInResponse(accessToken);
    }
}
```

로그인 처리 코드 안에는 JPA 조회, JWT 발급 API 호출과 Redis 저장 명령이 함께 들어 있다. 이들이 로그인 흐름에 섞여 있어 Session 저장 방식을 바꾸면 로그인 순서가 그대로여도 `SignInService`를 고쳐야 한다.

이 결합은 단위 테스트에도 이어진다. 외부 기술을 Mock으로 대체하려면 `RedisTemplate`의 호출 방식과 JWT Provider의 반환 형식에 맞춰 테스트를 작성해야 하므로, 저장 구현만 바뀌어도 테스트를 함께 수정할 수 있다. **애플리케이션이 하는 일과 그 일을 수행하는 기술이 한 코드에서 함께 움직이는 상태**다.

## 애플리케이션의 안과 밖을 나눈다

이 둘을 분리하기 위해 헥사고날 아키텍처는 애플리케이션을 중심으로 안과 밖을 나눈다. 로그인 순서와 정책은 안쪽에 두고, HTTP 요청이나 데이터 저장처럼 환경에 따라 달라지는 구현은 바깥쪽에 둔다.

![외부 요청이 입력 어댑터와 입력 포트를 거쳐 애플리케이션으로 들어오고 출력 포트와 출력 어댑터를 통해 외부 기술로 나가는 구조](/images/posts/hexagonal-architecture/ports-and-adapters.svg "애플리케이션을 중심에 둔 Port와 Adapter")

안쪽과 바깥쪽이 만나는 자리가 **Port**다. Port는 애플리케이션이 외부와 주고받을 작업을 정의한 API이며, Java에서는 주로 Interface나 메서드로 표현한다.

이 Port에 외부 기술을 연결하는 코드가 **Adapter**다. Controller는 HTTP 요청을 애플리케이션 입력으로 바꾸고, JPA나 Redis Adapter는 애플리케이션의 요청을 실제 기술로 처리한다.

이처럼 Port와 Adapter로 경계를 구성해 **Ports and Adapters Architecture**라고도 부른다. Hexagon의 여섯 면은 정해진 구성 요소 수가 아니라, 애플리케이션이 여러 외부 환경과 연결될 수 있음을 표현한 것이다.

로그인 예제에 적용하면 Controller에서 요청을 받는 쪽과 저장소 등에 작업을 맡기는 쪽으로 나눠 볼 수 있다. 먼저 요청이 들어오는 쪽부터 살펴보자.

## 들어오는 요청은 입력 Port로 받는다

Controller가 로그인 요청을 전달할 수 있도록 애플리케이션의 진입점을 정의한다.

```java
public interface SignInUseCase {

    SignInResult signIn(SignInCommand command);
}
```

`SignInUseCase`는 로그인에 필요한 입력과 반환할 결과를 정의한 **입력 Port**다. `SignInService`가 이 계약을 구현해 실제 로그인 흐름을 진행하고, Controller는 HTTP 요청을 입력 Port에 전달하는 입력 Adapter가 된다.

```java
@RestController
@RequiredArgsConstructor
public class UserController {

    private final SignInUseCase signInUseCase;

    @PostMapping("/sign-in")
    public SignInResponse signIn(@RequestBody SignInRequest request) {
        SignInCommand command = new SignInCommand(
            request.loginId(),
            request.password()
        );

        SignInResult result = signInUseCase.signIn(command);
        return SignInResponse.from(result);
    }
}
```

Controller는 JSON 요청을 `SignInRequest`로 받은 뒤, 로그인에 필요한 값을 `SignInCommand`에 담아 전달한다. HTTP 형식의 처리가 Controller에서 끝나므로 입력 Port는 애플리케이션 입력만 다룬다.

이렇게 진입점을 두면 다른 Adapter에서도 같은 로그인 기능을 호출할 수 있다. 관리자 CLI나 Message Consumer의 입력을 `SignInCommand`로 바꿔 전달하면, `SignInService`에 구현한 로그인 흐름을 그대로 사용한다.

Controller처럼 애플리케이션의 동작을 시작시키는 입력 Adapter는 **Driving Adapter** 또는 Primary Adapter라고도 부른다.

## 나가는 작업은 출력 Port에 맡긴다

요청을 받은 `SignInService`는 회원 조회, 비밀번호 비교, Token 발급과 Session 저장을 진행해야 한다. 앞에서는 각 기술의 API를 직접 호출했지만, 이번에는 애플리케이션에 필요한 작업을 Interface로 정의해 보자.

```java
public interface LoadUserPort {

    User loadByLoginId(String loginId);
}

public interface PasswordVerifier {

    boolean matches(String rawPassword, String encodedPassword);
}

public interface TokenIssuer {

    IssuedTokens issue(User user);
}

public interface LoginSessionStore {

    void save(LoginSession session);
}
```

이들은 애플리케이션이 외부에 요구하는 **출력 Port**다. 예를 들어 `LoginSessionStore`는 Session을 저장한다는 작업만 정의하므로, 이를 호출하는 쪽에서 Redis의 Key나 자료 구조를 다룰 필요가 없다.

이제 `SignInService`가 각 기술의 API 대신 이 Port들을 사용하도록 바꿔 보자.

```java
@Service
@RequiredArgsConstructor
public class SignInService implements SignInUseCase {

    private final LoadUserPort loadUserPort;
    private final PasswordVerifier passwordVerifier;
    private final TokenIssuer tokenIssuer;
    private final LoginSessionStore loginSessionStore;

    @Override
    public SignInResult signIn(SignInCommand command) {
        User user = loadUserPort.loadByLoginId(command.loginId());

        if (!passwordVerifier.matches(command.password(), user.getPassword())) {
            throw new InvalidPasswordException();
        }

        IssuedTokens tokens = tokenIssuer.issue(user);
        loginSessionStore.save(LoginSession.of(user.getId(), tokens));

        return SignInResult.from(tokens);
    }
}
```

`SignInService`는 Port를 호출하며 로그인 작업의 순서를 이어 간다. 회원을 어디에서 조회하고 Session을 어디에 저장할지는 각 Port를 구현한 Adapter가 맡는다.

## Adapter가 Port를 실제 기술로 연결한다

앞에서 Service가 직접 다루던 기술별 코드를 Adapter로 옮겨 보자. 회원 조회는 JPA Adapter가 `LoadUserPort`를 구현해 처리한다.

```java
@Component
@RequiredArgsConstructor
public class JpaUserAdapter implements LoadUserPort {

    private final SpringDataUserRepository userRepository;

    @Override
    public User loadByLoginId(String loginId) {
        return userRepository.findByLoginId(loginId)
            .orElseThrow(UserNotFoundException::new);
    }
}
```

Session 저장도 같은 방식으로 옮긴다. Redis Adapter가 `LoginSessionStore`를 구현하고, 그 안에서 Redis Key와 저장할 값을 지정한다.

```java
@Component
@RequiredArgsConstructor
public class RedisLoginSessionAdapter implements LoginSessionStore {

    private final RedisTemplate<String, String> redisTemplate;

    @Override
    public void save(LoginSession session) {
        redisTemplate.opsForValue().set(
            "login:" + session.userId(),
            session.refreshToken(),
            session.timeToLive()
        );
    }
}
```

JWT와 Spring Security를 다루는 Adapter도 각각 `TokenIssuer`와 `PasswordVerifier`를 구현한다. Spring이 이 구현체들을 `SignInService`에 주입하면 Port를 통해 실제 기술을 호출할 수 있다.

이렇게 분리하면 Redis 저장 방식을 바꿀 때 수정할 코드가 `RedisLoginSessionAdapter`에 모인다. 단위 테스트에서는 이 Adapter 대신 같은 Port를 구현한 가짜 저장소를 연결해, `RedisTemplate`의 호출 방식에 맞추지 않고도 Session이 저장되었는지 확인할 수 있다.

입력 Adapter가 애플리케이션을 실행시켰다면, 출력 Adapter는 애플리케이션의 호출을 받아 동작한다. 그래서 DB, Message Broker와 외부 API 쪽의 출력 Adapter를 **Driven Adapter** 또는 Secondary Adapter라고도 부른다.

## 실행 흐름과 코드 의존 방향은 다르다

Adapter로 구현을 옮긴 뒤에도 로그인 호출은 Controller에서 Redis까지 이어진다. 달라진 것은 코드의 참조 방향으로, 외부 기술이 있는 바깥쪽이 아니라 Port가 있는 안쪽을 향한다.

![로그인 실행 흐름은 Controller에서 Redis로 이어지고 코드 의존 방향은 입력 포트와 출력 포트를 향하는 구조](/images/posts/hexagonal-architecture/execution-and-dependency.svg "실행 흐름과 코드 의존 방향의 차이")

`SignInService`와 `RedisLoginSessionAdapter`는 모두 `LoginSessionStore`를 참조한다. Service는 이 계약을 사용하고 Adapter는 이를 구현하므로, Redis 코드가 애플리케이션이 정한 계약에 맞추게 된다.

이러한 구조가 **의존성 역전**(Dependency Inversion)이다. 호출은 바깥으로 나가지만 소스코드의 의존은 애플리케이션이 소유한 Port를 향한다.

## 패키지는 경계를 드러내는 만큼 나눈다

지금까지 만든 Port와 Adapter를 역할에 따라 배치하면 다음과 같은 패키지 구조가 된다.

```text
user
├── adapter
│   ├── in
│   │   └── web
│   │       ├── UserController
│   │       ├── SignInRequest
│   │       └── SignInResponse
│   └── out
│       ├── persistence
│       │   └── JpaUserAdapter
│       ├── redis
│       │   └── RedisLoginSessionAdapter
│       └── token
│           └── JwtTokenAdapter
├── application
│   ├── port
│   │   ├── in
│   │   │   └── SignInUseCase
│   │   └── out
│   │       ├── LoadUserPort
│   │       ├── LoginSessionStore
│   │       └── TokenIssuer
│   └── service
│       └── SignInService
└── domain
    └── User
```

이렇게 경계를 드러내면 외부 요청이 어디로 들어오고, 애플리케이션이 어떤 외부 기능을 필요로 하는지 찾기 쉽다. 다만 모든 경계에 Interface와 변환 코드를 두면 작은 기능을 수정할 때도 여러 파일을 오가게 된다.

규모가 작은 프로젝트라면 기존 기능별 패키지를 유지하면서 필요한 출력 Port부터 분리하는 방법도 있다.

```text
user
├── controller
│   ├── UserController
│   ├── SignInRequest
│   └── SignInResponse
├── service
│   ├── UserService
│   ├── SignInCommand
│   └── port
│       ├── LoginSessionStore
│       └── TokenIssuer
├── model
│   └── User
├── repository
│   └── UserRepository
└── infrastructure
    ├── redis
    └── token
```

이 구성에서는 Controller가 `UserService`를 직접 호출하되, Redis와 JWT처럼 변경과 테스트의 부담이 큰 지점에 출력 Port를 둔다. 이후 같은 기능을 HTTP와 Message Consumer가 함께 호출하게 되면 입력 Port도 분리할 수 있다.

DTO도 실제 변경이 어떻게 일어나는지 보고 나눌 수 있다. HTTP 요청 형식이 바뀌어도 로그인에 필요한 값은 그대로라면 `SignInRequest`와 `SignInCommand`를 분리할 이유가 생긴다. 입력 경로가 하나이고 두 형식이 함께 바뀌는 동안에는 하나로 시작할 수 있다.

패키지 구분에 더해 의존 방향까지 강제하고 싶다면 Gradle Module로 경계를 나눌 수 있다. 허용하지 않은 참조를 컴파일 단계에서 막을 수 있어 규모와 협업 인원이 커질수록 유용하다. 팀이 패키지 규칙만으로 경계를 유지할 수 있는지, 빌드 단계의 제약도 필요한지를 보고 선택하면 된다.

## 필요한 경계부터 적용한다

경계를 어디까지 나눌지는 변경과 테스트에서 얻는 이점, 그리고 추가 코드를 관리하는 부담을 함께 보고 정한다. 헥사고날 아키텍처로 변경 범위를 좁히고 독립적인 테스트를 작성하기 쉬워지는 만큼 Port, Adapter와 변환 코드도 관리해야 하기 때문이다.

| 상황 | 적용 방식 |
|---|---|
| 외부 API나 저장 기술의 변경이 Service까지 자주 번진다 | 출력 Port로 기술 경계를 분리한다 |
| 같은 기능을 HTTP, Batch와 Message Consumer가 호출한다 | 입력 Port로 애플리케이션의 진입점을 통일한다 |
| DB나 외부 서버 없이 핵심 흐름을 테스트하기 어렵다 | Port에 테스트용 Adapter를 연결한다 |
| 화면에 필요한 값을 단순 조회한다 | 기존 Service와 QueryRepository 흐름을 유지한다 |
| 업무 규칙과 기술 변경 가능성이 모두 낮은 CRUD다 | 추가 경계가 주는 이득을 확인한 뒤 도입한다 |

앞에서 본 로그인처럼 JPA, 암호화, JWT와 Redis가 하나의 작업에 참여한다면 각 기술과의 경계를 나누는 효과를 기대할 수 있다. 외부 결제 API, 파일 저장소와 Message Broker를 호출하는 기능에서도 같은 기준으로 검토하면 된다.

단순한 목록 조회라면 `Controller → Service → QueryRepository` 흐름이 더 읽기 쉬울 수 있다. 프로젝트 전체를 한 번에 바꾸기보다 변경과 테스트의 부담이 실제로 드러난 곳부터 Port와 Adapter를 적용한다.

## 정리

헥사고날 아키텍처는 애플리케이션이 외부 기술의 사용법을 직접 아는 구조를 다음과 같이 바꾼다.

```text
입력 Adapter → 입력 Port ← Application Service → 출력 Port ← 출력 Adapter
```

입력 Port는 애플리케이션이 제공하는 기능을 정의하고, 출력 Port는 애플리케이션에 필요한 외부 기능을 정의한다. Adapter는 HTTP, JPA, Redis와 외부 API를 각 Port에 연결한다.

두 Port 모두 애플리케이션이 계약을 정한다는 점에서 같다. 외부 기술을 다루는 코드가 이 계약에 맞춰 연결되므로, 계약을 유지한 채 구현을 바꿀 때 애플리케이션에 미치는 영향을 줄일 수 있다.

작은 프로젝트에서는 Service의 공개 메서드를 입력 경계로 사용하고, 기술 의존성이 큰 곳부터 출력 Port를 추가할 수 있다. 구조가 줄여 주는 변경 비용과 새로 생기는 코드의 비용을 기능마다 비교하면 된다.

## 참고 자료

### 공식 자료

- [Alistair Cockburn - Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/)
- [AWS Prescriptive Guidance - Hexagonal Architecture Pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/hexagonal-architecture.html)

### 구현 참고

- [Tom Hombergs - BuckPal](https://github.com/thombergs/buckpal)
- [Tom Hombergs - Hexagonal Architecture with Java and Spring](https://reflectoring.io/spring-hexagonal/)

### 국내 적용 사례

- [LINE Engineering - 지속 가능한 소프트웨어 설계 패턴: 포트와 어댑터 아키텍처 적용하기](https://engineering.linecorp.com/ko/blog/port-and-adapter-architecture/)
- [우아한형제들 기술블로그 - Spring Boot Kotlin Multi Module로 구성해보는 헥사고날 아키텍처](https://techblog.woowahan.com/12720/)
- [카카오뱅크 기술블로그 - 구해줘 홈즈! 은행에서 3천만 트래픽의 홈 서비스 새로 만들기](https://tech.kakaobank.com/posts/2411-creating-banking-app-home-screen-for-30-million-traffic/)
- [컬리 Developer Meetup - 헥사고날 아키텍처 적용을 위한 멀티 모듈 구조](https://helloworld.kurly.com/post-img/review-20220224-tech-meetup/keynote.pdf)
