---
title: "SpringApplication 초기화 알아보기 - 4. Bean Definition 등록"
date: 2026-04-01 11:00:00 +0900
categories: [Spring]
tags: [spring-boot, startup, bean-definition, component-scan, configuration-class-post-processor, cglib, auto-configuration]
---

> **시리즈: SpringApplication 초기화 알아보기**
> - [1. 초기화](/posts/SpringApplication-초기화-알아보기-1)
> - [2. Environment 준비](/posts/SpringApplication-초기화-알아보기-2)
> - [3. ApplicationContext 생성](/posts/SpringApplication-초기화-알아보기-3)
> - **4. Bean Definition 등록** ← 현재 글
> - [5. Bean Instance화](/posts/SpringApplication-초기화-알아보기-5)
> - [6. Context Refresh 완료 & Application 기동](/posts/SpringApplication-초기화-알아보기-6)

## 전체 개요

1. SpringApplication 초기화 - main() -> SpringApplication.run()
2. Environment 준비 - profile, 설정 파일 로드.
3. ApplicationContext 생성 - Context Instance 준비
4. **Bean Definition 등록** - refresh() 내부에서 "설계도"만 수집, 객체 생성 없음
5. Bean Instance화 - finishBeanFactoryInitialization() - 객체 실제 생성
6. Context Refresh 완료 - 웹 서버 기동
7. Application 기동 완료 - 요청 처리 준비 완료

---

## 4. Bean Definition 등록

- @ComponentScan -> @Controller, @Service, @Repository
- ConfigurationClassPostProcessor -> @Configuration, @Bean 처리
- Bean에 해당하는 것들을 Bean Definition으로 등록.
- @Import, @Conditional, @EnableAutoConfiguration 처리
- BeanFactoryPostProcessor실행 (placeholder 치환 등)
- BeanDefinition = "이름 + 클래스 + 의존성 목록"만 담은 메타데이터로 객체가 아님.

### 이전에는 어떠했는가? 무얼 해결하고 싶었는가?

```xml
<!-- 옛날 방식 — 개발자가 모든 Bean을 직접 선언 -->
<beans>
    <bean id="orderService" class="com.example.OrderService">
        <constructor-arg ref="orderRepository"/>
        <constructor-arg ref="paymentClient"/>
    </bean>
    <bean id="orderRepository" class="com.example.OrderRepository">
        <property name="dataSource" ref="dataSource"/>
    </bean>
    <!-- 수십 개의 Bean을 전부 손으로... -->
</beans>
```

XML 관리 및 유지보수의 어려움이 생김.
- 그래서 Spring 2.5에서 `@Component`, `@Autowired`가 등장하면서 "클래스에 애노테이션만 붙이면 Spring이 알아서 찾아준다"는 방식이 됐다. Phase 04가 바로 그 "알아서 찾는" 과정이다.

### "설계도를 수집하자"

Phase04는 `refresh()` 내부에서 일어난다.
- 핵심은 실제 객체를 만드는게 아니라
    - 어떤 "Bean"들이 있고, 어떻게 만들면 되는지?
    - 설계도(BeanDefinition)을 전부 수집하는것.

```kotlin
// AbstractApplicationContext.refresh() 내부 (단순화)
fun refresh() {
    // Phase 04-A: BeanFactory 준비
    prepareBeanFactory(beanFactory)

    // Phase 04-B: BeanDefinition 전부 수집 ← 핵심
    invokeBeanFactoryPostProcessors(beanFactory)

    // Phase 04-C: BeanPostProcessor 등록 (실행은 Phase 05)
    registerBeanPostProcessors(beanFactory)

    // Phase 05로 넘어감
    finishBeanFactoryInitialization(beanFactory)
}
```

### Phase04-A: BeanFactory 기본 설정

- ApplicationContextAwareProcessor 등록 -> Aware 콜백 담당자.
- ApplicationListenerDetector 등록 -> Bean이면서 Listener인것 감지

### Phase04-B : 어떻게 Bean Definition을 수집할까?

`invokeBeanFactoryPostProcessors(beanFactory)`

Phase03에서 @SpringBootApplication 클래스와 인프라 BeanDefinition들을 등록해뒀음. 여기서 퍼져나가자.

`@SpringBootApplication` = `@SpringBootConfiguration`(@Configuration) + `@EnableAutoConfiguration`(imports파일처리) + `@ComponentScan`(패키지 스캔)

BeanDefinition 추가/수정 가능한 마지막 시점이다.

**경로 1: @ComponentScan**

SpringApp의 패키지와 하위 패키지를 전부 뒤짐. @Component, @Service, @Repository, @Controller 클래스를 발견하여 등록.

```kotlin
@Service
class OrderService(val repo: OrderRepository)
// -> "orderService라는 이름의 bean, OrderService로 만들면 됨"을 등록
```

**경로 2: @Configuration + @Bean**

