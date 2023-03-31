---
title: 다국어 처리에 대한 고찰
date: 2023-04-01 10:00:00 +09:00
categories: [Deveploment, MutlLanguage]
tags: [web, mult-language, javascript, webpack, nuxt, i18n] # TAG는 반드시 소문자로 이루어져야함!
# image: /assets/img/avatar.png
# pin: true
---

# 배경

게임 이벤트 페이지 제작에 국문, 영문 다국어 처리를 해야 했다. 나는 단일페이지 개발이라 `HTML` + `CSS` + `VanillaJS` + `Webpack` 환경을 구성하고 페이지를 개발했다.

# 다국어 처리 프로세스

- 페이지 최초 접속 시 URL 뒤 쿼리스트링을 붙여 리다이렉트
- 이후 기본 URL로 접속 시 로컬 스토리지에 저장된 언어코드에 따라 URL 뒤 쿼리스트링을 붙여 리다이렉트
- Example <br /> `https://www.enter.nexon.com/mod/wings2022?lang=ko` <br /> `https://www.enter.nexon.com/mod/wings2022?lang=en`

# 내가 선택한 다국어 처리 방법

`<body>` 안에 표현되는 텍스트 데이터는 HTML의 `Custom Attribute`와 `Javascript`를 이용해 설정된 언어코드에 따라 동적으로 데이터를 넣어주는 형태로 구현했다.
```html
<span data-detact="home_text"></span>
```

```javascript
/*
  languageData 형식
  {
    ko: { home_text: '홈' },
    en: { home_text: 'Home' }
  }
*/
import languageData from './language';
 
/* HTML 태그 중 data-detact custom attribute */
const textData = document.querySelectorAll('[data-detact]');
 
/* 언어 코드에 따라 동적으로 다국어 데이터를 넣어준다. */
function initTextByLanguage(languageCode) {
  textData.forEach((node) => {
    node.innerText = languageData[languageCode][node.dataset.detact];
  });
}
```

# 중간 성찰

다국어 데이터 변환 시 단순히 텍스트만 변경하면 될 거로 생각했다. **하지만 한국어와 영어는 어순도 다르고 이에 따라 특정 스타일을 적용한 워딩의 위치가 서로 다르다는 것을 깨달았다**. 

![국문 번역 샘플](/assets/img/capture/multilang-1.png) <br /> 
![영문 번역 샘플](/assets/img/capture/multilang-2.png)

```json
{
  "ko": {
    "text_1_1": "PROJECT MOD",
    "text_1_2": "",
    "text_2": "개발환경",
    "text_3": "을 소개합니다."
  },
  "en": {
    "text_1_1": "Introducing",
    "text_1_2": "A Powerful",
    "text_2": "World Maker",
    "text_3": ""
  }
}
```

텍스트를 기준으로 다국어 처리를 할 경우 개행 지점과, 어순이 달라 위의 JSON 데이터처럼 공란이 발생하고 넘버링을 해줘야 하는 불편함이 있다. **이를 해결하려면 `HTML` 태그를 다국어 데이터에 포함하여 저장하는 방법이 더 좋을 것 같다고 생각했다.** 

아래와 같은 방식으로 코드를 개선해보자!
```json
{
  "ko": {
    "text": "PROJECT MOD <br /> <span style='color:orange;'>개발환경</span>을 소개합니다.",
  },
  "en": {
    "text": "Introducing  <br /> A Powerful <span style='color:orange;'>World Maker</span>",
  }
}
```
```javascript
function initTextByLanguage(languageCode) {
  textData.forEach((node) => {
    /* innerHTML 사용 */
    node.innerHTML = languageData[languageCode][node.dataset.detact];
  });
}
```

# 문제 발생

기획 측에서 추가 요구 사항으로 오픈그래프가 추가되면서, 오픈그래프에 표시되는 이미지, 텍스트 등도 URL에 포함된 언어 코드에 따라 다르게 출력해달라는 요청이 들어왔다. 

> 오픈 그래프 프로토콜은 어떠한 인터넷 웹사이트의 HTML 문서에서 head -> meta 태그 중 "og:XXX"가 있는 태그들을 찾아내어 보여주는 프로토콜이다. SNS를 통해 링크를 공유할 때 미리보기에 사용된다.
{: .prompt-tip }

