# Unstoppable

Example: link to [[Mermaid Diagrams]] under `Features`

The goal is stop pool from offering flash loan.
How does flash loan works?
In simple terms, flash loan are broken down into 3 steps
1) A loan from a pool of money is sent to receiver
2) Receiver will use the money to execute a certain action from another smart contract
3) Receiver will return the money with fixed rate (Something like an interest rate)

Reference: https://10clouds.com/blog/defi/understanding-flash-loans-in-defi/#:~:text=What%20is%20a%20DeFi%20Flash,loans%20from%20lenders%20without%20intermediaries.

Let's start by breaking down the source code into functions.
There are two files to ***UnstoppableLender.sol*** and ***ReceiverUnstoppable.sol*** 
***UnstoppableLender.sol*** will loan money to receiver and execute functions from ***ReceiverUnstoppable.sol***
So let's look at ***UnstoppableLender.sol*** first
There are a few variables declared, one consutructors and two functions in the smart contract

For starters, this contract mainly use damnValuableToken as their main currency - ```Solidity IERC20 public immutable damnValuableToken```
```Solidity uint256 public poolBalance``` is created to store a pool token ready to be loaned. 
Firstly, borrowamount needs to be more than 0.




Next, balancebefore is a variable to store the number of tokens before executing flash loan



Next, poolBalance must be the same as balancebefore


Poolbalance can only be added via depositTokens


However, we can add tokens directly into the pool. This will trigger at assert(poolBalance == balanceBefore) which will cause an error. So we can transfer token directly into the pool. However, poolbalance is not updated. Thus, balanceBefore will be most updated of the token pool but it will not match poolbalance


