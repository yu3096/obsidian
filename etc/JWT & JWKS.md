## JWT와 JWKS 개념 정리

### 1. 개요
본 문서는 분산 환경에서 상태 비저장(Stateless) 인증을 구현하기 위한 JWT(JSON Web Token)와 JWKS(JSON Web Key Set)의 핵심 개념 및 검증 흐름을 정리한 기술 문서입니다. 
단순한 이론 정리를 넘어, 실제 디즈니(Disney) 연동 프로젝트를 진행하며 직면했던 키 관리 및 캐싱 전략의 미흡했던 점을 회고합니다. 
이를 바탕으로 안전한 Key Rotation 운영 절차와 순수 Java 기반의 견고한 JWT 검증 및 JWKS 연동 로직 구현 방법을 다룹니다.

---
### 2. JWT(JSON Web Token)란 무엇인가
#### 2.1 정의
JWT(JSON Web Token)는 **서버가 상태를 저장하지 않고(Stateless) 사용자 인증 및 권한정보를 전달하기 위한 토큰 기반 인증 방식**.
- [RFC 7519 표준](https://www.rfc-editor.org/rfc/rfc7519.html#:~:text=JSON%20Web%20Token%20(JWT)%20is%20a%20compact%2C,claims%20to%20be%20transferred%20between%20two%20parties.)
- OAuth2, OpenID Connect에서 널리 사용
- 인증 서버가 발급
- 클라이언트가 요청 시마다 서버에 전달

주 사용처:
- 로그인 후 Access Token
- OAuth2 / OpenID Connect(OIDC)에서 ID Token / Access Token
- 서비스 간 호출(마이크로서비스)에서 "누가 호출했는지" 전달
> 핵심: **JWT는 암호화가 아니라 기본적으로 "서명된 데이터"** (암호화는 JWE 영역)

---
### 3. JWT의 구조
JWT는 3개의 영역으로 구성됩니다.
``` ln:false
Base64Url(Header).Base64Url(Payload).Base64Url(Signature)
```
예시:
``` ln:false
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTYiLCJyb2xlIjoiVVNFUiJ9.
XxXxXxXxXxXx
```
#### 3.1 Header
토큰 타입과 서명 알고리즘, 종종 `kid`(키 id)가 들어감
예: 
```json ln:false
{
    "alg": "RS256",
    "typ": "JWT",
    "kid": "key-id-001"
}
```
- `alg`: 서명 알고리즘 (HS256, RS256, ES256 등)
- `kid`: 어떤 키로 서명했는지 식별(키 로테이션에서 중요)
---
#### 3.2 Payload(Claims)
사용자/권한/만료시간 등 "토큰이 주장하는 내용(클레임)"이 들어갑니다.
```json ln:false
{
    "sub": "uracle",
    "iss": "auth-server",
    "aud": "my-api",
    "exp": 1710000000,
    "iat": 1759996400,
    "scope": "read:orders write:orders",
    "role": ["admin"]
}
```

##### Registered Claims(표준 클레임) 자주 쓰는 것들
- `iss`(Issuer): 발급자
- `sub`(Subject): 토큰 주체(보통 유저ID)
- `aud`(Audience): 토큰 대상(이 토큰을 받을 API)
- `exp`(Expiration): 만료 시각
- `nbf`(Not Before): 이 시각 전에는 무효
- `iat`(Issued At): 발급 시각
- `jti`(JWT ID): 토큰 고유 ID(리플레이 방지/블랙리스트에 활용)

---
#### 3.3 Signature
Header + Payload를 기반으로 생성
``` ln:false
Signature = Sign(Base64Url(Header) + "." + Base64Url(Payload), Secret or PrivateKey)
```
##### 서명 방식의 큰 분류: 대칭키(HMAC) vs 비대칭키(RSA/ECDSA/EdDSA)
JWT에서 가장 많이 쓰는 알고리즘은 아래 두 분류.
**A. 대칭키 방식(HMAC 계열)**
- 서명/검증에 같은 비밀키(secret) 사용
- 예: `HS256`, `HS384`, `HS512`
**B. 비대칭키 방식(Public Key 계열)**
- 서명: 개인키(private key)
- 검증: 공개키(public key)
- 예: `RS256/384/512`(RSA), `ES256/384/512`(ECDSA), `EdDSA`(Ed25519 등)

##### 서명 알고리즘 혼동 공격(Algorithm Confusion Attack)
JWT 검증 시 주의해야 할 대표적인 보안 이슈가 **alg 혼동 공격**입니다.

**공격 개념**
JWT Header에는 다음과 같이 `alg`가 포함됩니다.
```json ln:false
{
    "alg": "RS256"
}
```
문제는 서버가 이 `alg`값을 그대로 신뢰한다면 공격이 가능해집니다.
예를 들어:
1. 서버는 원래 RS256(비대칭키) 검증을 해야 함
2. 공격자가 JWT의 `alg`를 HS256으로 변경
3. 서버가 공개키를 HMAC secret처럼 사용해 검증
4. 공격자가 위조한 토큰이 검증 통과

**대응 원칙**
- 서버는 JWT Header의 `alg`를 그대로 신뢰하면 안 됨
- 허용할 알고리즘을 서버 설정에서 고정(화이트리스트)
- 예상하지 않은 알고리즘은 즉시 거부
>JWT 검증은 "토큰이 말하는 방식"이 아닌 "서버가 허용한 방식"으로 검증해야 한다.

---
### 4. JWT의 동작 방식
#### 4.1 인증 흐름
1. 사용자가 로그인 요청
2. 인증 서버가 JWT 발급
3. 클라이언트가 JWT 저장
4. 요청 시 Authorization 헤더에 포함
``` ln:false
Authorization: Bearer <JWT>
```
5. 서버는 서명 검증 후 요청 처리
##### JWT 검증 시 반드시 확인해야 할 것
- `exp`(만료)
- `iss`(발급자)
- `aud`(대상서비스)
- 허용된 `alg`만 허용
- 필요 시 `nbf`(Not Before), `iat`(발급 시각)
- 권한(`scope`, `role`)

---
#### 4.2 핵심 특징
- 서버가 세션을 저장하지 않음
- 토큰 자체에 인증 정보 포함
- 서명 검증만으로 신뢰성 확보
---
### 5. JWT의 장점
#### 5.1 Stateless 인증 가능
 - 세션 저장 불필요
 - 서버 확장에 유리
#### 5.2 빠른 처리
- DB 조회 없이 서명 검증만 수행
#### 5.3 마이크로서비스에 적합
- 여러 서비스 간 인증 공유 유리
#### 5.4 표준 기반
- 다양한 라이브러리 지원
- OAuth2, OIDC와 자연스럽게 통합
---
### 6. JWT의 한계
JWT 도입 이후 실제 운영 환경에서 발생 한 문제
#### 6.1 토큰 폐기(Revocation) 어려움
- 탈취 시 만료 전까지 무효화 어려움
- 블랙리스트 관리 시 다시 상태관리 필요
#### 6.2 토큰 크기 증가
- Claim 증가 시, 네트워크 부담 증가
#### 6.3 키 관리 문제(가장 중요)
JWT를 안전하게 사용하기 위해 RSA/ECDSA같은 비대칭키 방식이 많이 사용됩니다.
여기서 문제가 발생!

---
### 7. JWT 키 관리에서 발생 한 문제
#### 7.1 공개키 배포 문제
구조: 
- 인증서버 -> 개인키로 서명
- 여러 리소스 서버 -> 공개키로 검증
문제:
- 공개키를 어떻게 각 서비스에 배포 할 것인가?
- 키 변경 시, 모든 서버를 다시 배포해야 하는가?
#### 7.2 Key Rotation 문제
보안상 정기적으로 키 교체가 되어야 함.
그러나,
- 기존 토큰은 이전 키로 서명됨
- 새 키도 동시에 운영해야 함
- 여러 공개키를 관리해야 함
---
### 8. JWKS(JSON Web Key Set)가 등장 한 배경
위 문제를 해결하기 위해 JWKS(JSON Web Key Set)이 등장
#### JWK와 JWKS 차이
- JWK(JSON Web Key): 키 1개를 표현한 JSON 객체
- JWKS(JSON Web Key Set): 여러 JWK를 `keys[]` 배열로 묶은 문서
즉, 
- JWK = 키 하나
- JWKS = 키 여러 개 모음

---
### 9. JWKS란 무엇인가
#### 9.1 정의
JWKS는 **JWT 서명 검증에 필요한 공개키들을 JSON 형식으로 제공하는 표준**
보통 다음과 같은 URL로 제공됨.
``` ln:false
https://auth-server/.well-known/jwks.json
```
예시:
```json ln:false
{
    "keys": [
        {
            "kty": "RSA",
            "kid": "key-id-001",
            "use": "sig",
            "alg": "RS256",
            "n": "...",
            "e": "AQAB"
        },
        {
            "kty": "RSA",
            "kid": "key-id-002",
            "use": "sig",
            "alg": "RS256",
            "n": "...",
            "e": "AQAB"
        },
    ]
}
```

#### 9.2 RSA n/e 필드 인코딩 주의 사항(실무에서 자주 발생하는 오류)
JWKS에서 RSA 공개키는 다음과 같은 형태로 표현된다.
```json ln:false
{  
    "kty": "RSA",  
    "kid": "key-id-001",  
    "use": "sig",  
    "alg": "RS256",  
    "n": "....",  
    "e": "AQAB"  
}
```
여기에서 중요한 필드는
- `n` ->  RSA의 modulus (큰 정수값)
- `e` -> 공개 지수(보통 65537 -> AQAB)

#### 9.2.1 RSA 키 길이(비트 수) 문제
실제 외부 연동 시 다음과 같은 피드백을 받을 수 있습니다.

> [!디즈니와 연동할 때 CBS에서 받은 피드백]
    > SKB public key 설정이 제대로 설정되지 않을 것 같습니다.
    > RSA modulus (n값) 이 2352 비트로 확인되지만, 표준 RSA사이즈가 아니며, 스탠다드 2048비트로 요청드립니다.
    > 예)
    > {
    >     "kty": "RSA",
    >     "e": "AQAB",
    >     "use": "sig",
    >     "kid": "83c76126-65a2-4cee-8b1a-048059097ce2",
    >     "alg": "RS256",
    >     "n": "MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAglFAosU1o2Zu0LDlhPJJcWe4pNn62zGQgAmEx8c8Wq564CIjzIpxOZeBJhOdlM/rLTABC69g8WmyuEHcYPaeZRNYm7guZ4OOQ4KAEUl6xV0yUwZMXr0jZ7zDvE32dhzS+bh4f7WzC5ofjciUz4eoWra6frrhYC+uTXDlXT4m8wppeLF51G/ERWz6qRyu9kbohj/b0FbK1OJd9Jpjy6qgMvX/ERkY7H4ZFBWSvezFMaYrCKfFE82SNC8CxW0dwrsbmhJtS3lto5M+syCmho0y5eisxiOEBc1E8x1LqlvD4BKjYcvN9gYY+yGi+6EnirbvJcc/n+sZcUU05dBuJfKXaQIDAQAB"
    > }

##### 뭔 소리나면
RSA 키는 "몇 비트짜리 키인지"가 중요함.
보통 표준은
- 2048비트 (가장 일반적이고 표준)
- 3072비트
- 4096비트
그런데 2352비트처럼 애매한 길이는 일부 시스템에서 지원하지 않을 수 있음.

##### 문제가 되는 이유
- 외부 시스템이 2048비트만 허용
- 키 복원 시 오류 발생
- 검증 실패
##### 해결 방법
키 생성 시 2048비트로 명시
```java ln:false
KeyPairGenerator generator = KeyPairGenerator.getInstance("RSA");
generator.initialize(2048);  // 반드시 2048로 명시
KeyPair keyPair = generator.generateKeyPair();
```

---
### 9.2.2 DER 인코딩 vs Raw Integer 문제
실무에서 더 자주 발생하는 문제.
> [!디즈니와 연동할 때 CBS에서 받은 피드백]
    > DER encording 이 아닌 n 값은 raw interger byte 형태여야 합니다.

