---
description: CSRF는 무엇인가? Spring에서는 어떻게 처리하는가?
---

# Spring Security - CSRF

{% stepper %}
{% step %}
### CSRF란 무엇인가
{% endstep %}

{% step %}
### HTTP에서는 상태를 어떻게 유지할까

쿠키는 어떻게 자동으로 보내는걸까

쿠키 속성, RFC
{% endstep %}

{% step %}
### CSRF 는 어떻게 해결하지?
{% endstep %}

{% step %}
### Spring Security에서는 어떻게 CSRF를 대비할까
{% endstep %}
{% endstepper %}

## 1. CSRF란 무엇인가?

**Cross Site Request Forgery**

* **사이트 간 요청 위조**라는 웹 보안 공격이다.

인증된 사용자의 권한을 도용해, 나쁜 사이트에서 특정 웹사이트에 요청을 보내도록 하는 공격이다. 은행 서비스를 사용하던 사용자는 로그인을 하면 세션이 생성된다. 이 세션이 유지되는 상태에서 공격자의 **나쁜사이트**에 접속하게되면 공격자가 사용자의 세션을 사용하여 송금과 같은 요청을 보내고, 서버는 세션 정보가 일치하기에 정상 요청으로 생각해 처리해버리는 것이다.



흐름을 다시 정리해보면

1. 사용자가 은행에 로그인하여 세션 쿠키를 얻고 서비스를 이용한다.
2. 클릭을 잘못하거나 광고를 눌러서 공격자의 사이트로 접속한다.
3. 공격자의 사이트에서 **사용자 브라우저를 통해 은행에 악성요청**을 보낸다.
4. 브라우저는 세션을 요청에 자동으로 넣어서 보낸다.
5. 서버는 **세션정보가 일치하니 사용자라고 생각**해서 요청을 처리한다.



**웹 브라우저는 서버가 응답으로 전달한 세션ID를 쿠키에 저장하고, 사용자가 같은 사이트에 요청을 보낼때 세션ID가 담긴 쿠키를 자동으로 요청 헤더에 넣어 전송한다.**

{% hint style="info" %}
**HTTP 트랜잭션은 상태가 없다. 각 요청 및 응답은 독립적으로 일어난다.**

* HTTP 완벽가이드 11장
{% endhint %}

많은 웹사이트에서는 사용자의 상태를 남겨 여러 상호작용이 가능하도록 하길 원한다. 위에 적어놓았듯 **HTTP는 무상태**이기 때문에 웹사이트는 각 HTTP 요청을 식별할 방법이 필요하다.

## 2. HTTP에서 어떻게 클라이언트를 식별할까?



### **1. HTTP 헤더**

HTTP 헤더 중 사용자 정보를 표시하는 헤더가 여럿 존재한다. From 헤더에서는 사용자의 이메일 주소를 넣을 수 있지만 이를 누군가 수집하면 피싱의 대상이 될 수 있다. User-Agent는 브라우저의 버전, 이름, OS 정보등을 기술한다. Referer 헤더는 어떤 경로로 해당 페이지로 유입됐는지 표시한다.

<figure><img src="../../.gitbook/assets/Screenshot 2025-04-24 at 4.09.19 PM.png" alt=""><figcaption><p>크롬 개발자 도구</p></figcaption></figure>

* 실제로 스프링 공식문서 페이지 내에서 옮겨다니면 Referer 에 `https:docs.spring.io` 가 찍힌다.
* `User-Agent` 에는 내 OS와 브라우저까지 찍혀있다.

**하지만 이 정보로 사용자를 식별하기는 부족하다.**



### 2. IP 주소?

HTTP 헤더에 IP 주소 정보가 들어가지는 않는다. 네트워크 계층이 다르다! but 서버 입장에서는 TCP 커넥션을 요청한 클라이언트의 IP 주소를 알 수 있다.

하지만 IP 주소는 사용자를 가리킨다고 단언할 수 없다.

집에 있는 내 컴퓨터로 접속한사람이 나인지 부모님인지 알 수 없다. **IP주소는 디바이스를 가리키기** 때문이다. 또한 일반적인 가정의 경우 KT, SKT와 같은 ISP 의 인터넷을 사용하는데 이들은 별도로 신청하지 않는 이상 **동적으로 IP주소를 할당해준다.(DHCP를 찾아본다)**



<figure><img src="../../.gitbook/assets/image (1).png" alt=""><figcaption><p><a href="https://en.wikipedia.org/wiki/Proxy_server">https://en.wikipedia.org/wiki/Proxy_server</a></p></figcaption></figure>

프록시 서버를 사용하는 경우에도 식별이 어려워진다. 캐싱, 보안 등을 위해 프록시 서버를 활용하는 경우에 클라이언트는 프록시서버의 IP를 보게된다. 또한 실제 서버도 프록시서버로부터 요청을 포워딩 받기 때문에 활용하기 어려워진다.

* 물론 `X-Forwarded-For` 을 사용할 수도 있다.&#x20;

{% embed url="https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Headers/X-Forwarded-For" %}

