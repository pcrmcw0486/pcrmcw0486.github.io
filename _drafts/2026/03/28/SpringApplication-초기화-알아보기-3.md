---
title: "SpringApplication 초기화 알아보기 - 3. ApplicationContext 생성"
categories: [Spring]
tags: [spring-boot, startup, application-context, bean-definition, initializer, context-hierarchy]
---

> **시리즈: SpringApplication 초기화 알아보기**
> - [1. 초기화](/drafts/SpringApplication-초기화-알아보기-1)
> - [2. Environment 준비](/drafts/SpringApplication-초기화-알아보기-2)
> - **3. ApplicationContext 생성** ← 현재 글

### 전체 개요

1. SpringApplication 초기화 - main() -> SpringApplication.run()
2. Environment 준비 - profile, 설정 파일 로드.
3. **ApplicationContext 생성** - Context Instance 준비
4. Bean Definition 등록 - refresh() 내부에서 "설계도"만 수집, 객체 생성 없음
5. Bean Instance화 - finishBeanFactoryInitialization() - 객체 실제 생성
6. Context Refresh 완료 - 웹 서버 기동
7. Application 기동 완료 - 요청 처리 준비 완료

---

### 3. ApplicationContext 생성 - Context Instance 준비
ApplicationContextInitializedEvent / ApplicationPreparedEvent
Phase02에서 Environment들이 준비완료되었고 PropertySource체인들이 구성되었음.
이를 사용하여, Application Context를 생성함.

#### ApplicationContext가 뭔가?
ApplicationContext = Bean을 담는 상자 + 그 상자를 관리하는 관리자.
```
interface ApplicationContext {
    // 1. Bean 조회
    fun getBean(name: String): Any
    fun <T> getBean(requiredType: Class<T>): T

    // 2. Environment 접근
    fun getEnvironment(): Environment

    // 3. 이벤트 발행
    fun publishEvent(event: Any)

    // 4. 리소스 로딩 (파일, classpath 등)
    fun getResource(location: String): Resource

    // 5. 메시지 소스 (i18n)
    fun getMessage(code: String ...): String
}
```


### ApplicationContext 생성과 Bean refresh가 왜 두개로 나뉘어져있지? 한번에 생성하면 되는거아닌가?
Spring초창기에는 XML로 모든 설정을 미리 다 써두어, context를 만드는 동시에 Bean을 모두 초기화함.
- 테스트하기 어렵다.
    - 테스트에서 실제 DB대신 mock으로 바꿔치기 할 타이밍이 없다. 
    - 이를 분리함으로써 아래처럼 할 수 있다. 
    ```
    val context = AnnotationConfigApplicationContext() // 빈 상자

    // bean이 초기화 되기전 시점.
    context.environment.propertySources.addFirst(
        MapPropertySource("test", mapOf("spring.datasource.url" to "jdbc:h2:mem:test"))
    )
    // 이제 초기화 
    context.refresh()

    ---

    // Spring Boot Test가 이 구조를 활용한다.
    @SpringBootTest
    class OrderServiceTest{

        // @MockBean - refresh() 내부(Phase 04)에서 BeanFactoryPostProcessor를 통해
        // 실제 BeanDefinition을 Mock으로 교체하는 것.
        // (Spring Boot 3.4부터 deprecated → @MockitoBean으로 대체)
        @MockBean
        lateinit var paymentClient: PaymentClient
        // 실제 PaymentClient 대신 Mock이 Bean으로 등록됨.
    }
    ```
    - 여러 예시
        - 1. 테스트 환경에서 특정 Bean을 Mock으로 교체하거나
        - 2. 동적으로 어떤 Bean을 등록할지 조건부 결정
        - 3. 외부 설정을(Spring Cloud Config, SecretsManager)을 Bean 생성 전에 PropertySource로 주입
        - 4. 여러 Context를 Parent-Child로 연결 (테스트 격리)
    
- 중간에 끼어들 수 없다.

그래서 컨테이너만 먼저 만들고 그 사이에 Initializer 등을 끼워넣은 뒤, 준비가 되면 `refresh()`를 호출하는 구조가 되었다. 

