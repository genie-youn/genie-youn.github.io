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

즉, UI = f(state) 를 따르게 한다. f는 순수해야 하며, 항등성을 보장하도록 하고, state는 외부에서 주입받도록 한다.

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

이 글에서는 이러한 상황에서 선택할 수 있는 몇 가지 설계 전략을 비교하여 소개한다.

우선 많은 정보와 복잡한 UI를 표현해야 하는 페이지 PageA가 있다고 가정해보자.
경험적으로 이러한 PageA의 Container는 깊은 계층의 Component 트리를 갖게 된다.

PageA가 표현해야 하는 많은 정보들은 대부분 페이지와 동일한 라이프 사이클을 가지게 된다.
(물론 일부 개별적인 라이프 사이클을 갖는 정보들은 Store에서 전역적으로 관리되고 있을 것이다.)

다시 말해 페이지를 벗어나면 더 이상 의미 없는 정보를 가지며, 이 정보는 깊은 계층의 Component 트리는 페이지 전역에서 접근이 가능해야 한다.

> 하나의 페이지가 꼭 하나의 Context를 갖는건 아니다. 페이지는 더 작은 단위의 Context들의 집합으로 구성될 수 있으며, 이 경우 페이지 Container는 더 작은 Container로 구성되게 될 것이다.

이러한 상황에서 선택할 수 있는 패턴으로는 다음과 같이 총 4가지의 패턴이 있다.

1. Presentational and Container Components
2. Provide/Inject
3. Composition
4. Context

### Presentational and Container Components

