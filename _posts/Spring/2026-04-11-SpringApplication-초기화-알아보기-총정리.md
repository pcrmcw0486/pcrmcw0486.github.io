---
title: "SpringApplication 초기화 알아보기 - 총정리"
date: 2026-04-11 13:00:00 +0900
categories: [Spring]
tags: [spring-boot, spring-framework, startup, ioc, di, auto-configuration, annotation, reflection, aot]
---

> **시리즈: SpringApplication 초기화 알아보기**
> - [1. 초기화](/posts/SpringApplication-초기화-알아보기-1)
> - [2. Environment 준비](/posts/SpringApplication-초기화-알아보기-2)
> - [3. ApplicationContext 생성](/posts/SpringApplication-초기화-알아보기-3)
> - [4. Bean Definition 등록](/posts/SpringApplication-초기화-알아보기-4)
> - [5. Bean Instance화](/posts/SpringApplication-초기화-알아보기-5)
> - [6. Context Refresh 완료 & Application 기동](/posts/SpringApplication-초기화-알아보기-6)
> - **총정리** ← 현재 글

## Spring과 SpringBoot

Spring Framework
  = IoC/DI 컨테이너 + 웹(MVC) + 데이터 + 보안 등
    각종 모듈의 모음
  = "어플리케이션 개발에 필요한 것들을 제공하는 프레임워크"
  = 하지만 설정을 개발자가 전부 직접 해야 함
    (XML이든 Java Config든)

Spring Boot
  = Spring Framework를 편하게 쓰기 위한 것
  = 핵심은 세 가지:

    1. AutoConfiguration
       "클래스패스 보고 알아서 Bean 등록해줌"
       XML/Java Config 직접 안 써도 됨

    2. Starter 의존성
       "관련 라이브러리들을 검증된 버전으로 묶어줌"
       버전 충돌 걱정 없음

    3. 내장 서버
       "Tomcat을 JAR 안에 넣어서 main()으로 실행"
       외장 서버 + WAR 배포 불필요


### Spring

Spring(Framework)의 본질은 "객체를 대신 만들고, 연결/관리 해주는 컨테이너"이다.
```kotlin
// 1. IOC (Inversion of Control) - 객체 생성의 제어(Control)을 Spring에게 준다. (Inversion)
// 옛날 방식
class OrderService{
    val repo= OrderRepository() // 내가 직접 생성 - 결합도 높음.
}
// Spring 방식
class OrderService(
    private val repo: OrderRepository // Spring이 만들어서 주입.
)

//2. DI (Dependency Injection) - 의존성을 외부에서 주입
// OrderService는 OrderRepository가 어떻게 만들어지는지 모름.
// 테스트 시 mock으로 교체가능.
```
Spring Framework 자체는 이 컨테이너 기능과 그 위에 올라가는 모듈들의 모음입니다:
```
spring-core          ← IoC 컨테이너 핵심
spring-context       ← ApplicationContext, 이벤트 등
spring-beans         ← BeanFactory, BeanDefinition 등
spring-web           ← 웹 MVC 기반
spring-tx            ← 트랜잭션
spring-data          ← 데이터 접근 추상화
spring-security      ← 보안
...
```
그래서 이걸 전부 가져다 "라이브러리"형태로 사용가능한데 이거 직접 쓰려면..
```xml
<!-- 옛날: 이걸 직접 써야 했음 -->
<beans>
    <bean id="orderService" class="com.example.OrderService">
        <constructor-arg ref="orderRepository"/>
    </bean>
    <bean id="orderRepository" class="com.example.OrderRepository">
        <property name="dataSource" ref="dataSource"/>
    </bean>
    <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource">
        <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://localhost/mydb"/>
        <property name="username" value="root"/>
        <property name="password" value="1234"/>
    </bean>
    <!-- ... 수십 줄 더 ... -->
</beans>
```
처럼 직접 연결을 다 해주어야했음. 
강력하지만 **설정 비용이 너무 크다..**

### XML과 WAR 배포

SpringBoot이전에는 외장 톰캣이 있고. WAR로 만든 파일을 톰캣 내부에 배포하는 식
```
외장 Tomcat 실행
    ↓
WAR 파일 배포 (webapps/ 폴더에 복사)
    ↓
Tomcat이 WEB-INF/web.xml 읽음
    ↓
web.xml 안의 ContextLoaderListener, DispatcherServlet 등록 지시문 파싱
    ↓
Spring이 applicationContext.xml, dispatcher-servlet.xml 읽어서 빈 생성
```

