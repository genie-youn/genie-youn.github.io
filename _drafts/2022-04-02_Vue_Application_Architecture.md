---
layout: post
title: "Vue Application Architecture (Part1)"
author: "genie-youn"
categories: journal
tags: [Vue, Front End, Architecture, DDD]
image: DSC04956.jpg
---

> 해당 게시글엔 필자의 지극히 주관적인 의견들이 가득합니다. 설계엔 정답이 없다고 생각합니다. 다만 이 글에서 소개하는 설계가 필자가 해결하고자 하는 문제에 대해 꽤나 효과적이었고, 하나의 케이스로 비슷한 문제를 당면한분들께 참고할만한 부분이 있을까 싶어 글로 정리합니다.

수백개의 페이지와 이를 구성하는 천여개의 컴포넌트로 이루어진 Vue 애플리케이션을 3년간 개발하면서 기하급수적으로 늘어나는 애플리케이션의 복잡함을 다루기 위해 나름의 설계를 찾으려 애썼다.

여러 고민 끝에 몇 가지 원칙과 설계에 관한 계층 구조를 정의하였고 나의 이러한 고민이 비슷한 고민을 하는 프론트엔드 개발자들에게 조금이나마 도움이 되길 바라며 소개해본다.

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

우선 "UI와 관련된 책임들과 그렇지 않은 것들을 명확하게 분리해내야겠다."라는 생각이 들었다.

기본적으로 Layered Architecture를 채용해 계층별로 큰 틀의 책임을 부여하였다. UI, Application, API Client & Domain, Infra 네 개의 계층으로 구성된다.

계층 다이어그램 넣어주기

> 왜 Layered Architecture인가? 라고 묻는다면 사실 큰 이유는 없다. 필자가 그나마 가장 잘 이해하고 있는 애플리케이션 아키텍처이고 아마 동료들도 비슷할 것이라 생각했다. 서비스를 혼자 개발하는 것이 아닌 개발조직의 규모가 어느정도 꽤 있는 편이었기 때문에 모두가 잘 알고 있는 아키텍처를 채택해야 조직원 모두가 동일한 관점으로 전체 시스템을 바라보는 시각을 공유할 수 있을 것이라 판단했다.

가장 하위 계층인 Infra 계층부터 살펴보도록 하자.

# Infra

애플리케이션이 동작하기 위한 기반 기술을 제공하는 계층이다. 클라이언트 애플리케이션에서 HTTP 요청과 응답에 관한 책임을 갖는 모듈이나 로컬스토리지, 세션 스토리지와 같은 저장소에 접근하여 데이터를 영속화하는 책임을 갖는 모듈들이 이 계층에 속한다.

왜 이러한 모듈들을 별도 계층으로 격리해야 하는 걸까?

## 기반 기술의 구체적인 구현체는 "반드시" 변경해야 하는 순간이 온다.
이 계층의 모듈들은 그 특성상 애플리케이션 전반에서 광범위하게 의존하게 된다. 예를 들어 HTTP 요청과 응답에 관한 기반 기술의 구현체로 `axios`를 사용 중이라고 하자. 이 `axios`라는 구체적인 구현체를 별다른 격리 없이 다음과 같이 사용하였다.

AwesomeComponent.vue
```vue
<script setup>
import axios from "axios";

const action = () => {
  return axios.get("/apis/v1/my-awesome-action");
};
</script>
<template>
  <button @click="action">action!</button>
</template>
```

그러던 어느 날 세상에 없던 멋진 HTTP Client 라이브러리가 짜잔 하고 등장하였다.

합리적인 알고리즘으로 요청과 응답을 멋지게 캐시 해주고 그 결과 요청의 수도 획기적으로 줄일 수 있고 응답속도 또한 빨라진다고 한다.

프로젝트에 당장 적용하고 싶어졌다.

`axios`를 걷어내고 이 멋진 라이브러리로 대체하리라.

한데 위와 같이 `axios`에 직접 메시지를 전달하는 컴포넌트가 수백개쯤 된다면?

아마 저 멋진 라이브러리는 아직 스타도 몇 개 없고 안정성도 검증되지 않았다며 신 포도라고 여겼을 것이다.

필자가 개발하고 있는 서비스는 날짜/시간대 관련된 처리를 `moment.js`에 위임하고 있었다.

하지만 굴지의 `moment.js`는 더 이상 신규 개발 없이 유지보수만 하는 레거시 프로젝트로 전환하였고 이에 따라 새로운 라이브러리로 교체해야만 했다.

만약 이 `moment.js` 에 직접 의존하게 설계했다면 모든 사용처를 찾아 새로운 라이브러리에 맞게 변경해주었어야 했을 것이다.

