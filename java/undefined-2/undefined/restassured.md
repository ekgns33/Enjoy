---
description: Acceptance Test
---

# RestAssured로 시나리오 테스트하기

**기능하나를 붙일때마다, 애플리케이션을 실행하고 postman을 돌리는 것은 이제 그만.**

토이프로젝트를 진행하면서 이전부터 테스트코드에 대한 갈증을 느꼈다. 단위테스트뿐만 아니라 E2E테스트에 대한 절실함을 느끼고 있었다. 원인은 다음과 같다.



* 에러가 없는 것 같은데 한개씩 에러가 발생하더라.
* JPA, JPQL의 실제 쿼리도 확인하고 싶은데 Mock으로 고립된 단위테스트에서는 어렵더라.
* 여러 기능을 순차적으로 실행하고 결과를 보고 싶은데 서비스들을 호출하면 테스트코드가 너무 비대해지더라.



**첫번째 이슈는 당연 개발자의 실수 혹은 잘못이라고 치부할 수 있다.**&#x20;

하지만 커밋 전에 테스트코드를 통해 바로잡을 수 있다면 "공식적인" 실수는 아니지 않을까? ㅋㅋㅋ

**두번째 이슈는 DB까지 실제로 타는 테스트를 돌려보고싶은 나의 흑심이다.**

&#x20;JPA가 알아서 해주겠지만, JPQL을 잘못 작성하거나 테이블과 연동에서 오류가 있다면 확인할 수 있다.

* JPQL을 잘못작성하면 컴파일 단계에서 에러가 발생한다.

**세번재 이슈는 서비스를 하나씩 다 불러서 상황을 만드려면 많은 Fixture들과 Mock들이 필요해졌다.**

&#x20;내가 테스트해보고 싶은 시나리오에서 각 서비스들은 X라는 행위를 할 것을 기대하는데, 서비스들을 사용하면 특정 값을 리턴하라고 다 설정을 해야했다. 코드는 점점 커졌다.



***

**그래서 인수테스트라는 것을 작성해보기로 했다.**



기술 블로그나 유튜브에서 ATDD라는 단어를 본 적이있다. TDD는 TDD인데 앞에 **A**가 붙어서 이게 뭐지.. 했었다.

이 **A** 가 바로 Acceptance이다.



* 프로젝트의 참여자(Stakeholder)가 기능에 대한 인수 조건(Acceptance - Criteria)를 정의하고, 이를 만족시켜나가면서 TDD하는 것이라고 한다.
* 클래스 단위로 고립시켜서 동작을 확인하는 것이 아니라 **요구사항에 대해서 시스템 전체가 잘 동작하는지 확인한다.**

> TDD라는 개념은 아직도 익숙하지 않다. 경험이 부족해서 그런 걸지도 모르지만 기능을 개발하기 시작할때, Outside-Inside 방향으로 TDD를 하는 것이 자연스럽지 않다.
>
> But 연습할때마다 버그를 찾아내는 것을 보아... 단위테스트와 E2E는 필수라고 생각한 다.

***



**그런데 RestAssured는 누구시죠?**

컨트롤러 테스트를 작성할때, MockMvc를 주로 사용했다. 인수테스트에서도 이를 활용하면 되지않을까 고민을 했다.

* **BUT 많은 예제들이 RestAssured를 사용하고 있다. 이유가 뭘까?**



**WebMvcTest + MockMvc?**

Spring MockMvc의 경우 Spring MVC 계층에 대한 테스트이다. 실제 네트워크를 사용하지 않고, Mock으로 내부까지 요청을 하는 방식이다. 이게 가능한 이유는 어차피 실제 네트워크를 받으면 서블릿으로 들어오게되니 그부분을 그냥 내부호출 방식으로 처리하는 것이다.

> `MockMvc` aims to provide more complete testing support for Spring MVC controllers without a running server. It does that by invoking the `DispatcherServlet` and passing ["mock" implementations of the Servlet API](https://docs.spring.io/spring-framework/reference/testing/unit.html#mock-objects-servlet) from the `spring-test` module which replicates the full Spring MVC request handling without a running server.

스프링 공식문서에서도 다음과 같이 기술한다.

"MockMvc는 실제 서버를 구동하지 않고도 MVC컨트롤러를 테스트하게 도와줌. 서블릿 API를 Mock하여 전달하기에 서버 구동없이 요청을 처리할 수 있음."





**MockMvc는 표현계층을 고립시키고 이에 대한 테스트를 할때 유용하다.**

* **컨트롤러 테스트를 진행할때는 서비스레이어를 모킹하여 고립시키고 Validation, 응답 형식을 확인하는 것에 초점을 맞췄다.**
* 실제 서버를 돌리며 E2E를 하는게 아니다 :)