```
// SpringApplication.run() 내부 (단순화)
fun run(): ConfigurableApplicationContext {

    // Phase03-A : 빈 컨테이너 생성 - 아직 bean 없음.
    val context = createApplicationContext()

    // Phase03-B: Initializer 적용 (끼어들 수 있는 시점)
    applyInitializers(context)

    listeners.contextPrepared(context) // -> ApplicationContextInitializedEvent 발행

    // 소스 등록 (어디서부터 Bean을 스캔할지 알려줌)
    load(context, sources)

    listeners.contextLoaded(context) // -> ApplicationPreparedEvent 발행

    // Phase 04~06 : 실제 Bean 등록 + 초기화
    refreshContext(context)
    
    return context
}


// Phase01에서 감지한 ApplicationType이 여기서 사용된다.
// Phase03-A: 빈컨테이너 생성 
// ApplicationType에 따라 다른 구현체 생성
fun createApplicationContext(): ConfigurableApplicationContext { 
    ...
    return when (webApplicationType){
        // 일반 웹 (spring-boot-starter-web)
        WebApplicationType.SERVLET -> AnnotationConfigServletWebServerApplicationContext()
        // 리액티브 웹 (spring-boot-starter-webflux)
        WebApplicationType.REACTIVE -> AnnotationConfigReactiveWebServerApplicationContext()
        //웹 없음
        WebApplicationType.NONE-> AnnotationConfigApplicationContext()
    }
}

// 각각 세가지 모두 ApplicationContext의 구현체이다.
// SERVLET & REACTIVE는 내장 웹 서버 (Tomcat/Netty등) 을 띄우는 기능이 추가되어있다.

// ApplicationContext생성시, 내부에서는 이런것들이 자동으로 등록됨

// 1. BeanFactory 생성(실제 Bean을 담는 저장소), ApplicationContext는 BeanFactory를 감싼 래퍼
val beanFactory = DefaultListableBeanFactory()

// 2. 핵심 Bean PostProcessor 등록
// @Autowired를 처리하는 AutowiredAnnotationBeanPostProcessor
// @Value를 처리하는 Processor
// 등이 등록된다.

// 3. Environment연결
context.environment = environment 

// 4. ResourceLoader 설정
// classpath:, file: 등의 리소스 접근 방법.
```

#### load(context, sources) - 소스 등록?
```
// sources = @SpringBootApplication이 붙은 클래스
// 예: MyApp::class.java

fun load(context: ApplicationContext, sources: Array<Any>) {
    val loader = BeanDefinitionLoader(context, sources)
    loader.load()
}

// BeanDefinitionLoader가 하는 것:
// "@SpringBootApplication이 붙은 MyApp 클래스를 
//  BeanDefinition으로 딱 하나만 등록"

// 결과:
// BeanFactory에는 단 하나의 BeanDefinition만 존재
// "MyApp이라는 클래스가 있다"는 정보만
```

##### BeanDefinition과 실제 Bean 인스턴스는 뭐가 다른데?
- BeanDefinition = "Bean을 어떻게 만들지에 대한 설계도"
```
val bd = RootBeanDefinition(UserService::class.java).apply {
    scope = BeanDefinition.SCOPE_SINGLETON  // 싱글톤?
    isPrimary = true                         // Primary?
    isLazyInit = false                       // Lazy?
    // 생성자 인자, 프로퍼티 값 등 메타데이터만 담김
}
ctx.registerBeanDefinition("userService", bd)
// 이 시점에 UserService() 는 호출되지 않음

BeanDefinition 등록 (Phase 03~04)
    ↓
설계도만 BeanFactory에 쌓임
    ↓
Phase 05: finishBeanFactoryInitialization()
```
BeanDefinition이 "설계도"라는 게 중요한 이유는, 이 덕분에 @Conditional 평가, @Lazy, scope, Primary 같은 결정을 실제 객체 생성 전에 할 수 있습니다. 설계도 단계에서 "이 Bean 만들지 말지" 결정

Phase04에서 자동화된것 빼고 미리 할 수 있다.. 로 이해하면 될것 같다. 

###### 이런것도 있당
// 3. Spring Data가 Repository 인터페이스를 Bean으로 등록할 때
//    interface OrderRepository : JpaRepository<Order, Long>
//    → 인터페이스인데 어떻게 Bean이 되나?
//    → Spring Data가 내부적으로 Proxy 구현체 BeanDefinition을 등록
//       실제로 등록되는 건 JDK Dynamic Proxy 인스턴스

```
Phase 03 끝 시점의 BeanFactory:
  BeanDefinition 1개
    └── MyApp (=@SpringBootApplication 클래스)
    
의존성 그래프: 아직 없음
나머지 모든 Bean: 아직 모름

Phase 04에서 MyApp을 시작점으로
@ComponentScan, @Import 등을 따라가며
나머지 BeanDefinition들을 전부 수집함

// @SpringBootApplication을 열어보면
@SpringBootApplication
= @SpringBootConfiguration      // = @Configuration
+ @EnableAutoConfiguration      // AutoConfig 활성화
+ @ComponentScan                 // 패키지 스캔 시작점

// 이 클래스 하나가 Phase 04에서 모든 스캔의 출발점
```