구현을 제어할 수 없는 외부 라이브러리라면, 추상화된 인터페이스를 정의하고 상위계층에선 이 인터페이스를 의존하게 하여 변경시의 영향도를 최소화 할 수 있다.

우선 해당 기반 기술을 추상화하여 인터페이스를 정의한다.

/libs/date-time/DateTime.ts
```typescript
export default interface DateTime {
  isSameDay(dateLeft: Date, dateRight: Date): boolean;
  addDays(date: Date, amount: number): Date;
  // 기타 등등..
}
```

그리고 구체적인 구현체를 이 인터페이스에 맞게 조정한다.

이때 `Adapter`패턴을 활용할 수 있다.

> https://ko.wikipedia.org/wiki/%EC%96%B4%EB%8C%91%ED%84%B0_%ED%8C%A8%ED%84%B4

/libs/date-time/MomentAdapter.ts
```typescript
import moment from "moment";
import type DateTime from "./DateTime";

export default class MomentAdapter implements DateTime {
  isSameDay(dateLeft: Date, dateRight: Date): boolean {
    return moment(dateLeft).isSame(dateRight, "days");
  }
  addDays(date: Date, amount: number): Date {
    return moment(date).add(amount, "days").toDate();
  }
  // 기타 등등..
}
```

모듈 외부에선 굳이 이러한 구체적인 내용을 알 필요가 없다.
`index.js`는 여러모로 실패한 디자인이라고 많이 이야기하지만 개인적으로는 이럴 때 요긴하게 쓰고 있다.

/libs/date-time/index.ts
```typescript
import type DateTime from "./DateTime";
import Adapter from "./MomentAdapter";

const instance: DateTime = new Adapter();

export function isSameDay(dateLeft: Date, dateRight: Date): boolean {
  return instance.isSameDay(dateLeft, dateRight);
}
export function addDays(date: Date, amount: number): Date {
  return instance.addDays(date, amount);
}
// 기타 등등..
```

모듈 외부에서 접근하는 엔트리포인트라고 할 수 있는 `index.ts`에 구체적으로 어떤 구현체를 사용할지에 대한 책임을 부여한다.

사용하는 쪽에서는 기존에 사용하던 다른 모듈들을 사용할 때와 동일하게 사용한다.

AwesomeComponent.vue
```javascript
import {isSameDay} from "@/libs/date-time";

const isEditable = isSameDay(new Date(), article.createdDate);
```

기반 기술 중 서버와 HTTP를 통해 메시지를 주고받는 책임을 갖는 HTTP Client의 경우에 특별히 더 신경을 써줄 부분이 있다.

우선 동일하게 HTTP 요청/응답에 대한 인터페이스 `HTTPClient`를 정의한다.

/libs/http-client/HTTPClient.ts
```typescript
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

위 예제에서 `setSuccessFilter`와 `setFailFilter`가 그러한 역할을 수행하는데 자세한 내용은 이후 설명하겠다.

HTTP 요청을 주고받는 구체적인 구현체로 `fecth`를 사용한다면 마찬가지로 다음과 같이 Adapter를 구현할 수 있다.

/libs/http-client/FetchClient.ts
```typescript
import type HTTPClient from "./HTTPClient";
import type {HTTPClientBuilder, Filter} from "./HTTPClient";

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

/libs/http-client/index.ts
```typescript
import type {HTTPClientBuilder} from "./HTTPClient";
import {FetchClientBuilder} from "./FetchClient";

export function createHttpClient(): HTTPClientBuilder {
  const builder: HTTPClientBuilder = new FetchClientBuilder();

  return builder
    .setSuccessFilter((body: any) => body.result)
    .setFailFilter((body: any) => Promise.reject(body.error));
}
```

> 동일한 맥락으로 서버와 서로 주고받기로 약속된 헤더들을 미리 설정해준다거나 할 수 있을 것이다.

만약 어느 날 세상에 없던 멋진 HTTP Client 라이브러리가 짜잔 하고 등장한다면 `HTTPClient`의 인터페이스를 만족하는 Adapter를 구현한 뒤, `index.ts`에서 이를 사용하게 하면 된다.

모듈 외부의 별다른 변경 없이 새로운 문명의 이기를 누리리라.

간혹 다음과 같이 서버 API 응답 구조를 비즈니스 로직에까지 노출하는 경우가 있는데,

wow.js
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

