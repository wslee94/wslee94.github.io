---
title: 타임존 다루기 (feat. UTC offset, Timezone)
date: 2024-08-13 23:20:00 +09:00
categories: [Deveploment, timezone]
tags: [timezone, utc, global, summertime] # TAG는 반드시 소문자로 이루어져야함!
# image: /assets/img/avatar.png
# pin: true
---

# 요구사항
<img src="/assets/img/capture/utc-offset-1.png" alt="utc offset 선택 이미지" /> <br />
MapleStory Worlds 백오피스에서 사용자가 UTC offset(+07:00, +09:00)을 직접 선택하여 날짜 데이터를 입력하는 기능을 추가해달라고 요청받았다. 위 이미지와 같이 UI를 구성하면 백오피스 사용자가 날짜를 적용하려는 국가의 UTC offset을 직접 확인해 설정할 것이라는 답변 받았다. 하지만 내 생각은 사용자가 UTC offset을 직접 설정하는 것은 휴먼 에러가 발생할 수 있을 것 같아서 해당 내용을 정리한다.

# UTC offset 직접 선택?

## 사용자에게 편리한가?
전 세계에는 수십 개의 시간대가 존재하며, 각 시간대는 다른 UTC offset을 갖고 있다. 콘텐츠를 등록할 때마다 노출하려는 국가의 UTC offset을 찾아봐야 하는 번거로움이 존재하고 휴먼 에러가 발생할 가능성이 높다. 각각의 UTC offset에 해당하는 국가 목록은 [여기](https://en.wikipedia.org/wiki/List_of_UTC_offsets){:target="_blank"}를 참고하자!

## 일광절약 시간제 (Daylight Saving Time) a.k.a 서머 타임
>일광 절약 시간제(日光節約時間制, 미국 영어: Daylight saving time, DST) 또는 서머 타임(영국 영어: summer time)은 하절기에 표준시를 원래 시간보다 한 시간 앞당긴 시간을 쓰는 것을 말한다. 즉, 0시에 일광 절약 시간제를 실시하면 1시로 시간을 조정해야 하는 것이다. 실제 낮 시간과 사람들이 활동하는 낮 시간 사이의 격차를 줄이기 위해 사용한다. 여름에는 일조 시간이 길므로 활동을 보다 일찍 시작하여 저녁 때 직장이나 학교에서 이렇게 '절약된 낮 시간'을 더 밝은 상태에서 오후에 활동할 수 있게 하는 효과가 있으며, 또한 직장이나 학교에서의 조명과 연료 등의 절감 효과를 기대할 수 있기 때문이다. 온대 지역에서는 계절에 따른 일조량의 차이가 크므로 일광 절약 시간제는 보통 온대 지역에서 시행된다.<br />
출처: 위키피디아

흠 별걸 다 절약하는군...! 보다 쉽게 이해하기 위해 샌프란시스코 UTC Offset 이미지를 들고 왔다. <br />
<img src="/assets/img/capture/utc-offset-2.png" alt="샌프란시스코 UTC Offset" /> <br />
- 샌프란시스코의 여름(3월10일 ~ 11월2일): UTC-7
- 샌프란시스코의 겨울(11월3일 ~ 3월9일): UTC-8

이외에도 정부의 정책 변경으로 UTC offset이 변경될 수 있다. 예를 들어, 러시아는 2011년에 영구 서머타임을 도입해 모든 시간대를 1시간씩 앞당겼다가, 2014년에 다시 표준 시간대로 복귀했다.

이렇듯 시간대 변경은 휴먼 에러를 발생시킬 가능성이 높다고 생각한다.

# 결론
UTC offset을 선택하는 방법 대신 IANA 데이터베이스에 등록된 타임존 이름을 선택하는 방식이 적합할 것 같다. 시간 관련 라이브러리(ex. dayjs)에서 날짜와 타임존 이름을 전달하면 해당 지역의 시간으로 연산해 결과를 반환한다. 시간 관련 라이브러리를 사용하면 시간대 변경과 같은 부분의 유지보수를 라이브러리 차원에서 지원해 준다고 한다. 아래는 예제 코드이다.

`dayjs.tz("2013-11-18 11:55:20", "America/Toronto") // '2013-11-18T11:55:20-05:00'`

<img src="/assets/img/capture/utc-offset-4.png" alt="샌프란시스코 UTC Offset" /> <br />
자바스크립트에서 `Intl.supportedValuesOf('timeZone')` 코드를 통해 IANA에 등록된 타임존 이름 목록을 가져올 수 있다. 해당 목록을 입력으로 받는 자동 완성 컴포넌트를 통해 사용자가 자기가 노출하려고 하는 국가를 검색하고 선택할 수 있도록 하는 것이 좋을 것 같다.