#### Initializer? 
ApplicationContextInitializer - 지금 호출 되는 훅
Phase01에서 spring.factories로 수집해둔 Initializer들이 여기서 "실제로 호출"된다.
```
1. DelegatingApplicationContextInitializer
context.initializer.classes 프로퍼티를 읽어서
거기 적힌 Initializer들을 추가로 실행해주는 위임자
→ application.yml에서 Initializer를 지정할 수 있게 해줌
// "Initializer를 코드(spring.factories)가 아닌
//  application.yml에서 지정하고 싶다"
// application.yml
context:
  initializer:
    classes: com.example.MyInitializer, com.example.AnotherInitializer

2. ConditionEvaluationReportLoggingListener
@Conditional 평가 결과를 수집
기동 실패 시 "왜 이 Bean이 생성 안 됐는지" 리포트 출력하는 것

3. ConfigurationWarningsApplicationContextInitializer
잘못된 설정 감지해서 경고 출력
예: @ComponentScan을 너무 넓은 패키지에 걸었을 때

4. ServerPortInfoApplicationContextInitializer  
서버가 실제로 뜬 포트를 Environment에 등록
local.server.port 프로퍼티로 접근 가능하게 해줌
@SpringBootTest(webEnvironment = RANDOM_PORT) 할 때 쓰는 그것

직접 만드는 경우
class SecretsManagerInitializer : ApplicationContextInitializer<ConfigurableApplicationContext> {
    override fun initialize(ctx: ConfigurableApplicationContext) {
        // Bean 생성 전에 Secrets 값을 PropertySource에 추가
        val secrets = fetchSecretsSync()  // 동기 호출
        ctx.environment.propertySources.addFirst(
            MapPropertySource("secrets", secrets)
        )
    }
}

// spring.factories에 등록
// org.springframework.context.ApplicationContextInitializer=\
//   com.example.SecretsManagerInitializer
```

##### 코드로 자세히 알아보기
```
// applyInitializers() 내부 
fun applyInitializers(context: ConfigurableApplicationContext) {
    initializers
    .sortedBy { it.order } // 순서 있음. 
    .forEach { initializer ->
        initializer.initialize(context) // 각 Initializer 호출
    }
}

// Initializer가 할 수 있는 것.
class MyInitializer : ApplicationContextInitializer<ConfigurableApplicationContext> {
    override fun initialize(ctx: ConfigurableApplicationContext) {
        // 이 시점에서 가능한것 들

        // 예시 1. PropertySource 추가 (Phase02 이후 이지만 아직 bean이 생성 안되었으니 가능)
        ctx.environment.propertySources.addFirst(
            MapPropertySource("my-source", mapOf("key" to "value"))
        )
        // 예시 2. 활성 Profile추가.
        ctx.environment.addActiveProfile("my-profile")

        // 예시 3. BeanDefinition 직접 등록 (아직 refresh 전이라 가능) Phase04에서 하는걸 직접 진행 가능하다.
        // @Conditional 조건이 복잡해서 코드로만 표현이 가능하거나, 외부 라이브러리 클래스를 Bean 으로 등록하고싶을때 (외부 클래스에 @Component를 붙일 수 없는 경우)
        (ctx as GenericApplicationContext).registerBean(MyBean::class.java)

        // 불가능한것
        // ctx.getBean(...) -> refresh전이라 아직 Bean은 없음.
    }
}
```

#### Context의 계층 구조
Spring MVC 시절 (외장 tomcat시절)에는 context가 두개였다. 
```
Root ApplicationContext (ContextLoaderListener가 생성)
  └── Service, Repository, DataSource Bean들

  └── Child: DispatcherServlet WebApplicationContext
        └── Controller, ViewResolver Bean들
              (Root의 Bean을 참조 가능, 반대는 불가)
```
- 왜냐면 이때는 DispatcherServlet이 여러 개일 수 있었고, 공통Bean은 Root에 두고 재사용하기 위해서였음.
- SpringBoot는 이 복잡성을 없애고 하나의 Context로 통합하였음. (계층 구조를 다룰일은 없다.)