AwesomeComponent.vue
```vue
<script setup>
import {createHttpClient} from "@/libs/http-client";

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

# API Client & Domain -- 여기서부터

기반 기술들에 대한 책임을 갖는 Infra 계층 위로 서버와 API를 통해 메세지를 주고 받는 API Client 와 UI가 변경되더라도 변경되지 않는 서비스의 정책이나 업무 규칙을 처리하는 책임을 갖는 Domain 이 동일한 계층에 함께 위치한다.

> 클라이언트 애플리케이션에서 이 둘은 뗄려고 해도 뗄 수 없는 관계이다. 그래서 한 계층에 속하도록 하였다.

## API Client

왜 서버와 API를 통해 메세지를 주고 받는 책임을 갖는 API Client 를 별도의 계층으로 정의하고 격리해야할까?

### UI 컴포넌트와 API의 강한 결합은 복잡도를 증가시킨다.

서버 API를 호출하는데는 생각보다 많은 구체적인 사항들이 포함된다.

예를들면 "어떤 EndPoint의 어떤 API를 호출할지" 라거나 "API의 에러코드"에 따라 이후 흐름을 분기한다 거나 여타 등등이 있지만 UI 에선 관심없는 사항들이다.

문제는 이러한 서버 API를 호출하기 위한 구체적인 사항들이 가뜩이나 복잡한 UI 컴포넌트들에 스며들어가 더욱 복잡하게 만든다는 것이다.

특히 변경의 원인이 다른 이 두 요소는 그 변경의 주기 또한 다르기 때문에 애플리케이션이 변경에 유연하게 대응할 수 없도록 만든다.

이러한 구체적인 사항들을 별도의 계층으로 격리하고 계층 외부에서는 단순히 메세지만 전달하면 되도록 구성해보자.

우선 API Client 계층에 해당하는 `apis` 디렉토리를 생성한뒤 필요에 따라 모듈의 경계를 나눈다.

> 이 글에서는 서버가 MSA (Micro Service Architecture) 구조로 설계되어 있다고 가정하고 서버의 서비스별로 모듈을 구현하였다. 예제를 단순화하기 위해 `index.js` 에 바로 구현하였지만 하나의 API 서비스가 제공하는 API가 많다면 필요에 따라 관심별로 모듈을 나누고 `index.js` 에서 모아서 모듈 외부로 노출하도록 구현한다.

/apis/a-service/index.ts
```TypeScript
import {createHttpClient} from "@/libs/http-client";
import User from "@/domain/user/User";
import {ExpiredSessionError, UnknownError} from "@/errors";

const instance = createHttpClient()
    .setBaseUrl("https://my-awesome-api/a-service")
    .build();

type ErrorResponse = {
    code: string;
    message: string;
    extensions: unknown;
}

type UserApiResponse = {
    id: string;
    profilePictureUrl: string;
    username: string;
    email: string;
}

/**
 * @throws ExpiredSessionError
 * @throws UnknownError
 */
export function getUserById(id: string): Promise<User> {
    return instance.get(`/v1/users${id}`)
        .then((res: UserApiResponse) => mapUser(res))
        .catch((error: ErrorResponse) => dispatchError(error));
}

function dispatchError(error: ErrorResponse) {
    switch (error?.code) {
        case "1234":
            throw new ExpiredSessionError();
        // another..
        default:
            throw new UnknownError();
    }
}

function mapUser(res: UserApiResponse): User {
    return new User({
        id: res.id,
        userName: res.username,
        email: res.email,
        profilePictureUrl: res.profilePictureUrl,
    });
}
```

API Client 계층의 모듈들이 갖는 관심은 다음과 같다.

1 우선 사용할 HTTP Client 를 생성한다. 이때 API 의 BaseUrl 과 성공/실패시 응답구조를 벗겨낼 전략을 주입한다. 개발팀 내부에서 합의한 응답구조를 벗겨내는 전략은 기본적으로 포함되어 있으므로 위 예제에선 별다른 전략을 주입하지 않았다.

2 구체적인 API Endpoint 를 명시한다.

3 서버에서 사용중인 모델의 구조를 그대로 노출하는 것이 아닌 Vue 애플리케이션에서 정의한 도메인 모델 객체로 번역하여 반환한다.

4 서버의 예외코드는 해당 모듈 내부로 캡슐화하고 외부로는 정의한 에러를 던진다.

> 도메인 모델 객체에 대해서는 이후에 자세히 설명하겠다.

1과 2는 부가적인 설명이 필요없을 것 같지만, 3과 4는 설명이 필요할 것 같다.

### 서버에서 사용중인 모델의 구조를 그대로 노출하지 않는다.

왜 서버에서 사용중인 모델의 구조를 상위 계층으로 그대로 노출하면 안될까?

다음과 같은 게시글 정보를 조회하는 API가 있고 여기엔 글을 쓴 유저의 정보가 포함되어 있다고 가정하자.

```json
{
  "id": "twewghe-jtjejh-qweqwe",
  "subject": "정말 멋진 게시글",
  "writer": {
    "id": "bkdow-gjdkf-sdhbo",
    "userName": "genie",
    "email: "test@test.com",
    profilePictureUrl: "https://my-awesome-cdn/pictures/awesome-iamge.jpeg"
  },
  likeCount: 25123,
  createdDate: 1649940257643
}
```

API Client는 이 모델을 상위 계층에 그대로 노출하였고

/apis/article-service/index.ts
```TypeScript
type User = {
    id: string;
    profilePictureUrl: string;
    username: string;
    email: string;
}

