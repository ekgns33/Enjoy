---
description: JMeter 설치, 로그인 시나리오로 기본 익히기
---

# JMeter 설정, 시나리오 작성 기본

* 웹소켓을 사용하여 실시간 연결 서비스를 제공하는데, 기존의 방식과 적용하려는 방식이 얼마나 차이가 있는지 확인하기 위해 부하테스트를 진행한다.

## 1. Jmeter 환경 설정, 시나리오 작성

***

### 왜 JMeter?

우선 부하테스트 툴을 정해야했다. 기존에 사용해본 것은 <mark style="color:blue;">k6</mark>라는 그라파나사에서 만든 오픈소스 툴로 go기반의 경량 프레임워크이다. 자바스크립트로 스크립트를 작성하고 터미널에서 실행하면, 테스트 결과가 바로 분석되어 확인할 수 있었다.

* 이번에는 Jmeter?

주변에서는 k6를 추천해줘서 처음 사용했는데 스크립트를 js로 작성하는 부분에서 시간이 오래 걸렸다. GPT와 함께 낑낑대면서 노력해봤지만 시나리오 기반의 부하테스트, 통합테스트를 한번도 작성해보지 않은 나로선 어려운 작업이었다.

* **GUI가 있는 프레임워크를 사용하고 싶다...**

Jmeter는 swing기반의 gui를 제공하는 것을 보고 마음이 기울었다. 리소스 소모라던가.. js로 스크립트 파일 하나만 작성하면 끝나는 편의성만을 고려하면, k6가 더 좋은 선택일 수 있겠지만 당장에는 투입되는 인력이 빠르게 작업할 수 있는 툴을 선택하는게 맞다고 생각이 들었다. CI에 부하테스트를 넣고 인프라에서 돌린다면 k6로 전환할 수도 있을 것 같다. 물론 그만큼 리소스를 먹는다면 말이다.

{% hint style="info" %}
k6와 JMeter에 대한 비교는 그라파나에서 공식으로 제공하기도한다.

