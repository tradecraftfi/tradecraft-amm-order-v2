# Tradecraft V7 Order DAR

**Swapping and providing liquidity on Tradecraft V7 pools.**

A developer's reference for wallet providers and other partners that send orders to Tradecraft V7 pools. This guide is version 1.0.0 and is for the `tradecraft-amm-order-v2` package, version 2.1.0. For questions, contact us by email at [info@tradecraft.fi](mailto:info@tradecraft.fi).

---

## Contents

1. [Overview](#1-overview)
2. [The package](#2-the-package)
3. [Network reference](#3-network-reference)
4. [Discover pools](#4-discover-pools)
5. [Create orders](#5-create-orders)
6. [Swap orders](#6-swap-orders)
7. [Immediate mode: funding from the wallet](#7-immediate-mode-funding-from-the-wallet)
8. [The order lifecycle](#8-the-order-lifecycle)
9. [Liquidity orders](#9-liquidity-orders)
10. [Standing allocations](#10-standing-allocations)
11. [Funding from standing allocations](#11-funding-from-standing-allocations)
12. [Withdraw allocation funding](#12-withdraw-allocation-funding)
13. [Monitor orders](#13-monitor-orders)
14. [Type reference](#14-type-reference)

---

## 1. Overview

A V7 pool is a concentrated-liquidity pool on the Canton Network, written in [Daml](https://github.com/digital-asset/daml). This package contains the public contracts that you use to trade with a V7 pool and to provide liquidity to it. It does not contain the pool, its price math, or its liquidity positions. Those are in the Tradecraft core package, which you do not need.

You work with these templates. Funding uses the token-standard `Holding`, `TransferInstruction`, and `Allocation` interfaces.

**`Factory`** : one per pool\
The entry point of a pool. You exercise its choices to create orders and to create and fund standing allocations. All its choices are nonconsuming. The venue and the vault sign the factory, and it has no observers, so you submit with its disclosure (section 4).

**`Order`** : one per request\
One requested operation: a swap, a liquidity deposit, a liquidity withdrawal, or a fee collection. The actor (your user) and the vault sign it, and the venue observes it. No tokens move when you create an order. The venue executes the open orders of each pool in batches.

**`Result`** : one per finished order\
The record of a finished order: its outcome and the amounts that moved. The vault signs it. The actor and the venue observe it.

**`RecoveryReceipt`** : one per owner recovery\
The record that a position owner and the destination party receive when a pool's recovery authority moves the owner's positions out of the pool (section 9.4).

**`Allocation`** ([token standard V2, committed](https://github.com/canton-foundation/cips/blob/main/cip-0112/cip-0112.md#416-committed-allocations-for-prefunded-trading-and-iterated-settlement)) : per actor, per pool, per instrument\
A standing allocation locks the actor's tokens for settlement by the venue. Orders can settle from it trade after trade. Each settlement archives the allocation contract and creates it again with the new funding, so track allocations by their metadata, not by contract ID (section 10).

**`WithdrawalRequest`** : per actor, per pool, per instrument\
The actor's request to release standing allocation funding to the wallet. The venue fulfills or rejects it (section 12).

### The high-level workflow

1. **Discover** the pool and its factory (section 4).
2. **Submit** swap orders through the factory, funded from the user's wallet (sections 5 to 7).
3. **Monitor** each order until its result appears (sections 8 and 13).
4. **Provide liquidity** with deposit, withdrawal, and collection orders (section 9).
5. **Use standing allocations** for regular trading, and withdraw their funding when the user needs it (sections 10 to 12).

## 2. The package

- Package name - `tradecraft-amm-order-v2`
- Version - `2.1.0`
- Daml SDK - `3.5.1`, LF target `2.3`
- Modules:
  - [`TC.Public.Order`](src/TC/Public/Order.daml) - the `Factory`, `Order`, and `Result` templates
  - [`TC.Public.Order.Types`](src/TC/Public/Order/Types.daml) - order specifications, funding types, and outcomes
  - [`TC.Public.Order.Recovery`](src/TC/Public/Order/Recovery.daml) - the `RecoveryReceipt` template
  - [`TC.Public.Order.Allocation`](src/TC/Public/Order/Allocation.daml) - standing allocation checks and funding
  - [`TC.Public.Order.Withdrawal`](src/TC/Public/Order/Withdrawal.daml) - the `WithdrawalRequest` template
- Dependencies - the Splice Token Standard 0.8.1 release bundle. [`daml.yaml`](daml.yaml) reads these DARs from `../dars/`:
  - `splice-api-token-metadata-v1-1.0.0.dar`
  - `splice-api-token-holding-v1-1.0.0.dar`
  - `splice-api-token-holding-v2-1.0.0.dar`
  - `splice-api-token-allocation-instruction-v2-1.0.0.dar`
  - `splice-api-token-allocation-v2-1.0.0.dar`
  - `splice-api-token-transfer-instruction-v1-1.0.0.dar`
  - `splice-token-standard-utils-2.0.0.dar`
- License - see [`LICENSE`](LICENSE)

The package does not depend on the Tradecraft core package. It is the only Tradecraft package that your participant needs: for swaps, for liquidity, and for recovery receipts.

### The built DAR

The built package is attached to the [v2.1.0 release](https://github.com/tradecraftfi/tradecraft-amm-order-v2/releases/tag/v2.1.0) as [`tradecraft-amm-order-v2-2.1.0.dar`](https://github.com/tradecraftfi/tradecraft-amm-order-v2/releases/download/v2.1.0/tradecraft-amm-order-v2-2.1.0.dar). It was made with Daml SDK 3.5.1. It contains the package and all of its dependencies, so you upload only this file.

- Package ID - `00e7b3e8adb051dd4e79f86b87c55d2d6126d13f463a0f9b86ba2dcd5ad12350`
- SHA-256 - `de29c0c75de149b4133c332ad4bebefb766d2a4ec1f64c9555dab4c899678e6a`

Download the file and check it before you upload it:

```shell
curl -LO https://github.com/tradecraftfi/tradecraft-amm-order-v2/releases/download/v2.1.0/tradecraft-amm-order-v2-2.1.0.dar
shasum -a 256 tradecraft-amm-order-v2-2.1.0.dar
unzip -l tradecraft-amm-order-v2-2.1.0.dar | grep -o '/tradecraft-amm-order-v2-2.1.0-[0-9a-f]*\.dalf'
```

The name of the main `.dalf` file contains the package ID. After you upload the DAR, check that the package ID on your participant is the same.

## 3. Network reference

Treat these values as configuration that your application reads from one place.

**API documentation**

- All environments - `https://docs.tradecraft.fi/api`

**API base**

- Mainnet - `https://api.tradecraft.fi/v1`
- Testnet - `https://tradecraft.validator.test.canton.obsidian.systems/amm-http-api`
- Devnet - `https://tradecraft.validator.dev.canton.obsidian.systems/amm-http-api`

**V7 routes**

- `GET /v7/disclosures/factories` - the pool reference, tick spacing, and factory disclosure of each V7 pool
- `GET /v7/pools` - the current state of each V7 pool, including its square-root price
- `GET /v7/swap/quote/{inputToken}/{outputToken}?amount=` - an exact-input swap quote at the current pool state
- `GET /v7/deposit/quote/{token1}/{token2}?minPrice=&maxPrice=&amount1=` - a deposit quote: the tick range, the liquidity, and the amounts that the pool debits. Give the tokens in pool order and exactly one of `amount1` and `amount2`.

**Venue and vault**

The venue and vault parties of a V7 pool are in its pool reference (section 4). Read them from `GET /v7/disclosures/factories`. Do not hard-code them.

## 4. Discover pools

Fetch the factory of each V7 pool:

```shell
curl https://api.tradecraft.fi/v1/v7/disclosures/factories | jq
```

Each entry contains:

- `pool` - the pool reference
- `tickSpacing` - the pool's tick spacing
- `disclosure` - the factory's disclosed contract: `templateId`, `contractId`, `createdEventBlob`, and `synchronizerId`

The pool reference identifies the pool in every order:

```haskell
data PoolRef = PoolRef
  with
    venue : Party
    vault : Party
    ammId : Text
      -- ^ The pool ID, for example "TC CNTN/OSCoin #883eb79f70af32b5823f585db8df4d8b-2 CL7"
    instrument1 : InstrumentId
    instrument2 : InstrumentId
      -- ^ The instrument order is fixed at pool creation
```

- Copy the pool reference exactly. The venue executes an order only when the order's pool reference is equal to the pool's.
- Treat `ammId` as an opaque text. It is not a lookup key for instruments.
- Prices are in `instrument2` per `instrument1`.
- The factory has no consuming choices, so its disclosure stays valid. If a submission fails because the factory is not found, fetch the disclosure again.

### Pool modes

Pool creation fixes the **modes** of each pool:

- The funding types that the pool accepts (sections 7 and 11).
- The settlement policy: immediate only, or deferred allowed (section 11.3).

The modes are not in the pool reference. Ask Tradecraft for the modes of each pool. An order that uses a funding type that the pool does not accept completes as rejected. Every pool accepts withdrawal and collection orders, whatever its funding types. Admission (next section) can still reject them.

### Pool admission

Some pools admit only approved holders, only approved liquidity providers, or both. The pool checks admission when it executes an order:

- Liquidity-provider admission applies to deposit, withdrawal, and collection orders. It does not apply to swaps.
- Holder admission applies to swaps and deposits (both pool instruments), and to withdrawals and collections (each instrument that actually moves).

An order that fails the check completes as rejected, and no tokens move. Admission does not apply to creating, cancelling, or expiring an order, or to a `WithdrawalRequest` (section 12).

## 5. Create orders

The actor exercises `Factory_CreateOrders` with one specification per order. Include the factory disclosure in the submission. The choice creates one `Order` per specification, in the given order. The factory's signatories supply the vault authority that the order needs.

Order creation reserves nothing and moves no tokens. It does not check the pool's modes or the actor's funds. The venue checks those when it executes the order.

**Choice signature**

```haskell
nonconsuming choice Factory_CreateOrders : Factory_CreateOrders_Result
  with
    actor : Party
    specs : [OrderSpec]
      -- ^ Must not be empty
  controller actor
```

**Result type**

```haskell
data Factory_CreateOrders_Result = Factory_CreateOrders_Result
  with
    orderCids : [ContractId Order]
      -- ^ One order per specification, in input order
```

**The order specification**

```haskell
data OrderSpec = OrderSpec
  with
    operation : Operation
    input : Input
    output : Output
    expiresAt : Optional Time
      -- ^ None = no expiry. Creation rejects an expiry that has passed.

data Operation
  = OpSwap
    with
      direction : SwapDirection
      sqrtPriceLimit : Numeric 20
  | OpDeposit
    with
      range : TickRange
      requestedLiquidity : Decimal
  | OpWithdraw
    with
      range : TickRange
      liquidity : Decimal
        -- ^ Zero selects the fee refresh
  | OpCollect
    with
      range : TickRange
      cap1 : Optional Decimal
      cap2 : Optional Decimal
        -- ^ Optional per-instrument caps on the collected amounts

data Input = Input
  with
    maxDebits : Map InstrumentId Decimal
      -- ^ The most the actor pays of each instrument. A missing
      --   instrument permits no debit of that instrument.
    funding : Funding
      -- ^ See sections 7 and 11

data Output = Output
  with
    minOutputs : Map InstrumentId Decimal
      -- ^ The least the actor accepts of each instrument

data TickRange = TickRange
  with
    lower : Int
      -- ^ Inclusive
    upper : Int
      -- ^ Exclusive
```

**Example: a swap funded from wallet holdings**

This specification sells 100 of `sold` for at least 99 of `bought`:

```haskell
OrderSpec with
  operation = OpSwap with
    direction = SwapExactInput with
      inputInstrument = sold
      inputAmount = 100.0
    sqrtPriceLimit = limit
  input = Input with
    maxDebits = Map.fromList [(sold, 100.0)]
    funding = FundingHoldings with
      holdingCids = [holding1, holding2]
  output = Output with
    minOutputs = Map.fromList [(bought, 99.0)]
  expiresAt = Some deadline
```

**Creation checks**

`Factory_CreateOrders` fails if one specification is not valid. The `Order` template's `ensure` clause applies the same checks, except the expiry and funding-source checks.

- Every instrument in `maxDebits` and `minOutputs` is one of the two pool instruments.
- Each `maxDebits` value is positive and at most `1.0e17`. Each `minOutputs` value is from zero to `1.0e17`.
- A funding list (sections 7 and 11) is not empty and has no duplicates.
- Each source in the funding list is valid now: it is the actor's, it is for a pool instrument, and its amount is positive. A transfer offer must wait for acceptance. A standing allocation must have the shape of section 10.1.
- The expiry, if set, has not passed. An expiry equal to the current time is accepted.
- A tick range has `lower < upper`, is within `-230270` to `230270`, and both bounds are multiples of the pool's `tickSpacing`.
- The operation-specific checks of sections 6 and 9.

## 6. Swap orders

```haskell
data SwapDirection
  = SwapExactInput
    with
      inputInstrument : InstrumentId
      inputAmount : Decimal
        -- ^ Sell this amount
  | SwapExactOutput
    with
      outputInstrument : InstrumentId
      outputAmount : Decimal
        -- ^ Buy this amount
```

- **`SwapExactInput`** - sell a fixed amount of the input instrument. Receive what the pool pays.
- **`SwapExactOutput`** - buy a fixed amount of the output instrument. Pay what the pool charges.

The swap instrument must be a pool instrument, and the amount must be positive and at most `1.0e17`. `sqrtPriceLimit` must be positive. Set the limits as follows:

| Direction | `maxDebits` | `minOutputs` |
|---|---|---|
| `SwapExactInput i a` | `{i: a}` | `{other instrument: the least the user accepts}` |
| `SwapExactOutput o b` | `{other instrument: the most the user pays}` | `{o: b}` for a full fill, or less to accept a partial fill |

**Partial fills.** Both directions can fill in part. The fill stops when the price reaches `sqrtPriceLimit` or when the pool has no more liquidity in that direction. A partial fill must still meet each `minOutputs` value in full. The minimum does not decrease in proportion to the fill. A swap that must cross more than 128 initialized ticks completes as rejected; it does not fill in part.

**Price limit.** `sqrtPriceLimit` is the square root of the price (`instrument2` per `instrument1`) at which the fill stops. `GET /v7/pools` shows the current square-root price of each pool.

| The order | The price | The limit must be |
|---|---|---|
| Sells `instrument1` or buys `instrument2` | Goes down | Below the current square-root price |
| Sells `instrument2` or buys `instrument1` | Goes up | Above the current square-root price |

An order whose limit is on the wrong side of the current price completes as rejected. A limit equal to the current price gives no fill. Calculate the limit from a quote and the user's slippage tolerance. `GET /v7/swap/quote/{inputToken}/{outputToken}?amount=` gives an exact-input quote.

**Operator fee.** A pool can charge an operator fee in one designated instrument. Then `maxDebits` applies to the total debit, including the fee, and `minOutputs` applies to the net amount that the user receives. Ask Tradecraft for the fee settings of each pool.

| Order | Fee effect |
|---|---|
| Exact input in the designated instrument | The stated amount is the total debit. The fee is taken from it. |
| Exact output in the other instrument | The fee is added to the designated input debit. |
| Exact input in the other instrument | The fee is taken from the designated output. The user receives the net amount. |
| Exact output in the designated instrument | The stated amount is the net amount. The pool calculates the gross amount. |

Rejected orders and orders with no fill pay no fee.

## 7. Immediate mode: funding from the wallet

The `funding` field selects where the debit comes from. One order uses one funding type. The pool's modes (section 4) set which types it accepts.

An order funded from the user's wallet always settles immediately: the debit, the output, and any change move in the transaction that executes the order. There are three wallet funding types: `FundingHoldings`, `FundingTransfersToVault`, and `FundingTransfersToActor`. The other two types use standing allocations (section 11).

```haskell
data Funding
  = FundingStanding
    -- ^ Use the actor's standing allocations (section 11)
  | FundingAllocations
    with
      allocationCids : [ContractId AllocationV2.Allocation]
      -- ^ Standing allocations of the actor, named in the order (section 11)
  | FundingHoldings
    with
      holdingCids : [ContractId HoldingV1.Holding]
      -- ^ Unlocked holdings in the actor's wallet
  | FundingTransfersToActor
    with
      instructionCids : [ContractId TransferInstruction]
      -- ^ Pending transfer offers to the actor
  | FundingTransfersToVault
    with
      instructionCids : [ContractId TransferInstruction]
      -- ^ Pending transfer offers from the actor to the pool vault
```

| Funding type | The order names | How the debit is paid | Where a swap output goes |
|---|---|---|---|
| `FundingHoldings` | Unlocked holdings of the actor | The venue reads the list in order and uses the shortest start of the holdings for that instrument that covers the debit. It locks exactly the debit into the actor's allocation for the instrument, and the settlement pays from that allocation. The rest returns to the wallet as unlocked holdings. | The wallet, as unlocked holdings |
| `FundingTransfersToVault` | Pending offers from the actor to the vault | The vault accepts the offers and funds its own allocation with exactly the debit. A surplus returns to the actor as unlocked holdings. | The actor's allocation for the output instrument |
| `FundingTransfersToActor` | Pending offers to the actor | The offers are accepted with the actor's authority, which the order signature gives. Then the debit is paid as for holdings. | The actor's allocation for the output instrument |

### 7.1 Funding with holdings

A swap funded with `FundingHoldings` pays its output to the wallet as unlocked holdings, in the transaction that executes the swap, unless the venue settles the output into one of the actor's allocations. The venue decides this at execution. The output does not arrive as a transfer offer, and no transfer pre-approval is necessary.

The input is locked into the actor's allocation for the input instrument before settlement. If the actor has no such allocation, execution creates one. After settlement, it stays live with the remaining funding, which can be zero. A later order can reuse it.

### 7.2 Funding with transfer offers

- `FundingTransfersToVault` names pending transfer offers that the actor sent to the pool vault. The actor must not be the vault.
- `FundingTransfersToActor` names pending transfer offers that other parties sent to the actor. The input is locked into the actor's allocation for the input instrument, as for holdings (section 7.1). A pool with holder or liquidity-provider admission does not accept this funding type.

With either type, a swap output settles into the actor's allocation for the output instrument. If the actor has no allocation for that instrument, execution creates one. To move those funds to the wallet, use a `WithdrawalRequest` (section 12).

### 7.3 Source rules

- The venue reads the named sources in order and stops at the first set that covers the debit. A problem with a source after that point does not affect the order.
- The order does not lock the sources that it names. Do not use them for other purposes until the order finishes:
  - If a named source is spent or withdrawn before execution, the venue's batch fails, and the venue then completes the order as rejected.
  - If a named source is still live but not usable (for example, a locked holding, or a source that another order in the same batch already uses), only this order completes as rejected.

## 8. The order lifecycle

1. The actor creates the order (section 5). The order (template `Order` in module `TC.Public.Order`, signatories `actor` + `vault`) is queryable like any other contract.
2. The venue executes the open orders of each pool in batches, oldest first.
3. The venue finishes each order with `Order_Complete`. The choice archives the order and creates a `Result` with one of these outcomes:
   - `OutcomeSettled` - the order executed, and the tokens moved in the same transaction. A partial fill, or a fill of zero, also gives this outcome. Read `actualInputs` and `actualOutputs` for the amounts.
   - `OutcomeRejected reason` - the order did not execute, and no tokens moved. The `reason` text is diagnostic. Do not parse it.
   - `OutcomePendingSettlement` - only for some swaps funded from standing allocations, on pools that allow deferred settlement (section 11.3).
4. Alternatively, the actor finishes the order with `Order_Cancel` or `Order_Expire`.

```haskell
-- The actor cancels a pending order. No tokens move.
choice Order_Cancel : ContractId Result
  -- ^ Creates a Result with OutcomeCancelled
  controller actor

-- The actor retires an order after its expiry has passed.
choice Order_Expire : ContractId Result
  -- ^ Creates a Result with OutcomeRejected "Order expired"
  controller actor
```

The venue does not execute an order after its `expiresAt`. An expired order stays live until the actor exercises `Order_Expire`, or until the venue completes it as rejected.

When the venue stops execution for a pool, the pool's open orders stay open. They do not complete as rejected, but the venue can reject them. Use `expiresAt` to limit how long an order can wait.

**The `Result` template**

```haskell
template Result
  with
    orderCid : ContractId Order
      -- ^ The finished order
    actor : Party
    pool : PoolRef
    operation : Operation
    outcome : Outcome
    actualInputs : Map InstrumentId Decimal
      -- ^ What the actor paid, including any operator fee
    actualOutputs : Map InstrumentId Decimal
      -- ^ What the actor received
    positionSnapshot : Optional PositionSnapshot
      -- ^ Liquidity orders only
    executedAt : Time
  where
    signatory pool.vault
    observer actor, pool.venue

data Outcome
  = OutcomeCancelled
  | OutcomeRejected Text
  | OutcomePendingSettlement
  | OutcomeSettled
```

A rejected or cancelled result has no payment amounts.

> **Important:** a `Result` alone is not proof of settlement. The vault is its only signatory and can create a `Result` directly. Confirm that the result came from an `Order_Complete` exercise on your order. Then find the token movement in the same transaction. For a pending result, find it in the transaction that creates the settled result.

## 9. Liquidity orders

A liquidity provider adds liquidity to a tick range of a pool and earns a share of the swap fees in that range. Each actor has at most one position per pool and tick range. The actor does not see the position contract itself. The actor's record of a position is the `positionSnapshot` in the `Result` of each accepted liquidity order.

```haskell
data PositionSnapshot = PositionSnapshot
  with
    owner : Party
    ammId : Text
    range : TickRange
    liquidity : Decimal
    feeGrowthInside1Last : Numeric 20
    feeGrowthInside2Last : Numeric 20
    tokensOwed1 : Numeric 20
    tokensOwed2 : Numeric 20
      -- ^ Fees recorded as owed. Fees earned since the position last
      --   changed are added by a deposit, a withdrawal, or a fee refresh.
```

### 9.1 Deposit

`OpDeposit range requestedLiquidity` adds `requestedLiquidity` to the actor's position for `range`, and creates the position if necessary.

- The pool calculates the amounts of the two instruments. Put each instrument that the deposit can debit in `maxDebits`. A range fully above or below the current price takes only one instrument.
- `minOutputs` must be empty. `requestedLiquidity` must be positive and at most `1.0e17`.
- The pool rejects a deposit that makes the position's total liquidity too large for its tick range. This limit keeps every amount of the range within the pool's numeric bounds, so that a full withdrawal stays possible.
- A deposit can use any funding type that the pool accepts (sections 7 and 11). With holdings or transfer offers, list the sources for both instruments in the one funding list. The venue takes each instrument's debit from the sources of that instrument.
- `GET /v7/deposit/quote/{token1}/{token2}` gives aligned ticks, the liquidity, and the amounts. The amounts are rounded up, as the pool debits them.

### 9.2 Withdraw

`OpWithdraw range liquidity` removes `liquidity` from the position and pays the principal to the actor's allocations.

- `maxDebits` must be empty, and the funding must be `FundingStanding`.
- Use `minOutputs` for slippage protection.
- A `liquidity` of zero is a fee refresh: it moves the earned fees into the position's owed amounts and pays nothing.

### 9.3 Collect

`OpCollect range cap1 cap2` pays the position's owed fees to the actor's allocations. The amounts are rounded down to ten decimal places and limited by the pool reserve and the optional caps.

- `maxDebits` must be empty, and the funding must be `FundingStanding`.
- A collect order pays only the owed amounts that the position already records. A deposit or a withdrawal moves the earned fees into the owed amounts. To include the fees earned since the position last changed, send a zero-liquidity withdrawal (fee refresh) before the collect order. The venue executes orders oldest first, so create the fee refresh first.

A withdrawal does not pay the earned fees. To receive the fees, send a collect order.

Withdrawal and collection orders debit nothing, so they need no allocation in advance. Their outputs settle into the actor's allocations for the pool instruments (section 10). If the actor has no allocation for an instrument, execution creates one. To move the funds to the wallet, use a `WithdrawalRequest` (section 12).

### 9.4 Recovery receipts

A pool can have a recovery authority, which pool creation fixes. Under the pool's recovery policy, the authority can remove an owner's positions from the pool and pay their value to an approved destination party. The owner does not sign this operation, and no `Result` is created.

Each recovery creates one `RecoveryReceipt` (module `TC.Public.Order.Recovery`). The vault signs it, and the owner, the destination, and the venue observe it. It has no choices and does not change.

```haskell
template RecoveryReceipt
  with
    pool : PoolRef
    owner : Party
      -- ^ The owner of the recovered positions
    destination : Party
      -- ^ The party that received the payment
    instructionId : Text
      -- ^ The recovery authority's reference for this recovery
    positions : [RecoveredPosition]
      -- ^ Each recovered position, also one that paid nothing
    moved : Map InstrumentId Decimal
      -- ^ The total paid to the destination, per instrument
    executedAt : Time
      -- ^ The ledger time of the recovery
    externalCutoff : Time
      -- ^ The record date that the recovery authority set
  where
    signatory pool.vault
    observer owner, destination, pool.venue

data RecoveredPosition = RecoveredPosition
  with
    snapshot : PositionSnapshot
      -- ^ The position's state before the recovery
    principal : Map InstrumentId Decimal
    accruedFees : Map InstrumentId Decimal
      -- ^ Amounts per instrument, at payment scale. If an instrument is
      --   not in a map, the recovery paid nothing for it.
```

- Rounding remainders below the payment scale stay in the pool. They are not in the receipt.
- The recovery pays into the destination's allocation for each instrument that it moves. The destination needs a standing allocation for the pool and each of those instruments in advance (section 10). To move the funds to the wallet, the destination uses a `WithdrawalRequest` (section 12).

## 10. Standing allocations

A standing allocation holds the actor's tokens for repeated settlement by the venue. It is necessary in these cases:

- A swap or deposit funded with `FundingStanding` or `FundingAllocations` (section 11) needs a funded standing allocation, in advance, for each instrument that it debits.
- A recovery destination needs one for each instrument that the recovery moves (section 9.4).

Withdrawal and collection outputs, and the outputs of swaps funded with transfer offers, settle into the actor's allocations. Execution creates a missing one. Orders funded with holdings or transfer offers do not need an allocation in advance.

### 10.1 The allocation shape

A standing allocation is a token-standard V2 `Allocation` that locks the actor's tokens for settlement by the venue. Every choice that uses a standing allocation checks that it has this shape:

- `committed = True` - the owner cannot withdraw the funds directly.
- `transferLegSides = []` and `settlementDeadline = None`.
- `nextIterationFunding = Some funding` - the same allocation serves trade after trade. `funding` is empty for an empty receiving allocation, or one positive entry for the allocation's instrument.
- `settlement` - `executors = [venue]`, `id = "tc-settlement"`, `cid = None`, and empty `meta`.
- `authorizer` - the actor's token-standard account. Its owner must be the actor.
- `meta` - `tradecraft.fi/amm-id` is the pool's `ammId`, and `tradecraft.fi/instrument-id` is the instrument ID.

> **Important:** the owner cannot withdraw a standing allocation directly. Only the venue, the sole executor, can settle or cancel it. Funds leave a standing allocation only through settlement or a cancellation by the venue. To release the funds to the wallet, the actor creates a `WithdrawalRequest` (section 12). Treat standing allocations as working capital, not storage.

> **Note:** keep one allocation per pool and instrument. For each batch, the venue uses at most one allocation per actor and instrument. Create the allocation once, then top it up with `Factory_FundStandingAllocation` (section 10.3).

### 10.2 Create a standing allocation

The actor exercises `Factory_CreateStandingAllocation`. For the instrument that the actor sells, pass `initialFunding` and the holdings that fund it. For the instrument that the actor receives, the allocation can start empty: `initialFunding = None` and no holdings.

You need:

- The factory disclosure (section 4).
- `authorizerAccount` - the actor's token-standard (HoldingV2) account. Its owner must be the actor. For Amulet and for Utility registry tokens, this is `{owner = Some actor, provider = None, id = ""}`.
- `allocationFactoryCid` and `extraArgs` - the instrument's V2 allocation factory and its choice context. The instrument's token-standard registry serves them at `POST /registry/allocation-instruction/v2/allocation-factory`. The request body wraps the allocate choice that the factory will exercise: `{"choiceArguments": <AllocationFactory_Allocate arguments>, "excludeDebugFields": false}`. The response is `{"factoryId", "choiceContext"}`. Pass `factoryId` as `allocationFactoryCid`, build `extraArgs` from `choiceContext.choiceContextData` (with empty `meta`), and include `choiceContext.disclosedContracts` in the submission.

The `AllocationFactory_Allocate` arguments contain the settlement and allocation of section 10.1, the holdings, and `actors = [actor]`.

The registry must create the allocation in one step. The choice fails if `AllocationFactory_Allocate` returns a pending allocation instruction.

**Choice signature**

```haskell
nonconsuming choice Factory_CreateStandingAllocation : Factory_CreateStandingAllocation_Result
  with
    actor : Party
      -- ^ The user who owns the funds
    args : CreateSettlementAllocationArgs
  controller actor

data CreateSettlementAllocationArgs = CreateSettlementAllocationArgs
  with
    ammId : Text
      -- ^ The pool's ammId
    instruments : [InstrumentId]
      -- ^ [instrument1, instrument2] of the pool, in this order
    venue : Party
      -- ^ The pool's venue
    allocationActor : Party
      -- ^ Must be the actor
    instrument : InstrumentId
      -- ^ The instrument of the new allocation; one of the two pool instruments
    authorizerAccount : HoldingV2.Account
      -- ^ The actor's token-standard account; its owner must be the actor
    initialFunding : Optional Decimal
      -- ^ Some x (x > 0) = fund the allocation with x from inputHoldingCids.
      --   None = empty receiving allocation (inputHoldingCids must be empty)
    inputHoldingCids : [ContractId HoldingV2.Holding]
      -- ^ The actor's V2 holdings that fund the allocation
    allocationFactoryCid : ContractId AllocationInstructionV2.AllocationFactory
      -- ^ The instrument's V2 allocation factory
    extraArgs : ExtraArgs
      -- ^ Choice context for the allocation factory
```

**Result type**

```haskell
data Factory_CreateStandingAllocation_Result = Factory_CreateStandingAllocation_Result
  with
    allocationCid : ContractId AllocationV2.Allocation
      -- ^ The new standing allocation
    changeHoldingCids : [ContractId HoldingV2.Holding]
      -- ^ Change holdings returned to the actor
```

### 10.3 Top up a standing allocation

`Factory_FundStandingAllocation` adds funds to a live allocation from the actor's holdings in one atomic transaction. The choice locks the holdings in a new source allocation, then merges the source into the target with one settlement. It returns the target's successor.

You also need the `SettlementFactory` contract and its choice context for the instrument's admin. The same registry serves them at `POST /registry/allocation/v2/settlement-factory`. The request has the same shape as the allocation-factory call: the body wraps the `SettlementFactory_SettleBatch` choice arguments. The response's `factoryId` and `choiceContext` give `settlementFactoryCid` and `settlementExtraArgs`.

**Choice signature**

```haskell
nonconsuming choice Factory_FundStandingAllocation : Factory_FundStandingAllocation_Result
  with
    args : FundSettlementAllocationArgs
  controller args.allocationActor

data FundSettlementAllocationArgs = FundSettlementAllocationArgs
  with
    ammId : Text
    instruments : [InstrumentId]
      -- ^ [instrument1, instrument2] of the pool, in this order
    venue : Party
    userAccount : HoldingV2.Account
      -- ^ Must be the target allocation's authorizer account
    allocationCid : ContractId AllocationV2.Allocation
      -- ^ The live standing allocation to top up
    allocationActor : Party
      -- ^ The allocation owner; the choice controller
    instrument : InstrumentId
      -- ^ The allocation's instrument
    additionalFunding : Decimal
      -- ^ Must be positive
    inputHoldingCids : [ContractId HoldingV2.Holding]
      -- ^ The actor's V2 holdings that supply the additional funds
    allocationFactoryCid : ContractId AllocationInstructionV2.AllocationFactory
    allocationExtraArgs : ExtraArgs
    settlementFactoryCid : ContractId AllocationV2.SettlementFactory
      -- ^ Settlement factory of the instrument's admin
    settlementExtraArgs : ExtraArgs
```

**Result type**

```haskell
data Factory_FundStandingAllocation_Result = Factory_FundStandingAllocation_Result
  with
    allocationCid : ContractId AllocationV2.Allocation
      -- ^ The surviving allocation. The settlement archives the topped-up one.
    changeHoldingCids : [ContractId HoldingV2.Holding]
```

## 11. Funding from standing allocations

Two funding types use the actor's standing allocations (section 10) instead of the wallet. The `Funding` type is in section 7.

| Funding type | The order names | How the debit is paid | Where a swap output goes |
|---|---|---|---|
| `FundingStanding` | Nothing | The debit settles from the actor's standing allocation for the input instrument. | The actor's allocation for the output instrument |
| `FundingAllocations` | Standing allocations of the actor | The venue uses the first listed allocation that is valid and covers the debit. It does not add allocations together. (If the venue already uses one of the actor's allocations for the instrument, see section 11.2.) | The actor's allocation for the output instrument |

### 11.1 Swaps from standing allocations

A swap from standing allocations needs a funded allocation for the input instrument. The output settles into the actor's allocation for the output instrument. That allocation can start empty, and execution creates it if the actor has none. Each order that the venue accepts reserves its debit from the input allocation. When the remaining funding cannot pay an order, that order completes as rejected.

To move funds from an allocation to the wallet, use a `WithdrawalRequest` (section 12).

### 11.2 Notes

- With `FundingAllocations`, when the venue already uses one of the actor's allocations for the input instrument in the batch, the order must name that allocation. Keep one allocation per pool and instrument (section 10.1) to avoid this conflict.
- Withdrawal and collection orders (section 9) must use `FundingStanding`.
- With `FundingAllocations`, the venue skips a named allocation that is not valid or does not cover the debit. If none is usable, the order completes as rejected.
- Each settlement and top-up archives an allocation and creates its successor with a new contract ID. An order that names an allocation can use it only until that allocation next settles. If the venue reads a named allocation that is already archived, the venue's batch fails, and the venue then completes the order as rejected. For orders that wait, `FundingStanding` avoids this problem.

### 11.3 Deferred settlement

A pool's settlement policy is either immediate only or deferred allowed (section 4). On a pool that allows deferred settlement, a swap funded with `FundingStanding` or `FundingAllocations` can complete with `OutcomePendingSettlement`: the pool accepted the swap, and the payments follow in a later transaction. When the payments succeed, that transaction archives the pending `Result` and creates a new `Result` with `OutcomeSettled`.

Such a swap can also finish as `OutcomeSettled` directly, so do not rely on `OutcomePendingSettlement` for a given order. Orders funded from the wallet (section 7) always settle immediately.

## 12. Withdraw allocation funding

The actor cannot cancel a standing allocation (section 10.1). Instead, the actor creates a `WithdrawalRequest` for one pool and one instrument, and the venue cancels the allocation for the actor. The funds return to the actor's wallet as unlocked holdings. No transfer pre-approval is necessary.

**Creating the request**

The actor is the request's only signatory, so the actor creates it with a plain create command. No factory, choice, or disclosure is necessary. Each request covers one instrument. To empty both allocations of a pool, create one request per instrument.

**Template** (module `TC.Public.Order.Withdrawal`)

```haskell
template WithdrawalRequest
  with
    actor : Party
      -- ^ The allocation owner
    venue : Party
      -- ^ The pool's venue, from the pool reference
    ammId : Text
      -- ^ The pool's ammId
    instrument : InstrumentId
      -- ^ The pool instrument whose allocation to withdraw
    createdAt : Time
      -- ^ The current time
  where
    signatory actor
    observer venue
```

**Outcomes**

Every outcome archives the request:

- **Fulfilled** (`WithdrawalRequest_Fulfill`) - the venue cancels the actor's funded allocations for the pool and instrument. The entire funding of each cancelled allocation returns to the actor's wallet. Partial withdrawals are not supported. The choice checks on the ledger that each allocation has the shape of section 10.1 and belongs to the actor, the pool, and the instrument. The allocations must hold funding: an allocation with zero funding alone cannot be released.
- **Rejected with `RejectReason_PendingSettlements`** - the actor has trades on the pool that have not settled yet. Create a new request later.
- **Rejected with `RejectReason_InsufficientFunds`** - the actor has no funded allocation for the pool and instrument.
- **Rejected with `RejectReason_OtherReason reason`** - the venue declined for the stated reason.
- **Cancelled** (`WithdrawalRequest_Cancel`) - the actor takes the request back before the venue acts on it.

A rejected or cancelled request does not change the allocation or its funding. To learn the outcome, watch for the archival of the request (section 13).

**Fulfill result type**

```haskell
data WithdrawalRequest_Fulfill_Result = WithdrawalRequest_Fulfill_Result
  with
    withdrawnTotal : Decimal
      -- ^ The funding that the allocations held before cancellation
    allocationCids : [ContractId AllocationV2.Allocation]
      -- ^ The cancelled allocations
    releasedHoldingCids : [ContractId HoldingV2.Holding]
      -- ^ The holdings released to the actor's wallet
```

> **Note:** a fulfilled withdrawal removes the allocation for that instrument. Open orders on the pool that need it fail at execution. Cancel them first with `Order_Cancel` (section 8). To trade from standing allocations again, create a new allocation (section 10.2). To withdraw only part of the funding, withdraw all of it, then create a new allocation for the amount that you want to keep committed.

## 13. Monitor orders

**Recommended**\
Use PQS (Participant Query Store) to follow the `Order`, `Result`, `RecoveryReceipt`, and `WithdrawalRequest` templates and the token-standard `Holding` and `Allocation` interfaces for your actor.

**Without PQS**\
Query the participant's active contracts for your actor. When an order disappears, find the `Result` with that `orderCid`, then look up the transaction that archived the order. The debit, the change, and the output are in that same transaction.

- An order is finished when it is archived. `Order_Complete` means that the venue finished it. `Order_Cancel` and `Order_Expire` mean that the actor finished it.
- Change, refund, and holdings-swap output go to the actor's wallet as unlocked holdings.
- Every settlement and every top-up archives the standing allocation and creates it again with the new funding. Track allocations by their metadata (`tradecraft.fi/amm-id`, `tradecraft.fi/instrument-id`) and settlement ID (`tc-settlement`), not by contract ID.
- The actor signs the order, so the actor's transaction stream also shows the venue's execution choices on it (section 14).

## 14. Type reference

**Choices on `Factory`** (actor; nonconsuming; submit with the factory disclosure)

- `Factory_CreateOrders` → `Factory_CreateOrders_Result` (contains `[ContractId Order]`)
- `Factory_CreateStandingAllocation` → `Factory_CreateStandingAllocation_Result` (contains `ContractId AllocationV2.Allocation` + change holdings)
- `Factory_FundStandingAllocation` → `Factory_FundStandingAllocation_Result` (contains the surviving allocation + change holdings)

**Choices on `Order`**

- `Order_Cancel` → `ContractId Result` (actor; outcome `OutcomeCancelled`)
- `Order_Expire` → `ContractId Result` (actor; after expiry; outcome `OutcomeRejected "Order expired"`)
- `Order_Complete` → `ContractId Result` (venue and vault; finishes an executed or rejected order)
- `Order_ReadFundingSource` → `Either Text FundingSourceView` (venue and vault; nonconsuming; reads one source that the order names, with no token effect)
- `Order_PrepareFunding` → `Order_PrepareFunding_Result` (venue and vault; nonconsuming; converts the named sources into allocations at execution)

**Choices on other templates**

- `Result_MarkSettled` → `ContractId Result` (venue and vault; makes a pending result settled in the settlement transaction)
- `WithdrawalRequest_Cancel` → `()` (actor)
- `WithdrawalRequest_Reject` → `()` (venue; carries `reason : RejectReason`)
- `WithdrawalRequest_Fulfill` → `WithdrawalRequest_Fulfill_Result` (venue)

**Templates**

- `Factory` (`TC.Public.Order`) : one per pool; signatories `venue` + `vault`; no observers
- `Order` (`TC.Public.Order`) : pending order; signatories `actor` + `vault`; observer `venue`
- `Result` (`TC.Public.Order`) : finished order; signatory `vault`; observers `actor` + `venue`
- `RecoveryReceipt` (`TC.Public.Order.Recovery`) : record of one owner recovery; signatory `vault`; observers `owner`, `destination`, and `venue`; no choices
- `Allocation` (token standard V2) : standing allocation; archived and created again with new funding at every settlement and top-up
- `WithdrawalRequest` (`TC.Public.Order.Withdrawal`) : request to release standing allocation funding; signatory `actor`; observer `venue`

**Data types** (`TC.Public.Order.Types` unless stated)

- `PoolRef` : `venue`, `vault`, `ammId`, `instrument1`, `instrument2`
- `OrderSpec`, `Input`, `Output` : one order and its limits
- `TickRange` : `lower` (inclusive), `upper` (exclusive)
- `PositionSnapshot` : a position's state after a liquidity order
- `RecoveredPosition` (`TC.Public.Order.Recovery`) : one recovered position in a `RecoveryReceipt`
- `CreateSettlementAllocationArgs`, `FundSettlementAllocationArgs` (`TC.Public.Order.Allocation`) : arguments of the factory allocation choices
- `InstrumentContext`, `PlannedDebit`, `DebitSource` : arguments that the venue supplies to `Order_PrepareFunding`
- `FundingSource`, `FundingSourceView` : argument and result of `Order_ReadFundingSource`

**Enums**

- `Operation` : `OpSwap`, `OpDeposit`, `OpWithdraw`, or `OpCollect`
- `SwapDirection` : `SwapExactInput` (sell a fixed amount) or `SwapExactOutput` (buy a fixed amount)
- `Funding` : `FundingHoldings`, `FundingTransfersToVault`, `FundingTransfersToActor`, `FundingStanding`, or `FundingAllocations`
- `Outcome` : `OutcomeCancelled`, `OutcomeRejected reason`, `OutcomePendingSettlement`, or `OutcomeSettled`
- `RejectReason` : `RejectReason_PendingSettlements`, `RejectReason_InsufficientFunds`, or `RejectReason_OtherReason reason`

**Constants**

- Settlement ID - `tc-settlement`
- Allocation metadata keys - `tradecraft.fi/amm-id`, `tradecraft.fi/instrument-id`
- Amount bound - `1.0e17` for token and liquidity amounts
- Tick bounds - `-230270` to `230270`
