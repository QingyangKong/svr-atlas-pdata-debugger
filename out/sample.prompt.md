# Atlas pData simSolverCall Replay — `pdata0505`

I'm debugging an Atlas Protocol pData simulation on **Arbitrum (42161)**. Use the context below to investigate the trace at `out/pdata0505.trace.log`.


## Atlas Protocol overview

Atlas is an EVM smart-account / MEV-OEV auction infrastructure where searchers
bid for the right to bundle a `solverOp` next to a user's `userOp`. The
aggregate bundle is called a **metacall** and is simulated via
`Simulator.simSolverCall(bytes pData)`, which decodes pData into:

```solidity
(bool success, uint8 simResult, uint256 outcome) = simulator.call(pData);
```

For **Chainlink SVR (Secondary Value Recovery) feeds**, the userOp.data is a
nested call chain:

```
update(wrapper, bytes) -> forward(baseFeed, bytes) -> transmit(...)
```

This updates the `ChainlinkAtlasWrapper`'s OEV-captured price. The solver's
job is to capture OEV (liquidate, arbitrage, etc.) within the same atomic
metacall and pay a bid back to the bundler.

### Result code reference

`simResult` enum (`uint8`):

| value | name | meaning |
|---|---|---|
| 0 | `Unknown` | Default / uninitialized. |
| 1 | `VerificationSimFail` | Atlas-level validation rejected the bundle (signatures, deadline, dapp config, etc.). The `outcome` value here is a `ValidCallsResult` enum, not a bit set. |
| 2 | `PreOpsSimFail` | DApp `preOps` hook reverted. |
| 3 | `UserOpSimFail` | UserOp execution reverted. For SVR feeds this typically means the oracle update reverted (e.g. `StaleReport`, `OracleUpdateFailed`). |
| 4 | `SolverSimFail` | One of the `outcome` bits below was tripped. |
| 5 | `AllocateValueSimFail` | DApp `allocateValue` hook reverted. |
| 6 | `SimulationPassed` | The bundle would have succeeded on-chain. |

`outcome` bit flags (`uint256`, **only meaningful when `simResult == 4`**):

| bit | name | meaning |
|---|---|---|
| 8 | `InsufficientEscrow` | Solver hasn't bonded enough atlETH to cover its gas liability. |
| 12 | `PerBlockLimit` | Solver already executed within this block. |
| 16 | `PreSolverFailed` | DApp's `preSolverCall` hook reverted. |
| 17 | `SolverOpReverted` | The solver contract's own logic reverted. |
| 18 | `PostSolverFailed` | DApp's `postSolverCall` hook reverted. |
| 19 | `BidNotPaid` | Solver did not transfer the bid to the bundler. |
| 20 | `InvertedBidExceedsCeiling` | (Inverted-bid auctions only.) |
| 21 | `BalanceNotReconciled` | Solver took funds from atlas but did not repay. |
| 22 | `CallbackNotCalled` | Solver did not call `atlas.reconcile(...)`. |
| 23 | `EVMError` | Out-of-gas or non-revert EVM error. |

## Where to find Atlas source code

- **FastLane Labs Atlas monorepo**: https://github.com/FastLane-Labs/atlas
  - `src/contracts/atlas/Atlas.sol` — `metacall(...)` entrypoint
  - `src/contracts/atlas/AtlasVerification.sol` — verification logic (governs `simResult==1`)
  - `src/contracts/atlas/Escrow.sol` — `execute`, `_solverOpWrapper`, bid handling, escrow accounting (governs `simResult==4` outcomes)
  - `src/contracts/types/SolverOperation.sol` — `SolverOutcome` bit enum (the `outcome` value in this trace)
  - `src/contracts/types/ConfigTypes.sol` — `CallConfig` flags
  - `src/contracts/dapp/DAppControl.sol` — base for DAppControl modules
  - `src/contracts/helpers/Simulator.sol` — `simSolverCall(...)`
- **Atlas docs**: https://atlas-docs.fastlane.xyz/ (concepts: solver, dapp, bundler, escrow)
- **Chainlink SVR feed onboarding**:
  https://docs.chain.link/data-feeds/svr-feeds/searcher-onboarding-atlas

## Chain configuration: Arbitrum (42161)

