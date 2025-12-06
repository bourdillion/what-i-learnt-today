## Web3 Security Bug Deep Dive #2 - CapMoney's Zero Interest Rate Bug

**What I learnt today - 06/12/2025**

CapMoney has interest rate models for their lending vaults. The interest rate is supposed to adjust based on utilization, if 90% of the vault is borrowed, rates should be high to discourage more borrowing and incentivize repayment. If only 20% is borrowed, rates should be low to encourage borrowing.

The interest rate calculation uses a multiplier that adjusts over time as people borrow. This multiplier is supposed to increase when utilization is high and decrease when it's low. But there's a problem with how this multiplier gets initialized.

When a vault first launches, the multiplier starts at 0. Now if the vault immediately hits high utilization (above the "kink" threshold, around 80%), the interest rate calculation becomes 0, because the multiplier is 0 as you can see below. So even if the utilization rate is at 95%, interest rate will still be zero

```javascript
  interestRate = (baseRate + slopeRate * excess) * multiplier / 1e27
```
Now during borrowing, the _applySlope function below tries to adjust the multiplier when the utilization is high, but unfortunately only does it when the multiplier is above the maxMultiplier, and if the multiplier is still zero, then that is outright impossible.

```jascript
  if (_utilization > slopes.kink) {
      uint256 excess = _utilization - slopes.kink;
      utilizationData.multiplier = utilizationData.multiplier
          * (1e27 + (1e27 * excess / (1e27 - slopes.kink)) * (_elapsed * $.rate / 1e27)) / 1e27;
  
      if (utilizationData.multiplier > $.maxMultiplier) {
          utilizationData.multiplier = $.maxMultiplier;
      }

    interestRate = (slopes.slope0 + (slopes.slope1 * excess / 1e27)) * utilizationData.multiplier / 1e27;
}
```

The else part of the if-else statement also tries to adjust the multiplier, which actually does saves it, and adjust the multiplier correctly to the `minimumMultiplier`, exactly how the developer must have assume it to work. But one question the developers forgot to ask themselves is, “what if the utilization ratio starts above the kink”?

```javascript
    } else {
        utilizationData.multiplier = utilizationData.multiplier * 1e27
            / (1e27 + (1e27 * (slopes.kink - _utilization) / slopes.kink) * (_elapsed * $.rate / 1e27));
    
        if (utilizationData.multiplier < $.minMultiplier) {
            utilizationData.multiplier = $.minMultiplier; // <-- This saves it
        }
```

**Attack scenario:**

1. Vault launches with 10,000 USDC deposited

2. Alice borrows 9,000 USDC (90% utilization, above the kink)

3. Multiplier is 0, so interest rate = 0%

4. Alice keep the loan at 0% interest fee

5. Bob sees the opportunity and borrow the remaining money. The vault goes Kaboom!

The early borrowers basically get free money while lenders earn nothing despite their capital being fully utilized.

**The fix:**

The best way to fix it, as suggested by the whitehat is to add the same minimum check in the “else” part of the if-else statement to the “if” part too as well, see below;

```javascript
if (_utilization > slopes.kink) {

    if (utilizationData.multiplier > $.maxMultiplier) {
        utilizationData.multiplier = $.maxMultiplier;
    }
    if (utilizationData.multiplier < $.minMultiplier) { // <-- Add this
        utilizationData.multiplier = $.minMultiplier;
    }

    interestRate = (slopes.slope0 + (slopes.slope1 * excess / 1e27)) * utilizationData.multiplier / 1e27;
}
```

**Reflections**

Always think about initial state. A lot of bugs happen at boundaries, like initializations. The devs assumed the multiplier would naturally be non-zero above the kink because "it only increases there", but they forgot that it has to start somewhere. Zero is a dangerous number in multiplication, once you're stuck at zero, you can't multiply your way out.

Link to the report: https://github.com/sherlock-audit/2025-07-cap-judging/issues/139
