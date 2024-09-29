---
title:  "아마존의 가용 영역에서 알아야 할 점"
date:   2022-04-26 13:23:17 +0900
categories:
  - aws
tags:
  - aws
  - network
excerpt: "What you need to know about Amazon’s Availability Zones"
toc: true
toc_sticky: true
---
## 개요
AWS 비용 관련해서 리뷰를 받고 있었는데, AWS에서 표기되는 az(Availability Zone)과 실제 매칭되는 zone이 다르다는것을 알았다.  

## 확인
아마존에서 region을 선택하면 많게는 6개도 있는데, az을 설정할 수 있다.  
싱가폴(ap-southeast-1)의 경우 3개의 az이 있고 각각 ap-southeast-1a, ap-southeast-1b, ap-southeast-1c로 구분된다.

![az1]({{ "/assets/images/aws-availability-zone/az1.png" | relative_url }})  
위 이미지에서 보아야 할건 ZoneId와 ZoneName이다.  
ZoneName이 AWS 콘솔상에서 표기되는 이름이고, ZoneId가 매칭된 일종의 데이터센터이다.  

싱가폴 리전의 AWS account는 여러개 있었기에 다른 계정도 확인해보았다.  
![az2]({{ "/assets/images/aws-availability-zone/az2.png" | relative_url }})  

첫번째는 ID와 Name이 1-a, 2-b, 3-c로 예뻤(?)는데 두번째는 1과 2과 다르게 되어있다.  
1-b, 2-a, 3-c  

```
aws ec2 describe-availability-zones --region <리전> --output table --profile <profile명>
```
위 명령어를 통해서 확인해볼 수 있다.

## 왜 이런가 ?
보통 사람들이 zone을 선택하면 주로 1번을 선택할거고, 고가용성을 위해 여러개의 zone을 선택해도, 사람 심리상 1번과 나머지를 선택할것같다.  
아니면 특정 데이터센터를 선호하는 경우도 있지 않을까 싶다.  
아마존에서도 아마 이를 생각해서 특정 데이터센터에 리소스가 편향되는걸 막고자 콘솔상에서는 1~n번으로 표기하고 랜덤하게 데이터센터를 할당해서 리소스를 균등하게 분배하려는 전략이지 않을까 싶다.

## 결론
실제로 저 Zone ID가 할당되는건 AWS 계정을 생성했을때 임의로 할당된다.  
다만 이건 region마다 다르고, region별로 고정해서 Id를 발급하는 곳이 있고 아닌곳이 있다고 한다.  
서울(ap-northeast-2)은 고정, 싱가폴이나 버지니아같은 경우 임의로 할당된다고 함  
고정이라는 의미는 1-a, 2-b, ... 이런식으로 알파벳 순서와 번호가 일치한다는 뜻이다.
또한 한번 할당되면 저 값은 변경할 수 없다.

## 고려사항
이게 무시할 수 없는 경우가 있는데, 동일 VPC내거나 / VPC peering을 맺은 cross account간 트래픽 전송이 "같은 region에 있으면" 무료이다.  
즉 Zone Id가 같으면 무료이다.  
아마존 문서 내에서 "zone이 같으면" 이란 뜻은 zone id가 같다면 이란 말이다.  
[VPC 관련 AWS 비용 공지](https://aws.amazon.com/ko/about-aws/whats-new/2021/05/amazon-vpc-announces-pricing-change-for-vpc-peering/?nc1=h_ls)  

[Zone 참고 AWS 문서](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/using-regions-availability-zones.html#az-ids)  
이걸 위해 게임처럼 리세마라를 해야할 필요까진 없어보이지만, AWS 계정도 뽑기운(?)에 따라서 트래픽에 돈을 더 낼수도..