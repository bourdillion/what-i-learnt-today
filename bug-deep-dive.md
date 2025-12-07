## Web3 Security Bug Deep Dive #3 - CapMoney Fee Gaming
**What I learnt today - 07/12/2025**

CapMoney has a fee system that discourage depositors from depositing large amount into a particular asset, so as to maintain balanced asset allocations across the protocol. The idea is simple: if one asset becomes too dominant in the pool (say USDT goes from 30% to 60% of total value), that does create a risk. So in other to mitigate it, CapMoney introduced a dynamic fee curve, where fee increases as your deposit reaches the asset allocation level, just like the way we calculate income taxes.

Think of it like supply and demand. When an asset's allocation is low (say 20% of the pool), deposits are cheap, the protocol wants more of that asset. When an asset's allocation is high (say 80% of the pool), deposits get expensive, the protocol forces you to deposit that money in another asset.

The problem now is that, users can break their large deposit into multiple smaller one and pay less fees (especially if gas is negligible). Math nerds know that when a solution is non-linear, f(a) + f(b) ≠ f(a+b). When an investor mints, it calculate the amountOutBeforeFees, and a new ratio that is used to calculate the fee based on the amount the investor, as you can see in the code below;

```javascript
function _amountOutBeforeFee(address _oracle, IMinter.AmountOutParams memory params)
    internal
    view
    returns (uint256 amount, uint256 newRatio)
{
    //...other parts of the function...

    if (params.mint) {
        assetValue = params.amount * assetPrice / assetDecimalsPow;
        if (capSupply == 0) {
            newRatio = 0;
            amount = assetValue * capDecimalsPow / assetPrice;
        } else {
            // The new ratio is calculated here
@>          newRatio = (allocationValue + assetValue) * RAY_PRECISION / (capValue + assetValue);
            amount = assetValue * capDecimalsPow / capPrice;
        }
    }

    //...other parts of the function...
}

```

We get the new ratio by doing (allocationValue + assetValue) / (capValue + assetValue), where assetValue is the amount you are depositing. This new ratio is then used in this _applyFeeSlopes function to calculate the fee, and in the if-else part, we observe how the code balances out the fees. Here we already see the problem that each deposit calculates its fee independently based on the ratio at that moment.

```javascript
function _applyFeeSlopes(IMinter.FeeData memory fees, IMinter.FeeSlopeParams memory params)
    internal
    pure
    returns (uint256 amount, uint256 fee)
{
    uint256 rate;
    if (params.mint) {
        rate = fees.minMintFee;
        if (params.ratio > fees.optimalRatio) {
            if (params.ratio > fees.mintKinkRatio) {
                // High fee zone - steep slope
                uint256 excessRatio = params.ratio - fees.mintKinkRatio;
@>              rate += fees.slope0 + (fees.slope1 * excessRatio / (RAY_PRECISION - fees.mintKinkRatio));
            } else {
                // Medium fee zone - gradual slope
@>              rate += fees.slope0 * (params.ratio - fees.optimalRatio) / (fees.mintKinkRatio - fees.optimalRatio);
            }
        }
    }
    //...other parts of the function...
}

```

**Why does it work that way?**
If you want to deposit $10,000. The current allocation is at 50% and your deposit would push it to 52.38%.

If you deposit the entire $10,000 once, the protocol calculate the fee based on jumping from 50% → 52.38% in one go. Let's say that triggers a 0.785% fee = $78.50.

Now if you deposit 20 different ones instead of depositing all once, we see that;

First deposit: 50.00% → 50.12%

Second deposit: 50.12% → 50.24%

Third deposit: 50.24% → 50.37%

...for 20 more deposits

Final state: 52.38% (same end result)

Each small deposit stays in a lower part of the fee curve. Your average fee rate is lower because you're "climbing the curve" gradually instead of jumping once to the steep part.

Total fee with splitting will be approximately $66.40 (if you solved that with me). That means you saved $12.10 by depositing multiple times again assuming gas is negligible. Even if it is not for example, like ethereum mainnet, if the capital is huge, you will save way more than gas can take.

Below is the proof of contest of the actual code from the whitehat who found this bug;

```javascript
function test_audit_mint() public {
    _initTestVaultLiquidity(usdVault);

    vm.startPrank(user);
    usdt.mint(user, 10000e6);
    usdt.approve(address(cUSD), 10000e6);

    // Single mint
    uint256 amountIn = 10000e6;
    uint256 minAmountOut = 9500e18;
    uint256 deadline = block.timestamp + 1 hours;

    uint256 balance_before = cUSD.balanceOf(user);
    cUSD.mint(address(usdt), amountIn, minAmountOut, user, deadline);
    uint256 balance_after = cUSD.balanceOf(user);
    console.log("total", balance_after - balance_before);

    vm.revertTo(beforeState);

    // Split into 20 mints
    balance_before = cUSD.balanceOf(user);
    uint256 nSplit = 20;

    for (uint256 i = 0; i < nSplit; i++) {
        cUSD.mint(address(usdt), amountIn / nSplit, minAmountOut / nSplit, user, deadline);
    }
    balance_after = cUSD.balanceOf(user);
    console.log("split", balance_after - balance_before);
}

// Output:
// total: 9921488294314381270904
// split: 9933593396378017068848

```

The user receives 12.1 more tokens when splitting exactly how we got it above. This bug breaks the protocol intended fee structure, as more technical user can pay way less fees than regular users, also they reduce the protocol revenue, and can make assets become imbalance.

One way to fix the bug is by enforcing a minimum amount check in the mint function, which obviously can still be bypassed. Another way is to introduce time-lock cooldown in the mint function, which can slow down depositing multiple time.

```javascript
function mint(...) external {
    require(amountIn >= MIN_MINT_AMOUNT, "Deposit too small");
    // rest of function
}

```

```javascript
mapping(address => uint256) public lastMintTime;

function mint(...) external {
    require(block.timestamp - lastMintTime[msg.sender] >= COOLDOWN_PERIOD, "Too soon");
    lastMintTime[msg.sender] = block.timestamp;
    // rest of function
}
```

**Reflections**
This bug isn't a coding error, the math is correct, everything works fine as intended. The protocol fee curve just assume that the user will deposit one time, which is flawed.

When auditing similar protocols always ask: "Can users game this by splitting operations?" Progressive curves, rate limits, cooldowns are all vulnerable to this pattern of bug.

Link to report: https://github.com/sherlock-audit/2025-07-cap-judging/issues/20
```
