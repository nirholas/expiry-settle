---
name: use-expiry-settle
description: Build, test, integrate and deploy the ExpirySettleHook Uniswap v4 hook from the expiry-settle repo. Use when you need to work with this hook's contract, wire it into a pool, or deploy it with a mined CREATE2 salt.
---

# ExpirySettleHook

Gives a pool a maturity, and makes moving its price monotonically more expensive as that maturity approaches, so the settlement price is dearest to manipulate exactly when manipulating it would pay most.

Family: Time. Tags: expiry, settlement, dynamic-fee, derivatives, oracle-free. License: Apache-2.0. Source: `src/hooks/ExpirySettleHook.sol`. The machine-readable manifest is `hook.json` (ABI, errors, permissions, properties).

## Build and test

Foundry project (`foundry.toml`, solc 0.8.26, evm `cancun`). From a clone of the repo:

```bash
git clone https://github.com/nirholas/expiry-settle && cd expiry-settle
git submodule update --init --recursive   # dependencies live in lib/
forge build
forge test
```

Tests are in `test/ExpirySettleHook.t.sol`. Use `FOUNDRY_PROFILE=ci forge test` for the larger fuzz run (5000 runs).

## Hook permissions

Uniswap v4 reads a hook's permissions from the low bits of its address. This hook claims: `beforeSwap`, `beforeAddLiquidity`, `afterInitialize`. It is not upgradeable and has no custom swap data requirement (`hook.json` `properties`).

## Configuring a pool

The hook holds per-pool terms that are fixed for the pool's lifetime. Configure the key first, then initialize the pool:

```solidity
hook.configure(key, ExpirySettleHook.Config({ /* maturity (uint64), windowSeconds (uint32), baseFee (uint24), settlementFee (uint24) */ }));
poolManager.initialize(key, startingSqrtPriceX96);
```

Signature: `configure((address,address,uint24,int24,address),(uint64,uint32,uint24,uint24))`. A pool the hook was never configured for cannot be created (`PoolNotConfigured`). Anyone can configure a key that has no pool yet, so if someone front-runs your terms, pick a different `tickSpacing`, which is a different pool id.

## Deploying

A v4 hook must be deployed at an address whose bits match its permissions, so `script/Deploy.s.sol` mines a CREATE2 salt first. The dry run sends nothing:

```bash
export RPC_URL=https://...        # a chain with a Uniswap v4 PoolManager
forge script script/Deploy.s.sol --rpc-url $RPC_URL
```

Broadcasting spends gas from the signer (`PRIVATE_KEY`, or `--account` / `--ledger`) and is irreversible, so show the user the chain, the predicted hook address and the signer, and get an explicit yes before running:

```bash
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Re-running is safe: a hook already at its mined address is reported and skipped. If the miner fails with `EvmError: OutOfGas`, add `--gas-limit 30000000000`. `--verify` needs `ETHERSCAN_API_KEY`. The chain must be in the address book `src/libraries/Chains.sol`. `script/Deploy.s.sol` fails with "no Uniswap v4 PoolManager known for this chain" otherwise.

## Identifying a deployed hook

```solidity
IHookMetadata(hookAddress).hookName();
IHookMetadata(hookAddress).specURI();   // points at hook.json
```

## Errors you may meet

`FeeTooLarge(uint24)`, `InvalidPool()`, `InvalidWindow()`, `Matured(uint64)`, `MaturityTooSoon()`, `NotDynamicFee()`, `NotYetMatured(uint64)`, `PoolAlreadyInitialized()`, `PoolNotConfigured()`, `SettlementFeeBelowBase()`.

## Limitation

It raises the cost of manipulation, it does not prevent it. A manipulator whose payoff exceeds the ramped fee will still pay it, and the right response is to size `settlementFee` against the notional settling on the price rather than against ordinary trading. The hook also cannot know what the pool settles for, so if nothing actually references `settlementPrice`, the ramp is pure cost with no benefit.

More: `docs/integrating.md`, `docs/deploying.md`, and https://expiry-settle.pages.dev.