type Article = {
    id: string,
    subject: string,
    writer: User,
    likeCount: number,
    createdDate: Date,
}

export function fetchArticleById(id: string): Promise<Article> {
    return instance.get(`/v1/articles/${id}`)
}
```

게시글 페이지에선 이 API를 호출하여 이를 그대로 사용하였다.

```vue
<script setup>
import {reactive} from "vue";
import UserInfo from "@/components/UserInfo.vue";
import {useRoute} from "vue-router";
import {fetchArticleById} from "@/apis/article-service";

const route = useRoute();
const state = reactive({
  article: {
    id: "",
    subject: "",
    writer: {
      id: "",
      userName: "",
      email: "",
      profilePictureUrl: ""
    },
    likeCount: 0,
    createdDate: new Date(),
  }
});

fetchArticleById(route.params.articleId)
  .then(res => {
    state.article = res;
  })
</script>

<template>
  <div>
    <UserInfo :user="state.article.writer"/>
    // 생략
  </div>
</template>
```

게시글 페이지 상단에는 글 작성자에 대한 정보를 나타내는 `UserInfo` 라는 컴포넌트가 존재한다.

```vue
<script setup>
defineProps({
  user: {
    type: Object,
    default() {
      return {
        id: "",
        userName: "",
        email: "",
        profilePictureUrl: ""
      }
    },
  },
})
</script>
<template>
  <div>
    <img :src="user.profilePictureUrl"  alt="프로필 이미지"/>
    <h1>
      {{ user.userName }}
    </h1>
    contract: {{ user.email }}
  </div>
</template>
```

`User` 에 대한 정보를 받아서 이를 렌더링 하는 간단한 순수한 컴포넌트이다.

이후 내가 구독하고 있는 사용자들의 목록을 보여주는 페이지가 추가되었다고 하자. 목록에는 구독중인 사용자의 정보를 노출해야 하고 게시글 페이지와 동일한 UI를 갖기 때문에 `UserInfo` 컴포넌트를 재사용하기로 했다.

API 명세는 다음과 같다.

```json
{
  "follows": [
    "follow": {
      "id": "ebdsn-ehsdf-qwezd",
      "user": {
        "id": "bkdow-gjdkf-sdhbo",
        "name": "genie",
        "email": "genie@test.com",
        "thumbnailImageUrl": "https://my-awesome-cdn/pictures/awesome-iamge.jpeg"
      },
      "startDate": 1649940257643
    },
    "follow": {
      "id": "xgddh-eryjs-qwtdg",
      "user": {
        "id": "ehgbd-afghe-ehhrc",
        "name": "youn",
        "email": "youn@test.com",
        "thumbnailImageUrl": "https://my-awesome-cdn/pictures/wow-iamge.jpeg"
      },
      "startDate": 1649940769347
    }
  ]
}
```

API Client는 역시나 위 응답모델을 그대로 사용하였다.

/apis/follow-service/index.ts
```TypeScript
type User = {
  id: string;
  thumbnailImageUrl: string;
  name: string;
  email: string;
};

type Follow = {
  id: string;
  user: User;
  startDate: Date;
};

export function fetchAllFollows(id: string): Promise<Array<Follow>> {
  return instance.get(`/v1/follows`);
}
```

문제는 여기서 발생한다. 기존에 존재하던 `article.writer`의 모델과 `follow.user`의 모델이 상이한것이다.

누군가는 이를 맞춰줄 책임을 수행해야 한다.

API를 호출해온 Container가 수행하게 하면 다음과 같은 그림이 될 것이고,

```typescript
<script setup>
import { reactive } from "vue";
import UserInfo from "@/components/UserInfo.vue";
import { fetchAllFollows } from "@/apis/follow-service";

const state = reactive({
  follows: [],
});

fetchAllFollows().then((res) => {
  state.follows = res.follows.map(follow => ({
    id: follow.id,
    user: {
      id: follow.user.id,
      userName: follow.user.name,
      email: follow.user.email,
      profilePictureUrl: follow.user.thumbnailImageUrl,
    },
    createdDate: follow.createdDate,
  }));
});
</script>

<template>
  <div>
    <div :id="follow.id" v-for="follow in state.follows">
      <UserInfo :user="follow.user" />
    </div>
  </div>
