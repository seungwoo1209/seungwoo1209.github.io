---
title: "알람봇 V2 (2): API Gateway, 인증 인가 구조"
date: 2026-08-07 18:09:05 +0900
categories: [Project]
tags: [알람봇 V2]
---

### 현재 API 서버

- 현재의 API 서버는 단일 EC2 내,
    - 인증: discord OAuth2 회원가입 및 사용자 인증 + JWT 발급, API 호출 시 JWT 검증
    - 알림 등록: 지하철/점심알림 내역 등록
- 알림 등록 API 특성상 호출 횟수가 많지 않고, 봇 컨테이너(크롤링 및 알림 발송) 대비 부하가 현저히 적음
- 초기 설계 방향은 이렇다.
    - API 서버는 API Gateway + lambda로도 충분하므로, 서버리스 구조로 재구성한다.
    - RDS의 역할은 사용자 정보, 등록된 알림 정보 / 알림 기록 저장과 반환이다.
        - DynamoDB로 재구성이 가능한지 분석한다.

### API Gateway 타입: REST API vs HTTP API

요약하자면, HTTP API가 더 기능이 적은 대신 저비용이다.

- REST API에 있는데, HTTP에 없는 기능
    - API 키 발급
    - 클라이언트 별 throttling
    - 요청 트래픽 validation 처리
    - WAF 붙이기
    - private API endpoint
- 위 기능 중 WAF 말곤 필요없다.
    - WAF는 CloudFront를 API Gateway 앞에 두고 CloudFront에 붙일 수 있다.
- 따라서 REST API 대신 HTTP API Gateway 타입으로 API Gateway를 둔다.

### API Gateway의 Authorizer 설계

- REST API Gateway는 Lambda Authorizer과 JWT Authorizer를 지원한다.
- 현재 인증의 구현 요약은 다음과 같다.

<details markdown="1">
<summary>현재의 인증 구현 (AI 요약)</summary>

Discord OAuth로 사용자를 확인한 뒤, 백엔드가 자체 HS256(대칭키 서명) JWT를 발급한다. 

**1. 로그인 및 JWT 발급**

1. `GET /api/v1/auth/discord/login`
    - 5분짜리 OAuth `state` JWT를 생성합니다.
    - Discord 인증 페이지로 리다이렉트합니다.
2. Discord 콜백
    - state JWT의 서명·만료·`typ=state`를 검증합니다.
    - Redis의 `state_used:{jti}`를 `SET NX`로 기록해 재사용을 차단합니다.
    - Discord access token으로 사용자 정보를 조회합니다.
    - DB에서 사용자를 생성하거나 갱신합니다.
    - 자체 access/refresh JWT를 발급합니다.

**2. JWT 구조**

Access와 refresh JWT에는 다음 claim이 들어갑니다.

```json
{
  "sub": "사용자 DB ID",
  "discord_id": 123456789,
  "iat": 1700000000,
  "exp": 1700001800,
  "jti": "고유 UUID",
  "typ": "access 또는 refresh"
}
```

서명 방식과 기본 만료 시간은 다음과 같습니다.

- 알고리즘: `HS256`
- 서명키: `JWT_SECRET`
- Access 만료: 30분
- Refresh 만료: 30일
- OAuth state 만료: 5분

**3. Access Token 검증**

보호된 API는 `get_current_user` 의존성을 사용합니다.

1. `Authorization: Bearer <access_token>` 헤더 추출
2. HS256 서명과 `exp` 만료 검증
3. `typ == "access"` 확인
4. `sub`를 사용자 ID로 변환
5. DB에서 사용자를 다시 조회
6. 사용자가 없거나 `DELETED` 상태면 401 반환

따라서 권한과 사용자 상태는 JWT 내용만 믿지 않고 현재 DB 상태를 반영합니다. 관리자 권한도 DB에서 읽은 `user.role`로 검사합니다.

**4. Refresh Token 검증과 회전**

`POST /api/v1/auth/refresh` 요청 본문으로 refresh token을 전달합니다.

검증 과정은 다음과 같습니다.

1. JWT 서명·만료 확인
2. `typ == "refresh"` 확인
3. Redis에 `refresh_jti:{jti}`가 존재하는지 확인
4. 기존 JTI 삭제
5. 새로운 access/refresh JWT 발급
6. 새 refresh JTI를 Redis에 등록

즉, refresh token은 한 번 사용하면 폐기되는 **rotation 방식**입니다. 같은 토큰을 재사용하면 401이 발생합니다.

