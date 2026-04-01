---
title: "SpringApplication 초기화 알아보기 - 2. Environment 준비"
date: 2026-03-28 11:00:00 +0900
categories: [Spring]
tags: [spring-boot, startup, environment, property-source, profile, config-data]
---

> **시리즈: SpringApplication 초기화 알아보기**
> - [1. 초기화](/posts/Spring/SpringApplication-초기화-알아보기-1)
> - **2. Environment 준비** ← 현재 글
> - [3. ApplicationContext 생성](/posts/Spring/SpringApplication-초기화-알아보기-3)
> - [4. Bean Definition 등록](/posts/Spring/SpringApplication-초기화-알아보기-4)

### 전체 개요

1. SpringApplication 초기화 - main() -> SpringApplication.run()
2. **Environment 준비** - profile, 설정 파일 로드.
3. ApplicationContext 생성 - Context Instance 준비
> - [4. Bean Definition 등록](/posts/Spring/SpringApplication-초기화-알아보기-4)
4. Bean Definition 등록 - refresh() 내부에서 "설계도"만 수집, 객체 생성 없음
5. Bean Instance화 - finishBeanFactoryInitialization() - 객체 실제 생성
6. Context Refresh 완료 - 웹 서버 기동
7. Application 기동 완료 - 요청 처리 준비 완료

---

### 2. Environment 준비 - profile, 설정 파일 로드.
Event: ApplicationEnvironmentPreparedEvent
Environment는 "앱이 실행될때 필요한 모든 설정값"
- ex) application.yaml, 환경변수, JVM 프로퍼티, 커맨드라인 인자 등등.

#### 이해를 위한 역사..
Spring 3.0 이전에는 설정값을 읽는 방법이 제각각
```
Properties props = new Properties()
props.load(new FileInputStream("app.properties")) // 파일에서 직접

System.getenv("SERVER_PORT") // 환경변수
System.getProperty("jvm.option") // JVM 옵션 별도
```
환경마다 다른 설정을 쓰려면 코드를 직접 바꾸거나, 빌드 프로파일을 달리해야했다.
Spring 3.1에서 Environment와 Profile개념이 도입되어 "어디서 왔든 하나로 읽는다"는 컨셉이 생겼다.

#### 대략 알아보기
```
// SpringApplication.run() 내부
fun run (vararg args: String): ConfigurableApplicationContext {
    listeners.starting() // Phase01 완료
    // Phase 02 시작
    val environment = prepareEnvironment(listeners, DefaultApplicationArguments(args))

    // ...
}

private fun prepareEnvironment(
    listeners: SpringApplicationRunListeners,
    applicationArguments: ApplicationArguments
): ConfigurableEnvironment {

    // 1. 빈 Environment 생성 (타입에 따라 다른 구현체)
    val environment = createEnvironment()

    // 2. 기본 PropertySource들 붙이기
    configureEnvironment(environment, applicationArguments.sourceArgs)

    // 3. ConfigData 로드 (application.yml 등)
    ConfigDataEnvironmentPostProcessor.applyTo(environment)

    // 4. 리스너에게 알림
    listeners.environmentPrepared(bootstrapContext, environment)
    // → ApplicationEnvironmentPreparedEvent 발행

    return environment
}
```

#### PropertySource - 설정값의 출처
![alt text](/assets/posts/SpringApplication-초기화-알아보기-2/SpringApplication-초기화-알아보기-2_img_001.png)
```
// Environment 안의 구조 (개념적으로)
// 같은 키가 여러곳에 있으면 위에 있는 것(인덱스가 낮은 것)이 이김.
Environment
  └── MutablePropertySources (우선순위 순서)
        [0] CommandLinePropertySource       // --server.port=9090
        [1] SystemPropertiesPropertySource  // -Dserver.port=9090
        [2] SystemEnvironmentPropertySource // SERVER_PORT=9090
        [3] ConfigDataPropertySource        // application.yml
        [4] DefaultPropertiesPropertySource // SpringApplication.setDefaultProperties()
```

#### 그 중에서도 ConfigData 로드와 관련해서 - application.yaml이 읽히는 과정
```
// 이 파일들을 Spring이 자동으로 찾아서 읽음
// 우선순위: 아래로 갈수록 높음 (나중에 로드된 게 덮어씀)
// classpath = "JAR 안 또는 빌드 결과물 안"
// 현재 디렉토리 = "JAR 파일이 실행되는 위치의 파일 시스템"

// 1. classpath (JAR 안)
//    src/main/resources/application.yml

// 2. classpath:/config/
//    src/main/resources/config/application.yml

// 3. 현재 디렉토리
//    ./application.yml

// 4. 현재 디렉토리 /config/
//    ./config/application.yml

/home/ubuntu/app/                        ← 여기서 java -jar 실행
my-app.jar
└── BOOT-INF/
    └── classes/
        └── application.yml              ← classpath:/
        └── config/
            └── application.yml         ← classpath:/config/
├── application.yml                      ← 현재 디렉토리
└── config/
    └── application.yml                  ← 현재 디렉토리 config/
```
외부 config/application.yaml 만 변경해서 JAR을 다시 빌드하지 않고 설정만 변경 등으로 운영한다던가..

