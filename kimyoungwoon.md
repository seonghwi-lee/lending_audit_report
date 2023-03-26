# withdraw 예치금 계산 실수

## 설명

<aside>

### **키워드 : Mistake, Logic Bug**

### **심각도 : Medium**

예치금 계산 과정에서 일부 토큰을 정산하려고 시도 해도 전체 토큰에 대한 정산만 구현되어 있기에 일부 정산을 청구할 수 없다.

</aside>

```solidity
amount = (getAccruedSupplyAmount(tokenAddress) / WAD) * WAD;
            userBalances[msg.sender].balance +=
                amount -
                userBalances[msg.sender].balance;
            ERC20(usdc).transfer(msg.sender, amount);
            userBalances[msg.sender].balance -= amount;
```

`withdraw` 함수를 실행함에 있어서 usdc를 정산하면 amoun를 어떻게 지정하든 반드시 내가 공급한 usdc 공급량이 한 번에 정산되게 된다. `getAccruedSupplyAmount(tokenAddress)` 는 유저가 lending 서비스에 예치한 토큰 공급량의 이자를 더한 값인데 이를 바로 사용하고 있으며 amount를 사용하지 않고 계산한다. 따라서 지금껏 공급한 모든 예치금의 이자를 받으며 정산하는 로직으로 동작하게 된다.

## PoC

```solidity
function testPOC() external {
        usdc.transfer(address(0x03), 100 ether);
        vm.startPrank(address(0x03));
        {
            usdc.approve(address(lending), 100 ether);
            lending.deposit(address(usdc), 100 ether);
        }
        vm.stopPrank();
        lending.deposit{value: 100 ether}(address(0x00), 100 ether);
        lending.borrow(address(usdc), 100 ether);
        lending.deposit(address(usdc), 100 ether);
        lending.withdraw(address(usdc), 1 ether);
    }
```

```solidity
Logs:
  res 100000000000000000000 100000000000000000000
  res 0
```

출력 값은 순서대로 withdraw에서 사용한 `amount` , `userBalances[msg.sender].balance` , 모든 연산이 끝난 뒤의 `userBalances[msg.sender].balance` 이다.

즉 지금껏 예치한 모든 USDC가 인출되었음을 알 수 있다.

## 파급력

현재 인출하고자 하는 금액을 함께 계산하는 식의 부재로 발생한 로직 버그로 볼 수 있다. 또한 이는 사용할 수 있는 기능의 제한이며 불편한 사항일 뿐 유저의 손해가 발생하거나 이익이 발생하는 부분은 아니기에 위험도는 없다고 볼 수 있다.

## 해결 방법

```solidity
require(amount <= usdc.balanceOf(msg.sender));

amount = (amount/usdc.balanceOf(msg.sender)*(getAccruedSupplyAmount(tokenAddress) / WAD) * WAD;
```

다음과 같이 전체 공급량 중에 현재 정산하고자 하는 토큰의 비율을 구하여 연산에 포함시키는 로직을 추가하면 된다.

---

# withdraw round down으로 인한 인출 불가 및 부적절한 이자 산정

## 설명

<aside>

### **키워드 : 인출 불가, Numeric issue(Round down, integer underflow), 부적절한 이자 산정**

### **심각도 : Low**

Deposit한 토큰량이 1 ether 이하라면 WAD(10\*\*18)로 나눌 때 round down이 발생하여 인출량이 0으로 산정되고 integer underflow의 원인이 되어 인출이 불가능해진다.

</aside>

```solidity
uint256 public immutable WAD = 10 ** 18; // FixedPointMathLib's WAD constant

amount = (getAccruedSupplyAmount(tokenAddress) / WAD) * WAD;
            userBalances[msg.sender].balance +=
                amount -
                userBalances[msg.sender].balance;
```

WAD로 나눈 뒤 곱하는 연산을 취한다. 이 과정에서 만약 내가 정산받아야 할 총 금액이 1 ether를 넘지 못한다면, 혹은 1 ether로 나누어 떨이지지 않는다면 그 만큼 모두 내림이 발생하여 정산받지 못하게 된다.

심지어 1 ether 를 넘지 못하는 경우에는 amount가 0으로 산정되어 Integer underflow의 발생원인이 된다.

## PoC

```solidity
function testPOC() external {
        lending.deposit(address(usdc), 0.1 ether);
        lending.withdraw(address(usdc), 1 ether);
    }
```

```solidity
Running 1 test for test/LendingTest.t.sol:Testx
[FAIL. Reason: Arithmetic over/underflow] testPOC() (gas: 146056)
Test result: FAILED. 0 passed; 1 failed; finished in 5.50ms

Failing tests:
Encountered 1 failing test in test/LendingTest.t.sol:Testx
[FAIL. Reason: Arithmetic over/underflow] testPOC() (gas: 146056)
```

다음과 같이 0.1 ether를 예치한 결과

`(getAccruedSupplyAmount(tokenAddress)` 가 0.1 ether가 반환되고, 이를 10\*\*18로 나누기에 0으로 round down이 발생한다. 이후 이 값을 `userBalances[msg.sender].balance`  인 0.1 ether로 뺄셈을 시도하기에 integer underflow로 revert가 발생하게 된다. 이는 곧 인출의 불가를 의미한다.

### 파급력

개인이 본인의 예치금을 정산할 수 없는 상황과, 유저들의 예치금에 쌓이는 이자가 ether 단위 만큼 쌓이지 않으면 지급받지 못하는 상황도 발생할 수 있으며 토큰 사용 도중 잔액이 남는 순간 1 ether 이하는 아예 정산이 막히는 등 다양한 문제들의 원인이 될 수 있다.

이는 개인들의 손해에 그치며 이익을 취할 수단은 없지만, 발현 난이도가 상당히 쉬우며 의도하지 않아도 충분히 지속적으로 모든 유저들의 손해가 발생할 수 있는 부분이라고 생각하여 위험도는 중간이라고 판단하였다.

### 해결 방안

```solidity
getAccruedSupplyAmount(tokenAddress);
```

WAD 로 나누는 연산을 없애 round down을 방지할 수 있다.
