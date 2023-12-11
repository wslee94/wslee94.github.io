---
title: TS 다형성(Polymorphism) 적용해보기
date: 2023-12-11 12:25:00 +09:00
categories: [Deveploment, TS]
tags: [ts, oop, polymorphism] # TAG는 반드시 소문자로 이루어져야함!
# image: /assets/img/avatar.png
# pin: true
---

# 배경
Vue 2 지원이 올해 말(2023-12-31) 중단됨에 따라 현재 Nuxt 2 프레임워크를 사용하는 프로젝트들을 Nuxt 3으로 이관하는 작업이 필요하다. Vue 3는 폭넓은 TS를 지원하므로 이번 이관 작업에 TS 도입도 포함되어 있다. 이관이 필요한 프로젝트는 총 3개의 프로젝트가 있다.

- 포털웹 
- 크리에이터센터 
- ✅백오피스

 메이플W 백오피스는 UI/UX 개선 작업이 포함되면서 거의 새로운 프로젝트를 다시 만드는 수준으로 진행하고 있다. 백오피스에서 사용하는 라이브러리 중 Vue 3를 지원하지 않는 `@toast-ui/vue-editor` 라이브러리를 `@toast-ui/editor`로 교체하는 과정에서 TS도 도입했겠다, 이를 이용해 다형성을 적용해 본 경험을 이야기하려고 한다. 

# 다형성?
객체지향프로그래밍(OOP)은 4개의 중요한 개념이 있다.
- **Encapsulation (캡슐화)** <br />
연관 있는 데이터와 로직을 하나의 클래스에 잘 녹여내는 것을 의미한다. 또한 외부로 노출할 필요 없는 데이터와 메서드는 감추는 것이 중요하다.
- **Abstraction (추상화)** <br />
추상화란 비슷한 특징을 갖고 있는 여러 가지의 객체로부터 공통으로 사용할 수 있는 데이터와 메서드를 추출하는 과정을 의미한다. 즉, 추상화란 외부에서 어떤 형태로, 공통으로 어떻게 이 클래스를 이용하게 할 것인가? 를 고민하는 단계이다.
- **Inheritance (상속)** <br />
부모의 모든 데이터와 메서드를 자식이 물려받는 것을 의미한다. 상속은 꼭 필요할 때만 사용하기로 하자. 대부분 상속은 외부로부터 필요한 객체를 주입받는 컴포지션을 통해 대체할 수 있다. <br />
아래 상속의 단점을 보자. 
  - 상속은 수직적인 관계가 있어 부모 클래스가 수정되면 자식들에게 영향을 끼친다. (상속을 이용하는 장점이기도 하지만 단점이기도 하다.)
  - TS는 한 가지 이상 부모 클래스를 상속할 수 없다.
- **✅Polymorphism (다형성)** <br />
> __위키피디아__ <br /> 다형성이란 프로그램 언어 각 요소들(상수, 변수, 식, 객체, 메소드 등)이 다양한 자료형(type)에 속하는 것이 허가되는 성질을 가리킨다.

  다형성은 하나의 타입 만으로 여러 가지 타입의 객체를 참조할 수 있는 성질을 이용해 코드를 작성하는 것을 의미한다. 글로 이해하기 아려워 아래에서 예제를 통해 충분히 설명하려고 한다.

# Toast UI Editor의 커스텀 플러그인에 다형성 적용하기
![Toast UI Editor 커스텀 플러그인 종류](/assets/img/capture/ts-polymorphism-1.png) <br />
Toast UI Editor는 라이브러리에서 지원하는 기본 기능 외 개발자가 추가로 플러그인을 구현해 특정 기능을 추가할 수 있다. 위 그림을 보면 현재 백오피스에서 제공하는 커스텀 플러그인은 총 6개다. 

커스텀 플러그인의 툴바 아이템을 생성할 때 4개의 입력값이 필요하다.
1. 커스텀 플러그인의 이름
2. 커스텀 플러그인의 툴팁 메시지
3. 커스텀 플러그인의 아이콘 버튼
4. 커스텀 플러그인의 팝업 엘리먼트

## 인터페이스 정의
```typescript
export interface CustomPlugin {
  readonly name: string;              // 커스텀 플러그인의 이름
  readonly tooltip: string;           // 커스텀 플러그인의 툴팁 메시지
  getToolbarItem():HTMLButtonElement; // 커스텀 플러그인의 아이콘 버튼
  getPopup?(): HTMLDivElement;        // 커스텀 플러그인의 팝업 엘리먼트
}
```
6개의 플러그인은 동일한 기능을 제공하므로 동일한 인터페이스를 사용할 수 있다.

