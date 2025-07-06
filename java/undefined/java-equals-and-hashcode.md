---
description: 두 메소드를 항상 같이 만들자
---

# Java - equals & hashCode

{% stepper %}
{% step %}
### 해시값은 같은데, Equals는 다른 경우
{% endstep %}

{% step %}
### 해시값은 다른데, Equals만 같은 경우
{% endstep %}
{% endstepper %}

자바의 객체들은 모두 Object 클래스를 상속한다. Object 클래스에는 여러 메소드들이 있는데 그 중 객체의 동등성을 판단하는 equals와 hashcode를 오늘 알아보려한다.

<figure><img src="../../.gitbook/assets/Screenshot 2025-07-06 at 9.24.38 PM.png" alt=""><figcaption></figcaption></figure>

자바 독스를 확인해보면 `hashCode` 와 `equals` 가 항상 같이 엮여서 언급된다. 두 메소드가 같이 언급되는 이유는 다음과 같다. 객체가 동일하다면 두 객체의 해시값이 같다. 하지만 equals 값이 다른경우에는 문제가 발생할 수 있다.



## 1. 해시값은 같은데 equals는 다른 경우

```
private static class ValueObject {  
  
    private final String value;  
    private final Integer version;  
  
    public ValueObject(String value, Integer version) {  
        this.value = value;  
        this.version = version;  
    }  
  
    @Override  
    public boolean equals(Object obj) {  
        return false;  
    }  
  
    @Override  
    public int hashCode() {  
        return Objects.hash(value, version);  
    }  
}
```

ValueObject라는 클래스를 하나 만들어서 시험을 해보자.

* 값으로 사용할 클래스를 만들고 해시값은 value와 version이라는 값으로 만들도록 구성한다.
* **equals는 무조건 false를 반환하도록 오버라이딩했다.**



```
ValueObject vo = new ValueObject("a", 1);  
ValueObject vo2 = new ValueObject("a", 1);  
System.out.println(vo == vo2);  
System.out.println(vo.hashCode() == vo2.hashCode());  
System.out.println(vo.equals(vo2));
```

값으로 판별시에 동일한 객체 두개를 생성하고 동일성, equals, hashCode를 비교하면&#x20;

<figure><img src="../../.gitbook/assets/Screenshot 2025-07-06 at 9.00.35 PM.png" alt=""><figcaption></figcaption></figure>

`new` 를 사용해서 2개의 객체를 생성했기에 `==` 비교는 false, hashCode는 true, equals는 false 출력된다.



### **문제는 해시기반 자료구조**

해시 기반 자료구조에서는 hash값으로 데이터를 저장할 위치를 선정한다. 해시값으로 버킷이나 초기 위치를 정하게되는데 이때 Equals의 구현이 중요해진다. 두 가지 문제가 발생할 수 있다.



1. **원하는 객체를 찾지 못할 수 있다.**
2. **해시 충돌이 여러번 발생할 수 있다 : 의도치 않은 객체가 중복으로 삽입될 수 있다.**



**HashMap에서는 hashcode로 버킷을 찾고 equals를 기준으로 실제 객체를 찾는다.**



<figure><img src="../../.gitbook/assets/Screenshot 2025-07-06 at 9.03.28 PM.png" alt=""><figcaption></figcaption></figure>

* **따라서 equals가 다르면 우리가 찾고자하는 객체를 찾지 못할 수 있다!**



```
Map<ValueObject, Integer> map = new HashMap<>();  
map.put(vo, 1);  
map.put(vo2, 2);  
ValueObject vo3 = new ValueObject("a", 1);
System.out.println(map.size());
System.out.println(map.get(vo3));
```

앞서 만들었던 두 객체를 순서대로 삽입하고 코드를 실행하면



<figure><img src="../../.gitbook/assets/Screenshot 2025-07-06 at 9.07.12 PM.png" alt=""><figcaption></figcaption></figure>



<mark style="color:red;">비즈니스적으로 동일한 객체이지만 equals가 달라서 2개의 객체가 삽입되었다</mark>.&#x20;

<mark style="color:red;">map.get() 을 수행했을 때 객체를 찾지못하고 null을 반환한다.</mark>

## 2. 해시값이 다른경우

```
@Override  
public boolean equals(Object obj) {  
    return value.equals(((ValueObject) obj).value)   
        && version.equals(((ValueObject) obj).version);  
}  
  
@Override  
public int hashCode() {  
    int rand = (int) (Math.random() * 100);  
    return rand;  
}
```

`equals` 는 값으로 판단하고 해시값은 랜덤으로 생성하도록 설정했다.



```
ValueObject vo = new ValueObject("a", 1);  
ValueObject vo2 = new ValueObject("a", 1);  
System.out.println(vo == vo2);  
System.out.println(vo.hashCode() + " " +  vo2.hashCode() + " " + (vo.hashCode() == vo2.hashCode()));  

```

동일한 코드를 다시 실행하고, 출력할때 두 해시값이 다른것만 확인했다.



```
ValueObject vo3 = new ValueObject("a", 1);  
System.out.println(vo3.hashCode());  
System.out.println(map.size());  
System.out.println(map.get(vo3));
```

마찬가지로 새로운 vo3를 만들어서 get으로 질의했다.



<figure><img src="../../.gitbook/assets/Screenshot 2025-07-06 at 9.13.58 PM.png" alt=""><figcaption></figcaption></figure>

* <mark style="color:red;">**초기 v0, v1의 해시값이 다르기때문에 map에는 2개의 객체가 삽입됐다.**</mark>
* <mark style="color:red;">**b어떤 값을 get할지 알 수 없다. 컬렉션 프레임워크의 안정성이 깨진다.**</mark>
* <mark style="color:red;">**해시값이 다르기 때문에 v3로 질의했을때 일치하는 값을 찾을 수 없었다.**</mark>

***

## Equals와 HashCode는 항상 같이 구현하자.

실제로 테스트를 해보면서 어떤 경우에 문제가 발생하는지 확인했다. 개발을 하면서 Hash기반의 자료구조는 떼놓을 수 없는 자료구조이며 시스템의 안정성에 크게 작용한다.

### <mark style="color:red;">**클래스를 만들때 Equals를 구현해야한다면 항상 HashCode를 함께 오버라이딩하여 구현하자**</mark>