##### 뭔 소리냐면
RSA 공개키는 내부적으로 아래와 같은 구조를 가짐
```planetext ln:false
PublicKey = (modulus n, exponent e)
```
JWKS는 **이 중에서 n과 e값만 필요**함.

그런데 개발자가 이렇게 값을 넣는 실수가 많음.
```java ln:false
KeyPair.getPublic().getEncoded()
```
이 값은 

---
### 10. JWKS가 해결한 문제
#### 10.1 공개키 중앙 관리
- 인증서버가 JWKS endpoint 제공
- 리소스 서버는 필요 시 해당 endpoint에서 키 조회
- 수동 배포 불필요
---
#### 10.2 Key Rotation 지원
JWT Header에는 `kid`가 포함 됨.
```json ln:false
{
    "alg": "RS256",
    "kid": "key-id-002"
}
```
**검증 과정:**
1. JWT의 kid 확인
2. JWKS에서 동일한 kid 찾기
3. 해당 공개키로 검증
-> 여러 키를 동시에 운영 가능
---
#### 10.3 분산 시스템에서 안정적 운영
- 마이크로서비스 환경
- 외부 파트너 연동
- SaaS 멀티테넌트 구조
- Oauth2 / OIDC 환경
에서 표준으로 자리 잡음
---
### 11. JWT + JWKS 통합 검증 흐름
리소스 서버는 JWT를 수신했을 때 단순히 서명만 검증하는 것이 아니라, 다음과 같은 단계로 통합 검증을 수행한다.
#### 11.1 검증 단계
1. Authorization 헤더 확인
	- `Authorization: Bearer <JWT>` 형식인지 확인
2. JWT 분해
	- Header, Payload, Signature 추출
3. Header 검증
	- `alg`확인 (허용된 알고리즘인지 검증)
	- `kid`확인
4. 공개키 조회
	- 캐시에 `kid`에 해당하는 키가 있는지 확인
	- 없으면 JWKS endpoint 재조회
5. 공개키 생성
	- JWKS의 JWK를 기반으로 공개키 객체 구성
6. 서명(Signature) 검증
	- `Base64Url(Header) + "." +  Base64Url(Payload)`를 기준으로 검증 수행
7. Claim 검증
	- `exp` 만료여부
	- `iss` 신뢰 가능한 발급자인지
	- `aud` 이 서비스 대상 토큰인지
	- 필요 시 `nbf`, `iat`
8. 권한 검증
	- `scope`, `role`등을 기반으로 접근 권한 확인
9. 요청 처리
	- 모든 검증이 통과한 경우에 비즈니스 로직 수행

#### 11.2 검증 핵심 원칙
- JWT Header의 `alg`를 그대로 신뢰하지 않는다.
- 허용된 알고리즘만 서버 설정에서 고정한다.
- `iss`와 `aud`는 반드시 검증한다
- 서명 검증과 Claim검증은 모두 필수이다.
>JWT 검증은 단순한 "서명 확인"이 아니라,.
>"서명 + 정책 기반 Claim 검증"의 조합이다.

---
### 12. 전체 흐름 정리
```
세션 기반 인증의 확장성 한계
JWT 도입 (Stateless 인증)
RSA 기반 서명 사용 증가
공개키 배포 및 로테이션 문제 발생
JWKS 등장 (공개키 중앙 관리 표준)
```
---
### 13. 운영에서 가장 중요한 포인트: JWKS 캐싱
JWKS를 요청마다 원격 호출하면
- 네트워크 비용 증가
- 인증 서버 장애 시 API 전체 장애로 확산 가능
그래서 리소스 서버는 일반적으로 아래와 같이 처리
#### 13.1 캐싱 기본 전략
- JWKS를 메모리/로컬 캐시에 저장
- TTL 기반 갱신(예: 5분 ~ 1시간, 환경에 맞게)
- HTTP 캐시 헤더(Cache-Control, ETag 등)를 활용 가능하면 활용
#### 13.2 kid 미스매치 시 즉시 리프레시
토큰의 `kid`가 캐시에 없다면
- "키가 로테이션 됐을 가능성"이 큼
- 이때는 TTL을 기다리지 말고 **JWKS를 즉시 재조회** 하고 다시 검증 시도
#### 13.3 장애/지연 대응
- JWKS 조회 실패 시:
	- 캐시에 이전 키가 있으면 그걸로 검증 시도("stale-while-revalidate" 느낌)
	- 완전 실패면 401/503 정책을 명확히
- 너무 공격적으로 재조회 하지 않도록 **백오프/레이트리밋** 고려
---
### 14. Disney와의 연동에서 미흡했던 점
#### CBS에서 미흡한 점
- 캐싱 처리 안함
#### Disney에서 미흡한 점
- 내용 보면 kid가 없으면 즉시 pubkey 한번 갱신하라는데 안함
---

### 15. Key Rotation 운영 절차
JWT를 비대칭키(RSA/ECDSA/EdDSA) 기반으로 운영하는 경우,
보안 강화를 위해 정기적인 Key Rotation이 필요하다.

Key Rotation은 단순히 키를 교체하는 작업이 아니라,
**서비스 중단 없이 안전하게 키를 전환하는 운영 절차**이다.

---
#### 15.1 Key Rotation이 필요한 이유
- 키 장기 사용 시 유출 위험 증가
- 보안 정책상 주기적 교체 필요
- 키 노출 사고 발생 시 즉시 교체 필요
- 암호 알고리즘/키 길이 정책 변경 대응
>키는 영구적으로 사용하는 자산이 아니라, 주기적으로 교체되는 보안 자산이다.

---
#### 15.2 Key Rotation 기본 원칙
- `kid`는 키를 고유하게 식별해야 하며 **절대 재사용하지 않는다**.
- 구키와 신키는 일정 기간 **동시 운영**해야 한다.
- JWKS는 항상 현재 사용 중인 모든 공개키를 포함해야 한다.
- 구키 제거는 기존 토큰 만료 이후에 수행한다.
---
#### 15.3 단계별 Key Rotation 절차
##### 1단계: 새로운 키 쌍 생성
- 새로운 Private Key / Public Key 생성
- 새로운 `kid` 부여
- 기존 키는 유지
---
##### 2단계: JWKS에 새 공개키 추가
JWKS에 새 JWK를 추가한다.
이때 기존 공개키는 제거하지 않는다.
```json ln:false
{
    "keys": [
        {"kid": "key-old", ...},
        {"kid": "key-new", ...}
    ]
}
```
-> 리소스 서버는 두 키 모두 검증 가능 상태

---
##### 3단계: 서명 키 전환
인증 서버가 새로 발급하는 JWT부터는
새 Private Key(`kid=key-new`)로 서명하도록 변경한다.
기존 토큰은 여전히 `key-old`로 검증 가능해야 한다.

---
##### 4단계: 동시 운영 기간 유지
- Access Token 최대 만료 시간 동안 유지
- 모든 기존 토큰이 만료될 때까지 구키 유지
예:
- Access Token 만료: 15분
- 안전 마진 포함하여 20분 ~ 30분 유지
---
##### 5단계: 구키 제거
- 모든 구키 기반 토큰이 만료된 것을 확인
- JWKS에서 `key-old` 제거
- 구 Private Key 폐기
---
#### 15.4 운영 시 주의사항
##### 1. kid 재사용 금지
- 동일한 `kid`를 재사용하면 캐시 충돌 및 검증 오류 발생 가능
---
##### 2. JWKS 캐시 TTL(Time to Live) 고려
- 리소스 서버가 JWKS를 캐싱 중일 수 있음
- 구키 제거 시 캐시 만료 시간 고려 필요
---
##### 3. 트래픽 적은 시간대 수행 권장
- 대규모 서비스의 경우 트래픽이 낮은 시간에 수행
---
##### 4. 강제 교체(Compromise) 시 대응
키 유출 등 보안 사고 발생 시
1. 즉시 새 키 생성
2. JWKS 갱신
3. 기존 키 즉시 제거
4. 기존 토큰 강제 만료처리
5. 클라이언트 재인증 유도
---
### 15.5 Key Rotation 시나리오 예시
``` ln:false
T0: key-old 운영 중
T1: key-new 생성 및 JWKS 추가
T2: 새 토큰은 key-new로 서명
T3: 구 토큰 만료 완료
T4: key-old 제거
```
---
### 15.6 Key Rotation 체크리스트
- [ ] 새로운 키 생성 및 안전한 저장
- [ ] 새로운 kid 생성
- [ ] JWKS에 새 공개키 추가
- [ ] 인증 서버 서명 키 전환
- [ ] 기존 토큰 만료 시간 확인
- [ ] 구키 제거
- [ ] 로그/모니터링 이상 여부 확인
---
>Key Rotation은 단일 이벤트가 아니라,
>"생성 -> 병행 운영 -> 제거"의 단계적 프로세스이다.

---
### 16. 실제 구현 방법(JWT 발급/검증 + JWKS 연동)
#### 16.1 구현 범위 정의
일반적으로 구현은 2파트로 나뉜다.
- **인증 서버(Auth Server)**: JWT 발급(서명), `kid`관리, JWKS 제공
- **리소스 서버(Resource Server/API)**: JWT 검증(서명+클레임), JWKS 캐싱/갱신
---
#### 16.2 리소스 서버 JWT 검증 로직(필수 구현)
##### 16.2.1 검증 실패 케이스 정의
- 토큰 없음/형식 오류 -> 401
- 서명 검증 실패 -> 401
- exp 만료 -> 401
- iss/aud 불일치 -> 401
- 권한(scope/role) 부족 -> 403
##### 16.2.2 공통 검증 절차(미들웨어/필터 형태)
1. `Authorization: Bearer <Token>` 파싱
2. 토큰 헤더 파싱(서명 검증 전) -> `alg`, `kid`확인
3. `alg`화이트리스트 체크(서버 설정 기반)
4. `kid`로 공개키 조회(캐시 우선)
5. 서명 검증
6. 클레임 검증(`exp`, `iss`, `aud`, 필요시 `nbf/iat`)
7. 권한 검증(`scope/role`)
8. 요청 컨텍스트에 principal/claims 저장
---
#### 16.3 JWKS 연동 구현(캐싱/갱신 포함)
##### 16.3.1 JWKS 캐싱 기본
- JWKS는 **프로세스 메모리 캐시(또는 공용 캐시)**로 저장
- TTL 기반 갱신(예: 10~60분)
- HTTP 캐시 헤더(ETag/Cache-Control) 지원되면 활용
##### 16.3.2 kid 미스 매치 시 "즉시 갱신 후 재검증"
- 캐시에 `kid`가 없으면:
	1. JWKS를 즉시 재조회
	2. 다시 `kid`검색
	3. 있으면 검증 재시도
	4. 없으면 401
##### 16.3.3 장애 대응(권장)
- JWKS 조회 실패 시:
	- 캐시에 키가 남아있으면 그 키로 검증 시도
	- 캐시도 없으면 401 또는 503(정책 선택)
- 과도한 재조회 방지:
	- 백오프/레이트리밋(예: 30초 내 재조회 1회)
---
### 17. CBS 소스