</template>
```

이걸 `UserInfo` 에서 맞춘다고 하면 더 답도 없는 그림이 된다.

```vue
<template>
  <div>
    <img :src="user.profilePictureUrl || user.profilePictureUrl"  alt="프로필 이미지"/>
    <h1>
      {{ user.userName || user.name }}
    </h1>
    contract: {{ user.email }}
  </div>
</template>
```

가뜩이나 UI 자체만으로도 복잡한 컴포넌트들이 이러한 모델들을 맞추는 책임까지 끌어앉고 더욱 복잡해진다.

만약 API의 응답 모델이 변경된다면 어떻게 될까? API가 변경되었을 뿐인데 이를 의존하는 UI의 Component를 모두 찾아 변경해주어야 할 것이다.

이처럼 API와 관련된 요소들은 UI가 알고싶은 관심사가 아니다.

API Client 계층은 이러한 책임들을 격리하고 상위 계층에선 구체적인 부분에 대해 알 필요 없이 API가 내부적으로 약속한 도메인 모델 객체와 에러를 반환할것이란 확신을 가질 수 있도록 한다.

-- 맵핑하는 예제코드

위와같이 API Client 는 서버의 응답 모델을 그대로 반환하는게 아니라 이후 설명할 FE팀에서 정의한 Domain 계층의 모델로 번역하여 반환하는 책임을 갖는다.

> 가장 좋은 방법은 API와 클라이언트가 contract를 정의할때 일관된 모델을 사용하는것이지만, 현실적으로 굉장히 어렵다. 회사엔 정치와 각 팀의 사정이란게 있고 과거엔 맞았던게 지금은 아닐수 있고.. 그렇기 때문에 레거시 API와 새로운 API가 모델이 다를 수 있는건 어찌보면 당연한 수순이다. API가 제어할 수 없는 영역이라고 판단된다면 API Client 계층에 번역의 책임을 부여하고 이를 Anticorruption layer 로 활용하는것도 방법이다.

API 응답 모델이 변경된다면 해당 계층만 수정하면 된다.

-- 응답모델 변경에 대한 예제코드

### 서버의 예외코드는 해당 모듈 내부로 캡슐화하고 외부로는 정의한 에러를 던진다.

에러코드도 위와 비슷한 맥락이다. UI는 API가 어떤 상황에 몇번의 에러코드를 사용하는지 전혀 알 필요가 없다. 그저 "어떤 예외 상황"인지만 중요할뿐.

-- 에러코드를 에러로 변환하여 반환하는 예제코드

### 정리

서버의 API를 통해 메세지를 주고 받는 구체적인 모든 사항은 API Client 계층에 격리해라.

구체적인 사항에는 API Endpoint, 요청/응답모델, 에러코드등이 있다.

API의 응답 모델을 그대로 노출하지 않고 FE팀에서 정의한 모델로 번역하여 반환하게 하면 상위계층에선 일관된 모델이 반환될것이라고 신뢰할 수 있다.

API에 변경사항이 생기면 API Client 계층만 수정하고, UI는 변경할 필요가 없도록 구성해라.

## Domain
UI로부터 API에 관한 상세한 내용들을 분리해냈으니, 다음은 서비스의 정책이나 업무 규칙들이다.

### 왜 Domain 계층을 격리해야 하는가?

과거 서버에서 대부분 처리했던 비즈니스 로직들이 근래에는 클라이언트 애플리케이션에서 직접 처리하는 일이 많아졌다.

문제는 이러한 비즈니스 로직들이 가뜩이나 복잡한 UI를 처리하기 위한 로직들과 한곳에 섞여 복잡함을 증가시킨다는 것이다.

이 둘은 대게 그 변경의 주기가 다르다. 근본적인 서비스의 정책은 그대로인채 낡은 UI를 새로 개편한다거나 보다 실험적인 UX를 도전한다거나, Mobile 로만 제공하던 기능을 PC나 Tablet 에 맞게 확장된 UI로 제공하는등 UI가 좀 더 빈번히 변경된다.

이럴때 UI 곳곳에 스며들어 있는 서비스 정책과 관련된 로직들이 발목을 잡는다. UI 컴포넌트는 이 두 가지 책임이 섞여 거대해지고, 읽기 어려워지며 변경이 필요없는 비즈니스 로직까지 몽땅 재작성하는데 이르기까지 한다. 뒤엉킨 변경을 야기하는 것이다.

또한 이러한 서비스 정책에 관련된 로직들이 UI에 스며들어 있을때 높은 확률로 플랫폼 (PC, Mobile, Tablet)별로 산란되어 응집도가 떨어져 있는 경우가 많다. 전부 동일한 정책을 표현하는 코드들인데 말이다.

서비스 정책이 변경되면 이를 표현하는 코드들을 찾아 나선다. 산탄총 수술을 야기한다.

둘은 분리되어야 하는 책임이다.

UI에서 처리하고 있는 로직중 UI가 변경되어도 변경되지 않는 서비스의 정책이나 업무 규칙을 처리하는 로직이 있는지 확인하고, 이를 Domain 계층으로 분리하여 플랫폼간 공유하도록 한다.

이 기준이 어렵다면 개인적으론 현재 페이지를 CLI로 변경했을때 변경이 필요한가? 로 생각하면 명확하게 갈라낼 수 있었다.

이렇게 분리해낸 서비스 정책과 업무 규칙에 해당하는 로직들을 어떻게 모듈로 나누어 다른 계층에 인터페이스를 제공할것인지에 대해선 많은 방법들이 있겠지만

필자는 DDD에 소개된 내용들을 채용하여 구현하였다.

다만 한가지 중요한 부분은 클라이언트 애플리케이션에서 Domain 계층은 그 자체의 의미보다는 "복잡한 UI 로직"에서 도메인 로직을 분리해냈다는데 의미를 갖는다.

간혹 도메인 모델링을 하다보면 배보다 배꼽이 더 큰 상황이 발생할지도 모른다. 이때는 상황에 맞게 적절히 타협할 수 있도록 하자.

해당 계층에 참여하는 개념은 다음과 같다.

## Entity

## VO

## Service

## Repository


위에 인프라계층이랑 관계 다이어그램 한번 더

위에 axios 예제 빼먹었다.

레이어드 아키텍처랑 스캐폴딩이랑 함께 보여주면 좋을것 같다.

이제 UI와 관련이 없는 기반 기술, API에 대한 상세한 내용, 그리고 UI와 관련이 없는 서비스 정책이나 업무 규칙을 모두 분리해내고 온전히 UI 와 관련된 관심만이 남았다.

남아있는 UI 와 관련된 관심사들은 크게 두 계층 Application 과 단어적 정의 그대로의 UI 로 나눈다.

그리고 이 두 계층 모두 Vue 라는 프레임워크 위에 구현되게 된다.

## Application

해당 계층엔 애플리케이션이 동작하도록 하는데 관심을 갖는 객체들로 구성된다.

당연히도 클라이언트 애플리케이션이니 UI와도 밀접한 관련을 갖는다.

기반 기술도 아니고, 서비스의 정책이나 업무 규칙을 나타내는 도메인 객체도 아니면서 API 요청과 응답에 대한 객체가 아닌데다 UI와 관련이 있지만 UI 그 자체는 아닌 어디에도 속하지 않는 대부분이 이 계층에 위치할것이다.

예를들면 페이지간의 네비게이션을 담당하는 `router`, 애플리케이션 전역에서 접근이 필요한 상태를 관리하는 `store`, 애플리케이션의 특정 로직과 상태를 캡슐화한 `compositions (hooks)` 등등이 있겠다.

이중 `store` 의 설계에 대해서 자세히 설명하고 넘어가도록 하자.

### Store

`Store` 는 Vue 애플리케이션에서 "전역적인" "상태를 관리"하기 위한 저장소로 사용하여야 한다.

### 애플리케이션 전역에서 필요한 상태만을 담는다.

간혹 페이지 전역에서 접근이 필요한 상태를 저장하기 위해 `Store` 를 사용하는 경우가 있는데, 이는 곧 페이지 단위로 `Store` 를 구성하는 결과로 이어지고 많은 문제들을 야기한다.

우선 스토어의 네임스페이스가 오염된다. 페이지별로 하나의 `Store` 모듈이 생성된다면, 많은 페이지를 가진 애플리케이션이라면 다음과 같은 압도적인 글로벌 스토어를 마주해야 할 것이다.

--- 스크린샷

두번째 문제로는 상태가 파편화될 가능성이 높다.

예를들어, 현재 로그인한 사용자의 상태가 A페이지의 `Store` 와 B페이지의 `Store` 모두에 저장되어 있다면 개발자는 어떤 `Store`의 상태를 신뢰하고 사용해야 할까?

Store 는 그 본래의 역할에 맞게 애플리케이션 전역에서 접근이 필요한 상태를 관리하도록 하고 그 대상의 단위로는 앞서 설명했던 도메인 모델 객체를 단위로 삼는다.

--- 스크린샷

> 페이지 전역에서 접근이 필요한 도메인 모델 객체나, UI의 상태에 대해선 후에 서술한다.

### 상태의 라이프사이클을 관리하는 책임만을 부여한다.

간혹 이 Store 내에서 위에서 설명한 서비스 정책이나 업무 규칙을 처리하는 경우가 있다.

이러한 로직들이 간단하다면 상관없겠으나, UI에서 이러한 로직을 분리해냈듯이 상태의 라이프사이클을 관리하는 책임만으로 Store 는 충분히 복잡하다.

따라서 서비스 정책이나 업무 규칙에 대한 처리는 Domain 계층에게 위임한다.

Store는 각 도메인 모델 객체의 라이프사이클에만 관심을 갖도록 한다.

예를들어, 현재 로그인한 사용자의 정보의 경우 로그아웃을 하거나 마이페이지에서 변경하기 전까지는 변경 가능성이 낮고 애플리케이션 전반에서 접근이 필요하다.

따라서 이 상태는 Store에서 관리하고 로그인했을때 메세지를 받아 fetch 하여 메모리에 올려두고, 마이페이지에서 변경이 이루어진다면 다시 메세지를 받아 새로 fetch 한뒤 로그아웃시에 메세지를 받아 상태를 파기하도록 하면 상태를 효율적으로 관리할 수 있을 것이다.

이처럼 애플리케이션 전역에서 접근이 필요한 도메인 모델 객체를 선정하고, 각 모델에 맞게 라이프사이클을 정의하여 관리하도록 해라.

> 위 사용자 정보는 이해를 돕기 위한 예제일 뿐이다. 사용자의 접근 권한이 중요하다거나 하는 경우 매 순간 fetch 하여 상태를 갱신해주어야 할 것이다. 애플리케이션마다 이 객체들의 라이프사이클은 제각기 다를것이고, 신중히 고민하여 설계하는것이 중요하다.

## UI

이제 상태에 따라 화면을 랜더링하고, 유저와 인터랙션을 처리하며, 상태가 변경됐을때 다시 화면을 렌더링하는 UI의 본질적인 관심만이 남았다.

이 계층이 Vue 애플리케이션의 핵심이라 해도 과언이 아니다.

Vue 애플리케이션은 부모 컴포넌트로부터 주입받은 `props` 와 자신의 상태인 `data` 에 반응형으로 UI를 렌더링하는 `component` 들을 조합하여 전체 UI를 구성한다.

이 컴포넌트들은 재사용성을 확보하고, 요구사항이 변경되었을 때 기능을 확장하고 변경하기 쉽도록 책임에 따라 최소한의 단위로 작게 나누어진다.

때로는 하나의 컴포넌트가 여러 자식 컴포넌트를 갖기도 하고, 그 자식 컴포넌트가 또 다시 여러 자식컴포넌트를 갖기도 한다.

이에 따라 많은 정보를 표현해야 하는 웹 페이지는 많은 단계의 깊이를 가진 복잡한 계층구조의 컴포넌트 트리를 갖게되며, 이 많은 컴포넌트들이 정보를 표현하는데 필요로 하는 정보는 대개 페이지와 동일한 라이프사이클을 갖게되고 컴포넌트 트리 전역에서 접근할 수 있어야 한다.

이러한 상황에서 선택할 수 있는 컴포넌트들의 협력을 설계하는 몇가지 방법을 소개한다.

### Basic
가장 기본적인 설계로 각 컴포넌트들은 자신이 필요한 상태를 직접 관리한다.

컴포넌트가 생성될때 API를 통해 정보를 요청하고 이를 자신의 state 로 관리한다.

동일한 계층에 존재하는 컴포넌트들이 동일한 정보를 필요로 하다면, 이를 부모 컴포넌트에 위임하고 `props` 로 전달받도록 한다.

pros:
- 간단하다.
cons:
- 상태에 관련된 로직이 복잡할 경우, 컴포넌트에 상태에 관련된 로직과 이를 가지고 UI를 렌더링하는 로직, 사용자와의 상호작용을 처리하는 로직과 섞여 복잡도가 높은 컴포넌트를 만든다.

### Presentational and Container Components
기본적인 설계의 단점을 해결하기 위해 컴포넌트로부터 상태에 관련된 로직을 분리해낸 `Container` 컴포넌트 (이하 컨테이너)를 정의하고 상태에 관련된 로직을 가지고 있지 않은 `Presentational` 컴포넌트(이하 컴포넌트)로 분류한다.

상태 관리에 대한 책임은 컨테이너에게 부여하고 컴포넌트는 컨테이너로부터 `props`를 주입받아 상태에 따라 UI를 렌더링하고 사용자와의 상호작용을 처리하는 책임만을 갖는다.

컨테이너는 기본적으로 페이지 (라우팅의 대상이 되는) 단위로 구성된다.

```
PageAContainer
ㄴ ComponentA
  ㄴ ComponentAA
  ㄴ ComponentAB
    ㄴ ComponentABA
