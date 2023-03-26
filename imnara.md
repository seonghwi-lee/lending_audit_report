# liquidate의 조건 설정 미흡

## 설명

<aside>

### **키워드 : Bug - Condition mistake**

### **심각도 : Informational**

부등호를 잘못 지정하여 정상 실행해야 하는 경우에 revert가 발생한다.

</aside>

```solidity
require(
      _borrow[user] < 100 || amount == ((_borrow[user] * 25) / 100),
      ""
  );
```

`liquidate` 함수 실행 도중 100 ether 보다 적으면 즉시 청산 가능하며, 또는 25% 이하를 투입하여 청산 가능하다. 하지만 로직에서는 이를 등호로만 비교하기에 미만인 경우에도 정상작동 해야 하지만 revert되게 된다.

## 파급력

본래 실행되어야 할 로직이 실행되지 않는 잔버그라고 분류할 수 있다.

## 해결 방법

```solidity
require(
      _borrow[user] < 100 || amount <= ((_borrow[user] * 25) / 100),
      ""
  );
```

다음과 같이 부등호를 추가하여 로직에 맞게 변경할 수 있다.