##### context 계층을 이해해보자?
예전 XML
```
<!-- web.xml -->
<web-app>
    <!-- Root Context: 공통 Bean들 -->
    <listener>
        <listener-class>ContextLoaderListener</listener-class>
    </listener>
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/root-context.xml</param-value>
    </context-param>
    
    <!-- DispatcherServlet 1: /api/* 담당 -->
    <servlet>
        <servlet-name>api</servlet-name>
        <servlet-class>DispatcherServlet</servlet-class>
        <!-- /WEB-INF/api-servlet.xml 읽음 → 별도 Child Context -->
    </servlet>
    
    <!-- DispatcherServlet 2: /admin/* 담당 -->
    <servlet>
        <servlet-name>admin</servlet-name>
        <servlet-class>DispatcherServlet</servlet-class>
        <!-- /WEB-INF/admin-servlet.xml 읽음 → 또 다른 Child Context -->
    </servlet>
</web-app>

Root Context (root-context.xml)
  UserService, OrderService, DataSource ...
  ↑ (자식은 부모 Bean 참조 가능, 반대 불가)
  ├── api Child Context
  │     ApiController, ApiExceptionHandler ...
  │     → /api/* 요청 처리
  └── admin Child Context
        AdminController, AdminExceptionHandler ...
        → /admin/* 요청 처리
```

관련해서 spring mvc를 공부해보면 현재를 더 이해하기 좋은데, 현재는
```
옛날 문제:
  DispatcherServlet이 여러 개 → 각자 Child Context 필요
  Child마다 Controller, HandlerMapping 등 별도 존재
  설정이 xml 여러 개로 분산됨

지금 해결:
  DispatcherServlet이 딱 하나
  모든 요청을 이 하나가 받음
  URL 분기는 Context 분리가 아니라 @RequestMapping으로 처리

  @RestController
  @RequestMapping("/api")
  class ApiController { ... }   ─┐
                                  ├── 하나의 Context 안에서
  @RestController                 │   HandlerMapping이
  @RequestMapping("/admin")       │   URL 보고 적절한
  class AdminController { ... } ─┘   Controller로 라우팅

HandlerMapping이 하는 일:
"이 URL 처리할 수 있는 Controller 찾아줘" 요청을 받아 @RequestMapping 정보를 보고 적절한 메서드를 반환한다.
즉, XML시절의 URL분기를 'context분리'로 해결했고, 지금은 '같은 context안에서 HandlerMapping'이 routing해준다.
```

- 요즘 SpringBoot 단일앱에서는 거의 사용하지않지만, 아직 살아있는 곳이 있기는 하다.
1. Spring Batch - 기본은 JobScope/StepScope로 같은 Context 안에서 격리하지만, `@EnableBatchProcessing(modular=true)` + `GenericApplicationContextFactory`를 사용하면 Job 설정별 별도 Child Context를 opt-in할 수 있다.
2. 멀티 테넌시 
    - 테넌트마다 별도 context를 하고 싶다면.
3. 플러그인 시스템
4. Spring Cloud Function

#### Phase03 정리
```
Phase 03 끝 시점의 상태:

ApplicationContext 인스턴스 생성 완료 (ApplicationType에 따라 구현체 결정)
  └── BeanFactory(DefaultListableBeanFactory) 생성됨
  └── Environment 연결됨
  └── 핵심 BeanPostProcessor 등록됨 (@Autowired, @Value 처리기)
  └── Initializer들 실행 완료
  └── BeanDefinition 1개 등록됨 (= @SpringBootApplication 클래스)

아직 없는 것:
  나머지 BeanDefinition들 — Phase 04에서 @ComponentScan, @Import 등으로 수집
  Bean 인스턴스 — Phase 05에서 생성

이벤트:
  ApplicationContextInitializedEvent → Initializer 실행 후 발행
  ApplicationPreparedEvent → load(sources) 후 발행
```

---

### 참조
- [SpringApplication :: Spring Boot](https://docs.spring.io/spring-boot/reference/features/spring-application.html) — 기동 과정 전체 흐름
- [ApplicationContext Javadoc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/ApplicationContext.html) — ApplicationContext 인터페이스
- [Bean Overview :: Spring Framework](https://docs.spring.io/spring-framework/reference/core/beans/definition.html) — BeanDefinition 구조와 역할
- [Container Extension Points :: Spring Framework](https://docs.spring.io/spring-framework/reference/core/beans/factory-extension.html) — BeanFactoryPostProcessor, @MockBean 동작 원리
- [Late Binding :: Spring Batch](https://docs.spring.io/spring-batch/reference/step/late-binding.html) — JobScope/StepScope
- [GenericApplicationContextFactory :: Spring Batch](https://docs.spring.io/spring-batch/docs/current/api/org/springframework/batch/core/configuration/support/GenericApplicationContextFactory.html) — 모듈별 Child Context opt-in
