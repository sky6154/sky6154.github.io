---
title:  "kafka를 준비하며 고민한 여러 문제들"
date:   2024-03-12 11:36:22 +0900
categories:
  - spring
tags:
  - spring boot
  - msk
  - kafka
  - kubernetes
excerpt: "Various issues I thought about while preparing for Kafka"
toc: true
toc_sticky: true
---
## 개요
spring boot 프로젝트에서 kafka를 사용하면서 발생한 이슈들에 대해 정리해본다.  

## 이슈
### consumer rebalancing과 kubernetes rollout
kafka를 도입전 테스트하면서 정말 여러가지 시나리오를 생각해봤는데, 바로 consumer rebalacing에 드는 비용과 배포가 일어났을 경우이다.  

partition과 consumer가 매핑된 상태이고, partition의 개수보다 consumer 개수가 적을 때 consumer 수가 줄어들거나, 늘어나는 경우 재배치가 발생하고 이 때 assign 정책에 따라 메시지 소모가 멈출 수 있는데 정책은 다음과 같다.  

* Range assignor
* Round-robin assignor
* Sticky assignor
* Cooperative sticky assignor

각각의 특성은 있으나 보통 cooperative sticky assignor를 사용하는 편인데, 멈추는 시간이 일반적으로 가장 적기 때문이다.  
스트레스 테스트를 하면서 pod를 강제로 내려보고 얼마나 멈추는지 등을 확인해봤는데 가장 안정적이었기 때문이다.  

kubernetes는 기본 배포전략이 rollout이다.  
순단을 최대한 줄이고 안정적인 서비스를 하기 위함인데, 이 때 maxSurge와 maxUnavailable은 기본값이 25%이다.  
이 말은 배포 시 총 pod에서 25%를 내리고, 새로운 버전의 25%를 배포한다는 내용인데 이렇게 점진적으로 pod를 교체할 경우 한번의 배포에 총 8번의 rebalancing이 일어나게 된다.  
pod가 create되어 running 상태가 되어 consumer group에 join하게 되고, running 상태가 확인되면 오래된 pod가 terminate 되어 빠지게 되어 순간적으로 두번 발생한다.  
아직 많은 수의 partition과 consumer를 다루어보진 않았으나, 하나의 application이 뜨는 시간이 느리고 node가 provision되는 시간을 고려하면 모든 pod 교체까지 제법 오래걸리게 되고 rebalancing도 많이 일어난다.  

AWS등의 서비스를 사용할 경우 많은 수의 node를 바로 늘릴 수가 있기 때문에 QoS를 크게 고려할 필요는 없어 maxUnavailable은 0으로 두는 편이다.  
topic당 partition이 많진 않고 broker별 partition 수도 많진 않았기에 테스트시에 큰 문제는 없었지만 maxSurge를 50%로 두어 2번으로 모든 pod가 교체되게끔 변경했다.  

### spring boot의 health check
consumer가 여러개의 topic을 consume하고 있다고 가정했을 때 특정 topic에 이슈가 생기는 경우 health indicator를 어떻게 지정해주어야 하는지 ?

우선 서비스 특성 상 메시지가 소모되지 않아도 서비스에 지장없는 경우와 그렇지 않은 경우로 분류해보았다.  
하지만 서비스에 지장이 있는 토픽일지라도 그 개수가 여러개가 된다면 health check 실패가 되어도 맞는지를 고민해봤는데 답이없는 문제여서 kafka의 경우 항상 up 상태를 유지하게끔 변경했다.  
pod가 내려가고 새로 떠도 아마 해소가 안될거고, 오히려 pod가 새로 떴다가 죽는게 rebalancing을 유발하기만 하고, 하나의 topic 소모에 문제가 발생했다고 다른 topic 소모에 지장을 주는게 말이 안된다고 생각했다.  
offset lag을 확인하거나 메시지를 소비 못하는 등 topic 소모에 지장이 생길 경우 별도의 알람을 통해 처리하는것으로 변경했다.  
다만 이건 topic 읽기에 실패할 경우 DLQ 처럼 다른 topic이나 DB로 옮기고 재처리를 유도하거나 할 경우에 따라 대응 방식이 달라질 수 있으므로 application의 특징에 따라 갈릴것 같다.  

또한 consumer가 응답이 없을 경우 consumer group등에서 제외되는 시간등의 설정도 있는데, 꼼꼼하게 고려되어야 한다.  

