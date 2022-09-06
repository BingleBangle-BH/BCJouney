# Naive-Receiver

## Introduction

The goal is to drain the user tokens.
In this section, flash loan from ***NaiveReceiverLenderPool.sol*** impose a fixed interest rate of 1 ether (Holy jesus)
Therefore, 1 token will be extracted from the user for every flash loan.

## Code breakdown

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
```
// Transfer ETH and handle control to receiver
borrower.functionCallWithValue(
    abi.encodeWithSignature(
        "receiveEther(uint256)",
        FIXED_FEE
    ),
    borrowAmount
);
```