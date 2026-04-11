---
title: "SpringApplication 초기화 알아보기 - 6. Context Refresh 완료 & Application 기동"
date: 2026-04-11 12:00:00 +0900
categories: [Spring]
tags: [spring-boot, startup, lifecycle, graceful-shutdown, runner, kubernetes, readiness-probe]
---

> **시리즈: SpringApplication 초기화 알아보기**
> - [1. 초기화](/posts/SpringApplication-초기화-알아보기-1)
> - [2. Environment 준비](/posts/SpringApplication-초기화-알아보기-2)
> - [3. ApplicationContext 생성](/posts/SpringApplication-초기화-알아보기-3)
> - [4. Bean Definition 등록](/posts/SpringApplication-초기화-알아보기-4)
> - [5. Bean Instance화](/posts/SpringApplication-초기화-알아보기-5)
> - **6. Context Refresh 완료 & Application 기동** ← 현재 글
> - [총정리](/posts/SpringApplication-초기화-알아보기-총정리)

### 전체 개요

1. SpringApplication 초기화 - main() -> SpringApplication.run()
2. Environment 준비 - profile, 설정 파일 로드.
3. ApplicationContext 생성 - Context Instance 준비
4. Bean Definition 등록 - refresh() 내부에서 "설계도"만 수집, 객체 생성 없음
5. Bean Instance화 - finishBeanFactoryInitialization() - 객체 실제 생성
6. **Context Refresh 완료** - 웹 서버 기동
7. **Application 기동 완료** - 요청 처리 준비 완료

Phase 06과 07은 짧지만 **실제 서비스 운영과 직접 연결**되는 구간이라 함께 다룬다.

---

### 6. Context Refresh 완료

`refresh()`의 마지막 단계다.

```kotlin
// AbstractApplicationContext.refresh()의 가장 마지막
fun refresh() {
    // ... Phase 04/05 ...

    // Phase 06
    finishRefresh()
}

fun finishRefresh() {
    // 1. Lifecycle Bean들 시작
    getLifecycleProcessor().onRefresh()

    // 2. 웹 서버 기동 (WebApplicationType: SERVLET/REACTIVE일 때)
    createWebServer()  // 내장 Tomcat 또는 Netty 시작
    // → 이 시점부터 HTTP 요청 수신 가능

    // 3. ContextRefreshedEvent 발행
    publish(ContextRefreshedEvent(this))
}
```

#### Lifecycle Bean

`Lifecycle` 인터페이스는 "Spring Context와 생명주기를 함께하고 싶은 Bean"이 구현한다.

```kotlin
// 예: Kafka Consumer, Scheduler 등
@Component
class KafkaConsumerManager : SmartLifecycle {
    override fun start() {
        // Context Refresh 완료 시, 자동 시작
        consumer.subscribe(listOf("some-topic"))
    }

    override fun stop() {
        // Context 종료 시 자동 정지
        consumer.close()
    }

    override fun isRunning(): Boolean = consumer.isActive
}
```

---

### 7. Application 기동 완료

`refresh()` 이후 `SpringApplication.run()`으로 돌아와서 마무리한다.

```kotlin
// SpringApplication.run()의 마지막 부분
fun run(): ConfigurableApplicationContext {
    // ... refresh() 완료 — Phase 06까지 ...

    // Phase 07-A
    listeners.started(context)
    // → ApplicationStartedEvent 발행
    // → "Bean 다 만들어졌고, 서버도 떴는데, Runner 실행 전"

    // Phase 07-B: Runner 실행
    callRunners(context, applicationArguments)

    // Phase 07-C
    listeners.ready(context)
    // → ApplicationReadyEvent 발행
    // → 진짜 모든 준비 완료

    return context
}
```

#### ApplicationRunner vs CommandLineRunner

Runner는 두 가지가 있고, 기능은 같다. 차이는 인자 타입뿐.

```kotlin
// 1. ApplicationRunner — 파싱된 인자
@Component
class DataInitRunner : ApplicationRunner {
    override fun run(args: ApplicationArguments) {
        // args.getOptionValues("env") 같은 형태로 접근
        initializeData()
    }
}

// 2. CommandLineRunner — 원시 문자열 배열
@Component
class MigrationRunner : CommandLineRunner {
    override fun run(vararg args: String) {
        // args = ["--server.port=8080", "--spring.profiles.active=prod"]
        runMigration()
    }
}
```