ㄴ ComponentB
ㄴ ComponentC
  ㄴ ComponentCA
```

`PageA`에 필요한 상태를 관리하는 책임을 `PageAContainer`에게 부여한다.
`PageAContainer`는 API를 통해 `PageA`에 필요한 정보 `StateA`를 fetch한 뒤 이를 각 컴포넌트에 `props`로 전달할것이다.

이 과정에서 컴포넌트 트리 가장 마지막 계층에 있는 `ComponentABA`가 `StateA`의 `B`라는 상태를 필요로 한다면 `ComponentA`에게 `StateA.B`를 `props`로 주입하고,
`ComponentA`는 이를 자신의 자식 컴포넌트인 `ComponentAB`에게 주입하고, `ComponentAB`는 이를 다시 자신의 자식 컴포넌트인 `ComponentABA`에게 주입해야 하는 `Prop drilling`이 발생한다.

`ComponentABA` 와 같은 컴포넌트 트리 말단에 위치한 컴포넌트가 사용자와의 상호작용에 따라 `StateA.B`에 변경이 필요하다면 위 과정을 역으로 거쳐 이벤트를 발생시켜 `PageAContainer` 에게 메세지를 전달하거나, 이에 대한 콜백을 함께 `props`로 주입받아야 할 것이다.

<img width="710" alt="스크린샷 2021-12-16 오전 9 05 37" src="https://user-images.githubusercontent.com/16642635/146284032-6a6f94a4-3c99-4256-9f45-f667993f2bff.png">


만약 `PageA`의 상태관련된 로직이 너무 복잡하여 `PageAContainer`가 너무 복잡해지고 많은 책임을 지게 되었다면 이를 표현하는 상태를 기준으로 더 작은 컨테이너로 나눌 수 있다.

```
PageAContainer
ㄴ ContainerA
  ㄴ ComponentA
    ㄴ ComponentAA
    ㄴ ComponentAB
      ㄴ ComponentABA
