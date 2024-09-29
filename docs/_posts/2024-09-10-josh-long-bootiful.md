---
title:  "Bootiful Spring boot 3.x"
date:   2024-09-10 13:23:17 +0900
categories:
  - java
tags:
  - spring boot
  - java
excerpt: "Bootiful Spring boot 3.x by Josh Long"
toc: true
toc_sticky: true
---
## 개요
사내에 Java Champion이면서, Spring boot contributor중 한명인 Josh Long이 와서 Spring boot 3버전 소개를 진행했는데, 온라인에서 영어로 말하는걸 듣다보니 들은 키워드들과 내용만 정리해본다.

## 내용
### Java 21부터 많이 좋아졌어요
올릴 수 있으면 21로 빨리 올리는게 좋습니다!  
Java 23도 1주뒤에 나옵니다 !!  
(이 글을 쓰는 당시엔 나왔다..)

### DOP(Data Oriented Programming)
class를 통해 클래스 내에 data를 담고, 함수를 정의해서 특정 객체를 정의하는 OOP에서  
데이터와 함수를 구분지어 데이터를 기준으로 설계하려는 패러다임인 DOP로 변화하고 있다.  

데이터를 immutable하게 관리하고 이 데이터에 대해 독립적인 연산을 수행하는 코드를 별개로 작성해라  
이러면 데이터와 로직이 명확하게 분리된다.

### java에서 DOP를 달성하기 위해 나온 문법들
* switch return
* sealed class, permit
* record
* pattern matching

=> 특히 record는 lombok을 되도록 사용하지 않게 도와준다.

### if 보다 switch ?
if보다 switch를 쓰면 compiler가 실수할 여지를 잡아준다.
(모든 경우의 class를 구현하지 않으면 오류 반환)

{% highlight java %}
sealed interface Loan permits SecuredLoan, UnsecuredLoan {
}
record UnsecuredLoan(float interest) implements Loan {
}
final class SecuredLoan implements Loan {
}
class Loans {
	String displayMessageFor(Loan loan) {
		return switch (loan) {
			case SecuredLoan sl -> "good job! "; // case를 모두 정의하지 않으면 오류
			case UnsecuredLoan(var interest) -> "ouch! that " + interest + "% interest rate is going to hurt!";
		};
	}
	@Deprecated
	String badDisplayMessageFor(Loan loan) {
		var message = "";
		if (loan instanceof SecuredLoan) {
			message = "good job! ";
		}
		if (loan instanceof UnsecuredLoan usl) {
			message = "ouch! that " + usl.interest() + "% interest rate is going to hurt!";
		}
		return message;
	}
}
{% endhighlight %}

### class를 되도록 public으로 만들지 마라
* controller, service, repository, model 등
* java는 default 접근제어자가 public이 아니다!
* 한 패키지 내에서 쓰면 어떨까 ?

