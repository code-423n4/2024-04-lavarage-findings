## L-01 - Users can avoid paying borrowing fees

Users might submit an account they control as the `fee_account`. If the intention is to use just one account to gather fees, consider hardcoding the address instead of passing it as an argument:

```rust
**fee_account.try_borrow_mut_lamports()? += fee; 
```

https://github.com/code-423n4/2024-04-lavarage/blob/9e8295b542fb71b2ba9b4693e25619585266d19e/libs/smart-contracts/programs/lavarage/src/processor/swap.rs#L78


## L-02 - Off-by-one LTV check

[Docs](https://lavarage.gitbook.io/lavarage/key-parameters) specify that positions are liquidated when >= 90% LTV, but this is not the case:

```rust
require!(ctx.accounts.position_account.amount * 1000 / position_size  > 900, FlashFillError::ExpectedCollateralNotEnough );
```

Consider using `>=` instead of `>`, or change the docs accordingly.


## L-03 - Missing validations for canonical bump seed

There is a PDA check, but the code is missing a canonical bump seed check. Users should be able to specify the bump as an argument to ensure that it's the correct one:

```diff
  pub fn liquidate(
    ctx: Context<Liquidate>,
    position_size: u64,
+	bump: u64
	) -> Result<()> {
		...
		require!(pda == ctx.accounts.position_account.key() ,
			FlashFillError::AddressMismatch
    	);
		let seeds = &[
		    b"position", 
		    trader_key.as_ref(), 
		    trading_pool_key.as_ref(),
		    random_account_as_id_key.as_ref(),
		    &[bump_seed]
		];
+		require!(bump_seed == bump, FlashFillError::AddressMismatch);
```

https://github.com/code-423n4/2024-04-lavarage/blob/9e8295b542fb71b2ba9b4693e25619585266d19e/libs/smart-contracts/programs/lavarage/src/processor/liquidate.rs#L54-L64

Another similar issue can be found also in the [`borrow_collateral`](https://github.com/code-423n4/2024-04-lavarage/blob/9e8295b542fb71b2ba9b4693e25619585266d19e/libs/smart-contracts/programs/lavarage/src/processor/swapback.rs#L113-L124) function.


## L-04 - It's possible to underflow the borrowed amount

When borrowing, is it possible to specify a `user_pays` higher than `position_size`, as a result, the borrowed amount will underflow to the max value:

```rust
ctx.accounts.position_account.amount = position_size - user_pays;
```
https://github.com/code-423n4/2024-04-lavarage/blob/9e8295b542fb71b2ba9b4693e25619585266d19e/libs/smart-contracts/programs/lavarage/src/processor/swap.rs#L16

The entire `borrow` function will work with an extremely big amount of borrowed funds even if the user pays almost nothing.

However, the function will fail anyway later as the user would have to provide a large amount of collateral anyway in `add_collateral`:

```rust
require!(
      ((amt as u128)
      .checked_mul(ctx.accounts.trading_pool.max_borrow as u128).expect("overflow")
      .checked_div(10_u128.pow(ctx.accounts.mint.decimals as u32))).expect("overflow") as u64
       >= ctx.accounts.position_account.amount, 
      FlashFillError::ExpectedCollateralNotEnough);
```
https://github.com/code-423n4/2024-04-lavarage/blob/9e8295b542fb71b2ba9b4693e25619585266d19e/libs/smart-contracts/programs/lavarage/src/processor/swap.rs#L102

Consider using `checked_sub` anyway to remove any potential issues in the future.


## L-05 - Loans might be unliquidable if SOL value crashes

The liquidation check might overflow to a small value with the first multiplication:

```rust
require!(ctx.accounts.position_account.amount * 1000 / position_size  > 900, FlashFillError::ExpectedCollateralNotEnough );
```

The attacker needs an insane amount of SOL (2+ billion) so this attack is not feasible at the moment of writing, however, it might be in the future if SOL value crashes. If it happens, attackers can have unhealthy, unliquidable loans.

Consider using `saturating_mul` anyway to remove any potential issues in the future.