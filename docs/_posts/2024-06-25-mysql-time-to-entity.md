---
title:  "Entity에서 LocalTime 사용 시 유의점"
date:   2024-06-25 17:56:25 +0900
categories:
  - java
tags:
  - entity
  - jdbc
  - mysql
excerpt: "Things to keep in mind when using LocalTime in Entity"
toc: true
toc_sticky: true
---
## 개요
기존 사용하던 entity에 mysql의 `time` 컬럼을 추가할 일이 생겼다.    
모두 UTC로 통일하여 사용하고 있었기 때문에 기존 코드에 LocalDateTime을 사용해서 `created_at` 등의 컬럼을 파싱하고 있는게 있었고 큰 생각없이 LocalTime으로 column을 파싱했는데 -9시간이 되어 나왔다.  

## 확인
main 함수에서는 다음을 통해 timezone을 통일하고 있었다.  
{% highlight java %}
public static void main(String[] args) {
    TimeZone.setDefault(TimeZone.getTimeZone("UTC"));
    Locale.setDefault(Locale.forLanguageTag("en"));
    SpringApplication.run(Application.class, args);
}
{% endhighlight %}

DB와 연결할때도 `serverTimezone=UTC` 옵션을 주어 UTC로 설정하고, DB에 직접 붙어서 timezone을 확인해도 모두 UTC였다.  

DB의 `timestamp` 컬럼으로 된 값을 LocalDateTime으로 가져올 경우 LocalDateTime을 가져올때는 정상적으로 가져오는데, `time` 컬럼을 LocalTime으로 가져올때는 여전히 -9시간을 가져왔다.  
개발 PC가 mac이었는데, timezone이 서울이었다. UTC+0 시간의 지역으로 바꿔보니 정상동작했다...  

아마도 `timestamp`는 UTC를 기준으로 구하는거라 별도의 연산을 하지 않고, `time`은 zone 정보 없이 시간 정보만 존재하는 특징 때문에 그럴것으로 추측된다.  
TimeZone 코드를 봐도 우선순위 가장 높은게 default인데..  
`LocalTime.now()`를 찍어봐도 UTC로 잘 나온다.  
근데 DB도, java 설정도, connection 맺을때도 UTC인데 DB에서 23:00의 `time` 컬럼을 가져오는데 `14:00`으로 가져오고 mac의 시간을 Asia/Seoul에서 UTC+0 대역으로 바꾸면 -9시간을 하지 않는다..  
hibernate 설정이 뭐가 있는건지..  


## 해결
### OffsetTime
LocalTime 대신 OffsetTime을 사용한다.  

### java.sql.Time
java.sql 패키지의 Time을 사용한다.

### 로컬 PC를 UTC
이건 좀...

## 결론
원래 작성할때는 zone 정보가 있는걸 쓰는 편인데, 기존 시스템을 유지보수 하다보니 관성적으로 기존 코드 스타일을 따라가서 발생한 문제였다.  
시간 정보가 있는 entity를 모두 timezone이 들어간 클래스로 변경했고, ZonedDateTime / OffsetTime으로 테스트했는데 큰 문제는 없었다.  
container로 배포하는 경우 timezone이 UTC로 설정되기 때문에 실제로 개발환경에 배포했을때 이슈가 있지는 않았다.   
하지만 국내 대상으로 서비스한다고 하더라도 언제 글로벌 서비스도 할지 모르고, 시간 관련해서 이슈가 생길 경우 오류도 아니어서 찾기 쉽지 않기 때문에 항상 zone을 염두해두고 코드를 작성하는게 좋을것 같다.  
DB에서 값을 가져올 때 어떤식으로 가져오는지 디버깅을 해봐야 할듯하다. 당장은 시간이 없어 OffsetTime으로 수정했다.