* 프록시나 중간 서버를 거치는 경우 실제 IP를 얻기위해 해당 헤더를 사용한다. 하지만 그만큼 Private한 정보이기 때문에 조심히 사용해야한다.



### 3. 로그인 + 헤더

* 서버에서는 사용자의 이름과 비밀번호로 인증하는 것을 요구하여 식별할 수 있다.
* 이 정보를 Authorization 헤더에 넣어 요청을 보내면 모든 요청에 로그인 정보가 포함되기에 서버에서 확인할 수 있다.
* **but 헤더정보에 실제 데이터를 그대로 넣어서 전달하는 방법은 너무 위험하다.**



### 4.쿠키

* 사실상 가장 많이 사용하는 방법이다.

쿠키에는 **세션쿠키(Session Cookie)**&#xC640; **지속쿠키(Persistent Cookie)**&#xAC00; 있다. 세션 쿠키는 사용자가 사이트를 이용하면서 쌓은 데이터를 저장하는 쿠키로 브라우저 종료시에 삭제된다. 지속 쿠키는 말그대로 더 길게 유지된다. 이는 디스크에 저장되기에 우리가 웹을 껐다 켜도 계속 저장된다. 지속쿠키는 expire 시간을 지정할 수 있다.



<figure><img src="../../.gitbook/assets/Screenshot 2025-04-24 at 7.34.49 PM.png" alt=""><figcaption><p>크롬 개발자 도구</p></figcaption></figure>

**서버는 클라이언트에게 요청을 반환할때 Set-Cookie 헤더를 설정**하고, 클라이언트는 이를 받아들여 쿠키를 저장한다. 구글 크롬의 개발자 도구를 켜보면 쿠키가 저장되는 것도 확인할 수 있다.



브라우저는 이를 관리하는데, **사이트마다 일부 쿠키만** 전달한다.

* 모든 쿠키를 전달하면 성능이 안좋음
* 각 사이트가 식별하는 쿠키가 의미가 있음.
* 개인정보 문제



### **우리가 궁금했던 "브라우저가 뭔데 마음대로 사용하지?" 가 여기서 나온다.**



브라우저에서 서버로 요청을 보낼때, 같은 도메인, 경로, 보안속성에 맞는 URL로 요청을 보낸다면 브라우저는 저장된 쿠키를 헤더에 넣어 전송한다.

쿠키에는 몇가지 자주 사용되는 속성이 있다.



1. Secure : https 에서만 쿠키를 전송한다.
2. http-only: javascript에서 쿠키를 접근하는 것을 막는다.
3. **domain : 명시한 도메인에 대해서만 쿠키를 전송한다.**
4. **path : 명시한 서버의 resource에만 쿠키를 할당할 수 있다.**
5. expires: 쿠키의 지속 시간



**여기서 우리가 궁금했던 내용은 domain과 path쪽에 있음을 짐작할 수 있다.**

domain과 path의 기본 동작은 MDN 공식문서나 `RFC 6265`에서 확인가능하다.

{% embed url="https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Cookies" %}
MDN 쿠키 공식문서
{% endembed %}

{% embed url="https://www.rfc-editor.org/rfc/rfc6265.html#section-5.2.4" %}
HTTP 쿠키 RFC문서
{% endembed %}



<figure><img src="../../.gitbook/assets/Screenshot 2025-04-24 at 5.01.14 PM.png" alt=""><figcaption></figcaption></figure>

* **도메인이 명시되지 않으면 서버의 호스트를 기본으로 사용한다.**\


<figure><img src="../../.gitbook/assets/Screenshot 2025-04-24 at 5.04.16 PM.png" alt=""><figcaption></figcaption></figure>

* **Path도 Set Cookie를 전달하는 URL의 경로가 사용된다.**

<mark style="color:red;">**결국 웹 표준에 따라 브라우저는 쿠키를 자동으로 넣어서 보내는 것이다!**</mark>



***



## CSRF는 어떻게 해결할까?

* 쿠키가 자동으로 보내지면 CSRF는 어떻게 해결해야할까?

**1. 각 요청에 고유한 토큰을 넣어 서버가 이 토큰의 유효성을 검증하도록한다.**\
<mark style="color:red;">악의적인 사이트였다면 토큰 검증에 실패</mark>할 것이다. 따라서 쿠키가 존재하더라도 토큰 검증에 실패한다.

**2. Referer을 검증한다.**\
기존 도메인이 아니라 악의적인 호스트로부터 온 요청이라면 Referer에 그 사이트 주소가 찍힌다. 이를 활용하여 출처를 검증한다.

**3. CAPCHA**\
우리가 수없이 많이 사진 맞추기를 하던 캡챠다. 사용자가 직접 입력해야하는 프로세스를 두어 자동으로 처리되는 것을 막는다.

**4. SameSite 쿠키**

MDN 공식문서에 자세히 나와있다.

{% embed url="https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Cookies#controlling_third-party_cookies_with_samesite" %}

쿠키의 SameSite 속성을 사용해, 외부 사이트의 쿠키가 전송되지 않도록 제한한다.

