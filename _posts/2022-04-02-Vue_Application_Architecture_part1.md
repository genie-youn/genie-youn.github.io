---
layout: post
title: "Vue Application Architecture - 프론트앤드 애플리케이션에서의 설계, Infra (Part1)"
author: "genie-youn"
categories: journal
tags: [Vue, Front End, Architecture, DDD]
image: DSC04956.jpg
---

> 해당 게시글엔 필자의 지극히 주관적인 의견들이 가득합니다. 설계엔 정답이 없다고 생각합니다. 다만 이 글에서 소개하는 설계가 필자가 해결하고자 하는 문제에 대해 꽤 효과적이었고, 하나의 케이스로 비슷한 문제를 당면한 분들께 참고할만한 부분이 있을까 싶어 글로 정리합니다.

지난 3년간 서비스의 jsp로 구성되어 있는 프론트앤드를 Vue 애플리케이션으로 점진적으로 전환하는 일을 하였다.
Vue 애플리케이션의 페이지는 점점 늘어나 백여개를 넘어섰고, 이를 구성하는 Vue 컴포넌트 또한 천여개가 훌쩍 넘어섰다.
애플리케이션의 규모가 커질수록 복잡함은 기하급수적으로 증가했다.

FE 팀의 규모도 따라서 함께 커졌는데, 새로 합류하는 동료들은 커다란 코드 베이스와 설계에 대한 기준이 없음에 혼란함을 느꼈다.
프로젝트 초기부터 함께 했던 멤버들이 코드 리뷰를 통해 학습하며 가지고 있던 설계에 대한 암묵지들을 문서로 꺼내어 팀 구성원 전체가 동일한 시각으로 코드 베이스 전체를 바라볼 수 있어야 한다고 생각했다.

설계 가이드 문서를 작성하면서 복잡한 Vue 애플리케이션을 다루기 위해 적지 않은 고민을 하였고,
그 결과로 **개인적으로** 정의한 Vue 애플리케이션의 설계에 관련된 몇 가지 기준들을 나와 비슷한 고민을 하는 프론트엔드 개발자들에게 도움이 되길 바라며 소개한다.

# Table of Contents
1. 복잡함을 해결해야 하는 이유
2. Infra
2. [API Client - Part2]() (작성중..)
3. [Domain - Part3]() (작성중..)
4. [Application - Part4]() (작성중..)
5. [UI - Part5]() (작성중..)
6. [마치며 - Part5]() (작성중..)

# 복잡함을 해결해야하는 이유

개발하고 있는 프론트엔드 애플리케이션의 복잡함에 압도당해본 적이 있는가?

```
새로운 기능을 추가하거나, 기존 기능에 대한 개선을 요청받아서 관련된 코드를 열었는데 갑자기 숨이 턱 막힌다거나...
천라인이 넘어가는 `if` 로 점철된 컴포넌트를 만났다거나...
분명 5분 전에 배포한 건 서버 API인데 내 애플리케이션이 동작하지 않게 되었다거나..
브라우저의 네트워크 탭에 찍힌 응답 값은 분명 멀쩡한데 애플리케이션 내에서 돌고 돌아 갑자기 값이 뒤바뀌었다거나...
갑자기 옆 동료가 Git Blame 을 열더니 나를 째려보고 있다거나..
```

애플리케이션의 복잡해짐으로써 생기는 가장 큰 문제는 더 이상 변경에 유연하게 대응할 수 없다는 것이다.
시장은 빠르게 변화하고 우리가 사용자에게 제공하는 서비스는 이에 발맞춰 빠르게 변화할 수 있어야 한다.

하지만 복잡해진 애플리케이션은 이러한 시장의 요구사항에 기민하게 반응할 수 없다.
기획자나 디자이너에게 "그건 좀 어렵겠는데요."라고 얘기하거나 "이건 공수가 꽤 필요하겠는데요."라고 이야기 하는 빈도가 잦아진다.

