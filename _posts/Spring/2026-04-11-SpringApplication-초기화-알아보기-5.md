---
title: "SpringApplication 초기화 알아보기 - 5. Bean Instance화"
date: 2026-04-11 11:00:00 +0900
categories: [Spring]
tags: [spring-boot, startup, bean-instantiation, dependency-graph, beanpostprocessor, post-construct, aware, cglib]
---

> **시리즈: SpringApplication 초기화 알아보기**
> - [1. 초기화](/posts/SpringApplication-초기화-알아보기-1)
> - [2. Environment 준비](/posts/SpringApplication-초기화-알아보기-2)
> - [3. ApplicationContext 생성](/posts/SpringApplication-초기화-알아보기-3)
> - [4. Bean Definition 등록](/posts/SpringApplication-초기화-알아보기-4)
> - **5. Bean Instance화** ← 현재 글
> - [6. Context Refresh 완료 & Application 기동](/posts/SpringApplication-초기화-알아보기-6)
> - [총정리](/posts/SpringApplication-초기화-알아보기-총정리)

### 전체 개요

1. SpringApplication 초기화 - main() -> SpringApplication.run()
2. Environment 준비 - profile, 설정 파일 로드.
3. ApplicationContext 생성 - Context Instance 준비
4. Bean Definition 등록 - refresh() 내부에서 "설계도"만 수집, 객체 생성 없음
5. **Bean Instance화** - finishBeanFactoryInitialization() - 객체 실제 생성
6. Context Refresh 완료 - 웹 서버 기동
7. Application 기동 완료 - 요청 처리 준비 완료

---

### 5. Bean Instance화

#### 본질

Phase 04에서 BeanDefinition 설계도들을 전부 수집해두었다. Phase 05는 그 설계도들을 의존성 그래프 순서대로 꺼내서 **실제 객체를 만들어 BeanFactory에 넣는** 과정이다.

실행 지점은 `AbstractApplicationContext.refresh()` 안의 `finishBeanFactoryInitialization()`.

```kotlin
// AbstractApplicationContext.refresh() (단순화)
fun refresh() {
    prepareBeanFactory(beanFactory)

    // Phase 04-B: BeanDefinition 전부 수집
    invokeBeanFactoryPostProcessors(beanFactory)   // ← Phase 04

    // Phase 04-C: BeanPostProcessor 등록 (실행은 여기서)
    registerBeanPostProcessors(beanFactory)

    // Phase 05: 실제 Bean 객체 생성 ← 지금 여기
    finishBeanFactoryInitialization(beanFactory)

    // Phase 06: 웹 서버 기동 등
    finishRefresh()
}
```

#### 의존성 그래프(DAG)와 위상 정렬

`finishBeanFactoryInitialization()` 내부에서 Spring은 BeanDefinition들을 보고 의존성 그래프(DAG, Directed Acyclic Graph)를 구성한다.

- `@Autowired`, 생성자 파라미터 등에 기록된 의존성 메타데이터를 읽어 "A가 B를 필요로 한다"는 관계를 파악
- 위상 정렬(topological sort)로 **의존 대상이 먼저 초기화**되도록 순서를 결정
- `@DependsOn`으로 명시하지 않으면 Spring이 그래프를 보고 결정한다

```
// 예: OrderService → OrderRepository → DataSource
// 초기화 순서: DataSource → OrderRepository → OrderService
```

`@DependsOn`을 명시하지 않고 직접 `@Autowired`로 연결되지 않은 두 Bean은 의존 관계가 없으므로 초기화 순서가 불확정적이다. 순서에 의존하는 로직이 있다면 `@DependsOn`으로 명시해야 한다.

#### Non-Lazy 싱글톤 빈 순서대로 초기화

- `@Lazy`가 붙은 Bean은 이 시점에 만들지 않는다. 처음 `getBean()` 호출 시 생성.
- 그 외 싱글톤 Bean은 위상 정렬 순서대로 전부 만든다.
- Prototype 스코프 Bean도 이 시점엔 만들지 않는다 (요청 시마다 생성).

