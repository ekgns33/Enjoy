---
description: JDBC? 드라이버? 어떻게
---

# JDBC - Driver 로드, Java SPI

{% stepper %}
{% step %}
### JDBC란?
{% endstep %}

{% step %}
### JDBC 아키텍처

드라이버 등록하는 법
{% endstep %}

{% step %}
### Java SPI (Service Provider Interface)

JDBC + MySQL로 확인하기
{% endstep %}
{% endstepper %}

## JDBC (Java Database Connectivity)

* 데이터베이스 종류에 무관하게 java로 데이터베이스 작업을 수행할 수 있도록 해주는 API이다.
* `java.sql` 패키지에서 확인할 수 있다.



JDBC를 활용하여 일반적인 데이터베이스 기능들을 활용할 수 있다.

* 데이터베이스와 커넥션을 맺기
* SQL문 질의
* SQL문 생성
* 실행결과 조회 및 수정

Spring 프레임워크에서는 JDBC를 사용하기 편하게하도록 JdbcTemplate을 지원하기도한다.

***

## JDBC 아키텍처

JDBC는 크게 2가지로 나뉘게된다.

<figure><img src="../../.gitbook/assets/image.png" alt=""><figcaption><p><a href="https://www.geeksforgeeks.org/introduction-to-jdbc/">https://www.geeksforgeeks.org/introduction-to-jdbc/</a></p></figcaption></figure>



1. JDBC API : 자바에서 JDBC로 향하는 인터페이스
2. JDBC Driver API : 실제 데이터베이스 구현에 대한 드라이버

데이터베이스 사용에 대한 추상화를 제공한다고 생각하면 편하다. 우리는 데이터베이스 드라이버 설정만 마치면, api를 사용하여 여러가지 SQL을 수행할 수 있다.

**JDBC API를 사용하면 Driver Manager가 데이터베이스 구현체와 연결되는 드라이버를 사용한다.**

```java

public static void main(String[] args) throws ClassNotFoundException, SQLException {  
    Class.forName("com.mysql.cj.jdbc.Driver");  
  
    Connection connection = DriverManager.getConnection(  
        "jdbc:mysql://localhost:3306/mydatabase", "username", "password");  
}

```

위와 같이 Driver을 로드하고, DriverManager을 사용하여 커넥션을 수립할 수 있다. TCP 커넥션을 맺고나면 Stream 또는 채널을 통해 Read/Write를 수행하는 것 처럼 데이터베이스 또한 커넥션을 활용하여 여러 작업을 수행할 수 있다.

***

**드라이버를 등록하는 법**

1. `Class.forName()`

예시코드에서 사용한 방법이다. 자바는 클래스를 동적으로 로드한다는 점은 다들 알고있을 것이다. 메모리에 드라이버를 로드하기만하면 자동으로 등록된다.

{% hint style="info" %}
이는 JDBC 4.0부터 Service Provider Interface 매커니즘이 애플리케이션 클래스 패스의 모든 드라이버를 자동으로 등록할 수 있게된 이후 부터 가능해졌다.
{% endhint %}



2. `DriverManager.registerDriver()`

```java

public static void main(String[] args) throws ...{
	Driver driver = new com.mysql.cj.jdbc.Driver();
	DriverManager.registerDriver(driver);
} 

```

드라이버 매니저를 직접 사용하는 방법이다. JDK가 없는 상황에서는 이 방법을 사용해야한다.

***

## **Java SPI (Service Provider Interface)**



* **실제 구현체는 java.util의 Service Loader가 수행한다.**
* **java 6**부터 도입되었고, 드라이버의 JAR 파일 특징 설정을 기반으로 작동한다.

SPI에는 4가지 구성요소가 있다.

1. **서비스 인터페이스**
2. **서비스 제공자**
3. **서비스 로더**
4. **설정파일 (META-INF/services/서비스인터페이스의 FQCN)**

### **작동하는 과정**

1. classpath의 모든 JAR에서 META-INF/services에 존재하는 모든 설정파일을 탐색한다.
2. 파일에 적힌 클래스를 로드한다.
3. 인스턴스를 생성한다.
4. 인스턴스는 ServiceLoader에 캐시되며 재사용된다.

### **JDBC로 확인하기**

* JDBC를 예로 확인해보면 서비스 인터페이스는 `java.sql.Driver` 인터페이스이다.
* 서비스 제공자는 우리가 gradle을 통해 import한 mysql 드라이버이다.
* 서비스로더는 `java.util`의 ServiceLoader 클래스이다.
* 마지막으로 설정파일은 mysql드라이버의 메타정보 패키지에 있는 파일이다.

Service Loader 클래스는 `META-INF/services` 디렉토리의 설정파일을 읽어 구현체를 동적으로 로드한다.



<figure><img src="../../.gitbook/assets/Screenshot 2025-04-25 at 1.05.05 AM.png" alt=""><figcaption><p>mysql</p></figcaption></figure>

\
해당 파일을 열어보면 `com.mysql.cj.jdbc.Driver` 이라고 적혀있는데 이게 드라이버 클래스의 이름이다.

<figure><img src="../../.gitbook/assets/Screenshot 2025-04-25 at 1.10.12 AM.png" alt=""><figcaption><p>com.mysql.cj.jdbc.Driver.class</p></figcaption></figure>

JVM이 시작될때 DriverManager은 ServiceLoader을 사용해 모든 JDBC 드라이버를 검색한다.



* 해당 사진은 mysql의 드라이버 클래스이다.

**검색된 클래스는 static 블록에서 인스턴스화되며 DriverManager에 자기 자신을 등록한다.**

이처럼 사용자가 애플리케이션에 드라이버를 등록하는 코드를 작성하지 않아도 JVM 실행과정에서 자동으로 로드해주는 기능이 SPI이다.



***



### **최근에 사용한 SPI기능**

내가 개발중인 프로젝트는 Spring Boot 기반인데, sql 쿼리 로깅을 위해 P6Spy를 사용중이다. 이를 위해 resources/META-INF에 별도의 파일을 정의하는데 스프링 자동구성 방식에 따라 `DataSourceDecoratorAutoConfiguration` 클래스를 로드하고, 부트될때 DataSource를 P6Spy로 감싸는 데코레이터를 스프링 빈으로 등록한다.



***



**배운점**&#x20;

* JDBC 아키텍처는 일반적으로 2티어로본다.
* Driver로드 과정
* **Java SPI - 스프링같은 프레임워크 코드를 볼때마다 spi 패키지가 뭔가 했는데 이제 이해했다.**



참고한 자료

* [https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html)
* [https://download.java.net/java/early\_access/genzgc/docs/api/java.sql/java/sql/DriverManager.html](https://download.java.net/java/early_access/genzgc/docs/api/java.sql/java/sql/DriverManager.html)

