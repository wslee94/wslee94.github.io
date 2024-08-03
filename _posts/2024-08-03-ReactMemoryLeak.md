---
title: 리액트 메모리 누수 (feat. 클로저, 메모이제이션)
date: 2024-08-03 13:20:00 +09:00
categories: [Knowledge, React]
tags: [react, memoization, closure] # TAG는 반드시 소문자로 이루어져야함!
# image: /assets/img/avatar.png
# pin: true
---

# 들어가기 앞서
현재 대부분 Vue 3를 이용해 회사 프로젝트를 진행하고 있지만, 커피챗 도중 리액트 메모리 누수 아티클을 읽었는데 흥미로워 따로 정리하려고 한다. 원본 아티클은 [Sneaky React Memory Leaks: How useCallback and closures can bite you](https://schiener.io/2024-03-03/react-closures){:target="_blank"}이다.

# 클로저
리액트 메모리 누수에 대해 알아보기 전 클로저가 무엇인지 간단하게 알아보자!

> **Closures? (MDN)** <br />
> **A closure is the combination of a function bundled together (enclosed) with references to its surrounding state (the lexical environment).** In other words, a closure gives you access to an outer function's scope from an inner function. In JavaScript, closures are created every time a function is created, at function creation time.
{: .prompt-tip }

위의 굵은 글씨를 해석하면, 클로저는 주변 상태(the lexical environment)에 대한 참조와 함께 번들링된 함수의 조합이다. 이 설명만으로는 잘 와닿지 않으니, MDN 예제 코드를 보면서 따라가 보자.

**Lexical Scope**<br />
Javascript는 Lexical Scope를 따르는데 다른 말로 **함수의 스코프는 실행 시점이 아니라 선언 시점**에 결정된다.
```javascript
function init() {
  const name = "Mozilla"; 
  function displayName() {
    console.log(name); // "Mozilla"
  }
  displayName();
}
init();
```
위 코드를 보면, `displayName()`을 선언하는 시점의 외부 함수의 스코프에 접근할 수 있다. 즉 중첩된 함수 구조에서 내부 함수는 선언 시점의 외부 범위에서 선언한 변수에 접근할 수 있다.

**클로저**
```javascript
function makeFunc() {
  const name = "Mozilla";
  function displayName() {
    console.log(name); // "Mozilla"
  }
  return displayName;
}

const myFunc = makeFunc();
myFunc();
```
위 코드를 보면, `const myFunc = makeFunc()`을 통해 `makeFunc`함수는 `displayName` 내부 함수를 반환하고 실행을 종료한다. `myFunc()`를 실행하면 내부 함수가 실행되면서 종료된 외부 함수의 변수인 `name`에 접근할 수 있다. 실행이 종료된 외부 함수의 스코프에 어떻게 접근할 수 있는 걸까? 그 이유는 Javascript 함수가 클로저를 형성하기 때문이다. **클로저는 중첩된 함수 구조에서 외부 함수가 종료되어도 내부 함수가 선언된 시점의 스코프(=Lexical Scope)를 유지한다.**

# 클로저와 리액트
```react
import { useState, useEffect } from "react";

function App({ id }) {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(count + 1); // This is a closure over the count variable
  };

  useEffect(() => {
    console.log(id); // This is a closure over the id prop
  }, [id]);

  return (
    <div>
      <p>{count}</p>
      <button onClick={handleClick}>Increment</button>
    </div>
  );
}
```
리액트는 클로저와 긴밀하게 연관되어 있다. (함수형 컴포넌트, 훅, 이벤트 핸들러 등)

위의 예제 코드에서 `handleClick`이 왜 클로저일까?
1. 외부 함수인 `App`은 `return` 문을 만나 종료됐다.
2. `<button />` 의 클릭 이벤트에 연결된 `handleClick` 내부 함수는 종료된 외부 함수의 스코프(`count`)에 접근할 수 있다.

위 코드 자체로 메모리 누수는 발생하지 않는다. 왜냐하면 `App` 컴포넌트가 렌더링 될 때마다 새로운 클로저가 생성되고 이전 클로저는 메모리에서 해제된다. **하지만, 최적화를 위해 Memoization 기술이 들어가면 특정 상황에서 메모리 해제가 되지 않을 수 있다는 게 이 아티클의 주제이다!**

# useCallBack과 메모리 누수 
```react
import { useState, useCallback } from "react";

class BigObject {
  public readonly data = new Uint8Array(1024 * 1024 * 10);
}

export const App = () => {
  const [countA, setCountA] = useState(0);
  const [countB, setCountB] = useState(0);
  const bigData = new BigObject(); // 10MB of data

  const handleClickA = useCallback(() => {
    setCountA(countA + 1);
  }, [countA]);

  const handleClickB = useCallback(() => {
    setCountB(countB + 1);
  }, [countB]);

  return (
    <div>
      <button onClick={handleClickA}>Increment A</button>
      <button onClick={handleClickB}>Increment B</button>
      <p>
        A: {countA}, B: {countB}
      </p>
    </div>
  );
};
```
위 코드의 `handleClickA`, `handleClickB` 번갈아 가면서 각각 5번씩 클릭하면 어떻게 될까?

<img src="/assets/img/capture/react-memory-leak-4.png" alt="리액트 useCallback 사용 시 메모리 누수 화면" /> <br />
메모리 해제가 되지 않아 `bigData`가 11번 중첩되어 메모리에 적재된 것을 확인할 수 있다. 왜 이런 현상이 발생한 걸까? 아래 두 가지를 기억하고 각 단계를 살펴보자!
- `handleClickA`, `handleClickB`는 `App`이 실행(렌더링)될 때마다 외부 스코프에 접근할 수 있는 클로저 스코프를 생성한다.
- `handleClickA`, `handleClickB`에서 사용하지 않는 `bigData`도 결국 외부 함수의 스코프이므로 클로저 스코프에 포함된다.
- `useCallback`의 두 번째 인자인 의존성 배열에 입력한 상태 값이 변경되지 않으면 함수는 Memoization 된다.

**초기 렌더링** <br />
<img width=214 src="/assets/img/capture/react-memory-leak-5.png" alt="closure chain 0" /><br />
`handleClickA`과 `handleClickB`은 동일한 외부 함수 스코프를 갖고있다.

**Increment A 1회 클릭** <br />
<img width=1060 src="/assets/img/capture/react-memory-leak-6.png" alt="closure chain 1" /><br />
`handleClickA`는 `countA` 값이 변경되어 다시 생성됐다. (handleClickA#0 → handleClickA#1) <br />
`handleClickB`는 `countB` 값이 변경되지 않아 메모리 주소 값이 유지된다. (hold handleClickB#0)<br />
**여기서! handleClickB#0 은 메모리 해제되지 않아 외부 함수 스코프(App-Scope#0)는 그대로 유지한다.**

**Increment B 1회 클릭** <br />
<img width=1060 src="/assets/img/capture/react-memory-leak-7.png" alt="closure chain 2" /><br />
`handleClickA`는 `countA` 값이 변경되지 않아 메모리 주소 값이 유지된다. (hold handleClickA#1)<br />
`handleClickB`는 `countB` 값이 변경되어 다시 생성됐다. (handleClickB#0 → handleClickB#1)<br />
**여기서! handleClickA#1 은 메모리 해제되지 않아 외부 함수 스코프(App-Scope#1)는 그대로 유지한다.**

**Increment A 2회 클릭** <br />
<img width=1060 src="/assets/img/capture/react-memory-leak-8.png" alt="closure chain 3" /><br />
`handleClickA`는 `countA` 값이 변경되어 다시 생성됐다. (handleClickA#1 → handleClickA#2)<br />
`handleClickB`는 `countB` 값이 변경되지 않아 메모리 주소 값이 유지된다. (hold handleClickB#1)<br />
**여기서! handleClickB#1 은 메모리 해제되지 않아 외부 함수 스코프(App-Scope#2)는 그대로 유지한다.**

이런 식으로 외부 함수인 `App`의 스코프가 체이닝 되면서 Garbage collector 의해 메모리가 해제되지 않는 문제점이 발생한다.만약 메모이제이션(useCallback)을 하지 않았다면 각각 클릭마다 `handleClickA`, `handleClickB`가 모두 새롭게 생성되면서 이전에 참조 했던 외부 함수 스코프와의 연결고리는 끊어진다. 따라서 이전 외부함수의 스코프를 통해 접근할 수 있었던 변수, 함수 등이 Garbage collector에 의해 폐기되고 위의 메모리 누수는 나타나지 않을 것이다.

위의 예제에서는 `useCallback`을 2개밖에 사용하지 않았지만 이보다 더 많이 사용하고 컴포넌트 코드가 많다고 상상하면 메모리가 언제 해제되는지 파악하기가 매우 어려워질 것이다. 

# 결론
아티클에서 리액트 메모리 누수를 회피하기 위한 팁을 알려주고 있다.
>1. Keep your closure scopes as small as possible.
   - Write smaller components. This will reduce the number of variables that are in scope when you create a new closure.
   - Write custom hooks. Because then any callback can only close over the scope of the hook function. This will often only mean the function arguments.
2. Avoid capturing other closures, especially memoized ones.
3. Avoid memoization when it’s not necessary.

**"덮어 놓고 Memoization 쓰다 보면 메모리 누수 꼴을 못 면한다."**