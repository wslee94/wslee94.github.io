---
title: 인증(Authentication)과 인가(Authorization)
date: 2024-05-30 14:30:00 +09:00
categories: [Deveploment, Vue]
tags: [vue, nuxt, sso, oauth] # TAG는 반드시 소문자로 이루어져야함!
# image: /assets/img/avatar.png
# pin: true
---

# 개요
기존의 Maplestoryworlds 백오피스는 인증과 인가를 사내에서 제공하는 Proxy Server를 이용했다. 하지만, 신규 프로젝트에 더 이상 해당 서비스를 제공하지 않는다고 해서 HelloMaple 백오피스는 다른 방법으로 개발했다. 이 과정에서 배운 점을 기록하려고 한다.

# 인증(Authentication)과 인가(Authorization)
1. 인증(Authentication)은 자신의 신원을 증명하는 과정으로 로그인을 의미한다. 
2. 인가(Authorization)는 특정 리소스에 접근할 때 권한을 검증하는 것을 의미한다.

# Maplestoryworlds Backoffice API Sequence
![Maplestoryworlds Backoffice API Sequence](/assets/img/capture/auth-1.png)

기존의 API 흐름이다. 인증과 인가 체크를 모두 Proxy Server가 담당하는 것을 확인할 수 있다. 인증과 인가를 통과하지 못하면 백오피스에 접근할 수 없는 구조다.

# HelloMaple Backoffice API Sequence
![HelloMaple Backoffice API Sequence](/assets/img/capture/auth-2.png)

새로운 프로젝트인 HelloMaple 백오피스는 Proxy Server가 사라지고 인증(Authentication)과 인가(Authorization)를 Web Server에서 직접 구현했다.

## HelloMaple Backoffice Authentication Sequence
![HelloMaple Backoffice Authentication Sequence](/assets/img/capture/auth-3.png)

빨간색 부분은 Browser ←→ Web Server 간 세션이 유효하지 않을 때 Sequnce를 나타낸다. OAuth 2.0의 Authorization Code Grant 인증 방식을 사용하는데 Authorization Server가 Resource Server의 역할도 같이 수행한다.

<br />
각 단계에 대해 설명해보면, 
- **2-1** <br /> OAuth 2.0의 Authorization Code Grant 인증 방식은 Redirection 기반으로 진행되므로 세션이 만료된 경우 Authorization Server로 Redirection 한다.
- **2-2** <br /> 인증 코드는 Authorization Server에서 발급하는 액세스 토큰 가져오기 위한 일회성 코드다.
- **2-3~4** <br /> Authorization Server 자체에서 관리하는 세션을 체크한다. 세션이 유효할 경우 로그인 과정이 생략되고 바로 인증 코드를 발급한다. 인증 코드 발급 경로는 사전에 정의한 Redirect URL이다. 
- **2-5** <br /> Browser는 JWT를 발급받기 위해 전달 받은 인증 코드를 Web Server에 전달한다. 
- **2-6~9** <br /> Web Server는 전달받은 인증 코드를 기반으로 Authorization Server로부터 액세스 토큰을 발급받고 사내 프로필 정보를 조회한다.
- **2-10~11** <br /> 사내 프로필 정보 중 사번 등 API Server와 통신할 때 필요한 정보를 Payload에 담아 JWT를 생성한다.


<br />
**여기서! Browser ←→ Web Server 세션 유지용으로 Authorization Server가 발급한 액세스 토큰을 사용하지 않고 별도로 JWT를 사용하는 이유에 관해서 설명하면,** 

1. 보안 위험 최소화 <br /> 액세스 토큰을 가지고 사내 민감한 정보에 접근할 수 있다. Resource Owner(Browser)에 토큰을 저장하는 것은 디바이스가 해킹당할 경우, 액세스 토큰이 노출되어 보안 위험을 초래할 수 있다.
2. 액세스 토큰의 짧은 만료 시간 <br /> 액세스 토큰의 만료되면 Redirection 기반으로 인증을 진행하기 때문에 기존의 작업 중인 내용이 있으면 정보가 날아갈 위험이 있다. 리프레시 토큰을 이용해 액세스 토큰을 재발급받을 수도 있겠지만, 리프레시 유효시간도 짧을뿐더러 리프레시 토큰을 저장하기 위해 Web Server에 DB와 같은 별도 저장소가 필요하다.

결론은, OAuth 2.0 프로토콜에서 안정성, 관리의 효율성 및 사용자 경험을 고려하여 액세스 토큰은 사용자의 디바이스가 아닌 서버 측에서 관리하도록 권장한다.

## HelloMaple Backoffice Authorization Sequence
![HelloMaple Backoffice Authorization Sequence](/assets/img/capture/auth-4.png)

빨간색 부분은 로그인한 사용자가 권한이 없는 경우를 나타낸다. 인증(Authentication) 과정에서 얻은 프로필 정보를 기반으로 Open API Server에 권한 여부를 조회한다.