#### 17.1 DisneyJWKSRotation (키 로테이션)
```java ln:false
package com.uracle.ott.disney;

import com.uracle.ott.disney.bundle.enums.DISNEY_JWK_STATUS;
import io.jsonwebtoken.SignatureAlgorithm;

import java.math.BigInteger;
import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.interfaces.RSAPublicKey;
import java.text.SimpleDateFormat;
import java.util.*;

/**
 * [무엇을 하는 클래스인가?]
 *  - JWKS 키를 '정기적으로 교체(로테이션)'하기 위한 배치/스케줄 로직입니다.
 *
 * [키 로테이션이 왜 필요한가?]
 *  - 같은 키를 너무 오래 쓰면(혹시 유출되었을 때) 피해가 커집니다.
 *  - 그래서 일정 주기마다 새 키를 만들고, 무중단으로 전환하는 절차가 필요합니다.
 *
 * [이 코드의 로테이션 규칙(정책)]
 *  1) ACTIVE 키가 80~82일: 다음에 쓸 예비키(PENDING)를 미리 만들어 둡니다.
 *  2) ACTIVE 키가 83~89일: ACTIVE -> INACTIVE로 내리고, PENDING -> ACTIVE로 올립니다. (실제 전환)
 *  3) INACTIVE 키가 90일 이상: 더 이상 검증 유예가 필요 없다고 보고 ARCHIVE로 보냅니다.
 *
 * [초급자 주의]
 *  - 이 메서드가 동시에 2번 실행되면(서버가 2대, 스케줄 겹침 등) 중복 키 생성/상태 꼬임이 생길 수 있습니다.
 *    운영에서는 '단일 실행 보장(락)' 또는 DB 트랜잭션/조건부 업데이트로 안전장치를 두는 것을 권장합니다.
 */
public class DisneyJWKSRotation {
    /**
     * JWKS 레포지토리 인터페이스
     * 이 인터페이스는 JWKS 관련 데이터베이스 작업을 정의합니다.
     */
    private final DisneyJWKSRepository jwksRepository;

    /**
     * DisneyJWKSRotation 생성자
     *
     * @param jwksRepository JWKS 레포지토리 인스턴스
     */
    public DisneyJWKSRotation(DisneyJWKSRepository jwksRepository) {
        // 로테이션 로직에서 사용할 Repository를 주입받아 보관한다.
        this.jwksRepository = jwksRepository;
    }

    /**
     * 키 상태를 확인하고 필요한 경우 키를 회전합니다.
     * - ACTIVE 키가 80일이 되면 PENDING 키를 생성합니다.
     * - ACTIVE 키가 83일이 되면 INACTIVE로 전환하고 PENDING 키를 활성화합니다.
     * - INACTIVE 키가 90일이 되면 ARCHIVE로 이동합니다.
     *
     * @throws Exception 예외 발생 시
     */
    public void checkAndRotate() throws Exception {

        // 키 로테이션을 수행한다(예비키 생성 → 전환 → 구키 폐기까지).
        System.out.println("================== JWKS Rotation Check Start ==================");

        // 1) 오늘 날짜(YYYYMMDD)를 구한다.
        String today = getToday(); // 오늘 날짜(yyyyMMdd). DB에서 '유효한 키'를 찾을 때 기준이 됩니다.
        // 2) 현재 ACTIVE 키(서명에 쓰는 키)를 조회한다.
        HashMap<String, String> activeKey = jwksRepository.findActiveKey(today); // 현재 서명에 사용하는 ACTIVE 키(개인키 포함)를 조회합니다.

        Date staDate = null;
        long daysSinceStart = 0;

        // ACTIVE 키가 없으면 초기 상태이므로 새 키를 만든다.
        if (activeKey == null || activeKey.isEmpty()) { // ACTIVE 키가 없는 경우
            System.out.println("[INFO] No ACTIVE key found. Generating new ACTIVE key.");
            generateNewJWKey(DISNEY_JWK_STATUS.ACTIVE.getCode()); // 새로운 ACTIVE 키를 생성합니다.

        } else {
            System.out.println("[INFO] Found ACTIVE key: " + activeKey.get("KID"));

            staDate = parseDate(activeKey.get("STARTDATE"));
            daysSinceStart = daysBetween(staDate, new Date());

            System.out.println("[LOG] Checking key " + activeKey.get("KID") + " (age: " + daysSinceStart + " days)");

            if (daysSinceStart >= 80 && daysSinceStart <= 82) { // ACTIVE 키가 80~82일이 된 경우

                System.out.println("[INFO] ACTIVE Key " + activeKey.get("KID") + " is 80 days old over. Generating new key (PENDING).");

                // 3) 예비키(PENDING)가 있는지 조회한다(없으면 미리 만들어둔다).
                HashMap<String, String> pendingKey = jwksRepository.findLatestPendingKey(today); // PENDING 키를 조회합니다.

                // 예비키가 없으면 로테이션 대비를 위해 새 키(PENDING)를 생성한다.
                if (pendingKey == null || pendingKey.isEmpty()) { // PENDING 키가 없는 경우
                    System.out.println("[INFO] No PENDING key found. Creating new PENDING key.");
                    generateNewJWKey(DISNEY_JWK_STATUS.PENDING.getCode()); // 새로운 PENDING 키를 생성합니다.

                } else { // PENDING 키가 이미 존재하는 경우
                    System.out.println("[INFO] Existing PENDING key found. Skipping new key generation.");
                }

            } else if (daysSinceStart >= 83 && daysSinceStart <= 89) { // ACTIVE 키가 83~89일이 된 경우
                System.out.println("[INFO] Key " + activeKey.get("KID") + " is 83 days old over. Transitioning to INACTIVE.");

                // DB에서 키 상태를 전환한다(PENDING→ACTIVE, ACTIVE→INACTIVE 등).
                int uptCnt = jwksRepository.updateKeyStatus(activeKey.get("KID"), DISNEY_JWK_STATUS.INACTIVE.getCode()); // ACTIVE 키를 INACTIVE로 전환합니다.

                if (uptCnt > 0) { // 전환이 성공한 경우
                    System.out.println("[INFO] Key " + activeKey.get("KID") + " successfully transitioned to INACTIVE.");

                    HashMap<String, String> pendingKey = jwksRepository.findLatestPendingKey(today); // PENDING 키를 다시 조회합니다.
                    if (pendingKey != null) { // PENDING 키가 존재하는 경우
                        System.out.println("[INFO] Activating pending key: " + pendingKey.get("KID"));
                        jwksRepository.updateKeyStatus(pendingKey.get("KID"), DISNEY_JWK_STATUS.ACTIVE.getCode()); // PENDING 키를 ACTIVE로 전환합니다.
                    } else { // PENDING 키가 없는 경우 *특수한 상황으로 간주
                        System.out.println("[WARN] No pending key found to activate. Generating new ACTIVE key.");
                        generateNewJWKey(DISNEY_JWK_STATUS.ACTIVE.getCode()); // 새로운 ACTIVE 키를 생성합니다.
                    }
                } else {
                    System.out.println("[ERROR] Failed to transition key " + activeKey.get("KID") + " to INACTIVE.");
                }
            }
        }

        Map<String, String> inactiveKeysMap = jwksRepository.findInactiveKeys(); // INACTIVE 키를 조회합니다.

        if (inactiveKeysMap == null || inactiveKeysMap.isEmpty()) { // INACTIVE 키가 없는 경우
            System.out.println("[INFO] No inactive keys found for ARCHIVE check.");

        } else { // INACTIVE 키가 존재하는 경우

            String kid = (String) inactiveKeysMap.get("KID"); // INACTIVE 키의 식별자(kid)를 가져옵니다.
            String staDateStr = (String) inactiveKeysMap.get("STARTDATE"); // INACTIVE 키의 시작 날짜를 가져옵니다.

            staDate = parseDate(staDateStr); // 시작 날짜 문자열을 Date 객체로 변환합니다.
            daysSinceStart = daysBetween(staDate, new Date()); // 시작 날짜와 현재 날짜 사이의 일수를 계산합니다.

            System.out.println("[LOG] Checking inactive key " + kid + " (age: " + daysSinceStart + " days)");

            if (daysSinceStart >= 90) { // INACTIVE 키가 90일이 된 경우
                System.out.println("[INFO] Inactive key " + kid + " reached 90 days. Archiving and clearing key material.");
                // 유예 기간이 끝난 INACTIVE 키는 ARCHIVE로 폐기한다.
                jwksRepository.archiveKey(kid); // INACTIVE 키를 ARCHIVE로 전환하고 키 자료를 삭제합니다.
            }
        }

        System.out.println("================== JWKS Rotation Check End ==================");
    }

    /**
     * 새로운 JWKS 키를 생성하고 PENDING 상태로 저장합니다.
     *
     * @throws Exception 예외 발생 시
     */
    private void generateNewJWKey(String jwksStNm) throws Exception {
        KeyPair keyPair = generateRsaKeyPair(); // RSA 키쌍을 생성합니다.
        String kid = UUID.randomUUID().toString(); // 새로운 키 식별자(kid)를 생성합니다.

        String priKey = Base64.getEncoder().encodeToString(keyPair.getPrivate().getEncoded()); // 개인 키를 Base64로 인코딩합니다.

        RSAPublicKey rsaPublicKey = (RSAPublicKey) keyPair.getPublic(); // 공개 키를 RSAPublicKey로 캐스팅합니다.
        BigInteger modulus = rsaPublicKey.getModulus();          // == n
        BigInteger exponent = rsaPublicKey.getPublicExponent();  // == e

        String pubKey = base64UrlEncode(modulus.toByteArray());  // modulus 를 Base64URL 인코딩
        String e = base64UrlEncode(exponent.toByteArray()); // exponent 를 Base64URL 인코딩

        String today = getToday(); // 오늘 날짜(yyyyMMdd). DB에서 '유효한 키'를 찾을 때 기준이 됩니다.
        String endDate = getDateAfterDays(90); // 90일 후의 날짜를 "yyyyMMdd" 형식으로 가져옵니다.

        DisneyJWKSEntry newKey = new DisneyJWKSEntry(kid, pubKey, priKey, today, endDate, jwksStNm, e); // 새로운 JWKSEntry 객체를 생성합니다.
        jwksRepository.insertKey(newKey); // 새로운 JWK 를 데이터베이스에 저장합니다.

        System.out.println("[INFO] Created new JWK: " + kid);
    }

    /**
     * RSA 키쌍을 생성합니다.(TEST용)
     * 이 메서드는 RSA 2048 비트 키쌍을 생성하여 반환합니다.
     *
     * @return 생성된 RSA KeyPair
     * @throws Exception 키 생성 중 발생할 수 있는 예외
     */
    private KeyPair generateRsaKeyPair() throws Exception {
        // RSA 키쌍을 생성합니다. 2048 비트 키를 사용합니다.
        KeyPairGenerator generator = KeyPairGenerator.getInstance(SignatureAlgorithm.RS256.getFamilyName());
        generator.initialize(2048); // 키 크기를 2048 비트로 설정합니다.
        return generator.generateKeyPair();
    }

    /**
     * 바이트 배열을 Base64URL 형식으로 인코딩합니다.
     *
     * @param bytes 인코딩할 바이트 배열
     * @return Base64URL로 인코딩된 문자열
     */
    private String base64UrlEncode(byte[] bytes) {
        return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
    }

    /**
     * 현재 날짜를 "yyyyMMdd" 형식으로 반환합니다.
     *
     * @return 현재 날짜 문자열
     */
    private String getToday() {
        return new SimpleDateFormat("yyyyMMdd").format(new Date());
    }

    /**
     * 지정된 일수 후의 날짜를 "yyyyMMdd" 형식으로 반환합니다.
     *
     * @param days 일수
     * @return 지정된 일수 후의 날짜 문자열
     */
    private String getDateAfterDays(int days) {
        Calendar cal = Calendar.getInstance();
        cal.add(Calendar.DAY_OF_YEAR, days);
        return new SimpleDateFormat("yyyyMMdd").format(cal.getTime());
    }

    /**
     * 두 날짜 사이의 일수를 계산합니다.
     *
     * @param start 시작 날짜
     * @param end   종료 날짜
     * @return 두 날짜 사이의 일수
     */
    private long daysBetween(Date start, Date end) {
        long diff = end.getTime() - start.getTime();
        return diff / (1000 * 60 * 60 * 24);
    }

    /**
     * "yyyyMMdd" 형식의 문자열을 Date 객체로 변환합니다.
     *
     * @param yyyymmdd 날짜 문자열
     * @return 변환된 Date 객체
     * @throws Exception 파싱 오류 발생 시
     */
    private Date parseDate(String yyyymmdd) throws Exception {
        return new SimpleDateFormat("yyyyMMdd").parse(yyyymmdd);
    }


/*
 * [개선 포인트]
 *  1) 키 상태 전환(updateKeyStatus)은 '현재 상태가 무엇일 때만 변경' 같은 조건(WHERE status=...)을 걸어야 동시 실행에 안전합니다.
 *  2) privateKey를 DB에 그대로(Base64) 저장하면 DB 유출 시 즉시 토큰 위조가 가능합니다.
 *     => 가능하면 KMS/HSM 사용, 불가하면 최소한 암호화 + 접근통제 + 감사로그를 적용하세요.
 *  3) Repository 결과 Map에서 get("KID")/get("STARTDATE")처럼 대문자 키를 사용합니다.
 *     SQL alias는 소문자(kid/startDate)로 되어 있어 환경에 따라 null이 나올 수 있으니 키 이름을 통일하세요.
 */
}
```
#### 17.2 DisneyJWKSRepository (DB접근)
```java ln:false
package com.uracle.ott.disney;

import com.uracle.common.dbUtils.DatabaseUtils;
import io.jsonwebtoken.SignatureAlgorithm;

import java.sql.*;
import java.util.HashMap;
import java.util.Map;

/**
 * [무엇을 하는 클래스인가?]
 *  - JWKS 키 정보를 DB 테이블(IESM_JWKS_MGMT)에서 조회/저장/상태변경하는 Repository 입니다.
 *
 * [왜 필요한가?]
 *  - Auth 서버는 JWT를 서명하기 위해 '현재 서명키(ACTIVE)의 privateKey'가 필요합니다.
 *  - API 서버는 JWT를 검증하기 위해 'kid에 해당하는 publicKey'가 필요합니다.
 *  - 키 로테이션(정기 교체)을 하려면, 키의 상태(PENDING/ACTIVE/INACTIVE/ARCHIVE)와 유효기간을 DB로 관리하는 것이 편합니다.
 *
 * [초급자 주의]
 *  - 이 Repository의 메서드는 HashMap/Map 형태로 결과를 돌려줍니다.
 *    컬럼 alias(kid, startDate...)와 코드에서 get("KID")처럼 대문자로 꺼내는 부분이 섞이면 런타임에서 null이 나올 수 있어요.
 *    => 가능하면 DTO로 매핑하거나, key 이름을 상수로 통일하는 것을 추천합니다.
 */
public class DisneyJWKSRepository {

    protected DatabaseUtils dus = new DatabaseUtils(); // Database utility instance
    private final Connection conn; // Database connection instance

    /**
     * Constructor to initialize the DisneyJwtRepository with a database connection.
     *
     * @param conn the database connection to use
     * @throws SQLException if an error occurs while initializing the repository
     */
    public DisneyJWKSRepository(Connection conn) throws SQLException {
        // DB 연결을 받아 JWKS 테이블을 조회/갱신할 준비를 한다.
        this.conn = conn;
    }

    /**
     * Closes the database connection if it is open.
     *
     * @throws SQLException if an error occurs while closing the connection
     */
    public void close() throws SQLException {
        // Repository가 보유한 DB 커넥션을 닫는다(커넥션 풀/트랜잭션 환경이면 외부에서 관리하는 것이 일반적).
        if (conn != null && !conn.isClosed()) {
            conn.close();
        }
    }

    /**
     * Finds the latest pending JWKS key based on the provided date.
     *
     * @param today the date to check for pending keys
     * @return a map containing the latest pending key details
     * @throws SQLException if an error occurs while querying the database
     */
    public HashMap<String, String> findLatestPendingKey(String today) throws SQLException {

        // 오늘 기준으로 유효한 PENDING(예비키) 중 가장 최근 1개를 조회한다.
        StringBuffer strSQL = new StringBuffer();
        // SQL을 만들어 DB에서 키 정보를 조회한다.
        strSQL.append("SELECT  \n");
        strSQL.append("    JWKS_ID as kid, \n");
        strSQL.append("    JWKS_UTIL_SVC_NM as svcNm, \n");
        strSQL.append("    JWKS_ST_NM as status, \n");
        strSQL.append("    JWKS_STA_DT as startDate, \n");
        strSQL.append("    JWKS_END_DT as endDate, \n");
        strSQL.append("    JWT_CRE_ALGRM_NM as jwtAlg, \n");
        strSQL.append("    KEY_CRE_ALGRM_NM as keyAlg, \n");
        strSQL.append("    JWKS_PRIK as privateKey, \n");
        strSQL.append("    JWKS_PUBK as publicKey \n");
        strSQL.append("FROM IESM_JWKS_MGMT \n");
        strSQL.append("WHERE JWKS_ST_NM = 'PENDING' \n");
        strSQL.append("    AND JWKS_STA_DT <= #{jwksStaDt} AND JWKS_END_DT >= #{jwksEndDt} \n");
        strSQL.append("ORDER BY FST_RGST_DTM DESC FETCH FIRST 1 ROWS ONLY \n");

        Map<String, String> params = new HashMap();
        params.put("jwksStaDt", today);
        params.put("jwksEndDt", today);

        return dus.queryForMap(conn, strSQL.toString(), params);
    }

    /**
     * Finds the active JWKS key based on the provided date.
     *
     * @param today the date to check for active keys
     * @return a map containing the active key details
     * @throws SQLException if an error occurs while querying the database
     */
    public HashMap<String, String> findActiveKey(String today) throws SQLException {

        // 오늘 기준으로 유효한 ACTIVE(현재 서명키) 1개를 조회한다.
        StringBuffer strSQL = new StringBuffer();
        strSQL.append("SELECT  \n");
        strSQL.append("    JWKS_ID as kid, \n");
        strSQL.append("    JWKS_UTIL_SVC_NM as svcNm, \n");
        strSQL.append("    JWKS_ST_NM as status, \n");
        strSQL.append("    JWKS_STA_DT as startDate, \n");
        strSQL.append("    JWKS_END_DT as endDate, \n");
        strSQL.append("    JWT_CRE_ALGRM_NM as jwtAlg, \n");
        strSQL.append("    KEY_CRE_ALGRM_NM as keyAlg, \n");
        strSQL.append("    JWKS_PRIK as privateKey, \n");
        strSQL.append("    JWKS_PUBK as publicKey \n");
        strSQL.append("FROM IESM_JWKS_MGMT \n");
        strSQL.append("WHERE JWKS_ST_NM = 'ACTIVE' \n");
        strSQL.append("    AND JWKS_STA_DT <= #{jwksStaDt} AND JWKS_END_DT >= #{jwksEndDt} \n");
        strSQL.append("ORDER BY FST_RGST_DTM DESC FETCH FIRST 1 ROWS ONLY \n");

        Map<String, String> params = new HashMap();
        params.put("jwksStaDt", today);
        params.put("jwksEndDt", today);

        return dus.queryForMap(conn, strSQL.toString(), params);
    }

    /**
     * Finds all inactive JWKS keys.
     *
     * @return a map containing the inactive keys with their IDs and start dates
     * @throws SQLException if an error occurs while querying the database
     */
    public Map<String, String> findInactiveKeys() throws SQLException {

        // INACTIVE(이전 서명키, 검증 유예 상태) 키를 조회한다.
        StringBuffer strSQL = new StringBuffer();
        strSQL.append("SELECT JWKS_ID as kid, JWKS_STA_DT as startDate \n");
        strSQL.append("FROM IESM_JWKS_MGMT \n");
        strSQL.append("WHERE JWKS_ST_NM = 'INACTIVE' \n");

        return dus.queryForMap(conn, strSQL.toString(), new HashMap());
    }

    /**
     * Archives a JWKS key by its ID.
     *
     * @param kid the ID of the JWKS key to archive
     * @return the number of rows affected by the update
     * @throws SQLException if an error occurs while updating the database
     */
    public int archiveKey(String kid) throws SQLException {

        // 지정한 kid 키를 ARCHIVE(폐기) 상태로 변경한다.
        StringBuffer strSQL = new StringBuffer();
        strSQL.append("UPDATE IESM_JWKS_MGMT \n");
        strSQL.append("    SET JWKS_ST_NM = 'ARCHIVE' \n");
//        strSQL.append("        , JWKS_PRIK = 'ARCHIVED' \n");
//        strSQL.append("        , JWKS_PUBK = 'ARCHIVED' \n");
        strSQL.append("        , AUDIT_ID = 'SYSTEM' \n");
        strSQL.append("        , AUDIT_DTM = SYSDATE \n");
        strSQL.append("WHERE JWKS_ID = #{jwksId} \n");

        Map<String, String> params = new HashMap();
        params.put("jwksId", kid);

        // UPDATE/INSERT 실행 후 영향 받은 행 수를 반환한다.
        return dus.executeUpdate(conn, strSQL.toString(), params);
    }

    /**
     * Inserts a new JWKS key entry into the database.
     *
     * @param entry the JWKS entry to insert
     * @return the number of rows affected by the insert
     * @throws SQLException if an error occurs while inserting the entry
     */
    public int insertKey(DisneyJWKSEntry entry) throws SQLException {

        // 새 키(개인키/공개키/유효기간/상태)를 DB에 저장한다.
        StringBuffer strSQL = new StringBuffer();
        strSQL.append("INSERT INTO IESM_JWKS_MGMT ( \n");
        strSQL.append("    JWKS_ID, \n");
        strSQL.append("    JWKS_UTIL_SVC_NM, \n");
        strSQL.append("    JWKS_STA_DT, \n");
        strSQL.append("    JWKS_END_DT, \n");
        strSQL.append("    JWT_CRE_ALGRM_NM, \n");
        strSQL.append("    KEY_CRE_ALGRM_NM, \n");
        strSQL.append("    JWKS_PRIK, \n");
        strSQL.append("    JWKS_PUBK, \n");
        strSQL.append("    JWKS_ST_NM, \n");
        strSQL.append("    FST_RGST_ID, \n");
        strSQL.append("    FST_RGST_DTM, \n");
        strSQL.append("    AUDIT_ID, \n");
        strSQL.append("    AUDIT_DTM \n");
        strSQL.append(") VALUES ( \n");
        strSQL.append("    #{jwksId}, \n");
        strSQL.append("    #{jwksUtilSvcNm}, \n");
        strSQL.append("    #{jwksStaDt}, \n");
        strSQL.append("    #{jwksEndDt}, \n");
        strSQL.append("    #{jwtCreAlgrmNm}, \n");
        strSQL.append("    #{keyCreAlgrmNm}, \n");
        strSQL.append("    #{jwksPrik}, \n");
        strSQL.append("    #{jwksPubk}, \n");
        strSQL.append("    #{jwksStNm}, \n");
        strSQL.append("    #{fstRgstId}, \n");
        strSQL.append("    SYSDATE, \n");
        strSQL.append("    #{auditId}, \n");
        strSQL.append("    SYSDATE \n");
        strSQL.append(") \n");

        Map<String, String> params = new HashMap();
        params.put("jwksId", entry.getKid());
        params.put("jwksUtilSvcNm", "DISNEY_JWK"); // Assuming a service name for Disney JWT
        params.put("jwksStaDt", entry.getStartDate()); // Start date for the JWKS
        params.put("jwksEndDt", entry.getEndDate());   // End date for the JWKS
        params.put("jwtCreAlgrmNm", SignatureAlgorithm.RS256.getValue()); // Assuming RS256 as default
        params.put("keyCreAlgrmNm", SignatureAlgorithm.RS256.getFamilyName()); // Assuming RSA as default
        params.put("jwksPrik", entry.getPrivateKey()); // Private key, cut to fit 4000 bytes
        params.put("jwksPubk", entry.getPublicKey()); // Public key, cut to fit 4000 bytes
        params.put("jwksStNm", entry.getJwksStNm());  // New keys are inserted as PENDING
        params.put("fstRgstId", "SYSTEM"); // Assuming system as default registrar ID
        params.put("auditId", "SYSTEM"); // Assuming system as default audit ID

        return dus.executeUpdate(conn, strSQL.toString(), params);
    }

    /**
     * Updates the status of a JWKS key by its ID.
     *
     * @param kid    the ID of the JWKS key to update
     * @param status the new status to set for the JWKS key
     * @return the number of rows affected by the update
     * @throws SQLException if an error occurs while updating the database
     */
    public int updateKeyStatus(String kid, String status) throws SQLException {

        // 지정한 kid 키의 상태(PENDING/ACTIVE/INACTIVE/ARCHIVE)를 변경한다.
        StringBuffer strSQL = new StringBuffer();
        strSQL.append("UPDATE IESM_JWKS_MGMT \n");
        strSQL.append("    SET JWKS_ST_NM = #{status} \n");
        strSQL.append("        , AUDIT_ID = 'SYSTEM' \n");
        strSQL.append("        , AUDIT_DTM = SYSDATE \n");
        strSQL.append("WHERE JWKS_ID = #{jwksId} \n");

        Map<String, String> params = new HashMap();
        params.put("status", status);
        params.put("jwksId", kid);

        return dus.executeUpdate(conn, strSQL.toString(), params);
    }


/*
 * [개선 포인트]
 *  1) 반환 타입을 Map 대신 DTO로 바꾸면(예: JwksKeyRow) 키 이름 실수/캐스팅 실수를 줄일 수 있습니다.
 *  2) findInactiveKeys()는 이름은 복수인데 1건만 반환합니다.
 *     INACTIVE 키가 여러 개 생길 수 있다면 List로 받거나, SQL에서 '최신 1건' 정책을 명확히 해주세요.
 *  3) Connection 생명주기(close)를 Repository가 들고 가는 구조는 사용처에 따라 혼란이 생길 수 있습니다.
 *     (트랜잭션/커넥션 풀 환경에서는 보통 외부에서 관리)
 */
}
```

