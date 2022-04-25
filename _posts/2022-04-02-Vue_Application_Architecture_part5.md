---
layout: post
title: "Vue Application Architecture - UI, 마치며 (Part5)"
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
4. [Application - Part4](https://genie-youn.github.io/journal/Vue_Application_Architecture_part4.html)
5. UI
6. 마치며

> [Part4](https://genie-youn.github.io/journal/Vue_Application_Architecture_part1.html) 에서 이어집니다.

<img width="1024" alt="스크린샷 2022-04-19 오후 9 05 27" src="https://user-images.githubusercontent.com/16642635/163999425-3bddd4a8-3656-4f1e-bb3c-6587c24cae56.png">

# UI

이제 상태에 따라 화면을 렌더링하고, 유저와 인터렉션을 처리하며, 상태가 변경됐을때 다시 화면을 렌더링하는 UI의 본질적인 관심만이 남았다.

이 계층이 Vue 애플리케이션의 핵심이라 해도 과언이 아니다.

Vue 애플리케이션은 부모 컴포넌트로부터 주입받은 `props` 와 자신의 상태인 `data` 에 반응형으로 UI를 렌더링하는 `component` 들을 조합하여 전체 UI를 구성한다.

이 컴포넌트들은 재사용성을 확보하고, 요구사항이 변경되었을 때 기능을 확장하고 변경하기 쉽도록 책임에 따라 최소한의 단위로 작게 나누어진다.

때로는 하나의 컴포넌트가 여러 자식 컴포넌트를 갖기도 하고, 그 자식 컴포넌트가 또 다시 여러 자식컴포넌트를 갖기도 한다.

이에 따라 많은 정보를 표현해야 하는 웹 페이지는 많은 단계의 깊이를 가진 복잡한 계층구조의 컴포넌트 트리를 갖게되며, 이 많은 컴포넌트들이 정보를 표현하는데 필요로 하는 정보는 대개 페이지와 동일한 라이프사이클을 갖게되고 컴포넌트 트리 전역에서 접근할 수 있어야 한다.

이러한 상황에서 선택할 수 있는 컴포넌트들의 협력을 설계하는 몇가지 방법을 소개한다.

## Basic
기본적으로 UI계층은 Component와 Container로 구성된다.

### Component

Component는 최대한 상태를 가지는걸 지양하고 외부로 주입받은 상태 `props`에 의해 렌더링되며 사용자의 인터렉션에 대한 결과로 Container에게 `event`를 보내도록 순수하게 유지하는것을 지향한다.

즉, `UI = f(state)` 를 따르게 한다. f는 순수해야 하며, 항등성을 보장하도록 하고, state는 외부에서 주입받도록 한다.

사용자의 인터렉션에 대한 결과로 직접 다른 계층에 메시지를 전달해 side-effect를 발생시키는 것을 지양한다.

Component를 순수하게 유지할수록 테스트하기 쉽고, 재사용하기 쉬워진다.

> 하지만 이는 복잡한 컴포넌트 트리를 갖는 UI를 구현할 경우 모든 컴포넌트를 순수하게 구성하기란 현실적으로 많은 한계를 가진다. 복잡한 컴포넌트 트리를 갖는 UI를 구성할 때 Vue에서 활용 가능한 설계 전략은 이후에 설명한다.

### Container

Container는 Context의 단위이다. 하나의 Container에 포함된 Component들은 동일한 Context를 갖는다.

클라이언트 애플리케이션에서 가장 기본적인 Context의 단위는 하나의 페이지라고 할 수 있다.
따라서 Container의 기본 단위 또한 라우팅의 대상이 되는 페이지 전체의 Container이며 페이지의 Layout과 포함된 Component의 관계를 설정하는 책임을 갖는다.

Container는 UI에 관련된 상태를 가지며, 각 Component를 배치하여 Layout을 구성하고, Component끼리 서로 협력하는 UI에 관련된 로직을 처리한다.

필요에 따라 Domain 계층, Application 계층, Infra 계층에 메시지를 전달하여 side-effect를 발생시킬 수 있지만 직접 서비스 정책이나 업무 규칙에 대한 문제를 해결하는 것은 지양한다.

## 복잡한 UI 계층의 설계

페이지가 나타내야할 UI와 정보가 복잡할 경우, Component를 요구사항이 변경되었을 때 기능을 확장하고 변경하기 쉽도록 복잡함을 다루기 위해 최소한의 단위로 작게 나누게 된다.

때로는 하나의 Component가 여러 자식 Component를 갖고, 그 자식 Component가 또 다시 여러 자식 Component를 갖기도 한다.
이에 따라 각 페이지는 많은 단계의 계층을 가진 Component 트리를 갖게 된다.

또한 복잡한 UI의 상태를 관리하기 위한 로직들로 Container가 복잡해지기 시작한다.
깊은 계층의 Component 트리를 가진 Container는 읽기 어렵고 변경하기 어렵게 되는 결과로 이어진다.

이 글에서는 UI 계층을 설계할 때 선택할 수 있는 다음 네 가지 전략을 비교하여 소개한다.

1. Presentational and Container Components
2. Provide/Inject
3. Slots
4. Composition

### Presentational and Container Components

아마 많은 사람들이 익숙하게 받아들이고 있는 패턴으로 [Dan Abramov이 잘 정리한 포스트](https://medium.com/@dan_abramov)가 있다.

`Presentational and Container Components` 패턴은 Container와 Component를 나누고 Container에게 Context에 필요한 상태를 관리하는 책임을 부여한다.

Component는 `props`를 통해 Context에 관련된 필요한 상태를 주입받아 UI를 렌더링하고, 사용자의 인터렉션이 발생하면 이를 Container에게 `event`로 알린다.

Container는 Component로 부터 `event`를 받아 상태를 변경한다.

문제는 이 과정에서 Component의 복잡함을 낮추기 위하여 더 작은 단위의 Component들로 세분화 하였고 이로 인해 깊은 계층의 Component 트리를 갖게 되었을 경우이다.

트리 말단에 놓인 Component는 상태를 주입받기 위해 props drilling이 발생하고, 사용자의 인터렉션에 반응하여 side-effect를 발생시키기 위해선 이 순서의 역으로 거슬러 올라가 Container에게 event를 전달하거나, side-effect를 발생시키는 함수를 callback으로 주입받아야 할 것이다.

우선 많은 정보와 복잡한 UI를 표현해야 하는 페이지 PageA가 있다고 가정해보자.
경험적으로 이러한 PageA의 Container는 깊은 계층의 Component 트리를 갖게 된다.

PageA가 표현해야 하는 많은 정보들은 대부분 페이지와 동일한 라이프 사이클을 가지게 된다.
(물론 일부 개별적인 라이프 사이클을 갖는 정보들은 Store에서 전역적으로 관리되고 있을 것이다.)

다시 말해 페이지를 벗어나면 더 이상 의미 없는 정보를 가지며, 이 정보는 깊은 계층의 Component 트리는 페이지 전역에서 접근이 가능해야 한다.

<img width="845" alt="스크린샷 2022-04-23 오후 7 19 09" src="https://user-images.githubusercontent.com/16642635/164890449-e1a67080-d407-47aa-9299-ee412ab0bede.png">

페이지의 Context를 나타내는 `PageAContainer`는 페이지에 필요한 정보인 `follow`를 `Follow API`를 fetch하여 자신의 상탯값으로 관리한다.

가장 말단에 있는 컴포넌트 `ComponentABA`가 이 `follow` 중 `user`에 대한 정보를 렌더링한다면 다음과 같이 트리를 따라 가장 말단까지 props 를 전달해야 하는 prop drilling이 발생한다.

![:iijj 2](https://user-images.githubusercontent.com/16642635/164890759-83b08bde-2786-4076-ad37-6d11bebdeda7.png)

`ComponentABA`가 사용자의 인터렉션의 결과로 `user`에 관한 상탯값을 변경하고 싶다면 이를 역으로 거슬러 올라가 `PageAContainer`에게 메시지를 보내거나 `user`의 값을 변경할 수 있는 side-effect를 발생시키는 콜백을 prop으로 함께 전달해야 한다.

뿐만 아니라 Context의 상태를 관리하는 로직이 복잡해 질 경우 Container가 지나치게 복잡해지는 문제가 발생하게 된다.

pros:
- 컴포넌트로부터 상태를 관리하는 로직을 제거했기 때문에 컴포넌트는 `UI = f(state)`의 철학에 맞게 주어진 `state`로 UI를 구성하는 순수한 상태를 유지할 수 있다.
- 컴포넌트를 순수한 상태로 유지할 수 있기 때문에 재사용하기 쉽고 테스트하기 쉬운 컴포넌트를 작성할 수 있다.
- 컴포넌트의 복잡도를 많이 낮출 수 있다.

cons:
- prop drilling 으로 인해 prop을 아래로 전달하고 event를 위로 전파하는 기계적인 코드를 반복적으로 작성해야 한다.
- 컴포넌트가 순수하게 유지될 수 있도록 하기위해 side-effect를 일으킬 수 있는 함수를 `props`로 전달하거나 prop drilling의 역으로 이벤트를 전달받아야 한다.
- 상태 관리 로직이 복잡할경우 컨테이너가 가진 다른 관점의 책임들 (레이아웃을 구성하고, 자신의 자식 컴포넌트간의 관계를 설정하고, 자신의 자식 컴포넌트가 참여하는 flow 로직을 처리하는 등) 과 섞여 복잡해진다.

#### 더 이상 Container와 Component를 분리하지 말라고 하던대요?

Dan Abramov은 이제는 해당 역할을 hooks가 더 잘 수행해낼 수 있으므로 더 이상 무의미 하게 나누지 않는 편이 좋다고 설명하고 있지만, 개인적으로는 다음과 같은 이유들로 여전히 잘 사용하고 있다.

상태관련 복잡한 로직을 Container가 아닌 hooks (vue에는 동일한 개념의 composition api가 있다) 로 분리하더라도 Container에겐 **코드 베이스에 표현되는 Context의 단위**라는 의미있는 역할이 있다.

> 동일한 Context 내에서의 상태에 대한 관리를 hook으로 분리하는 부분은 바로 뒤에 소개한다.

클라이언트 애플리케이션에서 사람이 인지하는 기본적인 단위는 **페이지**이기 때문에 이를 Container의 기본적인 단위로 삼는다.

예를들어, 아래 페이지는 "여행 이력"라는 하나의 Context를 가지며 이 정보는 이 페이지에서만 유효하고 페이지와 그 라이프 사이클을 함께한다.

![제목_없는_아트워크 2](https://user-images.githubusercontent.com/16642635/164250503-f5dbe303-fe41-4f2d-a547-f53f726d6bb4.png)

코드 베이스엔 다음과 같이 표현한다.

<img width="628" alt="스크린샷 2022-04-20 오후 11 37 59" src="https://user-images.githubusercontent.com/16642635/164255807-c667514a-b4b4-48ec-b83f-27f52f492a2a.png">

모듈의 이름은 페이지 Container의 이름을 따 `my-trips`라는 이름을 부여하고 이 모듈 내부엔 `MyTripsPageContainer`와 페이지에 포함되는 Component들이 `component` 디렉토리에 페이지의 container와 component에서 사용하는 composition이 `composition` 디렉토리에 포함되어 있다.

하나의 페이지에 여러 Context가 존재하는 경우도 있다.

아래 페이지는 유저가 처음 랜딩하는 홈 화면이며, 숙소를 검색하기 위한 Context, 이벤트 베너를 노출하기 위한 Context, 유연한 검색 Context, 추천 지역들의 Context들이 포함되어 있다.

각 Context들은 하나의 페이지내에서 독립적인 라이프 사이클과 경계를 갖는다.

![제목_없는_아트워크 3](https://user-images.githubusercontent.com/16642635/164258418-aafea894-79bc-45f2-8f70-3a56adf78194.png)

코드 베이스엔 다음과 같이 표현한다.

<img width="585" alt="스크린샷 2022-04-24 오전 11 43 14" src="https://user-images.githubusercontent.com/16642635/164953853-5fb294f0-d045-4988-a072-f2481adcb2d8.png">

마찬가지로 모듈의 이름은 페이지 Container의 이름을 따 `home`라는 이름을 부여하고 이 모듈 내부엔 `HomePageContainer`와 각 Context를 각각의 모듈 `search-box`, `flex-search`, `recommend-area`, `recommend-activity` 로 구성한다.

`search-box` 모듈에는 `SearchBoxContainer`와 이 Context에 포함되는 Component들이 `component` 디렉토리에 포함되어 있으며 마찬가지로 페이지의 Container와 Component에서 사용중인 Composition들이 `composition` 디렉토리에 포함되어 있다.

정리하면, Container는 Context를 코드 베이스에서 표현하는 하나의 단위이자 경계이다.

Context별로 모듈을 나누고 Container를 마치 모듈의 엔트리포인트처럼 사용하여 개발자로 하여금 Context의 경계를 명확하게 인지하도록 한다.

> 결국 각 디렉토리에 해당하는 Container가 하나의 컴포넌트가 아닌가? 하는 생각이 들 수 있다. 맞다. 각 Container 디렉토리는 독립적으로 동작할 수 있는 하나의 컴포넌트이다. 디렉토리 내부에 포함된 Component와 Composition을 각 Container에 플랫하게 구현하면 그러한 모습이 될 것이다. 하지만 우리는 앞서 설명했듯 복잡함을 다루기위해 책임을 기준으로 최소한의 단위로 작게 나누어 관리하고 싶어한다. 이러한 의도로 작게 나누어진 Component와 분리된 Composition이 하나의 맥락이라는 경계를 제공하기 위해 사전적 의미 그대로의 Container를 두고 디렉토리로 코드 베이스에 표현한 것이다.

다양한 Context에서 공통으로 사용되는 `Button`, `List` 같은 요소들은 `atomic` 디렉토리를 두어 관리한다. 이 컴포넌트들은 최대한 순수하게 관리되어야 할 것이다.

<img width="588" alt="스크린샷 2022-04-23 오후 7 02 27" src="https://user-images.githubusercontent.com/16642635/164889895-f97d6472-a9ad-4a93-a536-14fb2339adbf.png">

### Provide/Inject
컨테이너-컴포넌트 구조를 유지하면서 prop drilling을 해결하기 위해 Vue의 `Provide/Inject`를 사용할 수 있다. 필요한 상태와 side-effect를 일으키는 콜백을 provide를 통해 주입받도록 하여 prop drilling을 해결한다.

> vue의 `Provide/Inject` 의 사용 방법 대해서는 [레퍼런스](https://vuejs.org/guide/components/provide-inject.html#provide)를 참고한다.

<img width="1045" alt="스크린샷 2022-04-23 오후 9 32 30" src="https://user-images.githubusercontent.com/16642635/164894620-22f5353d-4f80-4a00-8c3b-f1683d63fa0c.png">

`Provide/Inject`는 `Presentational and Container Components`의 prop drilling은 해결할 수 있지만, Context의 상태를 관리하는 책임을 Container가 지고 있기 때문에 이 로직이 복잡할 경우 Container가 함께 복잡해지는 문제는 여전히 발생하게 된다.

뿐만 아니라 Vue의 `provide/inject`는 코드에 명시적으로 드러나지 않고 컴포넌트 트리의 구성을 자세히 알아야만 하기 때문에 일종의 블랙박스처럼 다가온다.

pros:
- 컴포넌트로부터 상태를 관리하는 로직을 제거했기 때문에 컴포넌트는 `UI = f(state)`의 철학에 맞게 주어진 `state`로 UI를 구성하는 순수한 상태를 유지할 수 있다.
- 컴포넌트를 순수한 상태로 유지할 수 있기 때문에 재사용하기 쉽고 테스트하기 쉬운 컴포넌트를 작성할 수 있다.
- 컴포넌트의 복잡도를 많이 낮출 수 있다.
- 페이지(Context)에 필요한 상태에 대한 전역적인 접근 방법을 제공한다.

cons:
- provide-inject간 관계가 명시적으로 드러나지는 않기 때문에 코드 가독성이 저하될 수 있다.
- 상태 관리 로직이 복잡할경우 컨테이너가 가진 다른 관점의 책임들 (레이아웃을 구성하고, 자신의 자식 컴포넌트간의 관계를 설정하고, 자신의 자식 컴포넌트가 참여하는 flow 로직을 처리하는 등) 과 섞여 복잡해진다.

### Slots
또한 `Slots`를 사용한 컴포넌트 합성으로도 prop drilling를 해결 할 수 있다. 컴포넌트 구조가 반드시 중첩되어야 하는 구조인지, `Slots`를 활용해 합성 가능한 컴포넌트로 변경할 수 있는지를 검토하고 리팩터링 한다.

> vue의 `Slots` 의 사용 방법 대해서는 [레퍼런스](https://vuejs.org/guide/components/slots.html#slot-content-and-outlet)를 참고한다.

컴포넌트를 합성하여 근본적인 문제인 "깊은 계층의 컴포넌트 트리"를 개선할 수 있다.

![:iijj 3](https://user-images.githubusercontent.com/16642635/164895779-6e6796e6-bb2d-49ad-a7db-8bee1fe9de15.png)

계층의 깊이를 줄여 prop drilling을 해결할 수 있지만, `ComponentA`에게 새로운 관계 설정의 책임이 생기게 되며 `Slots`에 관련된 추가적인 코드들이 템플릿에 추가되어야 할 것이다.

또한 여전히 Container가 Context의 상태를 관리하는 책임을 지고 있기 때문에 Container가 복잡해지는 문제는 여전히 발생한다.

> 물론 `Slots`을 잘 활용하여 컴포넌트를 합성가능하도록 구성하면 높은 재사용성을 확보할 수 있는 좋은 패턴이지만, 현재 상황은 컴포넌트의 재사용성을 확보하기 위함이 아닌 컴포넌트의 복잡함을 다루기위해 작게 나눈 Context의 상태에 의존하는 컴포넌트들이다.

pros:
- 컴포넌트로부터 상태를 관리하는 로직을 제거했기 때문에 컴포넌트는 `UI = f(state)`의 철학에 맞게 주어진 `state`로 UI를 구성하는 순수한 상태를 유지할 수 있다.
- 컴포넌트를 순수한 상태로 유지할 수 있기 때문에 재사용하기 쉽고 테스트하기 쉬운 컴포넌트를 작성할 수 있다.
- 컴포넌트의 복잡도를 많이 낮출 수 있다.
- 컴포넌트의 계층을 줄여 prop drilling 을 해결하였다.

cons:
- `Slots`로 인한 템플릿에 추가적인 코드들이 필요하고 이 부분이 다소 복잡하게 느껴질 수 있다.
- 상태 관리 로직이 복잡할경우 컨테이너가 가진 다른 관점의 책임들 (레이아웃을 구성하고, 자신의 자식 컴포넌트간의 관계를 설정하고, 자신의 자식 컴포넌트가 참여하는 flow 로직을 처리하는 등) 과 섞여 복잡해진다.


### Composition API (hooks)
컴포넌트-컨테이너 구조에서 prop drilling과 컨테이너의 복잡함을 해결하기 위해 Context의 상태를 관리하는 책임을 `Composition API` 으로 구현한 `ContainerContext`에 위임하여 완전히 분리해낸다.

```javascript
/* usePageAContainerContext.js */
import { ref } from "vue";
import { onBeforeRouteLeave } from "vue-router";
import { getFollow } from "@api/follow-service";

const follow = ref({});
const user = ref({});

function setFollow(follow) {
  follow.value = follow;
}

function setUser(newUser) {
  user.value = newUser;
}

async function startContext() {
  const follow = await getFollow();
  follow.value = follow;
  user.value = follow.user;
}

function endContext() {
  follow.value = null;
  user.value = null;
}

export function usePageAContainerContext() {
  onBeforeRouteLeave(() => {
    endContext();
  });

  return {
    startContext,
    follow,
    setFollow,
    user,
    setUser,
  };
}
```

Container는 더 이상 상태관리의 책임을 지지 않으며, 생성시 `ContainerContext`에게 메세지를 전달하고 레이아웃을 구성하고, 자식 컴포넌트들의 관계를 설정하며 이들이 참여하는 flow 로직을 처리하는 책임을 갖는다.

```vuejs
<script setup>
/* PageAContainer */
import ComponentA from "./ComponentA.vue";
import ComponentB from "./ComponentB.vue";
import ComponentC from "./ComponentC.vue";
import { usePageAContainerContext } from "@/components/usePageAContainerContext";

const { startContext } = usePageAContainerContext();

startContext();
</script>
<template>
  <ComponentA />
  <ComponentB />
  <ComponentC />
</template>
```

`PageA`에 필요한 상태는 전부 `PageAContainerContext`에서 관리되며 각 컴포넌트들은 필요한 상태를 `PageAContainerContext`로부터 전달받고 메시지를 직접 전달하여 side-effect를 발생시킨다.

```vuejs
/* ComponentABA */
<script setup>
import { usePageAContainerContext } from "@/components/usePageAContainerContext";

const { user, setUser } = usePageAContainerContext();
</script>
<template>
  <div>
    <div>{{ user.name }}</div>
    <button @click="setUser({name: 'newUser'})"></button>
  </div>
</template>
```

이때 컴포넌트가 `PageAContainerContext`에 대한 의존성을 갖게 되어 더 이상 순수하지 않은 상태가 된다.

따라서 재사용성을 확보해야만 하기 때문에 순수하게 유지할 컴포넌트와 페이지 외의 다른 페이지에서 사용할 가능성이 없어 순수하게 유지할 필요는 없지만, 복잡함을 다루기 위해 작게 나눈 컴포넌트간의 구분이 필요하며 전자의 경우 `props`와 `event`를 통해 협력에 참여하게 하고, 후자의 경우 스토어를 통해 협력에 참여하게 한다.

> vue의 `Composition API` 의 사용 방법 대해서는 [레퍼런스](https://vuejs.org/guide/extras/composition-api-faq.html)를 참고한다.


<img width="1144" alt="스크린샷 2022-04-23 오후 10 23 13" src="https://user-images.githubusercontent.com/16642635/164896413-e00856e6-ed8e-48f8-a52b-c9be0ddaee1b.png">

더 이상 Container는 Context에 필요한 상태들을 관리하는 책임을 지지 않기 때문에 복잡함을 낮출 수 있게 되었지만, Component들이 `ContainerContext`에 대한 의존성을 갖게 되며 더 이상 순수하지 않은 상태가 되었다.

pros:
- 컨테이너의 복잡성을 낮출 수 있다.
- 컨테이너와 컴포넌트 모두에서 상태를 관리하기 위한 로직이 분리된다.
- 페이지(Context)에 필요한 상태에 대한 전역적인 접근 방법을 제공한다.

cons:
- 컴포넌트가 `ContainerContext`에 대한 의존성을 갖게 되어 순수한 컴포넌트가 주는 이점들이 사라지게 된다.

> 사실 평범한 `Composition`일 뿐이다. 다만 `Composition`에는 UI와 관련된 모든 논리가 담길 수 있고, 그 중 `Container의 Context와 관련된 상태들을 관리하는 책임`을 갖는 `Composition` 임을 강조하고 싶었다. 이름에서 알 수 있듯 최대 하나의 Container당 하나의 `ContainerContext`를 가질 수 있다. 따라서 최종적인 스캐폴딩은 다음과 같은 Container 단위 디렉토리가 중첩이 가능한 구조가 된다.

<img width="609" alt="스크린샷 2022-04-24 오전 11 48 22" src="https://user-images.githubusercontent.com/16642635/164953957-c6dde9f3-b6c5-46f7-822e-676d91f0302b.png">

> 생김새가 `Composition`과 `Store`를 합쳐놓은듯 한 느낌이 들지 않는가? 앞서 간단히 소개한 [Pina](https://pinia.vuejs.org/introduction.html)가 이러한 모습이다.

### 정리

> 여기서부터는 필자의 **아주아주아주아주 개인적인 생각**이 전개됩니다.

각 전략들을 정리하면 다음과 같다.

|전략|Container<br>복잡도|Component<br>순수 여부|코드 가독성|상태에 전역적인<br>접근 가능 여부|
|:---|:---|:---|:---|:---|
|Presentational and Container Components|😭 높음|😊 순수함|😊 단순함|😭 불가능|
|Provide/Inject|😭 높음|😊 순수함|😭 복잡함|😊 가능|
|Slots|😭 높음|😊 순수함|😭 복잡함|😭 불가능|
|Composition(ContainerContext)|😊 낮음|😭 순수하지 않음|😭 복잡함|😊 가능|

각 전략들이 서로 베타적인것은 아니다.

불필요한 컴포넌트 중첩을 막기 위하여 `Slots`을 활용해 합성 가능한 컴포넌트로 리팩터링하면서 Container의 복잡도를 낮추기 위해 `ContainerContext`를 사용할 수 있을 것이다.

놓여진 상황에 맞게, 문제가 되는 부분을 가장 잘 해결할 수 있는 전략을 트레이드 오프를 고려해 선택하면 된다.

하지만 전혀 문제를 어떻게 풀어야 할지 감이 안온다면 다음과 같은 접근방법을 추천한다.

1. 우선 `Presentational and Container Components`의 형태로 구현한다. 개인적으로는 협력이 가장 간단하게 구성되기 때문에 간단한 페이지의 경우 아직도 선호하는 전략이다. UI가 간단하다면 관리해야할 상태가 적기 때문에 Container의 복잡도를 염두할 필요가 없고, 렌더링할 컴포넌트 트리 또한 단순할 것이기 때문에 상태에 대한 전역적인 접근을 고려할 필요가 없다. Component를 순수하게 유지함에서 오는 이점과 심플한 설계가 주는 코드 가독성을 확보할 수 있다.
2. 상태를 관리하는데 필요한 로직이 복잡하여 Container가 지나치게 복잡해졌고, 컴포넌트 트리 또한 깊은 계층을 가지고 있어 트리 전역에서 접근을 필요로 한다면 상태를 관리하는 로직을 `ContainerContext`로 분리한다.
3. 상태 관리에 관련된 로직은 간단하나, 컴포넌트 트리가 깊은 계층을 가지고 있다면 `Slots`을 통해 합성 가능한 컴포넌트로 리팩터링을 하면 좋을 부분이 있을지 고려한다.
4. 그렇지 않을 경우 `Provide/Inject`를 통해 전역적인 접근 수단을 제공한다.

# 마치며

결국 정리하면 모든것을 Vue (React)를 통해 해결하려 하지 말라는 것이다. Vue는 UI를 위한 프레임워크이며 UI 에 대한 관심만을 가지게 할 때 애플리케이션은 변화에 유연하게 대응할 수 있게 된다.

안그래도 복잡한 UI를 처리하기 위해 충분히 복잡한 Vue Component 에서 UI 를 제외한 모든 관심을 Vue 외부의 계층으로 분리해낸다면 시장의 요구사항에 보다 발빠르게 대응할 수 있는 유연한 애플리케이션을 만드는데 도움이 될 것이다.

> 물론 모든것을 Vue를 통해 해결했음에도 복잡함이 문제가 되지 않는다면 구태어 별도의 계층으로 책임을 분리해내지 않는편이 좋다. 오히려 설계상의 복잡함이 문제로 작용할 것이다. 설계에 정답이란 없다. 문제에 맞게 트레이드 오프를 잘 고려하여 판단을 내리길 바란다.

막상 작성하고 보니 보는이로 하여금 `뭘 당연한 얘기를 이렇게 길게도 장황하게 써두었대` 하는 생각이 들지도 모르겠다는 느낌이 든다.

그래도 Vue 애플리케이션을 만들며 복잡함에 압도당하고 있는 누군가가 하나의 케이스 스터디로 이 글이 도움이 됐으면 하면 작은 바램을 가져본다.