| Field | Value |
|---|---|
| Atlas | [`0x8ad1aE9D97C79aA68A0a151E83ff3942f68F86C1`](https://arbiscan.io/address/0x8ad1aE9D97C79aA68A0a151E83ff3942f68F86C1) |
| Simulator | [`0x57FA2aBf1dc109C5F7ea2FB6A72358D2c624971d`](https://arbiscan.io/address/0x57FA2aBf1dc109C5F7ea2FB6A72358D2c624971d) |
| Wrapped native | WETH [`0x82aF49447D8a07e3bd95BD0d56f35241523fBab1`](https://arbiscan.io/address/0x82aF49447D8a07e3bd95BD0d56f35241523fBab1) |
| Native gas symbol | ETH |
| Block explorer | https://arbiscan.io |
| Block time | ~0.25s |

**Chain-specific quirks:**

- On Arbitrum, `block.number` inside Foundry forks returns the L1 block, while `arbBlockNumber()` (precompile `0x64`) returns the L2 block. The replay test mocks `arbBlockNumber()` to the L2 fork block so per-block-limit and timing checks behave correctly.
- `getPricesInArbGas()` (precompile `0x6C`) is mocked to ~zero L1 data fees so calldata-cost arithmetic in the gas calculator does not blow up at the fork point.

## This run's pData

| Field | Value |
|---|---|
| Source pData | `pdata0505.txt` |
| UserEOA | [`0xb6065f79d99f29c3eda0ed1bda7ff88e7ee12f1e`](https://arbiscan.io/address/0xb6065f79d99f29c3eda0ed1bda7ff88e7ee12f1e) |
| DAppControl (== DApp) | [`0xe15bba987c002ecc3586e81244517877d294d291`](https://arbiscan.io/address/0xe15bba987c002ecc3586e81244517877d294d291) |
| UserOp deadline (block) | 459535529 |
| UserOp gas / maxFeePerGas | 500000 / 30039000 (0.0300 Gwei) |
| CallConfig | `41732` |
| Solver contract | [`0x0e0d47c29cba6dcdbb345bd33e926e6776e4c9ca`](https://arbiscan.io/address/0x0e0d47c29cba6dcdbb345bd33e926e6776e4c9ca) |
| SolverEOA | [`0x00003f87cef82f2a4120118a962d956eccfb3cfd`](https://arbiscan.io/address/0x00003f87cef82f2a4120118a962d956eccfb3cfd) |
| Bid token | native ETH (`bidToken=0x0`) |
| Solver bid | 0.0826807959 ETH (82680795878240838 wei) |
| userOpHash | `0x170e42536a80577979f7339d843a39715950f5f0c0dc63a521f3b04d2c1139e4` |
| Bundler | [`0x9d8a4c00835bfb7bd967c91959a9d21603375140`](https://arbiscan.io/address/0x9d8a4c00835bfb7bd967c91959a9d21603375140) |
| ChainlinkAtlasWrapper | [`0x9cd5b3e0777b3c85803e6c54c48f905315b9bbe6`](https://arbiscan.io/address/0x9cd5b3e0777b3c85803e6c54c48f905315b9bbe6) |
| Base Chainlink feed | [`0xe7c522c60ba7f1b5e398d2312593713e2b19aeb0`](https://arbiscan.io/address/0xe7c522c60ba7f1b5e398d2312593713e2b19aeb0) — **BTC / USD** |
| Oracle median (raw int192) | `8117564124522` |
| Oracle observation time | `1777958705` (2026-05-05 05:25:05 UTC) |
| Oracle epoch.round | `1777958705` |
| # observations / # signatures | 10 / 4 |
| Fork block used by replay | 459535383 (oracle block 459535384 - 1 (no landing tx found)) |

## Forge replay result (`forge test -vvvv`)

- forge: **FAIL** in `test_replay()` — vm.mockCall: failed to get account for 0x000000000000000000000000000000000000006C: HTTP error 500 with body: {"id":20,"jsonrpc":"2.0","error":{"message":"Temporary internal error. Please retry, trace-id: d13566efdd917dc22e5249387856f00d","code":19}}
- ⚠️  The forge run looks like it hit a transient RPC issue (HTTP 5xx / missing trie node / rate limit). The simulation result below may be unreliable; consider retrying or using a more stable archive RPC.

Full trace: see `out/pdata0505.trace.log` (addresses pre-labeled with `vm.label` so they appear as `[Atlas]`, `[Solver]`, `[ChainlinkAtlasWrapper]`, `[<pair> Feed]`, etc.).

## What I want you to do

1. **Find the deepest revert.** Walk down the call tree to the leaf call that
   actually reverted, and decode its return data:
     - `0x08c379a0` prefix → `Error(string)`, decode the string.
     - `0x4e487b71` prefix → `Panic(uint256)`, decode the panic code.
     - other 4-byte prefix → custom error; identify the selector and look it
       up if you can, or list the candidate function/error signatures.
2. **Explain why** in plain language, in terms of THIS solver's intent
   (capture OEV from the price update at this block) — not in terms of "the
   call failed".
3. **Categorize the failure** as one of:
     - **Timing**: oracle not yet updated at this block, deadline passed,
       solver competing with another tx in the same block.
     - **Logic**: balance check / slippage / liquidation HF threshold / wrong
       swap path / underpriced bid.
     - **Config**: wrong DApp selector allowlisted, wrong solver registered,
       wrong CallConfig flags.
4. **Concrete next steps**: list the SHORTEST set of follow-ups that would
   confirm the diagnosis — e.g. "retry at block X", "show source of contract
   Y", "check token Z balance at block N", "decode selector 0xabcd1234".

If anything below is unclear or missing, list it explicitly rather than
guessing.
