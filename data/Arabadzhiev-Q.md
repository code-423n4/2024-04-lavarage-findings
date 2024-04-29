# [L-01] No restrictions are enforced on the `borrow` function's context `fee_receipient` parameter, making it possible for borrowers to avoid paying initial borrow fees

## Links to affected code
[swap.rs#L75-L89](https://github.com/code-423n4/2024-04-lavarage/blob/9e8295b542fb71b2ba9b4693e25619585266d19e/libs/smart-contracts/programs/lavarage/src/processor/swap.rs#L75-L89)

## Description
Callers of the `borrow` function can set an address controlled by them as the `fee_receipient` in order to avoid paying the  0.5% borrow fee

## Recommended Mitigation Steps
Implement new logic that either allows the `fee_recepient` to be set for each separate trading pool by its creator, or create a global configuration where the `fee_recepient` is set by a trusted entity

# [L-02] In the `liquidate` function , the node wallet is not verified to be the same as the one of the trading pool

## Links to affected code
[liquidate.rs#L23-L70](https://github.com/code-423n4/2024-04-lavarage/blob/9e8295b542fb71b2ba9b4693e25619585266d19e/libs/smart-contracts/programs/lavarage/src/processor/liquidate.rs#L23-L70)

## Description
The `liquidate` function has no validation that enforces the `node_wallet` address to be equal to the one of the `trading_pool`, making it possible to transfer the liquidated collateral tokens to a wrong node wallet operator.

## Recommended Mitigation Steps
Add validation logic which ensures that the `node_wallet` address is equal to the node wallet of the `trading_pool`

# [L-03] There are unused properties in the `NodeWallet` struct

## Links to affected code
[node_wallet.rs#L12](https://github.com/code-423n4/2024-04-lavarage/blob/9e8295b542fb71b2ba9b4693e25619585266d19e/libs/smart-contracts/programs/lavarage/src/state/node_wallet.rs#L12)

[node_wallet.rs#L15](https://github.com/code-423n4/2024-04-lavarage/blob/9e8295b542fb71b2ba9b4693e25619585266d19e/libs/smart-contracts/programs/lavarage/src/state/node_wallet.rs#L15)

## Description
The `NodeWallet` struct that is used for creating the node wallet PDAs has two unused properties - `maintenance_ltv` and `liquidation_ltv`. This will make the minimum rent exemption SOL amount bigger than it should be.

## Recommended Mitigation Steps
Remove the `maintenance_ltv` and `liquidation_ltv` properties from the `NodeWallet` sturct and update the `space` property of the `#account` attribute in the `CreateNodeWallet` struct to reflect those changes.

# [NC-01] Discrepancies between the code and documentation

## Links to affected code
[swapback.rs#L150](https://github.com/code-423n4/2024-04-lavarage/blob/9e8295b542fb71b2ba9b4693e25619585266d19e/libs/smart-contracts/programs/lavarage/src/processor/swapback.rs#L150)

[liquidate.rs#L27](https://github.com/code-423n4/2024-04-lavarage/blob/9e8295b542fb71b2ba9b4693e25619585266d19e/libs/smart-contracts/programs/lavarage/src/processor/liquidate.rs#L27)

## Description
There are two minor discrepancies between the code and [this part](https://lavarage.gitbook.io/lavarage/key-parameters#platform-parameters) of the protocol documentation. In the docs it is stated that the profit fee is 5%, while in the code it is 2%. Also, in the docs it is stated that the LTV should be greater than or **equal to** 90% in order for a position to be liquidatable, while in the code, it should be greater than 90%.

## Recommended Mitigation Steps
Change the code or the docs in order for them to be aligned with each other

# [NC-02] Basis points can be used for the `interest_rate` value

## Links to affected code
[lending.rs#L15](https://github.com/code-423n4/2024-04-lavarage/blob/9e8295b542fb71b2ba9b4693e25619585266d19e/libs/smart-contracts/programs/lavarage/src/processor/lending.rs#L15)

[lending.rs#L49](https://github.com/code-423n4/2024-04-lavarage/blob/9e8295b542fb71b2ba9b4693e25619585266d19e/libs/smart-contracts/programs/lavarage/src/processor/lending.rs#L49)

## Description
The `interest_rate` value is currently set within the range from 0 to 99. However, the logic for setting and reading it can be modified to utilise basis points, so that it can be set in the range from 0 to 9999. That way, it can be controlled in a more granular manner.

## Recommended Mitigation Steps
Consider using basis points for the `interest_rate` value

# [NC-03] Commented out code

## Links to affected code
[lib.rs#L55](https://github.com/code-423n4/2024-04-lavarage/blob/9e8295b542fb71b2ba9b4693e25619585266d19e/libs/smart-contracts/programs/lavarage/src/lib.rs#L55)

[swapback.rs#L66-L75](https://github.com/code-423n4/2024-04-lavarage/blob/main/libs/smart-contracts/programs/lavarage/src/processor/swapback.rs#L66-L75)

## Description
Commented out code serves no purpose apart from adding confusion to the readers of the code, so it can safely be removed

## Recommended Mitigation Steps
Remove the commented out code

# [NC-04] Comments at the wrong place

## Links to affected code
[position.rs#L4](https://github.com/code-423n4/2024-04-lavarage/blob/9e8295b542fb71b2ba9b4693e25619585266d19e/libs/smart-contracts/programs/lavarage/src/state/position.rs#L4)

## Description
The first comment in the `Position` struct is above its `pool` property, while it should be above the `interest_rate` property, as it is referencing that one

## Recommended Mitigation Steps
Move the comment to the appropriate place

# [NC-05] Inappropriate division error messages

## Links to affected code
[swap.rs#L97](https://github.com/code-423n4/2024-04-lavarage/blob/9e8295b542fb71b2ba9b4693e25619585266d19e/libs/smart-contracts/programs/lavarage/src/processor/swap.rs#L97)

[swap.rs#L101](https://github.com/code-423n4/2024-04-lavarage/blob/9e8295b542fb71b2ba9b4693e25619585266d19e/libs/smart-contracts/programs/lavarage/src/processor/swap.rs#L101)

## Description
Division can overflow for whole numbers, so the `"overflow"` error message at the specified code lines in inappropriate

## Recommended Mitigation Steps
Change the error messages to something more appropriate, such as `"division error"`

# [NC-06] Unused imports

## Links to affected code
[liquidate.rs#L2](https://github.com/code-423n4/2024-04-lavarage/blob/9e8295b542fb71b2ba9b4693e25619585266d19e/libs/smart-contracts/programs/lavarage/src/processor/liquidate.rs#L2)

[swapback.rs#L1](https://github.com/code-423n4/2024-04-lavarage/blob/main/libs/smart-contracts/programs/lavarage/src/processor/swapback.rs#L1)

## Description
There are a lot of unused imports through the code, which can be simply removed. The linked lines of code are just for reference, but they are not all instances of such imports through the code.

## Recommended Mitigation Steps
Remove the unused imports

# [NC-07] Code formatting issues

## Links to affected code
[lending.rs#L14](https://github.com/code-423n4/2024-04-lavarage/blob/9e8295b542fb71b2ba9b4693e25619585266d19e/libs/smart-contracts/programs/lavarage/src/processor/lending.rs#L14)

[lending.rs#L77-L81](https://github.com/code-423n4/2024-04-lavarage/blob/9e8295b542fb71b2ba9b4693e25619585266d19e/libs/smart-contracts/programs/lavarage/src/processor/lending.rs#L77-L81)

## Description
There are formatting/indentation issues in some parts of the code, which impact it's readability negatively

## Recommended Mitigation Steps
Consider using a code formatter such as Prettier, so that a consistent and easy to read code style can be enforced on the whole code base