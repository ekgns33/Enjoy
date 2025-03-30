---
description: 리스트를 json으로 디비에 저장해보자
---

# Json을 DB에 저장하기

우리가 RDBMS를 사용하면, 엔티티 간의 관계에 따라 정규화를 진행한다. Key들을 이용하여 연관관계를 나타내고 테이블을 분리하여 데이터를 저장한다.

* 그런데 모든 데이터를 정규화하여 저장해야할까?

사용자에게 보여지기만 하는 모델이나, 복잡한 내용이 엮여있을 때는 MongoDB와 같은 비정규형 데이터베이스를 사용하기도한다. NoSQL의 장점을 이용하는 것이다.

현재 진행하고 있는 프로젝트는 MySQL을 메인 데이터베이스로 사용중이다. 대부분의 데이터는 정규화되어 저장되어있지만 새로운 데이터를 저장해야하는 상황이 왔다.

**달리기 데이터 중 km별 페이스 정보를 저장해야한다.**

달리기 데이터는 클라이언트가 수집하여 저장만 서버가 담당한다. 따라서 서버입장에서는 변경되지 않는 정적인 데이터이다. 달리기 기록 ID는 보상을 지급할때 활용된다. 하지만 나머지 데이터들은 저장된 이후 조회용으로만 사용된다.

## Collection 정보를 저장하는 법?

JPA에서 Collection을 사용할때는 크게 두가지로 나뉜다.

1. `@OneToMany` 와 같은 연관관계 설정
2. `@ElementCollection` 을 활용한 값 컬렉션 매핑

**km별 페이스 정보는 값 그자체이다.**

따라서 1번은 제외한다. 그럼 2번을 고려해볼만한데 2번의 경우 값 컬렉션 매핑을 위한 테이블이 생성된다.

**테이블을 만들면서까지 저장할 데이터인가 고려해봤을때, 그정도는 아니라고 생각이 들었다.**\
그저 View를 위해 제공하기 때문이다. 서비스에서 직접적으로 데이터를 사용하지도 않는다.

* km별 페이스 정보를 단순 문자열로 표현했을때, 크기가 그렇게 크지 않을 것 같았다. 또한 데이터베이스에 저장할때마다 변환을 해줘야하기 때문에 성능 이슈가 발생할 수도 있다. 부하를 받은 적이 없어서 얼마나 지연이 발생할 지 모르겠다.
* 테이블을 분리한다고해도, eager-join을 해야한다. 테이블의 크기또한 기록이 커지면 매우 커질 것이다.

***

### 문자열로 저장하면 데이터 크기가 어느정도 될까?

```
[
  {"distance": 1.0, "pace": 732000},
  {"distance": 2.0, "pace": 650000},
  {"distance": 3.0, "pace": 605000},
  {"distance": 4.0, "pace": 615000},
  {"distance": 5.0, "pace": 590000},
  {"distance": 6.0, "pace": 600000},
  {"distance": 7.0, "pace": 620000},
  {"distance": 8.0, "pace": 630000},
  {"distance": 9.0, "pace": 610000},
  {"distance": 10.0, "pace": 600000}
]

```

화면에 보여지는 데이터를 그대로 저장한다면 이와 같은 형식으로 저장이된다.

문자열로 저장한다고 했을때, 계산을 해보자. 평균적으로 사람들은 5km를 달린다고 가정한다.

유니코드 1자리 : 3byte\
문자열 길이 : 5km달렸을때, 166글자\
크기 : $3 \* 166 = 498$ (약 0.5Kb)

테이블의 다른 데이터와 메타데이터까지 생각해서 어림잡으면, 메타 데이터 (261 bytes) + 페이스 데이터 (498 bytes) = 759 bytes (약 0.75KB) 정도라고 예상된다.

* 만약 10만개의 기록데이터가 쌓이면 얼마나 데이터가 쌓일까?

$100000 \* 0.75kb = 75mb$ 정도 일듯하다. 인덱스까지 고려해서 50% 더 준다고 해보면, 110 \~120 mb일 것 같다.

이정도면 그냥 저장해도 될 것 같다.

***

### AttributeConverter을 활용하여 객체를 문자열로 변환

그럼 데이터 객체를 저장할 수 있는 문자열로 변환해야할텐데, 좋은 방법이 있을까?

* 컨테이너 객체에 toString을 만들기
* Converter 사용하기



**컨테이너 객체에 toString만들기**

생각할 수 있는 가장 간단한 방법이다. 서비스에서 직접 문자열 변환을 하고 그대로 String으로 저장하는 것이다.

```
class RunningRecord {
	...
	String segmentPaceList;
}

```

뭔가 저런식으로 문자열을 저장하게 될 것 같은데, 선뜻 손이 가지 않는다. 데이터를 표현하는 변수명은 List인데 막상 DB에 저장하는 매핑값이다보니 String 타입이다.

DB에 저장할 때, String conversion을 해야하는 것을 모르는 사람이 봤을때는 이해하기 어려울 것 같다.



**Converter을 사용하기**

`jakarta.persistence.AttributeConverte` 을 사용하면, 엔티티의 필드를 저장하거나 조회할때, 자동으로 변환할 수 있다.

https://docs.oracle.com/javaee/7/api/javax/persistence/AttributeConverter.html

* **결국 문자열로 변환하는 것은 동일하지만 엔티티에 String으로 저장할 필요없이 도메인타입으로 저장한다.**
* 변환 과정 또한 추상화되기에 이 방법을 택했다.

**두 가지 함수를 구현해야한다.**

**`convertToDatabaseColumn`**

* 엔티티의 필드를 데이터베이스에 저장될 값으로 변환하는 함수이다.

**`convertToEntityAttribute`**

* 반대로 데이터베이스에서 가져온 값을 엔티티의 필드로 변환하는 함수이다.

```java
public class SegmentPaceConverter implements AttributeConverter<List<SegmentPace>, String>  
{  
  private ObjectMapper objectMapper = new ObjectMapper();  
  
  @Override  
  public String convertToDatabaseColumn(List<SegmentPace> segmentPaces) {  
    try {  // 문자열 변환
      return objectMapper.writeValueAsString(segmentPaces);  
    } catch (Exception e) {  
      throw new RuntimeException("Failed to convert segmentPaces to JSON", e);  
    }  
  }  
  
  @Override  
  public List<SegmentPace> convertToEntityAttribute(String s) {  
    TypeReference<List<SegmentPace>> typeRef = new TypeReference<List<SegmentPace>>() {};  
    try {  
      return objectMapper.readValue(s, typeRef);  
    } catch (Exception e) {  
      throw new RuntimeException("Failed to convert JSON to segmentPaces", e);  
    }  
  }  
}
```

다음과 같이 AttributeConverter를 구현한 클래스를 만들어주고 엔티티의 필드에 어노테이션을 붙여준다.

문자열 변환 시에는 `objectMapper.writeValueAsString(xxx)` 을 사용한다. 이 친구는 테스트코드에서 body에 들어갈 값을 문자열 변환할 때도 사용한 친구이다.

***

TypeReference로 이어진다..

{% content-ref url="../undefined/jackson-typereference.md" %}
[jackson-typereference.md](../undefined/jackson-typereference.md)
{% endcontent-ref %}



