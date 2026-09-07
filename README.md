# ExpirySettle

**Gives a pool a maturity, and makes moving its price monotonically more expensive as that maturity approaches, so the settlement price is dearest to manipulate exactly when manipulating it would pay most.**

A production Uniswap v4 hook. It prices every swap by overriding the pool's LP fee, so the value it captures is paid to in-range liquidity and never to the hook. No owner, no pause switch, no upgrade path.

- **Site:** https://expiry-settle.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/ExpirySettleHook.sol`](src/hooks/ExpirySettleHook.sol)
- **Licence:** Apache-2.0

## How it works

Any dated instrument that settles against a market price has the same problem at the end of its life. The payoff of pushing the price around is largest in the final minutes, because there is no time left for anybody to push it back, and the cost of pushing it is unchanged from any other moment. Traditional markets answer this with a settlement window: the official price is an average over the closing period rather than a single print, which makes a manipulator pay for the whole window instead of one instant.

A pool cannot average its own price without an oracle, but it can do something a traditional venue cannot: change what moving the price costs. As maturity approaches, this hook ramps the fee from `baseFee` up to `settlementFee` across the last `windowSeconds`, on a curve that is quadratic rather than linear so the final moments are much more expensive than the early part of the window: fee(t) = baseFee + (settlementFee - baseFee) * elapsed^2 / windowSeconds^2 A manipulator who wants to move the settlement print has to choose between acting early, where the fee is low but there is time for somebody to trade against them, and acting late, where nobody can respond but every basis point of the move costs several times more. The fee is paid to the liquidity that has to absorb the move, which is the party bearing the cost.

At maturity the pool stops trading. Swaps revert, so the price cannot move again, and `settlementPrice` is simply the pool's final tick. Liquidity may always be removed, including after maturity, because a matured pool that cannot be exited is a trap rather than an instrument.

Adding liquidity after maturity reverts: there is nothing left to provide liquidity for, and permitting it would only let somebody strand funds. The hook holds nothing, takes nothing for itself, and has no privileged role. The maturity is fixed before the pool exists and cannot be moved by anyone, which is the property that makes the instrument datable at all.

## Prior art

Dated AMMs exist (YieldSpace and Pendle-style curves converge to par at maturity), and hooks that halt trading on a schedule exist. Settlement-window design is standard in traditional derivatives. Making the *cost* of moving an AMM's price rise on a convex curve into its own settlement, as the on-chain substitute for a time-averaged settlement price, is the contribution here.

## Where it does not help

It raises the cost of manipulation, it does not prevent it. A manipulator whose payoff exceeds the ramped fee will still pay it, and the right response is to size `settlementFee` against the notional settling on the price rather than against ordinary trading. The hook also cannot know what the pool settles for, so if nothing actually references `settlementPrice`, the ramp is pure cost with no benefit.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
hook.configure(
    key,
    ExpirySettleHook.Config({
        maturity: /* uint64 */ 0,
        windowSeconds: /* uint32 */ 0,
        baseFee: /* uint24 */ 0,
        settlementFee: /* uint24 */ 0
    })
);

poolManager.initialize(key, startingSqrtPriceX96);
```

The pool's `fee` field must be `LPFeeLibrary.DYNAMIC_FEE_FLAG`. The hook rejects a pool initialized without it.

### Parameters

| Parameter | Type | Units |
| --- | --- | --- |
| `maturity` | `uint64` | unix seconds |
| `windowSeconds` | `uint32` | seconds |
| `baseFee` | `uint24` | hundredths of a bip (`3000` = 0.30%) |
| `settlementFee` | `uint24` | hundredths of a bip (`3000` = 0.30%) |

## What it reverts with

| Error | Meaning |
| --- | --- |
| `FeeTooLarge(uint24)` | A fee was configured above the protocol maximum of 100%. |
| `InvalidWindow()` | `windowSeconds` was zero, which would turn the ramp into a cliff at maturity. |
| `Matured(uint64)` | The pool has matured. Its price is final and it no longer trades. |
| `MaturityTooSoon()` | The maturity must be far enough ahead to contain the whole settlement window. |
| `NotDynamicFee()` | The hook was attempted to be initialized with a non-dynamic fee. |
| `NotYetMatured(uint64)` | The pool has not matured, so it has no settlement price yet. |
| `PoolAlreadyInitialized()` | The pool already exists, so its configuration is final. |
| `PoolNotConfigured()` | The pool was initialized without a configuration for this hook. |
| `SettlementFeeBelowBase()` | `settlementFee` must be at least `baseFee`; a ramp that gets cheaper into settlement inverts the point. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 3 of the fourteen:

- `afterInitialize`
- `beforeAddLiquidity`
- `beforeSwap`

Mask: `0x1880`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # ExpirySettle
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # expiry, settlement, dynamic-fee, derivatives, oracle-free
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/expiry-settle
cd expiry-settle
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.