```xml
<!-- WEB-INF/web.xml — Tomcat이 읽는 진입점 -->
<web-app>
    <!-- "Spring 컨텍스트 파일 위치가 여기다" -->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/applicationContext.xml</param-value>
    </context-param>

    <!-- Tomcat 시작 시 이 리스너가 실행됨 → Spring 컨텍스트 초기화 -->
    <listener>
        <listener-class>
            org.springframework.web.context.ContextLoaderListener
        </listener-class>
    </listener>

    <!-- URL 요청을 받는 서블릿 → 이게 dispatcher-servlet.xml 읽음 -->
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>
            org.springframework.web.servlet.DispatcherServlet
        </servlet-class>
    </servlet>
</web-app>

<!-- WEB-INF/applicationContext.xml — Spring이 읽는 빈 설정 -->
<beans>
    <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource">
        <property name="url" value="jdbc:mysql://localhost/mydb"/>
    </bean>

    <!-- 지금의 @ComponentScan에 해당 -->
    <context:component-scan base-package="com.example"/>
</beans>
```
Spring Boot는 이 모든 XML과 외장서버 의존성을 없애고, Tomcat을 JAR 안에 내장해서 main()으로 실행할 수 있게 만든거다. web.xml의 역할을 @SpringBootApplication이 대체한것이다.
그래서 SpringBoot 얘기에 내장 웹 서버 얘기가 빠질수가 없다.

### Spring Boot

위 연결 너무 싫어.. Spring을 "쓸 수 있는 상태"로 미리 조립하자.
- Spring + AutoConfiguration, Starter & Embedded Server
    - AutoConfiguration: 저 설정파일말고 잘 설정되도록.. 알아서 동작.
    - Starter : dependency 의존성 묶음.
    - Embedded Server: 외장 tomcat없이 main()으로 실행

무엇이 좋아졌나?
```
// Spring Boot 이전 — Redis 쓰려면
// 1. pom.xml에 jedis, spring-data-redis 버전 맞춰서 추가
// 2. RedisConnectionFactory Bean 직접 설정
// 3. RedisTemplate Bean 직접 설정
// 4. 직렬화 설정
// 5. ...

// Spring Boot — Redis 쓰려면
// build.gradle.kts에 한 줄
implementation("org.springframework.boot:spring-boot-starter-data-redis")

// application.yml에 한 줄
// spring.redis.host: localhost

// 끝. RedisAutoConfiguration이 나머지 다 해줌
```

### 정리

![Spring Boot / Spring Framework / JVM 레이어](/assets/posts/SpringApplication-초기화-알아보기-총정리/SpringApplication-초기화-알아보기-총정리_img_001.png)

Spring Framework = 컨테이너 + 모듈들 (직접 조립해야 함)
Spring Boot      = Spring Framework + 자동 조립 + 실행 환경

#### Starter가 하는 일

"관련있는 것들을 하나로 묶었다?"
```
// build.gradle.kts
implementation("org.springframework.boot:spring-boot-starter-web")
// 이 한 줄이 실제로 끌고 오는 것들:
// spring-web
// spring-webmvc
// spring-boot-autoconfigure  ← AutoConfiguration 포함
// jackson-databind            ← JSON 변환
// tomcat-embed-core           ← 내장 Tomcat
// tomcat-embed-websocket
// ... 등 수십 개
```
Starter 자체에는 코드가 거의 없습니다. `build.gradle` 역할을 하는 파일 하나가 전부입니다. "이것들을 같이 써야 동작해요"라는 의존성 묶음입니다.

---

## 전체 요약

1. SpringApplication 초기화 - main() -> SpringApplication.run()
2. **Environment 준비** - profile, 설정 파일 로드.
3. ApplicationContext 생성 - Context Instance 준비
4. Bean Definition 등록 - refresh() 내부에서 "설계도"만 수집, 객체 생성 없음
5. Bean Instance화 - finishBeanFactoryInitialization() - 객체 실제 생성
6. Context Refresh 완료 - 웹 서버 기동
7. Application 기동 완료 - 요청 처리 준비 완료

```
main()
  │
  ├─ Phase 01: Listener/Initializer 수집
  ├─ Phase 02: Environment(설정값) 준비
  ├─ Phase 03: ApplicationContext 생성 (빈 상자)
  │              └─ refresh() 시작
  ├─ Phase 04:   BeanDefinition 수집 (설계도)
  ├─ Phase 05:   Bean 인스턴스화 (실제 객체)
  ├─ Phase 06:   웹 서버 기동 ← HTTP 수신 가능
  │              └─ refresh() 종료
  └─ Phase 07: Runner 실행 → ApplicationReadyEvent
                             ← 트래픽 받을 준비 완료
```

![Phase 01~07 전체 요약](/assets/posts/SpringApplication-초기화-알아보기-총정리/SpringApplication-초기화-알아보기-총정리_img_002.png)

![JVM / Spring / Spring Boot 내부 구조](/assets/posts/SpringApplication-초기화-알아보기-총정리/SpringApplication-초기화-알아보기-총정리_img_003.png)

### 깔끔하게 말로 정리해보자면..

Spring Boot 기동은 크게 네 가지 흐름으로 볼 수 있습니다.
- 첫째, `준비 단계`입니다. 어떤 라이브러리가 있는지, 어떤 설정값들이 있는지 파악합니다. 클래스패스를 뒤져서 Listener와 Initializer를 수집하고(Phase 01), application.yml 등 설정 파일들을 읽어서 PropertySource 우선순위 체인으로 쌓습니다(Phase 02). 이 시점까지는 Spring 컨테이너 자체가 존재하지 않습니다.