**5. 로그아웃**

`POST /api/v1/auth/logout`은 refresh JWT의 JTI를 Redis에서 삭제합니다.

다만 access token은 블랙리스트로 관리하지 않기 때문에, 로그아웃해도 기존 access token은 최대 30분 동안 계속 사용할 수 있습니다.

</details>

- JWT Authorizer는 OIDC 및 OAuth 2.0 프레임워크의 JWT를 검증하는 방식대로 JWT를 검증한다.
    - [AWS 문서: HTTP API JWT authorizer](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-jwt-authorizer.html)
    - 공개키 검증 방식이 가능하며, 현재의 대칭키 인증방식(HS256)에서 공개키 인증방식(RSA) 로 전환해야 한다.
    - JWT Authorizer는 issuer's `jwks_uri` 에서 공개키를 가져와 JWT를 검증한다.
        - 이 jwks_uri에 대해 임의의 엔드포인트를 정하면, "Issuer must have a valid discovery endpoint ended with '/.well-known/openid-configuration'" 이라는 오류가 발생
        - 어딘가에 `??/.well-known/openid-configuration` URL을 가진 곳에 GET으로 공개키를 가져올 수 있게끔 세팅해놔야 한다.
        - 이를 위해선, S3 버킷이 적절하다.
            - S3 버킷에 파일을 저장한 후, 정적 호스팅 또는 cloudfront를 붙이고, route 53으로 도메인 네임 레코드를 두면 된다.
            - 누구나 접근할 수 있다면 HTTPS는 필요하지 않을까?
                - 누군가 중간에서 요청을 가로채 자신의 개인키로 바꿔치기할 수 있다.
            - S3 버킷 → cloudFront 배포 구조로 https 엔드포인트가 필요하다.
- JWT Authorizer로 요구사항을 모두 구현 가능하고, Lambda Authorizer는 별도 코드를 구현해야 하므로 최종적으로 JWT Authorizer를 둔다.
- API Gateway → JWT Authorizer → S3 버킷에서 공개키 가져옴 → 공개키(signature = 개인키(해시(Header.Payload))) == 해시(받은 JWT의 Header.payload) ? → 검증 완료 구조가 된다.

### 개인 키의 관리

- KMS customer managed key를 사용한다.
    - 이유
        - 내가 직접 공개키-개인키 페어를 생성 후 API를 통해 securestring로 저장하지 않으며, KMS API로 생성하고 공개키는 볼 수 있다.
        - 개인키 값은 아무도 볼 수 없으며, 암호화는 KMS의 API 콜 → AWS의 HSM(하드웨어 보안모듈) 내에서 이루어진다.
    - 비용 면에선 어떤가?
        - CMK 1개 당 고정 비용 1 USD
        - 사용자에게 JWT를 발급 시마다 KMS의 개인 키로 암호화 API 호출
            - RSA_2048 키로 Sign API를 호출: 백만 번 당 3 USD (요청당 과금)
            - 백만 개 JWT 당 3 USD, API Gateway HTTP API는 백만 요청당 1 USD
            → JWT 생성 비용이 일반 API 콜 3번 수준, 비용 상으로도 괜찮다.
- 기존의 대칭키 관리 방식에서의 JWT_SECRET은 ssm parameter store에서 했었다. 이와 비교하면 어떤가?
    1. 내가 직접 만들고 저장해야 함
    2. 권한 있는 사용자가 웹 콘솔에서 실제 값을 볼 수 있음
    3. 실수로 삭제하면 재발급해야 하며 모든 유저가 재로그인해야 함
- Secret Manager는 안 쓰나?
    - Secret Manager는 SSM parameter store securestring + 비밀번호 회전 기능과 더한 서비스다.
    - 동일하게 실제 값을 볼 수 있으며, 직접 만들고 저장해야 한다.
    - 100만번 API 호출당 USD 5로, 비용 상에서도 이점이 없다.
- KMS의 API 초당 request rate 관련
    - KMS의 Sign API 에 적용되는 Cryptographic operations (RSA) request rate 은 RSA KMS 키 당 초당 1000번이다.
    - 물론 현재는 나만 쓰는 웹 애플리케이션이므로 괜찮으나, 다른 웹 서비스 운영에서는 참고해야 할 점으로 생각된다.
    - request rate는 Service Quotas 에서 요청을 통해 늘릴 수 있다.

![](/assets/img/알람봇-v2-2-api-gateway-인증-인가-구조/kms-service-quotas.png)

