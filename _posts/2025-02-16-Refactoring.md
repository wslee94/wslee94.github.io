---
title: 리팩터링 조건부 로직을 다형성으로 바꾸기
date: 2025-02-16 18:30:00 +09:00
categories: [Deveploment, Refactoring]
tags: [refactoring, javascript, polymorphism] # TAG는 반드시 소문자로 이루어져야함!
# image: /assets/img/avatar.png
# pin: true
---

# 개요
![메이플스토리 월드 코인 내역 페이지](/assets/img/capture/if-refactoring-1.png)

Maplestory Worlds에서 사용하는 화폐를 코인(월드코인, 리워드코인)이라고 한다. 위 이미지는 코인 내역을 조회하는 페이지인데 사용자가 아바타 상품, 게임 아이템 등을 구매하거나 반대로 크리에이터가 아바타 상품, 게임 아이템 등을 판매해 벌어들인 코인 내역을 모두 조회할 수 있다. 

코인이 점점 고도화됨에 따라 컴포넌트 내부 조건식이 너무 많아 코드를 읽기 힘들었다. 할당된 작업도 모두 처리했겠다, 시간이 난 김에 리팩터링을 진행했다.  

# 개선점: 복잡한 조건식
위 이미지에서 "5천 메이플 포인트 교환권"을 가져오는 로직을 살펴보자.

```javascript
getItemContents(data) {
  switch (data.searchType) {
    case BILLING_SEARCH_TYPE.USAGE:
    case BILLING_SEARCH_TYPE.REVENUE:
      return data.itemType === 'NONE'
        ? // 중략...
        : // 중략...
    case BILLING_SEARCH_TYPE.CHARGE:
      return // 중략...
    case BILLING_SEARCH_TYPE.FUND:
      switch (data.chargeType) {
        case BILLING_CHARGE_TYPE.REWARD_EVENT_PARTICIPATION:
          // 중략...
        case BILLING_CHARGE_TYPE.REWARD_ERROR_COMPENSATION:
          // 중략...
        case BILLING_CHARGE_TYPE.REWARD_APOLOGY_COMPENSATION:
          // 중략...
        case BILLING_CHARGE_TYPE.FREE_WORLD_COIN_EVENT_PARTICIPATION:
          // 중략...
        case BILLING_CHARGE_TYPE.FREE_WORLD_COIN_ERROR_COMPENSATION:
          // 중략...
        case BILLING_CHARGE_TYPE.FREE_WORLD_COIN_APOLOGY_COMPENSATION:
          // 중략...
        case BILLING_CHARGE_TYPE.FREE_WORLD_COIN_PROMOTION_SUPPORT:
          // 중략...
        case BILLING_CHARGE_TYPE.FREE_WORLD_COIN_EVENT_WINNER:
          // 중략...
        case BILLING_CHARGE_TYPE.FREE_WORLD_COIN_THANK_YOU:
          // 중략...
        case BILLING_CHARGE_TYPE.FREE_WORLD_COIN_REALTIME_EVENT: {
          // 중략...
        }

        default:
          return '-'
      }
    case BILLING_SEARCH_TYPE.WITHDRAWAL:
      switch (data.itemCode) {
        case 'REJECTEDFEE':
          return // 중략...
        case 'WITHDRAWAL':
          return // 중략...
        default:
          return // 중략...
      }
  }
}
```
"5천 메이플 포인트 교환권" 값 이외에 다른 값을 가져올 때 역시 마찬가지로 `switch...case` 문을 활용해 타입에 알맞은 데이터를 계산하고 있다. 한 페이지 내부에 이러한 조건식이 너무 많아 어떤 타입의 코인 내역이 어떤 데이터를 활용하는지 한눈에 파악하기가 힘들다.

# 리팩터링: 복잡한 조건식
위에서 코인 내역 종류는 `searchType` 값에 따라 분기하는 것을 확인할 수 있다. 클래스와 상속을 활용해 복잡한 조건식을 없애보자!

## 부모 클래스 정의하기
```javascript
// 부모 클래스
class CoinHistory {
  constructor({ searchType, chargeType, itemType, ... }, i18n) {
    // 중략...
  }

  // 추상 메서드, 상속하는 클래스에서 구현해야 함
  // getItemContents(vue2 methods) → historyItemName(class getter)로 이름 변경
  get historyItemName() {
    throw new Error('Not implemented')
  }

  // 추상 메서드, 상속하는 클래스에서 구현해야 함
  // getChipText(vue2 methods) → historyStatus(class getter)로 이름 변경
  get historyStatus() {
    throw new Error('Not implemented')
  }

  // 중략... (모든 코인 내역에 공통으로 사용하는 필드는 부모클래스 getter 로 구현했음)

  // 팩터리 메서드, searchType 에 따라 CoinHistory 를 상속한 인스턴스 생성
  static create(data, i18n) {
    switch (data.searchType) {
      case BILLING_SEARCH_TYPE.USAGE:
        return new UsageCoinHistory(data, i18n)
      case BILLING_SEARCH_TYPE.REVENUE:
        return new RevenueCoinHistory(data, i18n)
      case BILLING_SEARCH_TYPE.CHARGE:
        return new ChargeCoinHistory(data, i18n)
      case BILLING_SEARCH_TYPE.FUND:
        return new FundCoinHistory(data, i18n)
      case BILLING_SEARCH_TYPE.WITHDRAWAL:
        return new WithdrawalCoinHistory(data, i18n)
      default:
        throw new Error('Invalid search type')
    }
  }
}
```
먼저 여러 가지 타입의 코인 내역 클래스가 상속할 부모 클래스를 선언한다. 
- 모든 코인 내역에서 공통으로 사용하는 데이터는 `getter` 를 직접 구현한다.
- 타입별로 달라지는 데이터는 자식 클래스에서 오버라이드할 수 있도록 `getter` 추상메서드 형식으로 작성한다. (Javascript에서는 추상 메서드를 지원하지 않아 `throw` 문법을 사용함)

