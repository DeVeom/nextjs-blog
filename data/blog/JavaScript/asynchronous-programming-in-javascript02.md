---
title: JavaScript 비동기 part 1, 두 번째 - Asynchronous programming in JavaScript
date: '2021-09-13'
tags: ['JavaScript', 'Asynchronicity', 'Axel Rauschmayer', 'impatient-js']
draft: false
summary: JavaScript 비동기 학습 내용의 첫 번째 포스트로, Dr. Axel Rauschmayer의 JavaScript for impatient programmers의 비동기 챕터 내용을 통해 JavaScript에서 비동기 프로그래밍이 어떻게 이루어지는지에 대한 개념과 전체적인 흐름 확인한다.
---

## 4. How to avoid blocking the JavaScript process

### 1) The user interface of the browser can be blocked

브라우저의 많은 사용자 인터페이스 메커니즘은 JavaScript 프로세스(작업으로)에서도 실행 된다. 따라서 오랫동안 실행되는 JavaScript 코드는 사용자 인터페이스를 차단할 수 있다. 이를 보여주는 웹 페이지를 살펴보자.

다음 HTML은 해당 페이지의 유저 인터페이스이다.

```html
<a id="block" href="">Block</a>
<div id="statusMessage"></div>
<button>Click me!</button>
```

"Block"을 클릭하면 JavaScript를 통해 긴 시간의 루프가 실행된다. 해당 루프 동안에는 브라우저, JavaScript 프로세스가 차단되어 버튼을 클릭할 수 없다. 이를 구현한 간단한 JavaScript 코드는 다음과 같다.

```javascript
document.getElementById('block').addEventListner('click', doBlock) // (A)

function doBlock(event) {
  // ...
  displayStatus('Blocking...')
  // ...
  sleep(5000) // (B)
  displayStatus('Done')
}

function sleep(milliseconds) {
  const start = Date.now()
  while (Date.now() - start < milliseconds);
}
function displayStatus(status) {
  document.getElementById('statusMessage').textContent = status
}
```

위의 코드 진행을 풀어보면 다음과 같다.

- Line A: ID가 block인 HTML 요소를 클릭할 때마다 브라우저에 `doBlock()`을 호출하도록 지시한다.
- `doBlock()`은 상태 정보를 표시한 다음 `sleep()`을 호출하여 5000밀리초 동안 JavaScript 프로세스를 차단한다. (Line B)
- `sleep()`은 충분한 시간이 경과할 때까지 반복하여 JavaScript 프로세스를 차단한다.
- `displayStatus()`는 ID가 statusMessage인 <div> 내에 상태 메시지를 표시한다.

### 2) How can we avoid blocking the browser?

오래 실행되는 작업이 브라우저를 차단하는 것을 방지할 수 있는 몇 가지 방법이 있다.

- 작업의 결과를 비동기적으로 전달 할 수 있다.

  : 다운로드와 같은 일부 작업은 JavaScript 프로세스와 동시에 수행될 수 있다. 이러한 작업을 발생시키는 JavaScript 코드는 작업이 완료되면 결과와 함께 호출되는 콜백을 등록한다. 호출은 작업 대기열 _task queue_ 을 통해 처리된다. 호출자가 결과가 준비될 때까지 기다리지 않기 때문에 결괄르 전달하는 이런 방식을 비동기적이라고 한다. 일반적인 함수 호출은 결과를 동기적으로 전달한다.

