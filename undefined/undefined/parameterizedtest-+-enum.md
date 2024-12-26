---
description: 여러 변수로 한번에 테스트하기
---

# @ParameterizedTest + Enum

유닛 테스트를 작성하다가 한번에 여러가지 경우를 테스트할 수 없을까? 라는 생각을 하다가 찾아보게 되었다.

* 하나의 함수에 여러가지 input이 있고 결과가 다른 경우

여러가지 input을 한번에 테스트할 때 JUnit에 `@ParameterizedTest`  라는 친구를 활용할 수 있는 것은 알고 있었지만 정확한 사용법은 몰랐기에 이번 기회를 통해 정리해보려고 한다.

***

### 여러 Enum중 특정 Enum이 아니면 false를 리턴하는지 확인하고 싶다.



내가 테스트하려는 경우는 위와 같았다. 하나의 함수를 테스트하지만 입력값이 여러 종류인 경우다.



<pre class="language-java"><code class="lang-java">public boolean isModifiable() {
<strong>    if(!DomainStatus.isWaiting(this.status)) {
</strong>        return false;
    }
    return true;
}
</code></pre>

어떤 객체의 상태를 확인하고 변경이 가능한지 true / false를 반환하는 함수가 있었다. 객체의 상태는 Enum으로 관리되고 있었고, `DomainStatus.WAITING`  인 경우에만 변경이 가능했다.



처음에 테스트코드를 다음과 같이 작성하려 했다.

```java
void test_is_modifiable_when_given_status() {
    // given
    Domain d = DomainFixtures.getCommonFixture();

    // when
    d.changeStatus(DomainStatus.CANCELED);
    
    // then
    assertFalse(d.isModifiable());
}
```

도메인 객체의 기본상태는 `WAITING` 이다. 그런데 `WAITING` 이 아닌경우 저 메소드는 false를 반환하는 지 확인하고 싶었다. 따라서 상태를 바꾸고 테스트하려는 메소드인 `isModifiable` 을 호출하여 비교했다.

* CANCELED말고도 여러가지 상태가 존재했기에 모든 상태에 대해서 테스트를 진행하고싶었다.&#x20;

but 위와 같은 방식으로는 **확인하고자 하는 상태마다 테스트코드를 작성해야하기**에 input으로 여러 상태를 주면 한번에 테스트를 진행할 수 있는 코드를 원했다. 그리고 `@ParameterizedTest`   를 봤었던 기억이 나서 적용하려했다.

***

### @ValueSource, @NullSource, @EnumSource

* ParameterizedTest를 사용하려면 어노테이션을 붙이고 어떤 입력값들이 있는지 설정해줘야한다.&#x20;

Baeldung에서 참고한 코드를 보면서 간단하게 익혀보자.

{% embed url="https://www.baeldung.com/parameterized-tests-junit-5" %}

```java
boolean isEmptyString(String s) {
    return s == null || s.trim().isEmpty()
}

@ParameterizedTest
@ValueSource(strings = {"", " "})
void isEmptyString_ShouldReturnTrueForNullOrBlankString(String input) {
    assertTrue(isEmptyString(input));
}
```

이렇게 함수를 작성하고, XXXSource에 내가 넣고싶은 입력들을 직접 타이핑하여 여러 변수에 대한 테스트를 진행할 수 있다. 이전까지는 `@ValueSource`  만 사용해봤기에 Enum도 그대로 사용하면 될 것 같았는데 아니었다.

* **Enum은 Enum전용 `@EnumSource` 를 사용한다!**

역시 공식 문서를 한번씩 보고 적용해야하는게 정답이다.. 열거형 클래스는 따로 어노테이션이 존재해서 참고하여 사용했다.

```java
public boolean isModifiable() {
    if(!DomainStatus.isWaiting(this.status)) {
        return false;
    }
    return true;
}
```

위에서 적었던 테스트하려는 함수이고

```java
@ParameterizedTest
@EnumSource(
    value = DomainStatus.class,
    names = {"WAITING"}
    mode = EnumSource.Mode.EXCLUDE)
void exceptWaiting_OthersReturnFalse(DomainStatus status) {
    // given
    Domain d = DomainFixtures.getCommonFixture();

    // when
    d.changeStatus(status);
    
    // then
    assertFalse(d.isModifiable());

}    
```

* value : 사용하려는 Enum 클래스
* names : 입력하려는 대상 (string으로 적었지만 형변환을 자동으로 해준다.)
* mode : EXCLUDE, MATCH\_ANY등 모드를 설정할 수 있다.

&#x20;나는 WAITING이 아니라면 False를 반환하기를 원했기 때문에 EXCLUDE 모드를 사용했다. 그런데 여기서는 읽기 쉬운 방식으로 적는게 좋을 것 같다.&#x20;

&#x20;코드의 의도가 Enum의 개수가 적고 WAITING말고는 몰라도 되면 EXCLUDE로 해도 문제 없어 보이지만, 다른 상태들이 매우 중요하다면 EXCLUDE가 아니라 그냥 입력하려는 Enum들을 적어서 x, y, z, q ... 상태들은 안되는구나\~ 를 인지시켜줘도 좋을 것 같다.&#x20;

***



### DisplayName 설정하기

* 아마도 테스트 코드를 작성하고나서 짜릿한 순간중 하나는 초록색 체크표시들이 나란히 출력되는 화면을 바라보는 것이다. 테스트가 어떤 행위를 테스트하는지, 어떤 입력을 검증하는지를 한눈에 보기 편하게 작성하면  결과만 보고도 위치를 찾을 수 있다.

@ParameterizedTest에서는 어노테이션 자체에 name이라는 속성을 사용한다. 예시는 다음과 같다.



```java
@ParameterizedTest(name = "{index} {0} 상태에서는 수정이 불가합니다.")
@EnumSource(
    value = DomainStatus.class,
    names = {"WAITING"}
    mode = EnumSource.Mode.EXCLUDE)
void exceptWaiting_OthersReturnFalse(DomainStatus status) {
    // given
    Domain d = DomainFixtures.getCommonFixture();

    // when
    d.changeStatus(status);
    
    // then
    assertFalse(d.isModifiable());

}  
```

여러가지 속성이 있는데 몇가지만 확인해보자.

1. {index} : 실행 index이다. N번의 입력테스트라면 1 \~ N까지 변한다.
2. {arguments} : 우리가 입력한 변수들의 리스트를 comma로 구분하여 표시한다.
3. {0}, {1} .. : 입력 변수 각각의 placeholder로 {0}에는 status가 담긴다. 첫번째 인자니까\~

<figure><img src="../../.gitbook/assets/Screenshot 2024-12-26 at 7.40.13 PM.png" alt=""><figcaption></figcaption></figure>

***

### 다른 어노테이션과 함수들?

* 위에 참고한 링크에서 살펴보면 알 수 있듯이 정말 많은 종류의 함수들이 있다.
  * csv, null, empty, ... 많은 종류의 source가 있다.
  * _심지어 메소드도 인자로 넘겨서 테스트할 수 있다._
  * 명시적 / 암묵적 형변환에 대한 내용도 나중에 쓸일이 있으면 봐야할 것 같다. string으로 입력해도 알아서 Enum이나 DateTime으로 변환해준다. 물론 직접 지정할 수 있다.
* 다음에 다른 source를 쓸때마다 업데이트를 해보려한다!

***

**참고한 자료**

[https://www.baeldung.com/parameterized-tests-junit-5](https://www.baeldung.com/parameterized-tests-junit-5)

