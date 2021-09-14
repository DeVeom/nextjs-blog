---
title: JavaScript 비동기 part 1, 첫 번째 - Asynchronous programming in JavaScript
date: '2021-09-13'
tags: ['JavaScript', 'Asynchronicity', 'Axel Rauschmayer', 'impatient-js']
draft: false
summary: JavaScript 비동기 학습 내용의 첫 번째 포스트로, Dr. Axel Rauschmayer의 JavaScript for impatient programmers의 비동기 챕터 내용을 통해 JavaScript에서 비동기 프로그래밍이 어떻게 이루어지는지에 대한 개념과 전체적인 흐름 확인한다.
---

# **Asynchronous programming in JavaScript**

**_이 포스트는 학습의 용도로 Dr. Axel Rauschmayer의 JavaScript for impatient programmers를 토대로 하여 작성되었습니다._**

## Contents

1. **A roadmap for asynchronous programming in JavaScript**
   1. Synchronous functions
   2. JavaScript executes tasks sequentially in a single process
   3. Callback-based asynchronous functions
   4. Promise-based asynchronous functions
   5. Async functions
2. **The call stack**
3. **The event loop**
4. **How to avoid blocking the JavaScript process**
   1. The user interface of the browser can be blocked
   2. How can we avoid blocking the browser?
   3. Taking breaks
   4. Run-to-completion semantics
5. **Patterns for delivering asynchronous results**
   1. Delevering asynchronous results via events
   2. Delevering asynchronous results via callbacks
6. **Asynchronous code : the downsides**

## 1. A roadmap for asynchronous programming in JavaScript

### 1) Synchronous functions

일반적인 함수들은 동기적으로 동작한다. 호출자는 피호출자가 계산을 끝마칠 때까지 기다린다. A 줄의 `divideSync()`는 동기적인 함수 호출의 예시이다.

```javascript
function main() {
  try {
    const result = divideSync(12, 3) // (A)
    assert.equal(result, 4)
  } catch (err) {
    assert.fail(err)
  }
}
```

### 2) JavaScript executes tasks sequentially in a single process

기본적으로, JavaScript 작업은 단일 프로세스에서 순차적으로 실행되는 함수이다. 그 모습은 아래의 코드와 같다.

```javascript
while (true) {
  const task = taskQueue.dequeue()
  task() // run task
}
```

이러한 loop는 event loop라고 불리는데, 마우스 클릭과 같은 이벤트들이 큐에 작업을 추가하기 때문이다. 이러한 협동 멀티태스킹(비선점형 멀티태스킹) 방식으로 인해, 작업이 실행되는 동안 다른 작업을 차단(block)하는 것을 원하지 않을 것이다.(예를 들어, 서버에서 오는 결과를 기다리는 등) 다음 섹션에서 이러한 경우를 처리하는 방법을 살펴보자.

### 3) Callback-based asynchronous functions

`divide()`가 결과를 계산하기 위해 서버가 필요하다면 어떻게 될까? 그렇다면 결과가 다른 방식으로 전달되어야 한다. 호출자는 결과가 준비될 때까지 기다릴 필요가 없으며(동기적), 결과가 준비되었을 때 통보를 받아야 한다(비동기적). 결과를 비동기적으로 전달하는 한 가지 방법은 호출자에게 결과를 알리는 콜백 함수를 `divide()`에 제공하는 것이다.

```javascript
function main() {
  divideCallback(12, 3, (err, result) => {
    if (err) {
      assert.fail(err)
    } else {
      assert.equal(result, 4)
    }
  })
}
```

비동기 함수인 `divdeCallback(x, y, callback)`가 호출되면 다음의 단계가 진행된다.

- `divideCallback()` 함수가 서버에 요청(request)을 보낸다.
- 그러면 현재 작업(task)인 `main()`이 종료되고, 다른 작업이 실행된다.
- 서버로부터 응답(response)이 도착하면, 다음 중 하나의 단계가 진행된다.
  - 에러 발생 `err`: 다음 작업이 큐에 추가된다. `taskQueue.enqueue(() => callback(err));`
  - 결과 `r`: 다음 작업이 큐에 추가된다. `taskQueue.enqueue(() => callback(null, r));`

### 4) Promise-based asynchronous funtions

Promise는 다음 두 가지로 정의될 수 있다.

- 콜백 작업을 더 쉽게 만드는 표준 패턴 _A standard pattern that makes working with callbacks easier._
- 비동기 함수가 구축되는 메커니즘 _The mechanism on which async functions are built._

Promise 기반 함수를 호출하는 것은 다음과 같다.

```javascript
function main() {
  dividePromise(12, 3)
    .then((result) => assert.equal(result, 4))
    .catch((err) => assert.fail(err))
}
```

### 5) Async functions

Promise 기반의 코드의 가독성을 높인 것이 async 함수이다.

```javascript
async function main() {
  try {
    const result = await dividePromise(12,3); // (A)
    assert.equal(result, 4);
  } catch (err) {
    assert fail(err);
  }
}
```