변화에 뒤처진 애플리케이션은 더 이상 사용자에게 유쾌한 경험을 주지 못하게 되고 서비스의 매력을 잃게 만들어 결국 시장에서 도태되게 만든다.
따라서, 개발자는 시장의 요구사항을 빠르게 수용할 수 있도록 애플리케이션의 **복잡함**을 관리해야만 한다.

> 만약 위와 같은 문제를 느낀 적이 없다면, 굳이 이 글을 더 이상 읽을 필요가 없다. 설계는 문제를 해결하기 위한 의사결정이고 반드시 트레이드오프가 따르게 된다. 현재 구조로도 충분히 새로운 기능을 추가하기 쉽고 기존 기능을 변경하는 게 수월하다면 그게 내가 만들고자 하는 애플리케이션에 최적화된 설계일 것이다.

# 내가 만든 애플리케이션이 복잡한 이유

복잡함이 왜 문제인지는 충분히 얘기한 것 같으니, 그럼 내가 만든 애플리케이션이 **왜** 복잡해졌는지를 이야기해볼까 한다.
애플리케이션이 복잡해지는 데는 여러 이유가 있겠으나 나의 경우는 결국 **책임**이 문제였다.

Vue 애플리케이션 본질적인 역할은 상태에 따른 UI를 렌더링하고 UI와 사용자 간의 인터렉션을 처리하며 이 인터렉션에 따라 상태를 업데이트하고 업데이트된 상태에 맞게 UI를 다시 렌더링하는 것이다.

요즘 서비스들은 사용자들에게 편리한 UI/UX를 제공하기 위해 끊임없이 고민하고 새로운 접근방식을 시도하기 때문에 위와 같은 메커니즘은 그 자체만으로도 많은 복잡성을 가지게 되고 변경 주기도 짧다.

문제는 그 자체만으로도 복잡한 이 메커니즘에 API를 통해 서버와 메시지를 주고받고, 클라이언트에서 처리해야 하는 서비스의 정책이나 업무 규칙까지 끌어안아 괴물이 되어버린 것이다.
우선 **"UI와 관련된 책임들과 그렇지 않은 것들을 명확하게 분리해내야겠다."**라는 생각이 들었다.

기본적으로 Layered Architecture를 채용해 계층별로 큰 틀의 책임을 부여하였다. UI, Application, API Client & Domain, Infra 네 개의 계층으로 구성된다.

<img width="1024" alt="스크린샷 2022-04-19 오후 9 05 27" src="https://user-images.githubusercontent.com/16642635/163999425-3bddd4a8-3656-4f1e-bb3c-6587c24cae56.png">

결론부터 얘기하면 UI와 관련이 없는 책임들은 모두 다른 계층으로 분리해내고, Vue는 오로지 UI와 관련된 책임만을 갖도록 한다.
UI, Application 계층은 Vue로 구현되지만 API Client & Domain과 Infra 는 Vue가 개입하지 않고 순수 자바스크립트 (혹은 타입스크립트)로 구현된다.

> 왜 Layered Architecture인가? 라고 묻는다면 사실 큰 이유는 없다. 필자가 그나마 가장 잘 이해하고 있는 애플리케이션 아키텍처이고 아마 동료들도 비슷할 것이라 생각했다. 서비스를 혼자 개발하는 것이 아닌 개발조직의 규모가 어느정도 꽤 있는 편이었기 때문에 모두가 잘 알고 있는 아키텍처를 채택해야 조직원 모두가 동일한 관점으로 전체 시스템을 바라보는 시각을 공유할 수 있을 것이라 판단했다.

가장 하위 계층인 Infra 계층부터 살펴보도록 하자.

# Infra

애플리케이션이 동작하기 위한 기반 기술을 제공하는 계층이다. 프론트엔드 애플리케이션에서 HTTP 요청과 응답에 관한 책임을 갖는 모듈이나 로컬스토리지, 세션 스토리지와 같은 저장소에 접근하여 데이터를 영속화하는 책임을 갖는 모듈들이 이 계층에 속한다.