## 클래스 정의
대표적으로 유투브 커스텀 플러그인만 살펴보자. (나머지 커스텀 플러그인들 모두 비슷함!)
```typescript
/**
 * 에디터 내부 유투브 삽입 & 렌더링 플러그인
 */
class YoutubeTuiPlugin implements CustomPlugin {
  readonly name = 'youtube';
  readonly tooltip = 'youtube';
  private readonly toolbarEls: {
    button: HTMLButtonElement;
    icon: HTMLElement;
  };
  private editor: EditorCore;
  private popupEls: {
    contents: {
      label: HTMLElement;
      input: HTMLInputElement;
    };
    footer: {
      cancelButton: HTMLButtonElement;
      okButton: HTMLButtonElement;
    };
  };
 
  constructor(editor: EditorCore) {
    this.editor = editor;
    this.toolbarEls = {
      button: document.createElement('button'),
      icon: document.createElement('img')
    };
    this.popupEls = {
      contents: {
        label: document.createElement('label'),
        input: document.createElement('input')
      },
      footer: {
        cancelButton: document.createElement('button'),
        okButton: document.createElement('button')
      }
    };
    this.handleEvents();
  }
 
  // Markdown → HTML 변환 시 호출되는 메서드
  static render(): PluginInfo {
    const toHTMLRenderers: CustomHTMLRenderer = {
      youtube(node) {
        const wrapperId = `yt${Math.random().toString(36).substring(2, 12)}`;
        const youtubeId = node.literal;
 
        setTimeout(() => {
          const el = document.querySelector(`#${wrapperId}`);
          if (el) {
            el.innerHTML = `<iframe  width="480" height="360" frameborder="0" allowfullscreen src="https://www.youtube.com/embed/${youtubeId}"></iframe>`;
          }
        }, 0);
 
        return [
          {
            type: 'openTag',
            tagName: 'div',
            outerNewLine: true,
            attributes: { id: wrapperId }
          },
          { type: 'closeTag', tagName: 'div', outerNewLine: true }
        ];
      }
    };
    return { toHTMLRenderers };
  }
 
  // 툴바 아이콘 정의
  getToolbarItem(): HTMLButtonElement {
    const { button, icon } = this.toolbarEls;
    button.classList.add('tui_toolbar_item');
    icon.setAttribute('src', youtubeIcon);
    icon.setAttribute('width', '26');
    button.appendChild(icon);
    return button;
  }
 
  // 툴바 아이콘 클릭 시 팝업 엘리먼트 정의
  getPopup(): HTMLDivElement {
    const { label, input } = this.popupEls.contents;
    const { cancelButton, okButton } = this.popupEls.footer;
 
    // Youtube ID 입력란
    label.className = 'tui_label first';
    label.innerText = 'Youtube ID';
    input.className = 'tui_input';
    input.type = 'text';
    input.placeholder = 'After typing on the keyboard, press Enter.';
 
    // 버튼
    cancelButton.innerText = '취소';
    cancelButton.className = 'tui_button close';
    okButton.innerText = '확인';
    okButton.className = 'tui_button ok';
 
    const wrapperButton = document.createElement('div');
    wrapperButton.className = 'footer';
    wrapperButton.appendChild(okButton);
    wrapperButton.appendChild(cancelButton);
 
    // 팝업 컨테이너 생성
    const container = document.createElement('div');
    container.id = 'tuiPopup';
    container.appendChild(label);
    container.appendChild(input);
    container.appendChild(wrapperButton);
 
    return container;
  }
 
  // 내부적으로만 사용하는 메서드들은 외부로 노출시킬 필요가 없기에 private 사용 
  private handleEvents(): void { /* ... */ }
  private onClickToolbarItem(e: MouseEvent): void { /* ... */ }
  private onClickOK(): void { /* ... */ }
  private onClickCancel(): void{ /* ... */ }
  private clearPopupValues(): void { /* ... */ }
  private getYoutubeID(): string { /* ... */ }
  private isValidInputValues(): boolean { /* ... */ }
}
 