* Strict : 동일한 사이트만 허용한다.
* Lax : 동일 사이트와 상위 레벨 탐색 (HTTPS GET)에 대해서는 허용
* None : HTTPS 일때만 허용. 서드파티 쿠키가 필요할 때 사용.

이전에 CSRF때문에 클라이언트를 작성할때 문제가 생긴적이 있었다. 이때 SameSite를 여러버전으로 바꿔보는등 삽질을 했었는데, default는 Lax이다.

***

## Spring Security에서 CSRF를 처리하는 법

스프링 시큐리티에서는 CSRF 필터에 대한 인터페이스를 제공하여 간편하게 설정할 수 있게 해준다.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(Customizer.withDefaults()); // 기본 CSRF 활성화
        return http.build();
    }
}

```

<figure><img src="../../.gitbook/assets/Screenshot 2025-04-24 at 5.29.43 PM.png" alt=""><figcaption><p><a href="https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html#csrf-token-repository-httpsession">https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html</a></p></figcaption></figure>

`Configuration`에서 위와 같이 CSRF를 활성화 할 수 있다. 스프링 시큐리티는 필터체인을 통해 여러가지 보안 사항을 하나씩 차례로 검증한다. 그리고 CSRF는 `CsrfFilter`에서 이를 처리한다.

`springframework.security.web.csrf.CsrfFilter.java` 를 살펴보면 필터가 공통적으로 구성하는 메소드인 `doFilterInterval(...)` 에 확인할 수 있다.

시퀀스 다이어그램은 스프링 공식문서에서 제공해준다.

<figure><img src="../../.gitbook/assets/Screenshot 2025-04-24 at 7.44.09 PM.png" alt=""><figcaption><p><a href="https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html#csrf-token-repository-httpsession">https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html</a></p></figcaption></figure>

* **대강 흐름을 따라가면, 토큰의 존재여부를 보고 가져와서 valid한지 확인하는 과정이다.**

<figure><img src="../../.gitbook/assets/Screenshot 2025-04-24 at 5.32.15 PM.png" alt=""><figcaption></figcaption></figure>

* CookieCsrfTokenRepository 구현체에서 `loadToken` 메소드를 보면 request쿠키에서 원하는 쿠키를 조회한 후 반환한다.

<figure><img src="../../.gitbook/assets/Screenshot 2025-04-24 at 5.34.42 PM.png" alt=""><figcaption></figcaption></figure>

이후 CsrfHandler에게 **해당 요청이 csrf 체크를 해야하는지 확인하는데**, 내부클래스로 정의되어있다. `allowMethods` 라는 Set에서 메소드가 포함되어있는지를 확인하고 결과를 리턴한다.

* 재밌는 점은 Default핸들러를 `private static`으로 정의하고 **객체 자체를 static하게 초기화해놨다.**
* EMPTY 오브젝트 같은것을 클래스에 정의해두고 쓰는 경우가 있는데 스프링에서도 이를 활용하는 것 같다.

<figure><img src="../../.gitbook/assets/Screenshot 2025-04-24 at 5.37.59 PM.png" alt=""><figcaption></figcaption></figure>

* `doFilterInternal()` 의 검증 부분을 살펴보면 더 명확히 알 수 있다.
* PathMatcher을 지나 이제는 **진짜 검증해야하는 요청이라면** 스프링 시큐리티의 토큰을 로드하여 비교한다.
  * **스프링 시큐리티는 기본적으로 CSRF 토큰을 HttpSession에 저장한다.**
  * `CookieCsrfTokenRepository` 를 사용할때는 XSRF-TOKEN헤더에 쿠키를 적고 이를 요청마다 확인한다. 만약 클라이언트가 직접적으로 쿠키를 사용하지 않는다면 `HttpOnly` 를 사용하는 것을 보안상 권고한다.

**커넥션이 수립되면, CSRF 토큰을 스프링이 내부에 만든다. 클라이언트는 요청마다 토큰을 사용하여, CSRF 검증을 한다는 것이 결론이다.**

***

## Stateless로 설정하고 사용중이라면?

방금 default는 세션에 저장하는 방식이라고 소개했는데, 만약 Stateless하게 서버를 사용하고 싶다면 어떻게 할까?

**CSRF는 사용자의 인증정보를 세션이 들고있는데 이를 활용하는 부분에서 생기는 문제점이다**

<mark style="color:red;">서버에서 사용자와 상태를 유지하지 않는다면 애초에 문제가 생기지 않는다.</mark>

하지만 다른 문제가 도래한다. **사용자인지 알려면 상태를 유지해야한다며.** 이를 해결하기 위해 토큰을 사용하는 방법이 있다. 토큰을 사용하는 방법은 다음에 알아보자!

***

#### 배운점

* csrf 공격
* 여러가지 Http Header
* Cookie 속성
* 스프링시큐리티의 CSRF 처리방식

참고한 자료

HTTP 완벽 가이드 - Oreilly

Spring security - [https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html](https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html#csrf-token-repository-httpsession)

MDN Reference&#x20;

* [https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Cookies)
* [https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Headers/X-Forwarded-For](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Headers/X-Forwarded-For)







***
