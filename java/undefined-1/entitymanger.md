---
description: Spring Boot에서 EntityManager은 어떻게 만들어질까?
---

# EntityManger는 어떻게 만들어질까

JPA를 사용할때, EntityManger을 `@PersistenceContext` 로 주입받아서 사용할 수 있다.

해당 어노테이션은 `EntityManagerFactory`에서 `EntityManager`을 만들어서 제공한다.



{% stepper %}
{% step %}
### Spring 공식문서 보고 EntityManagerFactory에 대해 정리

[#spring](entitymanger.md#spring "mention")

[#id-1.-localentitymangerfactorybean](entitymanger.md#id-1.-localentitymangerfactorybean "mention")

[#id-2.-entitymanager-through-jndi](entitymanger.md#id-2.-entitymanager-through-jndi "mention")

[#id-3.-localcontainerentitymanagerfactorybean](entitymanger.md#id-3.-localcontainerentitymanagerfactorybean "mention")
{% endstep %}

{% step %}
### Spring Boot에서 EntityManagerFactory가 빈으로 등록되는 과정 추적하기

[#entitymanagerfactory](entitymanger.md#entitymanagerfactory "mention")
{% endstep %}

{% step %}
### EntityManger를 주입받는 방법, 과정 살펴보기

[#entitymanager-di](entitymanger.md#entitymanager-di "mention")

[#entitymanager](entitymanger.md#entitymanager "mention")

[#undefined](entitymanger.md#undefined "mention")
{% endstep %}
{% endstepper %}

***

### Spring 공식문서 보고 EntityManagerFactory에 대해 정리

### Spring 공식문서 찾아보기

{% embed url="https://docs.spring.io/spring-framework/reference/data-access/orm/jpa.html" %}

* Spring Framework 공식문서의 Data Acess 파트를 살펴보면 JPA에 대한 자세한 내용을 읽을 수 있다.

Spring 프레임워크에서는 EntityManager을 위한 3가지 방법을 제시한다.



#### 1. LocalEntityMangerFactoryBean

* 이 옵션은 **Stand alone 애플리케이션이나 통합테스트같은 배포환경에서만 사용하라고 한다.**
* 애플리케이션이 오직 JPA Data Access만 사용하는 경우에 적합하다.
* JPA의 Java SE 부트스트래핑 메커니즘을 사용하여 `EntityManagerFactory`를 생성한다.
* 보통 PersistenceUnitName만 지정하면된다고 한다.



간단하지만 제한사항이 많은 옵션이다. **해당 옵션에서는 빈으로 등록된 `DataSource` 를 사용할 수 없고, `global transaction` 이 지원되지 않는다.**

* 여러 DataSource에 대한 지원이 불가능하므로 단일 데이터베이스 트랜잭션만 지원한다. Datasource또한 스프링 빈 컨테이너에 등록된것은 사용할 수 없기때문에 직접 연결해줘야한다.



또한 persistent class 에 대한 weaving이 제공자에 따라 달라진다고한다. 그렇기에 JVM 에이전트를 지정해야할 수도 있다고 적혀있다.

* **JPA에서는 프록시를 사용하여 Lazy하게 로딩을 하거나 Dirty check를 수행한다. 이는 바이트코드 조작을 활용하는데, Java SE가 기본적으로 지원하지 않으니 에이전트를 직접 설정해야한다는 것이다.**



#### 2. EntityManager through JNDI

* Jakarta EE Server을 배포할 때 이 옵션을 활용할 수 있다. 스프링 부트가 아니라 톰캣, JBoss같은 친구

JNDI로 EntityManager을 획득하는 경우에는 xml 설정을 해줘야한다.

**JNDI는 (Java Naming and Directory Interface)를 의미한다.**

* 서버 컨테이너에서 미리 등록한 리소스를 이름으로 찾아서 사용하는 방식이다.
* 서버에서 **JTA 트랜잭션**, **DataSource 풀링**, **EntityManagerFactory 관리**까지 알아서 해주는 경우에 사용한다고 한다.



#### 3. LocalContainerEntityManagerFactoryBean

* 이번 글에서 디버깅하면서 살펴봤던 옵션이다.

톰캣, standalone,... 모든 경우에서 JPA를 full로 사용할 수 있다!

**아마도 스프링 부트를 사용하면 해당 옵션을 사용하지 않을까?**



이 옵션에서는 `EntityManagerFactory` 에 대한 모든 설정을 허용한다.

* Datasource 스프링빈컨테이너에서 사용가능
*   `loadTimeWeaver` 로 바이트코드 위빙 설정가능

    * **하이버네이트에서는 필요없음.**



스프링 부트에서는 가장 기능도 다양하고 유연한 3번째 `LocalContainerEntityManagerFactoryBean` 을 사용한다.

***

### Spring Boot에서 EntityManagerFactory가 빈으로 등록되는 과정 추적하기

### EntityManagerFactory를 생성

* EntityManager의 경우 팩토리를 빈으로 두고 객체를 생성을 위임하는 방식으로 작동된다.
* 이때 `EntityManagerFactory` 는 애플리케이션 로드과정에서 1번 초기화된다.
* 디버깅 모드에서 `org.springframework:spring-orm.jpaLocalContainerEntityManagerFactoryBean.class` 의`afterPropertiesSet()`부터 살펴본다.
* EntityMangerFactory를 만들기 위해서 사전작업들이 있다. **PersistenceUnit이 먼저 필요하다.**
* PersistenceUnit에는 DataSource와 같은 Persistence와 관련된 설정 정보들이 포함된다.



**1. PersistenceUnit을 먼저 만든다.**

* spring의 boot과정에는 package를 스캔하는 과정이 있는데 그 과정에선 `@Entity` 어노테이션이 표시된 클래스에 대해서 미리 모아두는 것 같다. 찾아보자!

> Traditionally, JPA ‘Entity’ classes are specified in a `persistence.xml` file. With Spring Boot this file is not necessary and instead ‘Entity Scanning’ is used. By default all packages below your main configuration class (the one annotated with `@EnableAutoConfiguration` or `@SpringBootApplication`) will be searched.<br>
>
> https://docs.spring.io/spring-boot/docs/1.2.1.RELEASE/reference/html/boot-features-sql.html#boot-features-jpa-and-spring-data

패키지를 실제로 스캔하는 것은 `JpaBaseConfiguration`에서 정의된다.

PersistenceUnitManager에서는 저 값을 활용하여 PersistenceUnit을 만들어서 넘긴다!



**2. 스프링 빈으로 `LocalContainerEntityManagerFactoryBean` 을 만든다.**

`JpaBaseConfigurataion` 에는 `LocalContainerEntityManagerFactoryBean` 을 등록하는 메소드도 정의되어있다.

<figure><img src="../../.gitbook/assets/Screenshot 2025-04-18 at 2.47.00 PM.png" alt=""><figcaption><p>JpaBaseConfiguration</p></figcaption></figure>

이제 빈을 등록하기 위해서 `EntityManagerFactoryBuilder`을 호출하고 세팅을하게된다.

* PersistenceUnitManager
* PersistenceUnitName
* JpaVendorAdapter
* DataSource
* ManagedTypes, PackageScan
* Jpa Properties
* PostProcessor

일련의 과정으로 수행된다.

`afterPropertiesSet()` 메소드에서는 모든 세팅이 끝나고 JpaVendor에 대한 팩토리를 생성한다. Hibernate 부팅이 시작되는 것이다. 스프링이 관리했던 정보를 `createNativeEntityManagerFactory` 를 통해 전달한다.

> \`afterPropertiesSet() 메소드는 스프링 컨테이너가 해당 빈의 모든 의존성 주입을 마친 후 자동으로 호출하는 콜백이다!!

스프링 빈의 생성을 위한 모든 의존성주입을 마치고 콜백에서 실제 생성시점이 되는것이다.



**3. Hibernate EntityManagerFactory를 만든다.**

<figure><img src="../../.gitbook/assets/Screenshot 2025-04-18 at 1.51.58 PM.png" alt=""><figcaption></figcaption></figure>

* EntityManagerFactory를 만드는데 우리는 JPA의 구현체로 Hibernate를 사용하기 때문에 `SpringHibernateJpaPersistenceProvider`에서 Hibernate 팩토리 빌더를 통해서 제공받는다.

<br>

<figure><img src="../../.gitbook/assets/Screenshot 2025-04-18 at 2.02.14 PM.png" alt=""><figcaption></figcaption></figure>

* `EntityManagerFactoryBuilderImpl` 은 구상클래스로 실제 빌더를 위한 구현이 들어가있다.



<figure><img src="../../.gitbook/assets/Screenshot 2025-04-18 at 2.04.12 PM.png" alt=""><figcaption></figcaption></figure>

빌더의 생성자 부분으로 들어가 스프링으로부터 넘어온 **PersistenceUnit**에 어떤 값들이 들어있는지 확인해볼 수 있다. 현재는 예제로 만들었던 `Stock`클래스와 패키지 정보가 들어가있다.

***

## EntityManger를 주입받는 방법, 과정 살펴보기

### EntityManager을 DI 받기

엔티티매니저팩토리를 만들었으니, 엔티티 매니저를 생성받아서 사용하면 될 것이다!

* 스프링 부트에서는 `EntityManagerFactory` 가 스프링 빈 컨테이너에 등록되고 관리된다. 따라서 개발자가 이상한 행동을 하지 않는 이상 프로세스내에 하나의 인스턴스가 존재한다.
* 하지만 EntityManager의 경우 어떨까? 테스트에서 EntityManager에 어떤 값이 들어가는지부터 확인해보자.



<figure><img src="../../.gitbook/assets/Screenshot 2025-04-18 at 5.22.44 PM.png" alt=""><figcaption></figcaption></figure>

**Shared EntityManager Proxy가 주입되었다는 것을 확인할 수 있다.**

* Shared EntityManager?

엔티티매니저가 주입되어야할때는 우선 프록시가 주입된다.`PersistenceAnnotationBeanPostProcessor` 에서 어노테이션이 붙은 필드에 대해 주입을 하는데, `SharedEntityManagerCreator` 클래스에서 메소드로 생성한다.



<figure><img src="../../.gitbook/assets/Screenshot 2025-04-18 at 5.28.12 PM.png" alt=""><figcaption></figcaption></figure>

하지만 **실제 객체가 아니라 ProxyInstance로** 주입한다.

<figure><img src="../../.gitbook/assets/Screenshot 2025-04-18 at 5.32.26 PM.png" alt=""><figcaption></figcaption></figure>

트랜잭션을 시작하고 엔티티하나를 저장하는 테스트를 실행해봤는데 다음과 같은 <mark style="color:red;">오류가 발생했다.</mark>

`SharedEntityManager`에서는 트랜잭션을 생성할 수 없다는 에러다. 이게 뭘까?

**스프링에서는 EntityManager을 트랜잭션에 대응되도록 주입한다.**

트랜잭션이 시작되면 `TransactionSynchronizationManager` 에서 실제 `EntityManager`을 찾아서 주입해주는 방식이다. 위에서 확인한 <mark style="color:red;">**프록시 객체 자체에게는 트랜잭션을 시작하는 것을 허용하지 않는다!**</mark>

{% hint style="info" %}
**프록시 객체를 활용하고 싶으면 스프링에서 제공하는 `@Transactional` 을 사용하여 트랜잭션을 시작한다.** 이는 자동으로 `em.getTranscation.begin()` 을 실행하고 EntityManager로 직접 조작할 수 있게된다.
{% endhint %}

***

#### 직접 EntityManager 만들어서 사용하는 경우



**그래도 나는 직접 모든 트랜잭션을 관리하고 싶다?**

이때는 `EntityManagerFactory` 를 주입받아 직접 객체를 받아서 사용한다.

```kotlin
@Autowired
lateinit var emf : EntityManagerFactory

@Test
fun test_transaction() {
    var entityManager = emf.createEntityManager()
    try {
        entityManager.transaction.begin();
        val stock = Stock(
            stock = 100,
        );
        entityManager.persist(stock);
        entityManager.flush();
    } catch (e: Exception) {
        println("Error starting transaction: ${e.message}")
        entityManager.transaction.rollback()
    }
}

```

처럼 직접 만들어서 사용하면 에러가 해결된다.<br>

<figure><img src="../../.gitbook/assets/Screenshot 2025-04-18 at 5.48.08 PM.png" alt=""><figcaption></figcaption></figure>

**또한 Hibernate의 세션 구현체가 주입된다!** Spring에서는 EntityManager로 추상화되었지만 _실제 트랜잭션의 Manager인 하이버네이트의 세션으로 전환된다._

***

#### 스프링 트랜잭션 위에서 프록시를 사용하는 경우

<figure><img src="../../.gitbook/assets/Screenshot 2025-04-18 at 6.03.05 PM.png" alt=""><figcaption></figcaption></figure>

`SharedEntityManagerCreator` 에서 스레드-세이프한 프록시를 만들어서 주입한다.<br>

* `SharedEntityManagerInvocationHandler`에서 호출되면 현재 트랜잭션의 엔티티 매니저를 조회한다.

<figure><img src="../../.gitbook/assets/Screenshot 2025-04-18 at 6.04.00 PM.png" alt=""><figcaption></figcaption></figure>

`EntityManagerFactoryUtils`에서 `TransactionSynchronizationManager`를 사용하여 **스레드 세이프하게 엔티티매니저를 가져온다.**

<figure><img src="../../.gitbook/assets/Screenshot 2025-04-18 at 6.05.59 PM.png" alt=""><figcaption></figcaption></figure>

`TransactionSynchronizationManager` 에서 실제 엔티티 매니저를 찾는데, 스프링은 각 트랜잭션마다 **ThreadLocal에 진짜 EntityManager을 넣어놨다가 사용한다.**

스프링은 Hibernate구현체인 EntityManagerImpl을 꺼내고 언래핑하여 하이버네이트 Session을 사용한다!

***

#### 배운점 :clap:

* 스프링 부트가 실행될때 엔티티매니저팩토리가 어떤 단계를 거쳐서 생성되는지 확인했다.
* EntityManagerFactory에도 3가지 옵션이 존재한다는 것을 알게되었다.
* Spring 코드는 추상화가 정말 잘되어있는 것 같다. 어느 순간 Hibernate구현체로 바뀌는데 디버깅하면서 신기했다.

#### 더 공부할것 :thinking:

* EntityManager를 프록시방식으로 사용할때, 스프링 트랜잭션 위에서 시작해야하는데 스프링 Transaction에 대해 공부해야겠다. 다행히도 공식문서에 엄청난 분량으로 정리돼있다.
* Spring Bean Container와 스프링 빈쪽을 다시 봐야할 것 같다. 특히 afterPropeties 콜백을 정리해보자.





