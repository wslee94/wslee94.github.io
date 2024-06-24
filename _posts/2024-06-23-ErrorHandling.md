---
title: Typescript 에러 핸들링 (Nuxt 3)
date: 2024-06-23 21:20:00 +09:00
categories: [Deveploment, Typescript]
tags: [typescript, error, nuxt] # TAG는 반드시 소문자로 이루어져야함!
# image: /assets/img/avatar.png
# pin: true
---

HelloMaple 백오피스 작업할 때 HTTP Response 에러를 일괄적으로 처리한 방법에 대해서 정리했다.

# 구현하기
현재 다른 프로젝트들을 살펴보면 `try`, `catch` 구문을 이용해 에러를 핸들링하고 있다. `HelloMaple` 백오피스도 다른 프로젝트와 동일한 방법을 이용하려고한다.

## 커스텀 에러 클래스 정의 

### ErrorBase.ts
커스텀 에러 클래스가 상속할 부모 클래스를 정의한다.

```typescript
export default class ErrorBase<T extends string> extends Error {
  name: T;
  message: string;
  cause: any;

  constructor({
    name,
    message,
    cause
  }: {
    name: T;
    message: string;
    cause?: any;
  }) {
    super();
    this.name = name;
    this.message = message;
    this.cause = cause;
  }
}
```

### ResponseError.ts
HTTP 통신 응답 커스텀 에러 클래스 정의 (ErrorBase 클래스 상속), 다른 커스텀 에러가 필요한 경우 마찬가지로 `ErrorBase` 클래스를 상속한다.

```typescript
import ErrorBase from 'assets/ts/error/ErrorBase';

type ErrorName = 'RESPONSE_ERROR_API' | 'RESPONSE_ERROR_STATUS';

// HTTP 응답 에러 클래스 정의
export default class ResponseError extends ErrorBase<ErrorName> {
  statusCode: number; // HTTP 상태 코드
  apiCode?: number; // API 서버거 내려주는 코드 (0이 아닌 경우 에러)

  constructor({
    name,
    message,
    statusCode,
    apiCode
  }: {
    name: ErrorName;
    message: string;
    statusCode: number;
    apiCode?: number;
  }) {
    super({ name, message });
    this.statusCode = statusCode;
    this.apiCode = apiCode;
  }
}
```

## 에러 던지기
위에서 커스텀 에러 클래스를 정의 했으니 HTTP 응답을 받을 때 에러라고 판단한 부분에서 에러를 던져보자!

```typescript
export default defineNuxtPlugin(() => {
  // 중략 ...
  const fetchOptions: FetchOptions = {
    // 중략 ...
    onResponse(ctx) {
      if (ctx.response.ok && ctx.response._data) {
        const { code, message, data } = ctx.response._data;
        if (code === 0) {
          ctx.response._data = data;
        } else {
          // 💩 API 서버 에러 💩
          throw new ResponseError({
            name: 'RESPONSE_ERROR_API',
            message,
            statusCode: 200,
            apiCode: code
          });
        }
      }
    },
    onResponseError(ctx) {
      // 💩 status code !== 200 ~ 299 💩
      throw new ResponseError({
        name: 'RESPONSE_ERROR_STATUS',
        message: ctx.response.statusText,
        statusCode: ctx.response.status
      });
    }
  };
  // 중략 ...
});
```

> $fetch <br />
Nuxt includes the <b>ofetch library</b>, and is auto-imported as the $fetch alias globally across your application. It's what useFetch uses behind the scenes.
{: .prompt-tip }

Nuxt 3 는 기본적으로 ofetch 라이브러리를 사용한다. 클라이언트에서 API를 호출할 때마다 Nuxt plugin 에 정의한 ofetch api helper를 호출하기 때문에 해당 부분에 에러를 던지는 코드를 작성했다.

## 에러 핸들링 함수 정의
위에서 에러를 `throw` 했으니, `catch` 구문 안에서 에러를 핸들링할 함수를 정의하자!

```typescript
/**
 * HTTP Response 에러 핸들링 함수
 * @param error ResponseError 클래스 타입
 * @param overrideHandler 에러 핸들러 함수
 * @param overrideHandler.apiErrorHandler api 응답 에러 오버라이드 핸들러 함수
 * @param overrideHandler.statusErrorHandler status 응답 에러 오버라이드 핸들러 함수
 */
export async function handleResponseError(
  error: ResponseError,
  overrideHandler?: {
    apiErrorHandler?: (code: number) => void;
    statusErrorHandler?: (code: number) => void;
  }
) {
  const route = useRoute();
  switch (error.name) {
    // API 서버에서 에러가 발생한 경우
    case 'RESPONSE_ERROR_API':
      if (overrideHandler?.apiErrorHandler) {
        // custom 동작 & elary return
        overrideHandler.apiErrorHandler(error.apiCode!);
        return;
      }

      // default 동작
      showFailAlert({ text: ErrorMessages[error.apiCode!] });
      break;

    // HTTP status code 가 200 ~ 299 가 아닐 때 
    case 'RESPONSE_ERROR_STATUS':
      if (overrideHandler?.statusErrorHandler) {
        // custom 동작 & elary return
        overrideHandler.statusErrorHandler(error.statusCode);
        return;
      }
	
      // default 동작
      if (error.statusCode === 401) {
        const isOk = await showConfirm({
          html: '인증 세션이 만료됐습니다. <br/>다시 로그인해 주세요.',
          icon: 'warning',
          confirmButtonText: '로그인',
          showCancelButton: false
        });

        if (isOk) {
          sessionStorage.setItem('path_when_success_login', route.path);
          navigateTo(getSSOLoginURL(), { external: true });
        }
      } else {
        showError({ statusCode: error.statusCode });
      }
      break;

    default:
      throw new Error(`Unhandled response error: ${error.name}`);
  }
}
```

