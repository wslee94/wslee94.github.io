---
title: 페이지 분리하기
date: 2025-05-30 16:30:00 +09:00
categories: [Deveploment, Refactoring]
tags: [refactoring, javascript] # TAG는 반드시 소문자로 이루어져야함!
# image: /assets/img/avatar.png
# pin: true
---

# 개요
<div style="display:flex; gap:5px">
  <img src="/assets/img/capture/page-refactoring-1.png" alt="메이플스토리 월드코인 내역 페이지" />
  <img src="/assets/img/capture/page-refactoring-2.png" alt="메이플스토리 수익 내역 페이지" />
</div>

Maplestory Worlds에서 구매자가 월드코인을 이용해 아이템을 구매하면 "월드코인" 페이지에서 구매 내역을 조회할 수 있고, 아이템 판매자는 "수익" 페이지에서 판매 내역을 조회할 수 있다. 

리팩터링 전 "월드코인" 페이지와 "수익" 페이지는 하나의 페이지 컴포넌트를 공유하고 있었다. 프로젝트가 고도화 되면서 구매자 특화 로직과 판매자 특화 로직이 점점 늘어나게 되었고 페이지 코드는 두 개의 로직이 혼재해 주석으로 구매자 로직인지 판매자 로직인지 표시하고 있었다. 아래 리팩터링 전 코드를 보자!

# 리팩터링 전 코드 💩
## UI 분기
```vue
<template>
  <!-- 중략 -->

  <!-- 판매자 화면 -->
  <tr v-if="isRevenue" class="row_revenue">
    <th scope="row" class="table_header">
      <span class="txt_header">수익금액</span>
    </th>
    <td class="table_description">
      <!-- 중략 -->
    </td>
  </tr>
</template>
```

## 로직 분기

```javascript
// 환불 요청 가능 여부 (월드코인 페이지 전용)
isAbleToRequestRefund() {
  return (
    !this.isRevenue &&
    // 중략...
  )
}

// 환불 요청 중인 아이템 여부 (수익 페이지 전용) 
isRefundRequested() {
  return (
    this.isRevenue &&
    // 중략...
  )
},
```

위 코드를 보면 UI, 로직 모두 `isRevenue` 값을 이용해 "월드코인", "수익"을 분기하고 있는 것을 확인할 수 있다. UI에 공통적으로 겹치는 부분이 많아 페이지를 분리할까 말까 고민했지만, 분리하는게 더 좋다고 판단해 페이지를 분리하는 작업을 진행했다. <br /><br />

**기존 통합 페이지에서 "월드코인" 내역, "수익" 내역 페이지로 각각 분리하는게 더 좋다고 판단한 이유?**
- "월드코인", "수익" 내역에서 공통적으로 사용하는 UI는 많지만 서로 연관이 없는 로직들이 한 페이지에 존재해 분리하기로 결정함
- "월드코인", "수익" 내역을 조회하는 API가 분리됨에 따라 페이지를 분리하는게 좋다고 판단함, 각각 요청, 응답값 형태가 미묘하게 다르고 이는 개발자가 코드를 해석하기 어렵게 만듦


# 리팩터링 ✨
## "월드코인", "수익" 내역의 부모 페이지 만들기
```vue
<template>
  <div class="page_coin">
    <div class="header_coin">
      <h2 class="title_header">{ { pageTitle } }</h2>
    </div>
    <div class="content_coin">
      <p class="title_content">주문 상세 정보</p>
      <div class="wrapper_content">
        <!-- 월드코인, 수익 상세 내역이 표시되는 부분 -->
        <nuxt-child />
      </div>
    </div>
  </div>
</template>
```
우선, 기존 페이지에서 공통 레이아웃 부분만 남겨놓고 부모 페이지로 활용하자.

## 공통 UI 추출하기
개요에서 첨부한 페이지 이미지를 보면 "주문일자", "주문번호", "거래구분", "내용", "상품정보" 필드는 "월드코인" 내역과 "수익" 내역에서 공통적으로 사용하는 부분이다. 해당 부분을 발라내어 컴포넌트로 만들어보자.

**CoinDetailBase 컴포넌트**
```vue
<template>
  <coin-detail-table>
    <coin-detail-table-row label="주문일자">
      { { registerDate | dateformatCommon } }
    </coin-detail-table-row>
    <coin-detail-table-row label="주문번호">
      { { serialNumber } }
    </coin-detail-table-row>
    <coin-detail-table-row label="거래구분">
      { { orderType } }
    </coin-detail-table-row>
    <coin-detail-table-row label="내용">
      { { orderItemType } }
    </coin-detail-table-row>
    <coin-detail-table-row label="상품정보">
      <span v-if="gameName" class="txt_name">
        { { gameName } }
      </span>
      <span>
        { { orderItem } }
      </span>
    </coin-detail-table-row>
    <coin-detail-table-row label="상품가격">
      <img-icon name="w_coin" type="svg" :size="20" />
      <span class="txt_coin">
        { { usageAmount | formatNum } }
      </span>
    </coin-detail-table-row>
    
    <!-- 월드코인, 수익 전용 필드들은 slot을 이용해 사용하는 곳에서 입력받는다. -->
    <slot name="additionalRows" />
  </coin-detail-table>
</template>
```
공통적인 필드 외 "월드코인", "수익" 전용 필드는 vue의 slot 기능을 이용해 사용하는 곳에서 넣어주도록 한다. `CoinDetailTable`과 `CoinDetailRow` 컴포넌트는 기존에 `table` 관련 HTML 태그를 UI 컴포넌트로 만들었다. (이게 중요한게 아니니 Pass)

