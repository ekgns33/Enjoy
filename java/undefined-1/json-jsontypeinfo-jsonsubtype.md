---
description: 직렬화할때 다형성을 처리하는 방법
---

# Json 직렬화에 활용하는 @JsonTypeInfo, @JsonSubType

### Spring Boot 그리고 Jackson

Spring Boot를 사용하면 Json 직렬화에 기본적으로 Jackson을 사용한다. 특정 Dto를 매핑할때 ObjectMapper을 사용하는데, 내부적으로는 Jackson을 활용하게 된다. 오늘 다뤄볼 것은 Jackson의 @JsonType, @JsonSubType이다.&#x20;

JsonSerialize를 해서 외부에 보내야하는 이벤트와 같은 객체들에 대해서 다형성 처리를 할 때 사용한다.

### @JsonTypeInfo, @JsonSubType이란?

다형성을 지원하기 위한 어노테이션으로 직렬화/역직렬화 시 구체적인 타입을 결정하도록 타입 정보를 포함시킨다. 인터페이스나 추상 클래스에서 구현 클래스에 대한 타입 정보를 직렬화 시 include하도록 할 수 있다.

### @JsonTypeInfo 의 속성들

#### **use**

<mark style="color:red;">필수적으로</mark> 포함해야하는 속성값은 `use` 라는 속성이 있다. 이 속성은 타입 메타데이터의 종류를 가리킨다. 타입을 구분하는데 사용한다.

* `JsonTypeInfo.Id.Class` : 자바 클래스명
* `JsonTypeInfo.Id.Name` : 논리적 타입 이름
* `JsonTypeInfo.Id.MINIMAL_CLASS` : 최소 경로 Java 클래스 명
* `JsonTypeInfo.Id.DEDEUCTION` : 직렬화된 타입 프로퍼티 없이 프로퍼티를 기반으로 **타입 추론**
* `JsonTypeInfo.Id.None` : 명시적인 타입 메타데이터 X

```
@JsonTypeInfo(
  use = JsonTypeInfo.Id.Class
)
```

#### **include**&#x20;

optional하게 `include` 라는 속성도 지정할 수 있다. 해당 속성은 타입 메타데이터를 포함하는 방법을 지정한다.



* `JsonTypeInfo.As.PROPERTY`: 별도의 메타 프로퍼티로 포함 (기본값)
* `JsonTypeInfo.As.WRAPPER_OBJECT`: JSON 객체로 감싸서 포함
* `JsonTypeInfo.As.WRAPPER_ARRAY`: 2요소 JSON 배열로 래핑
* `JsonTypeInfo.As.EXTERNAL_PROPERTY`: 한 단계 상위 계층에 프로퍼티로 포함
* `JsonTypeInfo.As.EXISTING_PROPERTY`: 기존 프로퍼티 사용

```
@JsonTypeInfo(
  use = JsonTypeInfo.Id.Class,
  include = JsonTypeInfo.As.PROPERTY
)
interface AAAA {}
```

#### **property**

optional하게 지정하는 속성인데, 타입 정보를 저장할 프로퍼티의 이름을 지정한다.

```
@JsonTypeInfo(
  use = JsonTypeInfo.Id.Class,
  include = JsonTypeInfo.As.PROPERTY
  property = "type"
)
interface AAAA {}

```

위와 같이 설정하면 type이라는 json 필드에 역직렬화하는 데이터의 타입이 삽입된다.

#### **visible**

optional하게 설정할 수 있는데, 타입 식별자 값을 역직렬화 할때 전달할지를 결정한다. 기본값은 <mark style="color:red;">false</mark>이다.

#### **defaultImpl**

optional 하게 설정하는데, 타입 식별자가 없거나 매핑이 불가능할때 사용할 기본 구현 클래스를 지정할 수 있다.

{% embed url="https://fasterxml.github.io/jackson-annotations/javadoc/2.4/com/fasterxml/jackson/annotation/JsonTypeInfo.html" %}

### @JsonSubTypes

@JsonTypeInfo와 함께 사용되는데 직렬화 가능한 다형성 타입의 하위 타입들을 지정하고, Json 필드에서 사용되는 논리적인 이름을 연결한다. 해당 어노테이션으로 우리가 특정 타입을 Json에서 어떻게 분류할지 정할 수 있다.

```
@JsonTypeInfo(
	use = JsonTypeInfo.Id.NAME,
	include = JsonTypeInfo.As.PROPERTY
    property = "type"
)
@JsonSubTypes({
    @JsonSubTypes.Type(value = SubClass1.class, name = "type1"),
    @JsonSubTypes.Type(value = SubClass2.class, name = "type2")
})
public abstract class TypeA {}
```

가령 위와 같이 어노테이션을 사용할 수 있다.

1. 먼저 JsonTypeInfo에 대해 속성을 지정한다.
2. 이후 JsonSubTypes로 하위 클래스들에 대한 명세를 추가한다.

&#x20;       \-  TypeA라는 추상클래스의 하위 클래스로 SubClass1, SubClass2가 존재한다.



**>> 우리는 두 클래스를 직렬화 / 역직렬화할때 구분할때,  json 에서 "type" 이라는 필드를 보고 구분할 수 있다.**

#### JsonTypeInfo.include 좀 더 살펴보기

`include` 라는 속성이 꽤나 중요한데, Json의 구조까지 변경될 수 있어서 살펴봐야한다.

* AS.PROPERTY는 말 그래도 Json 프로러트에 타입 정보를 심겠다는 의미라서

```
{ "type": "type1", "field1" : "a" }
```

다음처럼 표시가 된다.

* AS.WRAPPER\_OBJECT는 JSON 객체로 감싸서 포함하는 것인데 실제로 사용해보진 않았다. 위에서 속성만 바꿔서 Json을 만들면

```
{
  "type1": {
    "field1" : "a"
  }
}
```

처럼 Json이 만들어진다. 특정 클래스를 key로 삼아 래핑하는 방식이다.

### JsonTypeInfo의 보안 경고

필수 속성인 `use` 에 Id.Class를 사용해서 직렬화하거나, `ObjectMapper.enableDefaultTyping`은 보안 위험이 있다고한다.

* 역직렬화 가젯을 통한 원격 코드 실행 공격

**Json에 클래스 이름을 담게되면 역직렬화 하는 과정에서 이 데이터가 어떤 클래스의 인스턴스로 만들어져야할지 포함된다.** 이러면 공격자가 서버내의 다른 허용하지 않은 클래스를 인스턴스화 할 수 있기 때문에 위험하다.

**요 이슈는 순수 Java에서도 동일하게 적용된다.**

Java는 Serializable 인터페이스를 구현하는 객체들을 ByteStream으로 직렬화했다가, ObjectInputStream으로 역직렬화하는 방식으로 객체를 복원한다.

그런데 Stream에 포함된 클래스 이름과 필드를 신뢰하여 JVM 클래스패스의 실제 존재하는 클래스를 로딩하고 필드를 복원하기 때문에 보안 취약점이 있었다. 초기 직렬화에서는 캘래스명, 필드 정보를 모두 포함했었지만 자바 버전이 업그레이드 되면서 역직렬화를 허용할 클래스 목록, 객체 그래프 제한, 필드/객체 개수 제약등의 정책이 추가됐다.

### How to Solve?

`use = Id.NAME` 처럼 개발자가 직접 명시적으로 허용한 타입 정보만 사용한다.

