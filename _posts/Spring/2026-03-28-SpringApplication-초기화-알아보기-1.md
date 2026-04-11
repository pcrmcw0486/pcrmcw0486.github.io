---
title: "SpringApplication 초기화 알아보기 - 1. 초기화"
date: 2026-03-28 10:00:00 +0900
categories: [Spring]
tags: [spring-boot, startup, spring-factories, application-listener, application-type]
---

> **시리즈: SpringApplication 초기화 알아보기**
> - **1. 초기화** ← 현재 글
> - [2. Environment 준비](/posts/SpringApplication-초기화-알아보기-2)
> - [3. ApplicationContext 생성](/posts/SpringApplication-초기화-알아보기-3)
> - [4. Bean Definition 등록](/posts/SpringApplication-초기화-알아보기-4)
> - [5. Bean Instance화](/posts/SpringApplication-초기화-알아보기-5)
> - [6. Context Refresh 완료 & Application 기동](/posts/SpringApplication-초기화-알아보기-6)
> - [총정리](/posts/SpringApplication-초기화-알아보기-총정리)

## 전체 개요

1. **SpringApplication 초기화** - main() -> SpringApplication.run()
2. Environment 준비 - profile, 설정 파일 로드.
3. ApplicationContext 생성 - Context Instance 준비
4. Bean Definition 등록 - refresh() 내부에서 "설계도"만 수집, 객체 생성 없음
5. Bean Instance화 - finishBeanFactoryInitialization() - 객체 실제 생성
6. Context Refresh 완료 - 웹 서버 기동
7. Application 기동 완료 - 요청 처리 준비 완료

---

## 1. Spring Application 초기화

![alt text](/assets/posts/SpringApplication-초기화-알아보기-1/SpringApplication-초기화-알아보기-1_img_001.png)

Event: ApplicationStartingEvent
- 1-1. SpringApplication이라는 클래스가 있고, 이의 생성자를 바탕으로 인스턴스 생성한다.
- 1-2. SpringApplication에 정의된 run을 실행한다.

### spring.factories ..?

- 평범한 텍스트파일
- JAVA의 ServiceLoader라는 플러그인 매커니즘을 확장한 것으로 "어떤 인터페이스를 구현한 클래스들을 자동으로 찾아서 등록"하는 방식을 만듦.
- spring-boot-autoconfiguration.jar/META-INF/spring.factories
- META-INF는 JAR의 "메타데이터"폴더로, JAVA 표준 규격(약속)임.
- 이 안에 spring.factories는 Spring과의 약속으로 설정파일이 있다.

```properties
# spring.factories (예시 - 실제 내용 축약)
org.springframework.context.ApplicationContextInitializer=\
  org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
  org.springframework.boot.context.ContextIdApplicationContextInitializer

org.springframework.context.ApplicationListener=\
  org.springframework.boot.context.logging.LoggingApplicationListener,\
  org.springframework.boot.context.FileEncodingApplicationListener
```

"이 인터페이스를 구현한 클래스들이 있다"를 선언한 목록으로, 기동 시, 모든 JAR의 spring.factories를 읽어서 Listener와 Initializer를 수집한다.

**Spring Boot 2.7 이후 부터는** spring.factories 대신 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 파일로 점차 이전되고 있다. 다만 구조는 동일함.

```text
이전 (Boot 2.7 이전):
  META-INF/spring.factories
    → 모든 것이 한 파일에 뒤섞임 (AutoConfiguration, Listener, Initializer 등)

이후 (Boot 2.7+, Boot 3.x 표준):
  META-INF/spring/
    org.springframework.boot.autoconfigure.AutoConfiguration.imports
      → AutoConfiguration 목록만 분리
  META-INF/spring.factories
    → 나머지 (Listener, Initializer 등)는 여전히 여기
```

AutoConfiguration 항목만 별도 파일로 쪼갠 것이다. spring.factories가 너무 커지고, 파싱 비용도 있고, 용도가 섞여있어서 관리가 어려웠기 때문이다.

### ApplicationType

- 클래스패스에 어떤 "라이브러리"가 있는지 보고 어플리케이션 타입을 자동 결정한다.
- 요 타입에 따라 나중에 어떤 ApplicationContext를 생성할지 결정되는 분류자이다.
- `WebApplicationType.REACTIVE` — reactor-netty가 있으면 리액티브 웹 앱
- `WebApplicationType.SERVLET` — spring-mvc가 있으면 일반 웹 앱
- `WebApplicationType.NONE` — 웹 없는 앱 (배치, CLI 등)

Type에 따라 내장 Tomcat서버가 동작 또는 Netty가 동작.

### Initializer와 Listener?

둘 다 "기동 중에 끼어드는 훅" 이지만 시점이 다르다.

```kotlin
// ApplicationContextInitializer — ApplicationContext가 생성된 직후, refresh() 전에 호출
class MyInitializer : ApplicationContextInitializer<ConfigurableApplicationContext> {
    override fun initialize(ctx: ConfigurableApplicationContext) {
        // 컨텍스트에 프로퍼티 소스를 추가하거나 설정을 덮어쓸 수 있음
        // 아직 빈은 없는 상태
        ctx.environment.propertySources.addFirst(
            MapPropertySource("custom", mapOf("my.key" to "value"))
        )
    }
}

// ApplicationListener — 이벤트가 발행될 때마다 호출
class MyListener : ApplicationListener<ApplicationStartingEvent> {
    override fun onApplicationEvent(event: ApplicationStartingEvent) {
        // 아주 초기 단계에서 로깅 설정이나 초기화 작업 가능
        println("Spring Boot 기동 시작!")
    }
}
```