ㄴ ComponentB
ㄴ ContainerC
  ㄴ ComponentC
    ㄴ ComponentCA
```

각각의 컨테이너는 독립적인 동작을 보장하는 컨텍스트의 단위가 된다. 즉 다음과 같이 사용이 가능해야 한다.

```
PageBContainer
ㄴ ContainerA
  ㄴ ComponentA
    ㄴ ComponentAA
    ㄴ ComponentAB
      ㄴ ComponentABA
```

pros:
- 컴포넌트로부터 상태를 관리하는 로직을 제거했기 때문에 컴포넌트는 `UI = f(state)`의 철학에 맞게 주어진 `state`로 UI를 구성하는 순수한 상태를 유지할 수 있다.
- 컴포넌트를 순수한 상태로 유지할 수 있기 때문에 재사용하기 쉽고 테스트하기 쉬운 컴포넌트를 작성할 수 있다.
- 컴포넌트의 복잡도를 많이 낮출 수 있다.
cons:
- prop drilling
- 컴포넌트가 순수하게 유지될 수 있도록 하기위해  side-effect를 일으킬 수 있는 함수를 `props`로 전달하거나 prop drilling의 역으로 이벤트를 전달받아야 한다.
- 상태 관리 로직이 복잡할경우 컨테이너가 가진 다른 관점의 책임들 (레이아웃을 구성하고, 자신의 자식 컴포넌트간의 관계를 설정하고, 자신의 자식 컴포넌트가 참여하는 flow 로직을 처리하는 등) 과 섞여 복잡해진다.

> Presentational and Container Component의 자세한 내용은 Dan Abramov의 [아티클](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0)을 참고.
> Dan Abramov는 현재는 컨테이너보단 hooks를 사용하여 상태 관리 로직을 분리할것을 권장하고 있음.

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

### Page Store
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

예를들어 네이버 회원 프로필 조회 Open API의 경우 다음과 같은 응답포맷을 갖는다.

이 부분은 뒤로 쭉 빼자

```json
{
  "resultcode": "00",
  "message": "success",
  "response": {
    "email": "openapi@naver.com",
    "nickname": "OpenAPI",
    "profile_image": "https://ssl.pstatic.net/static/pwe/address/nodata_33x33.gif",
    "age": "40-49",
    "gender": "F",
    "id": "32742776",
    "name": "오픈 API",
    "birthday": "10-01",
    "birthyear": "1900",
    "mobile": "010-0000-0000"
  }
}
```

> https://developers.naver.com/docs/login/profile/profile.md

네이버 회원 프로필 API 클라이언트는 다음과 같이 HTTPClient 를 생성한다.

NaverUserAPIClient.ts
```TypeScript
import {createHttpClient} from "@/libs/http-client";

const instance = createHttpClient()
    .setBaseUrl("https://네이버-OPEN-API")
    .setSuccessFilter((body: Record<string, unknown>) => body.response)
    .setFailFilter((body: Record<string, unknown>) => body.message)
    .build();
```