#### 17.3 DisneyJWKSEntry (DB 저장모델 + JWKS JSON)
```java ln:false
package com.uracle.ott.disney;

import io.jsonwebtoken.SignatureAlgorithm;

import java.util.HashMap;
import java.util.Map;

/**
 * [무엇을 하는 클래스인가?]
 *  - JWKS(JSON Web Key Set)에 들어가는 "키 1개(JWK)" 정보를 담는 모델 클래스입니다.
 *
 * [왜 필요한가?]
 *  - JWT는 서명(Signature)으로 위/변조를 막습니다.
 *  - 검증하는 쪽(API 서버)은 JWKS에서 공개키를 받아 서명을 검증합니다.
 *  - 그때 JWKS에 들어가는 필드(kid, n, e 등)를 코드에서 다루기 위해 이 모델이 필요합니다.
 *
 * [초급자 주의]
 *  - publicKey/privateKey는 둘 다 "문자열"처럼 보이지만 의미가 다릅니다.
 *    - privateKey(개인키): 토큰을 '만드는(서명하는)' 키. 유출되면 누구나 토큰을 위조할 수 있어 매우 위험합니다.
 *    - publicKey(공개키): 토큰을 '검증하는' 키. 외부에 공개해도 되지만, 변조되지 않도록(JWKS 신뢰 경로) 관리해야 합니다.
 */
public class DisneyJWKSEntry {
    // 키 식별자 (Key ID)
    // 키 식별자(kid): JWT 헤더에 들어가며, 검증 시 JWKS에서 어떤 키를 쓸지 결정한다.
    private String kid;
    // 공개 키 (Public Key)
    // 공개키: JWKS로 외부에 노출되는 값(검증용).
    private String publicKey;
    // 개인 키 (Private Key) - 선택적
    // 개인키: 서버 내부에서만 사용(서명용). 유출되면 토큰 위조가 가능하므로 매우 민감.
    private String privateKey;
    // 시작 날짜 (Start Date) - 선택적
    // 키 유효 시작일(YYYYMMDD).
    private String startDate;
    // 종료 날짜 (End Date) - 선택적
    // 키 유효 종료일(YYYYMMDD).
    private String endDate;
    // JWKS 상태 이름
    // 키 상태(PENDING/ACTIVE/INACTIVE/ARCHIVE).
    private String jwksStNm;
    // 키 알고리즘(예: RSA).  (주의) exponent(e) 같은 값과 혼동하지 않도록 주의
    private String keyAlg;

    /**
     * JWKSEntry 생성자
     *
     * @param kid         키 식별자
     * @param publicKey   공개 키
     * @param privateKey  개인 키
     * @param startDate   시작 날짜
     * @param endDate     종료 날짜
     * @param jwksStNm    JWKS 상태 이름
     */
    public DisneyJWKSEntry(String kid, String publicKey, String privateKey, String startDate, String endDate, String jwksStNm, String keyAlg) {
        this.kid = kid;
        this.publicKey = publicKey;
        this.privateKey = privateKey;
        this.startDate = startDate;
        this.endDate = endDate;
        this.jwksStNm = jwksStNm;
        this.keyAlg = keyAlg;
    }

    public String getKid() {
        return kid;
    }

    public void setKid(String kid) {
        this.kid = kid;
    }

    public String getPublicKey() {
        return publicKey;
    }

    public void setPublicKey(String publicKey) {
        this.publicKey = publicKey;
    }

    public String getPrivateKey() {
        return privateKey;
    }

    public void setPrivateKey(String privateKey) {
        this.privateKey = privateKey;
    }

    public String getStartDate() {
        return startDate;
    }

    public void setStartDate(String startDate) {
        this.startDate = startDate;
    }

    public String getEndDate() {
        return endDate;
    }

    public void setEndDate(String endDate) {
        this.endDate = endDate;
    }

    public String getJwksStNm() {
        return jwksStNm;
    }

    public void setJwksStNm(String jwksStNm) {
        this.jwksStNm = jwksStNm;
    }

    public String getKeyAlg() {
        return keyAlg;
    }

    public void setKeyAlg(String keyAlg) {
        this.keyAlg = keyAlg;
    }

    /**
     * JWKS JSON Entity 생성 메서드
     * n (modulus) 값은 publicKey 필드에서 추출되어야 하므로 이 예제는 Map 생성 구조만 정의
     */
    public Map<String, Object> toJsonEntity(String n, String e) {
        Map<String, Object> json = new HashMap<String, Object>();
        json.put("kty", SignatureAlgorithm.RS256.getFamilyName());
        json.put("kid", this.kid);
        json.put("alg", SignatureAlgorithm.RS256.getValue());
        json.put("use", "sig");
        json.put("n", n);
        json.put("e", e);
        return json;
    }

    /**
     * JWKS JSON Entity 생성 메서드
     * n (modulus) 값은 publicKey 필드에서 추출되어야 하므로 이 예제는 Map 생성 구조만 정의
     */
    @Override
    public String toString() {
        return "JWKSEntry{" +
                "kid='" + kid + '\'' +
                ", publicKey='" + publicKey + '\'' +
                ", privateKey='" + (privateKey != null ? "[PROTECTED]" : null) + '\'' +
                ", startDate='" + startDate + '\'' +
                ", endDate='" + endDate + '\'' +
                ", jwksStNm='" + jwksStNm + '\'' +
                ", keyAlg='" + keyAlg + '\'' +
                '}';
    }

/*
 * [개선 포인트]
 *  - 현재는 publicKey를 하나의 문자열로 들고 있는데, JWKS 표준(RSA)은 보통 n(modulus), e(exponent)로 표현합니다.
 *    필요하면 publicKeyN/publicKeyE로 분리하거나, JWK(JSON) 그대로 저장하는 방식도 고려하세요.
 *  - privateKey는 DB/로그 등에 노출되면 즉시 사고로 이어질 수 있습니다.
 *    운영에서는 KMS/HSM 사용, DB 저장 시 암호화(Envelope Encryption), 접근통제/감사로그가 권장됩니다.
 */
}
```
#### 17.4 DisneyJwtGenerator (JWT 발급)
```java ln:false
package com.uracle.ott.disney;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;

import java.math.BigInteger;
import java.security.*;
import java.security.spec.PKCS8EncodedKeySpec;
import java.security.spec.RSAPublicKeySpec;
import java.security.spec.X509EncodedKeySpec;
import java.sql.Connection;
import java.sql.DriverManager;
import java.text.SimpleDateFormat;
import java.time.Instant;
import java.util.*;

/**
 * [무엇을 하는 클래스인가?]
 *  - Disney 연동에서 필요한 JWT 토큰(Activation/Entitlement 등)을 생성(서명)하는 예시 코드입니다.
 *
 * [큰 흐름]
 *  1) DB에서 '오늘 기준으로 유효한 ACTIVE 키'를 가져옵니다. (서명에 필요한 privateKey 포함)
 *  2) JWT Header에 kid를 넣습니다. (검증 서버가 JWKS에서 어떤 공개키를 써야 하는지 알기 위해)
 *  3) payload(클레임)를 구성하고 privateKey로 서명하여 토큰 문자열을 만듭니다.
 *
 * [초급자 주의]
 *  - JWT는 기본적으로 '암호화'가 아니라 '서명'입니다. payload에 민감정보를 넣지 마세요.
 *  - alg(알고리즘)는 토큰 헤더에 들어가지만, 서버는 헤더 값을 그대로 신뢰하면 안 됩니다.
 *    리소스 서버에서는 허용 alg를 고정(화이트리스트)해야 합니다.
 */
public class DisneyJwtGenerator {

    private static PrivateKey privateKey; // 활성화 토큰 생성 시 사용하는 개인 키
    private static PublicKey publicKey;   // 권한 부여 토큰 생성 시 사용하는 공개 키
    // 1) DB 접속 정보를 설정 파일에서 읽는다.
    private static final ResourceBundle prop = ResourceBundle.getBundle("com.uracle.ott.disney.bundle.common.Disney");

    /**
     * Disney JWT Activation Token 생성
     *
     * 이 코드는 Disney OTT 서비스에 필요한 JWT Activation Token을 생성합니다.
     * 실제 서비스에서는 RSA 키쌍을 안전하게 관리하고, 필요한 경우 기존 키를 불러와야 합니다.
     */
    public String getActivationToken(Connection conn, String mgmtNum, String purcNo, String productId) throws Exception {

        // JWKS 레포지토리 초기화
        DisneyJWKSRepository repository = new DisneyJWKSRepository(conn);

        // 현재 날짜를 "yyyyMMdd" 형식으로 가져오기
        String today = new SimpleDateFormat("yyyyMMdd").format(new Date());
        HashMap<String, String> activeKey = repository.findActiveKey(today); // 현재 활성화된 키를 가져오기

        // 1. ACTIVE 개인 키 불러오기
        privateKey = getPrivateKeyFromString(activeKey.get("PRIVATEKEY")); // 활성화 토큰 생성 시 사용하는 개인 키
        publicKey = getPublicKeyFromString(activeKey.get("PUBLICKEY")); // 권한 부여 토큰 생성 시 사용하는 공개 키

        // 2. JWT 헤더
        Map<String, Object> header = new HashMap<>();
        header.put("kid", activeKey.get("KID")); // kid: '이 토큰을 어떤 키로 서명했는지' 표시. 검증 서버는 이 kid로 JWKS에서 공개키를 찾습니다.
        header.put("alg", activeKey.get("JWTALG")); // alg: 서명 알고리즘(예: RS256). (주의) 검증 서버는 헤더 alg를 그대로 믿지 말고 서버 설정으로 고정해야 합니다.

        // 3. 현재 시간 기준 iat/exp 계산
        long iat = Instant.now().getEpochSecond();  // 현재 시간 (초 단위)
        long exp = iat + 86400;                     // 24시간 후
//        long exp = iat + 3600;                      // 60분 후

        // 4. JWT 페이로드 구성
        Map<String, Object> payload = new LinkedHashMap<>();
        Map<String, Object> entitlements = new LinkedHashMap<>();
        Map<String, Object> entitlement = new LinkedHashMap<>();

        Map<String, Object> address = new LinkedHashMap<>();
        address.put("country", "KR");

        entitlement.put("products", productId);   // 사용자 권한 정보, 예시: {"1": "com.project.2"}
        entitlements.put(purcNo, entitlement);  // 구매번호

        payload.put("aud", prop.getString("api.aud.activation"));  // JWT의 대상 (audience), 활성화 : activation.disneystreaming.com
        payload.put("sub", mgmtNum);                                    // JWT의 주체 (subject), 사용자 ID 또는 계정 ID, 서비스번호
        payload.put("provider", prop.getString("api.provider"));   // JWT 공급자 (provider), 예시: SKB_KR, 사용자에게 제공할 자산과 메시지를 식별하는 기준
        payload.put("iss", prop.getString("api.iss"));             // JWT 발행자 (issuer), Disney에서 검증을 위한 PublicKey 요청 CBS 서버 URL
        payload.put("iat", iat);                                        // JWT 발급 시간 (issued at), 현재 시간 기준
        payload.put("exp", exp);                                        // JWT 만료 시간 (expiration time), 24시간 후, Disney는 토큰 만료 시간 60분을 권장
        payload.put("address", address);                                // 사용자 주소 정보, 예시: {"country": "KR"}
        payload.put("entitlements", entitlements);                      // 사용자 권한 정보, 예시: {"구매번호": {"products": "com.disney.partner.standalone"}}

        // 5. JWT 생성
        String token = Jwts.builder()
                .setHeader(header)
                .setClaims(payload)
                .signWith(privateKey, SignatureAlgorithm.RS256)
                .compact();

        System.out.println(">>> Activation Token:\nBearer " + token);

        return "Bearer " + token;
    }

    /**
     * Disney Entitlements JWT Bearer Token 생성
     *
     * 이 코드는 Disney OTT 서비스에 필요한 Entitlements JWT Bearer Token을 생성합니다.
     * 실제 서비스에서는 RSA 키쌍을 안전하게 관리하고, 필요한 경우 기존 키를 불러와야 합니다.
     */
    public String getEntitlementsToken(Connection conn, String audStr) throws Exception {

        // JWKS 레포지토리 초기화
        DisneyJWKSRepository repository = new DisneyJWKSRepository(conn);

        // 현재 날짜를 "yyyyMMdd" 형식으로 가져오기
        String today = new SimpleDateFormat("yyyyMMdd").format(new Date());
        HashMap<String, String> activeKey = repository.findActiveKey(today); // 현재 활성화된 키를 가져오기

        // 1. ACTIVE 개인 키 불러오기
        privateKey = getPrivateKeyFromString(activeKey.get("PRIVATEKEY")); // 활성화 토큰 생성 시 사용하는 개인 키
        publicKey = getPublicKeyFromString(activeKey.get("PUBLICKEY")); // 권한 부여 토큰 생성 시 사용하는 공개 키

        // 2. JWT 헤더
        Map<String, Object> header = new HashMap<>();
        header.put("kid", activeKey.get("KID")); // kid: '이 토큰을 어떤 키로 서명했는지' 표시. 검증 서버는 이 kid로 JWKS에서 공개키를 찾습니다.
        header.put("alg", SignatureAlgorithm.RS256.getValue());

        // 3. 현재 시간 기준 iat/exp 계산
        long iat = Instant.now().getEpochSecond();  // 현재 시간 (초 단위)
        long exp = iat + 86400;                     // 24시간 후
//        long exp = iat + 3600;                      // 60분 후

        // 4. JWT 페이로드
        Map<String, Object> payload = new LinkedHashMap<>();
        if(audStr != null && !audStr.isEmpty()) {
            if (audStr.equals("set")) {
                payload.put("aud", prop.getString("api.aud.entitlement.set")); // JWT의 대상 (audience), Entitlement : "set.entitlement.disneystreaming.com"
            } else if (audStr.equals("get")) {
                payload.put("aud", prop.getString("api.aud.entitlement.get")); // JWT의 대상 (audience), Entitlement : "get.entitlement.disneystreaming.com"
            } else {
                throw new IllegalArgumentException("Invalid audience (aud) value. Must be 'set' or 'get'.");
            }
        } else {
            throw new IllegalArgumentException("Audience (aud) value is required.");
        }
/*
        ArrayList<String> aud = new ArrayList<String>(Arrays.asList(
                prop.getString("api.aud.entitlement.get"),
                prop.getString("api.aud.entitlement.set")
        )); // JWT의 대상 (audience), Entitlement : "set.entitlement.disneystreaming.com" or "get.entitlement.disneystreaming.com"
*/
        payload.put("provider", prop.getString("api.provider"));       // JWT 발급자 (provider), 예시: SKBTest, 사용자에게 제공할 자산과 메시지를 식별하는 기준, 값은 Disney와 협력하여 결정, Disney는 단일 값 또는 여러 값을 사용할지 정의
        payload.put("iss", prop.getString("api.iss"));                 // JWT 발급자 (issuer), Disney에서 검증을 위한 PublicKey 요청 CBS 서버 URL
        payload.put("iat", iat);                                            // JWT 발급 시간 (issued at), 현재 시간 기준
        payload.put("exp", exp);                                            // JWT 만료 시간 (expiration time), 24시간 후, Disney는 토큰 만료 시간 60분을 권장

        // 5. JWT 생성
        String token = Jwts.builder()
                .setHeader(header)
                .setClaims(payload)
                .signWith(privateKey, SignatureAlgorithm.RS256)
                .compact();

        // 6. 생성된 JWT 출력
        System.out.println(">>> Entitlements Token:\nBearer " + token);

        return "Bearer " + token;
    }

    /**
     * Disney JWT Bearer Token 생성 테스트
     *
     * 이 코드는 Disney OTT 서비스에 필요한 JWT Bearer Token을 생성합니다.
     * 실제 서비스에서는 RSA 키쌍을 안전하게 관리하고, 필요한 경우 기존 키를 불러와야 합니다.
     */
    public static void main(String[] args) throws Exception {

        // 샘플 실행: DB에서 키를 읽어 Activation/Entitlement JWT를 생성하고, 공개키로 검증까지 해본다.
        ResourceBundle prop = ResourceBundle.getBundle("com.imnetpia.exbill.transfer.daemon.common.BatchDatabaseInfo");

        String SmsDbUrl = prop.getString("batch.SmsDbUrl");
        String SmsDbUser = prop.getString("batch.SmsDbUser");
        String SmsDbPW = prop.getString("batch.SmsDbPW");
        String SmsOracleDriver = prop.getString("batch.SmsOracleDriver");

        // 2) JDBC 드라이버를 로드한다.
        Class.forName(SmsOracleDriver);
        // DB 연결 초기화
        // 3) DB에 연결한다.
        Connection conn = DriverManager.getConnection(SmsDbUrl, SmsDbUser, SmsDbPW);

        DisneyJwtGenerator jwtBuilder = new DisneyJwtGenerator();

        // 활성화 토큰 생성
        // 4) Activation 토큰을 생성한다(서명은 ACTIVE privateKey로 수행).
        String activationToken = jwtBuilder.getActivationToken(conn, "9999900020", "123123", "com.disney.skb.kr.standalone.monthly.premium");
        System.out.println("Generated Activation Token: " + activationToken);

        // Entitlements 토큰 생성
        // 5) Entitlements 토큰을 생성한다.
        String entitlementsToken = jwtBuilder.getEntitlementsToken(conn, "set");
        System.out.println("Generated Entitlements Token: " + entitlementsToken);

        String aToken = activationToken.replace("Bearer ", "").trim();  // 접두어 제거 + 공백 제거
        // 6) 생성된 토큰을 공개키로 검증해서 payload를 확인한다(테스트용).
        Claims aClaims = Jwts.parserBuilder()
                .setSigningKey(publicKey)  // 공개키
                .build()
                .parseClaimsJws(aToken)
                .getBody();    // 예외 발생 시 서명 위조
        System.out.println("Activation Token Payload:");
        System.out.println("entitlements: " + aClaims.get("entitlements"));
        System.out.println("aud: " + aClaims.get("aud"));
        System.out.println("sub: " + aClaims.get("sub"));
        System.out.println("address: " + aClaims.get("address"));
        System.out.println("provider: " + aClaims.get("provider"));
        System.out.println("iss: " + aClaims.getIssuer());
        System.out.println("iat: " + aClaims.getIssuedAt());
        System.out.println("exp: " + aClaims.getExpiration());

        String eToken = entitlementsToken.replace("Bearer ", "").trim();  // 접두어 제거 + 공백 제거
        Claims eClaims = Jwts.parserBuilder()
                .setSigningKey(publicKey)  // 공개키
                .build()
                .parseClaimsJws(eToken)
                .getBody();    // 예외 발생 시 서명 위조
        System.out.println("Entitlements Token Payload:");
        System.out.println("aud: " + eClaims.get("aud"));
        System.out.println("provider: " + eClaims.get("provider"));
        System.out.println("iss: " + eClaims.getIssuer());
        System.out.println("iat: " + eClaims.getIssuedAt());
        System.out.println("exp: " + eClaims.getExpiration());
    }

    public static PrivateKey getPrivateKeyFromString(String privateKeyStr) throws Exception {


        // DB에 저장된 Base64 개인키 문자열을 PrivateKey 객체로 변환한다(서명에 사용).
        byte[] decoded = Base64.getDecoder().decode(privateKeyStr);
        PKCS8EncodedKeySpec keySpec = new PKCS8EncodedKeySpec(decoded);
        KeyFactory keyFactory = KeyFactory.getInstance("RSA");

        return keyFactory.generatePrivate(keySpec);
    }

    public static PublicKey getPublicKeyFromString(String pubKeyStr) throws Exception {

        // DB에 저장된 Base64 공개키 문자열을 PublicKey 객체로 변환한다(검증에 사용).
//        byte[] decoded = Base64.getDecoder().decode(pubKeyStr);
//        X509EncodedKeySpec keySpec = new X509EncodedKeySpec(decoded);
//        KeyFactory keyFactory = KeyFactory.getInstance("RSA");
//        return keyFactory.generatePublic(keySpec);

        BigInteger modulus = base64UrlToBigInt(pubKeyStr);
        BigInteger exponent = base64UrlToBigInt("AQAB"); // 일반적으로 공개 지수는 65537 (0x10001)로 설정됩니다.

        System.out.println("modulus.bitLength : " + modulus.bitLength());

        RSAPublicKeySpec keySpec = new RSAPublicKeySpec(modulus, exponent);
        KeyFactory factory = KeyFactory.getInstance("RSA");

        return factory.generatePublic(keySpec);
    }

    public static BigInteger base64UrlToBigInt(String base64Url) {

        // JWKS의 Base64URL 인코딩 값(n/e)을 BigInteger로 변환한다(공개키 복원에 사용).
        byte[] decoded = Base64.getUrlDecoder().decode(base64Url);
        return new BigInteger(1, decoded); // '1' to avoid negative
    }


/*
 * [개선 포인트]
 *  1) 이 클래스는 DB에서 가져온 alg 값을 그대로 헤더에 넣습니다.
 *     운영에서는 '우리 서비스가 허용하는 알고리즘(RS256 등)'을 코드/설정으로 고정하는 것을 권장합니다.
 *  2) privateKey/publicKey를 static 필드로 보관하면 멀티스레드 환경에서 오해/오염 가능성이 있습니다.
 *     가능하면 메서드 내부 지역변수로 사용하세요.
 *  3) 예외/로그에서 토큰/키 원문이 출력되지 않도록 주의하세요(민감정보).
 */
}
```