아마 많은 사람들이 익숙하게 받아들이고 있는 패턴으로 [Dan Abramov이 잘 정리한 포스트](https://medium.com/@dan_abramov)가 있다.

Dan Abramov은 이제는 해당 역할을 hooks가 더 잘 수행해낼 수 있으므로 더 이상 무의미 하게 나누지 않는 편이 좋다고 설명하고 있지만, 개인적으로는 다음과 같은 이유들로 여전히 잘 사용하고 있다.

상태관련 복잡한 로직을 Container가 아닌 hooks (vue에는 동일한 개념의 composition api가 있다) 로 분리하더라도 Container에겐 "코드 베이스에 표현되는 Context의 단위"라는 의미있는 역할이 있다.

클라이언트 애플리케이션에서 사람이 인지하는 기본적인 단위는 "페이지"이기 때문에 이를 Container의 기본적인 단위로 삼는다.

예를들어, 아래 페이지는 "여행 이력"라는 하나의 Context를 가지며 이 정보는 이 페이지에서만 유효하고 페이지와 그 라이프 사이클을 함께한다.

![제목_없는_아트워크 2](https://user-images.githubusercontent.com/16642635/164250503-f5dbe303-fe41-4f2d-a547-f53f726d6bb4.png)

코드 베이스엔 다음과 같이 표현한다.

<img width="628" alt="스크린샷 2022-04-20 오후 11 37 59" src="https://user-images.githubusercontent.com/16642635/164255807-c667514a-b4b4-48ec-b83f-27f52f492a2a.png">

모듈의 이름은 페이지 Container의 이름을 따 `my-trips`라는 이름을 부여하고 이 모듈 내부엔 `MyTripsPageContainer`와 페이지에 포함되는 Component들이 `component` 디렉토리에 페이지에서 사용하는 composition이 `composition` 디렉토리에 포함되어 있다.

하나의 페이지에 여러 Context가 존재하는 경우도 있다.

아래 페이지는 유저가 처음 랜딩하는 홈 화면이며, 숙소를 검색하기 위한 Context, 이벤트 베너를 노출하기 위한 Context, 유연한 검색 Context, 추천 지역들의 Context들이 포함되어 있다.

각 Context들은 하나의 페이지내에서 독립적인 라이프 사이클과 경계를 갖는다.

![제목_없는_아트워크 3](https://user-images.githubusercontent.com/16642635/164258418-aafea894-79bc-45f2-8f70-3a56adf78194.png)

코드 베이스엔 다음과 같이 표현한다.

마찬가지로 모듈의 이름은 페이지 Container의 이름을 따 `home`라는 이름을 부여하고 이 모듈 내부엔 `HomePageContainer`와 각 Context를 각각의 모듈 `search-box`, `flex-search`, `recommend-area`, `recommend-activity` 로 구성한다.

`search-box` 모듈에는 `SearchBoxContainer`와 이 Context에 포함되는 Component들이 `components` 디렉토리에 포함되어 있다.

정리하면, Container는 Context를 코드 베이스에서 표현하는 하나의 단위이자 경계이다.

Context별로 모듈을 나누고 Container를 마치 모듈의 엔트리포인트처럼 사용하여 개발자로 하여금 Context의 경계를 명확하게 인지하도록 한다.

`Presentational and Container Components` 패턴은 Container와 Component를 나누고 Container에게 Context에 필요한 상태를 관리하는 책임을 부여한다.

Component는 props를 통해 Context에 관련된 필요한 상태를 주입받아 UI를 렌더링하고, 사용자의 인터렉션이 발생하면 이를 Container에게 event로 알린다.

Container는 Component로 부터 event를 받아 상태를 변경한다.

문제는 이 과정에서 Component의 복잡함을 낮추기 위하여 더 작은 단위의 Component들로 세분화 하였고 이로 인해 깊은 계층의 Component 트리를 갖게 되었을 경우이다.

트리 말단에 놓인 Component는 상태를 주입받기 위해 props drilling이 발생하고, 사용자의 인터렉션에 반응하여 side-effect를 발생시키기 위해선 이 순서의 역으로 거슬러 올라가 Container에게 event를 전달하거나, side-effect를 발생시키는 함수를 callback으로 주입받아야 할 것이다.


pros:
- 컴포넌트로부터 상태를 관리하는 로직을 제거했기 때문에 컴포넌트는 `UI = f(state)`의 철학에 맞게 주어진 `state`로 UI를 구성하는 순수한 상태를 유지할 수 있다.
- 컴포넌트를 순수한 상태로 유지할 수 있기 때문에 재사용하기 쉽고 테스트하기 쉬운 컴포넌트를 작성할 수 있다.
- 컴포넌트의 복잡도를 많이 낮출 수 있다.
cons:
- prop drilling 이 발생한다.
- 컴포넌트가 순수하게 유지될 수 있도록 하기위해 side-effect를 일으킬 수 있는 함수를 `props`로 전달하거나 prop drilling의 역으로 이벤트를 전달받아야 한다.
- 상태 관리 로직이 복잡할경우 컨테이너가 가진 다른 관점의 책임들 (레이아웃을 구성하고, 자신의 자식 컴포넌트간의 관계를 설정하고, 자신의 자식 컴포넌트가 참여하는 flow 로직을 처리하는 등) 과 섞여 복잡해진다.

### Provide/Inject
컨테이너-컴포넌트 구조를 유지하면서 prop drilling을 해결하기 위해 Vue의 Provide/Inject를 사용할 수 있다. 필요한 상태와 side-effect를 일으키는 콜백 provide를 통해 주입받으면 컨테이너-컴포넌트의 단점을 해결할 수 있다.

-- 이거 예제 vue3로 다시쓰기

PageAContainer.vue
```vue
<template>
  <div>
    <ComponentA />
    <ComponentB />
    <ComponentC />
  </div>
</template>

<script>
import ComponentA from "./components/ComponentA";
import ComponentB from "./components/ComponentB";
import ComponentC from "./components/ComponentC";
import { fetchStateA } from "@/api/awsome-api";
import { computed } from "vue";

export default {
  name: "PageAContainer",
  components: {
    ComponentA,
    ComponentB,
    ComponentC,
  },
  data() {
    return {
      stateA: {},
    };
  },
  created() {
    fetchStateA().then((res) => {
      this.stateA = res;
    });
  },
  provide() {
    return {
      B: computed(() => this.stateA.B),
    };
  },
};
</script>
```

ComponentABA.vue
```vue
<template>
  <span>{{ B }}</span>
</template>

<script>
export default {
  name: "ComponentABA",
  inject: ["B"],
};
</script>
```

<img width="838" alt="스크린샷 2021-12-16 오전 9 07 56" src="https://user-images.githubusercontent.com/16642635/146284227-50681d62-36cf-4e6e-b0e5-3c8694d231de.png">

pros:
- 컴포넌트로부터 상태를 관리하는 로직을 제거했기 때문에 컴포넌트는 `UI = f(state)`의 철학에 맞게 주어진 `state`로 UI를 구성하는 순수한 상태를 유지할 수 있다.
- 컴포넌트를 순수한 상태로 유지할 수 있기 때문에 재사용하기 쉽고 테스트하기 쉬운 컴포넌트를 작성할 수 있다.
- 컴포넌트의 복잡도를 많이 낮출 수 있다.
cons:
- provide-inject간 관계가 명시적으로 드러나지는 않기 때문에 코드 가독성이 저하될 수 있다.
- 상태 관리 로직이 복잡할경우 컨테이너가 가진 다른 관점의 책임들 (레이아웃을 구성하고, 자신의 자식 컴포넌트간의 관계를 설정하고, 자신의 자식 컴포넌트가 참여하는 flow 로직을 처리하는 등) 과 섞여 복잡해진다.

### Composition



###
컴포넌트-컨테이너 구조에서 prop drilling과 컨테이너의 복잡함을 해결하기 위해 페이지의 상태를 관리하는 책임을 페이지 스토어를 정의하고 이에 위임한다.

컨테이너는 더 이상 상태관리의 책임을 지지 않으며, 생성시 스토어에게 메세지를 전달하고 레이아웃을 구성하고, 자식 컴포넌트들의 관계를 설정하며 이들이 참여하는 flow 로직을 처리하는 책임을 갖는다.

`PageA`에 필요한 상태는 전부 `PageAStore`에서 관리되며 각 컴포넌트들은 필요한 상태를 `PageAStore`로부터 `getters`를 통해 접근하고 `actions`를 통하여 side-effect를 발생시킨다.

이때 컴포넌트가 스토어에 대한 의존성을 갖게 되어 더 이상 순수하지 않은 상태가 된다.

따라서 재사용성을 확보해야만 하기 때문에 순수하게 유지할 컴포넌트와 페이지 외의 다른 페이지에서 사용할 가능성이 없어 순수하게 유지할 필요는 없지만, 유지보수성을 확보하기 위해 컴포넌트화한 컴포넌트간의 구분이 필요하며 전자의 경우 `props`와 `event`를 통해 협력에 참여하게 하고, 후자의 경우 스토어를 통해 협력에 참여하게 한다.

--- 예제 다시

ContainerA.vue
```vue
<template>
  <div>
    <ComponentA />
    <ComponentB />
    <ComponentC />
  </div>
</template>

<script>
import { mapActions } from "vuex";
import ComponentA from "./components/ComponentA";
import ComponentB from "./components/ComponentB";
import ComponentC from "./components/ComponentC";

export default {
  name: "PageAContainer",
  components: {
    ComponentA,
    ComponentB,
    ComponentC,
  },
  created() {
    this.initialize();
  },
  methods: {
    ...mapActions("PageAStore", ["initialize"]),
  },
};
</script>
```

ComponentABA.vue
```vue
<template>
  <span>{{ B }}</span>
</template>

<script>
import { mapGetters } from "vuex";

export default {
  name: "ComponentABA",
  computed: {
    ...mapGetters("PageAStore", ["B"]),
  },
};
</script>
```

pros:
- 컨테이너의 복잡성을 낮출 수 있다.
- 컨테이너와 컴포넌트 모두에서 상태를 관리하기 위한 로직이 분리된다.
- 페이지에 필요한 상태에 대한 전역적인 접근 방법을 제공한다.
cons:
- 스토어는 기본적으로 특정 페이지를 위한 저장소가 아니고 애플리케이션 전체에서 전역적으로 접근이 필요한 상태들을 관리하는 저장소이다.
- 그렇기 때문에 페이지 단위로 라이프사이클을 관리할 수 없으며 글로벌 스토어가 이러한 페이지를 위한 스토어들이 모두 등록될 경우 스토어의 네임스페이스가 오염될 가능성이 높다.
- 이러한 단점은 컨테이너의 인스턴스에 페이지 스토어를 직접 import하여 별개의 접근자로 접근하도록 하면 해결할 수 있다. (`this.$pageStore = PageAStore`)

### Page Context

# 마치며

결국 정리하면 모든것을 Vue 를 통해 해결하려 하지 말라는 것이다. Vue는 UI를 위한 프레임워크이며 UI 에 대한 관심만을 가지게 할 때 애플리케이션은 변화에 유연하게 대응할 수 있게 된다.

안그래도 복잡한 UI를 처리하기 위해 충분히 복잡한 Vue Component 에서 UI 를 제외한 모든 관심을 Vue 외부의 계층으로 분리해낸다면 시장의 요구사항에 보다 발빠르게 대응할 수 있는 유연한 애플리케이션을 만드는데 도움이 될 것이다.

막상 작성하고 보니 보는이로 하여금 "뭘 당연한 얘기를 이렇게 길게도 장황하게 써두었대" 하는 생각이 들지도 모르겠다는 느낌이 든다.

그래도 Vue 애플리케이션을 만들며 복잡함에 압도당하고 있는 누군가가 하나의 케이스 스터디로 이 글이 도움이 됐으면 하면 작은 바램을 가져본다.



굳이 domain 계층에 안두고 application 에 둬도 된다.

vue component에 다 때려박으려 하지 말아봐라

요즘 클라에도 처리할 도메인 로직이 많다