#### 각 Bean의 초기화 순서

Bean 하나가 만들어질 때마다 아래 순서를 밟는다.

```
① 생성자 / @Bean 메서드 실행
② 의존성 주입 (@Autowired 등)
③ Aware 콜백 (setApplicationContext 등)
④ @PostConstruct
⑤ BeanPostProcessor
```

```kotlin
// 예: ApplicationContextAware 구현 Bean
@Component
class MyBean(
    private val someService: SomeService  // ② 생성자 주입
) : ApplicationContextAware {

    private lateinit var context: ApplicationContext

    override fun setApplicationContext(ctx: ApplicationContext) {  // ③ Aware 콜백
        this.context = ctx
    }

    @PostConstruct
    fun init() {  // ④ @PostConstruct
        // Bean 준비 완료 직전 초기화 로직
    }
}
// ⑤ 이후 BeanPostProcessor들이 이 Bean을 받아 처리
//    @Transactional이 붙어있다면 AbstractAutoProxyCreator가 AOP 프록시로 감쌈
```

#### BeanPostProcessor의 역할 (Phase 04에서 등록 → 여기서 실행)

Phase 04-C에서 미리 인스턴스화해 등록된 BeanPostProcessor들이 이 시점에 실행된다.

- `AutowiredAnnotationBeanPostProcessor` → `@Autowired` 처리 (② 의존성 주입)
- `CommonAnnotationBeanPostProcessor` → `@PostConstruct`, `@PreDestroy` 처리 (④)
- `AbstractAutoProxyCreator` → `@Transactional`, AOP 대상 Bean을 CGLIB/JDK 프록시로 감싸기 (⑤)

모든 Bean에 대해 `postProcessBeforeInitialization()` → 초기화 → `postProcessAfterInitialization()` 순서로 호출된다.

#### 순환 의존성 감지

Phase 04에서는 BeanDefinition의 의존성 메타데이터만 수집했기 때문에 순환 의존성을 미리 잡지 않는다. 실제 Bean을 만들다가 순환 참조가 발생하면 이 시점(Phase 05)에 `BeanCurrentlyInCreationException`이 발생한다.

```
Bean A를 만들다가 → Bean B가 필요 → B를 만들다가 → A가 필요
→ "A is currently in creation" 예외
```

Spring Boot 2.6+에서는 기본적으로 순환 의존성을 금지한다 (`spring.main.allow-circular-references=false`).

---

### Phase 05 전체 흐름

![Phase 05 전체 흐름](/assets/posts/SpringApplication-초기화-알아보기-5/SpringApplication-초기화-알아보기-5_img_001.png)

---

### 간단 정리

```
Phase 05 완료 후 BeanFactory 상태
  singletonObjects: 수백 개의 싱글톤 Bean 인스턴스
                    (environment, dataSource, service, repository, ...)
  beanDefinitionMap: 설계도 여전히 존재 (Prototype/Lazy Bean 생성에 계속 사용)
  beanPostProcessors: 이미 실행됨 (각 Bean 생성 시마다)
```

다음 Phase 06에서는 `finishRefresh()`가 호출되면서 내장 웹 서버(Tomcat/Netty)가 기동된다.

---

### 참조
- [AbstractApplicationContext 소스 (GitHub)](https://github.com/spring-projects/spring-framework/blob/main/spring-context/src/main/java/org/springframework/context/support/AbstractApplicationContext.java) — refresh() 전체 흐름
- [Bean lifecycle callbacks :: Spring Framework](https://docs.spring.io/spring-framework/reference/core/beans/factory-nature.html) — ①~⑤ 초기화 순서 공식 문서
- [Customizing beans using BeanPostProcessor](https://docs.spring.io/spring-framework/reference/core/beans/factory-extension.html#beans-factory-extension-bpp) — BeanPostProcessor 실행 시점
- [Dependencies and configuration in detail](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-dependson.html) — @DependsOn, 의존성 그래프
