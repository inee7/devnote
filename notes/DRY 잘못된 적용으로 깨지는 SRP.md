---
tags: [oop, dry, srp, design]
---

# DRY 잘못된 적용으로 깨지는 SRP

## 한 줄 요약

DRY 원칙은 코드 중복이 아니라 개념 중복을 제거하는 것이며, 개념이 다르면 코드가 같아도 분리해야 한다

```java
class RegularFee {
  private static final int MIN_FEE = 0;
  private final int amount;

  RegularFee(final int amount){
    if(amount < MIN_FEE) {
        throw new IllegalArgumentException("수수료는 0원 이상이어야 합니다.");
    this.fee = amount;
  }
}


class AdultDiscountFee {
  private static final int MIN_FEE = 0;
  private static final int DISCOUNT_FEE = 100;
  private final int amount;

  AdultDiscountFee(final RegularFee fee) {
    int discountedFee = fee.amount - DISCOUNT_FEE ; 
    if(discountedAmount < MIN_FEE) {
      discountedFee = MIN_FEE;
    }
    amount = discountedFee;
  }
}

class YongDiscountFee {
  private static final int MIN_FEE = 0;
  private static final int DISCOUNT_FEE = 500;
  private final int amount;

  YongDiscountFee(final RegularFee fee) {
    int discountedFee = fee.amount - DISCOUNT_FEE ; 
    if(discountedAmount < MIN_FEE) {
      discountedFee = MIN_FEE;
    }
    amount = discountedFee;
  }
}


```

## 핵심 정리

- DRY 원칙: "코드 중복 금지"가 아니라 "개념 중복 금지"
- 코드가 같아도 개념이 다르면 분리해야 함
- 무리한 중복 제거는 강한 결합을 만들고 SRP를 위반

## 상세 내용

위 코드를 보면 `MIN_FEE` 검증 로직이 중복되어 보이지만, 실제로는 다른 개념이다.

만약 Young 할인에서 "정가 수수료의 5% 할인"으로 요구사항이 바뀌면 코드가 완전히 달라진다. 중복이 아니었던 것이다.

### DRY 원칙의 진짜 의미

> 모든 지식은 시스템 내에서 단 한 번만, 애매하지 않고 권위 있게 표현되어야 한다

여기서 "지식"은 **비즈니스 개념**을 의미한다.

- 어른 수수료와 청소년 수수료는 **다른 개념**
- 코드가 우연히 같을 뿐, 개념적으로 독립적
- 각각 다른 이유로 변경될 수 있음

### 무리한 DRY 적용의 문제

개념적으로 다른 것까지 무리하게 중복 제거하면:
- 강한 결합 상태가 됨
- 한 개념의 변경이 다른 개념에 영향
- **단일 책임 원칙(SRP) 위반**

## 실무 적용

비즈니스 도메인에서 같은 로직처럼 보여도 **변경의 이유**가 다르면 분리한다.

```kotlin
// ✅ 개념적으로 분리
class RegularPrice(val amount: Money)
class MembershipPrice(val amount: Money)

// 현재는 같은 검증 로직이지만
// 각각 다른 정책으로 변경될 가능성이 있음
```


---

**출처**
- 내 코드가 그렇게 이상한가요 (마츠오카 켄타로), 11장

