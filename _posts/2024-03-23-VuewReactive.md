---
title: Vue 3 반응형 원리
date: 2024-03-23 14:20:00 +09:00
categories: [Deveploment, Vue]
tags: [vue, vue3, reactive] # TAG는 반드시 소문자로 이루어져야함!
# image: /assets/img/avatar.png
# pin: true
---

# 배경
Maplestory Worlds 관련 프로젝트는 Vue 라이브러리를 사용한다. Vue 2 → Vue 3 마이그레이션을 진행하는 중 Vue 3는 반응형을 어떻게 구현했는 지 궁금해서 정리해보았다.

# 반응형이란?
공식 홈페이지에서 반응성은 선언적인 방식으로 변화에 적응할 수 있는 프로그래밍 패러다임 이라고 소개하고 있다. 

> **선언적인 방식?** <br />
> 어떻게(How)에 대한 부분은 추상화 하고 무엇을(What)에 집중하는것을 선언적 프로그래밍이라고 한다. <br /> 아래 코드를 보자.
{: .prompt-tip }

```javascript
// 명령형 방식 (HOW)
function double(arr) {
  let results = [];
  for (let i = 0; i < arr.length; i++) {
      results.push(arr[i] * 2)
  }
  return results;
}
 
// 선언형 방식 (WHAT)
function double(arr) {
  return arr.map((item) => item * 2)
}
```
→ 명령형 방식에서 `for`문 내부의 변수가 외부로부터 노출되지 않도록 `map`이란 함수로 캡슐화 시켰기 때문에 선언형이 되었다고 해석이 가능하다.

엑셀을 통해 반응형을 설명해보자.

![엑셀로 바라본 반응형](/assets/img/capture/vue3-reactivity-excel.png)

`A2 = A0 + A1` 수식을 사용했을 때 `A0` 혹은 `A1` 값이 변경되면 `A2`도 자동으로 업데이트된다. 이게 반응형이라고 한다. 하지만 자바스크립트에서는 별도 처리 없이 위와 같은 반응형은 동작하지 않는다.
```javascript
let A0 = 1
let A1 = 2
let A2 = A0 + A1
 
console.log(A2) // 3
 
A0 = 2
console.log(A2) // 여전히 3
```

# Vue 3에서 반응형 동작 원리
객체 속성의 읽기 및 쓰기를 가로채는 방식을 사용해 반응형을 구현한다. 아래는 Vue 3에서 데이터를 정의할 때 자주 사용하는 `reactive`와 `ref` 함수 구현 코드다.

```javascript
function reactive(obj) {
  return new Proxy(obj, {
    // 객체 속성을 읽을 때 호출
    get(target, key) {
      track(target, key)
      return target[key]
    },
    // 객체 속성을 변경할 때 호출
    set(target, key, value) {
      target[key] = value
      trigger(target, key)
    }
  })
}
 
function ref(value) {
  const refObject = {
    // 객체 속성을 읽을 때 호출
    get value() {
      track(refObject, 'value')
      return value
    },
    // 객체 속성을 변경할 때 호출
    set value(newValue) {
      value = newValue
      trigger(refObject, 'value')
    }
  }
  return refObject
}
```

## track
객체의 속성과 의존 관계에 있는 함수를 등록한다 → 반응형으로 실행할 코드를 저장한다. 여기서 effects란 객체의 속성이 구독하고 있는 반응형 이벤트 함수들을 관리하는 Set 자료구조이다.

```javascript
let activeEffect // 이 변수의 역할은 아래에 설명하겠다.
 
function track(target, key) {
  if (activeEffect) {
    const effects = getSubscribersForProperty(target, key) // 객체(target)의 특정 키(key)가 구독하고 있는 이펙트 함수들을 가져오는 역할을 한다. 없으면 빈 Set 생성
    effects.add(activeEffect)
  }
}
```

## trigger

```javascript
function trigger(target, key) {
  const effects = getSubscribersForProperty(target, key) // 객체(target)의 특정 키(key)가 구독하고 있는 이펙트 함수들을 가져오는 역할을 한다. 없으면 빈 Set 생성
  effects.forEach((effect) => effect())
}
```

