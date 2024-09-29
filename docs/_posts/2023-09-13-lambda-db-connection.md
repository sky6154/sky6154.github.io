---
title:  "람다에서 DB connection pool 유의할점"
date:   2023-09-13 11:18:22 +0900
categories:
  - serverless
tags:
  - lambda
  - RDS
excerpt: "Precautions when using DB connection in Lambda"
toc: true
toc_sticky: true
---
## 개요
대량메일 전송 시스템을 lambda를 이용해서 개발했는데, 메일 테스트 중 다음과 같은 이슈가 생겼다.   
스트레스 테스트중이었고 50만건가량을 메일 전송중이었는데, RDS에서 알림이 발생했고, connection이 부족하다는 알람이었다.  
개발환경은 instance가 작아서 max_connection이 1000개였는데, 1000개를 다 쓰고있었다.

lambda가 돌고 종료되면 DB connection이 바로 release될줄 알았는데 아니었다.    
혹시 몰라서 500건만 실행 해봤는데 역시나 마찬가지였다..

## 해결
### callbackWaitsForEmptyEventLoop
lambda는 javascript의 event loop이 끝날때까지 대기한다.  

```
context.callbackWaitsForEmptyEventLoop=false;
```
따라서 위 설정을 해주지 않으면 DB connection등으로 event loop이 비어있지 않게 되고,  
lambda의 timeout 시간까지 버티다가 강제로 종료된다.  

### lambda의 cold start & warm start
위 설정을 하더라도 많은 람다를 실행시키다보면 결국 connection이 가득 차게 된다.  
확실한건 아니지만 TCP에서 time_wait와 연관이 있어보이는데, AWS 환경에서 디버깅이 쉽지 않아 추측할뿐이다..
![tcp_close]({{ "/assets/images/lambda-db-connection/tcp_close.png" | relative_url }})  
위와 같이 OS에서 close를 하더라도 TIME_WAIT까지 대기하게 된다.  

코드에서 명시적으로 close 해봤는데 가득 찬다..
![db_connection]({{ "/assets/images/lambda-db-connection/db_connection.png" | relative_url }})  

Lambda는 4가지 과정을 통해 실행이 되게 된다.  
1. 람다를 구동시키기 위한 instance를 할당받고, 내가 작성한 람다 코드를 다운로드 받는다.
2. 런타임 환경이 instance에 구성된다.
3. handler 밖에 있는 global 영역의 코드가 실행된다.
4. lambda handler가 실행된다.

위와 같은 과정을 실행하는데, 처음 람다가 실행되는 경우 1~3 과정을 거치는데 이를 cold start 라고 한다.  
그러나 직후에 바로 람다를 다시 실행하면 1~3을 건너뛰고 4번만 실행하게 되는데, 이를 warm start 라고 한다.

테스트해본 코드는 다음과 같다.

{% highlight javascript %}
let counter = 0;
module.exports.handler = (event, context, callback) => {
    counter++;
    console.log(counter);
    callback(null, { count: counter });
}
{% endhighlight %}

처음 람다를 실행해보면 counter가 1이 되는데, 바로 다시 람다를 수행해보면 2가 된다.  
이를 응용해서 직전에 맺은 DB connection이 있다면 재사용할 수 있다.

{% highlight javascript %}
let dataSource;
export const connectDB = async () => {
    try {
        if(!dataSource){
            dataSource = await dataSource();
        }
        console.log('Data Source has been initialized!');
        return dataSource;
    } catch (err) {
        console.error('Error during Data Source initialization', err);
        throw new Error(`DB연결 오류 - ${err.message}`);
    }
};
{% endhighlight %}

위와 같이 db connection을 이전에 썼다면 재사용하고, 아니면 생성하게끔 한다.

이렇게 하고 500건을 실행 해봤는데 문제 없이 잘 되는걸 확인했다.  
![lambda_invocation_after]({{ "/assets/images/lambda-db-connection/lambda_invocation_after.png" | relative_url }})  
![db_connection_after]({{ "/assets/images/lambda-db-connection/db_connection_after.png" | relative_url }})  
6개정도 connection을 더 사용했고 큰 문제가 없었다.

### RDS proxy
다른 방법이지만 AWS에서 connection pooling을 위한 솔루션을 제공한다.  
RDS proxy를 쓰면 이 proxy가 대신 connection을 pooling 하고 proxy에 연결하여 idle connection을 할당받아 사용하는 구조이다.  
다만.. 람다 호출비용보다 가격이 더 나와서, 배보다 배꼽이 좀 큰느낌이 든다.  
지금은 내부에서만 사용할 목적으로 개발한거라 나중에 프로덕션용으로 api 외부 노출하고 사용할때  별도로 테스트해봐야 할듯      
드물게 대량으로 메일 보낼때만 스파이크 치는데 이것도 람다 동시실행 개수를 설정하면 connection도 거의 대부분 재사용해서 큰 문제가 없었다.  
따라서 이번 경우엔 고려하지 않았다.  

## 주의사항
lambda를 사용할때는 connection execution count를 꼭 설정하는게 좋다.  
일단 AWS 계정내에서 동시에 실행 가능한 람다의 수가 다 합쳐서 1000개로 제한이 있어, 다른곳에서 람다를 사용하게 된다면 영향을 미칠 수 있다.  
스트레스 테스트 해보면서 느낀건데 람다가 무거운 작업을 하지 않는 이상 1000개 다쓰기가 쉽지않을거같긴 하다.  

## 특이사항
메일을 전송하는 lambda는 큰 문제가 없었는데, SES의 event를 SNS => SQS를 통해 trigger 되는 lambda에서  
concurrent를 20으로 설정해놨는데 이보다 높게 잡혔다.  
![lambda_event]({{ "/assets/images/lambda-db-connection/lambda_event.png" | relative_url }})  
그래프는 70으로 높게 보이는데, 이게 기간을 짧게 보면 줄어든다. 기간을 길게하면 집계되는 수치가 합쳐져서 그렇다.  
기간을 좀 줄여봐도 순간적으로 20보다 높게 30정도로 잡히는 경우가 있었는데, 종료되는 람다와 새로 실행되는 람다가 동시에 잡혀서 그렇지 않나.. 싶다.  
FIFO SQS에서 lambda를 trigger 시킬 때 모든 message group id를 동일하게 해서 SQS에 넣어봤는데, 중복 메시지로 처리해서 lambda는 1개만 trigger되어야 할것으로 예상되었는데 2개로 찍혔었다.