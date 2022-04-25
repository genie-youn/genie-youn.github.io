---
layout: post
title: "Vue Application Architecture - Domin (Part3)"
author: "genie-youn"
categories: journal
tags: [Vue, Front End, Architecture, DDD]
image: DSC04956.jpg
---
# Table of Contents
1. [복잡함을 해결해야 하는 이유 - Part1](https://genie-youn.github.io/journal/Vue_Application_Architecture_part1.html)
2. [Infra - Part1](https://genie-youn.github.io/journal/Vue_Application_Architecture_part1.html)
2. [API Client - Part2](https://genie-youn.github.io/journal/Vue_Application_Architecture_part2.html)
3. Domain
4. [Application - Part4](https://genie-youn.github.io/journal/Vue_Application_Architecture_part4.html)
5. [UI - Part5](https://genie-youn.github.io/journal/Vue_Application_Architecture_part5.html)
6. [마치며 - Part5](https://genie-youn.github.io/journal/Vue_Application_Architecture_part5.html)

> [Part2](https://genie-youn.github.io/journal/Vue_Application_Architecture_part1.html) 에서 이어집니다.

# Domain
UI로부터 API에 관한 상세한 내용들을 분리해냈으니, 다음은 서비스의 정책이나 업무 규칙들이다.

<img width="1024" alt="스크린샷 2022-04-19 오후 9 05 27" src="https://user-images.githubusercontent.com/16642635/163999425-3bddd4a8-3656-4f1e-bb3c-6587c24cae56.png">

## 왜 Domain 계층을 격리해야 하는가?

과거 대부분 서버에서 처리했던 비즈니스 로직들이 근래에는 프론트엔드 애플리케이션에서 직접 처리하는 일이 많아졌다.
문제는 이러한 비즈니스 로직들이 가뜩이나 복잡한 UI를 처리하기 위한 로직들과 한곳에 섞여 복잡함을 증가시킨다는 것이다.

이 둘은 대게 그 변경의 주기가 다르다.
근본적인 서비스의 정책은 그대로인 채 낡은 UI를 새로 개편한다거나 보다 실험적인 UX를 도전한다거나, Mobile로만 제공하던 기능을 PC나 Tablet에 맞게 확장된 UI로 제공하는 등 UI가 좀 더 빈번히 변경된다.

이럴때 UI 곳곳에 스며들어 있는 서비스 정책과 관련된 로직들이 발목을 잡는다.
UI 컴포넌트는 이 두 가지 책임이 섞여 거대해지고, 읽기 어려워지며 변경이 필요 없는 비즈니스 로직까지 몽땅 재작성하는데 이르기까지 한다.
뒤엉킨 변경을 야기하는 것이다.

또한 이러한 서비스 정책에 관련된 로직들이 UI에 스며들어 있을때 전부 동일한 정책을 표현하는 코드지만 높은 확률로 플랫폼 (PC, Mobile, Tablet)별로 분산되어 응집도가 떨어져 있는 경우가 많다.
서비스 정책이 변경되면 이를 표현하는 코드들을 찾아 나선다.
산탄총 수술을 야기한다.

예를들어, 예약 서비스에서 date-picker에서 예약 시작일을 선택하면 종류별 최소/최대 예약 가능일과 같은 내부 정책에 의해 예약 가능일이 결정되고 이를 다시 date-picker에 반영해주어야 한다고 가정하자.
Vue component나 composition(hook), 혹은 store.. 에서 이를 처리하게 한다면 다음과 같이 파편화되어 있을 확률이 높다.

이때 이 예약 가능 기간에 대한 정책이 변경된다면 UI의 변경도 아닌데 Vue가 개입하는 코드들을 수정해주어야 한다.

<img width="1020" alt="스크린샷 2022-04-20 오후 10 42 39" src="https://user-images.githubusercontent.com/16642635/164243983-5a2ad88e-02fb-4e81-8424-6f32500233d4.png">

이러한 서비스 정책, 업무 규칙을 Domain 계층으로 격리하여 변경의 범위를 고립시킬 수 있다.

<img width="1039" alt="스크린샷 2022-04-20 오후 10 43 02" src="https://user-images.githubusercontent.com/16642635/164244056-c4ede05d-1edf-4170-902a-5eda62334d01.png">

> 물론 많은 부분의 서비스 정책이나 업무 규칙은 서버에서 처리된다. 위의 경우 동시에 예약을 시도하는 사용자는 적지만 사용자가 빈번하게 예약 가능일자를 변경하는 use case를 가진다면 클라이언트에서 이 규칙을 처리하고, 실제 예약 실행시 서버에서 검증하는게 합리적일 것이다. 이처럼 반응성이나 UI와 밀접한 관련이 있거나, 그 효율에 있어 클라이언트에서 처리해야하는 서비스 정책이나 업무 규칙도 존재하기 마련이다.

> 물론2 이러한 논리를 UI Component에서 책임의 단위로 분리해내고 재사용성을 확보하기 위한 Composition API(리액트에선 hook과 같은)가 존재한다. 서비스 정책이나 업무 규칙을 Composition으로 관리하여도 복잡함이 문제가 되지 않는다면 구태어 Domain 계층을 정의하지 않고 플랫폼 별로 이 Composition을 공유하게 해도 된다. 하지만 Composition은 애플리케이션에 필요한 모든 논리들을 담고 있기 때문에 애플리케이션의 규모가 커지면 함께 그 수가 증가하게 된다. 이 경우 "UI가 변해도 유지되는 업무 규칙"이라는 기준으로 UI에 대한 책임을 갖는 Vue에서 이를 완전히 격리시키면 복잡성을 낮출 수 있다. Vue의 개입 없이 순수한 자바스크립트로 구현되어 테스트하기 쉽고 공유하기 쉬우며, 서비스 정책이나 업무 규칙을 응집력있게 모아놓은 덕분에 이를 파악하기 쉽다는 이점은 덤이다.

즉, 둘은 분리되어야 하는 책임이다.

UI에서 처리하고 있는 로직 중 UI가 변경되어도 변경되지 않는 서비스의 정책이나 업무 규칙을 처리하는 로직이 있는지 확인하고, 이를 Domain 계층으로 분리하여 플랫폼 간 공유하도록 한다.

이 기준이 어렵다면 개인적으론 "현재 페이지를 CLI로 변경했을 때 변경이 필요한가?"고 생각하면 명확하게 갈라낼 수 있었다.

이렇게 분리해낸 서비스 정책과 업무 규칙에 해당하는 로직들을 어떻게 모듈로 나누어 다른 계층에 인터페이스를 제공할 것인지에 대해선 많은 방법이 있겠지만, 앞서 얘기했듯 조직 구성원 모두가 같은 시각으로 코드 베이스를 바라볼 가이드가 필요했고 이를 위해 잘 알려진 [에릭 에반스의 "도메인 주도 설계"](https://www.codecademy.com/learn/learn-typescript/modules/learn-typescript-advanced-object-types/cheatsheet)의 내용을 채용하였다.

다만 한 가지 중요한 부분은 프론트엔드 애플리케이션에서 Domain 계층은 그 자체의 의미보다는 "복잡한 UI 로직"에서 도메인 로직을 분리해냈다는데 의미를 갖는다.
간혹 도메인 모델링을 하다 보면 배보다 배꼽이 더 큰 상황이 발생할지도 모른다. 이때는 상황에 맞게 적절히 타협할 수 있도록 하자.

> 이 글에서는 프론트앤드 애플리케이션에서 Domain 계층을 어떻게 구성하는지에 대해서만 간단히 설명한다.
> 자세한 설명은 ["도메인 주도 설계"](https://www.codecademy.com/learn/learn-typescript/modules/learn-typescript-advanced-object-types/cheatsheet)를 읽어보길 권한다.

우선 Domain 계층에 해당하는 `domain` 디렉토리를 생성한다.

도메인 모델링의 산출물이 이 디렉토리에 위치하게 되며, 기본적으로는 루트 ENTITY 단위로 모듈을 구성한다.

각 모듈은 루트 ENTITY와 AGGREGATE를 이루는 ENTITY, VALUE OBJECT 그리고 이들과 관련된 SERVICE, FACTORY, REPOSITORY가 위치하며 AGGREGATE에 해당하는 UBIQUITOUSE LANGUAGE를 정의해 README.md 로 관리한다.

각 모듈은 상위 계층인 UI, Application 계층에게 메시지를 받아 서비스의 정책이나 업무 규칙에 관련된 문제를 해결하며 이 과정에서 API Client 계층에 메시지를 전달할 수 있다.

예를들어, REPOSITORY는 라이프사이클 중간 단계의 도메인 모델 객체를 획득하기 위해 API Client에게 메시지를 전달하고 서버에 영속화되어 있는 객체를 획득할 수 있다. (아마 이후 이를 Store에 저장해 애플리케이션 전역에서 접근할 수 있도록 할 것이다.)

## ENTITY
식별성과 연속성으로 정의되는 객체이다. 필자가 애정하는 서비스 [airbnb](https://www.airbnb.co.kr/)를 예를 들면 숙소를 나타내는 개념인 `Room`같은 객체가 이에 해당한다.

각 개별 숙소는 식별성 자체가 본질을 정의하는 요소이다.
같은 이름을 갖는 숙소일지라도 동일한 숙소가 아닌것처럼 말이다.

또한 `Room`과 변경을 함께하는 연관관계를 맺는 수많은 개념들이 있으므로 AGGREGATE의 루트 ENTITY라고 할 수 있겠다.
이 `Room`을 루트 ENTITY로 갖는 AGGREGATE의 디렉토리 `room`을 `domain` 하위에 만든 뒤 `Room.ts`를 추가한다.

모듈이 가리키는 개념을 default로 export한다. 당연히도 이 경우는 `Room` class가 해당된다.

```typescript
/* @/domain/room/Room.ts */
export default class Room {
    id: number;
    name: string;
    location: Location;
    reservationableDates: Date[];

    // 고객등급, 이벤트로 인한 할인율등 내부 정책에 의해 결정된 숙박 가격을 반환한다.
    get price(checkInDate: Date, checkOutDate: Date) {
        // calc..
    }
    // 희망한 체크인 날짜로부터 가능한 체크아웃 날짜를 반환한다.
    findReservationableDatesFromCheckInDate(checkInDate: Date) {
        // calc..
    }
}
```

숙소의 최소 예약 가능일수를 고려해 체크인 날짜를 기준으로 가능한 체크아웃 날짜가 언제인지,
선택한 기간동안 고객의 등급이나 할인율을 고려한 최종 가격을 얼마인지 등등의 정보는 UI와는 전혀 상관없는 서비스의 정책에 의해 결정되는 부분이다.

(필요하다면 API Client에게 메시지를 보내 선택한 날짜에 해당하는 가격을 서버에서 받아올 수도 있을것이다.)

이러한 로직들은 UI에서 분리해내 도메인 계층에 위임하고 구체적인 내용은 캡슐화한다.

체크인/체크아웃 날짜를 선택하는 UI Component에선 최소 예약 가능일 수와 같은 정책에 대한 관심 없이 `Room`에게 메시지를 보내 반환받은 날짜만 선택 가능하도록 처리한다.


```vue
<script setup>
import {reactive} from "vue";
defineProps({
  room: {
    type: Room,
    required: true,
  },
})
const reservationalbeDates = reactive([]);

const selectCheckIn = (checkInDate) => {
  // 직접 계산하지 않고 Room에게 위임한다.
  reservationalbeDates = room.findReservationableDatesFromCheckInDate(checkInDate);
};
</script>
<template>
  <!-- 단순히 반환 받은 날짜 외의 영역을 disabled 하기만 하고 체크인 날짜를 선택 시 새로 계산한다.  -->
  <Calendar :available-dates="reservationalbeDates" @select-check-in="selectCheckIn" />
</template>

```

## VALUE OBJECT
개념적 식별성 없이 사물의 어떤 특징을 나타내기 위한 객체이다.

각 숙소가 갖추고 있는 편의시설 정보는 식별성이 중요한게 아닌 특징을 나타내기 위한 값 객체이다.
숙소가 존재하지 않으면 존재하지 않는 `Room`에 포함되는 개념이므로 room 디렉토리 하위에 위치시킨다.

마찬가지로 모듈이 가르키는 개념을 default로 export한다.

```typescript
/* @/domain/room/Amenity.ts */
export default class Amenity {
    hasOceanView: boolean;
    hasWifi: boolean;
    hasTv: boolean;
    // more..
}

```

## SERVICE
서비스의 정책이나 업무 규칙에 해당되지만 개념적으로 어떠한 객체에도 속하지 않는 연산들이 존재한다.
이를 억지로 특정 객체에 책임을 부여하려 하기보다는 SERVICE로 정의할 수 있다.

위의 예제에서 숙소에서 체크인 날짜를 기준으로 체크아웃 가능한 날짜를 계산하는 연산이 `Room`의 본질적인 속성이 아니라고 생각될 수 있다.

이 경우 예약가능일자에 대한 연산들을 SERVICE로 정의한다.

상단에 인터페이스를 명시하고 구현을 바로 반환하게 하였다.

```typescript
/* @/domain/reservation/ReservationService.ts */
import {isBefore, addDays} from "@/lib/date-time";

interface ReservationService {
    findReservationableDatesFromCheckInDate(room: Room, checkInDate: Date);
}

function findReservationableDatesFromCheckInDate(room: Room, checkInDate: Date) {
    return room.reservationableDates
        .filter(date => isBefore(date, addDays(checkInDate, room.minReservationableDays)))
        .filter(date => //...)
}

// more..

export default {
    findReservationableDatesFromCheckInDate,
    // more..
} as ReservationService;
```

이 외에도 도메인 모델 객체를 생성하는 책임을 갖는 FACTORY나 라이프 사이클 중간 단계에 있는 도메인 모델 객체를 생성하기 위한 REPOSITORY와 같은 요소들을 활용해 Domain 계층을 구성할 수 있다.

특히 REPOSITORY의 경우 라이프 사이클 중간 단계의 도메인 객체는 보통 서버에 영속화되어 있어 있기 때문에 API Client에 메시지를 전달해 이를 반환하는식으로 구현될 것이다.

```typescript
import {fetchRooms} from "@/api/room-service";

interface RoomRepository {
    findByCheckInDateAndCheckOutDate(checkInDate: Date, checkOutDate: Date): Room[]
}

function findByCheckInDateAndCheckOutDate(checkInDate: Date, checkOutDate: Date): Room[] {
    return fetchRooms(checkInDate, checkOutDate);
}

export default {
    findByCheckInDateAndCheckOutDate
} as RoomRepository
```

## 정리
UI와 서비스 정책, 업무 규칙은 변경의 원인이 다르며 이 둘이 한데 섞이면 읽기 어렵고 변경하기 어려워진다.

UI가 변경이 되더라도 변하지 않는 서비스 정책이나 업무 규칙을 별도의 Domain 계층으로 격리한다.

Domain 계층을 구성하는 방법은 여러가지가 있겠지만 그 중 잘 알려진 ["도메인 주도 설계"](https://www.codecademy.com/learn/learn-typescript/modules/learn-typescript-advanced-object-types/cheatsheet)의 내용을 채용한다면 모두가 동일한 시각으로 코드 베이스를 바라보는데 도움이 될 수 있다.

다양한 플랫폼 별로 패키지가 분리되어 있다면 UI는 플랫폼별로 두고 이 Domain 계층을 공유하게 할 수 있다. (이전에 설명했던 Infra, API Client 계층도 마찬가지다.)

프론트앤드 애플리케이션에서 처리해야할 서비스 정책, 업무 규칙이 복잡하지 않은 경우 UI Component로부터 논리를 책임단위로 분리할 수 있는 Composition API(hook)로 분리해내어 관리할 수 있지만 복잡할 경우 별도의 Domain 계층을 두는것을 고려해보는걸 추천한다.

이렇게 별도도 분리한 Domain 계층은 Vue(React)의 개입 없이 순수한 자바스크립트(타입스크립트)로 구현되어 테스트하기 쉽고 공유하기 쉽다.

뿐만 아니라 도메인에 관련된 로직을 응집력있게 구성하여 팀에 새로 합류한 신규 개발자로 하여금 서비스 정책, 업무 규칙에 대한 insight를 제공할 수 있다.

**props: **
- 서비스 정책이나 업무 규칙이 변경되지 않고 UI/UX만 개편하는 경우 변경이 용이하다.
- 서비스 정책이나 업무 규칙이 변경되었을 경우 그 변경의 여파를 줄일 수 있다.
- 서비스 정책이나 업무 규칙에 관련된 로직이 별도의 계층으로 격리되어 UI, Application 계층의 복잡도를 낮출 수 있다.
- 서비스 정책이나 업무 규칙에 관련된 로직이 Vue의 개입 없이 순수 자바스크립트로 구현되어 테스트하기 쉽고 공유하기 쉽다.
- 응집력 있는 Domain 계층은 개발자에게 도메인과 관련된 insight 를 제공한다.

**cons:**
- 도메인 계층을 설계하는데 생각보다 큰 비용을 지불해야 한다.
- 협력에 참여하는 요소들의 수가 늘어나 구조가 복잡하게 느껴질 수 있다.

<img width="630" alt="스크린샷 2022-04-20 오후 10 44 43" src="https://user-images.githubusercontent.com/16642635/164244420-8683ff84-3993-4371-b0ef-f7bab7086dc3.png">

<img width="1542" alt="스크린샷 2022-04-20 오후 10 45 19" src="https://user-images.githubusercontent.com/16642635/164244544-ffee52c3-d818-40eb-8efa-7399f9d52f5b.png">

이제 UI와 관련이 없는 기반 기술, API에 대한 상세한 내용, 그리고 UI와 관련이 없는 서비스 정책이나 업무 규칙을 모두 분리해내고 온전히 UI 와 관련된 관심만이 남았다.

남아있는 UI 와 관련된 관심사들은 크게 두 계층 Application 과 단어적 정의 그대로의 UI 로 나눈다.

그리고 이 두 계층 모두 Vue 프레임워크 위에 구현되게 된다.

> [Part4로 이어집니다.](https://genie-youn.github.io/journal/Vue_Application_Architecture_part4.html)