Runner는 보통 아래와 같은 용도로 사용된다:
- 기동 시 데이터 초기화
- 외부 연결 상태 체크
- 배치성 작업 (WebApplicationType.NONE 타입 앱)
- 캐시 웜업

---

### K8s 연계: 기동과 종료

K8s 환경에서 Phase 06/07은 Pod의 생사와 직결된다.

#### Liveness & Readiness Probe

```yaml
# application.yml
management:
  endpoint:
    health:
      probes:
        enabled: true  # /actuator/health/liveness, /actuator/health/readiness 활성화

# K8s deployment.yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness   # 프로세스 살아있나
readinessProbe:
  httpGet:
    path: /actuator/health/readiness  # 트래픽 받을 준비 됐나
    # ApplicationReadyEvent 발행 전까지 DOWN 상태
```

Runner에서 웜업 등 오래 걸리는 작업을 수행하면 `ApplicationReadyEvent` 발행이 늦어지고, Readiness Probe가 실패할 수 있다.

#### SmartLifecycle의 Phase와 기동/종료 순서

```
start() 순서: phase 낮은 것 먼저
  phase 0:   DB 커넥션 풀 시작
  phase 100: Kafka Consumer 시작
  phase MAX: 웹 서버 시작 (가장 마지막) ← 모든 게 준비된 후 트래픽 받음

stop() 순서: phase 높은 것 먼저 (start의 역순)
  phase MAX: 웹 서버 중단 (새 요청 거부) ← 가장 먼저
  phase 100: Kafka Consumer 중단
  phase 0:   DB 커넥션 풀 종료 (가장 마지막)
```

웹 서버가 가장 마지막에 시작되고 가장 먼저 종료되는 설계다.

#### Graceful Shutdown 상세 흐름

```
SIGTERM 수신 (K8s Pod 종료 신호)
        ↓
새 요청 수신 거부 (Readiness → DOWN)
        ↓
진행 중인 요청 완료 대기
  (spring.lifecycle.timeout-per-shutdown-phase: 30s)
        ↓
SmartLifecycle.stop() 역순 호출
  웹 서버 stop()
  Kafka Consumer stop()
  DB 커넥션 풀 stop()
        ↓
@PreDestroy 호출
        ↓
singletonObjects 정리
        ↓
프로세스 종료
```

```kotlin
// Graceful Shutdown 실무 패턴
@Component
class PaymentKafkaConsumer : SmartLifecycle {

    private var running = false

    override fun start() {
        consumer.subscribe(listOf("payment-events"))
        running = true
    }

    override fun stop(callback: Runnable) {
        // 처리 중인 메시지 완료 대기
        consumer.wakeup()
        processingLatch.await(30, TimeUnit.SECONDS)
        consumer.close()
        running = false
        callback.run()  // 완료 신호 — 이걸 호출해야 다음 단계로 넘어감
    }

    override fun getPhase(): Int = 100  // 웹 서버(MAX)보다 나중에 stop됨
    override fun isRunning(): Boolean = running
}
```

K8s의 `terminationGracePeriodSeconds`(기본 30초)와 Spring의 `timeout-per-shutdown-phase` 간의 시간 차이를 잘 고려해야 한다. App의 처리가 끝나기 전에 K8s가 SIGKILL로 강제 종료할 수 있기 때문.

---

### 요약

![Phase 06 + 07 전체 흐름](/assets/posts/SpringApplication-초기화-알아보기-6/SpringApplication-초기화-알아보기-6_img_001.png)

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

---

### 참조
- [SmartLifecycle Javadoc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/SmartLifecycle.html) — phase 기반 기동/종료 순서
- [Graceful Shutdown :: Spring Boot](https://docs.spring.io/spring-boot/reference/web/graceful-shutdown.html) — Graceful Shutdown 설정
- [Kubernetes probes with Spring Boot Actuator](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html#actuator.endpoints.kubernetes-probes) — Liveness/Readiness Probe
- [Application Events and Listeners :: Spring Boot](https://docs.spring.io/spring-boot/reference/features/spring-application.html#features.spring-application.application-events-and-listeners) — ApplicationStartedEvent, ApplicationReadyEvent
