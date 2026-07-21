---
title: "OAuth2와 OIDC (1) - OAuth의 기본 개념과 인증 플로우"
date: 2026-07-21 19:12:19 +0900
categories: [CS]
tags: [OAuth, OIDC, Auth, Security]
---

### OAuth가 왜 필요해졌나?

- OAuth 전, 원래 한 서비스가 **다른 서비스의 데이터**에 접근하려면?
    - 아이디와 비밀번호를 대신 받아서 로그인 하는 방식이었음
    - ex: 어떤 사진 인쇄서비스가 사용자 인스타 사진 가져오기 → 그 서비스에다가 인스타 비번을 쳐야만 했음
- 이 방식이 뭐가 문제가 있나?
    - 내가 넘기는 권한을 통제할 수 없음
        - 비밀번호만 넘기면 이 서비스가 접근해서 프사 변경 계정 삭제 남한테 메세지 보내기 다 가능
        - 사진 나열하기, 사진 접근 권한만 있으면 좋을 텐데..
    - 권한 회수 방법이 모호. 권한 회수하려면 비번 바꿔야 함
    - 서드파티가 내 특정 서비스 계정 비번을 보관 → 보안상 좀..

### OAuth의 등장

- 이걸 해결하기 위한 표준이 생김: OAuth 1.0을 거쳐 OAuth 2.0이 2012년에 표준이 정의됨
    - OAuth 2.0은 프로토콜이 아닌 "Framework" 이다.
        - 명세 자체에서 끝난다. 구현 수준의 디테일은 없다.
        - 따라서 OAuth 자체가 여러 구현 선택지를 주며, 그 위에 실제 구현체가 올라간 대표 예시가 OIDC다.

    > **Framework**가 무엇인가?
    >
    > 1. 그냥 딱 Framework라 생각하면 Spring Framework에서의 Framework가 생각난다.
    이건 소프트웨어 수준의 Framework이며, 뼈대(frame)은 마련되어 있으며 개발자가 작성해야 하는 곳에 코드를 작성하는 방식이었다.
    비교 대상은 Library, 이건 개발자가 모든 실행 흐름을 설계하고 구현해야 하며 유틸리티처럼 동작하는 것.
    > 2. OAuth 2.0은 명세이자 표준이며,
    여기에서의 "프레임워크"는 Spring Framework와 다른 층위의, **명세 수준의 Framework**를 뜻한다.
    골격은 제공하나 **세부 인증절차의 구현**은 내가 해야 한다는 뜻이다.
    > 3. 비교 대상은 프로토콜로, 프로토콜은 따르면 끝이다.
    구현 디테일까지 다 정해 준 것, 진짜 코드로 옮기면 끝나는 것이다.
    e.g. HTTP, TLS
    {: .prompt-info }

- 우리가 구글/카카오톡 로그인으로 특정 사이트에 접속하는 건 OAuth가 아닌 그 위에 올라가는 OIDC다.
    - OAuth는 본래 "서비스 안 접근 권한"을 다른 애플리케이션에게 위임하기 위한 것. 사용자의 신원 확인은 OAuth가 하는 일이 아님.
    - 명확하게 구분해야 할 필요가 있으며,

        | 구분 | 인증 Authentication | 인가 Authorization |
        | --- | --- | --- |
        | 개념 | 지금 얘 이 사람 맞네<br>신원 확인 | 이 사람은 뭐가 가능하냐?<br>에 따라 적절한 권한을 주기 |
        | 예시 | 비밀번호 입력해서 확인 | 결제 API 호출권한 부여, 사진 목록 접근 권한 부여 |
        | 표준 | OpenID Connect, SAML | OAuth 2.0 |

### OAuth 기초, 용어 정리(4개 역할, 토큰 종류, scope, 클라이언트 종류)

- 4개의 역할이 있다.
    - **Resource Owner**
        - 권한 위임을 승인하는 주체 = **사용자**.
    - **Client**
        - 권한을 위임받으려는 주체 = 서드파티 서비스
    - **Authorization Server**
        - Resource Owner가 맞음을 인증
        - Resource Owner에게서 Client에게 권한을 부여함 대해 동의받음
        - Client에게 토큰 발급
        - ex: `accounts.google.com`
    - **Resource Server**
        - Authorization Server에서 받은 토큰에 대해, 해당 권한을 사용하는 곳
        - ex: `gmail.googleapis.com`
