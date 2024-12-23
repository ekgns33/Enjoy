---
description: Access OS Memory through DirectBuffer
---

# Java NIO - Direct Buffer 사용하기

## Direct Buffer

* 이전 글에서 ByteBuffer를 활용하여 시스템 메모리에 직접 접근할 수 있는 구현체인 Direct Buffer의 장단점까지 살펴보았다. 이번에는 간단하게 사용하는 방법을 살펴본다.

> C언어를 활용하여 리눅스 시스템 프로그래밍을 경험한 사람이라면, 결국 Direct Buffer을 활용하여 메모리 작업을 하는 것 이 C언어를 자바로 래핑한 것에 불과함을 알 수 있다. JNI가 애초에 Native Operation에 대한 인터페이스이므로 당연하다! 편하게 구경하자.

## 목차&#x20;

{% stepper %}
{% step %}
### 메모리 할당하기

allocateDirect()
{% endstep %}

{% step %}
### 메모리 읽기 쓰기

position이 버퍼의 오프셋!
{% endstep %}

{% step %}
### 참조를 여러개 만들기

duplicate로 여러 buffer생성하기
{% endstep %}
{% endstepper %}

***



Direct Buffer는 시스템 메모리에 직접 Java를 활용하여 IO를 할 수 있도록 제공한다. 정말 많은 함수들이 있고 오라클 문서에서 확인할 수 있다. 그 중에서 POSIX와 겹쳐보였던 것들을 소개하려한다. 더 궁금한 사람은 링크를 참조하자!

{% embed url="https://docs.oracle.com/javase/6/docs/api/java/nio/ByteBuffer.html" %}
Oracle ByteBuffer Docs
{% endembed %}

###

***

###

### 메모리 할당하기 - allocateDirect()

우선 메모리를 할당해야 읽던 쓰던 할 수 있겠지? ByteBuffer의 `allocate(int capacity)` 는 JVM의 힙에 바이트 버퍼를 생성한다. 하지만 우리는 시스템 메모리에 할당하고 싶다.

* `public static ByteBuffer allocateDirect(int capacity)` 를 사용한다.&#x20;

```java
// Direct Buffer 할당
ByteBuffer directBuffer = ByteBuffer.allocateDirect(1024);

// 확인하기
Boolean isDirect = directBuffer.isDirect();

```



내가 사용하는 버퍼가 Direct / Non-Direct Buffer인지 확인하는 방법은 두번째 메소드인 `isDirect()` 를 사용하면된다.



***



### 메모리에 자바 자료형으로 읽고 쓰기&#x20;

버퍼가 byte 단위이긴 하지만 우리는 편의상 String, int와 같은 친구들을 적고 싶다. 그래서 여러가지 구현체(IntBuffer, CharBuffer)들을 제공한다.



```java
ByteBuffer buf = ByteBuffer.allocateDirect(64);
IntBuffer ibuf = buf.asIntBuffer(); // wrap up!

ibuf.put(1024);
ibuf.put(2048);
// same as
// ibuf.put(1024).put(2048);

System.out.println(buf.position() + " : " + buf.limit() + " : " + buf.capacity());

while(buf.hasRemaining()) {
    System.out.print(buf.get() + " ");
}

System.out.println(buf.position() + " : " + buf.limit() + " : " + buf.capacity());

```

<figure><img src="../../.gitbook/assets/Screenshot 2024-11-25 at 11.37.02 AM.png" alt=""><figcaption><p>실행결과</p></figcaption></figure>

실행결과를 살펴보자

#### 1. Position, Limit, Capacity의 변화

* `position`: 현재 읽기/쓰기 위치.
* `limit`: 읽기/쓰기가 가능한 최댓값.
* `capacity`: 버퍼의 전체 크기.

read전에는 position이 0이지만, read하고 나서는 position이 10으로 변경되었다. 버퍼에서 현재 읽고 있는 오프셋을 position으로 생각하면 된다. FILE IO시에 우리는 **lseek**같은 함수들로 파일내의 오프셋을 조작했다. File IO에서도 동일하게 position, limit, capacity를 통해 **오프셋을 조정**하게 된다.

