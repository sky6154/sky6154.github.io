---
title:  "Spring boot 3버전업 하면서 겪은점"
date:   2024-06-20 14:13:52 +0900
categories:
  - spring
tags:
  - spring boot
excerpt: "What I experienced while upgrading to Spring Boot 3"
toc: true
toc_sticky: true
---
## 개요
spring boot 2 => 3으로 버전업 하면서 생긴 이슈들을 정리해본다.

## 문제
spring boot 2에는 몇가지 문제가 있었다.  
그중 하나는.. CVE-2016-1000027

![spring_doc]({{ "/assets/images/spring-boot-3-version-up/vulnerability.png" | relative_url }})

위 취약점이 CRITICAL로 매번 잡힌다.. 근데 저건 spring 6버전부터 수정되었다.  
고치려면? spring boot 3으로 올리고 JDK 17로 올려야 한다.

해당 취약점은 request 받은 값을 이용해서 Serializable 객체로 직렬화하여 사용하는 경우 sql injection처럼 직렬화 과정에서 오염된 request를 통해 의도하지 않은 무언가를 원격으로 실행 가능하게 되는 취약점이다.  
일단 보통의 spring boot web은 별다른 설정을 건들지 않는 경우 HttpMessageConverter인 objectMapper(jackson 라이브러리)를 이용하여 json으로 값을 주고받는다.  
따라서 request의 값을 받아서 그걸 객체로 직렬화 하는 경우는.. 어지간해서는 거의 없는데, 그렇게 사용할 경우 엄청 취약할 수 있으므로 CRITICAL로 잡히는것 갈다.   

일단 해당 문제 이슈업은 2016년도에 되었으나 아직까지 고치지 못했는데..  
해당 이슈를 고치려면 web을 근본부터 다 뜯어고쳐야 한다고 한다.  