**하지만 우리는 시스템 전반에 대한 인수테스트를 진행하려한다.**

* 따라서 실제 서버를 띄우고 테스트를 진행한다.
* 또한 원격의 CI서버에서 통합테스트를 진행할 수 있기 때문에 RestAssured를 사용한다.
* 개인적으로 BDD스타일의 코드작성은 이해를 쉽게만든다. :thumbsup:

***

### 비즈니스 요구 사항을 적어보기

테스트를 작성하고 싶은 기능에 대해서 요구사항을 먼저 적었다.

1. 사용자의 달리기에 대해 보상을 제공한다.
2. 1km마다 보상이 누적된다.
3. 1주일 간 첫번째 보상인 경우 특별 보상이 추가된다.



3가지의 요구사항을 만족시키기 위해서 시나리오는 다음과 같다.

* 회원가입된 사용자가 달리기를 마치고 기록한다.
* 기록된 데이터로 보상을 요청한다.

```java
@Test  
@Sql(scripts = "/sql/user_item_test_data.sql", executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD)  
void 달리기_기록_저장_후_주간_첫번째_달리기보상_수령() throws JsonProcessingException {
	...
}
```

테스트 명은 알아보기 쉽게 한글로 적었다. 환경을 설정하기 위해 SQL 스크립트를 작성했고 테스트 실행전에 먼저 스크립트가 실행된다.



**스크립트 실행 이후 데이터 저장 환경**

유저 1이 저장된다.\
유저 1의 달리기 기록들이 저장된다.\
서비스의 아이템 정보들이 저장된다.



* 3가지 정보를 저장한 이유는 다음과 같다.

주간 첫번째 달리기라면 추가보상을 지급해야하기에 데이터가 필요함.\
회원가입된 유저에 대해서만 보상을 지급하기에 유저가 필요함.\
지급하는 아이템 정보들은 DB에 저장되므로 기본 데이터가 필요함.



**유저 스토리에 따라 코드 작성**

```java
// 요청 만들기
RecordSaveRequest request = new RecordSaveRequest(  
    pivotTime,  
    pivotTime.plusMinutes(20),  
    1000L,  
    1000L);

// 1. 기록 저장
ValidatableResponse res = given()  
    .header("Authorization", header)  
    .body(objectMapper.writeValueAsString(request))  
    .contentType(ContentType.JSON)  
    .when()  
    .post("/api/v1/records")  
    .then()  
    .log().ifValidationFails()  
    .statusCode(HttpStatus.CREATED.value())  
    .body("payload", notNullValue())  
    .body("payload.saved_id", notNullValue());  
  
Integer recordId = res.extract().path("payload.saved_id");  
  
RewardClaimRequest rewardClaimRequest = new RewardClaimRequest(Long.valueOf(recordId));  

// 2. 보상 요청  
given()  
    .header("Authorization", header)  
    .body(objectMapper.writeValueAsString(rewardClaimRequest))  
    .contentType(ContentType.JSON)  
    .when()  
    .post("/api/v1/rewards/runnings")  
    .then()  
    .log().all()  
    .body("payload", notNullValue());
```

단순히 Request를 만들고 요청을 보내는 코드들이지만, 원하는 요구사항을 만족했는지 확인하기에 적절했다.

* **테스트 이름으로 어떤 요구사항에 대한 시나리오인지 확인**
* **테스트 코드에서 어떤 데이터로 테스트했는지 확인**



**배운점**

**ATDD를 했다기 보다는 구현을 먼저하고 E2E 테스트를 작성했다에 가깝다.**

* 그럼에도 불구하고 이제는 테스트를 위해 **버튼하나만 누르면 되는** 효율을 얻었다.
* 팀원들이 테스트코드를 봤을때 어떤 기능에 대한 테스트인지 알아보기 쉽다고 말했다.

**결국 중요한건 테스트를 작성하는 것**

* 테스트코드를 작성할만한 클래스, 함수인지
* Mock은 어디서 쓰고 데이터는 어떻게 세팅할 것인지
* 이런 것들이 더 중요한 것 같다.