#### spring.config.import
- Phase02에서 처리되지만, application.yml이 로드 된 이후에 실행.
- "기본 설정을 읽고나서, 거기서 추가로 읽어올 곳을 지정"
    - file: # 파일에서 추가로드
    - classpath: # classpath에서 추가로드.
    - aws-secretsmanager # aws secrets manager
    - configserver # spring cloud config
    - "optional:" 접두어가 있으면 해당 소스가 없어도 넘어감.
```
application.yml 로드
    ↓
spring.config.import 항목 발견
    ↓
import 목록을 순서대로 로드
    ↓
import된 값들이 PropertySource 체인에 추가됨
(application.yml보다 높은 우선순위로 들어감)
```
- import된 파일은 application.yml보다 높은 우선순위로 들어간다.
- 단일 문서 내에서 `spring.config.import`의 위치는 결과에 영향을 주지 않는다. YAML은 파일 전체를 하나의 문서로 파싱하기 때문.
```
# application.yml — 단일 문서
spring:
  redis:
    port: 6380
  config:
    import: "classpath:redis-defaults.yml"   # port: 6379 정의됨

# 결과: port = 6379  ← import된 게 이김!
# import는 "선언한 문서 바로 아래에 삽입" 되므로 선언 문서보다 우선순위가 높다.
# 같은 문서 안에서 import 선언 위치를 바꿔도 결과는 동일 (문서 단위로 처리됨)
```

##### 멀티 도큐먼트 YAML (`---` 구분) 에서는?
- `---`로 구분하면 각각 독립된 "논리적 문서"로 처리되고, 위에서 아래 순서로 처리된다. 아래 문서가 이김.
- import는 "선언한 문서 바로 아래에 삽입"이므로, 그 아래에 있는 `---` 문서가 import 값을 다시 덮을 수 있다.
```
# application.yml — 멀티 도큐먼트
spring:
  config:
    import: "classpath:redis-defaults.yml"   # port: 6379
---
spring:
  redis:
    port: 6380   # ← 이 문서가 import보다 아래이므로 이게 이김! 결과: 6380
```

