---
description: JPA로 낙관락 사용해보기
---

# Optimistic Lock

**상품의 재고를 감소시키는 기능 구현**

* 데이터베이스에는 상품의 전체 재고 정보가 저장되어있다. 고객이 상품을 구매할때, 상품의 재고를 감소시켜야한다.

동시에 여러 고객들이 재고를 사용한다면 어떻게될까?? 테스트를 작성하여 확인해보자.

단순한 재고 값을 들고있는 엔티티를 먼저 생성한다.

```kotlin
@Entity
class Stock(
    var stock: Int = 100,

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
) {

    fun use() {
        this.stock--;
    }
}
```

```kotlin
@Test  
fun `재고 감소 동시테스트`() {  
    val threadCount = 10  
    val startLatch = CountDownLatch(threadCount)  
    val finishLatch = CountDownLatch(threadCount)  
    val executor = Executors.newFixedThreadPool(threadCount)  
  
    for (i in 1..threadCount) {  
        executor.execute {  
            startLatch.countDown()  
            startLatch.await()  
            try {  
                stockService.use(1L)  
            } finally {  
                finishLatch.countDown()  
            }  
        }  
    }      
    finishLatch.await(10, TimeUnit.SECONDS)  
    executor.shutdown()  
  
    val stock = stockRepository.findById(1L)  
    assertEquals(90, stock.get().stock)  
}
```

* 실행 결과&#x20;

<figure><img src="../../.gitbook/assets/Screenshot 2025-03-03 at 7.50.55 PM.png" alt=""><figcaption></figcaption></figure>



* 10번 감소해야하지만 2번밖에 감소하지 않았다. Race가 발생했다는 이야기.

**해결책 1 : synchronized**

* 모니터락을 걸어서 수정하는 임계영역에 하나의 스레드만이 접근할 수 있도록 바꿔보자.

```kotlin
@Transactional  
fun use(id: Long) {  
    synchronized(this) {  
        val stock = stockRepo.findById(id).get()  
        stock.use()  
    }  
}
```

* 코드가 제대로 작동할까?? !\[\[Screenshot 2025-03-03 at 7.53.47 PM.png]] 정답은 **아니요** 이다. 분명 모니터락을 걸었는데, 어째서 기존의 예외가 그대로 생기는 걸까? 답은 트랜잭션 어노테이션에 있다.

스프링에서는 기본적으로 트랜잭션을 위해 프록시를 사용한다. 즉, 실제 객체를 사용하는게 아니라 프록시객체를 활용하여 로직을 수행하는데, 트랜잭션의 경우 커밋시점이 메소드가 실행된 이후이다. 따라서 synchronized 블럭 내에서 트랜잭션 커밋이 이뤄지는 것이 아니라 밖에서 일어난다.

`트랜잭션 시작 -> synchronized -> logic -> synchronized -> 트랜잭션 커밋`

* 그럼 트랜잭션이 걸리는 부분 밖에서 synchronized를 걸어보자.

```kotlin
@Component  
class StockServiceFacade (  
    private val stockService: StockService,  
){  
  
    fun useSynchronized(id : Long) {  
        synchronized(this) {  
            stockService.use(id)  
        }  
    }  
}
```

서비스 파사드를 두어 밖에 레이어에서 모니터 락을 걸면 이제 해결된다.

```kotlin
@Test  
fun `재고 감소 동시테스트`() {  
    ...
  
    for (i in 1..threadCount) {  
        executor.execute {  
            startLatch.countDown()  
            startLatch.await()  
            try {  
                stockServiceFacade.useSynchronized(1L)  
            } finally {  
                finishLatch.countDown()  
            }  
        }  
    }  
    ...
    val stock = stockRepository.findById(1L)  
    assertEquals(90, stock.get().stock)  
}


```

하지만 이 방식은 너무 주먹구구식이다. 락을 기반으로 하기에 다른 스레드들은 기다려야하고, 프로세스단위의 락이기에 다중 서버에서는 제대로 동작하지 않는다.

<figure><img src="../../.gitbook/assets/Screenshot 2025-03-03 at 8.04.35 PM.png" alt=""><figcaption></figcaption></figure>

#### 2. Optimistic Lock

이번 테스트에서는 낙관적 락을 사용하는 방법에 대해서 공부하고 있기때문에 DB의 X-Lock, S-Lock은 다음에 다루려고 한다. 동시성 제어에는 여러 방법이 있는데, 서비스의 성격, 규모에 따라 적절한 전략을 선택해야한다고 알려져있다.