- AS와 RS는 같은 회사가 운영 가능
    - 하지만 OAuth 명세에서는 논리적으로 명확히 분리되는 단위임.
- OAuth에서 발급하는 토큰 3가지
→ Access/Refresh 토큰에 Authorization Code가 추가됨
    - **Authorization Code**
        - Access Token을 받기 위한 일회성 교환용 코드
        - 수명이 제일 짧음(30~600초), 1회용
        - 브라우저에게 노출됨
    - **Access Token**
        - Resource Server에 리소스 접근 시 사용
        - 5분~1시간 정도의 사용 기간
        - Client, Resource Server에게만 노출
    - **Refresh Token**
        - Access token 만료 시 재발급을 위한 용도
        - 며칠에서 수 개월 정도의 가장 긴 수명
        - Client에게만 노출
- **Scope**
    - Client가 요청하는 **권한의 범위를 표현**하는 문자열
    - client의 **최소 권한 원칙 준수** 필요
    - 예시

        ```
        scope=read:email write:calendar
        ```

    - scope에 적혀있는 권한을 Resource Server가 실제로 접근해도 괜찮을지 검증한다.

> 참고로, OAuth에서의 권한의 흐름은
>
> - **권한을 사용자에게 최소한으로 요청하는 건 client의 책임**
> - 권한 최소한으로 허용/거부하는 건 사용자의 책임
> - AS의 책임:
>     - 사전에 신고한 scope 만큼만 요청하는지 검증
>     - client가 최소한으로 요청한 권한을 사용자에게 보여주고 동의 구함
>     - 최종 허용한 만큼만 권한을 부여
{: .prompt-info }

- **Confidential Client vs Public Client**
    - AS가 준 client_secret을 안전하게 보관 가능 여부에 따라 구분

    | 구분 | Confidential Client | Public Client |
    | --- | --- | --- |
    | 예시 | 백엔드 서버<br>server side rendering 웹앱 | SPA, 모바일 네이티브 앱, 데스크톱 앱 |
    | 인증 | client_secret, mTLS, JWT 등으로 가능 | 불가능 또는 매우 제한적 |
    | 권장 흐름 | Authorization Code Grant | Authorization Code Grant<br>+ PKCE |

    - SPA와 모바일 앱은 코드가 브라우저에 다운로드됨 → 디컴파일 시 문자열이 노출됨
    - 이런 환경에서 client_secret을 알게 되면 Client를 공격자가 위조 가능, 따라서 비밀에 의존하지 않도록 PKCE를 사용

### OAuth의 Grant Type: 사용자의 허가부터 Client가 Access/Refresh 토큰을 받기까지의 과정

- **Authorization Code Grant(Confidential Client 전용이었던 방식)**
    - **OAuth 2.1에서: 클라이언트 뭐든 새 시스템에서는 PKCE 써라**
    - 백엔드 서버를 가진 전통적인 웹 애플리케이션에서의 사용이 대표적
    - 진행 단계

        ![](/assets/img/oauth2와-oidc-1-oauth의-기본-개념과-인증-플로우/image.png)

        1. Client가 사용자의 브라우저를 AS의 `/authorize` 엔드포인트로 리다이렉트
        2. AS는 사용자에게 로그인 화면을 보여주고, 어떤 권한을 어떤 클라이언트에 줄지 동의(consent)를 받는다
        3. AS는 사전에 등록된 `redirect_uri`로 브라우저를 다시 리다이렉트하면서 인가 코드(authorization code)를 쿼리 파라미터로 전달
        4. 브라우저가 Client의 백엔드로 authorization code를 전달
        5. 이제 Client가 자신의 `client_secret`과 함께 authorization code를 AS의 `/token` 엔드포인트에 보냄
        6. AS가 코드를 검증하고 Access Token (선택적으로 Refresh Token)을 응답
        7. Client가 Access Token으로 Resource Server에 API 요청
    - HTTP payload 예시

        (1) Authorization Request

        ```
        GET /authorize?
            response_type=code
            &client_id=s6BhdRkqt3
            &redirect_uri=https%3A%2F%2Fclient.example.com%2Fcb
            &scope=read%3Aemail
            &state=xyzABC123
        HTTP/1.1
        Host: authorization_server.example.com
        ```

        (5) Token Request

        ```
        POST /token HTTP/1.1
        Host: as.example.com
        Content-Type: application/x-www-form-urlencoded
        Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW

        grant_type=authorization_code
        &code=SplxlOBeZQQYbYS6WxSbIA
        &redirect_uri=https%3A%2F%2Fclient.example.com%2Fcb
        ```

        - Client Secret이 대체 어디에서 전달되고 있는 거지?

            ```
            Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
            # = Basic base64("s6BhdRkqt3:gX1fBat3bV")
            # = Basic base64(client_id + ":" + client_secret)
            # 이게 client_secret_basic 방식의 클라이언트 인증
            ```

        (6) Token Response

        ```
        HTTP/1.1 200 OK
        Content-Type: application/json
        Cache-Control: no-store
        Pragma: no-cache

        {
          "access_token": "2YotnFZFEjr1zCsicMWpAA",
          "token_type": "Bearer",
          "expires_in": 3600,
          "refresh_token": "tGzv3JOkF0XG5Qx2TlKWIA",
          "scope": "read:email"
        }
        ```

    - 특징
        - 브라우저에는 Access Token이 노출되지 않으며 only **인증코드만 노출**되게 됨(코드는 일회용, 수명 짧음)
        - **Confidential Client 전용** 인증 플로우
            - 인증코드와 client_secret이 있어야 토큰을 받을 수 있으므로 공격자가 인증코드를 어찌저찌 얻어도 교환 불가능
        - 추후 다룰 **state** 파라미터로 CSRF를 방지
        - 한계점: Public Client 처럼 **client_secret를 안전하게 보관 불가능한 경우**, 공격자가 인증코드 + client_secret 모두를 알게 되어 Access/Refresh Token을 탈취당할 수 있음.