**오픈 그래프 예시 화면** <br />
![오픈 그래프 예시 화면](/assets/img/capture/multilang-3.png)

처음에 단순히 Javascript를 이용해 쿼리스트링에 따라 meta 태그를 동적으로 변경하면 될 줄 알았다. 그러나 이미 SNS Scapper가 meta 태그를 풀링한 이후이므로 소 잃고 외양간 고치는 격이라는 것을 깨달았다.

# 해결방법

## CSR
영문, 국문 HTML 템플릿 2개를 만들고 `<head>` 안 meta 태그들을 각각 언어에 맞게 설정한다.
그 후에, 웹팩에 의해 번들링된 JS를 HTML 템플릿 각각 넣어주면 `<body>` 내부 소스 코드 중복 없이 국문, 영문 페이지를 제공할 수 있다고 생각했다.

```javascript
/* webpack.config.js */
const HtmlWebpackPlugin = require('html-webpack-plugin');
 
module.export = {
  // ...config
  entry: './src/index.js',
  plugins: [
    // ...plugins
    new HtmlWebpackPlugin({
        template: './src/index_ko.html',
        filename: 'ko/index.html',
    }),
    new HtmlWebpackPlugin({
        template: './src/index_en.html',
        filename: 'en/index.html',
    }),
  ]
}
```

## SSG, SSR
`i18n(@nuxtjs/i18n)` 라이브러리를 이용해 다국어 처리 시 헤더 정보를 언어별로 따로 설정할 수 있다. default 언어 외 다른 언어가 설정되면 URL 최상단에 언어코드 경로가 추가된다. Nuxt로 다국어 처리를 해보자!

```javascript
/* nuxt.config.js */
export default {
  // ... config
  modules: ["@nuxtjs/i18n"],
  i18n: {
    locales: [
      { name: "한국어", code: "ko", iso: "ko-KR" },
      { name: "English", code: "en", iso: "en-US" },
    ],
    defaultLocale: "ko",
    vueI18n: {
      fallbackLocale: "ko",
      messages: {
        ko: {
          welcome: "반가워요.",
        },
        en: {
          welcome: "Welcome",
        },
      },
    },
  },
}
```
```vue
/* layouts/default.vue */
<template>
  <Nuxt />
</template>
<script>
export default {
  head() {
    /* 언어 코드 별 og meta tag 설정 */
    if(this.$i18n.locale === 'ko') {
      return {
        meta: [
          {
            hid: 'og:title',
            property: 'og:title',
            content: '타이틀'
          },
          {
            hid: 'og:description',
            property: 'og:description',
            content: '설명'
          },
          {
            hid: 'og:image',
            property: 'og:image',
            content: ''
          },
        ]
      }
    }
    return {
      meta: [
        {
          hid: 'og:title',
          property: 'og:title',
          content: 'Title'
        },
        {
          hid: 'og:description',
          property: 'og:description',
          content: 'Description'
        },
        {
          hid: 'og:image',
          property: 'og:image',
          content: ''
        },
      ]
    }
  },
}
</script>
<style>
</style>
```

### SSR 결과
![SSR 국문 메타 태그 이미지](/assets/img/capture/multilang-4.png) <br />
![SSR 영문 메타 태그 이미지](/assets/img/capture/multilang-5.png) <br />

### SSG 결과

📦dist <br />
 ┣ 📂_nuxt <br />
 ┣ 📂en <br />
 ┃ ┗ 📜index.html <br />
 ┣ 📂ko <br />
 ┃ ┗ 📜index.html <br />
 ┗ 📜index.html <br /> 

![SSG 국문 메타 태그 이미지](/assets/img/capture/multilang-6.png) <br />
![SSG 영문 메타 태그 이미지](/assets/img/capture/multilang-7.png) <br />

# 교훈

- 다국어 데이터는 텍스트만 고려하지 말고 스타일(태그)까지 고려해서 작성하자!
- 오픈 그래프 다국어 지원도 고려하고 상황에 맞게 CSR, SSR, SSG를 선택하자!