* **낙관적 락 (Optimistic Lock)은 데이터베이스의 락을 사용하기 보다는 별도의 버전 컬럼과 어플리케이션단의 로직으로 동시성을 제어하는 방법이다.**

엔티티에 버젼 컬럼을 추가한다. 한 트랜잭션이 시작될때마다 초기의 버전값을 기억하고 로직을 수행한다. 마지막 데이터베이스의 커밋시점에 버전값이 달라진다면, 낙관락 예외를 발생한다.

<figure><img src="../../.gitbook/assets/image (1) (1) (1).png" alt=""><figcaption><p><a href="https://systemdesignschool.io/blog/optimistic-locking">https://systemdesignschool.io/blog/optimistic-locking</a></p></figcaption></figure>



* 이게 무슨말?

각 트랜잭션은 커밋시점에 버전값을 1씩 올리게되는데, **자신이 커밋하기 직전 이미 다른 트랜잭션에서 커밋을 해 버전이 달라졌다면, 트랜잭션을 포기하는 것**이다. (버전은 항상 숫자일 필요는 없다 타임스탬프를 사용할 수도 있다.)

따라서 두 트랜잭션 중 **하나의 트랜잭션은 일관성을 보장한다**. 나머지 하나의 트랜잭션은 재시도 로직을 통해 갱신을 요청하거나 실패로 처리된다.

**재시도를 반드시 고려하자.**

* 낙관적 락은 구현하려는 서비스가 트랜잭션 경합이 빈번하게 발생하지 않는다는 가정하에 보통 사용한다. 일관성을 지키기 위해 재시도 로직을 구현해야하는데, 경합이 빈번하게 발생하면 재시도의 횟수도 증가한다.

100명의 유저가 동시에 하나의 리소스에 대해 경합한다면, 100명중 1명만이 성공하고 99명은 재시도를 통해 다시 업데이트 요청을 하게된다. DB 커넥션을 너무 오래잡아 데드락이 발생할 위험도 있으며, DB에 계속하여 요청을 하기때문에 부하가 생길 수 있다.

**어쨌든 적용해보자**

* JPA 엔티티에는 버전 컬럼을 추가해준다.

```kotlin
@Entity
class Stock(

    @Version
    var version: Long = 0,

    var stock: Int = 100,

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
) {

    fun use() {
        this.stock--;
    }
}
```

* 서비스 부분에는 Retry 어노테이션을 추가한다. 직접 구현할 수도 있고, 라이브러리를 사용할 수도 있다.

```kotlin
@Retryable(retryFor = [OptimisticLockingFailureException::class],
        backoff = Backoff(delay = 1000, multiplier = 2.0),
        maxAttempts = 10
    )
@Transactional
fun use(id: Long) {
    val stock = stockRepo.findById(id).get()
    stock.use()
}
```

* spring-retry 라이브러리를 사용하여 재시도 로직을 구현할 수 있다. https://www.baeldung.com/spring-retry
* 테스트도 통과한다. 하지만 **재시도 횟수나, 주기를 바꾸면 테스트를 통과하지 못할 수도 있다.**
* **또한 재시도를 계속 실행하기 때문에 전체가 수행될때까지 시간은 더 오래 걸릴 수 있다.**&#x20;
* 반드시 요청이 성공할때까지 재시도를 하는 것이 아니라 최대 몇회까지 재시도를 하는 로직이기에 유저는 에러 메세지를 수신할 수 있다.

랜덤 테스트케이스에 대해 통과하는 테스트이기때문에 이를 숙지해야한다 :)

**배운것**

* JPA에서 낙관적 락을 적용하는 방법
* 낙관락을 사용할 때는 서비스의 특성을 고려한다.
* 재시도 로직을 꼭 확인하자.
* 커넥션이 모두 낙관락 재시도에 사용되면 데드락이 발생할 수도 있다.

**참고한 자료**

{% embed url="https://www.baeldung.com/spring-retry" %}

{% embed url="https://www.blog.ecsimsw.com/entry/DB-%EB%9D%BD%EC%9D%98-%EC%BB%A4%EB%84%A5%EC%85%98-%EC%A0%90%EC%9C%A0-%ED%99%95%EC%9D%B8%EA%B3%BC-%ED%95%B4%EA%B2%B0-%EC%A0%90%EC%9C%A0-%EC%8B%9C%EA%B0%84%EC%9D%84-%EC%A4%84%EC%9D%B4%EA%B8%B0-%EC%9C%84%ED%95%9C-%EB%85%B8%EB%A0%A5%EB%93%A4" %}