export default YoutubeTuiPlugin;
```
인터페이스에 정의된 데이터와 메서드를 제외하고 내부적으로 필요한 데이터와 메서드는 private 연산자를 사용해 외부로 노출되지 않도록 했다. 아래 다른 플러그인들 **모두 동일한 인터페이스(CustomPlugin)**를 구현하도록 했다. 
```typescript
class LinkTuiPlugin implements CustomPlugin { /* */ }
class ImageTuiPlugin implements CustomPlugin { /* */ }
class FileTuiPlugin implements CustomPlugin { /* */ }
class ChildItemsTuiPlugin implements CustomPlugin { /* */ }
class ChildCardsTuiPlugin implements CustomPlugin { /* */ }
```

## 다형성 적용
```typescript
// 6개의 커스텀 플러그인 클래스는 모두 같은 자료형(=CustomPlugin)에 담을 수 있다.
const customPlugins: Array<CustomPlugin> = [
  new LinkTuiPlugin(editor),
  new ImageTuiPlugin(editor),
  new FileTuiPlugin(editor),
  new YoutubeTuiPlugin(editor),
  new ChildItemsTuiPlugin(editor),
  new ChildCardsTuiPlugin(editor)
];

// 같은 자료형에 담았으므로 배열을 순회하면서 플러그인들을 사용할 수 있다.
customPlugins
  .forEach((plugin: CustomPlugin, index) => {
    editor.insertToolbarItem(
      { groupIndex: 5, itemIndex: index },
      {
        name: plugin.name,
        tooltip: plugin.tooltip,
        el: plugin.getToolbarItem(),
        popup: plugin.getPopup ? { body: plugin.getPopup() } : undefined
      }
    );
  });
```
위 코드에서 주목할 점은 <br />
- 6개의 커스텀 플러그인의 객체에 동일한 타입(= CustomPlugin)을 정의했다. 
- 배열을 이용해 순회하면서 커스텀 플러그인의 툴바 아이템을 생성하고 있다.

**즉, 하나의 타입(= CustomPlugin)만으로 여러 가지 타입의 객체를 참조할 수 있는 다형성을 이용했다!**  <br />
다형성을 통해 하나의 타입에 다양한 형태의 객체를 참조하고 객체들은 각각 다르게 동작하고 있다.

![다형성을 통한 동일한 인터페이스 제공](/assets/img/capture/ts-polymorphism-2.png) <br />
위 이미지를 보면 다형성을 통해 6개의 커스텀 플러그인은 외부에 동일한 인터페이스를 제공할 수 있다. 6개의 커스텀 플러그인 인스턴스는 동일한 타입을 가지고 있기 때문에 배열에도 담을 수 있고 또 반복문을 사용할 수 있다. 만약 다형성을 이용하지 않았다면, `const customPlugins: Array<object>` 혹은 `const customPlugins: Array<any>`로 코드를 작성할 수도 있을 텐데 이렇게 되면 TS를 사용하는 의미가 줄어들게 된다. 

# 다형성을 이용하면
1. 유지 보수가 용이하다.
  - 새로운 기능 추가 혹은 기존 기능 삭제 시 인터페이스를 수정하면 이를 구현한 클래스는 컴파일 단계에서 에러를 발견할 수 있다. (= 런타임에서 에러가 발생하는 상황을 피할 수 있다.)
  - 새로운 커스텀 플러그인이 추가된다면 CustomPlugin 인터페이스를 구현하면 된다. (= 기존의 커스텀 플러그인들을 사용하는 코드를 수정할 필요가 없다.)
2. 공통된 기능을 추상화한 인터페이스는 좋은 문서화의 효과를 낼 수 있다. (자동완성 및 코드 가독성 향상 등)
더 많은 장점이 있겠지만 체감되는 것은 위의 2가지 정도이다. 개발을 더 열심히 하다 보면 더 많은 것을 체감할 수 있을 것 같다. 

# 참고
다형성은 상속을 이용해서 구현할 수도 있다.
```typescript
class VideoPlayer {
  play() { /* ... */ }
  stop() { /* ... */}
}
 
class YotubePlayer extends VideoPlayer {}
class TwitchPlayer extends VideoPlayer {
  play() { /* 트위치만의 특별한 로직이 있는 경우 메서드 오버라이딩 */ }
  stop() { /* 트위치만의 특별한 로직이 있는 경우 메서드 오버라이딩 */}
}
 
// 다형성을 이용하면
const player1:VideoPlayer = new YoutubePlayer()
const player2:VideoPlayer = new TwitchPlayer()
player1.play(); 
player2.play(); // 같은 타입이지만 다른 기능을 수행한다.
```
TS를 도입하면서 잊혀져가던 OOP를 상기할 수 있어서 좋았다.