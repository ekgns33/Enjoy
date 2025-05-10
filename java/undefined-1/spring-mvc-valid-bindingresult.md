---
description: 어떤 필드가 틀렸는지 제공하자.
---

# Spring MVC - @Valid와 BindingResult로 응답 제공하기

스프링 연습을 위해 데모 프로젝트를 구현하던 도중 <mark style="color:red;">**Field Validation에서 오류**</mark>가 발생하면, **에러 메세지를 출력**하는 요구사항이 있었다.

```
{
 "success": false,
 "error": {
   "code": "INVALID_INPUT",
   "message": "상품 등록에 실패했습니다.",
   "details": {
     "name": "상품명은 필수 항목입니다.",
     "base_price": "기본 가격은 0보다 커야 합니다."
   }
 }
}
```

위 예시와 같이 데이터 필드 검증에 실패한 것들에 대해 각각 메세지를 구조화하여 응답해야한다. 지금까지는 필드 오류가 나면 단순하게 필드오류가 있음을 알려주기만하고 상세한 값들을 내려주지 않았던 것 같다. 클라이언트에게 이런 정보들은 내려주는 경험이 처음이라 글로 정리하였다.

### Spring @Valid 이란?

앞서 필드 검증에서 실패한 값들에 대해 에러 메세지를 구조화한다고 했는데 일반적인 비즈니스 로직의 흐름을 살펴보자.

<figure><img src="../../.gitbook/assets/Screenshot 2025-05-06 at 11.08.12 PM.png" alt=""><figcaption></figcaption></figure>

* 계층형 구조라고 가정하고 요청의 흐름을 생각해보면, Controller단에서 사용자의 요청을 핸들링하고 요청 파라매터, 바디를 매핑하게된다.



스프링은 `@ModelAttribute`, `@RequestBody` 와 같은 어노테이션을 제공하여 HTTP 요청을 매핑하도록 돕는다. 이 경우 새로운 DTO 클래스를 만들고 원하는 필드들을 정의하는데, **Spring Bean Validation**을 사용하여 매핑과정에서 값에 대한 검증을 할 수 있다.



따라서 서비스 로직으로 도달하지 않고도 요청을 처리하는 Controller의 매핑과정에서 검증을 수행할 수 있는 것이다.

스프링에서는 Spring Bean Validation 말고도 **Validator 인터페이스를 활용할 수도 있다.** 하지만 Spring Bean Validation의 경우 선언형으로 사용할 수 있는 장점이 있다.

```java
@Getter
@AllArgsConstructor
public class SignUpRequestDto { 
  private String name; 
  private int age; 
}
```

다음과 같은 Request Dto가 있을 때 두가지 방법으로 검증 로직을 추가할 수 있다.



**1. Validator Interface**

```java
public class SignUpRequestDtoValidator implements Validator {

  public boolean supports(Class clazz) {
    return SignUpRequestDto.class.equals(clazz);
  }


  public void validate(Object obj, Errors e) {
      ValidationUtils.rejectIfEmpty(e, "name", "이름이 공백입니다.");
      SignUpRequestDto request = (SignUpRequestDto) obj;
      if (request.getAge() < 0) { 
         e.rejectValue("age", "나이는 음수값이 아닙니다."); 
      } else if (request.getAge() > 110) { 
         e.rejectValue("age", "나이는 110세 미만으로 입력해야합니다."); 
      } 
  }
}

```

* Validator 인터페이스를 구현한다. 디테일한 커스터마이징은 해당 방법으로 가능하다. `ContraintValidator` 인터페이스를 구현하여 커스텀으로 제약을 만들 수 있다.



해당 방법으로 더욱 복잡한 조건이나 필드 검증을 할 수 있다. 그러나 **개발자가 직접 구현한 구현체이므로 등록해야한다**. 이는 `@InitBinder` 를 컨트롤러에 추가하거나 `WebDataBinder`에 수동으로 추가하여 해결한다.