<figure><img src="../../.gitbook/assets/Screenshot 2024-11-25 at 11.49.20 AM.png" alt=""><figcaption><p>write, read 시 position의 변화</p></figcaption></figure>

#### 2. 출력 결과가 4, 8??

어째서 1024, 2048이 아닌가 생각할 수 있다. 하지만 교묘하게 코드에 IntBuffer가 아닌 ByteBuffer을 읽는다. 바이트 단위의 입력에서는 int가 4바이트로 저장된다. 그리고 1바이트는 8비트인 것을 우리는 알고있다.&#x20;

* 퀴즈 : 1024는 2진법으로 바꾸면 어떻게 될까요?

1024는 $$2^{10}$$이다. 즉 0000 0010 0000 0000 으로 표현할 수 있다. (편의상 16개의 비트를 적었다.)

이 친구를 바이트에 적으면?   \[0000 0010] \[0000 0000] > 4 0 으로 변한다! (8비트는 1바이트니까!)



"바이트로 저장한 값을 출력하니 저렇게 나오는 것이구나!" 라고 이해하면 굳또! :thumbsup:



***



### 참조를 여러개 만들기 - duplicate()

* 리눅스에서 한 파일에 대해서 fd를 복사할때 dup이나 dup2를 사용했다.
* C언어에서는 메모리에 대한 참조는 **포인터를 활용했다.**

#### Java에서는 참조를 어떻게 하지? :thinking:

* 자바는 call by value 언어임을 기억하자.
* primitive는 그냥 복사된 값이 전달되고, Reference 타입은 참조하는 주소값이 전달된다.
* 그래서 자료구조의 내부 값도 변환해야하면 우리는 파라매터로 List\<Integer> 같은 친구들을 넘겨줬다.



**그런데 우리는 메모리에 직접 접근해야하는데?**

* ByteBuffer의 duplicate를 사용한다!

버퍼의 복제본을 만들어서 참조에 사용한다. 그런데 다른 메모리공간에 버퍼를 할당하는게 아니라 대상이 되는 원본이 가리키는 메모리 공간을 참조하는 버퍼로 생성한다.

**중요 : 같은 공간을 참조하기 때문에 원본, 복제본 둘 중 하나에서 수정하면 다른쪽에서도 반영된다.**

```java
ByteBuffer buf = ByteBuffer.allocateDirect(10);

buf.put((byte)1).put((byte)2).put((byte)3);

ByteBuffer copyBuf = buf.duplicate();
copyBuf.put((byte)8);
System.out.println("\n"+ buf.position() + " : " + buf.limit() + " : " + buf.capacity());
System.out.println(copyBuf.position() + " : " + copyBuf.limit() + " : " + copyBuf.capacity());

buf.clear(); // buf의 position을 초기화
while(buf.hasRemaining()) {
  System.out.print(buf.get() + " ");
}
```

<figure><img src="../../.gitbook/assets/Screenshot 2024-11-25 at 12.13.27 PM.png" alt="" width="249"><figcaption><p>실행결과</p></figcaption></figure>

* **복사한 버퍼에서 쓰기를 한다고 다른 버퍼의 position이 바뀌지 않는다**
  * position, limit, capacity는 버퍼마다 고유한 속성이다.
* **복사한 버퍼에서 쓰기를 하면 다른 버퍼에서도 적용된다.**
  * copyBuf에서 put한 내용을 buf에서도 읽을 수 있다. 같은 메모리를 참조하기 때문이다.



버퍼를 직접 사용하는 함수들 중 몇가지를 살펴보았다. Channel을 버퍼와 함께 사용하게되면 Buffer의 속성으로 소개했던 position, limit, capacity가 정말 자주 사용된다. 메모리 공간을 직접사용하다보니 offset으로 조작하는 것에 익숙해질 필요가 있다. 다음 글에서는 Channel을 통해 IO를 하는 방법과, Scattering, Gathering을 알아보자