#### 17.5 DisneyJwtValidChecker (JWT 검증 + JWKS에서 키 찾기)
``` java ln:false
package com.uracle.ott.disney;

import java.math.BigInteger;
import java.security.KeyFactory;
import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.NoSuchAlgorithmException;
import java.security.PrivateKey;
import java.security.PublicKey;
import java.security.spec.InvalidKeySpecException;
import java.security.spec.RSAPublicKeySpec;
import java.util.Base64;
import java.util.Date;

import org.json.simple.JSONArray;
import org.json.simple.JSONObject;
import org.json.simple.parser.JSONParser;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.ExpiredJwtException;
import io.jsonwebtoken.Jws;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.MalformedJwtException;
import io.jsonwebtoken.SignatureAlgorithm;
import io.jsonwebtoken.UnsupportedJwtException;

/**
 * [무엇을 하는 클래스인가?]
 *  - (테스트/예시용) JWT를 검증하는 과정을 단계별로 보여주는 유틸성 코드입니다.
 *
 * [JWT 검증이란?]
 *  - 토큰을 만든 쪽(Auth 서버)은 privateKey로 서명합니다.
 *  - 검증하는 쪽(API 서버)은 publicKey로 서명을 검증합니다.
 *  - 서명이 맞으면 payload(클레임)가 위/변조되지 않았다고 볼 수 있습니다.
 *
 * [이 클래스가 보여주는 핵심 단계]
 *  1) Authorization 헤더에서 JWT를 꺼낸다(Bearer 제거)
 *  2) JWT 헤더를 Base64Url 디코딩해서 kid(키 ID)를 읽는다
 *  3) JWKS(keys 배열)에서 kid가 같은 JWK를 찾는다
 *  4) JWK의 n/e로 RSA PublicKey를 복원한다
 *  5) publicKey로 JWT 서명을 검증한다
 *
 * [초급자 주의]
 *  - 실서비스에서는 JWKS를 매번 파싱/조회하지 말고 캐싱(TTL)해야 합니다.
 *  - 서명 검증만으로 끝내지 말고 exp/iss/aud 같은 클레임 검증도 반드시 해야 합니다.
 */
public class DisneyJwtValidChecker {
    private static PrivateKey SECRET_KEY;
    private static final long TOKEN_VALIDITY = 60 * 1000; // 1분 유효

    /**
     * (TEST용) JWT 토큰을 생성합니다.
     * 현재 시간 기준으로 1분 후에 만료되는 토큰을 생성합니다.
     *
     * @return 생성된 JWT 토큰 문자열
     */
     /*
    public static String generateToken() {
        // 테스트용 JWT를 생성한다(샘플; 실제 운영에서는 Auth 서버가 발급).
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + TOKEN_VALIDITY);

        return Jwts.builder()
                .setSubject("test user")
                .setIssuedAt(now)
                .setExpiration(expiryDate)
                .signWith(SECRET_KEY, SignatureAlgorithm.RS256)
                .compact();
    }
    */

    /**
     * JWT 토큰이 만료되었는지 확인합니다.
     * 만료된 경우 true를 반환하고, 그렇지 않으면 false를 반환합니다.
     *
     * @param token JWT 토큰 문자열
     * @return 토큰이 만료되었는지 여부
     */
    public static boolean isTokenExpired(String token) {
        // JWT의 exp(만료시간)이 지났는지 확인한다.
        try {
            Jwts.parserBuilder().setSigningKey(SECRET_KEY).build().parseClaimsJws(token);
            return false; // 만료되지 않음
        } catch (ExpiredJwtException e) {
            return true; // 만료됨
        }
    }

    /**
     * Header로 넘어 온 jwt의 kid 조회
     * @param authorizationHeader
     * @return
     */
    public static String extractKidFromAuthorization(String authorizationHeader, String key) {
        // Authorization 헤더에서 JWT를 꺼내고, 헤더의 kid 값을 추출한다.
        try {
            // Authorization 헤더는 보통 "Bearer <토큰>" 형태입니다. 앞의 "Bearer "를 제거합니다.
            if (authorizationHeader == null || !authorizationHeader.startsWith("Bearer ")) {
                throw new IllegalArgumentException("Invalid Authorization header");
            }

            String token = authorizationHeader.substring("Bearer ".length());
            String[] parts = token.split("\\.");

            if (parts.length < 2) {
                throw new IllegalArgumentException("Invalid JWT token format");
            }

            // JWT의 첫 번째 파트(header)는 Base64Url 인코딩된 JSON입니다. 디코딩해서 JSON으로 파싱합니다.
            String headerJson = new String(Base64.getUrlDecoder().decode(parts[0]));

            JSONParser parser = new JSONParser();
            JSONObject headerObj = (JSONObject) parser.parse(headerJson);

            return (String) headerObj.get(key);

        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }

    public static PublicKey getPublicKeyFromJwks(JSONObject jwksJson) throws NoSuchAlgorithmException, InvalidKeySpecException {
        String n = (String) jwksJson.get("n");
        String e = (String) jwksJson.get("e");

        // Base64 URL-safe decode (RFC 7517)
        byte[] nBytes = Base64.getUrlDecoder().decode(n);
        byte[] eBytes = Base64.getUrlDecoder().decode(e);

        BigInteger modulus = new BigInteger(1, nBytes);
        BigInteger exponent = new BigInteger(1, eBytes);

        RSAPublicKeySpec keySpec = new RSAPublicKeySpec(modulus, exponent);
        KeyFactory keyFactory = KeyFactory.getInstance("RSA");

        return keyFactory.generatePublic(keySpec);
    }
    
    /**
     * JWT 토큰의 유효성을 검사합니다.
     * "Bearer " 접두사를 제거하고, JWT 구조를 검사하며, 서명을 검증합니다.
     *
     * @param tokenWithBearer JWT 토큰 문자열
     * @param publicKey JWT 서명을 검증할 공개 키
     * @return 토큰이 유효한 경우 true, 그렇지 않으면 false
     */
    // 1) Bearer 접두어가 포함된 토큰 문자열을 받는다.
    // 4) JWKS의 n/e로 RSA PublicKey를 복원한다.
    public static boolean isValidJwtToken(String tokenWithBearer, PublicKey publicKey) {
        // 공개키로 JWT 서명을 검증한다(위조/변조 여부 확인).
        try {
            // 1. "Bearer " 제거 및 trim
            // 2) 'Bearer '를 제거하고 순수 JWT 문자열만 남긴다.
            String token = tokenWithBearer.replace("Bearer ", "").trim();

            // 2. 구조 검사: header.payload.signature
            String[] parts = token.split("\\.");
            if (parts.length != 3) {
                System.err.println("? Invalid JWT structure (must have 3 parts).");
                return false;
            }

            // 3. 서명 유효성 검증
            Jws<Claims> jws = Jwts.parserBuilder()
                    .setSigningKey(publicKey)
                    .build()
                    .parseClaimsJws(token);

            // 4. 페이로드 접근 가능 시 유효한 토큰
            // 5) 공개키로 서명을 검증하면서 Claims(payload)를 파싱한다.
            Claims claims = jws.getBody();
            System.out.println("? JWT is valid.");
            System.out.println("aud: " + claims.get("aud"));
            System.out.println("iss: " + claims.getIssuer());
            System.out.println("iat: " + claims.getIssuedAt());
            System.out.println("exp: " + claims.getExpiration());
            return true;

        } catch (ExpiredJwtException e) {
            System.err.println("? JWT expired: " + e.getMessage());
        } catch (UnsupportedJwtException e) {
            System.err.println("? Unsupported JWT: " + e.getMessage());
        } catch (MalformedJwtException e) {
            System.err.println("? Malformed JWT: " + e.getMessage());
        } catch (IllegalArgumentException e) {
            System.err.println("? Illegal argument: " + e.getMessage());
        }
        return false;
    }
    
    /**
     * JWKS내 kid값 확인
     * @param jwks
     * @param kid
     * @return
     */
    // 3) JWT 헤더를 Base64URL 디코딩해 kid를 읽는다.
    public static JSONObject findJwkByKid(JSONObject jwks, String kid) {
        // JWKS(keys 배열)에서 kid가 일치하는 JWK를 찾아 반환한다.
        if (jwks == null) throw new IllegalArgumentException("JWKS is null");
        if (kid == null || kid.isEmpty()) throw new IllegalArgumentException("kid is null/empty");

        Object keysObj = jwks.get("keys");
        if (!(keysObj instanceof JSONArray)) {
            throw new IllegalArgumentException("JWKS does not contain 'keys' array");
        }
        JSONArray keys = (JSONArray) keysObj;

        for (Object o : keys) {
            if (o instanceof JSONObject) {
                JSONObject jwk = (JSONObject) o;
                Object k = jwk.get("kid");
                if (k != null && kid.equals(k.toString())) {
                    return jwk; // 매칭된 키
                }
            }
        }
        return null; // 매칭 없음
    }

    // 문자열 형태의 JWKS를 바로 받는 오버로드(선택)
    public static JSONObject findJwkByKid(String jwksJson, String kid) throws Exception {
        JSONObject jwks = (JSONObject) new JSONParser().parse(jwksJson);
        return findJwkByKid(jwks, kid);
    }
    

    /**
     * (TEST용) 메인 메서드: JWT 토큰을 생성하고 유효성을 검사합니다.
     * RSA 키쌍을 생성하고, JWT 토큰을 생성한 후, 유효성 검사 및 만료 여부를 확인합니다.
     *
     * @param args 명령줄 인자
     * @throws Exception 예외 발생 시
     */
     /*
    public static void main(String[] args) throws Exception {

        KeyPair keyPair = generateRsaKeyPair();
        SECRET_KEY = keyPair.getPrivate();
        PublicKey PUBLIC_KEY = keyPair.getPublic();

        String token = generateToken();
        System.out.println("Generated Token: " + token);

        // JWT 유효성 검사
        if (isValidJwtToken(token, PUBLIC_KEY)) {
            System.out.println("Token is valid");
        } else {
            System.out.println("Token is invalid");
        }

        Thread.sleep(70 * 1000); // 70초 대기

        // 토큰 만료 여부 확인
        if (isTokenExpired(token)) {
            System.out.println("Token is expired");
        } else {
            System.out.println("Token is not expired");
        }
    }
    */

    /**
     * (TEST용) RSA 키쌍을 생성합니다.
     * 이 메서드는 RSA 2048 비트 키쌍을 생성하여 반환합니다.
     *
     * @return 생성된 RSA KeyPair
     * @throws Exception 키 생성 중 발생할 수 있는 예외
     */
    public static KeyPair generateRsaKeyPair() throws Exception {
        KeyPairGenerator generator = KeyPairGenerator.getInstance(SignatureAlgorithm.RS256.getFamilyName());
        generator.initialize(2048);
        return generator.generateKeyPair();
    }


/*
 * [개선 포인트]
 *  1) isTokenExpired()는 SECRET_KEY로 파싱합니다. (현재 코드에서는 privateKey)
 *     실제 검증 서버는 publicKey로 서명 검증해야 하므로, 검증 키를 publicKey로 통일하는 것이 안전합니다.
 *  2) 실서비스에서는 kid 미스매치 시 JWKS를 즉시 갱신하고, JWKS를 TTL 캐싱해야 합니다.
 *  3) iss/aud/alg 화이트리스트 검증을 추가하면 보안 사고(alg 혼동 등)를 줄일 수 있습니다.
 */
}
```



























