---
description: OAuth2.0, OIDC, 회원 탈퇴
---

# 애플 소셜로그인, 회원 가입/탈퇴 구현하기

iOS앱을 배포하기 위해서는 애플 로그인을 필수로 구현해야한다. 카카오 로그인까지는 적용을 시켰었는데 이번 기회에 애플 로그인 연동을 했다.

소셜 로그인의 방식에는 OAuth2 방식과 OIDC 방식이 있다.

### OAuth 2.0

OAuth2.0은 사용자 인증에 사용되는 개방형 프로토콜로, 서드파티 애플리케이션 측에서 사용자의 비밀번호 없이도 해당 사용자의 리소스를 조회할 수 있는 방법이다.

* AccessToken을 통해 리소스 접근을 권한 관리한다.
* 인증보다는 인가에 가깝다.
* 서비스 제공자의 API를 이용할 수 있는 권한을 부여한다.

<figure><img src="../../.gitbook/assets/image.png" alt=""><figcaption><p><a href="https://upload.wikimedia.org/wikipedia/commons/7/72/Abstract-flow.png">https://upload.wikimedia.org/wikipedia/commons/7/72/Abstract-flow.png</a></p></figcaption></figure>

**1. 소셜 로그인을 진행하는 사용자의 정보는 리소스 소유자 측에 있다.**

* 구글, 애플, 카카오등 사용자 정보를 제공하는 측이 데이터를 소유한다.
* 이 회사들은 OAuth2.0 프로토콜을 통해 리소스에 대한 액세스 권한을 부여한다.

**2. 서드파티 애플리케이션은 이 리소스에 접근할 수 있도록 권한을 얻어야한다.**

* 소셜 로그인을 통한 정보를 필요로하는 우리(서드파티)는 권한이 필요하다.
* Authorization Server에 권한을 얻기 위해 통신한다.
* Authorization Server에서 받는 데이터는 클라이언트로 노출하지 않고 서버가 통신한다.

**3. 얻은 권한으로 리소스 서버에 사용자 정보를 요청한다.**

* 2의 과정을 통해 우리는 리소스에 접근할 권한을 받았다. 이제 요청한다.
* 이때 Authorization Server로부터 받은 AccessToken을 사용한다.

간추렸지만 3개의 과정을 통해 서버는 사용자의 정보를 소셜 벤더측으로부터 얻고 이를 활용하여 자체 회원가입을 진행한다.

***

### OIDC - OpenID Connect

OIDC는 OAuth2.0을 확장한 **인증 프로토콜**로, 사용자 인증에 필요한 ID 토큰을 추가로 제공한다.

* OIDC는 인증에 목적을 둔다.
* OAuth2.0 기반으로 구축되었다.

**OAuth2.0과 다른점?**

* **OIDC는 인증에 초점을 둔다.**
* OAuth2.0에서는 AccessToken만 제공하지만, OIDC는 ID 토큰까지 제공한다.
* 사용자 정보는 표준화된 클레임으로 제공된다.
  * email, name, 식별자등

OAuth2.0에서는 권한을 부여받고 토큰으로 리소스를 조회했다면, OIDC에서는 사용자가 실제 벤더사에 인증된 회원인지만 인증하고 표준클레임의 최소정보만 전달받는다.

**선택은 당신의 몫**

* OIDC는 OAuth2.0의 확장팩인 개념이다. 따라서 소셜로그인을 구현할때 둘 중 어떤것을 택해도 상관없다.
* 나의 경우 우리 서비스에서는 애플, 카카오의 **다른 API들을 사용하지 않는다**. **따라서 사용자의 신원 검증 및 정보 조회만 필요하여 OIDC를 선택했다.**

***

## 애플 로그인을 구현하기

소셜로그인을 처음 구현할때 상당히 삽질을 많이 했었다. 하지만 공식문서와 OAuth2.0 프로토콜의 흐름을 이해하면 다음부터는 쉽게 연동할 수 있다.

**흐름**

1. iOS 클라이언트에서 SDK로 인증을 받고 서버에 Authorization Code와 Verifier을 전달한다.
2. 서버는 애플 연동시 개발자계정에서 발급받은 Key값으로 Apple Client Secret을 얻는다.
3. 이를 사용하여 애플에 토큰을 요청한다.
4. 서비스 회원가입을 시킨다.

***

#### Secret Key 생성

APPLE은 시크릿키를 동적으로 생성하는 것을 권장한다. 따라서 요청하기 전에 개발자계정에서 발급받은 키로 Secret Key를 생성해야한다.

```java

private String generateAppleClientSecret() {  
    try {  
        // 현재 시간 및 6개월 후 만료 시간 설정  
        long nowMillis = System.currentTimeMillis();  
        long expMillis = nowMillis + 180L * 24 * 60 * 60 * 1000; // 180일  
        byte[] decodedKey = Base64.getDecoder().decode(formattedKey);  
  
        // 개인 키 생성  
        KeyFactory keyFactory = KeyFactory.getInstance("EC");  
        PKCS8EncodedKeySpec keySpec = new PKCS8EncodedKeySpec(decodedKey);  
        PrivateKey privateKey = keyFactory.generatePrivate(keySpec);  
  
        // JWT 클라이언트 시크릿 생성  
        return JWT.create()  
            .withHeader(Map.of("kid", appleKeyId))  
            .withIssuer(appleTeamId)  
            .withIssuedAt(new Date(nowMillis))  
            .withExpiresAt(new Date(expMillis))  
            .withAudience("https://appleid.apple.com")  
            .withSubject(clientId)  
            .sign(Algorithm.ECDSA256((ECKey) privateKey));  
  
    } catch (Exception e) {  
        log.error("Failed to verify Apple access token", e);  
        throw new RuntimeException("Failed to generate Apple client secret", e);  
    }  
}

```