[https://grafana.com/blog/2021/01/27/k6-vs-jmeter-comparison/](https://grafana.com/blog/2021/01/27/k6-vs-jmeter-comparison/)
{% endhint %}

***

### 어쨌든 설치 및 시작

* 설치는 여기서 [https://jmeter.apache.org/download\_jmeter.cgi](https://jmeter.apache.org/download_jmeter.cgi)

파일 압축을 해제하고 JMeter을 작동하는데 CLI에서 무시할 수 없는 문구를 확인했다.

<figure><img src="../../.gitbook/assets/Screenshot 2025-01-23 at 3.02.38 AM.png" alt=""><figcaption><p>test with cli plz!!</p></figcaption></figure>

* 부하테스트를 진행할때, GUI를 사용하지 마라고 친절하게 달아주었다. GUI 프로그램 자체도 JVM위에서 동작하기 때문에 리소스를 많이 먹는다. (인텔리제이를 떠올리자...)

나의 경우 시나리오를 작성하고 부하테스트를 해보는 과정을 익히는 것이 가장 큰 목적이기에 GUI로 여러가지를 작성했다.

***

### 테스트에서 고려할 것들.

시나리오는 다음과 같다.

1. **누군가 실시간 세션을 만든다. (HTTP)**
2. **사람들이 실시간 세션에 참가하여, 상호작용을 진행한다. (WebSocket)**

정말 간단한 두 단계처럼 보이지만, 각 사용자들은 로그인도 진행해야한다. **단순한 HTTP 시나리오가 아니라 웹소켓을 사용하는 부분도 들어가서 로직이 복잡했다.**



* **STOMP의 특수성**

현재 프로젝트에서는 STOMP 프로토콜을 사용한다. 웹소켓 프로토콜을 기반으로 메세지 형식을 STOMP에 맞게 사용하는데, 스프링에서 STOMP를 위한 구현이 이미 존재해 편하게 개발할 수 있다.

STOMP 프로토콜의 동작 방식은 아래 그림과 같다. 스프링 공식 문서에서 제공하는 그림으로 내장 브로커를 사용하는 경우다.

<figure><img src="../../.gitbook/assets/Pasted image 20250123031222.png" alt=""><figcaption><p><a href="https://docs.spring.io/spring-framework/reference/web/websocket/stomp/message-flow.html">https://docs.spring.io/spring-framework/reference/web/websocket/stomp/message-flow.html</a></p></figcaption></figure>

STOMP은 다음과 같은 흐름으로 진행된다.

1. 서버와 웹소켓 커넥션을 맺는다.
2. CONNECT 메세지로 서버와의 연결을 완료한다.
3. SUBSCRIBE 메세지로 특정 토픽에 구독한다.
4. SEND 메세지로 메세지를 전송한다.

1-2-3-4 일련의 과정이 **순차적으로 진행되야한다.**

기존에는 Postman으로 한땀한땀 연결하고 메세지 수정하고, id수정하고 요청하고... 말도 안되는 수작업을 했었다.

* 개발자 a.k.a 자동화 기계

이제는 암울했던 과거는 저멀리. 시나리오를 작성해서 자동화하자!

***

## **2. 로그인 시나리오 작성**

로그인을 해야 기능들을 사용할 수 있기 때문에 모든 테스트에는 로그인 로직이 포함된다.

1. 로그인 요청을 전송한다.
2. 응답에서 받은 JWT 토큰을 헤더에 저장한다.

이 두가지를 수행하는 작업을 만들어보자.

### **2.1 사용자 그룹 만들기 (클라이언트)**

* Test Plan에 Thread Group을 추가한다.&#x20;

<figure><img src="../../.gitbook/assets/Screenshot 2025-01-23 at 3.17.23 AM.png" alt=""><figcaption></figcaption></figure>

우클릭해서 추가하고, 메타데이터를 입력한다. 사진의 예시에서는 11명의 유저가 1번 실행하는 작업임을 의미한다.

* **Number of Threads** : 사용자 수
* **Ramp-up Period** : 얼마나 빠르게 사용자를 추가할 건지. 이게 뭐냐면, 사용자를 한번에 모두 추가하는게 아니라 Ramp-up Period에 걸쳐 추가하는 것이다.
* **Loop Count** : 반복 횟수

***

### **2.2 Controller 정의하기**&#x20;

<figure><img src="../../.gitbook/assets/Screenshot 2025-01-23 at 3.21.57 AM.png" alt=""><figcaption></figcaption></figure>



이제 실질적 로직을 구현하는 Controller을 추가해야한다. 사진과 같이 여러종류의 컨트롤러가 있는데, 트랜잭션 컨트롤러를 골랐다.

### **2.3 데이터 셋 import하기**

* 사용자 데이터는 데이터베이스에 저장되어있다. 시나리오 테스트에서는 csv 파일을 넘겨주면 이를 한줄 씩 읽어서 사용할 수 있다. 테스트 더미데이터를 만들어서 `users.csv` 파일로 저장했다.

내가 만든 csv에서는 사용자들의 id들만 포함하고 있다.

* 읽은 데이터를 **클라이언트의 변수**로 저장한다.

variable로 저장하면 나중에 그 값을 스크립트에서 사용할 수 있다.

<figure><img src="../../.gitbook/assets/Screenshot 2025-01-23 at 3.27.36 AM.png" alt=""><figcaption></figcaption></figure>

### **2.4 작업 만들기**



<figure><img src="../../.gitbook/assets/Screenshot 2025-01-23 at 3.30.41 AM.png" alt=""><figcaption></figcaption></figure>

* 로그인을 진행시키려 하기 때문에 HTTP Request Sampler을 만든다.&#x20;

#### **JWT 받은 거로 Request Header에 설정하기**

응답에서 값을 추출하고 헤더에 적용하기 위해서는 두가지 단계가 필요하다.

1. JsonExtractor로 추출하여 variable에 저장하기
2. Http Header Manager에서 설정하기

#### **Sampler에 Extractor을 추가한다.**

<figure><img src="../../.gitbook/assets/Screenshot 2025-01-23 at 3.33.35 AM.png" alt=""><figcaption></figcaption></figure>

* Json에서 path를 잘 찾아서 저장해야한다.

<figure><img src="../../.gitbook/assets/Screenshot 2025-01-23 at 3.34.02 AM.png" alt=""><figcaption></figcaption></figure>

Header Manager을 만들어 각자 설정에 맞게 수정한다. 내 프로젝트에서는 Authorization 헤더에 저런식으로 저장하기 때문에 바꿔준다.

{% hint style="info" %}
요청이 실패하는 경우가 생길 수 있는데, _**content-type을 명시하지 않으면 text로 요청이 날라가기 때문에 추가로 설정해야한다.**_
{% endhint %}



### **2.5 결과 확인하기**

결과는 Listener의 View Result Tree로 확인할 수 있다.

<figure><img src="../../.gitbook/assets/Screenshot 2025-01-23 at 3.36.33 AM.png" alt=""><figcaption></figcaption></figure>

* Listener에는 결과 트리를 제외하고도 여러가지 친구들이 있다. 각자 필요한 리스너를 사용하면 된다.
* 나는 Response를 확인할때는 View Result Tree를 사용하고, 전체 테스트의 summary를 볼때는 `Summary Report`와 `jp@gc - Response times Over Time` 을 사용했다. 후자의 경우는 플러그인으로 추가해야한다.

플러그인 설치는 아래 블로그를 참고했다.

[https://sooo-9.tistory.com/37](https://sooo-9.tistory.com/37)



***



시나리오에 많은 작업들이 있는데, 양이 너무 많아서 나누어 작성하려한다! 프로메테우스와 그라파나로 결과를 확인하는 부분까지 호흡이 상당히 긴 작업이었는데 차근차근 적어보려한다 :)



참고한 자료들

* JMeter 플러그인 설치 : [https://sooo-9.tistory.com/37](https://sooo-9.tistory.com/37)
* STOMP 프로토콜 in Spring : [https://docs.spring.io/spring-framework/reference/web/websocket/stomp/message-flow.html](https://docs.spring.io/spring-framework/reference/web/websocket/stomp/message-flow.html)
* 헤더 설정 : [https://stackoverflow.com/questions/13032753/how-to-send-request-with-content-type-header](https://stackoverflow.com/questions/13032753/how-to-send-request-with-content-type-header)

***