### JWT의 scope 클레임으로 일반 사용자/관리자 1차 인가

- 일반 사용자와 관리자를 구분해야 한다.
    - 관리자는 관리자 페이지에 대한 접근할 수 있다.
- API Gateway에서 1차적으로 일반 사용자에 대해 관리자 페이지의 접근을 차단한다.
    - Authorizer를 붙이고, Authorization scope를 달 때 적용된다.
    - 다만 이는 자격이 박탈된 내용이 데이터베이스에 적용되었으나 토큰은 만료되지 않은 관리자는 못 막고, 따라서 lambda 내에서 DB 조회를 통해 한번 더 확인해야 한다.
    - JWT 발급 시, claim에 scope:admin 과 scope:user로 구분하며 이를 api gateway에서 검증한다.

![](/assets/img/알람봇-v2-2-api-gateway-인증-인가-구조/scope-authorization.png)

### API에서의 최종 인가

- 관리자 권한이 박탈되었음에도 불구하고 만료되지 않은 토큰에는 관리자 권한이 살아 있는 경우
- 사용자의 다른 사용자의 범위의 API 콜 (e.g. 다른 사용자의 알람을 조회하려는 경우)

이런 경우를 위해 실제 데이터베이스의 user 테이블을 조회해야 한다.

- 기존에는 이를 RDS에 조회하였다.
- 다만 lambda를 API 서버로 사용하는 지금의 아키텍처에서 사용자 조회에 RDS를 쓰는 데 의문점이 생긴다.
    - lambda를 private subnet에 두고 RDS Proxy를 추가로 두어야 함, 아키텍처가 복잡해짐
    - 굳이 조인이나 range 쿼리는 없고, 일치 쿼리만 사용하는데도 RDBMS를 사용할 이유가 부족함
    - RDS의 비용(월별 고정비)
- 따라서 DynamoDB 테이블을 두고, 해당 테이블에 RDS에 저장했던 사용자 정보를 저장한다.
- API에서 최종 인가를 위해 DynamoDB 테이블에 쿼리한다.

### 지금까지의 구조

![](/assets/img/알람봇-v2-2-api-gateway-인증-인가-구조/architecture.png)

### Access Token의 처리

> **Discord OAuth 로그인 수행 과정**
>
> 1. `GET /api/v1/auth/discord/login` 엔드포인트로 요청
> 2. API 서버가 state JWT를 만들고, JWT에 다음 정보를 claim으로 포함하여, url에 `...&state=<state JWT>&...` 로 붙여 redirect (307)
>     - 5분 만료 시간 (claim: `exp`)
>     - 128비트 랜덤문자열(claim: `jti`, `uuid4().hex`)
>     - 토큰의 타입(claim: `typ`, 값은 "state")
> 3. 사용자는 discord로 리다이렉트된 후 필요한 절차를 거친다.
>     - discord 로그인
>     - API 서버가 요청한 scope에 대한 허가
>         - `identify` 와 `applications.commands` 스코프를 허가한다.
>             - identify: 사용자 기본 프로필 조회, Discord ID, username 등 조회
>             - `applications.commands`: `integration_type=1`과 함께 사용하면 사용자 계정에 앱을 설치(이제 DM을 보낼 수 있다)
> 4. discord에 의해 사용자가 리다이렉트되어 `GET /api/v1/auth/discord/callback?code=...&state=...` 형태로 다시 서버에 API 콜, API 서버는 JWT를 검증한다.
>     - 서명이 올바른가?
>     - `exp`가 지나지 않았는가?
>     - `typ == "state"`?
>     - `jti`가 존재하는가?
> 5. `state_used:<jti값>` 을 redis에 저장
>     - **목적: 재사용 방지**
>     - 세부 쿼리: `SET state_used:<jti값> "1" NX EX 300`
>         - 키가 없을 때만 저장하며, TTL은 5분으로 설정하는 쿼리
>         - 이 쿼리는 atomic하게 수행된다. 즉, 동시에 두 jti 값을 가진 요청이 도착해서, 두 요청에 대해 없음을 동시에 확인한 후 둘 다 저장에 성공하는 race condition의 발생을 방지한다.
>     - 만약 저장에 실패 시, INVALID_OAUTH_STATE 에러 코드와 401을 반환한다.
> 6. redis에 저장 성공 시, authorization code를 Discord access token으로 교환
>     - token으로 discord user id와 username을 수집
>     - user 테이블에 사용자 추가
{: .prompt-info }

