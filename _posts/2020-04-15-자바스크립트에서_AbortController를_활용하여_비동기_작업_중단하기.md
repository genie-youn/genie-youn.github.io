---
layout: abortControllerPost
title: "자바스크립트에서 AbortController 를 활용하여 비동기 작업 중단하기"
author: "genie-youn"
categories: journal
tags: [javascript, AbortController]
image: DSC04273.jpg
---

## 들어가며
자바스크립트에서 Fetch 요청을 중단시킬 수 있는 AbortController 에 대해 소개하고, 이를 응용하여 비동기 테스크를 중단시키는 방법을 기술한 [Aborting a signal: How to cancel an asynchronous task in JavaScript](https://ckeditor.com/blog/Aborting-a-signal-how-to-cancel-an-asynchronous-task-in-JavaScript/) 를 저자의 허락을 얻어 번역한다.

## 전문
비동기 작업은 다루기 까다롭다. 실수로 시작되거나 더 이상 필요 없는 비동기 작업에 대하여 중단할 방법을 제공하지 않는 몇몇 프로그래밍 언어에서는 더욱이 까다롭게 느껴진다. 다행히도, 자바스크립트에서는 비동기 작업을 손쉽게 중단할 수 있는 방법을 제공한다.

### Abort signal
`Promise` 가 ES2015를 통해 표준 스펙에 포함되고 몇몇 Web API 들이 이 새로운 비동기 해결책을 지원하기 시작하자 중간에 [비동기 작업을 취소해야 하는 필요성](https://github.com/whatwg/fetch/issues/27)이 대두되었다. 이 문제를 해결하기 위한 첫 번째 시도는 추후 [ECMAScript의 표준 스펙으로 포함될 수 있는 범용적인 해결책](https://github.com/tc39/proposal-cancellation)을 만드는 데 초점을 두었다. 하지만 얼마가지 못해 문제를 해결하지 못한 채 논의가 중단되었다.

그 후 WHATWG는 독자적으로 [`AbortController` 를 DOM에 도입](https://dom.spec.whatwg.org/#aborting-ongoing-activities)한다. 이 `AbortController` 는 Node.js 환경에선 사용할 수 없기 때문에 명백한 단점을 가지고 있다고 할 수 있다.

DOM 스펙문서에서 확인할 수 있듯, `AbortController` 는 범용적으로 설계되어 있다. 그렇게 때문에 어떤 비동기 API에도, 심지어 아직 존재하지 않는 API 일지라도 사용가능하다. (현재는 `Fetch API` 만이 공식적으로 지원한다.)

그럼 `AbortController` 가 어떻게 동작하는지 잠시 살펴보도록 하자.

```javascript
const abortController = new AbortController(); // 1
const abortSignal = abortController.signal; // 2

fetch( 'http://example.com', {
  signal: abortSignal // 3
} ).catch( ( { message } ) => { // 5
  console.log( message );
} );

abortController.abort(); // 4
```

위의 코드를 보면, 우선 `AbortController` DOM 인터페이스의 새로운 인스턴스를 만든 후 (1), 인스턴스의 `signal` 프로퍼티를 (2) `fetch` 의 `signal` 옵션에 할당하는 것을 볼 수 있다. (3)

패칭을 중단하기 위해서는 단순히 `abortController.abort()` 를 호출하기만 하면 된다. (4) `abort` 를 호출하게 되면 `fetch` 의 Promise 는 자동으로 `reject` 되게 되고 제어는 `catch()` 블럭으로 진입하게 된다. (5)

이 흥미로운 `signal` 프로퍼티가 가장 중요한 부분이다. 이 프로퍼티는 사용자가 이미 `abortController.abort()` 메소드를 호출했는지 여부에 대한 정보를 나타내는 `aborted` 프로퍼티를 가진  [AbortSignal DOM interface](https://dom.spec.whatwg.org/#interface-AbortSignal) 의 인스턴스이다. 또한 `abortController.abort()` 가 호출될때 발생하는 `abort` 이벤트에 리스너를 등록할 수 있다. 즉, `AbortController` 는 단지 `AbortSignal` 의 퍼블릭 인터페이스일 뿐이다.

`AbortController` 를 공식적으로 지원하는 `Fetch API` 외에도, 일반적으로 우리가 작성하는 비동기 작업을 다루는 코드에서도 사용할 수 있다.

### Abortable function
매우 복잡한 연산을 하는 비동기 함수(예를들어 굉장히 큰 배열을 비동기적으로 처리하는 함수) 가 있다고 상상해보자. 예제를 간단하게 하기 위해 5초동안 복잡한 연산을 하고 가정하고, 5초 뒤에 결과를 반환하는 함수로 대체한다.

```javascript
function calculate() {
  return new Promise( ( resolve, reject ) => {
    setTimeout( ()=> {
      resolve( 1 );
    }, 5000 );
  } );
}

calculate().then( ( result ) => {
  console.log( result );
} );
```

그리고 이 비용이 큰 연산을 중간에 취소하고 싶은 사용자들을 위해 연산을 시작하고, 중단할 수 있는 버튼을 추가하자.

```html
<button id="calculate">Calculate</button>

<script type="module">
  document.querySelector( '#calculate' ).addEventListener( 'click', async ( { target } ) => { // 1
    target.innerText = 'Stop calculation';

    const result = await calculate(); // 2

    alert( result ); // 3

    target.innerText = 'Calculate';
  } );

  function calculate() {
    return new Promise( ( resolve, reject ) => {
      setTimeout( ()=> {
        resolve( 1 );
      }, 5000 );
    } );
  }
</script>
```

위의 코드를 보면 우선, 버튼의 `click` 이벤트에 비동기로 동작하는 함수를 리스너 등록한다. (1) 이 리스너는 `calculate()` 를 호출하게 된다. (2) 5초가 지난 후 결과를 표시하는 알럿이 노출되게 된다. (3)

이제 이 비동기작업을 취소하는 기능을 추가해보자.

```javascript
{ // 1
  let abortController = null; // 2

  document.querySelector( '#calculate' ).addEventListener( 'click', async ( { target } ) => {
    if ( abortController ) {
      abortController.abort(); // 5

      abortController = null;
      target.innerText = 'Calculate';

      return;
    }

    abortController = new AbortController(); // 3
    target.innerText = 'Stop calculation';

    try {
      const result = await calculate( abortController.signal ); // 4

      alert( result );
    } catch {
      alert( 'WHY DID YOU DO THAT?!' ); // 9
    } finally { // 10
      abortController = null;
      target.innerText = 'Calculate';
    }
  } );

  function calculate( abortSignal ) {
    return new Promise( ( resolve, reject ) => {
      const timeout = setTimeout( ()=> {
        resolve( 1 );
      }, 5000 );

      abortSignal.addEventListener( 'abort', () => { // 6
        const error = new DOMException( 'Calculation aborted by the user', 'AbortError' );

        clearTimeout( timeout ); // 7
        reject( error ); // 8
      } );
    } );
  }
}
```

코드가 훨씬 더 길어졌지만, 당황하지 말고 천천히 들여다 보도록 하자. 아마 쉽게 이해할 수 있을 것이다.

모든 코드는 블럭으로 감싸져 있고 (1) 이는 마치 [IIFE (즉시 실행 함수)](https://exploringjs.com/es6/ch_core-features.html#sec_from-iifes-to-blocks) 와 같은 효과를 갖는다. 이 덕분에 `abortController` 변수를 전역 스코프로 불필요하게 노출하지 않을 수 있다. (2)

우선 이 값을 `null` 로 초기화한다. 이 값은 버튼을 마우스로 클릭하면 `AbortController` 의 새로운 인스턴스로 바뀌게 된다. (3) 이후 인스턴스의 `signal` 프로퍼티를 `calculate()` 함수에 인자로 전달하게 된다. (4)

사용자가 5초가 지나기 전에 버튼을 다시 클릭하면, `abortController.abort()` 를 호출하게 된다. (5)  이것은 다시 `calculate()` 에 인자로 전달했던 `AbortSignal` 인스턴스의 `abort` 이벤트를 발생시키게 된다. (6)

`abort` 이벤트의 리스너 내부에선 동작하고 있던 타이머를 중단시키고 (7) 적절한 에러와 함께 프로미스를 `reject` 한다. (8; [스펙에 의하면](https://dom.spec.whatwg.org/#abortcontroller-api-integration) 이 에러는 `AbortError` 타입의 `DOMException` 이어야만 한다.) 이 에러는 제어를 `catch` 와 `finally` 블럭으로 넘기게 된다. (10)

추가적으로 다음과 같은 상황을 대비할 코드도 작성해야 한다.

```javascript
const abortController = new AbortController();

abortController.abort();
calculate( abortController.signal );
```

위와 같은 경우에, `abort` 이벤트가 `signal` 프로퍼티가 `calculate()` 함수에 전달되기 전에 발생했기 때문에 이벤트 리스너가 동작하지 않게 된다. 이러한 경우에도 정상적으로 동작할 수 있도록 조금 리팩토링 해보자.

```javascript
function calculate( abortSignal ) {
  return new Promise( ( resolve, reject ) => {
    const error = new DOMException( 'Calculation aborted by the user', 'AbortError' ); // 1

    if ( abortSignal.aborted ) { // 2
      return reject( error );
    }

    const timeout = setTimeout( ()=> {
      resolve( 1 );
    }, 5000 );

    abortSignal.addEventListener( 'abort', () => {
      clearTimeout( timeout );
      reject( error );
    } );
  } );
}
```

에러를 우선 제일 위로 옮긴다. (1) 이 덕분에 두 부분의 코드를 재활용할 수 있게 되었다. 그리고 이미 요청이 취소되었는지를 확인하는 `abortSignal.aborted` 에 대한 가드를 추가한다. (2)

이 가드가 `true` 로 평가된다면 `calculate()` 함수는 더이상의 처리 없이 Promise 를 에러와 함께 거절한다.

### 마치며
자바스크립트에서 `fetch` 를 비롯한 비동기 작업을 중단할 수 있는 `AbortController` 에 대하여 알아보았다.

실제 동작하는 예제는 [https://blog.comandeer.pl/assets/i-ciecie/](https://blog.comandeer.pl/assets/i-ciecie/) 에서 확인할 수 있으며,

아티클 원문은 [https://ckeditor.com/blog/Aborting-a-signal-how-to-cancel-an-asynchronous-task-in-JavaScript/](https://ckeditor.com/blog/Aborting-a-signal-how-to-cancel-an-asynchronous-task-in-JavaScript/) 에서 확인할 수 있다.
