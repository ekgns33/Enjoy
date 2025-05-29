---
description: 동기화를 위한 일회성 클래스 Latch
---

# Java - Latch

### CountDownLatch

Latch는 스스로 터미널 상태에 이를 때까지의 스레드가 동작하는 과정을 늦출 수 있도록하는 동기화 클래스이다.

* 흐름 제어를 위한 카운터라고 생각하면 편하다.

> Simply put, a _CountDownLatch_ has a _counter_ field, which you can decrement as we require. We can then use it to block a calling thread until it’s been counted down to zero.  [https://www.baeldung.com/java-countdown-latch](https://www.baeldung.com/java-countdown-latch)

***

### 언제 쓰지?

테스트코드를 작성할때 동시성이슈를 검증하기 위해 **여러 스레드**로 요청을 날릴때가 있음.

* 작업을 수행하고 N초 뒤에 완료되었는지
* K번의 작업 수행을 마쳤는지
* **병렬처리에서 흐름을 제어**

등등 여러가지 로직을 동기화 클래스인 Latch를 사용하여 해결할 수 있다.



래치가 한 번 터미널 상태에 다다르면 <mark style="color:red;">**그 상태를 다시 이전으로 되돌릴 수는 없다.**</mark> 한 번 열린 관문은 계속 열린상태로 유지됨. 따라서 **단일 동작이 완료되기 전까지 막아야하는 로직**을 작성할 때 유효하다.



* 특정 자원을 확보하기 전에는 작업을 시작하지 말아야 하는 경우 (바이너리 래치)
* 의존성을 가진 서비스가 시작하고나서 실행하는 경우
* 특정 작업에 필요한 모든 객체가 실행할 준비를 기다리는 경우



***

### 테스트 코드로 학습

* 여러 스레드에서 특정 작업을 수행하는데 **동시에 수행할때** 얼마나 걸리는지 측정.



```java
int nThreads = 10;  
  
Runnable task = () -> {  
    System.out.println(Thread.currentThread().getName());  
};  
final CountDownLatch startGate = new CountDownLatch(1);  
final CountDownLatch endGate = new CountDownLatch(nThreads);

```

Latch를 양의 정수를 사용하여 초기화하는데 이 정수가 발생해야하는 이벤트의 건 수임. 즉 메인 흐름에서 막혀서 N개의 이벤트가 끝날때까지 대기함.



스레드 이름을 출력하는 Runnable를 하나 만들고 Latch를 두개 만들었다.\
첫번째 startGate는 1개의 countDown이 발생하면 흐름이 시작된다. 두번째는 nThread개의 countDown을 기다린다.

```java

for(int i = 0; i < nThreads; i++) {  
    Thread thread = new Thread() {  
        public void run() {  
            try {  
                startGate.await();  
                try {  
                    task.run();  
                } finally{  
                    endGate.countDown();  
                }  
            } catch (InterruptedException e) {  
                throw new RuntimeException(e);  
            }  
        }  
    };  
    thread.start();  
}
```

10개의 스레드를 시작하는데 각각의 스레드는 시작하자마자 멈추게된다.

* **startGate에 대해 await를 하고 있는데 아직 카운트다운이 발생하지 않았기 때문.**

```java
long start = System.currentTimeMillis();  
startGate.countDown();  
try {  
    endGate.await();  
} catch (InterruptedException e) {  
    throw new RuntimeException(e);  
}  
long end = System.currentTimeMillis();  
System.out.println("Time taken: " + (end - start) + " ms");
```

모든 스레드를 실행하고 countDown을 한다. 그러면 각 스레드들이 작업을 수행하고 카운트 다운을 한다. 10개의 스레드가 모두 작업을 마치면 `endGate` 가 통과되면서 프로세스가 종료된다.

간단한 코드에서 원했던 사항은 **여러개의 스레드가 동시에 시작됐을때 걸리는 시간을 파악**하는 것이다. for문으로 돌면서 각각 실행하는게 아니라, latch를 다운하는 순간 await 이후의 코드 블럭이 시작된다.



전체 코드

```java
 @Test
    void testLatch() {


        int nThreads = 10;

        Runnable task = () -> {
            System.out.println(Thread.currentThread().getName());
        };
        final CountDownLatch startGate = new CountDownLatch(1);
        final CountDownLatch endGate = new CountDownLatch(nThreads);

        for(int i = 0; i < nThreads; i++) {
            Thread thread = new Thread() {
                public void run() {
                    try {
                        startGate.await();
                        try {
                            task.run();
                        } finally{
                            endGate.countDown();
                        }
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }
                }
            };
            thread.start();
        }

        long start = System.currentTimeMillis();
        startGate.countDown();
        try {
            endGate.await();
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        long end = System.currentTimeMillis();
        System.out.println("Time taken: " + (end - start) + " ms");
    }

```



배운점

* Spring Kafka쪽에서 어떤 PR이 올라오는지 보는데 비동기 로직, 병렬 로직에 대한 테스트 코드를 작성할때 Latch, Future 같은 것들이 많이 사용됐다.
* Latch는 일회성이기 때문에 테스트 코드에서 특정 상황을 만들기 위한 카운터변수로 활용하기 좋을 것 같다.

참고한 자료

{% embed url="https://www.baeldung.com/java-countdown-latch" %}

{% embed url="https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/CountDownLatch.html" %}

* 자바 병렬 프로그래밍 - 브라이언 게츠&#x20;
