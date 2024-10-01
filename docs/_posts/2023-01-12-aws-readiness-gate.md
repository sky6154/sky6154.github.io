---
title:  "ALB / EKS와 pod readiness gate"
date:   2023-01-12 13:24:32 +0900
categories:
  - kubernetes
tags:
  - load balancer
  - kubernetes
  - pod
excerpt: "ALB and EKS pod readiness gate"
toc: true
toc_sticky: true
---
## 개요
AWS에서는 pod중에 ALB와의 연동을 위해 `aws-load-balance-controller` 라는 pod를 제공한다.  
기존에는 kubernetes의 service를 AWS의 target group과의 연결을 위해 아마존에서 만든 CRD인 `targetgroupbinding` 을 통해 pod를 ALB에 연결하고 있었다.  
사내 운영툴의 경우 1대만 띄워놓는 경우가 많은데 순단이 발생하는 현상을 확인하여 원인을 찾아보았고, 서비스 안정성을 좀 더 개선할 수 있는 방법을 찾아 이를 정리한다.  

## 설명
deployment의 이미지가 교체되어 pod가 rollout 될 때 pod가 트래픽을 받을 준비가 되었다고 판단되면 교체를 진행한다.  
pod가 준비 되었다고 판단하는 기준은 readiness probe가 정해진 threshold만큼 성공을 반환하면 진행되는걸로 알고있었는데, 이게 kubernetes 내부에서는 상관이 없지만 외부 서비스와 연동되는 경우를 위해 kubernetes는 readiness gate를 만들었다.  
readiness probe 뿐만 아니라 readiness gate까지 통과해야 pod를 사용 가능한뜻으로 판단한다.  
`aws-load-balance-controller`가 pod의 lifecycle에 관여할 수 있는데, namespace에 특정 label의 존재 유무를 판단하여 target group의 상태까지 판단할 수 있게끔 설정이 가능하다.  

## 사용 이유
AWS의 target group도 health check를 진행한다.  target group이 health check을 통과하지 않는 경우 LB는 트래픽을 흘려보내지 않는다.  
readiness gate를 사용하지 않는 경우 pod는 교체되었으나 target group의 health check이 통과하지 않아 rollout 비율에 따라 나머지 pod들에 과도하게 트래픽이 몰릴 수 있거나, 1대인 경우 순단이 발생할 수 있다.  

아래 이미지는 pod가 1대일 때 target group과 pod의 상태에 따라 순단이 발생할 수 있는 경우를 보여준다.  
![state1]({{ "/assets/images/aws-readiness-gate/state1.png" | relative_url }})  

아래 이미지는 pod가 2대일 때 순간적으로 pod 1대에 트래픽이 몰릴 가능성을 보여준다.  
![state2]({{ "/assets/images/aws-readiness-gate/state2.png" | relative_url }})  

따라서 target group의 health check를 통과했을 때 pod 교체가 일어나야 하며, 이를 위해 aws의 `aws-load-balance-controller`가 readiness gate 설정을 자동으로 해줄 수 있게끔 설정할 수 있다.  
이 경우 namespace에 특정 label이 있고 svc와 연결된 target group binding CRD가 있는 경우에 설정이 가능하다.

## 설정 방법
namespace의 label에 `elbv2.k8s.aws/pod-readiness-gate-inject: enabled` 를 설정한다.  
[AWS 문서](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.1/deploy/pod_readiness_gate/) 참고

## 결론
이를 위해(원래 해야하지만) aws-load-balance-controller의 로그나 kubernetes events 를 잘 보아야 한다.  
target group arn이 없거나, ingress를 통해 LB를 생성하거나, 여러가지 AWS 리소스와의 싱크를 맞추는 역할을 aws-load-balance-controller가 해주는데 싱크가 맞지 않는 경우 reconciled에 실패했다고 나오는데 이게 좀 오래되거나 양이 많거나 하는 경우 aws-load-balance-controller가 먹통이 되어 올바르게 싱크를 맞춰놔도 싱크를 맞추지 못하고 먹통이 되는 현상을 개발환경에서 몇번 겪었었다.   
