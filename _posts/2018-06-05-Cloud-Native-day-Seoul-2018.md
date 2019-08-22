---
layout: post
title: "Cloud Native day Seoul 2018 후기"
author: "genie-youn"
categories: journal
tags: [documentation,sample]
---

Pivotal이 한국에서 처음 `Cloud Native Day`라는 이름으로 세미나를 진행하였다. Spring의 리드그룹인 Pivotal이 어떤 방향성을 갖고 있는지 궁금하기도 했고, 혹시 향후에 Spring으로 MSA를 개발할 때 도움이 될까 싶어 참여하게 됐다.
히로쿠의 [The Twelve-Factor app](https://12factor.net/ko/) 같은걸 기대하고 갔다.

진행세션은 다음과 같다.

```
환영사 및 글로벌 트렌드 소개
클라우드 네이티브로의 전환을 위한 여정과 성공사례
넷플릭스 서비스와 피보탈
Spring Project와 Pivotal Cloud Foundry의 최신 업데이트
Google Cloud Platform을 통한 Pivotal Cloud Foundry의 확장
클라우드 네이티브 엔터프라이즈 사례 발표 (Case studies)
```

## 세션2. 클라우드 네이티브로의 전환을 위한 여정과 성공사례

클라우드 네이티브는 세가지로 구성된다고 할 수 있다. devOps, Cloud Native App(Microservice Architecture), Organazation / Process.

### 데브옵스

데브옵스는 애플리케이션과 서비스를 빠른 속도로 제공할 수 있도록 조직의 역량을 향상시키는 문화 철학, 방식 및 도구의 조합입니다. 기존의 소프트웨어 개발 및 인프라 관리 프로세스를 사용하는 조직보다 제품을 더 빠르게 혁신하고 개선할 수 있습니다. 이러한 빠른 속도를 통해 조직은 고객을 더 잘 지원하고 시장에서 좀 더 효과적으로 경쟁할 수 있습니다. - _[amazon](https://aws.amazon.com/ko/devops/what-is-devops/)_

이 말이 제일 인상깊었다. Optimizing time to value. 변화하는 요구사항에 맞춰 코드를 작성하고, 이 코드가 실제 사용자에게 도달하여 가치를 창출하는 시간을 줄여라. 그리고 그 사용자의 피드백을 빠르게 받아들여라.

이를 위해 빌드/배포 파이프라인을 구성하고 지속적통합, 지속적전달을 통해 빌드/배포를 자동화하여 빠르고 쉽게 만들것.
Platform팀은 개발자들이 비즈니스 로직을 코드로 구현하는데 집중하게 하기 위해 코드형 인프라/서비스를 제공할것.

### 클라우드 네이티브 앱
마이크로서비스 아키텍처
기존의 monolithic 시스템은 여러 단점이 있다. -> MSA가 기존 monolithic한 시스템의 문제들을 해결해 줄것이다.
마이크로 서비스 아키텍처에 대해선 [이 글](https://www.slideshare.net/saltynut/building-micro-service-architecture) 과 [이 글](https://medium.com/coupang-tech/%ED%96%89%EB%B3%B5%EC%9D%84-%EC%B0%BE%EA%B8%B0-%EC%9C%84%ED%95%9C-%EC%9A%B0%EB%A6%AC%EC%9D%98-%EC%97%AC%EC%A0%95-94678fe9eb61)을 읽으면 도움이 될 것이다.


### Organization / Process
XP를 해라..

그뒤로는 Pivotal이 위에것들을 도와줄 수 있다.. 하는 영업

## 세션3. 넷플릭스 서비스와 피보탈
클라우드 네이티브한 MSA를 가져가기 위해서 아래 책들은 좋은 레퍼런스가 될것이다.

```
클라우네이티브자바 - 조쉬롱
카프카 데이터플랫폼의 최강자
```

넷플릭스가 왜 기존의 거대한 monolithic 시스템을 MSA로 바꾸었는지에 대한 이야기. 다른 자료도 많지만 [이 글](https://media.netflix.com/ko/company-blog/completing-the-netflix-cloud-migration)을 보자

이렇게 MSA로 바꿔나가면서 넷플릭스는 MSA에 꼭 필요하다고 생각되는 컴포넌트들을 오픈소스화 하여 MSA 진영을 이끄는데 기여하고 있다.

각각의 마이크로서비스가 뜰 때 설정 정보를 주입해주는
configuration server - [archalus](https://github.com/Netflix/archaius)

어떤 마이크로서비스가 어디에(어떤 ip/호스트로) 떠있는지 알려주는 전화번호부
service discovery(registry) - [eureka](https://github.com/Netflix/eureka)

장애가 생겼을 때 장애를 고립시켜 다른 마이크로 서비스에 영향을 주지 않게 돕는 누전차단기
circuit breaker - [Hysrix](https://github.com/Netflix/Hystrix)

Zuul is a gateway service that provides dynamic routing, monitoring, resiliency, security, and more.
api gateway - [zuul](https://github.com/Netflix/zuul)

Ribbon is a Inter Process Communication (remote procedure calls) library with built in software load balancers. The primary usage model involves REST calls with various serialization scheme support.
load balacer - [ribobn](https://github.com/Netflix/ribbon)

모니터링 도구
realtime monitoring - [atlas](https://github.com/Netflix/atlas) / [spectator](https://github.com/Netflix/spectator)

빌드 / 배포도구
zero downtime delivery - (CI/CD)[spinnaker](https://www.spinnaker.io/) (canary)kayenta

fault injection

랜덤하게 인스턴스나 DB를 죽이고 다니는 카오스 몽키 - 이 인스턴스가 죽으면 어떤 영향이 생기지?
[chaos monkey](https://github.com/Netflix/chaosmonkey)

클라우드를 위한 분산 메모리 캐시
[evcache](https://github.com/Netflix/EVCache)

기타 등등 많은데 유투브에 `넷플릭스 혼돈의제왕`을 검색해보면 좀 더 자세한 설명을 볼 수 있다.

## 세션4. Spring Project와 Pivotal Cloud Foundry의 최신 업데이트
PCF에 대한 간략한 소개. 스프링 부트로 마이크로 서비스를 만들면 PCF에서 제공하는 cli로 이걸 배포하고, 모니터링하고 위에 넷플릭스가 만든 OSS들을
스프링 환경에서 쉽게 사용할 수 있도록 준비해두었다. actuator라던지..

## 세션5. Google Cloud Platform을 통한 Pivotal Cloud Foundry의 확장
PCF 가 GCP를 워크로드로 쓸꺼다. GCP이 최고다.

## 세션6. 클라우드 네이티브 엔터프라이즈 사례 발표 (Case studies)
기존의 많은 IT와 거리가 먼 엔터프라이즈들, 거대 금융회사나 포드같은 자동차 제조사들이 IT를 내제화 하는데 관심을 갖고 있고 Pivotal을 이에 컨설팅을 제공한다.
그들이 devOps 개발팀을 가질 수 있도록 페어로 붙어서 함께 일하고 그들이 가진 monolithic한 시스템중 가장 코어한 부분을 마이크로 서비스로 분리해내는 일을 함께 하며
노하우를 축적한 쿡북을 만든다.

USA airforce pivotal로 검색하면 좀 더 자세한 사례를 볼 수 있다.
중요한건 한 이터러블이 끝날때 피쳐가 동작하고 피드백을 받을 수 있어야 한다는것.

## 후기.

사실 좀 더 기술적인 이야기를 할 줄 알았는데, 이 점에 좀 실망했다. 하지만 적어도 Spring이, 그리고 Spring을 이끄는 Pivotal이 옳다고 생각하는 방향에서 Cloud Native와 MSA는
빼놓을 수 없는 키워드인것 같다. 분명 매력적이다.
