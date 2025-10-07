---
title: 부수효과 코드 분리하기
date: 2025-07-15 16:30:00 +09:00
categories: [Deveploment, Refactoring]
tags: [refactoring, javascript] # TAG는 반드시 소문자로 이루어져야함!
# image: /assets/img/avatar.png
# pin: true
---

# 개요
<img src="/assets/img/capture/side-effect-refactoring-1.png" alt="메이플스토리 월드 알림 내역" width="170px" />

Maplestory Worlds에서 알림 타입 추가 작업을 진행하다가 개선점을 발견해 리팩터링 했다. 리팩터링 대상 컴포넌트는 AlarmMenu, AlarmCard 총 2개다. 각각 컴포넌트 역할은 아래와 같다.
- AlarmMenu: AlarmCard의 부모 컴포넌트로 여러 개의 AlarmCard 컴포넌트를 리스트 형태로 렌더링하는 컴포넌트다. 내부에는 무한 스크롤링 로직이 적용되어 있으며 알림 내역을 fetch하는 로직이 들어있다.
- AlarmCard: 알림 내용을 보여주는 컴포넌트다.

# 리팩터링 전 코드 💩
AlarmCard 컴포넌트 내부를 살펴보자.

```vue
<template><!-- 중략 --></template>
<script>
// 중략 ...
export default {
  methods: {
    // 중략 ...

    // 알림 읽음 처리
    async readAlarm() { /* 알림 조회 API 호출 */ },    
    async onClickNewsOff() { /* 특정 월드 알림 수신 비활성화 API 호출 */ },
    async onClickReportAlarm() { /* 알림 신고 가능 여부 조회 API 호출 및 신고 모달 띄우기 */ },
    async onClickDeleteAlarm() { /* 알림 삭제 API 호출 */ },
  }
}
</script>
<style lang="scss" scoped>/* 중략 ... */</style>
```
위 코드는 AlarmCard 컴포넌트 내부에서 비동기 API 호출과 같은 비즈니스 로직을 발췌했다. AlarmCard의 본래 목적은 데이터를 화면에 시각적으로 표현하는 것인데, 비즈니스 로직과 뷰 로직이 혼재될 경우 컴포넌트 역할이 모호해지고, 유지보수가 어려워지는 신호가 될 수 있다.

관련해서 적용할 수 있는 리팩터링은 질의함수와 변경함수 분리하기다. 함수형 프로그래밍에서는 액션(=부수효과를 일으키는 코드)과 계산을 분리한다고도 한다.

질의함수와 변경함수 개념은 마틴파울러의 리팩터링 2판에서 나오는 개념으로 짧게 요약하면
- 질의함수: 여러 번 호출되도 외부에 영향을 주지 않는 함수를 의미한다. 단순히 필드를 가져오거나 혹은 데이터를 가공하는 함수를 말한다.
- 변경함수: 외부에 영향을 주는 함수로 주로 DB 데이터를 업데이트하거나, 메일을 발송하는 등 어떠한 이펙트가 발생하는 함수를 말한다.

AlarmCard에서 부수효과를 일으키는 부분을 분리해 순수함수(=질의함수)로 만들어보자!

# 리팩터링 ✨
**AlarmMenu**
```vue
<template>
  <div>
    <!-- 중략 ... -->
    <alarm-card
      v-for="alarm in alarmList"
      :key="alarm.recodeId"
      alarm-info="alarm"
      @readAlarm="readAlarm"
      @unsubscribeAlarm="unsubscribeAlarm"
      @reportAlarm="reportAlarm"
      @deleteAlarm="deleteAlarm"
    />
  </div>
</template>
<script>
export default {
  methods: {
    // 중략 ...

    // 알림 읽음 처리
    async readAlarm() { /* 알림 조회 API 호출 */ },    
    async unsubscribeAlarm() { /* 특정 월드 알림 수신 비활성화 API 호출 */ },
    async reportAlarm() { /* 알림 신고 가능 여부 조회 API 호출 및 신고 모달 띄우기 */ },
    async deleteAlarm() { /* 알림 삭제 API 호출 */ },
  }
}
</script>
<style lang="scss" scoped>/* 중략 ... */</style>
```

**AlarmCard**
```vue
<template><!-- 중략 ... --></template>
<script>
export default {
  // 중략 ...
  methods: {
    // 중략 ...
    handleMoreClick() {
      this.isOpenMore = !this.isOpenMore
      if (this.isOpenMore) {
        this.$emit('readAlarm', this.alarmInfo)
      }
    },
    handleOptionClick(idx) {
      const value = this.options[idx]?.value
      switch (value) {
        case MORE_ACTION.NEWS_OFF:
          this.$emit('unsubscribeAlarm', this.alarmInfo)
          break
        case MORE_ACTION.REPORT:
          this.$emit('reportAlarm', this.alarmInfo, this.parsedContents)
          break
        case MORE_ACTION.DELETE:
          this.$emit('deleteAlarm', this.alarmInfo)
          break
      }
    }
  }
}
</script>
<style lang="scss" scoped>/* 중략 ... */</style>
```

- AlarmMenu: AlarmCard에 필요한 데이터와 비동기 로직 이벤트를 전달하는 역할을 수행한다.
- AlarmCard: 사용자에게 보여질 View를 담당하며, 전달받은 데이터를 UI에 뿌려주는 역할을 수행한다. 호출 횟수와 관계없이 동일한 입력이 들어오면 동일한 결과를 출력한다. 

# 결론
이번 리팩터링으로 인해 
- AlarmCard 내 비즈니스 로직과 뷰를 분리해 구조가 명확하다.
- AlarmCard의 UI를 재사용하기 용이하다. (= 특정 비즈니스 로직에 종속되지 않음)
- AlarmCard는 순수함수(=외부 의존성을 줄이고, 예측 가능한 결과를 반환하는)로 테스트가 용이하다.

더 나아가서 <br />
이번 리팩터링을 통해 부수효과를 일으키는 비즈니스 로직으로 AlarmMenu 컴포넌트로 끌어올렸다. AlarmMenu의 비즈니스 로직도 분리하려면 Vue의 Composable이나, React의 Custom Hook을 이용해 AlarmMenu 내 비즈니스 로직을 뷰와 분리시킬 수 있다. <br /><br />


마지막으로 리팩터링 2판 내용의 일부를 발췌하면서 마무리 하겠다.

> 우리는 외부에서 관찰할 수 있는 겉보기 부수효과가 전혀 없이 값을 반환해주는 함수를 추구해야한다. 이런 함수는 어느 때건 원하는 만큼 호출해도 아무 문제가 없다. 호출하는 문장의 위치를 호출하는 함수 안 어디로든 옮겨도 되며 테스트하기도 쉽다. 한마디로, 이용할 때 신경 쓸 거리가 매우 적다.

> 나는 값을 반환하면서 부수효과도 있는 함수를 발견하면 상태를 변경하는 부분과 질의하는 부분을 분리하려 시도한다. 무조건이다!