왜 이러한 모듈들을 별도 계층으로 격리해야 하는 걸까?

## 기반 기술의 구체적인 구현체는 "반드시" 변경해야 하는 순간이 온다.

이 계층의 모듈들은 그 특성상 애플리케이션 전반에서 광범위하게 의존하게 된다. 예를 들어 HTTP 요청과 응답에 관한 기반 기술의 구현체로 `axios`를 사용 중이라고 하자. 이 `axios`라는 구체적인 구현체를 별다른 격리 없이 다음과 같이 사용하였다.

```vue
<script setup>
import axios from "axios";

const action = () => {
  return axios.get("@/api/v1/my-awesome-action");
};
</script>
<template>
  <button @click="action">action!</button>
</template>
```

<img width="520" alt="스크린샷 2022-04-19 오후 9 13 03" src="https://user-images.githubusercontent.com/16642635/164000680-df12326c-4ad4-43f8-a060-98d800ac86fb.png">

그러던 어느 날 세상에 없던 멋진 HTTP Client 라이브러리가 짜잔 하고 등장하였다.
합리적인 알고리즘으로 요청과 응답을 멋지게 캐시 해주고 그 결과 요청의 수도 획기적으로 줄일 수 있고 응답속도 또한 빨라진다고 한다.

프로젝트에 당장 적용하고 싶어졌다.
`axios`를 걷어내고 이 멋진 라이브러리로 대체하리라.
한데 위와 같이 `axios`에 직접 메시지를 전달하는 컴포넌트가 수백개쯤 된다면?

아마 저 멋진 라이브러리는 아직 스타도 몇 개 없고 안정성도 검증되지 않았다며 신 포도라고 여겼을 것이다.

<img width="554" alt="스크린샷 2022-04-19 오후 9 21 59" src="https://user-images.githubusercontent.com/16642635/164002482-cb741647-7733-44ab-8a3b-78392ad0e7c8.png">

필자가 개발하고 있는 서비스는 날짜/시간대 관련된 처리를 `moment.js`에 위임하고 있었다.
하지만 굴지의 `moment.js`는 더 이상 신규 개발 없이 유지보수만 하는 레거시 프로젝트로 전환하였고 이에 따라 새로운 라이브러리로 교체해야만 했다.
만약 이 `moment.js` 에 직접 의존하게 설계했다면 모든 사용처를 찾아 새로운 라이브러리에 맞게 변경해주었어야 했을 것이다.

구현을 제어할 수 없는 외부 라이브러리라면, 추상화된 인터페이스를 정의하고 상위계층에선 이 인터페이스를 의존하게 하여 변경시의 영향도를 최소화 할 수 있다.

우선 infra 계층에 해당하는 `lib` 디렉토리를 만들고 해당 기반 기술을 추상화하여 인터페이스를 정의한다.

```typescript
/* @/lib/date-time/DateTime.ts */
export default interface DateTime {
  isSameDay(dateLeft: Date, dateRight: Date): boolean;
  addDays(date: Date, amount: number): Date;
  // more..
}
```

그리고 구체적인 구현체를 이 인터페이스에 맞게 조정한다.

이때 `Adapter`패턴을 활용할 수 있다.