### schema registry와 evolution
보통 api를 작성하거나, 암호화를 적용하거나, 직렬화를 하거나 등등 뭔가를 개발할때마다 항상 고려해야 하는건 이전 버전과의 호환성이다.  
현재 서비스하는 버전이 뭔가 개선되어 스펙이 변경된다면 결국 두개의 호환성을 해결해야 하는데, 이를 해결해야 한다.  
여러 schema registry가 있으나, AWS glue schema registry가 있었고 이를 사용했다.  
결국 schema registry와의 통신도 어떤 네트워크망을 탈것이냐인데, aws 서비스를 사용한다면 aws 망내에서만 통신이 발생하기 때문에 여러 이슈로부터 자유로울 수 있을거같았고, registry를 유지하기 위한 비용 또한 크게 들지 않을거라고 생각되었다.  

만약 schema registry를 사용하지 않는다면 메시징 구조에 대해 고려가 필요하다.  
구조에 대해 정의하지 않을 경우 schema를 topic에 매번 실어보내야 하는데, 메시지 크기 제한도 있을뿐만 아니라 매번 비싼 네트워킹 비용을 치를 수 있다.  
네트워크내 불필요한 트래픽 양도 많이 늘고, 파싱하는 입장에서도 java등의 강타입 언어를 사용한다면 결국 Map을 사용할수밖에 없다.  

또한 schema registry에서도 고려할 전략이 있는데, 다음과 같다.  

* BACKWARD
* BACKWARD_TRANSITIVE
* FORWARD
* FORWARD_TRANSITIVE
* FULL
* FULL_TRANSITIVE
* NONE

각각 다음 버전으로 메시지 구조를 변경할 경우 어떻게 할지에 대한 전략이다.  
예를 들어 backward의 경우 새 버전이 나올 경우 이전 버전도 consume 가능하게 보장하는걸 의미한다.  
따라서 필드는 삭제만 가능하며, optional 필드의 경우 추가가 가능하다.  
또한 consumer부터 업데이트가 완료된 이후 producer가 배포되어야 하는 특징을 갖는다.  
`TRANSITIVE`가 포함되면 직전 버전뿐만아니라, 여태 발행된 모든 스키마에 대해 consume 가능한걸 의미한다.  

다만 변경 허용등에 대해 확장이 제한적이므로, 사전에 많은 고민을 하지 않는다면 유지보수에 엄청 고생할 수 있다.  

### message serialize 전략
JSON처럼 자주 사용되는 구조는 여러 언어에서 지원되기 때문에 범용성면에서 좋지만, 속도, 용량에서 많은 차이를 보인다.  
그래서 직렬화/역직렬화 시 속도나 메시지 용량을 줄이기 위해 여러 라이브러리들이 있는데 고민한건 다음과 같다.  

* Protobuf
* avro

두개 말고도 더 있으나 glue schema registry가 지원을 하지 않는게 가장 컸다.  
어떤 직렬화 라이브러리를 사용하느냐에 따라 라이브러리 특징에 따라 몇몇 기능이 제한될 수 있어 주의해야 한다.  
예를 들어 FORWARD에서 protobuf는 신규 타입을 schema에 추가할 수 없다.  

테스트를 해봤을때 각자의 장단점이 있었고 다음과 같다.  
* JSON
  * 장점 : 프로그래밍 언어에 종속적이지 않음
  * 단점 : 용량이 큼(아래 두개보다 1.5배~2배정도 ?)
* AVRO
  * 장점 : protobuf보다 직렬화시 사이즈가 작음, 지원되는 언어 확인 필요
  * 단점 : protobuf보다 조금 느림
* Protobuf
  * 장점 : 직렬화&역직렬화 속도가 제일 빠름, 지원되는 언어 확인 필요
  * 단점 : avro보다 사이즈가 조금 큼


## 고찰
그 밖에도 여러 고려사항이 있었으나 다 적기엔 양이 많을듯 싶다.  
metric별로 어떤 기준에 따라 알람을 받을것인지, topic 생성 정책, isr / replica 수, broker 수, 비용 등등..  

무언가를 도입할 때 항상 고민이 든다.  
빠르게 도입하고 결정할지, 여러가지를 따져본뒤 도입할지..  
보통 중요하지 않은 서비스부터 시작해서 점진적으로 늘려가긴 하는데, 전혀 예측못한 이슈들이 발생할 수 있으므로 항상 장애에 대한 여러 시나리오를 만들고 준비하는게 중요하다고 생각된다.  
kafka는 잘못 관리되면 SPOF가 되거나 장애가 전파될 수도 있으므로 특히 주의해야 한다.  