- Valkey(Elasticache) 에서 state 재사용을 방지하기 위해 `jti` 값을 저장한다.
    - 값을 그대로 DynamoDB로 저장하면 되긴 하지만, `SET NX` 의 원자성에 대해 고려할 필요가 있다.
    - DynamoDB에서의 쓰기 API인 PutItem은 ConditionExpression을 통해 여러 조건을 설정할 수 있으며, 그 중 attribute_not_exists를 조건으로 추가하면 항목이 존재하지 않을 때만 추가하며, 존재한다면 실패한다.
    - DynamoDB에서 create, read, update, and delete(CRUD) 연산은 atomic 하다.
    - 따라서 `SET NX` 의 원자성은 dynamoDB로 충분히 구현 가능하여, **OAuth state 재사용 방지는 Valkey(Redis) 대신 DynamoDB를 사용해도 문제없다.**

### Refresh Token의 저장과 처리

<details markdown="1">
<summary>현재 refresh token의 관리 방식 (AI 요약)</summary>

현재 refresh token은 **30일 만료 JWT + Redis JTI 화이트리스트 + 매 갱신 시 rotation** 방식으로 동작한다(아래는 AI 요약). 토큰 원문은 서버에 저장하지 않고, Redis에는 해당 토큰의 식별자인 `jti`만 저장한다.

**1. 설정**

기본 설정은 다음과 같습니다.

```
JWT_SECRET=...
JWT_ALGORITHM=HS256
JWT_REFRESH_EXPIRY_DAYS=30
```

Access·refresh·OAuth state JWT 모두 같은 secret과 알고리즘을 사용하고, `typ` claim으로 용도를 구분합니다.

**2. Refresh JWT 생성**

`create_refresh_token(user_id, discord_id)`가 다음 payload를 생성합니다.

```json
{
  "sub": "사용자 DB ID",
  "discord_id": 123456789,
  "iat": 1770000000,
  "exp": 1772592000,
  "jti": "uuid4 hex 문자열",
  "typ": "refresh"
}
```

세부 동작은 다음과 같습니다.

- `iat`: 현재 UTC 시각
- `exp`: 현재 시각 + 기본 30일
- `jti`: `uuid4().hex`로 매번 새로 생성
- `sub`: 문자열 형태의 DB 사용자 ID
- 반환값: `(서명된 JWT, jti)`
- 서명: 기본 `HS256`, `JWT_SECRET` 사용

`aud`, `iss`, refresh token family ID 같은 claim은 현재 없습니다.

**3. 최초 발급**

Discord OAuth 콜백이 성공하면 다음 순서로 처리됩니다.

1. Discord 사용자 정보를 가져옵니다.
2. DB에 사용자를 생성하거나 갱신합니다.
3. Access token과 refresh token을 발급합니다.
4. refresh token의 JTI를 Redis에 등록합니다.
5. 프론트엔드로 리다이렉트합니다.

Redis에는 다음 형태로 저장됩니다.

```
Key:   refresh_jti:{jti}
Value: 사용자 DB ID
TTL:   JWT_REFRESH_EXPIRY_DAYS × 86400초
```

실제 명령은 사실상 다음과 같습니다.

```
SET refresh_jti:{jti} "{user_id}" EX 2592000
```

현재 토큰은 프론트엔드 URL의 query string으로 전달됩니다.

```
{frontend_url}/?access_token=...&refresh_token=...
```

백엔드는 이후 프론트엔드가 refresh token을 어디에 저장하는지 관여하지 않습니다. HttpOnly 쿠키 설정 등의 서버 로직도 없습니다.

**4. Refresh 요청**

엔드포인트는 다음과 같습니다.

```
POST /api/v1/auth/refresh
Content-Type: application/json

{
  "refresh_token": "..."
}
```

Access token이 없어도 호출할 수 있으며, `Authorization` 헤더를 확인하지 않습니다.

응답은 새 토큰 쌍입니다.

```json
{
  "access_token": "...",
  "refresh_token": "...",
  "token_type": "bearer"
}
```

**5. JWT 자체 검증**

먼저 `decode_token(token, TokenType.REFRESH)`를 호출합니다.

```
jwt.decode(
    token,
    JWT_SECRET,
    algorithms=[JWT_ALGORITHM],
)
```

PyJWT가 서명과 존재하는 표준 시간 claim을 검증하고, 코드에서 추가로 다음을 확인합니다.

```
payload["typ"] == "refresh"
```

다음 경우 `INVALID_AUTH_TOKEN`, HTTP 401이 발생합니다.