- **Authorization Code Grant with PKCE(Public Client, OAuth 2.1부터는 모든 Client, 신규 시스템 대상 권장)**
    - SPA, 모바일 앱, 데스크톱 앱 등에서 client_secret가 유출될 수 있다.
    - client_secret이 유출된 상황에서 Authorization Code를 공격자가 얻게 되면 Access/Refresh Token도 얻을 수 있다.
    - 이런 경우를 위한 인증 플로우가 **Authorization Code Grant with PKCE** 다.
        - PKCE: Proof Key for Code Exchange.
        - 매 인증 플로우마다 일회성 비밀인 `code_verifier`를 만들며, 코드교환시 증명하게 된다.
    - 인증 플로우
        - 모바일 앱 같은 경우엔 client_secret이 유출될 수 있으므로 아예 사용하지 않는다.
        - 웹 서버 같은 경우엔 client_secret도 사용한다.

        ![](/assets/img/oauth2와-oidc-1-oauth의-기본-개념과-인증-플로우/image-1.png)

        1. Client가 무작위 문자열을 생성하며 이게 `code_verifier`가 된다.
            - `code_verifier`는 43~128자 길이이며 `base64url-safe` 하다
        2. **`code_verifier`를 SHA-256으로 해시한 뒤 base64url 인코딩**
            - 이게 `code_challenge`가 됨(method=S256)
        3. Authorization Request(Client가 사용자를 AS의 authorization endpoint로 이동시킬 때)에서 `code_challenge`와 `code_challenge_method=S256` 을 함께 보냄, AS가 이 값을 코드와 묶어 저장함
        4. Token Request 시에 AS가 클라이언트를 리다이렉트하여 받은 Authorization Code(= `code`)와 자기 메모리에 저장돼 있던 `code_verifier` 원본을 함께 AS의 token endpoint에 POST 요청으로 보냄
        5. AS는 받은 `code_verifier`를 SHA-256 해시, 이전에 저장한 `code_challenge`와 일치 여부 확인
            - code_verifier는 일회용, 이후에 다른 누군가 보내면 요청 거부 및 추가로 발급된 토큰 무효화 조치 시행 가능
    - 그럼 이 방법의 보안성은?
        - 공격자는 Public Client가 보낸 **code_challenge를 탈취**
        - 이건 Authorization Code Grant에서 client_secret이 유출된 것과 대응되는데
            - Authorization Code Grant는 client_secret이 유출되면?
            - client_secret을 authorization code와 AS에게 보내서 access token을 받을 수 있음
        - 공격자는 code_challenge와 인증코드를 탈취 후,
            - code_challenge에게서 code_verifier를 알아내서 AS에게 보내면 토큰을 얻을 수 있음
            - 근데 SHA-256 해싱된 값이라 절대 알아낼 수 없음
        - code_verifier는 Client에게서 Authorization Server로 최종 토큰을 얻을 때 한 번 보냄
            - 이전에는 네트워크에 노출되지 않으므로 외부 공격자일 경우엔 절대 알수 없음
            - 물론 이 때도 TLS로 L7 내용은 암호화됨(OAuth 2.0/2.1 스펙 상 token endpoint는 반드시 HTTPS 여야 함)
        - "TLS는 반드시 완벽하게 작동함" 을 가정함
            - TLS가 깨지면 오가는 client_secret, access_token을 공격자가 모두 볼 수 있음
            - 따라서 TLS가 깨지면 OAuth의 모든 보안은 의미가 없어짐
    - HTTP 페이로드 예시

        (1) Authorization Request

        ```
        GET /authorize?
            response_type=code
            &client_id=spa-client-123
            &redirect_uri=https%3A%2F%2Fapp.example.com%2Fcb
            &scope=read%3Aemail
            &state=xyz
            &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
            &code_challenge_method=S256
        HTTP/1.1
        Host: as.example.com
        ```

        - method=plain 방식이면 code_challenge = code_verifier가 된다.
        - 이러면 공격자가 code_challenge를 가로채면 그대로 verifier로 사용하면 된다.
        - 따라서 이런 경우 보안성은 없으며, OAuth 2.1에서 plain은 제거되었다.

        (5) Token Request

        ```
        POST /token HTTP/1.1
        Host: as.example.com
        Content-Type: application/x-www-form-urlencoded

        grant_type=authorization_code
        &code=SplxlOBeZQQYbYS6WxSbIA
        &redirect_uri=https%3A%2F%2Fapp.example.com%2Fcb
        &client_id=spa-client-123
        &code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
        ```

