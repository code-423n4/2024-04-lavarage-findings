## Impact
`accrued_interest()` has a signer, so only the trader can see the accrued interest. Anchor will validate the signature against the trader key, so this can be called only by the position owner.
Even if it is designed this way that, only the trader can see their `accrued_interest`, it is not practical, due to the signature requirement.

In Solana, programs are used to change state, not to get result. to get result, just read direclty from the client side.


## Proof of Concept

- `accrued_interest` function uses **RepaySOL** as a context.

[swapback.rs#L216](https://github.com/code-423n4/2024-04-lavarage/blob/main/libs/smart-contracts/programs/lavarage/src/processor/swapback.rs#L216)

- **RepaySOL** context has the trader as a signer

[repay_sol.rs#L12](https://github.com/code-423n4/2024-04-lavarage/blob/main/libs/smart-contracts/programs/lavarage/src/context/repay_sol.rs#L12)



## Tools Used
Manual analysis

## Recommended Mitigation Steps

Read the account data direclty from the client side. 