```kotlin
@Configuration
class RedisConfig {
    @Bean
    fun redisTemplate(): RedisTemplate<*,*> = ...
}
// -> redisTemplate라는 이름의 Bean
// RedisConfig.redisTemplate() 메서드 호출하면 됨 을 등록
// 설계도만 등록
```

**경로 3: @EnableAutoConfiguration**

- `META-INF/spring/...AutoConfiguration.imports` 파일을 읽음
- 각 AutoConfiguration 클래스에서 @Conditional 평가
- 조건이 맞으면 Bean Definition을 등록

#### ConfigurationClassPostProcessor

이 모든 스캔을 실제로 수행하는 클래스로, BeanFactoryPostProcessor의 구현체이며, Phase04에서 가장 먼저 실행됨.

- Phase03에서 Context생성시, ConfigurationClassPostProcessor의 BeanDefinition을 등록해두었다.
- Phase04-B가 실행될때 우선 ConfigurationClassPostProcessor를 인스턴스화 하고 실행한다. (특별히 먼저 꺼내어서)

```kotlin
// 동작 순서 (단순화)
class ConfigurationClassPostProcessor : BeanDefinitionRegistryPostProcessor {

    override fun postProcessBeanDefinitionRegistry(registry: BeanDefinitionRegistry) {

        // 1. @SpringBootApplication(@Configuration) 클래스 발견
        val configClasses = findConfigurationClasses(registry)

        // 2. 각 @Configuration 클래스 파싱
        val parser = ConfigurationClassParser(...)
        parser.parse(configClasses)

        // 파싱 중에:
        // @ComponentScan → 해당 패키지 전체 탐색
        // @Import → 가져온 클래스 추가 처리
        // @Bean 메서드 → 메서드 시그니처만 기록 (실행 안 함)
        // @ImportResource → XML 파일이면 XML도 처리

        // 3. 파싱 결과를 BeanDefinition으로 변환해서 등록
        val reader = ConfigurationClassBeanDefinitionReader(...)
        reader.loadBeanDefinitions(parser.configurationClasses)
    }
}
```

조금 더 단순화해본다면, 아래와 같은 @Configuration이 있을때:

```kotlin
@Configuration
class MyConfig {
    @Bean
    fun orderService(): OrderService {
        println("나 실행됨!")  // Phase 04에서는 출력 안 됨
        return OrderService()
    }
}
```

@Bean 메서드를 발견하면:

```kotlin
fun readBeanMethod(method: Method) {
    val bd = ConfigurationClassBeanDefinition()
    bd.setBeanClass(method.returnType)          // OrderService::class
    bd.setFactoryBeanName("myConfig")           // 어느 Config 클래스에서
    bd.setFactoryMethodName("orderService")     // 어느 메서드로 만드는지
    // method.invoke() 호출 안 함 ← 핵심

    registry.registerBeanDefinition("orderService", bd)
    // "나중에 orderService Bean이 필요하면
    //  myConfig.orderService() 호출하면 됨" 메모만 남김
}
```

추후 실제 Bean 생성 시점(Phase 05)에서:

```kotlin
val configInstance = beanFactory.getBean("myConfig")
val bean = method.invoke(configInstance) // 이때 println이 출력됨.
```

#### @Configuration 클래스와 CGLIB 프록시

ConfigurationClassPostProcessor는 BeanDefinition 수집이 끝나면, @Configuration 클래스를 CGLIB 프록시로 감싼다.

**CGLIB이란?**
- Code Generation Library. 런타임에 바이트코드를 생성해서 기존 클래스의 서브클래스(프록시)를 만드는 라이브러리.
- JDK Dynamic Proxy는 **인터페이스 기반**으로만 프록시를 만들 수 있지만, CGLIB은 **클래스 자체를 상속**해서 프록시를 만든다.

```kotlin
// JDK Dynamic Proxy — 인터페이스 필요
interface OrderRepository { fun findById(id: Long): Order }
// → Proxy.newProxyInstance()로 인터페이스 구현체 생성

// CGLIB — 클래스 직접 상속
class OrderService { fun findOrder() = ... }
// → OrderService를 상속한 OrderService$$EnhancerByCGLIB$$xxx 클래스를 런타임에 생성
// → 메서드 호출을 가로채서(intercept) 추가 로직 실행 가능
```

**왜 @Configuration에 CGLIB 프록시가 필요한가?**

```kotlin
@Configuration
class AppConfig {
    @Bean
    fun orderRepository(): OrderRepository {
        return OrderRepository(dataSource()) // ← dataSource()를 직접 호출
    }

    @Bean
    fun dataSource(): DataSource {
        println("DataSource 생성!")
        return HikariDataSource(...)
    }
}
```

CGLIB 프록시가 없으면:
- `orderRepository()`가 `dataSource()`를 호출 → 새 DataSource 생성
- Spring이 `dataSource` Bean을 만들 때 → 또 새 DataSource 생성
- **싱글톤이 깨진다!** DataSource가 2개.