# 구현하기
인증(Authentication) & 인가(Authorization) 로직을 구현할 때 크게 두 가지를 신경써야 했다. 
1. 페이지 이동(= Route 변경) 시 Web Server 호출이 없는 SPA로 동작 하므로 이를 체크하기 위한 별도 API가 필요했다.
2. CRUD에 해당하는 API는 Backoffice Web Server를 경유해 Backoffice API Server로 가기에 Web Server의 미들웨어에서 처리했다.

## 페이지 이동 시 체크하기
Route 변경 시 인증(Authentication) & 인가(Authorization)를 체크하기 위해 Backoffice Web Server API를 호출해야 한다.

**서버: server/routes/apis/auth/validation.ts**
```typescript
// route 변경 시 인증 & 인가 체크하는 API
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig();
  const authToken = getCookie(event, 'auth_token') || '';
  const { pagePath } = getQuery(event);
  const isSkipAuthCheck = PATHS_NO_AUTH.includes(pagePath as string); // 인증코드 발급 페이지 제외 처리

  if (isSkipAuthCheck) {
    return setResponseStatus(event, 200);
  }

  // 인증 상태 체크
  try {
    await verifyToken(authToken);
  } catch {
    return setResponseStatus(event, 401);
  }

  // 권한 상태 체크
  const { empNo } = decodeToken(authToken);
  const hasRole = await verifyRole({ empNo, resource: pagePath as string });
  if (!hasRole) {
    return setResponseStatus(event, 403);
  }

  return setResponseStatus(event, 200);
});
```

**클라이언트: middleware/auth.global.ts**
```typescript
// route 변경 시 인증 세션 체크 (CSR 로 동작하므로, 라우트 변경 시 서버에 인증 & 권한 체크 API 호출)
export default defineNuxtRouteMiddleware(async (to) => {
  try {
    await checkAuth(to.path);
  } catch (error: any) {
    // 인증(Authentication) 에러
    if (error.statusCode === 401) {
      sessionStorage.setItem('path_when_success_login', to.path);
      return navigateTo(getSSOLoginURL(), { external: true });
    }
    // 인가(Authorization) 에러
    if (error.statusCode === 403) {
      showError({ statusCode: 403 });
      return abortNavigation(error);
    }
  }
});

// 참고를 위해 asset/ts/auth/util.ts, checkAuth 함수 코드 아래에 기입
/**
 * 라우트 변경 시 마다 호출되는 세션 체크 함수
 * @param pagePath - 라우트 경로
 */
export async function checkAuth(pagePath: string) {
  const { boURL } = useRuntimeConfig().public;

  // 에러 시 401 응답 코드 반환
  await $fetch('/apis/auth/validation', {
    baseURL: boURL,
    query: { pagePath }
  });
}
``` 

## CRUD API 호출 시 체크하기 
Backoffice API Server에서 제공하는 API를 호출할 경우 Backoffice Web Server 앞 단의 미들웨어를 통해 인증(Authentication) & 인가(Authorization)를 체크한다. 

**서버: server/middleware/01.auth.ts**
```typescript
// CRUD API 호출 시 인증 & 인가 체크
export default defineEventHandler(async (event) => {
  const { pathname: apiPath } = getRequestURL(event);
  const authToken = getCookie(event, 'auth_token') || '';
  const isProxyApi = PROXY_APIS.some((path) => apiPath.startsWith(path)); // CRUD 관련 API

  if (isProxyApi) {
    // 인증(Authentication) 체크
    try {
      await verifyToken(authToken);
    } catch {
      // 인증이 유효하지 않을 때 요청을 다음 미들웨어로 전달하지 않고 응답
      event.node.res.statusCode = 401;
      event.node.res.end();
      return;
    }

    // 인가(Authorization) 체크
    const { empNo } = decodeToken(authToken);
    const hasRole = await verifyRole({ empNo, resource: apiPath });
    if (!hasRole) {
      // 권한이 없는 경우 요청을 다음 미들웨어로 전달하지 않고 응답
      event.node.res.statusCode = 403;
      event.node.res.end();
      return;
    }
  }

  // 모두 통과되면 다음 미들웨어로 요청 전달
});
```

**클라이언트: plugins/api.ts**
```typescript
export default defineNuxtPlugin(() => {
  const config = useRuntimeConfig();
  const baseURL = config.public.boURL;
  const route = useRoute();

  const fetchOptions: FetchOptions = {
    // 중략...
    async onResponseError(ctx) {
      // 인증 & 인가 에러 발생한 경우
      if (ctx.response.status === 401) {
        // 인증(Authentication) 에러
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
      } else if (ctx.response.status === 403) {
        // 인가(Authorization) 에러 403
        showError({ statusCode: 403 });
      }

      throw new ResponseError(
        'RESPONSE_ERROR_STATUS',
        ctx.response.statusText,
        ctx.response.status
      );
    }
  };
});
```

<br /><br />
좋은 기회가 되어 OAuth 2.0, SSO, 인증과 인가에 대해 생각해 보고 개발해 볼 수 있었다.