A줄의 `dividePromise()` 함수는 이전 섹션의 Promise 기반의 함수와 같은 함수이다. 이전과 다른 것은 호출을 처리하는데 동기식 형태의 문법으로 작성할 수 있다는 것이다. `await`는 특별한 종류의 함수인 `async` 함수 내에서만 사용할 수 있다. `await`는 현재 `async` 함수를 일시 중지하고 반환(return)한다. 대기한(awaited) 결과가 준비되면 함수 실행은 중단된 곳에서 계속된다.

## 2. The call stack

함수는 다른 함수를 호출할 때마다 호출된 함수가 완료된 후 어디로 돌아갈지 기억해야 한다. 이는 일반적으로 스택(호출 스택 the call stack)을 통해 수행된다. 호출자는 반환할 위치를 스택에 푸시하고 피호출자는 작업이 완료된 후 해당 위치로 이동(jump)한다.

다음은 여러 호출이 발생했을 때의 예시이다.

```javascript
function h(z) {
  const error = new Error() // 2
  console.log(error.stack) // 3
}
function g(y) {
  h(y + 1) // 6
} // 7
function f(x) {
  g(x + 1) // 9
} // 10
f(3) // 11
// done // 12
```

이 코드 조각을 실행하기 전, 처음에는 호출 스택은 비어 있는 상태이다. 11번 행의 `f(3)`를 호출한 후, 스택에는 하나의 항목이 존재하게 된다.

- Line 12 (location in top-level scope)

9번 행의 `g(x + 1)`를 호출한 후, 스택에는 두 개의 항목이 존재한다.

- Line 10 (location in `f()`)
- Line 12 (location in top-level scope)

6번 행의 `h(y + 1)`을 호출한 후, 스택에는 세 개의 항목이 존재한다.

- Line 7 (location in `g()`)
- Line 10 (location in `f()`)
- Line 12 (location in top-level scope)

3번 행의 `error` 로그는 다음과 같이 출력된다.

```
DEBUG
Error:
		at h (file://demos/async-js/stack_trace.mjs:2:17)
    at g (file://demos/async-js/stack_trace.mjs:6:3)
    at f (file://demos/async-js/stack_trace.mjs:9:3)
    at file://demos/async-js/stack_trace.mjs:11:1
```

이를 Error 객체가 생성된 위치의 이른바 스택 추적 _stack trace_ 이라고 한다. 반환 위치가 아닌 호출이 일어난 시점에서 기록된다는 점에 주의하자. 2번 행에서 예외를 생성하는 것은 또 다른 호출이며, 이것이 스택 추적에 `h()` 내부의 위치를 포함하는 이유이다.

3번 행 이후, 각 함수가 종료되고 매번 호출 스택에서 최상위 항목이 제거된다. 함수 `f`가 완료되면 top-level scope로 돌아가고 스택은 비어있게 된다. 코드 조각이 종료되면 이는 마치 암시적인 반환(implicit return)과 같다. 만약 코드 조각(code fragment)을 실행되는 작업으로 간주한다면, 빈 호출 스택을 반환함으로써 작업을 종료한다.

## 3. The event loop

기본적으로 JavaScript는 웹 브라우저와 Node.js 모두에서, 단일 프로세스(single process) 내에서 실행된다. 소위 이벤트 루프 _event loop_ 라고 불리는 것은 해당 프로세스 내에서 작업(코드 조각)을 순차적으로 실행시킨다. 이벤트 루프의 구조는 그림과 같다.

![Task sources add code to run to the task queue, which is emptied by the event loop](https://i.imgur.com/b1XIt2i.png)

그림에서 볼 수 있듯이 두 개의 영역이 작업 대기열 _task queue_ 에 엑세스하고 있다.

- 작업 소스 _task sources_ 는 대기열 queue에 작업을 추가한다. 이러한 소스 중 일부는 JavaScript 프로세스와 동시에 실행된다. 예를 들어, 사용자 인터페이스 이벤트를 처리하는 하나의 소스가 있다고 할 때, 사용자가 어딘가를 클릭하고 이벤트 리스너가 등록된 경우 해당 리스터의 호출이 작업 대기열에 추가된다.
- 이벤트 루프 _event loop_ 는 JavaScript 프로세스 내에서 계속해서 실행된다. 각 루프가 반복될 동안 대기열 queue 에서 하나의 작업을 가져와 실행한다. (대기열이 비어 있으면, 비어 있지 않을 때까지 기다린다.) 해당 작업은 호출 스택이 비어있고 반환되었을 때만 완료된다. 이벤트 루프로 제어가 다시 넘어가고 대기열 queue 에서 다음 작업을 검색하고 실행한다.

다음 JavaScript 코드는 이벤트 루프와 비슷한(이벤트 루프에 근사한) 코드이다.

```javascript
while (true) {
  const task = taskQueue.dequeue()
  task() // run task
}
```