## 에러 핸들링 함수 사용
```typescript
async function saveCategory() {
  // ... 중략
  try {
    isUpdate.value
      ? await $api.category.updateCategory(payload)
      : await $api.category.createCategory(payload);

    showSimpleMessage();
    emit('onSave');
    close();
  } catch (error) {
    if (error instanceof ResponseError) {
      // 핸들링 함수 호출
      await handleResponseError(error);

      // 만약 기본 동작말고 커스텀하게 에러를 처리하고 싶은 경우 아래와 같이 코드를 작성
      await handleResponseError(error, {
        apiErrorHandler: (apiCode) => {
          // apiCode 에 따라 에러 핸들링
        }
      });

    } else {
      throw error;
    }
  }
}
```
`catch` 내부에서 사용하는 `handleResponseError` 의 첫번째 인자로 `ResponseError` 타입만 전달받을 수 있으므로 `instanceof` 타입 가드를 이용해 타입을 좁혔다. 다른 방법으로 타입 단언(`as`) 구문을 이용할 수 있지만 `try` 구문 내부에 `ResponseError` 에러 외 다른 에러가 발생할 수 있기 때문에 사용하지 않는게 좋다. 

또 주목할 점은 `catch` 구문 내부에서 `ResponseError` 외에 에러를 다시 `throw` 했다. 그 이유는 예상치 못한 에러가 무시될 수 있기 때문에 에러를 다시 던졌다. 


# 더 나아가기
`try`, `catch` 문을 사용해 예상할 수 있는 에러를 핸들링 하는 것에 부정적인 시각이 있다. 이유를 살펴보기에 앞서 에러와 예외를 구분해보자!

- <b>Error (expected)</b>
  - 예상할 수 있는 에러를 의미한다.
  - 예를 들면, 로그인 실패는 앱을 사용하는 유즈케이스 중 발생할 수 있는 케이스 중 하나이기 때문에 에러에 해당한다.
- <b>Exception (unexpected)</b>
  - 예외는 예상할 수 없다.
  - 예를 들면, 특정한 경로의 파일을 읽어서 데이터를 보여주는 앱은 해당 결로가 100% 존재한다고 확신하지만 만약의 경우를 대비해서 파일을 읽지 못하는 예외를 처리해야한다. 다른 예로는 메모리 부족, 보안 에러 등 시스템적인 예외가 있다.

그렇다면 에러(위에서 설명한 Error)를 `try`, `catch`로 처리하는 것에 대해 왜 부정적인 시각이 있을까? 

1. <b>예외 누락의 가능성</b>
  -  `try`, `catch` 블록을 사용할 때, 예외를 적절히 처리하지 않고 넘어가는 경우 심각한 버그로 이어질 수 있다.
  - 예를 들면, `catch` 문 내부 단순 로그만 찍는 경우 앱 실행이 중단되지 않고 지속돼 의도치 않은 버그가 발생할 수 있다.
2. <b>코드 가독성 및 유지 관리</b>
  - `try`, `catch`를 사용하면 코드의 흐름이 복잡해지고 가독성이 떨어진다. `try` 블록 안에 많은 코드가 포함되어 있을 경우, 어떤 부분에서 예외가 발생했는지 추적하기 어려워진다.
3. <b>성능 최적화 문제</b>
  - `try`, `catch` 구문안에 있는 코드는 컴파일러가 최적화 하지 못한다.

## `try`, `catch` 사용하자 않고 에러 핸들링하기
```typescript
type ErrorState = {
  result: "fail";
  reason: "offline" | "down" | "timeout";
};

type SuccessState = {
  result: "success";
};

type ResultState = SuccessState | ErrorState;
```
에러 데이터의 타입(`ErrorState`)과 성공 시 데이터 타입(`SuccessState`)을 정의한다.
<br /><br />

```typescript
class UserService {
  // 중략...

  login(): ResultState {
     // 💩 에러 발생 상황 가정 💩
    return { result: "fail", reason: "timeout" };
  }
}
```
로그인 상황에서 예측 가능한 에러가 발생했다고 가정했다.
<br /><br />

```typescript
const res = userService.login();
if (res.result === "success") {
  // ...
} else {
  // res.reason 에 따라 에러 핸들링, 대화 상자를 보여주는 등
}
```
`else` 블록에서 `ErrorState` 타입에 정의한 `reason`에 따라 적절하게 에러를 핸들링한다.
<br /><br />

위 방법도 언젠가 사용하면 좋을 것 같다.