[링크](https://github.com/spring-projects/spring-framework/issues/24434) 참고  
몇몇 논쟁이 있으나, spring 개발자가 지금 당장 못고치고 spring 6버전부터 다 갈아엎고 고칠거임 이라고 못박아놔서 spring boot 2 이하 버전을 쓴다면  
해당 취약점은 피할 수 없어보인다..  

## 그래서 뭘 바꿨나
위의 문제만으로 버전업을 진행한건 아니고 계속해서 spring boot 3 얘기가 나오고는 있었다.  
다만 여러 보안심사를 앞두고 저거 계속 나올건데 known issue로 둘거냐.. 하다가 중간에 시간이 남게되어 진행했다.  

### gradle 버전업
spring boot 3부터는 gradle 7.5버전 이상이 요구된다.  
[공식문서](https://docs.spring.io/spring-boot/system-requirements.html#getting-started.system-requirements) 참고

### JDK 버전업
java 17부터 지원된다.

### lombok 버전업
lombok에서 java 17은 1.8.22버전 이상부터 지원한다.  
[lombok 문서](https://projectlombok.org/changelog) 참고

### querydsl 수정
```
api "com.querydsl:querydsl-jpa:${dependencyManagement.importedProperties['querydsl.version']}:jakarta"
annotationProcessor "com.querydsl:querydsl-apt:${dependencyManagement.importedProperties['querydsl.version']}:jakarta"
annotationProcessor "jakarta.annotation:jakarta.annotation-api"
annotationProcessor "jakarta.persistence:jakarta.persistence-api"
```
(기존 querydsl이 Entity를 찾 을 때 javax 패키지에서 찾는걸 jakarta로 변경)

### javax => jakarta
`import javax.*` -> `import jakarta.*`  
Java EE의 상표권 문제로 Jakarta EE로 변경하면서 패키지가 변경 되었다.  
javax.sql.DataSource 처럼 spring boot가 아닌 Java쪽의 javax 패키지를 쓰는 경우가 있어 무작정 다 고칠 경우 오류를 보게 된다.  
(한방에 바꾸고 빌드 오류뜨는거 하나씩 잡는게 더 빠를수도..?)

### deprecated 된 코드 변경 
[spring 2.7.1 문서](https://docs.spring.io/spring-boot/docs/2.7.11/api/deprecated-list.html) 참고  
JPA 설정중에 `org.springframework.boot.orm.jpa.hibernate.SpringPhysicalNamingStrategy` 이게 사라져서  
`org.hibernate.boot.model.naming.CamelCaseToUnderscoresNamingStrategy` 로 바꿨다.

### hibernate 버그
####  Hibernate 6.4.1 version 아래로 쓰고 있다면 올려야 한다.  
where in 절에 numeric/decimal을 사용하는 코드에서 오류가 난다..  
[이슈](https://hibernate.atlassian.net/browse/HHH-17492) 참고

#### @CreationTimestamp, @UpdateTimestamp
`@CreationTimestamp`, `@UpdateTimestamp` 사용시 `source = SourceType.DB` 옵션을 넣어주었다.  
hibernate 6버전 부터 기본값으로 `SourceType.VM` 이 되었는데, host의 timezone을 바라보게 되어 개발 PC에서 KST로 들어가는 현상이 있었다.  
DB로 하게 된다면 DB의 timestamp를 따라간다.  

다만 spring에서 제공하는 `@CreatedDate`, `@LastModifiedDate`는 이슈가 없었다.  

#### query bind 값 노출하기
개발할때만 해당 기능이 제법 편한데,  
`logging.level` 을 `org.hibernate.type.descriptor.sql.BasicBinder` 를 `TRACE`로 하면 ?로 출력되는 쿼리에 bind 되는 값이 추가로 노출했었다.   
spring boot 3부터는 패키지가 변경되었다.  
`org.hibernate.orm.jdbc.bind=TRACE` 로 하면 된다.  

다만 이 기능은 서비스에서는 끄는걸 권장한다.  
[메타 과징금 기사](https://news.hada.io/topic?id=16987)
얼마전에 비밀번호를 평문으로 로그에 찍었다가(일종의 저장) 1300억의 과징금을..  
관련해서 request나 response로 나가는 로그도 모두 끄는게 보안상 맞다.  
예전에 datadog trace에 post 요청 payload까지 모두 남기는 기능이 있는데, 이를 활성화하고 사용하는 사람이 있어 껐던 기억이 있다.  
물론 민감한 정보를 다루는 서비스는 아니어서 개발자 원인파악 편의를 위해 남기는 경우가 있긴 했는데, 이런 한두개 예외를 두다가..

#### swagger
```
implementation "org.springdoc:springdoc-openapi-starter-webmvc-ui:2.3.0"
```
로 바꾸면 된다.  
위 버전부터 javadoc을 swagger가 지원하는데, javadoc을 자주 써왔다면 별다른 설정 없이 swagger에 설명 추가가 가능하다.  

```
implementation "com.github.therapi:therapi-runtime-javadoc::0.15.0"
annotationProcessor "com.github.therapi:therapi-runtime-javadoc-scribe:0.15.0"
```
위에 추가하면 javadoc을 swagger가 알아서 파싱한다.  

`org.springdoc.core.GroupedOpenApi` 패키지가 다음으로 교체되었다.  
`org.springdoc.core.models.GroupedOpenApi`


#### spring batch
`org.springframework.batch.core.configuration.annotation.DefaultBatchConfigurer` 패키지가 spring boot 3에서 사라지고 다음으로 교체되었다..  
`org.springframework.batch.core.configuration.support.DefaultBatchConfiguration`

1. setDatasource 관련 메소드가 사라짐 
   * 내부 로직을 확인하면 자동으로 DataSource bean을 가져가게끔 되어있음 
   * 이름이 반드시 dataSource인 bean 이어야 함 
     * 아니라면 getDatasource를 override 하면 됨 
   * PlatformTransactionManager 가 반드시 transanctionManager 라는 bean으로 정의되어 있어야 함 
     * 아니라면 getTransactionManager를 override 해야함
2. @EnableBatchProcessing 혹은 DefaultBatchConfituration 설정 시 @ConditionalOnMissingBean에 의해 JobLauncherApplicationRunner가 실행되지 않음
   * JobLauncherApplicationRunner는 argument로 받은 job name을 실행시켜주는 class
   * `@EnableBatchProcessing`이 main에 있으면 `job.name`을 args로 안넘겨도 전체 job이 실행됨
   * --spring.batch.job.names={jobName} 를 넣으면 이미 job이 실행되었다고 오류가 발생함
   * `@EnableBatchProcessing` 빼던지, jobName을 주던지 해야함
3. Spring batch에서 쓰이는 Table 생성이 반드시 필요함
   * 테이블 사용으로 jobParameter로 중복이 안되는 값을 argument로 넣어 실행해야 함
     * Spring batch에서 테이블 select 후 똑같은 batch가 돌았는지 중복체크 목적
4. JobBuilderFactory, StepBuilderFactory 및 tasklet(Tasklet tasklet) 메소드가 deprecated 되고 삭제 예정

{% highlight java %}
@Bean(JOB_NAME)
public Job job() {
    return jobBuilderFactory.get(JOB_NAME)
        .start(step())
        .build();
}
@Bean(STEP_NAME)
public Step step() {
    return stepBuilderFactory.get(STEP_NAME)
        .tasklet((contribution, chunkContext) -> {
            JobParameters jobParameters = getJobParameters(chunkContext);
            // do something
            return RepeatStatus.FINISHED;
        })
        .build();
}
{% endhighlight %}
위 코드를

{% highlight java %}
@Bean(JOB_NAME)
public Job job(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
    return new JobBuilder(JOB_NAME, jobRepository)
        .start(step(jobRepository, transactionManager))
        .build();
}
@Bean(STEP_NAME)
public Step step(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
    return new StepBuilder(STEP_NAME, jobRepository)
        .tasklet((contribution, chunkContext) -> {
            JobParameters jobParameters = getJobParameters(chunkContext);
            // do something
            return RepeatStatus.FINISHED;
        }, transactionManager)
        .build();
}
{% endhighlight %}

이런식으로 변경 해주어야 함

5. `@EnableBatchProcessing` 를 SpringApplication내에서 여러번 설정하면 bean 초기화 오류가 발생함
   * SpringApplication 쪽에 한번만 넣어주는게 좋을 듯 ?

#### base64 encoder의 depreacted
`Base64Utils.encodeToString` 이 deprecated 되었다.  
`Base64.getEncoder().encodeToString` 로 변경하면 된다.

#### spring cloud gateway
4버전대가 spring boot 3, spring 6과 호환된다.  
버전업 해야 한다..

1. custom exception들이 `HttpStatus`를 갖는게 아닌 상위 인터페이스의 `HttpStatusCode`를 갖게끔 변경되었다.
2. `ResponseStatusException`에서 `StatusCode` 함수가 사라지고 `getStatusCode` 로 변경되었다.
3. WebClient의 exchangeToMono 시 response에서 status code를 파싱할 경우
    ```java
      final int beginStatusCode = response.rawStatusCode() / 100; // 변경전
      final int beginStatusCode = response.statusCode().value() / 100; // 변경후
    ```

#### pebble 버전업
```
// 변경전
implementation "io.pebbletemplates:pebble:3.2.1"
implementation "io.pebbletemplates:pebble-spring5:3.2.1"

// 변경후
implementation "io.pebbletemplates:pebble:3.2.2"
implementation "io.pebbletemplates:pebble-spring6:3.2.2"
```

#### Spring boot starter redis 설정
RedisTemplate을 설정하는 과정에서 Jackson2JsonRedisSerializer를 사용할 경우 setObjectMapper가 deprecated 되었다.

```java
// 변경전
Jackson2JsonRedisSerializer<Object> serializer = new Jackson2JsonRedisSerializer<>(Object.class);
serializer.setObjectMapper(objectMapper());

// 변경후
Jackson2JsonRedisSerializer<Object> serializer = new Jackson2JsonRedisSerializer<>(objectMapper(), Object.class);
```

또한 자동 설정을 사용하는 경우.. application yaml 설정을 바꾸어야 한다.  
`spring.redis.*` -> `spring.data.redis.*`

#### spring-boot-kafka
spring boot 3 부터 kafka에 대한 모니터링(micrometer)을 metric으로 지원함
(default: false)

필수는 아니나 적용 검토 필요해볼 수 있을듯..
```
spring:
  kafka:
    listener:
      observation-enabled: true
    template:
      observation-enabled: true
```

spring boot 3(spring 6) 지원하는 라이브러리로 버전업 필요

```
implementation 'org.springframework.kafka:spring-kafka:3.1.4'
```

#### jwt 관련 라이브러리
jwt관련 라이브러리인 jjwt 를 사용중이라면 아래 라이브러리 추가 필요
```
implementation 'javax.xml.bind:jaxb-api:2.3.1'
```

#### endpoint 뒤 / 유무
spring boot 2 이하에는 `/users/` 와 `/users` 를 동일하게 판별했다.  
근데 보안상 이슈로 둘을 다르게 판단한다.  
[문서](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide#spring-mvc-and-webflux-url-matching-changes) 참고  

둘을 같게 처리하고 싶으면..
{% highlight java %}
@Configuration
public class MvcConfiguration implements WebMvcConfigurer {
    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        configurer.setUseTrailingSlashMatch(true);
    }
}
{% endhighlight %}

해도 되는데, 위 옵션 자체가 deprecated 되어서 정 하고싶다면 filter등으로 request 전처리를 해야 할듯하다..

## 느낀점
거의 모든 프로젝트의 설정등의 구성을 동일하게 해서 사용하고 있었기 때문에 특정 이슈에 대해 한번만 고생하면 나머지는 동일하게 수정할 수 있었기 때문에 그나마 수월하게 진행할 수 있었다..  
모든 개발자가 각자의 설정가지고 사용했다면 버전업 하는데 엄청 힘들었을것 같다.  
이래서 대부분의 조직들이 버전 통일해서 쓰다가 특정 버전에서 검증되면 다같이 올리는건지 싶었다.   

버전업을 안하게 되면 결국 기술부채라서 일정 주기마다 버전 따라잡는게 중요할거같긴 한데, 시간과 비용이..  
(물론 서비스 안정성이 제일 중요하지만)  

요즘 java 버전업 속도가 엄청 빠르고, 관련해서 나오는 기술들도 한순간 놓쳐버리면 버전업할때 엄청 많은 버전을 점프뛰어야 따라잡을거 같은데  
버전업 하면서 기능 검증이며, 코드도 엄청 바뀔수도 있고, 서비스 안정성 챙기고, 사람 설득하고, 그러면서 하던일 계속하는거까지.. 쉽지 않을듯 하다..  