## 자식 클래스 정의하기
```javascript
// 자식클래스: "사용"한 코인 내역 클래스
class UsageCoinHistory extends CoinHistory {
  // 오버라이드
  get historyItemName() {
    return this.itemType === ITEM_TYPE.AVATAR_FEE
      ? // 중략...
      : // 중략...
  }
  
  // 오버라이드
  get historyStatus() {
    if (this.usageValid === USAGE_VALID_TYPE.REFUND) {
      // 중략...
    }
    return ''
  } 

  // 중략...
}

// 자식클래스: "수익" 코인 내역 클래스
class RevenueCoinHistory extends CoinHistory {
  // 중략...
}

// 자식클래스: "충전" 코인 내역 클래스
class ChargeCoinHistory extends CoinHistory {
  // 중략...
}

// 자식클래스: "적립" 코인 내역 클래스
class FundCoinHistory extends CoinHistory {
  // 중략...
}

// 자식클래스: "출금" 코인 내역 클래스
class WithdrawalCoinHistory extends CoinHistory {
  // 중략...
}
```
타입(사용, 수익, 충전, 적립, 출금)에 따라 자식 클래스를 생성한다. 타입별로 달라지는 내용은 자식 클래스에서 메서드 오버라이드 한다.

## 컴포넌트 사용 코드
### CoinList.vue
```vue
<template>
<!-- 중략... -->
  <ul v-show="!noData" class="list_coin_history">
    <li
      v-for="(data, idx) in coinHistoryList"
      :key="idx"
      class="item_coin_history"
    >
      <coin-item :page-type="pageType" :item="createCoinHistory(data)" />
    </li>
  </ul>
<!-- 중략... -->
<template>
<script>
export default {
  // 중략...
  methods: {
    createCoinHistory(data) {
      return CoinHistory.create(data, this.$i18n)
    }
  }
  // 중략...
}
</script>
```

### CoinItem.vue
```vue
<template>
<!-- 중략... -->
  <div class="content_item_name">
    <span v-show="item.worldName" class="txt_shop_name">
      { { item.worldName } }
    </span>
    <span class="txt_item_name">
      { { item.historyItemName } }
    </span>
    <span v-show="item.withdrawalReason" class="txt_withdrawal_reason">
      { { item.withdrawalReason } }
    </span>
  </div>
<!-- 중략... -->
  <div class="area_amount_info">
    <div class="content_amount">
      <span v-if="item.historyStatus">
        { { item.historyStatus } }
      </span>
    </div>
  </div>
<!-- 중략... -->
</template>
```
컴포넌트 내부에서 코인 타입 인스턴스의 필드를 사용하므로 `switch...case` 혹은 `if`로 분기하는 로직들이 사라진 것을 확인할 수 있다.

이렇게 복잡한 조건부 로직을 다형성을 이용해 리팩터링 하면,
- 복잡한 조건식이 사라져 가독성이 향상한다.
- 특정 타입의 코인 내역에서 문제가 발생할 때 해당 클래스만 수정하면 되므로 유지보수가 용이하다. 
- 새로운 타입의 코인 내역이 추가될 때 `CoinHistory`를 상속하는 자식 클래스를 정의하기만 하면 컴포넌트 단에 소스코드를 수정할 필요가 없다.

# 부록 (매직넘버 없애기)
## 리팩터링 전
```javascript
if (
  data.searchType === BILLING_SEARCH_TYPE.WITHDRAWAL &&
  data.usageValid === 2
) {
  // 중략...
}
```
`usageValid` 값 2가 의미하는 게 무엇인지 코드를 봤을 때 이해할 수 있나? 이러한 매직 넘버는 코드 가독성을 저하시킨다. 코드 곳곳에 이러한 매직 넘버가 있어 리팩터링 하면서 매직 넘버를 모두 없애려고 한다.

## 리팩터링 후
```javascript
const USAGE_VALID_TYPE = {
  REFUND: 0,
  NORMAL: 1,
  ROLLBACK: 2
}
```
```javascript
if (
  data.searchType === BILLING_SEARCH_TYPE.WITHDRAWAL &&
  data.usageValid === USAGE_VALID_TYPE.REFUND
) {
  // 중략...
}
```
상수를 정의해 매직 넘버를 제거했다.