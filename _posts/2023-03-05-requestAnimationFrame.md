---
title: 복잡한 인터렉션 적용하기 (requestAnimationFrame)
date: 2023-03-05 14:00:00 +09:00
categories: [Deveploment, motion]
tags: [web, css, javascript, animation, motion] # TAG는 반드시 소문자로 이루어져야함!
# image: /assets/img/avatar.png
# pin: true
---

# 나에게 시련이...

![web motion](/assets/img/gif/msw-wings2022-motion.gif)

입사하고 나서 얼마 지나지 않아 MapleStory Worlds의 [크리에이터 모집 페이지](https://enter.nexon.com/mod/wings2022?lang=ko){:target="_blank"} 제작 업무를 받았다. 디자인 시안에서 기획서 검토할 때 없었던 인터랙션들이 많아 당황했다. 전 직장에서 CSS의 기본적인 것만 활용할 줄 알았던 나에게 여러 가지 인터랙션이 존재하는 웹페이지 제작은 커다란 두려움으로 다가왔다. 

이미 크리에이터 모집 일정은 정해져 있었고 개발 기간 연장은 불가능한 상황이어서 디자이너가 애써 작업한 인터랙션들은 제외하고 개발해야하던 찰나에 선임 두 분이 한 섹션씩 분담해 일정을 맞출 수 있었다.

위에 첨부한 인터랙션은 선임이 개발했었는데 그 당시 나에게 매우 난해했던 모션이었다. **에셋들이 왼쪽으로 끊임없이 이동하면서 특정 지점에서 점프해야 하는데 CSS로는 머릿속에서 어떻게 구현해야 할지 떠오르지 않았다.** 

그 당시 선임이 ***requestAnimationFrame***을 이용해 문제를 해결한 것을 알았다. 그렇게 시간이 흘러... 얼마 전 넥러닝이라는 회사에서 들을 수 있는 강의 플랫폼에서 requestAnimationFrame 주제를 접하게 되었다. 지난날 위에서 언급한 막막했던 인터랙션이 떠올라 직접 한번 개발해보기로 마음먹었다.

# requestAnimationFrame?

> The window.requestAnimationFrame() method tells the browser that you wish to perform an animation and requests that the browser calls a specified function to update an animation before the next repaint. The method takes a callback as an argument to be invoked before the repaint
{: .prompt-tip }

MDN에서 설명하기를 브라우저에 수행하기를 원하는 애니메이션을 알리고 다음 리페인트가 진행되기 전에 해당 애니메이션을 업데이트하는 함수를 호출하는 것이라고 한다. 음... 그림으로 알아보자!

![requestAnimationFrame](/assets/img/capture/requestAnimationFrame-1.jpg)

위 이미지는 Javascript 런타임에서 Render 부분만 추려냈다. Event Loop는 60fps(16.7ms)  주기로 Render Process를 실행한다. 그 이유는 사람 눈은 초당 60번 정도의 Frame 변화가 자연스럽게 느껴지기 때문이다. 60fps 이상은 실제로는 애니메이션이 더 부드럽겠지만, 사람 눈으로 그 차이를 인지하지 못한다고 한다.

requestAnimationFrame의 콜백 함수는 paint 전에 호출되며 스타일을 변경했다면 화면에 반영된다. 지속적인 애니메이션 효과를 만들어내려면 콜백 함수 내부에서 requestAnimationFrame을 호출하면 된다. 지금은 이해가 안될 수 있지만, 이따 코드를 보면서 다시 한번 설명하겠다.

requestAnimation의 특징을 정리해보면,
- Render Process 중 Paint 이전에 실행된다.
- 모니터 주사율에 따라 반복 주기가 결정된다. (보통 60fps) 즉, 개발자가 반복 주기를 컨트롤할 수 없다.
- 브라우저에서 다른 탭 켜기 등의 화면 이탈 시 실행을 중지한다. (= 리소스 낭비 X)
 
# requestAnimationFrame을 이용한 인터랙션 구현

![requestAnimationFrame motion](/assets/img/gif/requestAnimationFrame-motion.gif)

부분적인 코드를 첨부하고 설명하겠다. 전체적인 코드는 [링크](https://github.com/wslee94/interaction/tree/master/moveAndJump){:target="_blank"}를 확인하면 된다!

## 1. 초기화
```javascript
class MoveAndJumb {
  #parentElement; // 동적으로 생성된 블록을 붙힐 부모 엘리먼트
  #blockNum; // 동적으로 생성할 블록 개수
  #blockArr = []; // 블록을 관리할 배열
  #blockWidth = 140; // 블록 너비
  #jumpStartingXPoint = 0; // 점프를 시작할 X 좌표

  constructor(parentElement, blockNum) {
    this.#parentElement = parentElement;
    this.#blockNum = blockNum;

    this.init();
  }

  init() {
    for (let i = 0; i < this.#blockNum; i++) {
      const newBlock = document.createElement("div");
      newBlock.classList.add("block");
      newBlock.style.transform = `matrix(1, 0, 0, 1, ${(this.#blockWidth + 20) * i}, 0)`;
      this.#blockArr.push(newBlock);
      this.#parentElement.appendChild(newBlock);
    }

    this.#jumpStartingXPoint = parseInt(window.innerWidth * (2 / 3));
    window.addEventListener("resize", (e) => {
      this.#jumpStartingXPoint = parseInt(e.target.innerWidth * (2 / 3));
    });
  }
}
```
`init()` 함수 내부에서 블록들을 생성하고 각 블록의 초기 위치를 잡아주었다. `jumpStartingXPoint`는 브라우저 길이의 ⅔ 지점으로 지정하고, 브라우저가 resize 될 때마다 값을 업데이트한다.

## 2. registerAnimation 작성 (= 애니메이션 등록)
```javascript
class MoveAndJumb {
  registerAnimation(element) {
    let moveReq;
    let jumpReq;

    const move = () => {
      const xValue = getTranslateXValue(element); // 블록의 기존 x좌표 값을 가져온다.
      const nextXValue = xValue - 1; // 블록을 기존 x좌표에서 -1 만큼 왼쪽으로 이동
      element.style.transform = `matrix(1, 0, 0, 1, ${nextXValue}, 0)`; // 블록의 스타일을 업데이트한다.

      // 블록이 아직 왼쪽으로 이동할 수 있으면
      if (nextXValue >= -this.#blockWidth) {
        moveReq = requestAnimationFrame(move); // 다시 requestAnimationFrame을 호출한다.
      } 
      // 블록이 왼쪽으로 이동하다가 사라지게 되면
      else {
        cancelAnimationFrame(moveReq); // requestAnimationFrame을 취소한다.
        this.removeBlock(element); // 해당 블록을 제거한다.
        this.addBlock(); // 배열의 맨 끝에 새로운 블록을 추가한다.
        return;
      }
    };

    let isTouchPeek = false;
    const jump = () => {
      const yValue = parseFloat(getComputedStyle(element).getPropertyValue("bottom"));  //  블록의 기존 y좌표 값을 가져온다.
      let nextYValue;

      // 블록이 아직 최고지점까지 점프하지 않았으면
      if (!isTouchPeek && yValue < 100) {
        nextYValue = yValue + 2; // 블록을 2만큼 점프시킨다.
      } 
      // 블록이 최고지점까지 점프하고 바닥까지 내려오지 않았으면
      else if (yValue > 0) {
        isTouchPeek = true; // 최고지점 도달 여부 flag 업데이트
        nextYValue = yValue - 2; // 블록을 2만큼 하락한다.
      }

      const xValue = getTranslateXValue(element); // 블록의 기존 x좌표 값을 가져온다.
      // 블록이 점프구간에 들어오면
      if (xValue > this.#jumpStartingXPoint - 200 && xValue <= this.#jumpStartingXPoint) {
        element.style.bottom = `${nextYValue}px`; // 블록의 스타일을 업데이트한다.
      } 
      // 블록이 점프구간을 지나치면
      else if (xValue <= this.#jumpStartingXPoint - 200) {
        element.style.bottom = `${0}px`; // 블록의 bottom을 0으로 초기화한다.
        cancelAnimationFrame(jumpReq); // requestAnimationFrame을 취소한다.
        return;
      }

      jumpReq = requestAnimationFrame(jump); // 다시 requestAnimationFrame을 호출한다.
    };

    moveReq = requestAnimationFrame(move); // 초기 실행을 위해 직접 호출
    jumpReq = requestAnimationFrame(jump); // 초기 실행을 위해 직접 호출
  }
}
```
크게 2가지의 애니메이션이 있다. `move()`, `jump()` 각각 살펴보자.

`move()`
- 블록을 왼쪽으로 이동하는 애니메이션 함수
- 블록이 계속 왼쪽으로 이동할 수 있는 것은 함수 `move()` 내부에서 다시 `requestAnimationFrame`을 호출하기 때문이다. (16.7ms 주기로 반복 호출됨)
- 블록이 계속 왼쪽으로 이동하다가 사라질 경우 `requestAnimationFrame`을 취소한다. 그리고 해당 엘리먼트를 배열에서 삭제하고 배열 끝에 새로운 블록을 추가한다. 위의 코드에는 없지만 `addBlock()` 내부에서 `registerAnimation`을 호출하기 때문에 새로 추가된 블록도 애니메이션이 적용된다.

`jump()`
- 블록이 점프 구간으로 설정한 지점에 도달하면 블록을 위로 이동하고 아래로 이동하는 애니메이션 함수
- 마찬가지로 블록이 점프할 수 있는 것은 `jump()` 내부에서 다시 `requestAnimationFrame`을 호출하기 때문이다. (16.7ms 주기로 반복 호출됨)
- 블록이 점프 구간을 지나치면 `requestAnimationFrame`을 취소한다.

## 애니메이션 실행

```javascript
class MoveAndJump {
  run() {
    for (let i = 0; i < this.#blockArr.length; i++) {
      this.registerAnimation(this.#blockArr[i]);
    }
  }
}
```

# requestAnimationFrame vs CSS Animation

> Browsers are able to optimize rendering flows. In summary, we should always try to create our animations using CSS transitions/animations where possible. If your animations are really complex, you may have to rely on JavaScript-based animations instead.
{: .prompt-tip }

MDN에서는 대부분의 경우에 같은 성능을 보여주기 때문에 CSS를 이용한 모션 처리하라고 권고한다. CSS로 작성 시 코드양이 줄고 조금 더 직관적인 장점이 있다. 실제로 실무에서 requestAnimation을 사용한 경험이 거의 없다. 대부분 CSS의 `transition`, `animation` 문법을 활용해 모션 처리를 해왔다. 하지만 위에서 구현한 것과 같이 2개 이상의 복합적인 애니메이션을 적용해야 하고 특정 지점 (특히 브라우저 크기에 따라 가변일 경우) 애니메이션 처리를 해야 하는 등 더 복잡한 애니메이션을 구현할 때 Javascript의 `requestAnimationFrame`을 이용해 애니메이션을 구현하는 게 더 좋을 듯하다.