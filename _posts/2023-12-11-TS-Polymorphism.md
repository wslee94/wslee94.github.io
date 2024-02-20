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
- ✅ 백오피스

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
- **Polymorphism (다형성)** <br />
> __위키피디아__ <br /> 다형성이란 프로그램 언어 각 요소들(상수, 변수, 식, 객체, 메소드 등)이 다양한 자료형(type)에 속하는 것이 허가되는 성질을 가리킨다.

  다형성은 하나의 타입 만으로 여러 가지 타입의 객체를 참조할 수 있는 성질을 이용해 코드를 작성하는 것을 의미한다. 글로 이해하기 아려워 아래에서 예제를 통해 충분히 설명하려고 한다.

# Toast UI Editor의 커스텀 플러그인에 다형성 적용하기
![Toast UI Editor 커스텀 플러그인 종류](/assets/img/capture/ts-polymorphism-1.png) <br />
Toast UI Editor는 라이브러리에서 지원하는 기본 기능 외 개발자가 추가로 플러그인을 구현해 특정 기능을 추가할 수 있다. 위 그림을 보면 현재 백오피스에서 제공하는 커스텀 플러그인은 총 6개다. 

커스텀 플러그인의 툴바 아이템을 생성할 때 3개의 입력값이 필요하다.
1. 커스텀 플러그인의 이름
2. 커스텀 플러그인의 툴팁 메시지
3. 커스텀 플러그인의 아이콘 버튼

## 인터페이스 정의
```typescript
export interface CustomEditorPlugin {
  readonly name: CustomEditorPluginNames;
  readonly tooltip: string;
  readonly icon: HTMLElement;
}
```
6개의 플러그인은 동일한 기능을 제공하므로 동일한 인터페이스를 사용할 수 있다.

## 클래스 정의
대표적으로 Link 커스텀 플러그인만 살펴보자. (나머지 커스텀 플러그인들 모두 비슷함!)
```typescript
import type { EditorCore, MdNode } from '@toast-ui/editor';
import type { Context } from '@toast-ui/editor/types/toastmark';
import type { CustomEditorPlugin } from '@/types/editor';
import 'assets/styles/plugin/tui-custom-plugin.scss';
import { showAlert } from '@/assets/ts/common/alert';
 
/**
 * 에디터 내부 링크 삽입 & 렌더링 플러그인
 * 기본 기능을 사용하지 않고 커스텀으로 만든 이유는 링크 삽입 시 target을 지정하기 위해 사용
 */
class LinkTuiPlugin implements CustomEditorPlugin {
  private readonly editor: EditorCore;
  private readonly openDialog?: () => void;
  readonly name = 'link';
  readonly tooltip = '링크';
  readonly icon;
 
  constructor(editor: EditorCore, openDialog?: () => void) {
    this.editor = editor;
    this.openDialog = openDialog; // 아이콘 버튼 클릭 시 외부에서 주입받은 dialog(vuetify) 활성화 함수를 실행한다.
    this.icon = document.createElement('i');
    this.icon.className = 'tui_toolbar_item mdi mdi-link-variant';
    this.icon.style.cssText = 'font-size: 26px;';
    this.icon.addEventListener('click', this.onClickIcon.bind(this));
  }
 
  // Markdown → HTML 변환 시 호출되는 메서드
  static render(_node: MdNode, context: Context) {
    const { origin, entering } = context;
    const result: any = origin === undefined ? {} : { ...origin() };
    const regex = /\{.*\}/g;
 
    try {
      if (entering && 'attributes' in result) {
        const href = decodeURIComponent(result.attributes.href);
        if (regex.test(href)) {
          result.attributes.href = href.replace(regex, '');
          const match = href.match(regex)?.[0];
          if (match) {
            const json: { target: string } = JSON.parse(match);
            result.attributes.target = json.target;
          }
        }
      }
    } catch {}
    return result;
  }
 
  // 툴바 아이콘 클릭 시 실행하는 메서드
  private onClickIcon(e: MouseEvent): void {
    if (this.editor.isWysiwygMode()) {
      e.stopPropagation();
      showAlert({
        title: '지원하지 않는 모드',
        text: 'Markdown 모드에서 실행해주세요.'
      });
    } else {
      this.openDialog?.();
    }
  }
}
 
export default LinkTuiPlugin;
```
인터페이스에 정의된 데이터와 메서드를 제외하고 내부적으로 필요한 데이터와 메서드는 private 연산자를 사용해 외부로 노출되지 않도록 했다. 아래 다른 플러그인들 **모두 동일한 인터페이스(CustomEditorPlugin)**를 구현하도록 했다. 
```typescript
class YoutubeTuiPlugin implements CustomEditorPlugin { /* */ }
class ImageTuiPlugin implements CustomEditorPlugin { /* */ }
class FileTuiPlugin implements CustomEditorPlugin { /* */ }
class ChildItemsTuiPlugin implements CustomEditorPlugin { /* */ }
class ChildCardsTuiPlugin implements CustomEditorPlugin { /* */ }
```