직접 만들어 사용하는 경우는 많지 않지만. SpringBoot 내부적으로 수십개가 자동 등록됨.
ex) 콘솔에 뜨는 Spring 배너도 ApplicationListener가 출력함.

더해서 여기서의 Listener는 우리가 흔히 쓰는 @Listener와 같이 Bean이 아닌 Boot 인프라 리스너이다.
다만, @EventListener의 경우 EventListenerMethodProcessor라고하는 BeanPostProcessor가 관련 메서드들을 찾아서 내부적으로 "ApplicationListener(boot 인프라 리스너)에 추가" 한다.

#### 예시로 알아보기

- 예를들면 Spring 이 구동될때 뜨는 "SPRING" 배너는 boot 인프라의 리스너이다.
- boot 인프라 리스너들은 모두 observer pattern으로 구현되어있다. 즉, Spring의 관리영역이 아니게 동작.

```kotlin
// spring.factories 파일 내용 (Spring Boot 내부)
// org.springframework.context.ApplicationListener=\
//   org.springframework.boot.context.logging.LoggingApplicationListener

// Spring Boot가 Phase 01에서 실제로 하는 것
class SpringApplication(primarySource: Class<*>) {

    val listeners: List<ApplicationListener<*>>

    init {
        // 이게 전부 — 그냥 new로 생성, Bean이 아님
        listeners = SpringFactoriesLoader
            .loadFactoryNames(ApplicationListener::class.java, classLoader)
            .map { className ->
                Class.forName(className).newInstance() as ApplicationListener<*>
            }
        // ApplicationContext? 없음
        // @Autowired? 동작 안 함 — Spring 컨테이너 밖
    }
}

// 그래서 LoggingApplicationListener 내부를 보면
class LoggingApplicationListener : ApplicationListener<ApplicationEvent> {
    // @Autowired 필드가 하나도 없음
    // 생성자 주입도 없음
    // 모든 걸 직접 new로 만들거나 static으로 처리

    override fun onApplicationEvent(event: ApplicationEvent) {
        when (event) {
            is ApplicationStartingEvent -> initializeLogging()
            is ApplicationEnvironmentPreparedEvent -> applyLogConfig(event.environment)
        }
    }

    private fun initializeLogging() {
        // LogbackLoggingSystem 직접 생성
        val system = LogbackLoggingSystem(ClassUtils.getDefaultClassLoader())
        system.initialize(...)
    }
}
```

#### 그럼 배너같은 리스너를 바꾸고 싶으면..?

- 여기에서의 리스너들은 "Bean"이 아니라는거, spring이 뜨기도 전인 spring과 상관없는 JVM app이라는것이다.
- 특정 리스너들은 확장 포인트들을 제공하고 있다.
    - resources/banner.txt를 생성하면, 이를 사용해줌. (변수도 사용가능, spring.application.name 등..)
    - 또는 Banner 인터페이스를 직접 구현:

```kotlin
// Banner 인터페이스 구현 — 코드로 완전히 제어
class MyBanner : Banner {
    override fun printBanner(
        environment: Environment,
        sourceClass: Class<*>,
        out: PrintStream
    ) {
        val appName = environment.getProperty("spring.application.name") ?: "MyApp"
        out.println("""
            ╔══════════════════════════╗
            ║  $appName               ║
            ║  v${readVersion()}      ║
            ╚══════════════════════════╝
        """.trimIndent())
    }

    private fun readVersion() =
        MyApp::class.java.`package`?.implementationVersion ?: "dev"
}

// main()에서 SpringApplication에 직접 주입
// run() 호출 전에 설정해야 함 — 이 시점엔 Bean이 없기 때문
fun main(args: Array<String>) {
    val app = SpringApplication(MyApp::class.java)
    app.setBanner(MyBanner())                    // ← 여기
    app.setBannerMode(Banner.Mode.CONSOLE)
    app.run(*args)
}
```

만약 리스너 자체를 교체하고싶다면, 당연하게도 "스프링"에 알려주는 방식의 동작을 그대로 따라가야함.

```kotlin
// 방법 A: spring.factories에 직접 등록
// src/main/resources/META-INF/spring.factories
// org.springframework.context.ApplicationListener=\
//   com.example.MyStartingListener

// Bean이 아니므로 @Autowired 사용 불가 — 직접 new로 해결해야 함
class MyStartingListener : ApplicationListener<ApplicationStartingEvent> {
    override fun onApplicationEvent(event: ApplicationStartingEvent) {
        println("커스텀 시작 리스너!")
        // 주의: 이 시점엔 Environment도 없고, 빈도 없음
    }
}

// 방법 B: main()에서 직접 추가
fun main(args: Array<String>) {
    val app = SpringApplication(MyApp::class.java)
    app.addListeners(ApplicationListener<ApplicationStartingEvent> { event ->
        println("람다로 등록한 리스너")
    })
    app.run(*args)
}
```
