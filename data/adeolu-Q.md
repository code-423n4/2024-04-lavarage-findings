# [L-01] - Possible to open many little borrow positions just to aviod paying fees. 

## Impact
because fee calc returns 0 when `borrow_amount * 5 ` is lower than 1000, it is possible a malicious user who doesnt want to pay protocol fees opens many small borrow positions to avoid paying this fee. 

The fee calc logic is shown below 

https://github.com/code-423n4/2024-04-lavarage/blob/9e8295b542fb71b2ba9b4693e25619585266d19e/libs/smart-contracts/programs/lavarage/src/processor/swap.rs#L76
```
   let fee = borrow_amount * 5 / 1000;
```

if borrow_amount for one position is `199`, this will mean fee will be `199 * 5 / 1000 = 995 /1000 = 0 `

user decides to make 2000 borrow positions, each borrowing the small amount 199. This means user has sucessfully borrowed a value of `398000` via the protocol without paying any fees. 

if it intended for strict fees payment for every position opened then the borrow function should revert if fee calc returns 0.  

## Recommended Mitigation Steps
add require statemwent that revert if fee calc returns 0.  