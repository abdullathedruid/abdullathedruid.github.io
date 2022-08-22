---
Title: On the topic of gas optimisation
Date: 2022-02-03 
Category: Smart Contracts
Tags: ethereum, vyper, gas
---

## Introduction

A few topics recently crossed my attention. A user on reddit lost 500K by sending WETH to the WETH contract address

[Reddit: Did I just lose half a million dollars by sending WETH to WETH's contract address?](https://www.reddit.com/r/ethereum/comments/sfz4kw/did_i_just_lose_half_a_million_dollars_by_sending)

There has been some discussion recently on gas optimisations in smart contracts, which has been particularly relevant due to increasing gas prices.

Should a smart contract prevent users from performing actions that would result in the loss of user funds?

Adding extra checks (such as ensuring that the `sender` and `receiver` are not the same as the contract address) adds an additional gas stipend onto each transaction.  
An argument could be made that it should be the responsibility of wallets and front-ends to prevent these transactions from ever reaching the blockchain. However, given that it is fairly common for tokens to be sent to the contract address, we know that this is not the case.

I thought it would be a good idea to look into this further to decide for myself whether it made sense to add these checks from an economic perspective.

### Idea

I would write two smart contracts representing ERC20 tokens: an 'unsafe' ERC20 which allows any transfer (ofcourse, the standard rules of balances and approvals will remain) and a 'safe' ERC20 which checks that the receiver and sender are valid addresses (by ensuring that they are not the zero address, or the contract address)

From these contracts, I will get an estimate of the average gas cost for each function. 

I will then try and use these figures to get an idea of the gas saving that would have occurred during the lifespan of the WETH contract. I will also get an idea of how much WETH has been 'lost' after being sent to the WETH contract and compare these figures


### Data

All contracts/scripts are available on [Github](https://github.com/abdullathedruid/gas-cost-erc20).

The gas costs for each of the functions is displayed below, I will be using the average for my calculations. The average has been calculated from 1000 executions with random variables

```python
safeERC20 <Contract>
   ├─ approve      -  avg:  42565  avg (confirmed):  42565  low:  24521  high:  44117
   ├─ transfer     -  avg:  31945  avg (confirmed):  31945  low:  27012  high:  35808
   └─ transferFrom -  avg:  20442  avg (confirmed):  20442  low:  18819  high:  29250
unsafeERC20 <Contract>
   ├─ approve      -  avg:  42519  avg (confirmed):  42519  low:  24475  high:  44071
   ├─ transfer     -  avg:  31875  avg (confirmed):  31875  low:  26942  high:  35738
   └─ transferFrom -  avg:  20400  avg (confirmed):  20400  low:  18782  high:  29176
```

In order to get a list of transactions on the WETH contract, I initially performed the following query:

```SQL
SELECT
    input, gas_price, gas
FROM `bigquery-public-data.crypto_ethereum.transactions` as tx
WHERE
  tx.to_address = '0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2'
```

I then realised that this misses out on the majority of transactions on the WETH contract given that many contracts will sub-call the WETH contract, therefore the `to` address on the transaction will not be the WETH address.

My new updated query looks like:
```SQL
WITH traces AS (
SELECT tr.transaction_hash, tr.gas_used, SUBSTRING(tr.input, 0, 10) as sig_hash
FROM `bigquery-public-data.crypto_ethereum.traces` tr --tracks internal transactions
WHERE tr.to_address = '0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2' -- WETH
AND tr.status = 1
AND tr.call_type = 'call'
),

txs AS (
SELECT tr.sig_hash, (tx.gas_price)/1e9 as gwei
FROM `bigquery-public-data.crypto_ethereum.transactions` as tx
JOIN traces tr 
ON tr.transaction_hash = tx.hash
)

SELECT sig_hash, SUM(gwei) as gwei, COUNT(gwei) FROM txs
GROUP BY sig_hash
```
 > I didn't think that this query would also return the calls for `totalSupply()` `decimals()`, etc. In future queries, I'll make sure to filter these out!

The script `get_gas_saving.py` iterates through the functions above and sums the gas saved for each function. The output is as follows:

```python
Running 'scripts/get_gas_saving.py::main'...
970677015.604 Gwei/gas spent on performing transferFrom(). Each function saves 42 gas
Total ETH saved by transferFrom(): 40.768
--
8570367257.755 Gwei/gas spent on performing transfer(). Each function saves 70 gas
Total ETH saved by transfer(): 599.926
--
311808609.989 Gwei/gas spent on performing approve(). Each function saves 46 gas
Total ETH saved by approve(): 14.343
--
Total ETH saving across all functions: 655.037 
Total number of transactions: 91959066
```

## Discussion

A cursory glance at the etherscan page shows that the WETH contract has 432.162 WETH lost within the contract, which _is_ lower than the 655 ETH saved in gas fees.  
The conclusion that I would draw here is that users of a smart contract would end up overpaying for the additional sanity checks discussed above.

It may be interesting to perform the same query looking at all ERC20 tokens, rather than just WETH to see whether the same conclusion is valid across all ERC20s or just WETH.

However, another argument could be made that it doesn't really make too much of a difference given the sheer number of transactions that have taken place. The difference of 223 ETH, when amortised over 92 million transactions is a value of 0.0000024 ETH per transaction.  

After I published this post, I spoke with a few other people. One interesting perspective was that the additional sanity checks act as a form of socialised insurance. I personally do not mind paying fractionally more on a transaction, knowing that it might prevent somebody else from losing their funds.