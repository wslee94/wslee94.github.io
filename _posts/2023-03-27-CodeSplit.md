---
title: 코드 분할(Code splitting)
date: 2023-03-27 14:00:00 +09:00
categories: [Deveploment, Javascript]
tags: [web, javascript, bundler] # TAG는 반드시 소문자로 이루어져야함!
# image: /assets/img/avatar.png
# pin: true
---

# 번들링

번들링으로 브라우저에서 서버로의 자원 요청 횟수는 줄어들어 네트워크 비용은 줄일 수 있었다. 장점만 있을 수 없진 않은가... 웹 애플리케이션 규모가 커지고 서드파티 라이브러리를 많이 사용할수록 번들링되는 **Javascript의 코드의 양은 터무니없이 커지게 되었다**. 이는 페이지 진입 시 페이지가 그려지는 속도가 느리다던가, 페이지는 보이나 인터랙티브하지 않는 문제점이 발생하게 됐다.

# 코드 분할?
> Code splitting is the splitting of code into various bundles or components which can then be loaded on demand or in parallel.
{: .prompt-tip }

**그러면 다시 쪼개보자!** 라는 개념이 코드 분할(Code Splitting)이다. 하나의 Javascript 파일이 비대해지는 것을 방지하기 위해 여러 개의 chunk로 나눈다. 브라우저는 초기 페이지 진입 시에 필요한 chunk만 다운로드하고 페이지가 그려지고 인터랙티브하게 되면 그 후 다른 chunk를 로드해 성능을 향상한다. 

# Lazy loading

# code splitting 예

# React에서의 Code Split

# Webpack + Vanilla JS Code Split