- 별도의 프로세스에서 긴 계산을 수행한다.

  : 이른바 *Web Workers*를 통해 이를 수행할 수 있다. Web Workers는 메인 프로세스와 동시에 실행되는 무거운 프로세스이다. 각각 자체 런타임 환경(전역 변수 등)을 가지고 있으며, 완전히 독립되어 있고 메시지 전달을 통해 통신해야 한다. 자세한 내용은 해당 내용을 다룬 [MDN 문서](https://developer.mozilla.org/ko/docs/Web/API/Web_Workers_API)를 참고하자.

  > 웹 워커(Web worker)는 스크립트 연산을 웹 어플리케이션의 주 실행 스레드와 분리된 별도의 백그라운드 스레드에서 실행할 수 있는 기술입니다. 웹 워커를 통해 무거운 작업을 분리된 스레드에서 처리하면 주 스레드(보통 UI 스레드)가 멈추거나 느려지지 않고 동작할 수 있습니다.
  >
  > _Web Workers API_, MDN Web Docs

- 긴 계산 동안 작업을 중단한다. (다음 섹션을 통해 그 방법을 알아보자.)

### 3) Taking breaks

다음 전역 함수는 ms 밀리초의 지연 후에 매개변수 콜백을 실행한다. (`setTimeout()에 대한 단순한 설명이며, 더 많은 기능이 있다.)

```javascript
function setTimeout(callback: () => void, ms: number): any
```

위의 함수는 다음 전역 함수를 통해 타임아웃(콜백 실행 취소)을 지우는데(_clear_) 사용할 수 있는 _handle_(ID)을 반환한다.

```javascript
function clearTimeout(handel?: any): void
```

`setTimeout()`은 브라우저와 Node.js 모두에서 사용 가능하다.

**_`setTimeout()`은 작업들을 중단시킨다._** 이는 `setTimeout()` 이 현재 작업을 중단하고 콜백을 통해 나중에 계속 된다는 것을 의미한다.

### 4) Run-to-completion semantics

JavaScript는 그 안에서 이루어지는 작업들에 대해서 다음을 보장한다.

_각 작업은 다음 작업이 실행되기 전에 항상 완료된다. ("run to completion")_

결과적으로, 각 작업은 진행되는 동안 데이터가 변경되는 것에 걱정할 필요가 없다(동시 수정 _concurrent modification_) . 이는 JavaScript 프로그래밍을 단순화한다. 다음의 예시는 이러한 보장을 보여준다.

```javascript
console.log('start')
setTimeout(() => {
  console.log('callback')
}, 0)
console.log('end')

// Output:
// 'start'
// 'end'
// 'callback'
```

`setTimeout()`은 매개변수를 작업 대기열 task queue 에 넣는다. 따라서 매개변수는 현재 코드(작업)가 완전히 완료된 후에 실행된다.

`ms` 매개변수는 작업이 정확히 실행될 때가 아니라 작업이 대기열 queue 에 들어갈 때만 지정된다. 예를 들어 대기열에 종료되지 않은 작업이 있는 경우엔, 실행되지 않을 수 있다. 이는 위 코드의 매개변수 `ms`의 값이 0임에도 `'callback'`보다 먼저 `'end'`를 출력하는지를 설명한다.

## 5. Patterns for delivering asynchronous results

긴 실행 작업이 완료될 때까지 기다리는 동안 메인 프로세스가 차단되는 것을 방지하기 위해 JavaScript에서 결과가 비동기적으로 전달되는 경우가 많다. 다음은 결과를 비동기적으로 전달하는 세 개의 대중적인 패턴이다.

- Events
- Callbacks
- Promises

이번 포스트에서는 첫 번째와 두 번째 패턴을 먼저 확인하고, Promises는 다음 포스트에서 다뤄보도록 하겠다.

### 1) Delivering asynchronous results via events

패턴으로서의 이벤트는 다음과 같이 동작한다.

- 값을 비동기적으로 전달하는데 사용된다.
- 0번 이상 그렇게 동작한다.
- 이벤트 패턴에는 세 가지 역할이 있다.
  - _event_(an object)는 전달되어야 하는 데이터를 운반한다.
  - *event listner*는 매개변수를 통해 이벤트를 수신하는 함수이다.
  - *event source*는 이벤트를 전송하고 event listner를 등록할 수 있도록 한다.

JavaScript의 세계에는 이 패턴의 여러 유형이 있다. 다음 세 가지 예시를 살펴보자.

**1-1) Events: IndexedDB**

IndexedDB는 웹 브라우저에 내장된 데이터베이스이다. 다음은 그 사용 예시이다.

```javascript
const openRequest = indexedDB.open('MyDatabase', 1) // (A)

openRequest.onsuccess = (event) => {
  const db = event.target.result
  // ...
}

openRequest.onerror = (error) => {
  console.error(error)
}
```

`indexedDB`에는 작업을 호출하는 일반적이지 않은 방법이 있다.

- 각 작업에는 요청 객체 _request objects_ 를 만들기 위한 관련 메서드가 존재한다. 예를 들어, Line A 에서 작업은 "open"이고, 메서드는 `.open()`이며, 요청 객체는 `openRequest`이다.
- 작업에 대한 매개변수는 메소드의 매개변수가 아니라 요청 객체를 통해 제공된다. 예를 들어, 이벤트 리스너(함수)는 `.onsuccess`와 `.onerror` 속성에 저장된다.
- 작업 호출은 메서드(inline A)를 통해 작업 대기열 _task queue_ 에 추가된다. 즉, 호출이 이미 대기열에 추가된 후 해당 작업을 구성한다. run-to-completion 구문만이 이러한 경쟁 조건을 빠져나올 수 있게 해주고, 현재 코드 조각이 완료된 후에 작업이 실행되도록 한다.

**1-2) Events: XMLHttpRequest**

XMLHttpRequest API를 사용하면 웹 브라우저에서 다운로드 작업을 할 수 있다. 다음은 http://example.com/textfile.txt 파일을 다운로드 하는 예시이다.

```javascript
const xhr = new XMLHttpRequest() // (A)
xhr.open('GET', 'http://example.com/textfile.txt') // (B)
xhr.onload = () => {
  // (C)
  if (xhr.status == 200) {
    processData(xhr.responseText)
  } else {
    assert.fail(new Error(xhr.statusText))
  }
}
xhr.onerror = () => {
  // (D)
  assert.fail(new Error('Network error'))
}
xhr.send() // (E)

function processData(str) {
  assert.equal(str, 'Content of textfile.txt\n')
}
```

이 API를 사용하여 먼저 요청 객체를 생성한 다음(Line A), 객체를 구성하고, 활성화한다(Line E). 이 구현은 다음과 같이 구성된다.

- 사용할 HTTP 요청 방법 지정(Line B): `GET`, `POST`, `PUT`, etc.
- 뭔가를 다운로드할 수 있다는 알림을 받을 수 있는 리스너를 등록한다(Line C). 리스너 내부에서 요청한 다운로드가 포함되었는지 혹은 오류를 전달하는지 파악해야 한다. 결과 데이터의 일부는 요청 객체인 `xhr`을 통해 전달된다는 점을 주목하자.
- 네트워크 오류가 발생한 경우 알림을 받는 리스너를 등록한다(Line D).

**1-3) Events: DOM**

우리는 이미 DOM 이벤트의 동작을 4-1. "The user interface of the browser can be blocked" 챕터에서 살펴봤다. 다음은 `click` 이벤트를 처리하는 예시 코드이다.

```javascript
const element = document.getElementById('my-link') // (A)
element.addEventListner('click', clickLisnter) // (B)

function clickListener(event) {
  event.preventDefault() // (C)
  console.log(event.shiftKey) // (D)
}
```

먼저 브라우저에 ID가 `'my-link'`인 HTML 요소를 검색하도록 요청한다(Line A). 그런 다음 모든 `click` 이벤트에 대한 리스너를 추가한다(Line B). 리스너에서 우리는 먼저 브라우저에 기본 작업(해당 링크로 이동하는)을 수행하지 않도록 지시한다(Line C). 그런 다음 shift 키가 현재 눌러져 있으면 콘솔에 출력한다(Line D).

### 2) Delivering asynchronous results via callbacks

콜백은 비동기 결과를 처리하기 위한 또 다른 패턴이다. 일회성 결과에만 사용되며 이벤트보다 덜 장황하다는(코드가 간결하다는) 장점이 있다.

예를 들어, 텍스트 파일을 읽고 그 내용을 비동기적으로 반환하는 함수 `readFile()`을 떠올려보자. Node.js 스타일의 콜백을 사용하는 경우에`readFile()`을 호출하는 방법은 다음과 같다.

```javascript
readFile('some-file.txt', { encoding: 'utf8' }, (error, data) => {
  if (error) {
    assert.fail(error)
    return
  }
  assert.equal(data, 'The content of some-file.txt\n')
})
```

위의 코드에서는 성공과 실패를 처리하는 하나의 콜백이 존재한다. 첫 번째 매개변수가 `null`이 아니면 오류가 발생한 것이고, 그렇지 않다면 두 번째 매개변수에서 결과를 찾을 수 있다.

## 6. Asynchronous code: the downsides

우리는 브라우저나 Node.js 내의 많은 상황에서 선택의 여지 없이 비동기적 코드를 사용해야 한다. 앞서 그러한 코드를 사용할 수 있는 몇 가지 패턴을 살펴보았다. 이러한 패턴 모두 다음과 같은 두 가지 단점이 존재한다.

- 비동기적 코드는 동기적 코드보다 더 장황하다. (복잡하고 가독성이 좋지 않다.)
- 비동기적 코드를 호출하는 경우, 호출하는 코드도 비동기적으로 동작되어야 한다. 비동기적 결과를 동기적으로 기다릴 수 없기 때문이다. 즉, 비동기적 코드는 전염성을 가지고 있다.

첫 번째 단점은 Promise를 사용하면 많이 개선되고, async 함수를 사용함으로써 대부분 사라진다.

그럼에도 비동기적 코드의 전염성은 사라지지 않지만, async 함수를 사용하면 쉽게 동기와 비동기 간의 전환을 할 수 있다는 점에서 그 단점이 조금 완화된다.

---

여기까지 *Axel Rauschmayer*의 *JavaScript for impatient programmers*의 _chater 41. Asynchronous programming in JavaScript_ 를 통해 자바스크립트 내에서 비동기적인 프로그래밍을 구현하는 방법과 전체적인 흐름을 살펴볼 수 있었다.

다음 'JavaScript 비동기 part 2' 포스트에서는 _chater 42. Promises for asynchronous programming [ES6]_ 를 통해 Promise와 그 사용 방법 등을 공부하고 다루어 보도록 하겠다.
