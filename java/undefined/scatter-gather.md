---
description: Scatter / Gather에 대해서 알아보자.
---

# Scatter / Gather

***

{% stepper %}
{% step %}
### Scatter / Gather IO?

Scattering/Gathering은 사실 Java의 고유한 기능이 아니다?
{% endstep %}

{% step %}
### Zero-Copy를 위한 Scatter / Gather

Zero-Copy란 뭘까? Scatter/Gather은 왜 필요할까?
{% endstep %}
{% endstepper %}

***



### 1. Scatter/Gather IO?

* Scatter : 흩뿌리다. 산발하다.
* Gather : 모으다.

영어사전에 Scatter이라는 단어를 검색하면 위와 같은 한글뜻을 제공한다. 흩뿌리다, 산발하다. 이름으로만 들었을때는 IO를 쪼개서 실행하는? IO를 나누어 실행하는 느낌을 받았다. 실제로는 어디에서 어떻게 사용되는 것일까?





**1.1 Linux의 Scatter/Gather**

Scatter을 구글에 입력하면 몇가지 블로그 글과 스택오버플로우 질문글이 나타났다. 스택오버 플로우의 한 질문을 확인하면서 다음과 같은 내용을 알게되었다.

* 리눅스 운영체제에서 gather은 `readv` 이며, 여러개의 버퍼를 한번에 읽는 기술이다.
* `readv` 는 연속적이지 않은 데이터 블록을 동시에 작업할 수 있게해준다.
* 짝꿍인 scatter은 `writev` 이다.

**수업시간에 읽었던 Linux System Programming 책을 다시 들여다 보자.**

챕터 4.1에 벡터 입출력이라는 고급 파일 입출력 파트가 있다.

> 벡터 입출력은 **한번의 시스템 콜을** 사용하여 **여러개의 버퍼 벡터에** 쓰거나 여러 개의 버퍼 벡터로 읽어들일 때 사용하는 입출력 메서드이다.

* 선형 입출력 메서드에 비해 몇가지 장점을 갖는다.

1. **좀 더 자연스러운 코딩 패턴** : 구조체 여러개가 잇을때 한번에 입출력 가능하다는 것을 의미하는 것 같다.
2. **효율** : 여러번의 시스템콜을 대체할 수 있다. 이게 포인트인 것 같다.
3. **성능** : 시스템콜의 호출을 줄이고, 내부적으로 최적화된 구현을 제공한다.
4. **원자성** : OS가 동작하는 IO는 atomic하지만 여러번의 systemcall이라면 직접 원자성을 보장해야한다. 하지만 동시에 여러 버퍼에 한번의 시스템콜로 IO작업을 하기에 원자성을 OS가 보장한다.



`readv` 함수는 파일 디스크립터 fd에서 데이터를 읽어 count 개수만큼 iov 버퍼에 저장한다.

```c
#include <sys/uio.h>

ssize_t readv (int fd, const struct iovec *iov, int count);
```

일반적으로 우리가 사용하는 `read`함수와 동일하게 동작하지만 여러개의 버퍼를 사용한다는 점이 다르다고한다.

> 사실 리눅스 커널 내부의 모든 입출력은 벡터 입출력이다. read(), write() 구현 역시 하나짜리 세그먼트를 갖는 벡터 입출력이다.

이 부분이 제일 흥미로웠다. 운영체제는 볼때마다 새롭고 재미있다!





#### 1.2 왜 사용될까?

"왜 사용될까?" 라는 질문에는 당연히 **"시스템콜의 횟수를 줄여주기 때문이다."** 라고 답할 수 있다.

시스템콜을 호출하여 IO를 프로그램에서 요청하게 되면 컨텍스트 스위칭이 발생하고, 컨텍스트 스위치는 당연히 오버헤드를 만든다. 실제로 멀티 프로세스/스레드 프로그래밍을 하면서 프로파일링을 해보면 시스템콜로 인한 시간이 가장 점유율이 높은 것을 학교 과제에서 여러번 마주했다.

<figure><img src="../../.gitbook/assets/Screenshot 2024-12-02 at 2.59.09 PM.png" alt=""><figcaption><p>Gathering Write 과정</p></figcaption></figure>

그림과 같이 여러 버퍼에서 Gather하여 커널의 버퍼에 한번에 write한다. **시스템콜의 횟수가 3회에서 1회로 줄어들게 된다.**





### 2. Zero-Copy를 위한 Scatter / Gather

* 해당 내용에 대해 설명하기 전에 먼저 하나 알고갈 것이 있다. Scatter/Gather은 DMA 엔진이 해당 기능을 제공해야 사용할 수 있다.

**Zero-Copy**

단어 그대로 직역하면, "복사 0번"이다. 벌써 공간 효율적인 것 같아보인다.

지금까지 공부하거나 사용해본 소프트웨어 중 카프카에서 zero-copy이라는 개념을 처음 접했다.

일반적인 프로그램으로 IO작업 (네트워크, 파일접근)을 하게되면 유저 영역에서 커널영역으로 시스템콜을 호출하고, 커널영역은 Device에 요청을 하게 된다.

