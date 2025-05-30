---
title: Electron 환경변수 사용하기
date: 2025-05-30 16:30:00 +09:00
categories: [Deveploment, Javascript]
tags: [javascript, electron] # TAG는 반드시 소문자로 이루어져야함!
# image: /assets/img/avatar.png
# pin: true
---

# 개요
이번에 팀에서 넥슨 회원의 게임 플레이 이력을 영수증 형태로 뽑아주는 "나의 넥슨 기록"이라는 프로젝트를 인수/인계받았다. 해당 서비스는 데스크탑 애플리케이션 형태로 제공되어야 해서 Electron 프레임워크 기반으로 구현되어 있었다.

# NODE_ENV?
기존 코드를 살펴보는 중에 각 환경(DEV, QA, LIVE 등)별 빌드 시 스크립트에서 NODE_ENV 값을 주입하고 애플리케이션 내부에서 해당 값을 기준으로 환경별 변수를 불러오는 코드가 있었다. 여기서, NODE_ENV는 Node.js 애플리케이션 환경을 구분하기 위해 사용하는 환경 변수로 주로 개발 환경과 배포 환경을 구분하는 데 사용되며, 이를 통해 각 환경에 맞는 설정 및 동작을 수행할 수 있다. 보통 로컬 개발 환경에서는 NODE_ENV 값은 development 로 자동으로 설정되고, 빌드 시 production 으로 설정된다.

NODE_ENV는 React와 같은 라이브러리 혹은 Webpack, Vite와 같은 번들러 도구들이 참조하고 있는 값으로 NODE_ENV의 값에 따라 내부 동작을 다르게 수행한다. 예를 들어 NODE_ENV가 production 일 경우에 번들링 시 압축 및 난독화를 수행하거나 퍼포먼스 향상을 위해 최적화를 수행한다. 하지만, 개발자가 애플리케이션 환경 분기하기 위해 NODE_ENV 값을 변경할 경우 앞서 얘기한 라이브러리, 번들러 도구들이 개발자 의도와 다르게 동작할 수 있다. 그렇기 때문에 환경별 분기용으로는 적절하지 않다고 생각한다. 실제로 QA 환경은 LIVE와 동일한 환경에서 테스트를 해야하기 때문에 LIVE와 동일하게 빌드되어야 하는데 NODE_ENV를 애플리케이션 환경 분기용으로 사용할 경우 동일한 환경을 구성하기 어렵다. 또 NODE_ENV에 예약된 값은 development, test, production인데 애플리케이션 환경이 세 가지보다 더 많은 경우도 있다.

# 코드
변경 전 코드와 변경 후 코드를 살펴보자!

## 변경 전
```json
"scripts": {
  "build:develop": "cross-env NODE_ENV=development nextron build",
  "build:test": "cross-env NODE_ENV=test nextron build",
  "build:live": "cross-env NODE_ENV=production nextron build"
}
```
```javascript
// main/background.js
const env = {
  local: {
    TARGET_URL: "...",
    API_URL: "...",
    // ...
  },
  dev: { /* ... */ },
  test: { /* ... */ },
  production: { /* ... */ },
};
 
const node_env = process.env.NODE_ENV;
const targetURL = env[node_env].TARGET_URL;
```

## 변경 후
```json
"scripts": {
  "build:develop": "cross-env APP_ENV=dev nextron build",
  "build:test": "cross-env APP_ENV=test nextron build",
  "build:live": "cross-env APP_ENV=production nextron build"
}
```
여기서 빌드된 애플리케이션에서 APP_ENV 접근 시 undefined 값이 나오는 이슈가 있었다. 원인은 컴파일 단계에서만 해당 값에 접근할 수 있고, 빌드된 애플리케이션 환경에서는 유실되기 때문이다. 따라서 별도 처리가 필요하다. NODE_ENV 값을 직접 변경할 때 해당 이슈가 발생하지 않은 이유가 궁금해서 찾아봤는데 빌드 시 Webpack이 번들링하면서 process.env.NODE_ENV를 CLI에서 주입한 문자열 값으로 치환한다고 한다. 

```javascript
// electron-builder.js
module.exports = {
  // ...
  extraMetadata: {
    APP_ENV: process.env.APP_ENV, // 빌드한 결과물에서 사용할 수 있도록 APP_ENV 주입
  },
};

// main/background.js
import { app } from "electron";

const env = {
  local: {
    TARGET_URL: "...",
    API_URL: "...",
    // ...
  },
  dev: { /* ... */ },
  test: { /* ... */ },
  production: { /* ... */ },
};
 
const getEnv = () => {
  const isLocal = process.env.NODE_ENV === "development";
  if (isLocal) {
    // 로컬 개발 환경
    return env[process.env.APP_ENV];
  } else {
    // 빌드한 애플리케이션 환경
    const appPath = app.getAppPath();
    const pkg = JSON.parse(fs.readFileSync(path.join(appPath, "package.json")));
    return env[pkg.APP_ENV];
  }
};
 
const targetURL = getEnv().TARGET_URL;
```
위 코드를 설명하면
1. 빌드 시 electron-builder.js가 실행되는데 이때 extraMetadata 옵션을 사용해 package.json 에 property를 주입한다.
2. 애플리케이션 실행 시 package.json 을 읽어와 주입한 변수(APP_ENV)를 가져온다.

# 결론
애플리케이션 환경을 분리하기 위해 NODE_ENV 값을 수정하지 말고 APP_ENV와 같은 새로운 변수를 지정하자.