- JWT 형식 오류
- 서명 위조
- 만료된 토큰
- 다른 알고리즘
- `typ`이 `refresh`가 아님
- Redis 화이트리스트에 JTI가 없음

다만 `exp`, `sub`, `jti`, `discord_id`를 필수 claim으로 명시한 `require` 옵션은 사용하지 않습니다.

**6. Redis 화이트리스트 검증**

JWT 검증 후 payload에서 다음 값을 꺼냅니다.

```
old_jti = str(payload["jti"])
user_id = int(payload["sub"])
discord_id = int(payload["discord_id"])
```

이어서 Redis에서 다음 키가 존재하는지 확인합니다.

```
EXISTS refresh_jti:{old_jti}
```

키가 없으면 401입니다. 키가 있다면 refresh token이 아직 활성화된 것으로 판단합니다.

Redis에 저장된 값인 `user_id`는 현재 읽지 않습니다. 즉 다음의 일치 여부는 확인하지 않습니다.

```
JWT의 sub == Redis에 저장된 user_id
```

Redis 키의 존재 여부만 사용합니다.

**7. Rotation**

검증에 성공하면 기존 refresh token을 폐기하고 새 토큰 쌍을 발급합니다.

```
기존 JTI 존재 확인
  → 기존 JTI 삭제
  → 새 refresh JWT/JTI 생성
  → 새 JTI Redis 등록
  → 새 access + refresh 반환
```

구체적인 Redis 상태 변화는 다음과 같습니다.

```
DEL refresh_jti:{old_jti}
SET refresh_jti:{new_jti} "{user_id}" EX 2592000
```

따라서 동일한 refresh token을 순차적으로 두 번 사용하면:

- 첫 번째 요청: 성공
- 두 번째 요청: 기존 JTI가 삭제됐으므로 401

새 refresh token의 만료는 다시 현재 시점부터 30일로 설정됩니다. 사용자가 30일 이내에 계속 갱신하면 세션을 계속 연장할 수 있고, 별도의 절대 최대 세션 수명은 없습니다.

**8. 로그아웃**

엔드포인트는 다음과 같습니다.

```
POST /api/v1/auth/logout
Content-Type: application/json

{
  "refresh_token": "..."
}
```

처리 과정은 다음과 같습니다.

1. JWT 서명·만료·`typ=refresh` 검증
2. payload에서 `jti` 추출
3. `refresh_jti:{jti}` 삭제
4. `204 No Content` 반환

`DEL`은 키가 없어도 오류가 아니므로, 아직 만료되지 않은 동일 토큰으로 로그아웃을 반복하면 계속 204를 반환합니다.

로그아웃의 영향 범위는 해당 refresh JTI 하나뿐입니다.

- 다른 브라우저나 기기의 refresh token은 유지됩니다.
- 이미 발급된 access token도 폐기되지 않습니다.
- 기존 access token은 기본 30분 만료까지 계속 사용할 수 있습니다.

**9. 만료 처리**

Refresh token에는 두 가지 만료 조건이 함께 적용됩니다.

1. JWT의 `exp`가 지나면 만료
2. Redis의 JTI 키가 사라지면 비활성화

둘 중 하나라도 실패하면 갱신할 수 없습니다.

Redis TTL은 JWT의 실제 `exp`에서 계산하지 않고 고정적으로 `30 × 86400`초를 적용합니다. JWT `exp`는 정수 초로 절삭되기 때문에 두 만료 시점에 미세한 차이가 생길 수 있습니다. 다만 JWT 검증이 먼저 수행되어 보안 영향은 거의 없고, Redis 키가 아주 조금 더 오래 남을 수 있습니다.

**10. 사용자 DB 상태와의 관계**

Refresh 과정에서는 사용자 테이블을 다시 조회하지 않습니다. JWT의 다음 값을 그대로 신뢰해 새 토큰을 만듭니다.

```
user_id = payload["sub"]
discord_id = payload["discord_id"]
```

따라서 사용자가 삭제됐거나 DB에 존재하지 않더라도, JTI가 Redis에 남아 있으면 refresh 자체는 성공할 수 있습니다.

다만 새 access token으로 보호 API를 호출하면 `get_current_user`가 DB를 조회하므로:

- 사용자가 없으면 401
- 사용자가 `DELETED`면 401

즉 삭제된 사용자가 실제 보호 API에 접근하지는 못하지만, 사용할 수 없는 새 토큰을 계속 발급받을 수는 있습니다.