---
### 17. 구현 예시(순수 Java)
#### 17.1 구현 개요
리소스 서버는 다음 구성요소로 JWT/JWKS 검증을 구현한다.
- `JwksProvider`: JWKS 다운로드 + TTL 캐싱 + kid 미스매치 시 즉시 갱신
- `JwkVerifier`: 토큰 파싱/서명검증/클레임 검증/권한 파싱
- `AuthFilter`(선택): HTTP 요청에서 Bearer 토큰을 꺼내 검증 후 사용자 컨텍스트 주입

#### 17.2 JWKS Provider(TTL 캐시+kid 미스매치 즉시 갱신+레이트리밋)
```java
import com.nimbusds.jose.jwk.JWK;
import com.nimbusds.jose.jwk.JWKSet;

import java.net.URL;
import java.time.Instant;
import java.util.concurrent.atomic.AtomicLong;

/**
 * JWKS(JSON Web Key Set) 다운로드/캐싱 담당 클래스.
 *
 * 목적:
 * - 리소스 서버가 JWT를 검증할 때 필요한 공개키(JWK)를 kid로 찾아 제공한다.
 * - JWKS를 매 요청마다 호출하면 네트워크 비용/장애 전파가 생기므로 TTL 캐싱을 한다.
 * - 키 로테이션 상황에서 "kid가 캐시에 없을 때"는 즉시 JWKS를 재조회하여 새 키를 반영한다.
 * - 다만 kid 미스매치가 대량 발생하거나 JWKS 서버 장애 시 폭주를 막기 위해 강제갱신 레이트리밋을 둔다.
 */
public final class JwksProvider {

  /** JWKS endpoint URL (예: https://auth-server/.well-known/jwks.json) */
  private final URL jwksUrl;

  /** JWKS 캐시 TTL(초). 예: 600(10분) ~ 3600(1시간) */
  private final long ttlSeconds;

  /**
   * 프로세스 메모리에 JWKS를 보관한다.
   * - volatile: 여러 스레드에서 최신 참조를 볼 수 있도록 함(락 없이도 최신 값 가시성 확보)
   */
  private volatile JWKSet cached;

  /** cached JWKS가 만료되는 epoch second */
  private volatile long expiresAtEpochSec = 0;

  /**
   * kid 미스매치 시 "즉시 강제 갱신"을 하게 되는데,
   * 이걸 무제한 허용하면 장애 시 JWKS endpoint로 요청이 폭주할 수 있다.
   *
   * lastForceRefreshEpochSec: 마지막으로 강제갱신을 시도한 시각(초)
   * AtomicLong: 멀티스레드 환경에서 compareAndSet으로 간단 레이트리밋 구현
   */
  private final AtomicLong lastForceRefreshEpochSec = new AtomicLong(0);

  /** 강제갱신 최소 간격(초). 예: 30~60초 */
  private final long minForceRefreshIntervalSeconds;

  public JwksProvider(URL jwksUrl, long ttlSeconds, long minForceRefreshIntervalSeconds) {
    this.jwksUrl = jwksUrl;
    this.ttlSeconds = ttlSeconds;
    this.minForceRefreshIntervalSeconds = minForceRefreshIntervalSeconds;
  }

  /**
   * kid를 받아서 JWKS에서 해당 키(JWK)를 찾아 반환한다.
   *
   * 동작:
   * 1) TTL 캐시 확인 → 만료면 갱신
   * 2) 캐시에서 kid 매칭되는 키 검색
   * 3) 없으면(키 로테이션 가능성) 즉시 JWKS 재조회 후 재시도
   * 4) 그래도 없으면 null 반환 (검증 실패로 401 처리)
   */
  public JWK getByKid(String kid) throws Exception {
    // (1) 캐시가 없거나 TTL 만료면 JWKS 갱신
    JWKSet jwks = getCachedOrRefresh();

    // (2) 캐시에서 kid로 키 검색
    JWK key = (kid == null) ? null : jwks.getKeyByKeyId(kid);
    if (key != null) return key;

    // (3) kid 미스매치 → 키 로테이션/캐시 지연 가능성
    //     TTL을 기다리지 말고 즉시 JWKS 재조회(단, 레이트리밋 적용)
    forceRefreshWithRateLimit();

    // (4) 재조회 후 다시 kid로 키 검색
    jwks = getCachedOrRefresh();
    return (kid == null) ? null : jwks.getKeyByKeyId(kid);
  }

  /**
   * 캐시를 반환하되, 없거나 TTL이 지났으면 refreshNow() 수행.
   * - 여기서는 "평상시 갱신(TTL)" 로직
   */
  private JWKSet getCachedOrRefresh() throws Exception {
    long now = Instant.now().getEpochSecond();
    if (cached == null || now >= expiresAtEpochSec) {
      refreshNow();
    }
    return cached;
  }

  /**
   * JWKS endpoint에서 최신 JWKS를 다운로드해 캐시에 저장한다.
   * synchronized: 동시에 여러 스레드가 갱신하려고 해도 1번만 수행되도록.
   */
  private synchronized void refreshNow() throws Exception {
    // Nimbus 유틸: URL에서 JWKS JSON을 가져와 JWKSet으로 파싱
    this.cached = JWKSet.load(jwksUrl);

    // TTL 설정: 지금 시각 + ttlSeconds 까지 유효
    this.expiresAtEpochSec = Instant.now().plusSeconds(ttlSeconds).getEpochSecond();
  }

  /**
   * kid 미스매치 시 즉시 갱신 로직.
   * - 다만 폭주 방지를 위해 "최소 간격" 내에는 재조회하지 않는다.
   *
   * 동작:
   * 1) 마지막 강제갱신 시각(last) 확인
   * 2) now-last < minInterval이면 갱신 스킵
   * 3) compareAndSet으로 1개 스레드만 갱신하도록 보장
   */
  private void forceRefreshWithRateLimit() throws Exception {
    long now = Instant.now().getEpochSecond();
    long last = lastForceRefreshEpochSec.get();

    // 최소 간격 내이면 강제갱신 스킵
    if (now - last < minForceRefreshIntervalSeconds) {
      return;
    }

    // CAS 성공한 스레드만 refresh 수행 (다른 스레드는 스킵)
    if (lastForceRefreshEpochSec.compareAndSet(last, now)) {
      refreshNow();
    }
  }
}
```
---
#### 17.3 JwtVerifier (alg/iss/aud/exp 강제 + scope/role 파싱)
```java
import com.nimbusds.jose.*;
import com.nimbusds.jose.crypto.*;
import com.nimbusds.jose.jwk.*;
import com.nimbusds.jwt.*;

import java.time.Instant;
import java.util.*;

/**
 * JWT 검증 담당 클래스.
 *
 * 검증 범위:
 * 1) Header 검증: alg 화이트리스트(서버 설정) + kid 추출
 * 2) JWKS에서 kid로 공개키 조회(캐시 우선, 미스매치 시 즉시 갱신)
 * 3) 서명(Signature) 검증
 * 4) Claims 검증: iss/aud/exp(+ nbf)
 * 5) 권한 파싱: scope/role 등
 *
 * 주의:
 * - JWT Header의 alg를 그대로 믿으면 안 된다(alg 혼동 공격).
 *   따라서 allowedAlgs는 서버 정책으로 고정한다.
 */
public final class JwtVerifier {

  private final JwksProvider jwksProvider;

  /** 신뢰하는 발급자(iss). 예: https://auth-server */
  private final String expectedIssuer;

  /** 이 서비스가 받는 토큰의 대상(aud). 예: my-api */
  private final String expectedAudience;

  /** 허용 알고리즘 목록(화이트리스트). 예: RS256만 허용 */
  private final Set<JWSAlgorithm> allowedAlgs;

  public JwtVerifier(JwksProvider jwksProvider, String expectedIssuer, String expectedAudience) {
    this.jwksProvider = jwksProvider;
    this.expectedIssuer = expectedIssuer;
    this.expectedAudience = expectedAudience;

    // ✅ alg 화이트리스트를 서버에서 고정한다.
    this.allowedAlgs = Set.of(JWSAlgorithm.RS256);
  }

  /**
   * 토큰 검증 후 검증된 결과(주체, 클레임, 권한 등)를 반환한다.
   * 실패하면 예외를 던진다(상위에서 401/403으로 매핑).
   */
  public VerifiedJwt verify(String token) throws Exception {
    // SignedJWT: JWS 기반 JWT(서명된 JWT) 파싱
    SignedJWT jwt = SignedJWT.parse(token);

    // ---------------------------
    // 1) Header 검증 (alg 고정)
    // ---------------------------
    JWSHeader header = jwt.getHeader();

    // 헤더가 주장하는 alg를 그대로 신뢰하지 말고, 서버가 허용한 목록인지 확인
    if (!allowedAlgs.contains(header.getAlgorithm())) {
      throw new SecurityException("Disallowed alg: " + header.getAlgorithm());
    }

    // kid는 JWKS에서 어떤 키를 선택할지 결정하는 식별자
    String kid = header.getKeyID();

    // --------------------------------------------
    // 2) 공개키 조회 (캐시 우선, 미스매치 시 갱신)
    // --------------------------------------------
    JWK jwk = jwksProvider.getByKid(kid);
    if (jwk == null) {
      // kid에 대응되는 키가 없으면 서명 검증을 할 수 없음 → 401
      throw new SecurityException("No matching key for kid=" + kid);
    }

    // ---------------------------
    // 3) 서명(Signature) 검증
    // ---------------------------
    // 여기서는 RSA(RS256) 케이스로 구현
    RSAKey rsaKey = jwk.toRSAKey();

    // RSASSAVerifier: RSA 공개키로 서명 검증 수행
    JWSVerifier verifier = new RSASSAVerifier(rsaKey);

    // 서명 검증 실패 시 위변조 가능성이므로 즉시 거부
    if (!jwt.verify(verifier)) {
      throw new SecurityException("Signature verification failed");
    }

    // ---------------------------
    // 4) Claims 검증 (정책 검증)
    // ---------------------------
    JWTClaimsSet claims = jwt.getJWTClaimsSet();

    // (4-1) iss 검증: 신뢰하는 인증서버가 발급한 토큰인지 확인
    if (!expectedIssuer.equals(claims.getIssuer())) {
      throw new SecurityException("Invalid iss");
    }

    // (4-2) aud 검증: 이 서비스 대상으로 발급된 토큰인지 확인
    List<String> aud = claims.getAudience();
    if (aud == null || !aud.contains(expectedAudience)) {
      throw new SecurityException("Invalid aud");
    }

    // (4-3) exp 검증: 만료 여부 확인
    Date exp = claims.getExpirationTime();
    if (exp == null || exp.toInstant().isBefore(Instant.now())) {
      throw new SecurityException("Token expired");
    }

    // (4-4) nbf 검증(선택): 활성화 시각 이전이면 거부
    Date nbf = claims.getNotBeforeTime();
    if (nbf != null && nbf.toInstant().isAfter(Instant.now())) {
      throw new SecurityException("Token not active yet");
    }

    // ---------------------------
    // 5) 권한 정보 파싱
    // ---------------------------
    // 시스템마다 claim 스키마가 달라질 수 있어 유연하게 처리
    Set<String> scopes = parseScopes(claims);
    Set<String> roles = parseRoles(claims);

    // 최종적으로 검증된 사용자 컨텍스트 반환
    return new VerifiedJwt(
      claims.getSubject(), // sub
      claims,
      kid,
      scopes,
      roles
    );
  }

  /**
   * scope 파싱:
   * - "read:orders write:orders" 같은 공백 구분 문자열을 많이 사용
   * - 또는 ["read:orders","write:orders"] 배열일 수도 있음
   */
  private Set<String> parseScopes(JWTClaimsSet claims) {
    Object scopeObj = claims.getClaim("scope");
    if (scopeObj == null) return Set.of();

    // case 1) 문자열
    if (scopeObj instanceof String s) {
      String[] parts = s.trim().split("\\s+");
      Set<String> out = new HashSet<>();
      for (String p : parts) if (!p.isBlank()) out.add(p);
      return out;
    }

    // case 2) 배열/컬렉션
    if (scopeObj instanceof Collection<?> col) {
      Set<String> out = new HashSet<>();
      for (Object o : col) if (o != null) out.add(o.toString());
      return out;
    }

    return Set.of();
  }

  /**
   * role 파싱:
   * - ["admin","user"] 배열
   * - 또는 "admin" 문자열
   */
  private Set<String> parseRoles(JWTClaimsSet claims) {
    Object roleObj = claims.getClaim("role");
    if (roleObj == null) return Set.of();

    if (roleObj instanceof String s) {
      return Set.of(s);
    }
    if (roleObj instanceof Collection<?> col) {
      Set<String> out = new HashSet<>();
      for (Object o : col) if (o != null) out.add(o.toString());
      return out;
    }
    return Set.of();
  }

  /**
   * 검증 결과를 담는 DTO
   * - 필요하면 tenant, client_id, user_name 등도 추가 가능
   */
  public record VerifiedJwt(
    String subject,
    JWTClaimsSet claims,
    String kid,
    Set<String> scopes,
    Set<String> roles
  ) {}
}
```