## 페이지 분리하기
### "월드코인" 내역 페이지
```vue
<template>
  <div class="page_coin_detail_purchase">
    <!-- CoinDetailBase 컴포넌트를 이용해 공통 UI 표시 -->
    <coin-detail-base
      :register-date="registerDate"
      :serial-number="serialNumber"
      :search-type="searchType"
      :item-type="itemType"
      :shop-name="shopName"
      :item-name="itemName"
      :usage-amount="usageAmount"
    >
      <!-- 월드코인 내역 UI -->
      <template #additionalRows>
        <coin-detail-table-row label="판매자">
          <button
            v-if="isCancelablePurchase"
            class="button_status"
            @click="cancelPurchase"
          >
            청약 철회 하기
          </button>
          <button
            v-else-if="isEnableInquiry"
            class="button_status"
            @click="openChatPanel"
          >
            1:1 대화하기
          </button>
          <span>{ { sellerText } }</span>
        </coin-detail-table-row>
        <coin-detail-table-row label="상태">
          <div class="wrapper_status">
            <div class="wrapper_txt">
              <span class="txt_status">
                { { statusText } }
              </span>
              <span v-if="statusSubtext" class="txt_status_date">
                ({ { statusSubtext } })
              </span>
            </div>
            <button
              v-if="isRefundPending"
              class="button_status"
              @click="cancelRefundRequest"
            >
              환불 요청 취소
            </button>
            <button
              v-else-if="isAbleToRequestRefund"
              class="button_status"
              @click="openRefundRequest"
            >
              환불요청
            </button>
          </div>
        </coin-detail-table-row>
      </template>
    </coin-detail-base>
  </div>
</template>
<script>
// 중략 ...
export default {
  name: 'PageCoinHistoryDetailPurchase',
  components: { CoinDetailTableRow, CoinDetailBase },
  async asyncData(context) {
    // 중략 ... (월드코인 조회 API 호출)
  },
  computed: {
    // 구매자 전용 computed
    sellerText() { /* 중략 */ },
    statusText() { /* 중략 */ },
    statusSubtext() { /* 중략 */ },
    computedUsageValid() { /* 중략 */ },
    isMSWProduct() { /* 중략 */ },
    isEnableInquiry() { /* 중략 */ },
    isCancelablePurchase() { /* 중략 */ },
    isRefundPending() { /* 중략 */ },
    isAbleToRequestRefund() { /* 중략 */ },
  },
  methods: {
    // 구매자 전용 methods
    cancelPurchase() { /* 중략 */ },
    getInquiryTargetWithStatus() { /* 중략 */ },
    openChatPanel() { /* 중략 */ },
    openRefundRequest() { /* 중략 */ },
    cancelRefundRequest() { /* 중략 */ }
  }
}
</script>
```

### "수익" 내역 페이지
```vue
<template>
  <div class="page_coin_detail_revenue">
    <!-- CoinDetailBase 컴포넌트를 이용해 공통 UI 표시 -->
    <coin-detail-base
      :register-date="registerDate"
      :serial-number="serialNumber"
      :search-type="searchType"
      :item-type="itemType"
      :shop-name="shopName"
      :item-name="itemName"
      :usage-amount="usageAmount"
    >
      <!-- 수익 내역 UI -->
      <template #additionalRows>
        <coin-detail-table-row label="구매자">
          <span>{ { buyerText } }</span>
        </coin-detail-table-row>
        <coin-detail-table-row label="상태">
          <div class="wrapper_status">
            <div class="wrapper_txt">
              <span class="txt_status">
                { { statusText } }
              </span>
              <span v-if="statusSubtext" class="txt_status_date">
                ({ { statusSubtext } })
              </span>
            </div>
          </div>
        </coin-detail-table-row>
        <coin-detail-table-row
          class="row_revenue"
          label="수익금액"
        >
          <div class="txt_amount" :class="{ refunded: refundCompleted }">
            <span class="txt_coin revenue">
              { { amount | formatNumDecimalThree } } USD
            </span>
          </div>
          <div v-if="isPended && !refundCompleted" class="txt_charge">
            (지급 예정: { { chargeDate | dateformatCommon } })
          </div>
          <button v-if="!refundCompleted" class="button_status" @click="refund">
            환불하기
          </button>
        </coin-detail-table-row>
      </template>
    </coin-detail-base>
  </div>
</template>
<script>
// 중략 ...
export default {
  name: 'PageCoinHistoryDetailRevenue',
  components: { CoinDetailTableRow, CoinDetailBase },
  async asyncData(context) {
    // 중략 ... (수익 조회 API 호출)
  },
  computed: {
    // 판매자 전용 computed
    buyerText() { /* 중략 */ },
    statusText() { /* 중략 */ },
    statusSubtext() { /* 중략 */ },
    computedUsageValid() { /* 중략 */ },
    isRefundRequested() { /* 중략 */ },
    refundCompleted() { /* 중략 */ },
    revenueSharedPerson() { /* 중략 */ }
  },
  methods: {
    // 판매자 전용 methods
    refund() { /* 중략 */ }
  }
}
</script>
```

# 결론
각각 페이지를 분리함으로써 "월드코인", "수익" 관련 로직을 파악하기 쉬워진 것 같다. 또한 "월드코인" 관련 소스코드를 수정할 때 "수익"에 영향이 가지 않으므로(반대도 마찬가지) 수정을 조금 더 맘 편히? 할 수 있게 된 것 같다. 

페이지가 점차 고도화되면서 페이지 내부 분기 처리로 인해 코드를 이해하기 어렵다면 페이지를 분리하는 것이 좋은 해결방안이 될 수 있을 것 같다.

