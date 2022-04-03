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

### 기반 기술의 구체적인 구현체는 "반드시" 변경해야 하는 순간이 온다.
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

하지만 굴지의 `moment.js` 는 더 이상 신규 개발없이 유지보수만 하는 레거시 프로젝트로 전환하였고 이에 따라 새로운 라이브러리로 교체해야만 했다.

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

### 서버 API의 구체적인 구현 내용은 다른 계층에서 알게해선 안된다.
그 중 서버와 API를 통해 메세지를 주고 받는 책임을 갖는 API Client의 경우에 특별히 더 신경을 써줄 부분이 있다.

우선 동일하게 HTTP 요청/응답에 대한 인터페이스 `HTTPClient` 를 정의한다.

개발팀 내부에서 정의한 응답 포맷이 있을테고 기본적으로 이 포맷에 맞춰 응답을 벗겨낸 후 반환하게 될테지만, 간혹 외부 서비스의 API를 요청하는 경우도 종종 있으므로 사용하는쪽에서 이 전략을 선택할 수 있는 여지를 주면 좋다.

표현력을 높이기 위해 객체 생성을 위한 별도의 Builder를 제공하는 방향으로 구현하였다.

해당 인터페이스와 사용하려는 구체적인 구현체의 인터페이스를 맞춰주는 Adapter를 구현한다. 서버와 약속된 프로토콜 (서버에서 요청시 포함시켜달라고 요청한 헤더나 응답코드/예외코드에 대한 처리 규칙등)을 캡슐화하여 외부에서는 이 내용에 대해서 신경쓰지 않아도 되도록 한다.

이제 이 `HTTPClient` 를 사용하여 실제로 서버 API 에 요청을 보내는 책임을 갖는 삐삐를 구현한다.

이때 유의할점은 서버의 응답을 그대로 다른 계층에 노출하지 않아야 한다는 것이다.

서버의 응답을 서버의 응답을 그대로 상위계층에 노출할 경우 다음과 같은 문제가 발생한다.