```
@RestController
public class MyController {

    @Autowired
    private MyCustomValidator myCustomValidator;

    @InitBinder // 특정 DTO에 연결하기
    protected void initBinder(WebDataBinder binder) {
        binder.addValidators(myCustomValidator);
    }
}
```



**2. Spring Bean Validation**

**Spring Boot 2.3버전 이후에는 개발자가 직접 의존성을 추가해야한다.**

```
implementation 'org.springframework.boot:spring-boot-starter-validation'
```

부트과정에서 `LocalValidatorFactoryBean` 이 Spring 컨테이너에 `Jakarta Validator` 를 등록한다. 1번 방식과 다른점은 개발자가 구현해서 사용하는 것이 아닌 **Hibernate**의 validator 라이브러리를 사용한다.

<figure><img src="../../.gitbook/assets/Screenshot 2025-05-07 at 12.07.25 AM.png" alt=""><figcaption></figcaption></figure>

\
spring boot를 사용중이라면, AutoConfiguration이 활성화되면서 Validator가 주입된다.





<figure><img src="../../.gitbook/assets/Screenshot 2025-05-07 at 12.08.26 AM.png" alt=""><figcaption></figcaption></figure>

LocalValidatorFactoryBean이 디폴트로 등록이된다. LocalValidatorFactoryBean에서는 Provider을 주입한다.



<figure><img src="../../.gitbook/assets/Screenshot 2025-05-07 at 12.22.19 AM.png" alt=""><figcaption></figcaption></figure>

빈으로 등록하는 과정이 끝나면, `@Valid` 어노테이션을 선언한 부분에 자동 주입되어 검증 로직을 수행한다.

```java
@Getter
@AllArgsConstructor
public class SignUpRequestDto { 
  @NotEmpty
  private String name; 
  @Max(110)
  @Min(0)
  private int age; 
}
```

클래스에서 선언형으로 제약 조건을 명시할 수 있다.



***



### BindingResult

Validator는 요청을 객체로 바인딩하는 과정에서 필드를 검증한다. Spring에서는 데이터를 바인딩할때, `DataBinder`을 사용하는데, 검증과정에서 명시된 제약사항을 어기는 경우가 생기면 오류를 `BindingResult` 에 저장한다.



**DataBind의 결과를 담는 컨텍스트라고** 이해하면 편하다. 따라서 바인딩을 수행하는 과정이 선행되어야하는데,`@ModelAttribute` , `@RequestBody` 와 같은 어노테이션을 통해 객체가 바인딩될때 결과가 누적되어 반환된다.

**Controller의 파라매터로 BindingResult를 곧바로 얻을 수 있다.** 실행흐름을 개발자의 코드로 다시 가져오기 때문에 400번 응답이 아닌 값도 처리할 수 있다.



&#x20;하지만 모든 요청에 대해 바인딩 결과를 이렇게 **컨트롤러에서 처리하다보면 중복 코드가 많아진다.**

```java
...

public ResponseEntity<Void> post(@ModelAttribute XRequest request, BindingResult bindingResults) {

...

  if(bindingResults.hasErrors()) {
	// do something

  }

}
```



따라서 Spring Validator가 Validation과정에서 throw하는 `MethodArgumentNotValidException`을 **전역 핸들러에서 처리하고 원하는 공통 포맷으로 반환하는 것이 서버와 클라이언트 모두 편하다.**

* 물론 특별하게 처리해야하는 경우라면 컨트롤러에서 처리하면 된다. 나는 필드값오류에 대해서는 공통 예외 클래스를 활용하고 `fieldname : error phrase`와 같은 형식으로 반환했기에 에러 처리기에서 수행했다.



```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<?> handleValidation(MethodArgumentNotValidException ex) {
    BindingResult result = ex.getBindingResult();

    List<String> errors = result.getFieldErrors().stream()
        .map(e -> e.getField() + ": " + e.getDefaultMessage())
        .toList();

    return ResponseEntity.badRequest().body(errors);
}

```

원하는 형식으로 직접 매핑할 수도 있다. 기본적인 처리 코드는 위와 같다.

***