> [어답터 패턴](https://ko.wikipedia.org/wiki/%EC%96%B4%EB%8C%91%ED%84%B0_%ED%8C%A8%ED%84%B4)

```typescript
/* @/lib/date-time/MomentAdapter.ts */
import moment from "moment";
import type DateTime from "./DateTime.js";

export default class MomentAdapter implements DateTime {
  isSameDay(dateLeft: Date, dateRight: Date): boolean {
    return moment(dateLeft).isSame(dateRight, "days");
  }
  addDays(date: Date, amount: number): Date {
    return moment(date).add(amount, "days").toDate();
  }
  // more..
}
```

모듈 외부에선 굳이 이러한 구체적인 내용을 알 필요가 없다.
`index.js`는 여러모로 실패한 디자인이라고 많이 이야기하지만 개인적으로는 이럴 때 요긴하게 쓰고 있다.

```typescript
/* @/lib/date-time/index.ts */
import type DateTime from "./DateTime.js";
import Adapter from "./MomentAdapter.js";

const instance: DateTime = new Adapter();

export function isSameDay(dateLeft: Date, dateRight: Date): boolean {
  return instance.isSameDay(dateLeft, dateRight);
}
export function addDays(date: Date, amount: number): Date {
  return instance.addDays(date, amount);
}
// more..
```

모듈 외부에서 접근하는 엔트리포인트라고 할 수 있는 `index.ts`에 구체적으로 어떤 구현체를 사용할지에 대한 책임을 부여한다.

사용하는 쪽에서는 기존에 사용하던 다른 모듈들을 사용할 때와 동일하게 사용한다.

```javascript
import {isSameDay} from "@/lib/date-time";

const isEditable = isSameDay(new Date(), article.createdDate);
```

<img width="528" alt="스크린샷 2022-04-19 오후 9 23 11" src="https://user-images.githubusercontent.com/16642635/164002692-dadc812b-054b-42e5-918c-2de652bda174.png">

날짜/시간대에 대한 일반적 기술의 구현체를 `date-fns`로 변경한다고 한다면 `date-fns`를 사용해 `DateTime`의 인터페이스를 구현하는 `DateFnsAdapter`를 구현한뒤

```typescript
import {addDays, isSameDay} from "date-fns";
import type DateTime from "@/libs/date-time/DateTime";

export default class DateFnsAdapter implements DateTime {
  addDays(date: Date, amount: number): Date {
    return addDays(date, amount);
  }
  isSameDay(dateLeft: Date, dateRight: Date): boolean {
    return isSameDay(dateLeft, dateRight);
  }
}
```

`index.ts`가 `DateFnsAdapter`를 사용하도록 변경한다.

```typescript
import type DateTime from "./DateTime.js";
import Adapter from "./DateFnsAdapter.js";

const instance: DateTime = new Adapter();

export function isSameDay(dateLeft: Date, dateRight: Date): boolean {
  return instance.isSameDay(dateLeft, dateRight);
}
export function addDays(date: Date, amount: number): Date {
  return instance.addDays(date, amount);
}
```

<img width="513" alt="스크린샷 2022-04-19 오후 9 30 35" src="https://user-images.githubusercontent.com/16642635/164003907-ad6dbfea-e28f-4d3d-9c00-d24f00376e5b.png">

기반 기술 중 서버와 HTTP를 통해 메시지를 주고받는 책임을 갖는 HTTP Client의 경우에 특별히 더 신경을 써줄 부분이 있다.

우선 동일하게 HTTP 요청/응답에 대한 인터페이스 `HTTPClient`를 정의한다.

```typescript
/* @/lib/http-client/HTTPClient.ts */
export default interface HTTPClient {
  get(url: string): Promise<unknown>;
}

export type Filter = (body: Record<string, unknown>) => Record<string, unknown>;

export interface HTTPClientBuilder {
  setBaseUrl(url: string): HTTPClientBuilder;
  setSuccessFilter(filter: Filter): HTTPClientBuilder;
  setFailFilter(filter: Filter): HTTPClientBuilder;
  build(): HTTPClient;
}
```

> 예제의 단순화를 위해 `get`메소드만 구현하였다. `Builder`의 경우 이외에도 헤더를 설정하거나 재시도 전략을 주입하는 등 다양한 팩터리 메소드들을 제공할 수 있을 것이다.

개발팀 내부에서 정의한 응답 포맷이 있을 테고 기본적으로 이 포맷에 맞춰 응답을 벗겨낸 후 반환하게 될 테지만, 간혹 외부 서비스의 API를 요청하는 경우도 종종 있다.
이러한 상황을 대비해 사용하는 쪽에서 이 전략을 선택할 수 있는 여지를 주면 좋다.

> 위 예제에서 `setSuccessFilter`와 `setFailFilter`가 그러한 역할을 수행하는데 자세한 내용은 이후 설명하겠다.

HTTP 요청을 주고받는 구체적인 구현체로 `fecth`를 사용한다면 마찬가지로 다음과 같이 `Adapter`를 구현할 수 있다.

```typescript
/* @/lib/http-client/FetchClient.ts */
import type HTTPClient from "./HTTPClient.js";
import type {HTTPClientBuilder, Filter} from "./HTTPClient.js";

const identifier: Filter = (_) => _;

export default class FetchClient implements HTTPClient {
  private _baseUrl = "";
  private _successFilter = identifier;
  private _failFilter = identifier;

  set baseUrl(url: string) {
    this._baseUrl = url;
  }
  set failFilter(filter: Filter) {
    this._failFilter = filter;
  }
  set successFilter(filter: Filter) {
    this._successFilter = filter;
  }

  get(url: string): Promise<unknown> {
    return fetch(`${this._baseUrl}${url}`, {
      method: "GET",
      credentials: "include",
    }).then((res) => {
      return res.status !== 200
        ? res.json().then((body) => this._failFilter(body))
        : res.json().then((body) => this._successFilter(body));
    });
  }
}

export class FetchClientBuilder implements HTTPClientBuilder {
  private readonly instance: FetchClient;

  constructor() {
    this.instance = new FetchClient();
  }

  setBaseUrl(url: string): HTTPClientBuilder {
    this.instance.baseUrl = url;
    return this;
  }

  setFailFilter(filter: Filter): HTTPClientBuilder {
    this.instance.failFilter = filter;
    return this;
  }

  setSuccessFilter(filter: Filter): HTTPClientBuilder {
    this.instance.successFilter = filter;
    return this;
  }

  build(): HTTPClient {
    return this.instance;
  }
}
```

> 마찬가지로 예제의 단순화를 위해 query나 여타 옵션들에 대한 구현은 생략하였다.

구체적으로 어떤 구현체를 사용할지에 대한 책임은 모듈의 `index.ts`에 부여한다.

만약 개발팀 내부에서 API 응답 포맷을 정상응답일 경우는 다음과 같이 정의하고,

```json
{
  "result": {
    // 응답
  }
}
```

실패의 경우

```json
{
  "error": {
    "errorCode": "에러코드",
    "message": "에러메세지",
    "appendedInfo": {
      // 예외 관련 응답
    }
  }
}
```

으로 정의했다면 `body => body.result`와 `body => Promise.reject(body.error)`를 기본 `Filter` 로 부여하여 약속대로 응답을 벗겨내도록 한다.

```typescript
/* @/lib/http-client/index.ts */
import type {HTTPClientBuilder} from "./HTTPClient.js";
import {FetchClientBuilder} from "./FetchClient.js";

export function createHttpClient(): HTTPClientBuilder {
  const builder: HTTPClientBuilder = new FetchClientBuilder();

  return builder
    .setSuccessFilter((body: any) => body.result)
    .setFailFilter((body: any) => Promise.reject(body.error));
}
```

> 동일한 맥락으로 서버와 서로 주고받기로 약속된 헤더들을 미리 설정해준다거나 할 수 있을 것이다.

만약 어느 날 세상에 없던 멋진 HTTP Client 라이브러리가 짜잔 하고 등장한다면 `HTTPClient`의 인터페이스를 만족하는 `Adapter`를 구현한 뒤, `index.ts`에서 이를 사용하게 하면 별다른 모듈 외부의 없이 새로운 라이브러리로 변경할 수 있다.

간혹 다음과 같이 서버 API 응답 구조를 비즈니스 로직에까지 노출하는 경우가 있는데,

```javascript
// axios의 응답 스키마 (response.data)와 팀 내부에서 정의한 API 응답 포맷 (result) 전부 비즈니스 로직까지 노출되었다.
const user = (await axios.get("/users")).data.result;

try {
  await axios.post("/update-users", user);
} catch (error) {
  // axios의 에러 응답 스키마 (error.response.data) 와 팀 내부에서 정의한 API 에러 응답 포맷 (error) 그리고 API의 에러코드 (1234) 가 모두 비즈니스 로직까지 노출되었다.
  if (error.response.data.error.errorCode === "1234"); // Do Something..
  if (error.response.data.error.errorCode === "1235"); // Do Another..
}
```

> fetch보다 axios의 응답 인터페이스가 더욱 극단적인 효과가 있는 것 같아 뜬금없는 느낌이 들지만 axios로 예를 들었다. (fetch의 응답 구조는 어쨌든 표준이니까..)

이렇게 될 경우 구태여 기반 기술의 구체적인 내용을 캡슐화해둔 의미가 사라진다.
`HTTPClient`를 다른 라이브러리로 변경하거나, 서버 API 응답 포맷이나 에러코드가 변경된 경우 의존하는 모든 부분을 찾아 바꿔주어야 할 것이다.

기술과 관련된 구체적인 사항들은 숨기고, 사용하는 쪽에서 관심 있는 응답이 나타내는 개념만을 상위계층으로 노출하도록 하자.

```vue
<script setup>
import {createHttpClient} from "@/lib/http-client";

const instance = createHttpClient()
  .setBaseUrl("https://my-awesome-api")
  .build();

const action = async () => {
  // 기술과 관련된 구체적인 사항들은 숨기고, 사용하는 쪽에서 관심있는 "user" 를 바로 반환한다.
  const user = await instance.get("/v1/users");
  // ...
};
</script>
<template>
  <button @click="action">action!</button>
</template>
```

하지만 위 예제처럼 HTTP Client를 생성하고 특정 API를 호출해하는 일련의 과정들도 UI와는 전혀 상관없는 부분들이다.

이러한 관심은 이후 설명할 `Domain & API CLIENT` 계층에서 처리하도록 위임한다.

## 정리

정리하면, 기반 기술의 구체적인 구현체는 언제든지 변경될 수 있다.
특히 그 대상이 내가 제어할 수 없는 영역이라면 별도의 계층으로 분리하여 격리한다.

이 대상으로는 오픈소스 라이브러리나, 사내 다른 부서에서 제공하는 SDK들이 있다.
기반 기술에 대한 인터페이스를 자체적으로 정의하고 이 인터페이스와 사용하려는 구체적인 구현체 사이의 `Adapter`를 구현한다.

인터페이스를 자체적으로 정의하기 어렵다면 사용하려는 구체적인 구현체의 인터페이스가 충분히 보편적인지, 우리의 문제를 해결할 수 있는지 확인하고 이를 사용하는 것도 좋은 방법이다.

구체적으로 어떤 구현체를 사용할지에 대한 책임은 모듈의 `index.js` 에 부여하고 모듈 외부와 상위 계층에선 이 인터페이스에 의존하도록 한다.

이렇게 낮춘 결합도는 추후 갑자기 사용 중인 라이브러리가 유지보수 단계에 돌입하거나, 협업 부서에서 새로운 SDK를 도입할 테니 변경에 협조해달라는 메일을 받았을 때 안도의 한숨을 내쉴 수 있게 할 것이다.

**pros:**
- 애플리케이션 전반에서 의존하는 일반적인 기술의 구현체가 변경되었을 경우 모듈 외부에 별다른 수정 없이 변경이 가능하다.
- 기반 기술의 구체적인 사항이 별도의 계층으로 격리되어 UI, Application 계층의 복잡도를 낮출 수 있다.

**cons:**
- 기반 기술의 인터페이스를 정의하는 비용을 지불해야 한다.
- 협력에 Wrapper 클래스가 추가되어 구조가 보다 복잡하게 느껴질 수 있다.

현재까지의 스캐폴딩이다.

<img width="619" alt="스크린샷 2022-04-19 오후 10 01 02" src="https://user-images.githubusercontent.com/16642635/164009874-9e7008a2-7520-4ecf-81ac-0b9750a2cf22.png">
