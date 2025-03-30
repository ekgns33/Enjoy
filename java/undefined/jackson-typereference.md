---
description: Type Erasure을 극복하는 방법
---

# Jackson이 제네릭 타입을 보존하는법 -TypeReference

### TypeReference?

이번 기능 개발에서 처음 마주한 클래스라서 정리를 해보려한다.

현재 프로젝트에서 직렬화/역직렬화에 `Jackson` 라이브러리를 사용중이다. TypeReference 역시 잭슨 안에 내장된 클래스이다.

> This generic abstract class is used for obtaining full generics type information by sub-classing; it must be converted to ResolvedType implementation (implemented by JavaType from "databind" bundle) to be used. Class is based on ideas from http://gafter.blogspot.com/2006/12/super-type-tokens.html , Additional idea (from a suggestion made in comments of the article) is to require bogus implementation of Comparable (any such generic interface would do, as long as it forces a method with generic type to be implemented). to ensure that a Type argument is indeed given.

javadocs에 나온 설명이다.

* Jackson으로 JSON 데이터를 특정 타입으로 역직렬화할 때, **제네릭 타입**인 경우 타입 정보가 지워지기 때문에 정확한 타입을 제공하기 위해 사용한다고 한다.
* 타입이 지워진다는 것은 Type Erasing을 의미하는데, 자바에서는 런타임에 제네릭은 타입이 지워진다.

**Type Erasure + TypeReference**

[https://www.baeldung.com/java-type-erasure](https://www.baeldung.com/java-type-erasure)

우리의 초록사이트에서 친절하게 설명해준다. 정리를 하자면

자바는 컴파일 타임에는 타입 제한을 강제하지만 런타임에는 무시한다는 것이다.

**우리는 런타임에도 지정한 타입으로 변환을 시켜줘야하는데 타입이 지워지면 해결할 수 없다.**

```
ObjectMapper objectMapper = new ObjectMapper();
String json = "[{\"name\": \"John\"}, {\"name\": \"Jane\"}]";

List<Person> people = objectMapper.readValue(json, List.class);
```

* 런타임에 `Person` 이라는 타입으로 변환을 해야하는데, 모든 타입이 지워져 `List<Object>` 로 역직렬화한다. 따라서 우리가 원하는 변환이 불가능해진다.

**여기서 `TypeReference<T>` 가 사용되는 것이다. 강제로 타입을 담고있는 레퍼런스객체를 두고 런타임에 이를 사용하여 타입추론을 하는 것이다.**

주석에 나와있는 링크 [https://gafter.blogspot.com/2006/12/super-type-tokens.html](https://gafter.blogspot.com/2006/12/super-type-tokens.html) 를 확인하면 어떤 과정으로 이 클래스가 탄생했는지 나와있다.

**SuperTypeToken 패턴**

* 익명 내부 클래스로 TypeRefernce를 상속받고 이를 통해 익명클래스의 타입정보를 보존한다.
* 익명 내부 클래스의 타입정보는 런타임에도 남아있는 점을 이용한다.
* 저장한 익명 내부 클래스의 상위클래스가 `TypeReference<List<XXX>>` 이므로 ParameterizedType으로 캐스팅하여 추출한다.

```java
protected TypeReference()  
{  
    Type superClass = getClass().getGenericSuperclass();  
    if (superClass instanceof Class<?>) { // sanity check, should never happen  
        throw new IllegalArgumentException("Internal error: TypeReference constructed without actual type information");  
    }  
    /* 22-Dec-2008, tatu: Not sure if this case is safe -- I suspect  
     *   it is possible to make it fail?     *   But let's deal with specific     *   case when we know an actual use case, and thereby suitable     *   workarounds for valid case(s) and/or error to throw     *   on invalid one(s).     */    _type = ((ParameterizedType) superClass).getActualTypeArguments()[0];  
}
```

우리가 `TypeReference<List<Person>>` 을 사용하면, 내부에 익명 클래스가 생성된다. 코드에서 보이는 것 같이

```java
Type superClass = getClass().getGenericSuperclass();  
```

익명 클래스의 Class객체를 얻어와서 상위클래스의 타입 정보를 얻는다.

* 상위 클래스는 `TypeReference<List<Person>>` 이 된다.

이를 `ParameterizedType` 으로 캐스팅한다.

* `ParameterizedType` 은 제네릭을 표현하는 인터페이스이다.
* 우리는 여기서 `Person` 타입을 얻어 `_type` 에 저장한다.

**이때 부터는 제네릭이 아니라 타입을 저장했기때문에 런타임에 타입정보가 보존된다.**

테스트를 돌려보자

```java
@Test  
void typeReference() {  
  TypeReference<List<SegmentPace>> type = new TypeReference<List<SegmentPace>>() {};  
  System.out.println(type.getType());  
  
  List<SegmentPace> list = new ArrayList<>();  
  System.out.println(list.getClass().getGenericSuperclass());  
  
}
```

* 첫번째는 타입을 보존하기에 우리가 저장한 타입이 나올것이고,
* 아래 친구는 제네릭으로 인해 타입이 지워졌을 것이다.

<figure><img src="../../.gitbook/assets/Screenshot 2025-03-30 at 4.22.54 PM.png" alt=""><figcaption></figcaption></figure>

첫번째는 타입이 보존되었지만 두번째는 E로 날아갔다.

***

**배운점**

* 자바의 Type Erasure 방식
* Generic에서 타입 지우기를 어떻게 TypeReference는 해결했는지