CGLIB 프록시가 있으면:
- `AppConfig`가 `AppConfig$$EnhancerByCGLIB$$xxx`로 교체됨
- `dataSource()` 호출이 가로채져서, 이미 만들어진 싱글톤 Bean이 있으면 그걸 반환
- **싱글톤 보장**

```kotlin
// CGLIB이 하는 일 (개념적)
class AppConfig$$EnhancerByCGLIB : AppConfig() {
    override fun dataSource(): DataSource {
        // BeanFactory에 이미 있으면 기존 것 반환
        if (beanFactory.containsSingleton("dataSource")) {
            return beanFactory.getBean("dataSource") as DataSource
        }
        // 없으면 원본 메서드 실행
        return super.dataSource()
    }
}
```

참고: `@Bean(proxyBeanMethods = false)`로 설정하면 CGLIB 프록시를 끌 수 있다. @Bean 메서드 간 호출이 없는 경우 (라이트 모드) 기동 속도가 빨라진다. Spring Boot의 AutoConfiguration 클래스 대부분이 이 옵션을 사용한다.

#### PropertySourcesPlaceholderConfigurer — ${...} 치환

Phase 02에서 Environment에 등록된 설정값이 있다면, BeanDefinition 안의 `${...}` placeholder를 실제 값으로 치환하는 것도 Phase 04에서 일어난다.

```kotlin
// PropertySourcesPlaceholderConfigurer는 BeanFactoryPostProcessor 구현체
// invokeBeanFactoryPostProcessors() 에서 실행됨

// 예: BeanDefinition에 이런 값이 있으면
@Value("\${redis.host}")  // 아직 문자열 "${redis.host}" 그대로

// PropertySourcesPlaceholderConfigurer가
// Environment의 PropertySource 체인에서 "redis.host" 를 찾아서
// BeanDefinition 안의 "${redis.host}" → "actual-redis.com" 으로 치환

// Phase 05에서 Bean이 만들어질 때는 이미 실제 값이 들어가 있음
```

즉, Phase 02에서 PropertySource에 값이 준비되고 → Phase 04에서 BeanDefinition의 placeholder가 치환되고 → Phase 05에서 Bean이 만들어질 때 바인딩되는 흐름이다.

### BeanDefinition 의존성 정보

- 각 BeanDefinition에는 생성자/필드 파라미터 정보가 포함되어 있어 A→B→C 의존 관계 메타데이터를 파악할 수 있다.
- 다만 실제 의존성 해소와 순환 의존성 감지는 Phase 05(Bean 인스턴스 생성 시점)에 발생한다.
  - Bean A를 만들다가 Bean B가 필요 → B를 만들다가 A가 필요 → 이때 순환 의존성 감지.

### Phase04-C : BeanPostProcessor 등록 (실행은 Phase 05)

- BeanPostProcessor들을 미리 "인스턴스화"해서 등록 (다른 Bean보다 먼저 만들어져야하므로)
- `AutowiredAnnotationBeanPostProcessor` -> @Autowired 처리를 위함
- `CommonAnnotationBeanPostProcessor` -> @PostConstruct, @PreDestroy 처리
- `AbstractAutoProxyCreator` -> @Transactional, AOP Proxy 생성 담당

#### BeanFactoryPostProcessor과 뭐가 다른가요?

- **BeanFactoryPostProcessor**는 BeanDefinition(설계도)를 조작하는 훅
- **BeanPostProcessor**는 Bean 인스턴스 생성 전후를 가로채는 훅
    - 막 생성된 bean 객체를 받아서 조작/ 값주입, Proxy로 감싸거나 등등.

## 간단정리

"Bean들을 최대한 찾고, 찾기 위한 특수 Bean들은 먼저 만들어지고,
이를 이용해서 Bean들을 찾아 의존성을 등록한다"

```text
BeanFactory 안의 상태
  singletonObjects:  environment, ConfigurationClassPostProcessor,
                     AutowiredAnnotationBeanPostProcessor 등 소수
  beanDefinitionMap: 수백 개의 설계도 (아직 인스턴스 없음)
  beanPostProcessors: PostProcessor 인스턴스 리스트 (대기 중)
```

---

### 참조
- [ConfigurationClassPostProcessor Javadoc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/ConfigurationClassPostProcessor.html)
- [Full @Configuration vs "lite" @Bean mode :: Spring Framework](https://docs.spring.io/spring-framework/reference/core/beans/java/configuration-annotation.html) — CGLIB 프록시와 proxyBeanMethods
- [Container Extension Points :: Spring Framework](https://docs.spring.io/spring-framework/reference/core/beans/factory-extension.html) — BeanFactoryPostProcessor vs BeanPostProcessor
- [PropertySourcesPlaceholderConfigurer Javadoc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/support/PropertySourcesPlaceholderConfigurer.html) — ${...} placeholder 치환
