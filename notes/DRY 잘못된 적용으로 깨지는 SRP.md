#  DRY 잘못된 적용으로 깨지는 SRP

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

위 코드를 보면 같은 코드라서 중복이라고 생각할 수 있지만 그렇지 않다 
만약 YONG에서는 정가수수료에서 5프로 할인한다 라고 바뀌면 중복 적이지 않다 
DRY원칙은 코드 중복 하지 말라가 아니라 개념을 중복하지 말라이다
*모든 지식은 시스템 내에서 단 한 번만, 애매하지 않고 권위 있게 표현되어야한다* 
지식은 여러가지가 있겠지만 그 중 하나는 비지니스 개념을 나타낸다.
어른수수료와 아이수수료는 다른 개념이다. 
개념적으로 다른 것까지도 무리하게 중복을 제거하려 하면 강한 결합 상태가 된다. 단일 책임 원칙을 깨는 것이다. 

#OOP #DRY #SRP
