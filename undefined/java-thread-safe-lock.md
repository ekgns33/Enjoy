---
description: 자바의 희노애 "락"
---

# Java - Thread Safe / lock

### Thread-safe



> 객체가 스레드에 안전해야 하느냐는 해당 객체에 여러 스레드가 접근할지의 여부에 달렸다.

객체가 뭘 하느냐의 문제보다는 **프로그램에서 객체가 어떻게 사용되느냐**의 문제이다. 여러 스레드가 하나 이상의 상태변수에 접근하고 WRITE/READ를 하면 해당 변수에 접근할때 모든 스레드가 동기화를 해야한다.

* 스레드의 실행 순서는 보장되지않는다. OS시간때 여러 방법을 배웠다! 무수한 race를 경험하고, 무지성 lock을 사용하던 작년의 내가 생각난다.

> 경쟁상태가 발생했을 때 고치는 3가지 방법
>
> * 해당 상태변수를 스레드간 공유하지 않는다.
> * 해당 상태변수를 변경할 수 없게 만든다.
> * 해당 상태변수에 접근할 때 항상 동기화를 사용한다.

일이 터지고 나서 고치려하면 설계를 뒤집어야하는 경우가 많기에 처음부터 Thread-safe한 설계가 중요하다.



자바 병렬 프로그래밍 책에서는 다음과 같이 서술한다.

> * 프로그램 상태를 잘 캡슐화할수록 프로그램을 스레드에 안전하게 만들기 쉽다.
>
>
>
> * 여러 스레드가 클래스에 접근할 때, 실행 환경이 해당 스레드들의 실행을 어떻게 스케줄하든, 어디에 끼워 넣든, 호출하는 쪽에서 추가적인 동기화나 다른 조율 없이도 정확하게 동작하면 해당 클래스는 스레드 세이프하다고 할 수 있다.

* 첫번째 문장은 잘 이해가 되지않았다. 그런데 여러 오픈소스들을 구경하다보니 뭔말인지 조금은 이해가 간다.&#x20;
  * 특정 메소드나 필드에 직접 접근하기 보다는 캡슐화를 통해 직접 접근은 막고 thread-safe하게 설계된 인터페이스를 제공함으로써 예상치 못한 경쟁상태를 예방할 수 있다는 것 같다.



***



### Java언어에서 락의 재진입성

* 스레드는 다른 스레드가 가진 락을 요청하면 블락된다.
* Java에서 암묵적인 락은 재진입 가능하기 때문에 특정 스레드가 자기가 이미 획득한 락을 다시 확보할 수 있다.

> 재진입성은 확보 요청 단위가 아닌 스레드 단위로 락을 얻는다는 것이다.



* 자바에서는 synchronized를 통해 락을 걸게되면 객체에 대해 락을 걸게된다.



여러가지 상황들에 대해서 JVM은 어떻게 하는지 살펴보자

* 락을 획득하면 확보 횟수와 확보한 스레드를 연결 시킨다.&#x20;
* 확보 횟수가 0 이면 락이 해제된 상태이다.&#x20;
* 스레드가 락을 확보하면 JVM이 락에 대한 소유 스레드를 기록하고 확보 횟수를 1로 지정한다.&#x20;
* 락을 확보한 스레드가 다시 요구하면 1을 증가시킨다.
* 블록을 빠져나오면서 확보횟수를 줄이다 0이되면 락이 해제된다.

```
synchronized function A() {}

synchronized(this) { // increase monitor lock value + 1
    //do something
    
    A(); // increase monitor lock value + 1
    
    // decrease monitor lock value -1
}
decrease monitor lock value -1 ; free
```

대충 위와같은 형태!



모든 자바 객체들은 Object를 상속하고 Object는 monitor을 갖는다. JVM에서는 synchronized 를 만나면 monitorenter operation을 수행하여 락을 획득하고 블럭에서 빠져나오면서 monitorend operation을 수행한다.

* 따라서 객체의 상속여부, 부모 클래스가 갖고있는 synchronized 블럭에 상관없이 동일한 락을 획득한다.
* **모든 객체는 암묵적으로 lock을 갖고있다라고 해석할 수 있다.**
* 메인 스레드의 스레드 덤프를 JConsole에서 확인해보면 아래와 같다.

<figure><img src="../.gitbook/assets/Screenshot 2024-09-17 at 4.36.38 PM.png" alt=""><figcaption><p>lock을 확인기</p></figcaption></figure>

락을 두번 걸게되는데 상속에 관계없이 동일한 객체에 거는 것을 알 수 있다.



**락의 재진입성이 뭣이 중헌디**

```
public class A {

  public synchronized void doSomething() {}
}

public class B extends A {
  public synchronized void doSomething() {
    super.doSomething();
  }
}
```

이 코드가 락의 **재진입성이 없었다면 데드락이 걸리는 코드**라고 책에서 서술한다.

왜 데드락이 걸리는지 알아보자.

어떤 스레드 T가 B의 doSomething()을 실행했다.

T는 Object B에 대해서 락을 얻고 함수 블록 내에 진입했다. 그런데 super.doSomething()을 마주한다. 이 함수도 synchronized 선언이 되있기에 락을 획득해야하는데, Object B에 대한 락을 이미 자신이 들고있기에 데드락이 발생한다.

**하지만 자바는 암묵적인 락에 대해서 재진입성을 보장하기에 동일한 락에 대해서는 그냥 진입할 수 있게된다!**



***

참고한 자료들

https://docs.oracle.com/javase/specs/jvms/se6/html/Threads.doc.html https://stackoverflow.com/questions/6470888/reentrant-lock-and-deadlock-with-java https://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.1
