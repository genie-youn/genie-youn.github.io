---
layout: post
title: "Vue Application Architecture (Part1)"
author: "genie-youn"
categories: journal
tags: [Vue, Front End, Architecture, DDD]
image: DSC04956.jpg
---

기본적으로 Layered Architecture 를 채용한다. `ui`, `application`, `domain`, `infra` 네개의 계층으로 구성된다.

가장 하위 계층인 `infra` 계층부터 살펴보도록 하자.

# Infra
애플리케이션이 동작하기 위한 기반 기술을 제공하는 계층이다. 클라이언트 애플리케이션에서 서버와 메세지를 주고 받는 책임을 갖는 삐삐나 로컬스토리지, 세션스토리지와 같은 저장소에 접근하여 데이터를 영속화하는 책임을 갖는 삐삐등이 이 계층에 속한다.

왜 이러한 삐삐들을 별도 계층으로 격리해야 하는걸까?

### 기반 기술의 구체적인 구현체는 "반드시" 변경해야 하는 순간이 온다.
이 계층의 삐삐들은 그 특성상 애플리케이션 전반에서 광범위하게 의존하게 된다. 예를들어 서버와 메세지를 주고 받는 기술의 구현체로 `axios` 를 사용중이라고 하자. 이 `axios` 라는 구체적인 구현체를 별다른 격리 없이 다음과 같이 사용하였다.

AwesomeComponent.vue
```vue
<template>
  <button @click="action">액션!</button>
</template>
<script>
import axios from "axios";

export default {
  methods: {
    action() {
      axios.post("/my-awesome-action");
    },
  }
}
</script>
```

그러던 어느 날 세상에 없던 멋진 HTTP Client 라이브러리가 짜잔하고 등장하였다.

합리적인 알고리즘으로 요청과 응답을 멋지게 캐시해주고 그 결과 요청의 수도 획기적으로 줄일 수 있고 응답속도 또한 빨라진다고 한다.

프로젝트에 당장 적용하고 싶어졌다. `axios` 를 걷어내고 이 멋진 라이브러리로 대체하리라.

헌데 위와 같이 `axios` 에 직접 메세지를 전달하는 컴포넌트가 수백개쯤 된다면?

아마 저 멋진 라이브러리는 아직 스타도 몇개 없고 안정성도 검증되지 않았다며 신포도라고 여겼을 것이다.

내가 개발하고 있는 서비스는 날짜/시간대 관련된 처리를 `moment.js` 에 위임하고 있었다.

하지만 굴지의 `moment.js` 는 더 이상 신규 개발없이 유지보수만 하는 레거시 프로젝트로 전환하였고 이에 따라 새로운 라이브러리로 교체해야만 했다.

만약 이 `moment.js` 에 직접 의존하게 설계했다면 모든 사용처를 찾아 새로운 라이브러리에 맞게 변경해주었어야 했을 것이다.

정리하면 기반 기술의 구체적인 구현체, 특히 구현을 제어할 수 없는 외부 라이브러리라면 추상화된 인터페이스를 정의하고 상위계층에선 이 인터페이스를 의존하게 해라.

DateTime.ts
```javascript
import MomentAdapter from "./moment/MomentAdapter";

function isSameDay(dateLeft, dateRight) {
  return MomentAdapter.isSameDay(dateLeft, dateRight);
}

function addDays(date, amount) {
  return MomentAdapter.addDays(date, amount);
}

//.. 생략

export default {
  isSameDay,
  addDays,
}
```
