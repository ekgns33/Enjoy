# MDC를 활용해 로그 남기기

#### Spring + Slf4j로 로그를 남기기

Spring Boot를 사용할때, Slf4j를 사용해서 로그를 남기는 것을 주로 사용했다. 어노테이션을 추가하고 `log.info(...)` 처럼 사용하면 원하는 로깅 레벨에 따라서 로그를 찍는다.

로컬에서 개발할때는 sysout으로 로그를 모두 출력하는데, 운영환경에 배포할때는 logback을 사용해서 특정 path에 로그를 저장하고, 일정 크기보다 커지면 삭제하는 작업도 수행할 수 있다.

* **logback**

Logback은 Slf4j의 구현체로 최신 스프링 부트를 사용하면 기본적으로 내장되어있다. JPA의 구현체가 Hibernate인 것 처럼 Slf4J 스펙을 구현하고 있다. 프로젝트에서 사용할때는 `logback.xml` 파일을 설정하여 원하는 방식으로 로그를 남길 수 있다.

> \[!info] Slf4j 가 뭐지 Simple Logging Facade for Java의 줄임말이다.

사이드 프로젝트를 진행할때 보통 `logback.xml` 에 다음과 같은 설정을 한다.

* **로깅 형식** : 스레드 이름 + 날짜 같은 조합을 구성
* **어떤 로그**를 찍을지 : jdbc, hibernate ...
* **RollingFileAppender** : 로그파일을 대체하는 정책을 구성한다.
  * 파일 크기, 시간같은 설정 값을 기준으로 로그파일을 관리한다.

트래픽이 거의 없는 사이드 프로젝트에서는 대부분 로그를 확인하기 쉽다. 어떤 요청에 따라서 로그들이 찍혔는지 선형적으로 보이기 때문이다. 그런데 트래픽이 터지는 환경으로 가면 로그추적이 불편해진다.

#### 로그에도 id를 부여하자.

그러면 로그에도 id를 부여해서 사용자의 요청을 트래킹하는게 좋아보인다. 톰캣 + 스프링을 사용하면 기본적으로 스레드모델을 사용한다. 하나의 요청은 하나의 스레드에서 수행되기때문에 요청 내에서 스레드 로컬에 유니크 키를 저장하면 한 사용자의 요청에 대한 흐름을 파악할 수 있다.

* 스레드 로컬에 저장하는 방법

Spring의 요청 처리흐름을 생각하면, 요청 필터나 인터셉터에서 스레드 로컬에 uuid를 부여하고 로깅하도록하면 가능할 것 같다는 생각이 든다.

```
public class LoggingInterceptor extends HandlerInterceptor {

	ThreadLocal<String> LOG_ID = new ThreadLocal<>();

	@Override
	public boolean preHandle(HttpServletRequest request, HttpServletResponse response) {
	
		final String logId = UUID.randomUUID().toString();
		LOG_ID.set(logId);
		return true;
	}


	@Override
	public void afterCompletion(HttpServletRequest request, HttpServletResponse response) {
		LOG_ID.remove();
	}
}

```

정말 생각나는데로 적어보았다. 필터는 아니고 인터셉터단에서 스레드 로컬에 UUID를 저장하는 방식이다. 주의할 점은 스레드는 재사용되기 때문에 스레드로컬의 값을 삭제시켜주는것이다.

필터에다가 둔다면 doFilter() 이후에 finally로 잡아주는 방법도 있겠다.

#### 편하게 MDC

물론 위 방법으로도 가능하겠지만 로깅을 찍는 곳마다 스레드 로컬의 id를 꺼내서 로그에 반영해야한다. 기능적인 코드가 침투하게된다. 이를 간편하게 처리해주는게 MDC이다.

MDC는 Mapped Diagnostic Context로 logback에서 제공하고 있다. logback에서만 독자적으로 제공하는건 아니고, 로깅 프레임워크에서 스레드의 메타정보를 저장하는 공간의 의미로 사용된다. https://www.baeldung.com/mdc-in-log4j-2-logback

**사용하는 이유**

* Map기반의 자료구조처럼 사용할 수 있다. 로깅할때 넣고 싶은 데이터들은 해당 스코프에서만 활용가능한데, MDC에 넣고 활용가능하다.
* logback과 사용하기 아주 편하다. xml설정으로 로깅형식 지정하면 바로 출력 가능하다.

\`\`

```
public class LogTest {

	private final Transaction tx;
	
	...
	
	public void saveLog() {
		MDC.put("transaction.id", tx.getTransactionId())
		xxxSaveService.save(...);
		MDC.clear();
	}
}
```

요런식으로 지정할 데이터를 MDC에 넣고 로그를 찍을 수 있다. 물론 logback.xml에서 설정이 좀더 필요하다.

```
<configuration> 
	<appender name="stdout" class="ch.qos.logback.core.ConsoleAppender"> 
		<encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder"> 
		<pattern>%-4r [%t] %5p %c{1} - %m - tx.id=%X{transaction.id}tx.owner=%X{transaction.owner}%n</pattern>
		</encoder> </appender>
		<root level="TRACE">
			<appender-ref ref="stdout" />
		</root>
</configuration>
```

**해당 패턴에 맞게 MDC의 데이터를 로깅하겠다는 설정이다.**

#### 근데 스레드풀에서 비동기로 돌리면 어떻게하나요

안타깝게도 동기적으로 수행하면 너무 오래걸리거나, 그럴 필요가 없는 로직들이 있다. 우리는 비동기적으로 로직을 수행하고 싶은데, 스레드풀을 사용하면 로직이 수행되는 스레드가 달라진다

* MDC는 스레드 로컬이라면서요

맞다. 스레드 로컬을 활용하는 MDC는 비동기일때 어떻게 해야할까? 고민해보면... 스레드에 넘길때 MDC정보를준다? 요건 스레드 풀에 데코레이터를 설정해주면 된다고 한다.

{% embed url="https://blog.gangnamunni.com/post/mdc-context-task-decorator" %}