**11. Redis/ElastiCache 의존성**

`AuthService`는 애플리케이션 공용 Redis 클라이언트를 주입받습니다.

- 로컬: `REDIS_URL`
- 운영 IAM 모드: ElastiCache에 TLS + IAM 인증
- 테스트: `fakeredis`

Redis 장애 시 refresh, logout, 최초 로그인 발급 과정이 정상 완료되지 않습니다. 애플리케이션 시작 때도 Redis `PING`을 실행하므로 시작 시점부터 연결되지 않으면 서버가 기동하지 않습니다.

</details>

#### 현재의 방식 vs opaque token

- 현재 JWT는 payload의 jti 필드에 랜덤 16진수를 넣고 서명을 통해 변조여부를 검사한다. 형식은 JWT이나 변형 opaque token에 가깝다.
- 변조 여부를 검사 하는게 의미가 있는지 살펴본다.
    - 오버헤드 면
        - 토큰을 변조하거나 invalid 토큰 → DB까지 안 간다. 바로 받자마자 실패 처리된다.
        - Access Token에서의 JWT의 역할: 모든 요청에 대해 검사하며 API Gateway의 JWT Authorizer에서 서명, 유효일자, audience, scope 검사 후 1차 인가, 애플리케이션 서버에서 데이터베이스를 조회하며 2차(최종) 인가가 이루어졌었다.
            - 1차 인가에서 별도 데이터베이스 조회 없이 인가를 수행하기 위해 자체 검증이 가능한 JWT 형식을 사용했었다.
        - Refresh Token의 경우엔 API Gateway에서의 검증 대신 특정 엔드포인트(`POST /api/v1/auth/refresh`, `POST /api/v1/auth/logout`) 에서만 사용되며, 게이트웨이에서의 검사는 없다.
            - 일반 API 콜보다 많이 호출될 것 같진 않다.
        - 따라서 오버헤드 면에서보면, **Access Token과 달리 Refresh Token일 때 JWT와 Opaque 방식은 크게 차이가 없다.**
    - 보안성 면
        - 현재의 jti 필드는 `uuid4().hex` 를 통해 만든다.
            - 내부적으로 `os.urandom(16)` 을 호출하므로, 128비트짜리 길이의 random hex string이 된다.
            - 서명을 생성하는 경우, SHA-256 → 반드시 256 비트.
        - 다른 사용자의 권한을 얻기 위해선 jti 필드의 128비트 또는 opaque token 값을 찍어서 맞춰야 한다.
            - 따라서 JWT를 쓰든 opaque token을 쓰든 이 부분(탈취 위험성, 추측 가능성 면)에선 같다.
            - 어차피 128비트를 찍어서 맞추기 전에 rate limit이나 CloudFlare DDoS Protection에 걸리게 된다. 해당 설계는 추후 진행 예정
        - 따라서 보안성 면에서 보면, JWT와 Opaque 방식은 큰 차이가 없다.
    - 결론
        - Opaque token을 사용하는 것으로 바꾼다.
        - 기존의 JWT_SECRET은 이제 Access Token에서는 대신 KMS CMK를 사용하며, Refresh token은 이제 별도의 키가 필요하지 않으므로, 시원하게 삭제한다.

#### opaque token 검증: redis 대신 DynamoDB TTL

- 기존에는 JWT의 jti 값을 redis를 통해 검사 후 TTL 설정을 통해 만료시켰다.
    - ElastiCache Serverless(Valkey) 의 경우, 100MB 이하라도 1달에 만원 가까이 과금된다.
    - 현재 Valkey(redis)를 사용하는 이유는 성능이 아닌 TTL에 가깝다.
    - 따라서 refresh token의 검증은 dynamoDB를 통해 처리하는 것이 적절하다.
- DynamoDB 또한 TTL을 지원한다.
    - 다만 background job이 주기적으로 스캔하여 만료된 항목을 삭제하는 방식이며, 48시간까지 늦어질 수 있다.
    - 따라서 만료 여부 판단에 사용하면 안 되고, 만료 레코드 정리 용도로만 사용해야 한다.

### 요약

- HTTP 타입 API Gateway에서 JWT Authorizer로 1차 인가 / API 서버에서 데이터베이스까지 들어가는 2차 인가
- 기존 RDS의 user 테이블은 dynamoDB가 담당한다.
- 기존 Redis의 OAuth state 재사용 방지는 dynamoDB가 담당한다.
- 기존 Redis의 refresh token 검증은 dynamoDB가 담당한다.