[MVC 예제 코드](https://github.com/joshlong-attic/bootiful-spring-boot-2024-blog/tree/main/service/src/main/java/com/example/service)

{% highlight java %}
import my.app.controller.*
import my.app.services.*
// ...
{% endhighlight %}  
위와 같은 코드는 controller등에서 여러 service를 import 하거나, service에서 repository, model 등을 import 하는 경우에 많이 볼 수 있다.  
근데 이렇게 코드짜면 package 밖으로 벗어나므로 결국 모든 class가 public이 될 수 밖에 없는 상황이 발생..

### Event driven
이벤트를 발생시키고 그걸 구독하는게 자원 효율적이다.  
spring은 EventPublisher를 제공한다.
* @ApplicationEventPublisher
* @EventListener

@ApplicationModulerListener = @Async + @Transactional + @EventListener  
@Externalized = 외부 이벤트를 수신 가능하게끔 해줌  
```
republish-outstanding-events-on-restart = true
schema-initialization.enabled = true
```
이런 옵션을 주고 로컬에서 별도의 다른 저장소 없이 이벤트를 발행하고, spring boot을 중지하고, 다시 시작했을 때  
spring boot을 중지해서 버려진 event가 유지되고 처리되었음  
이건 시연해주는걸로만 봐서 좀 파봐야할듯..  

### CQRS
* 데이터를 저장하는곳과 가져오는곳을 다르게 하는 구성
* 대부분의 서비스에서는 읽기만 발생하고 쓰기와 수정은 매우 간헐적으로 일어난다.
* 훨씬 효율적

### 문서화
```
ApplicationModules.of()
new Documenter().writeDocumentation()… // puml을 자동으로 만들어준다.
```
위와 같은 식의 코드를 사용했는데 구조도가 파일로 그냥 튀어나왔다..  
근데 이건 intellij에서도 지원해주긴 하는데, 파일로 따로 빼주는건 알아두면 좋을듯  

### Spring AI
ChatClient / ChatClient.builder 를 이용해서 builder 패턴으로  
{% highlight java %}
builder.system()
.defaultSystem()
.defaultAdvisor() // RAG
.build();
{% endhighlight %}  
이런식으로 AI 모델 집어넣고, RAG 설정까지 가능함  

예전에는 컴퓨터가 안좋았는데 이젠 spring boot + AI + event driven 등을 하나의 노트북에서 동시에 할 수 있는 세상이 왔다고 감탄함..  

### java는 환경 친화적 ?
[언어별로 전력 소모량을 정리한 글](https://thenewstack.io/which-programming-languages-use-the-least-electricity/)  
위 링크를 보면 C가 전력 소모량이 1로 1위라고 했을때, java가 1.98로 5위이다.  
VM을 가진 언어중에서는 1위 !!  

josh long은 javascript를 극혐하는듯 보였다.  
type system이 없어 타입 체킹이 안되고 typescript를 해도 결국 바닐라 js로 변환되니.. 기분만 내는거라고 함  

C가 1위, java는 5위, 수치상 C의 2배  
python이 AI에서 많이 쓰임 => 근데 환경친화적이진 않음(27위, 수치상 java의 38배) => java는 좋음(5위) => spring ai !!  

프로그램을 돌리는데 전력이 필요 => 데이터센터 => 발전소 => 터빈 => 나무소모  
java 대신 python 돌리면 38그루의 나무가 죽는다 !  

java를 쓰는것 => spring을 쓰는것 => 환경을 지키는것..  
기적의 논리..

### Project Loom(Virtual thread)
테스트를 위해 [https://httpbin.org/delay/5](https://httpbin.org/delay/5) 요런 사이트를 사용했는데, path variable로 second에 해당하는 값을 넣으면 해당 초 만큼 뒤에 응답을 준다.  
이건 좀 개발에 유용할듯..  

hey라는 부하를 주는 cli tool을 설치해서 사용했다.  

1. 5초간 기다렸다가 응답주는걸 tomcat.threads.max: 10 으로 띄움(전력 소모를 강조함)    
2. hey를 통해 concurrent 20으로 40개의 request 날림

```
spring.threads.virtuals.enabled = true // 설정 하나면 virtual thread 활성이 된다 !
```

테스트 결과  
![virtual_thread_test]({{ "/assets/images/josh-long-bootiful/virtual_thread_test.png" | relative_url }})  
java 21에 virtual thread 사용하면 좋다 !!  

### virtual thread 사용 시 주의점
virtual thread는 작은 작업에만 사용해라.  
CPU를 많이 쓰는 암호화 등의 작업에는 맞지 않다.

virtual thread에서 thread local을 쓰는 경우 좋지만, 항상 초기화 해주어야 하는 부분을 주의해라.  
web은 thread 돌려쓰니 특히 주의  

virtual thread는 수백만개를 띄울 수 있을 정도로 가볍지만  
모두 thread local을 쓴다면 사실상 엄청나게 많은 map을 생성하는 것이기 때문에 garbage가 되지 않도록 조심해라  

Josh long이 언급하진 않았지만 전에 project loom 문서롤 봤었을때  
java 21에서 Scoped Value가 preview 버전으로 있기는 해서..([JEP-446](https://openjdk.org/jeps/446))  
할거없을때 한번 테스트해보면 좋을듯  

### Graal VM
gunnarmorling 이라고 Josh Long 친구인데, 마찬가지로 자바 고수..
java로 10억개의 텍스트(13GB)파일을 얼마나 빠르게 aggregate 할 수 있는지를 테스트했다.(1brc)  
[깃허브 링크](https://github.com/gunnarmorling/1brc)

결과는 32 core AMD EPYC™ 7502P (Zen2), 128 GB RAM를 사용해서 00:01.535로 0.1초가 걸림  
10억 row를 가져오는데 0.1초가 걸렸음, Graal VM을 사용함.  

Graal VM은 대부분의 java 프로젝트에서 사용되는 JIT compile이 아닌, 전체 compile을 해서 build 시간이 매우 길지만 프로그램 수행 시간이 매우 짧다는 장점을 가지고 있음.  
위에서 구현한 모든 예제의 Spring boot web 프로젝트를 만들었을 때 빌드시간이 51초가 걸렸음  
하지만 spring boot 구동 시간이 0.1초임  
Spring AI + DB CRUD + event driven 등을 하는 spring boot 띄우는데 매우 빨랐음, 또한 RAM도 132MB만 사용하였음  

장단점이 매우 명확함  
급하게 핫픽스 나가야 하거나 하는 경우에는 빌드 시간이 진짜 엄청 오래걸릴건데,  
대신 scale-out 하거나 할때는 0.1초만에 스프링이 떠버려서 서비스 안정성면에서는 되게 좋아보였음

### Rubber duck debugging
=> 고무오리를 옆에 두고 그 오리에게 코드를 설명하면서 디버깅하는 방식  
=> 인간은 누군가에게 설명하면서 다른 해결책을 떠올릴 수 있다..

뭔가 철학적이었음  

### 주워들은 키워드들
#### Hexagonal Architecture
[이 문서가 좋은듯 ?](https://engineering.linecorp.com/ko/blog/port-and-adapter-architecture)  
DataSource나 다른 Spring 소스코드내에서 많이 볼 수 있는 패턴을 설명한듯 하다.  
결국 로직의 변환 없이 잘 정의된 interface의 구현체만 바꿔끼면서 원하는 구현체로 교체할 수 있음  
#### reactor/blockhound
non-blockhing thread에서 blocking을 감지해주는 오픈소스
#### java가 kotlin보다 나은 점 ??
non blocking I/O가 java의 syntax가 낫다..;;

{% highlight java %}
// kotlin
suspend fun getCustomer() : Customer {
	val customer = webClient.get().....await()
}
// java
Customer getCustomer() {
	// ...
}
{% endhighlight %}  