* 애플 PRIVATE 키를 발급받아본 사람은 알겠지만 일정한 형식이 요구된다.

```
-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
```

* 이는 애플리케이션에서 처리하거나 자체로 저장해서 사용하면된다.

***

#### ID Token 발급받기

* 위에서 만든 SecretKey로 ID token을 발급받자. 클라이언트로부터 전달받은 `authcode` 와 `verifier` 을 담아서 전달해야한다.

```java
HttpHeaders headers = new HttpHeaders();  
headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);  
  
MultiValueMap<String, String> map = new LinkedMultiValueMap<>();  
map.add("client_id", clientId);  
map.add("client_secret", generateAppleClientSecret());  
map.add("code", authCode);  
map.add("grant_type", "authorization_code");  
map.add("redirect_uri", redirectUri);  
map.add("code_verifier", codeVerifier);  
  
HttpEntity<MultiValueMap<String, String>> request = new HttpEntity<>(map, headers);  
  
ResponseEntity<String> response = restTemplate.postForEntity(  
    "https://appleid.apple.com/auth/token", request, String.class);

```

* 이제 전달받은 데이터에서 서비스에 필요한 데이터를 사용하면된다.

***

### 받은 데이터로 바로 유저를 저장?

간단한 구현일 줄 알았지만 문제는 여기서 생겼다. 서비스의 UI 플로우와 요구사항이 다음과 같았다.

{% hint style="info" %}
**"소셜 로그인이 끝나고 서비스에 필요한 정보들을 더 받고 회원가입 시킨다."**
{% endhint %}

소셜로부터 인증은 끝났지만 아직 정보를 입력하지 않아서 회원가입을 시킬 수는 없는 상황이 되었다. 즉, 인증의 시점과 가입의 시점이 다르다는 것인데 문제를 쪼개보자.

흐름은 다음과 같다.

1. 사용자가 인증을 하면 사용자 데이터를 서버가 받는다.
2. 클라이언트가 개인 정보를 더 받아서 회원가입을한다.
3. 서버는 그 정보를 받아서 회원가입 시킨다.

3번의 흐름에서 서버는 그 클라이언트의 신원을 파악할 수 없다. **인증 시점과 가입 시점이 다르기 때문이다.** 따라서 클라이언트의 상태를 유지해야한다.

***



### **상태를 어떻게 유지하지?**

사용자의 상태를 유지하는 방법으로는 크게 2가지가 있다. **세션과 토큰.**

세션은 메모리나 데이터베이스에 클라이언트의 정보를 일정 기간 서버에서 유지하는 방법이다. 토큰의 경우 클라이언트 - 서버는 무상태지만 carry-on 토큰에 정보를 담아서 사용한다.



**우리는 데이터베이스에 UUID와 소셜 정보를 임시로 저장하기로했다.**

1. 식별할 수 있는 방법이 필요한데 정수값은 위험하다고 판단했다.
2. 단순 UUID만 저장하는 것이 아니라 소셜 정보도 일부 저장해야한다.



***

### **소셜 정보를 왜 저장해야하지?**

회원가입 대상자라면, **JWT 토큰의 페이로드에 소셜정보들을 담아서 클라이언트에 제공하는 방안이 있었다.**

* **회원가입 탈퇴를 위해 refresh\_token을 데이터베이스에 저장해야했다.**

서버에서 이를 저장하지 않으면 회원탈퇴 과정에서 애플로그인을 해야하는 기괴한 UI흐름이 생긴다. refresh\_token은 서비스마다 애플에서 발급하는 영구적인 값이기 때문에 노출하고 싶지 않았다.&#x20;

탈취되었을 때 위험성은.. 익명 나쁜 사람이 사용자를 계속 탈퇴시킨다?는 점이다. 애플에서는 애플이 제공하는 정보들에 대해 보안성을 지킬것을 요구하기에 외부로 노출하지 않기로 했다.



**결정된 흐름**



<figure><img src="../../.gitbook/assets/Screenshot 2025-04-15 at 6.27.39 PM.png" alt=""><figcaption><p>sequence-diagram</p></figcaption></figure>

* 서버에서는 인증 이후에 받은 정보를 **임시 테이블에 저장한다.**
* 클라이언트에 회원가입만을 위한 임시 토큰을 발급한다. 이때 Expire Time을 10분 이내로 짧게 가져간다.
* 클라이언트는 정보와 함께 토큰을 서버에 다시 전달한다. **내가 인증된 사용자라는 것을 증명하는 수단이다.**
* 서버에서는 토큰을 검증하고 최종 회원가입 시킨다.

***

**후기**

회원 탈퇴를 고려하지 않고 설계를 했다가 애플에서 제공한 리프레쉬 토큰이 필요한 것을 알았다. 이후 초기 설계를 엎고 다시 구현을 했어야했다. 애플 로그인은 처음이었지만 오히려 이 경험을 통해 더 많이 배운것 같다.

**참고한 자료**

{% embed url="https://guide.ncloud-docs.com/docs/b2bpls-oauth2" %}

{% embed url="https://oauth.net/articles/authentication/" %}

{% embed url="https://developer.apple.com/documentation/accountorganizationaldatasharing/generate-and-validate-tokens" %}

{% embed url="https://developer.apple.com/documentation/accountorganizationaldatasharing/generate-and-validate-tokens" %}