## 다형성 적용
```typescript
// 6개의 커스텀 플러그인 클래스는 모두 같은 타입에 담을 수 있다.
const { customToolbarItems } = options;
const customPlugins: Array<CustomEditorPlugin> = [
  new LinkTuiPlugin(editor, options.onClickLinkToolbarItem),
  new ImageTuiPlugin(editor, options.onClickImageToolbarItem),
  new FileTuiPlugin(editor, options.onClickFileToolbarItem),
  new YoutubeTuiPlugin(editor, options.onClickYoutubeToolbarItem),
  new ChildItemsTuiPlugin(editor),
  new ChildCardsTuiPlugin(editor)
];

// 같은 타입에 담았으므로 배열을 순회하면서 플러그인들을 사용할 수 있다.
customPlugins
  .filter((plugin: CustomEditorPlugin) => {
    return customToolbarItems.includes(plugin.name);
  })
  .forEach((plugin: CustomEditorPlugin, index) => {
    const el = plugin.icon;
    editor.insertToolbarItem(
      { groupIndex: 5, itemIndex: index },
      {
        name: plugin.name,
        tooltip: plugin.tooltip,
        el,
        onUpdated({ disabled }) {
          disabled
            ? el.classList.add('disabled')
            : el.classList.remove('disabled');
        }
      }
    );
  });
```
![다형성을 통한 동일한 인터페이스 제공](/assets/img/capture/ts-polymorphism-2.png) <br />

위 코드에서 주목할 점은 <br />
- 6개의 인스턴스에 동일한 타입을 할당했다.
- 하나의 타입에 다양한 형태의 인스턴스를 참조 하면서 각각 다르게 동작하게 할 수 있다. (다형성)
- 하나의 타입으로 관리되기 때문에 같은 타입의 배열에 담아 반복문을 사용할 수 있다. 만약 다형성을 이용하지 않았다면, `const customPlugins: Array<any>`로  코드를 작성할 수도 있었을텐데 그러면 타입스크립트를 사용하는 의미(컴파일 단계 에러 체크, 자동 완성)가 줄어들게 된다.


# 다형성을 이용하면
### 확장성이 높다
확장성이 높다는 말은 기존 코드를 변경하지 않으면서도 새로운 기능을 추가하거나 기존 기능을 변경할 수 있다는 의미이다. 기존에 사용하던 인스턴스를 동일한 타입을 가진 다른 인스턴스로 교체할 경우 기존 코드의 수정 없이 기능을 변경할 수 있다.

위의 예시에서는 확장성 관련해 마땅히 설명할 수 없어 이해를 돕기 위한 자바 소스코드를 가져왔다.
```java
public interface ProductSortStrategy {
    List<Product> sort(List<Product> products);
}
 
public class PriceSortStrategy implements ProductSortStrategy {
    @Override
    public List<Product> sort(List<Product> products) {
        // 가격으로 정렬 후 반환
    }
}
 
public class NameSortStrategy implements ProductSortStrategy {
    @Override
    public List<Product> sort(List<Product> products) {
        // 이름으로 정렬 후 반환
    }
}
```
위의 코드는 제품을 정렬하는 클래스 2개가 동일한 인터페이스(ProductSortStrategy)를 구현했다.
```java
public class ProductSorter {
    private ProductSortStrategy strategy;
     
    // 생성자 인자로 ProductSortStrategy 인터페이스를 구현한 인스턴스를 받고 있다.
    public ProductSorter(ProductSortStrategy strategy) {
        this.strategy = strategy;
    }
     
    // sort 메서드는 외부에서 주입받은 strategy 변수에 따라 동작을 달리한다.
    public List<Product> sort(List<Product> products) {
        return strategy.sort(products);
    }
}
```
ProductSorter는 정렬 방식의 변경이 필요하면 ProductSortStrategy 구현한 다른 인스턴스로 교체할 수 있다. 즉, ProductionSorter 코드의 변경 없이 새로운 기능으로 교체가 가능하다. 만약 새로운 정렬 방식이 필요하면 ProductSortStrategy를 구현한 다른 인스턴스를 만들면된다.

### 에러를 발견하기 쉽다.
인터페이스가 수정되거나 인터페이스 내용을 따르지 않는 인스턴스가 있는 경우 컴파일 단계에서 에러를 발견할 수 있다.

### 좋은 문서화의 효과를 가질 수 있다.
공통된 기능을 추상화한 인터페이스는 자동완성 및 코드 가독성 향상 등 의 효과를 낼 수 있다.

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