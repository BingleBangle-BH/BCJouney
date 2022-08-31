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
There are two files to ***UnstoppableLender.sol*** and ***ReceiverUnstoppable.sol***.

***UnstoppableLender.sol*** will loan money to receiver and execute functions from ***ReceiverUnstoppable.sol***.

So let's look at ***UnstoppableLender.sol*** first.

There are a few variables declared, one consutructors and two functions in the smart contract.

For starters, this contract mainly use damnValuableToken as their main currency - ```Solidity IERC20 public immutable damnValuableToken```.
Openzeppelin is used to create the token.
Reference: https://docs.openzeppelin.com/contracts/2.x/api/token/erc20#IERC20

```Solidity uint256 public poolBalance``` is created to store a pool token ready to be loaned. 

Next, a constructor is called whenever the smart contract is deployed.
Next, the tokens will be named as ***damnValuableToken***.
```Solidity
    constructor(address tokenAddress) {
        require(tokenAddress != address(0), "Token address cannot be zero");
        damnValuableToken = IERC20(tokenAddress);
    }
```
It requires ***tokenAddress*** and it performs a check if the address is 0. This will ensure the address integrity.

Now, we can talk about the functions in question.
Let's start with ***depositTokens***.
```Solidity
    function depositTokens(uint256 amount) external nonReentrant {
        require(amount > 0, "Must deposit at least one token");
        // Transfer token from sender. Sender must have first approved them.
        damnValuableToken.transferFrom(msg.sender, address(this), amount);
        poolBalance = poolBalance + amount;
    }
```

The functions reqire ***amount*** to be specified and more than 0 when it's called.
As part of an innate function from IERC20 openzeppelin, ```Solidity damnValuableToken.transferFrom(msg.sender, address(this), amount);``` will then transfer ```Solidity amount``` to ```Solidity address(this)``` from ```Solidty msg.sender```
```Solidity msg.sender``` = sender address (deployer)
```Solidity address(this)``` = receiver address (pool smart contract address)
```Solidity amount``` = amount of tokens deposited to pool

***poolBalance*** will be updated along with the new amount.
Note that this is the only place where ***poolBalance*** can be updated.


Next, ***flashLoan*** is the main function when user requires a flash loan
```Solidity
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


Next, poolBalance must be the same as balancebefore


Poolbalance can only be added via depositTokens


However, we can add tokens directly into the pool. This will trigger at assert(poolBalance == balanceBefore) which will cause an error. So we can transfer token directly into the pool. However, poolbalance is not updated. Thus, balanceBefore will be most updated of the token pool but it will not match poolbalance


