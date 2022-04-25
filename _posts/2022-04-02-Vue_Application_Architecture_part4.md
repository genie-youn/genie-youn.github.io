---
layout: post
title: "Vue Application Architecture - Application (Part4)"
author: "genie-youn"
categories: journal
tags: [Vue, Front End, Architecture, DDD]
image: DSC04956.jpg
---
# Table of Contents
1. [복잡함을 해결해야 하는 이유 - Part1](https://genie-youn.github.io/journal/Vue_Application_Architecture_part1.html)
2. [Infra - Part1](https://genie-youn.github.io/journal/Vue_Application_Architecture_part1.html)
2. [API Client - Part2](https://genie-youn.github.io/journal/Vue_Application_Architecture_part2.html)
3. [Domain - Part3](https://genie-youn.github.io/journal/Vue_Application_Architecture_part3.html)
4. Application
5. [UI - Part5](https://genie-youn.github.io/journal/Vue_Application_Architecture_part5.html)
6. [마치며 - Part5](https://genie-youn.github.io/journal/Vue_Application_Architecture_part5.html)

> [Part3](https://genie-youn.github.io/journal/Vue_Application_Architecture_part1.html) 에서 이어집니다.

이제 UI와 관련이 없는 기반 기술, API에 대한 상세한 내용, 그리고 UI와 관련이 없는 서비스 정책이나 업무 규칙을 모두 분리해내고 온전히 UI 와 관련된 관심만이 남았다.

남아있는 UI 와 관련된 관심사들은 크게 두 계층 Application 과 단어적 정의 그대로의 UI 로 나눈다.

그리고 이 두 계층 모두 Vue 라는 프레임워크 위에 구현되게 된다.

<img width="1024" alt="스크린샷 2022-04-19 오후 9 05 27" src="https://user-images.githubusercontent.com/16642635/163999425-3bddd4a8-3656-4f1e-bb3c-6587c24cae56.png">

# Application

해당 계층엔 애플리케이션이 동작하도록 하는데 관심을 갖는 객체들로 구성된다.
당연히도 프론트엔드 애플리케이션이니 UI와도 밀접한 관련을 갖는다.

기반 기술도 아니고, 서비스의 정책이나 업무 규칙을 나타내는 도메인 객체도 아니면서 API 요청과 응답에 대한 객체가 아닌데다 UI와 관련이 있지만 UI 그 자체는 아닌 어디에도 속하지 않는 대부분이 이 계층에 위치할것이다.

예를들면, 페이지간의 네비게이션을 담당하는 `router`, 애플리케이션 전역에서 접근이 필요한 상태를 관리하는 `store`, 애플리케이션의 특정 로직과 상태를 캡슐화한 `composition (hook)` 등등이 있겠다.

기존에 Vue 애플리케이션을 구현하던 것 처럼 구현하면 된다. 이중 `store` 의 설계에 대해서 간단히 설명하고 넘어가도록 하자.

## Store

`Store` 는 Vue 애플리케이션에서 "전역적인" "상태를 관리"하기 위한 저장소로 사용하여야 한다.

### 애플리케이션 전역에서 필요한 상태만을 담는다.

간혹 페이지 전역에서 접근이 필요한 상태를 저장하기 위해 `Store` 를 사용하는 경우가 있는데, 이는 곧 페이지 단위로 `Store` 를 구성하는 결과로 이어지고 많은 문제들을 야기한다.

우선 스토어의 네임스페이스가 오염된다. 페이지별로 하나의 `Store` 모듈이 생성된다면, 많은 페이지를 가진 애플리케이션 이라면 압도적인 글로벌 스토어를 마주해야 할 것이다.

두번째 문제로는 상태가 파편화될 가능성이 높다.

예를들어, 현재 로그인한 사용자의 상태가 A페이지의 `Store` 와 B페이지의 `Store` 모두에 저장되어 있다면 개발자는 어떤 `Store`의 상태를 신뢰하고 사용해야 할까?

`Store` 는 그 본래의 역할에 맞게 애플리케이션 전역에서 접근이 필요한 상태를 관리하도록 하고 그 대상의 단위로는 앞서 설명했던 도메인 모델 객체를 단위로 삼는다.

> 페이지 전역에서 접근이 필요한 도메인 모델 객체나, UI의 상태에 대해선 후에 서술한다.

### 상태의 라이프사이클을 관리하는 책임만을 부여한다.

간혹 이 `Store` 내에서 위에서 설명한 서비스 정책이나 업무 규칙을 처리하는 경우가 있다.

이러한 로직들이 간단하다면 상관없겠으나, UI에서 이러한 로직을 분리해냈듯이 상태의 라이프사이클을 관리하는 책임만으로 `Store` 는 충분히 복잡하다.

따라서 서비스 정책이나 업무 규칙에 대한 처리는 Domain 계층에게 위임한다.

`Store`는 각 도메인 모델 객체의 라이프사이클에만 관심을 갖도록 한다.

예를들어, 현재 로그인한 사용자의 정보의 경우 로그아웃을 하거나 마이페이지에서 변경하기 전까지는 변경 가능성이 낮고 애플리케이션 전반에서 접근이 필요하다.

따라서 이 상태는 `Store`에서 관리하고 로그인했을때 메세지를 받아 fetch 하여 메모리에 올려두고, 마이페이지에서 변경이 이루어진다면 다시 메세지를 받아 새로 fetch 한뒤 로그아웃시에 메세지를 받아 상태를 파기하도록 하면 상태를 효율적으로 관리할 수 있을 것이다.

이처럼 애플리케이션 전역에서 접근이 필요한 도메인 모델 객체를 선정하고, 각 모델에 맞게 라이프사이클을 정의하여 관리하도록 해라.

> 위 사용자 정보는 이해를 돕기 위한 예제일 뿐이다. 사용자의 접근 권한이 중요하다거나 하는 경우 매 순간 fetch 하여 상태를 갱신해주어야 할 것이다. 애플리케이션마다 이 객체들의 라이프사이클은 제각기 다를것이고, 신중히 고민하여 설계하는것이 중요하다.

> namespace와 관련된 문제는 Vue의 새로운 상태관리 도구인 [`pinia`](https://pinia.vuejs.org/introduction.html)를 사용하면 많은 부분 해결할 수 있다.

<img width="619" alt="스크린샷 2022-04-20 오후 10 48 01" src="https://user-images.githubusercontent.com/16642635/164245140-ffce8331-7bbb-4905-bcf5-eb37b499b9e4.png">

<img width="1680" alt="스크린샷 2022-04-20 오후 10 48 30" src="https://user-images.githubusercontent.com/16642635/164245223-cbe7b01d-30f0-48be-8cce-5adc8574a3ba.png">