- 둘째, `컨테이너 생성 단계`입니다. ApplicationContext라는 빈 상자를 만듭니다(Phase 03). 이 안에는 Bean들을 담는 BeanFactory, 설정값들을 담은 Environment, 이벤트 시스템 등이 함께 들어있습니다. Bean은 아직 없지만 컨테이너 자체가 존재하게 됩니다.

- 셋째, `Bean 준비 단계`입니다. 먼저 설계도를 수집합니다(Phase 04). @ComponentScan으로 패키지를 뒤지고, @Bean 메서드 시그니처를 기록하고, AutoConfiguration 조건을 평가해서 BeanDefinition이라는 설계도를 BeanFactory에 쌓아둡니다. 이 단계에서는 실제 객체가 생성되지 않고, 의존성 그래프만 완성됩니다. 그다음 그 설계도를 토대로 의존성 순서대로 실제 객체를 만듭니다(Phase 05). 생성자 → 주입 → 콜백 → 초기화 → Proxy 생성 순서로 각 Bean이 완성되고 BeanFactory의 싱글톤 저장소에 들어갑니다.

- 넷째, `가동 단계`입니다. 만들어진 Bean들 중 Lifecycle을 구현한 것들을 순서대로 가동합니다(Phase 06). Tomcat도 그중 하나입니다. 마지막으로 Runner를 실행하고 ApplicationReadyEvent를 발행하면 모든 준비가 완료됩니다(Phase 07).

### 커스터마이징 포인트 지도

![Spring Boot 커스터마이징 포인트 전체 지도](/assets/posts/SpringApplication-초기화-알아보기-총정리/SpringApplication-초기화-알아보기-총정리_img_004.png)

- Bean이 없는 시점?
    - spring.factories 등록 혹은 main()에 직접 추가.
    - 또는 전용 인터페이스를 사용 (Banner, EnvironmentPostProcessor 등)
- Bean이 생긴후에 끼어들고 싶다면..?
    - 자유롭게.. 늘 하던대로.

### 각 Bean의 초기화 순서

```
① 생성자 / @Bean 메서드 실행
② 의존성 주입 (@Autowired 등)
③ Aware 콜백 (setApplicationContext 등)
④ @PostConstruct
⑤ BeanPostProcessor
```
이 과정은 빈 하나하나가 만들어질때마다 이 순서를 밟는다.
그럼 만약 문제 상황에서 RedisCacheManager가 ApplicationContextProvider를 Autowired 한다면?

---

## 더 알아보기

### Annotation은 단순 포스트잇인데.. 그럼 @SpringBootApplication 어노테이션 같은게 동작할 수 있는 이유는 뭘까?

- @Retention이 핵심이다.
- 이게 없거나 @Retention(AnnotationRetention.SOURCE) 로 설정되면, 컴파일 후 .class 파일에서 사라진다.
- @Retention(AnnotationRetention.RUNTIME)이 있으면 JVM이 실행중에도 "해당 메서드나 클래스에 어노테이션이 붙어있다"는 정보를 기억한다.
- JVM은 Reflection이라는 기능을 제공하고, (실행 중 클래스 구조를 볼 수 있는) 이를 이용해서 제어할 수 있다.
- ex) Lombok의 @Data처럼 getter/setter 코드를 만들고 없어지거나, KAP/KSP 기반 라이브러리에서 generated 소스를 만들고 사라지는 케이스
- ex) @Bean, @Component등, 빌드 시에는 아무것도 생성안되지만 런타임에 처리
```kotlin
// SOURCE: 컴파일러용. .class 파일에 없음
// 예: @Suppress("unused") — 컴파일러 경고 억제용
@Retention(AnnotationRetention.SOURCE)
annotation class SuppressWarning

// BINARY: .class에 있지만 JVM이 런타임에 못 읽음
// 라이브러리 내부 마킹용
@Retention(AnnotationRetention.BINARY)

// RUNTIME: .class에 있고 JVM이 런타임에 읽을 수 있음 ← Spring 애노테이션 전부 이것
@Retention(AnnotationRetention.RUNTIME)
annotation class Bean
```
- 다만, 기동 시 클래스를 전부 "스캔"해야하는 비용이 있다. (기동 시 오래걸리는 이유)
- 이 단점을 해결하기 위해 나온게 Quarkus나 Micronaut같은 프레임워크로 빌드 시 Reflection 결과를 미리 계산하는 방식 (AOT, Ahead of Time)을 사용한다.
- Spring도 Spring Native / Spring AOT로 이 방향을 추가 지원 중이다.

---

### TODO
- [ ] SpringEvent 기반으로 각 Phase에서 어떤 이벤트가 발행되는지 전체 정리

---

### 참조
- [Spring Framework Overview](https://docs.spring.io/spring-framework/reference/overview.html) — Spring 모듈 구조
- [Spring Boot Features](https://docs.spring.io/spring-boot/reference/features/spring-application.html) — SpringApplication 기동 과정
- [Auto-configuration :: Spring Boot](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html) — AutoConfiguration 동작
- [Annotation Processing](https://docs.oracle.com/javase/tutorial/java/annotations/) — Java 어노테이션 기초