## activeEffect 변수는 무엇일까?
`activeEffect` 변수는 이펙트 함수를 등록할 때만 `track`을 실행하기 위해 존재한다. 일반적으로 `ref` 혹은 `reactive` 데이터에 접근하는 경우에는 `track`을 실행할 필요가 없기 때문이다.

```javascript

function ref(value) {
  const refObject = {
    get value() {
      track(refObject, 'value')
      return value
    },
    set value(newValue) {
      value = newValue
      trigger(refObject, 'value')
    }
  }
  return refObject
}
 
function track(target, key) {
  /*
    activeEffect 조건식을 통해 이펙트를 추가하는 작업을 매번 실행하지 않는다.
  */
  if (activeEffect) {
    const effects = getSubscribersForProperty(target, key) // 객체(target)의 특정 키(key)가 구독하고 있는 이펙트 함수들을 가져오는 역할을 한다. 없으면 빈 Set 생성
    effects.add(activeEffect)
  }
}
 
let activeEffect
/*
  공홈에서는 함수명이 whenDepsChange이지만 이해의 편의를 위해 watchEffect로 변경
  -> 둘이 거의 비슷한 역할을 수행한다고 함
*/
function watchEffect(update) {
  const effect = () => {
    activeEffect = effect
    update()
    activeEffect = null
  }
  effect()
}
```

## 전체코드 살펴보기
`ref`를 예시로 들어서 설명해보겠다. `reactive` 또한 동일하다! 이벤트 구독은 전역적으로 관리하는 `WeakMap<target, Map<key, Set<effect>>>` 데이터 구조에 저장된다.
```javascript

// Vue 코드
function ref(value) {
  const refObject = {
    get value() {
      track(refObject, 'value')
      return value
    },
    set value(newValue) {
      value = newValue
      trigger(refObject, 'value')
    }
  }
  return refObject
}
 
let activeEffect
 
function track(target, key) {
  if (activeEffect) {
    const effects = getSubscribersForProperty(target, key)
    effects.add(activeEffect)
  }
}
 
function watchEffect(update) {
  const effect = () => {
    activeEffect = effect
    update() // 아래의 sum 함수를 실행함! 그 과정에서 A0.value, A1.value를 읽기 때문에 track이 호출됨!
    activeEffect = null
  }
  effect()
}
 
function trigger(target, key) {
  const effects = getSubscribersForProperty(target, key)
  effects.forEach((effect) => effect())
}






// 본문 코드
import { ref, watchEffect } from 'vue'
 
const A0 = ref(0)
const A1 = ref(1)
const A2 = ref()
 
// 이펙트 함수 정의
const sum = () => {
  // A0과 A1을 추적함
  A2.value = A0.value + A1.value
}
 
// 이펙트 함수가 등록됨 (즉시 호출되면서 위의 A0, A1의 value를 읽음 -> track 호출됨)
watchEffect(sum)
 
// 이펙트(sum)가 실행됨 (trigger 호출됨)
A0.value = 2
```
위의 코드를 그림으로 나타내면

![Vue 반응형 데이터 구조](/assets/img/capture/vue3-reactivity-data-structure.png)

요약하면, <br />
`watchEffect`, `watch` 등에게 이펙트 함수를 전달한다고 가정하면

1. 이펙트 함수 내부에서 `ref`, `reactive` 로 선언된 변수 접근 시 `track`이 호출된다.
2. `track`을 호출할 경우 위 그림과 같이 특정 객체의 키로 접근할 수 있는 `Set` 자료구조에 이펙트 함수를 등록(구독)한다.
3. 이후 이펙트 함수를 구독하고 있는 특정 객체의 키의 값을 변경하면 `trigger`가 호출되며 구독하고 있는 모든 이펙트 함수를 실행한다.

# 유용한 반응형 이펙트 예
상태값이 변경될 때 DOM을 업데이트 시키는 방법으로 실제 Vue 컴포넌트가 상태와 뷰를 동기화 상태로 유지하는 방법과 매우 유사하다고 한다.

```javascript
import { ref, watchEffect } from 'vue'
 
const count = ref(0)
 
watchEffect(() => {
  document.body.innerHTML = `숫자 세기: ${count.value}`
})
 
// DOM 업데이트
count.value++
```