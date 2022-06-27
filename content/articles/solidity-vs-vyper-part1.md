---
Title: Solidity vs Vyper (Part 1 - Arithmetic overflow)
Date: 2022-06-27 19:30
Category: Smart Contracts
Tags: ethereum, solidity, vyper, erc20
---

On ethereum, there are two main smart contract languages which are currently being used to deploy smart contracts: Solidity and Vyper. In this series of posts, I will discuss some of the peculiarities of writing smart contracts and use examples from both languages to explain them.

The first contract that we will look at is the `ERC20 token`, of which there are many examples such as `Dai`, `WETH` and `UNI`. The smart contracts for these tokens act as a distributed ledger, which keeps track of the balances of all users. When a user transfers their tokens to another user, the smart contract must first check whether the user has sufficient balance, and then update the balances of both the sender and receiver. To highlight some of the differences between Solidity and Vyper, I will display just the code required to transfer tokens from the sender to another user. Full implementations can be found for [Solidity](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol) and [Vyper](https://github.com/vyperlang/vyper/blob/master/examples/tokens/ERC20.vy).

## Solidity

```solidity
pragma solidity ^0.8.0;

contract ERC20 {
    mapping(address => uint256) public balanceOf;

    function transfer(address to, uint256 amount) public returns (bool) {
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }
}

```

## Vyper

```vyper
# @version ^0.3.0
balanceOf: public(HashMap[address, uint256])

@external
def transfer(to: address, amount: uint256) -> bool:
    self.balanceOf[msg.sender] -= amount
    self.balanceOf[to] += amount
    return True
```

It should be apparent that both smart contract languages are very similar in their implementation of an ERC20 token. Both contracts utilise a mapping from an `address` to a `uint256` in order to signify a users balance. 

It is important to highlight what would happen with arithmetic over/underflow. If we allow a user to send tokens when they have an insufficient balance, this would result in that user ending with a negative amount of tokens. As the balance is stored as a `uint256`, this would underflow - causing the user to have a balance in the region of `2 ** 256` which would obviously be a very serious bug! All versions of Vyper natively check for these arithmetic errors, and cause the transaction to be reverted. This feature was also added to Solidity in version 0.8.0. Anybody choosing to write a contract in an older version of Solidity will need to manually check for over/underflow or utilise a library to do this.

To give an illustration of what would happen if we use an older version of Solidity, I have used the exact same code as presented above but changed the compiler version to 0.7.0. If we run the code, we get the following:

```python
>>> sender = accounts[0]
>>> receiver = accounts[1]
>>> overflow_erc20 = sender.deploy(ERC20)
Transaction sent: 0xd7f0c229b122fb0795d03693222b527c5a7f0077651f36c93ec9e58e01125d8a
  Gas price: 0.0 gwei   Gas limit: 12000000   Nonce: 0
  ERC20.constructor confirmed   Block: 1   Gas used: 116515 (0.97%)
  ERC20 deployed at: 0x299784aE2AF66133155848a9D2bcAde3B9f5b32c

>>> overflow_erc20.transfer(receiver, 1, {"from": sender})
Transaction sent: 0x4383921e24d79d235318c2fbad56a191586ab113f1c515ad94bd500f42c156ad
  Gas price: 0.0 gwei   Gas limit: 12000000   Nonce: 1
  ERC20.transfer confirmed   Block: 2   Gas used: 63634 (0.53%)

<Transaction '0x4383921e24d79d235318c2fbad56a191586ab113f1c515ad94bd500f42c156ad'>
>>> overflow_erc20.balanceOf(receiver)
1
>>> overflow_erc20.balanceOf(sender)
115792089237316195423570985008687907853269984665640564039457584007913129639935
```

> This overflow error has resulted in the user now having more tokens than there are atoms in the universe!

Developers must be very careful to ensure that these errors do not occur. In particular, it is important to be aware of the different behaviour provided by different compiler versions. When deploying any smart contract, it is important to have the code reviewed (either by external reviewers, or full audits) to reduce the risk of accidentally introducing any error that could result in significant loss for its users.