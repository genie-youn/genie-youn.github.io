---
layout: post
title: "Vue Application Architecture - API Client (Part2)"
author: "genie-youn"
categories: journal
tags: [Vue, Front End, Architecture, DDD]
image: DSC04956.jpg
---
# Table of Contents
1. [복잡함을 해결해야 하는 이유 - Part1](https://genie-youn.github.io/journal/Vue_Application_Architecture_part1.html)
2. [Infra - Part1](https://genie-youn.github.io/journal/Vue_Application_Architecture_part1.html)
2. API Client
3. [Domain - Part3]() (작성중..)
4. [Application - Part4]() (작성중..)
5. [UI - Part5]() (작성중..)
6. [마치며 - Part5]() (작성중..)

> [Part1](https://genie-youn.github.io/journal/Vue_Application_Architecture_part1.html) 에서 이어집니다.

# API Client & Domain

기반 기술들에 대한 책임을 갖는 Infra 계층 위로 서버와 API를 통해 메시지를 주고받는 API Client와 UI가 변경되더라도 변경되지 않는 서비스의 정책이나 업무 규칙을 처리하는 책임을 갖는 Domain이 동일한 계층에 함께 위치한다.

> 프론트엔드 애플리케이션에서 이 둘은 떼려고 해도 뗄 수 없는 관계이다. 그래서 한 계층에 속하도록 하였다.

<img width="1024" alt="스크린샷 2022-04-19 오후 9 05 27" src="https://user-images.githubusercontent.com/16642635/163999425-3bddd4a8-3656-4f1e-bb3c-6587c24cae56.png">

# API Client

왜 서버와 API를 통해 메세지를 주고 받는 책임을 갖는 API Client 를 별도의 계층으로 정의하고 격리해야할까?

## UI 컴포넌트와 API의 강한 결합은 복잡도를 증가시킨다.

서버 API를 호출하는데는 생각보다 많은 구체적인 사항들이 포함된다.

예를들면 "어떤 EndPoint의 어떤 API를 호출할지"라거나 "API의 에러코드"에 따라 이후 흐름을 분기한다 거나 여타 등등이 있지만 UI에선 관심없는 사항들이다.

문제는 이러한 서버 API를 호출하기 위한 구체적인 사항들이 가뜩이나 복잡한 UI 컴포넌트들에 스며들어가 더욱 복잡하게 만든다는 것이다.
특히 변경의 원인이 다른 이 두 요소는 그 변경의 주기 또한 다르기 때문에 애플리케이션이 변경에 유연하게 대응할 수 없도록 만든다.

이러한 구체적인 사항들을 별도의 계층으로 격리하고 계층 외부에서는 단순히 메시지만 전달하면 되도록 구성해보자.

우선 API Client 계층에 해당하는 `api` 디렉토리를 생성한뒤 필요에 따라 모듈의 경계를 나눈다.

> 이 글에서는 서버가 MSA (Micro Service Architecture) 구조로 설계되어 있다고 가정하고 서버의 서비스별로 모듈을 구현하였다. 예제를 단순화하기 위해 `index.js` 에 바로 구현하였지만 하나의 API 서비스가 제공하는 API가 많다면 필요에 따라 관심별로 모듈을 나누고 `index.js` 에서 모아서 모듈 외부로 노출하도록 구현한다.

```typescript
/* @/api/a-service/index.ts */
import {createHttpClient} from "@/lib/http-client";
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

1. 우선 사용할 HTTP Client를 생성한다. 이때 API의 BaseUrl과 성공/실패 시 응답 구조를 벗겨낼 전략을 주입한다. 개발팀 내부에서 합의한 응답 구조를 벗겨내는 전략은 기본적으로 포함되어 있으므로 위 예제에선 별다른 전략을 주입하지 않았다.

2. 구체적인 API Endpoint를 명시한다.

3. 서버에서 사용 중인 모델의 구조를 그대로 노출하는 것이 아닌 프론트엔드 애플리케이션에서 정의한 도메인 모델 객체로 번역하여 반환한다.

4. 서버의 에러 코드는 해당 모듈 내부로 캡슐화하고 외부로는 정의한 에러를 던진다.

> 도메인 모델 객체에 대해서는 이후에 자세히 설명하겠다.

1)과 2)는 부가적인 설명이 필요 없을 것 같지만, 3)과 4)는 설명이 필요할 것 같다.

### 서버에서 사용 중인 모델의 구조를 그대로 노출하지 않는다.

왜 서버에서 사용 중인 모델의 구조를 상위 계층으로 그대로 노출하면 안 될까?

다음과 같은 게시글 정보를 조회하는 API가 있고 여기엔 글을 쓴 유저의 정보가 포함되어 있다고 가정하자.

```json
{
  "id": "twewghe-jtjejh-qweqwe",
  "subject": "정말 멋진 게시글",
  "writer": {
    "id": "bkdow-gjdkf-sdhbo",
    "userName": "genie",
    "email": "test@test.com",
    "profilePictureUrl": "https://my-awesome-cdn/pictures/awesome-iamge.jpeg"
  },
  "likeCount": 25123,
  "createdDate": 1649940257643
}
```

API Client는 이 모델을 상위 계층에 그대로 노출하였고

```typescript
/* @/api/article-service/index.ts */
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
import {fetchArticleById} from "@/api/article-service";

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
    <img :src="user.profilePictureUrl" alt="프로필 이미지"/>
    <h1>    
      {% raw %}{{ user.userName }}{% endraw %}
    </h1>
    contract: {% raw %}{{ user.email }}{% endraw %}
  </div>
</template>
```

`User`에 대한 정보를 받아서 이를 렌더링 하는 간단한 순수한 컴포넌트이다.

이후 내가 구독하고 있는 사용자들의 목록을 보여주는 페이지가 추가되었다고 하자. 목록에는 구독중인 사용자의 정보를 노출해야 하고 게시글 페이지와 동일한 UI를 갖기 때문에 `UserInfo` 컴포넌트를 재 사용하기로 했다.

API 명세는 다음과 같다.

```json
{
  "follows": [
    {
      "id": "ebdsn-ehsdf-qwezd",
      "user": {
        "id": "bkdow-gjdkf-sdhbo",
        "name": "genie",
        "email": "genie@test.com",
        "thumbnailImageUrl": "https://my-awesome-cdn/pictures/awesome-iamge.jpeg"
      },
      "startDate": 1649940257643
    },
    {
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

```typescript
/* @/api/follow-service/index.ts */
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

export function fetchAllFollows(id: string): Promise<Follow[]> {
  return instance.get(`/v1/follows`);
}
```

문제는 여기서 발생한다. 기존에 존재하던 `article.writer`의 모델과 `follow.user`의 모델이 상이한것이다.

누군가는 이를 맞춰줄 책임을 수행해야 한다.

API를 호출해온 Container가 수행하게 하면 다음과 같은 그림이 될 것이고,

```vue
<script setup>
import {reactive} from "vue";
import UserInfo from "@/components/UserInfo.vue";
import {fetchAllFollows} from "@/api/follow-service";

const state = reactive({
  follows: [],
});

fetchAllFollows().then((res) => {
  state.follows = res.follows.map(follow => ({
    id: follow.id,
    user: {
      id: follow.user.id,
      // 기존 모델에 맞게 번역한다.
      userName: follow.user.name,
      email: follow.user.email,
      // 기존 모델에 맞게 번역한다.
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

> 혹은 API를 fetch하는 composition(hook)에서 번역할 수도 있지만 Container가 번역할 때와 문제는 동일하다.

이걸 `UserInfo`에서 맞춘다고 하면 더 답도 없는 그림이 된다.

```vue
<template>
  <div>
    <img :src="user.profilePictureUrl || user.profilePictureUrl" alt="프로필 이미지"/>
    <h1>
      {% raw %}{{ user.userName || user.name }}{% endraw %}
    </h1>
    contract: {% raw %}{{ user.email }}{% endraw %}
  </div>
</template>
```

가뜩이나 UI 자체만으로도 복잡한 컴포넌트들이 이러한 모델들을 맞추는 책임까지 끌어안고 더욱 복잡해진다.

만약 API의 응답 모델이 변경된다면 어떻게 될까? API가 변경되었을 뿐인데 이를 의존하는 UI의 Component를 모두 찾아 변경해주어야 할 것이다.

<img width="1300" alt="스크린샷 2022-04-20 오후 10 36 42" src="https://user-images.githubusercontent.com/16642635/164242822-0bf723b5-64b0-45b1-a3af-41570ed7e5a4.png">


이처럼 API와 관련된 요소들은 UI가 알고 싶은 관심사가 아니다.

API Client 계층은 이러한 책임들을 격리하고 상위 계층에선 구체적인 부분에 대해서 모르게한다.
서버 API 응답 모델을 그대로 상위계층에 노출하지 않고, FE 팀 내부적으로 정의한 모델로 번역하여 API Client 계층이 항상 일관된 모델을 반환하게 해라.

적어도 UI와 Application 계층 내에서는 하나의 개념에 대해 하나의 일관된 모델을 사용할 수 있도록 한다면, 코드는 한층 더 간결해지고 API가 변경되었을때도 API Client만 수정한다면 상위 계층엔 변경이 없을 것이다.

필자는 UI와 Application 계층 내에서 사용할 일관된 모델을 Domain 계층에 정의된 객체를 사용하기로 했다.
`User`, `Article`, `Follow`를 Domain 계층에 정의하고 서버의 응답 모델을 이 모델들로 번역하는 책임을 API Client에 부여했다.

```typescript
/* @/api/article-service/index.ts */
import {createHttpClient} from "@/lib/http-client";
import User from "@/domain/user/User";
import Article from "@/domain/Article";

const instance = createHttpClient()
  .setBaseUrl("https://my-awesome-api/article-service")
  .build();

type ArticleAPIResponse = {
  id: string;
  subject: string;
  writer: {
    id: string;
    userName: string;
    email: string;
    profilePictureUrl: string;
  };
  likeCount: number;
  createdDate: Date;
};

export function fetchArticleById(id: string): Promise<Article> {
  return instance.get(`/v1/articles/${id}`).then((res: ArticleAPIResponse) => {
    // API 응답 모델을 Domain 계층의 객체로 번역할 책임을 갖는다.
    return new Article({
      id: res.id,
      subject: res.subject,
      writer: new User({
        id: res.writer.id,
        userName: res.writer.userName,
        email: res.writer.email,
        profilePictureUrl: res.writer.profilePictureUrl,
      }),
      likeCount: res.likeCount,
      createdDate: res.createdDate,
    });
  });
}
```

```typescript
/* @/api/follow-service/index.ts */
import {createHttpClient} from "@/lib/http-client";
import Follow from "@/domain/follow/Follow";
import User from "@/domain/user/User";

const instance = createHttpClient()
  .setBaseUrl("https://my-awesome-api/follow-service")
  .build();

type FollowAPIResponse = {
  id: string;
  user: {
    id: string;
    thumbnailImageUrl: string;
    name: string;
    email: string;
  };
  startDate: Date;
};

export function fetchAllFollows(): Promise<Array<Follow>> {
  return instance.get(`/v1/follows`).then((res: FollowAPIResponse) => {
    return new Follow({
      id: res.id,
      user: new User({
        id: res.user.id,
        userName: res.user.name,
        email: res.user.email,
        profilePictureUrl: res.user.thumbnailImageUrl,
      }),
      startDate: res.startDate,
    });
  });
}

```

> 가장 좋은 방법은 API와 클라이언트가 contract를 정의할 때 일관된 모델을 사용하는것이지만, 현실적으로 굉장히 어렵다. 회사엔 정치와 각 팀의 사정이란 게 있고 과거엔 맞았던 게 지금은 아닐 수 있고... 그렇기 때문에 레거시 API와 새로운 API가 모델이 다를 수 있는 건 어찌 보면 당연한 수순이다. API가 제어할 수 없는 영역이라고 판단된다면 API Client 계층에 번역의 책임을 부여하고 이를 Anti-Corruption Layer로 활용하는 것도 방법이다.

API 응답 모델이 변경된다면 해당 계층만 수정하면 된다.

<img width="1315" alt="스크린샷 2022-04-20 오후 10 37 23" src="https://user-images.githubusercontent.com/16642635/164242992-d446c693-582c-40ac-8e5a-47cc3cc339a8.png">

### 서버의 예외코드는 해당 모듈 내부로 캡슐화하고 외부로는 정의한 에러를 던진다.

에러코드도 위와 비슷한 맥락이다. UI는 API가 어떤 상황에 몇번의 에러코드를 사용하는지 전혀 알 필요가 없다. 그저 "어떤 예외 상황"인지만 중요할뿐.

```typescript
function dispatchError(error: ErrorResponse) {
    switch (error?.code) {
        case "1234":
            throw new ExpiredSessionError();
        // another..
        default:
            throw new UnknownError();
    }
}
```

서버 API와 일반적인 상황에서 범용적으로 사용되는 에러 코드 (사용자게에게 API의 메시지를 바로 노출하면 된다거나, 인증이 만료되었다거나, 등등) 가 있다면 이를 `failFilter`에 추가하여 글로벌하게 활용하거나 할 수 있을 것이다.

<details markdown=block>
<summary>setSuccessFilter와 setFailFilter에 대해서</summary>
위에서 이야기한


```
개발팀 내부에서 정의한 응답 포맷이 있을 테고 기본적으로 이 포맷에 맞춰 응답을 벗겨낸 후 반환하게 될 테지만, 간혹 외부 서비스의 API를 요청하는 경우도 종종 있다.
이러한 상황을 대비해 사용하는 쪽에서 이 전략을 선택할 수 있는 여지를 주면 좋다.
위 예제에서 `setSuccessFilter`와 `setFailFilter`가 그러한 역할을 수행하는데 자세한 내용은 이후 설명하겠다.
```

내용에 대한 부연 설명이다.

보통 서비스를 개발하다 보면, 사내 다른 부서나 외부의 서버 API를 활용할 일이 생긴다.

당연히 서비스팀 내부 API와는 다른 응답 구조를 지니고 있을테고 `successFilter`와 `failFilter`는 이러한 상황에서 응답 구조를 벗겨내는 전략을 `HTTPClient`의 사용처인 API Client에서 결정할 수 있게 한다.

예를 들어 사내 다른 부서의 API가 다음과 같은 응답 구조를 갖는다고 가정해보자.

```json
{
  "resultcode": "00",
  "message": {
    "result": {
      "email": "test@abc.com",
      "nickname": "genie",
      "profile_image": "https://my-awesome-image/test.gif",
      // .. 등등
    }
  }
}
```

API Client는 다음과 같이 `HTTPClient`를 생성한다.

```typescript
/* @/api/external/another-service/index.ts */
import {createHttpClient} from "@/lib/http-client";

const instance = createHttpClient()
    .setBaseUrl("https://너네서비스-게이트웨이")
    .setSuccessFilter((body: Record<string, unknown>) => body.message.result)
    .build();
```

꼭 외부의 API뿐만 아니라, 서비스팀 내부 API 중 클라이언트와 contract가 생성되기 전 레거시 API들을 `filter`가 약속한 contract로 번역해주도록 하면 프론트엔드 애플리케이션에서는 이에 대한 자세한 사항을 알 필요 없이 contract에 맞춰 개발할 수 있다는 장점이 있다.

</details>

### 정리

서버의 API를 통해 메세지를 주고받는 구체적인 모든 사항은 API Client 계층에 격리한다.

구체적인 사항에는 API Endpoint, 요청/응답 모델, 에러 코드등이 있다.

API의 응답 모델을 그대로 노출하지 않고 FE 팀에서 정의한 모델로 번역하여 반환하게 하면 상위계층에선 일관된 모델이 반환될 것이라고 신뢰할 수 있다.

API에 변경 사항이 생기면 API Client 계층만 수정하고, UI는 변경할 필요가 없도록 구성한다.

> 추가적으로 이렇게 분리한 API Client 계층을 가지고 contract의 변경을 주기적으로 감시하는데 활용할 수 있을 것이다.

**pros:**
- API에 변경이 생겼을 때 그 변경의 여파를 줄일 수 있다.
- API를 통한 요청/응답의 구체적인 사항이 별도의 계층으로 격리되어 UI, Application 계층의 복잡도를 낮출 수 있다.

**cons:**
- 단점이 뭐가 있을까..

현재까지의 스캐폴딩이다.

<img width="625" alt="스크린샷 2022-04-20 오후 10 38 41" src="https://user-images.githubusercontent.com/16642635/164243288-c9871c9b-adf2-4f99-8201-3f296f77ed6b.png">

<img width="1536" alt="스크린샷 2022-04-20 오후 10 40 46" src="https://user-images.githubusercontent.com/16642635/164243598-c186087c-525e-4408-bf4a-3f6f5f3c2e78.png">