이 과정에서 여러 버퍼들이 생기게된다. 버퍼를 사용하는 이유는 이미 첫번째 글에서 설명했다. 효율적으로 데이터를 전송하고 저장하기 위해서인데, 이 버퍼링 과정 마저도 줄이기 위해 **유저 프로그램에 버퍼를 두지않고 커널의 버퍼, DMA 엔진을 이용하는 것이 제로카피 기술이다.**





#### 2.1 일반적인 데이터 전송

<figure><img src="../../.gitbook/assets/Screenshot 2024-12-02 at 3.29.19 PM.png" alt=""><figcaption><p>IBM: <a href="https://developer.ibm.com/articles/j-zerocopy/">https://developer.ibm.com/articles/j-zerocopy/</a></p></figcaption></figure>

그림과 같이 애플리케이션 영역에 버퍼로 copy가 일어나게된다. CPU가 이를 담당하게된다. 그림을 보면서 과정을 생각해보자.

1. 애플리케이션에서 read를 호출한다.
2. DMA 엔진에 데이터 접근을 호출하고 이를 커널의 버퍼에 복사한다.
3. 유저영역의 버퍼로 커널의 버퍼를 복사한다. (CPU)
4. 전송할 소켓의 버퍼로 또 유저 영역의 버퍼를 복사한다. (CPU)
5. DMA에 디바이스 버퍼로 복사를 요청한다.

**총 4번의 복사과정이 들어간다.**





#### 2.2 transferTo()를 사용한 데이터 전송

그런데 유저영역의 버퍼를 사용하지 않고 커널의 버퍼만을 사용하면 어떻게 될까?

<figure><img src="../../.gitbook/assets/Screenshot 2024-12-02 at 3.33.21 PM (1).png" alt=""><figcaption><p>IBM : <a href="https://developer.ibm.com/articles/j-zerocopy/">https://developer.ibm.com/articles/j-zerocopy/</a></p></figcaption></figure>

1. 애플리케이션에서 transferTo()를 호출한다.
2. DMA 엔진에 데이터 접근을 호출하고 이를 커널의 버퍼에 복사한다.
3. 전송할 소켓의 버퍼로 커널의 버퍼 내용을 복사한다.
4. DMA에 디바이스 버퍼로 복사를 요청한다.

**유저공간으로의 복사가 사라지고 복사횟수가 1회 줄어든다.**





#### 2.3 Zero-Copy를 달성한 데이터 전송

**갑자기 Zero-Copy는 왜...?**

Scatter / Gather은 여러 버퍼, 공간에 대해서 한번의 IO로 읽기 / 쓰기를 할 수 있도록 한다고 공부했다. 또 실제 메모리공간에 직접 접근하는 것도 배웠다. DMA가 SG(이하 Scatter/Gather)을 지원한다면 우리는 버퍼간의 CPU 복사가 아닌 NIC의 Gathering을 통해서 CPU 복사 없이 데이터를 전달할 수 있다!

<figure><img src="../../.gitbook/assets/Screenshot 2024-12-02 at 3.39.08 PM.png" alt=""><figcaption><p>IBM : <a href="https://developer.ibm.com/articles/j-zerocopy/">https://developer.ibm.com/articles/j-zerocopy/</a></p></figcaption></figure>

1. 애플리케이션에서 transferTo()를 호출한다.
2. DMA 엔진에 데이터 접근을 호출하고 이를 커널의 버퍼에 복사한다.
3. NIC가 DMA의 Gather을 사용하여 Read Buffer에 직접 접근한다.

**디스크에서 커널로 읽어오는 1번의 과정(DMA)를 제외하면 CPU복사는 전부 사라졌다.**

* SG가 가능해야 이 작업이 가능한 이유?

SG가 가능하다는 것은 DMA 엔진이 물리적으로 떨어져있는 공간에 대해서 합산하여 한번의 데이터 전송으로 처리가 가능하다는 것을 의미한다. **중요한 점은 직접 메모리공간의 데이터를 접근하는 것이다.**

**전송하려는 데이터는 연속적으로 저장되어 있을 수도, 아닐수도** 있기에 버퍼에 복사를 했었는데 이제 떨어져있어도 우리는 직접 접근을 할 수 있기에 복사가 필요없다. 커널의 메모리 공간에 대한 직접적인 주소를 전달하기 때문에 NIC 버퍼에게도 직접적인 주소를 전달할 수 있는 것이다.



***

**배운 것들**

* Scatter / Gather은 자바만의 기술이 아니라 **운영체제에서 사용되는 기술이다.**
* Scatter / Gather을 통해 **시스템콜의 횟수를 줄이고 효율적인 IO**를 수행할 수 있다.
* **Zero-Copy**를 달성하기 위해서 Scatter / Gather기술이 필요하다

**참고한 글들**

* https://www.linuxjournal.com/article/6345?page=0,1
* https://man7.org/linux/man-pages/man2/sendfile.2.html
* https://stackoverflow.com/questions/9770125/zero-copy-with-and-without-scatter-gather-operations
* https://en.wikipedia.org/wiki/Vectored\_I/O
* 리눅스 시스템 프로그래밍 - Oreilly
