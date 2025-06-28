---
description: 자바의 스레드풀, Executor 프레임웤
---

# Java - Thread Pool, Executor

## 1. Executor

자바에서는 Executor라는 프레임쿼크를 제공하여, 스레딩이나 비동기 처리에 대한 추상화를 제공한다. Executor은 작업 등록 / 작업 실행을 분리하는 표준적인 방법이며, 각 작업은 `Runnable` 로 정의한다. Executor의 구조는 프로듀서 - 컨슈머 패턴에 기반한다. 작업을 실행하는 스레드가 프로듀서, 처리하는 스레드가 컨슈머가 된다.

## 2. ThreadPool of Executor

작업을 처리하는 동일한 형태의 스레드들을 풀의 형태로 관리한다. 스레드 풀은 내부에 처리할 작업들에 대한 큐를 두는 것이 일반적이다. 작업 큐에서 실행할 다음 작업을 가져오고, 작업을 실행한 후 대기한다.

스레드풀을 사용하면 매번 스레드를 생성하는 것보다 자원을 아낄 수 있다. 스레드를 처리할 **작업마다 생성하는게 아니라 기존에 대기중인 스레드를 이용**하기에 딜레이가 발생하지 않는다.



* `newFixedThreadPool` : 처리할 작업이 등록되면 스레드를 하나씩 생성한다. 최대 개수는 제한되어있다.
* `newCachedThreadPool` : 풀에 존재하는 스레드가 작업의 개수보다 많을때 대기하는 스레드를 종료한다. 스레드 수에는 제한을 두지 않는다.
* `newSingleThreadExecutor` : 싱글 스레드로 동작하며 작업을 하나씩만 처리한다. 작업큐에서 지정하는 순서에 따라 순차적으로 처리된다. FIFO, LIFO, PRIORITY등을 설정할 수 있다.
* `newScheduledThreadPool` : 일정 시간 / 주기적으로 작업을 실행할 수 있다.



```java
ExecutorService executorService = 
  new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS,   
  new LinkedBlockingQueue<Runnable>());
  
Runnable runnableTask = () -> {
    try {
        TimeUnit.MILLISECONDS.sleep(300);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
};

Callable<String> callableTask = () -> {
    TimeUnit.MILLISECONDS.sleep(300);
    return "Task's execution";
};

List<Callable<String>> callableTasks = new ArrayList<>();
callableTasks.add(callableTask);
callableTasks.add(callableTask);
callableTasks.add(callableTask);

List<Future<String>> futures = executorService.invokeAll(callableTasks);

```



**JVM은 모든 스레드가 종료되기 전에는 종료하지 않고 대기한다. 따라서 Executor을 제대로 종료시켜야한다.** Executor은 비동기적으로 실행되기에 작업의 상태를 특정시점에 파악하기 어렵다. 대기큐에 있을 수도 있고 이미 처리됐을 수도 있다.



* `shutdownNow` : 강제 종료 절차를 밟게됨
* `shutdown` : Graceful shutdown으로 이어진다.



## **3. 지연 작업 / 주기적 작업**

Java의 **Timer** 클래스를 사용하면 특정 시간 이후에 원하는 작업을 실행할 수 있다.

* Timer은 상대시각/절대 시각을 지원한다. 따라서 **절대 시각을 사용하면 하드웨어의 시간에 따라 작업이 변경될 수 있다.**
* **ScheduledThreadPoolExecutor은 상대시각만 지원한다.**

### Timer의 문제점

<mark style="color:red;">**Timer 클래스는 등록된 작업을 실행시키는 스레드를 하나만 생성해 사용한다**</mark><mark style="color:red;">.</mark> 특정 작업이 너무 오래 수행되면 대기중인 다른 작업이 지연될 수 있다. but `ScheduledThreadPoolExecutor`은 **지연 작업과 주기적 작업마다 스레드를 할당**해 작업을 실행하기에 어긋나는 일이 없게끔한다.

<mark style="color:red;">**Timer의 작업이 실행되다가 예외를 발생시켜도 Timer 스레드는 예외를 처리하지 않는다.**</mark> 따라서 스레드 자체가 멈출 수 있다. 등록한 다른 작업들도 수행되지 않을 수 있다.

## 4. Callable, Future

스레드풀에 작업을 넘길때 `Runnable`을 사용한 경우 리턴값을 받을 수도 없고 예외에 대한 처리를 명시할 수도 없다. 따라서 전역적인 상태 변화를 처리하려면 공유 저장소를 사용해야한다.

**결과를 얻는 것을 기다리는 경우에는 `Callable`을 사용하는게 좋다**. `Callable` 에서는 실행의 결과 값을 반환받을 수 있고 예외도 던질 수 있다.

`Future`은 특정 작업이 정상적으로 완료되었는지, 아니면 취소됐는지 등에 대한 정보를 확인하도록 만들어진 클래스이다. `Future` 의 동작사이클에서 주의할 점은 한 번 지나간 상태를 되돌릴 수 없다는 것이다. 완료된 작업은 완료 상태에서 끝난다.

`get()` 을 통해서 Future의 결과에 대기하거나 바로 값을 반환 받을 수 있다. 예외가 발생했다면 `ExecutionException` 에 예외를 래핑해서 넘겨준다. 작업이 취소되면 `CancellationException` 이 발생한다.

## 5. 작업을 쪼개야하는가?

* IO를 기다리는 작업과 computation을 수행하는 작업 두개를 병렬처리한다고 생각해보자.

<mark style="color:red;">**computation에 드는 시간이 매우 짧고 IO를 기다리는시간이 수십배 더 길다면 병렬처리로 얻을 수 있는 이점이 크게 없다.**</mark> 오히려 코드의 복잡성만 늘리게 될 수가 있다.

**애플리케이션의 작업을 쪼개서 병렬로 처리한다면, 작업의 단위를 잘 정해야 이득을 볼 수 있다.**



***

#### 참고한 자료

* 자바 병렬 프로그래밍 - 브라이언 게츠
* [https://www.baeldung.com/java-executor-service-tutorial](https://www.baeldung.com/java-executor-service-tutorial)
* 모던 자바 인 액션