- **Refresh Token Grant**: Access Token이 만료되어 재발급 받을 때

    ```
    POST /token HTTP/1.1
    Host: as.example.com
    Content-Type: application/x-www-form-urlencoded
    Authorization: Basic <base64(client_id:client_secret)>

    grant_type=refresh_token
    &refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
    &scope=read:email
    ```

    **핵심 보안 메커니즘:**

    - **Refresh Token Rotation:** Refresh Token을 사용할 때마다 새 Refresh Token을 발급하고 기존 것은 즉시 폐기. Public Client에서 강력히 권장
    - **Reuse Detection:** 폐기된(이미 쓰인) Refresh Token이 다시 사용되는 것이 감지되면, 해당 사용자의 *모든* Refresh Token 패밀리를 무효화. (Refresh Token이 유출이 의심되는 상황이므로)

    ```
    시나리오: Refresh Token RT1이 공격자에게 유출됨

    t=0:  합법적 Client가 RT1으로 토큰 갱신 → AT2, RT2 받음 (RT1 폐기)
    t=1:  공격자가 가로챈 RT1으로 토큰 갱신 시도
            → AS가 "이미 사용된 RT" 감지
            → AS가 RT2까지 모두 무효화
            → 사용자 재로그인 강제, 침해 사실 인지
    ```

    이 메커니즘 덕에 SPA처럼 Refresh Token 보관이 어려운 환경에서도 안전하게 쓸수 있다?

- 기타 Grant Type
    - Client Credentials Grant: 사용자가 끼지 않는, *Client 자체의 권한*으로 리소스에 접근할 때
    - Device Authorization Grant: 사용자 입력이 어렵거나 브라우저가 없는 스마트TV 게임콘솔 IoT CLI도구 등에서 인증할때

### OAuth의 토큰(Opaque/JWT)

- Access Token의 형식은 OAuth2 명세에 정의되어 있지 않음 ⇒ 합의하는 대로 사용 가능.
- 그래서 Opaque Token 또는 JWT를 쓸 수 있음.
- **Opaque Token**
    - 무작위 문자열로 AS가 내부 DB에 매핑정보를 저장함
    - RS는 토큰을 받고 AS랑 통신하여 유효성과 권한을 검증
    - 장점: 즉시 폐기 가능하고, 토큰 안에 정보가 없다
    - 단점: AS 호출에 따른 부하(서버 부하, 네트워크 통신으로 부하와 지연)
- **JWT**
    - 토큰 자체에 정보가 있음
    - 서명을 통해 무결성을 보장하는 구조
    - 장점: AS한테 물어볼 필요 없음
    - 단점: 발급 후 즉시 폐기하기 어려움, 상대적으로 페이로드는 크다