##### 멀티 모듈 (여러 JAR)에서의 주의점
- 여러 JAR이 각각 `application.yml`을 가지면 **first-match-wins** — classpath 순서에 의존하므로 순서가 보장되지 않는다.
- Spring Boot 팀도 이 문제를 인지하고 있으나 아직 설계 검토 중 ([spring-boot#24688](https://github.com/spring-projects/spring-boot/issues/24688))
- 권장: 모듈별 고유 config 파일명 사용 (ex: `module-a-defaults.yml`) + 메인 앱에서 `spring.config.import`로 명시적 로드

### EnvironmentPostProcessor
- ApplicationEnvironmentPreparedEvent가 발행되면, `EnvironmentPostProcessorApplicationListener`가 이 이벤트를 받아 등록된 EnvironmentPostProcessor들을 실행한다. 즉 이벤트 처리 과정 중에 실행되는 훅이다.
- Environment가 준비됐는데, Bean 만들기 전에 PropertySource를 코드로 추가/수정하고 싶은 경우에 사용한다.
- ApplicationContext가 생기기전에 동작하는 것으로 (Bean이 아님), spring.factories에 등록필요. 아래같은 형식
```
# META-INF/spring.factories
org.springframework.boot.env.EnvironmentPostProcessor=\
  com.example.SecretsManagerEnvironmentPostProcessor
```
spring.config.import
→ 선언적 (yml 파일에 작성)
→ 파일, URL, 외부 서비스에서 설정값 로드
→ Spring이 지원하는 소스만 사용 가능

EnvironmentPostProcessor
→ 코드로 직접 작성
→ 어떤 소스든 가능 (커스텀 암호화 해제, 동적 계산 등)
→ 더 유연하지만 spring.factories 등록 필요
→ AWS Secrets가 spring.config.import로 지원되기 전에는 이 방식으로 구현했음

#### 프로파일 설정
```
// 프로파일 활성화 방법들
// 1. application.yml
spring.profiles.active: prod

// 2. 환경변수
// SPRING_PROFILES_ACTIVE=prod

// 3. 커맨드라인
// java -jar app.jar --spring.profiles.active=prod

// 4. 코드에서
fun main(args: Array<String>) {
    val app = SpringApplication(MyApp::class.java)
    app.setAdditionalProfiles("prod")
    app.run(*args)
}
```
##### Environment와 PropertySource, yaml..
Environment = PropertySource들의 정렬된 목록
PropertySource = 설정값의 출처 하나 (Yaml 파일 하나, 환경변수 등.)
- environment.getProperty("redis.host")를 호출하면, 내부적으로 PropertySource목록을 순서대로 뒤진다.
![alt text](/assets/posts/SpringApplication-초기화-알아보기-2/SpringApplication-초기화-알아보기-2_img_002.png)
![alt text](/assets/posts/SpringApplication-초기화-알아보기-2/SpringApplication-초기화-알아보기-2_img_003.png)

```
# application.yml
spring:
  redis:
    host: ${redis.host}        # → 문자열 "${redis.host}" 그대로 저장
    port: 6379                 # → 문자열 "6379" 저장

// PropertySource 내부 (개념적으로)
MapPropertySource("application.yml", mapOf(
    "spring.redis.host" to "\${redis.host}",   // placeholder 문자열 그대로
    "spring.redis.port" to "6379"              // 실제 값
))


// Phase 05: Bean 생성 시
@ConfigurationProperties(prefix = "spring.redis")
data class RedisProperties(val host: String)

// Spring이 하는 일:
// 1. PropertySource 체인에서 "spring.redis.host" 찾음
// 2. 값이 "${redis.host}" 임을 발견
// 3. 다시 PropertySource 체인에서 "redis.host" 찾음
// 4. AWS Secrets PropertySource에서 "redis.host" = "actual-redis.com" 발견
// 5. host = "actual-redis.com" 바인딩 완료

```




#### 설정값을 사용해보자.
Phase02에서 `Environment`가 준비가 되고나면 이후 phase에서 bean들이 값을 읽어간다.
- @Value
- @ConfigurationProperties(prefix = xxx)
로 보통 사용하듯이 사용하면됨.

#### 정리
Phase 02 끝 시점의 상태
```
Environment 객체가 완성됨
  └── PropertySources 우선순위 체인 구성 완료
  └── 활성 Profile 결정됨
  └── application.yml / application-{profile}.yml 로드 완료

아직 없는 것:
  ApplicationContext — Phase 03에서 생성
  Bean — Phase 04~05에서 생성

  → @Value, @ConfigurationProperties 바인딩도 아직 안 됨
    Bean이 만들어질 때 비로소 Environment에서 값을 읽어감

이후에 ApplicationEnvironmentPreparedEvent가 발행되고,
Phase01에서 등록한 인프라 리스너들(boot 리스너 Phase01참조)이 이벤트를 받아 처리합니다.

ex) LoggingApplicationListener가 이 시점에 logging.level을 가져가서 로그 레벨을 세팅합니다.
```

##### Phase02에서 더알아보기.
- 그럼 Spring Cloud Config는 어떻게 동작하는건데요..?
    - @RefreshScope가 동작하는 방식 등을 알아보자.
    - 고민해보면 PropertySource값 중 우선순위가 높을것이고, 그걸 바꿔서 뭔가 새롭게 할 수 있지않을까? 하고 추측은 가능하다.
    - pod간 동기화는 고민의 덤.
    - 그리고 모두 설정을 refresh할 수 있다고 좋은건 아니다.. ex) 운영 중 db설정이 바뀐다던가..가 되버리면.. 나중에 더 알아보면 좋다.

---

### 참조
- [Spring Boot Externalized Configuration (3.4)](https://docs.spring.io/spring-boot/3.4/reference/features/external-config.html) — PropertySource 우선순위, Config Data 로딩 순서
- [Spring Boot Config Data Import (3.4)](https://docs.spring.io/spring-boot/3.4/reference/features/external-config.html#features.external-config.files.importing) — spring.config.import 동작 방식
- [Config File Processing in Spring Boot 2.4 (공식 블로그)](https://spring.io/blog/2020/08/14/config-file-processing-in-spring-boot-2-4/) — ConfigData 도입 배경, 멀티 도큐먼트 YAML 처리 변경사항
- [EnvironmentPostProcessor API](https://docs.spring.io/spring-boot/api/org/springframework/boot/env/EnvironmentPostProcessor.html)
- [spring-boot#24688](https://github.com/spring-projects/spring-boot/issues/24688) — 멀티 모듈 JAR에서 application.properties 기여 문제 (pending-design-work)
