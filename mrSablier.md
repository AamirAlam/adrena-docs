# MrSablier – Code Structure Guide

This document explains the repository layout and the responsibilities of each module so you can quickly navigate the codebase.

## What this repo is

`MrSablier` is an off-chain Rust **keeper** that:

- consumes Solana account updates via **Yellowstone gRPC** (Geyser)
- maintains in-memory indexes of key Adrena accounts (positions, order books, user profiles, custodies)
- periodically evaluates liquidation / SL / TP / limit-order conditions
- submits transactions to the **Adrena program** using `anchor-client`

## High-level architecture

The process has two main "inputs" and one "output":

- **Input A: Account updates (stream)**
  - Source: Yellowstone gRPC subscription
  - Used for: keeping local indexed state up to date

- **Input B: Price updates (poll)**
  - Source: `https://datapi.adrena.xyz/last-trading-prices`
  - Used for: deciding whether an automated action should be executed

- **Output: Transactions**
  - Destination: Solana JSON-RPC endpoint (via `anchor-client`)
  - Purpose: execute Adrena instructions (liquidations, SL/TP closes, limit order execution)

## Repository layout

```text
MrSablier/
  Cargo.toml
  README.md
  src/
    client.rs
    process_stream_message.rs
    evaluate_and_run_automated_orders.rs
    update_indexes.rs
    priority_fees.rs
    api/
      mod.rs
      last_trading_prices.rs
    handlers/
      mod.rs
      create_ixs.rs
      liquidate_long.rs
      liquidate_short.rs
      sl_long.rs
      sl_short.rs
      tp_long.rs
      tp_short.rs
      execute_limit_order_long.rs
      execute_limit_order_short.rs
    utils/
      mod.rs
      derive_discriminator.rs
```

Notes:

- The crate builds a single binary named `mrsablier` (see `Cargo.toml` `[[bin]]`), with entrypoint `src/client.rs`.
- The repo depends on `adrena-abi` (Git dependency) which provides:
  - program IDs / PDAs / constants
  - account types (Borsh/Anchor layouts)
  - instruction argument structs and account metas

## Core entrypoint

### `src/client.rs`

This is the main binary:

- **CLI parsing** (`clap`)
  - `--endpoint`: gRPC endpoint used to connect to the Yellowstone stream
  - `--x-token`: optional auth token header for some gRPC providers
  - `--commitment`: stream commitment level
  - `--payer-keypair`: keypair used to sign and pay for transactions

- **Connectivity**
  - Creates a `GeyserGrpcClient` and opens a `subscribe_with_request` stream.
  - Creates an `anchor_client::Client` to talk to Solana JSON-RPC.

- **In-memory state**
  - Uses `Arc<RwLock<HashMap<Pubkey, ...>>>` maps for indexed state:
    - positions
    - custodies
    - limit order books
    - user profiles

- **Startup indexing** (RPC scans)
  - Fetches `Pool` and some baseline custodies (ex: USDC custody)
  - Loads existing program accounts via `program.accounts::<T>(filters)`
    - filters are based on Anchor discriminators (`derive_discriminator`) and optional datasize filters

- **Background tasks**
  - priority fee polling (every ~5 seconds)
  - price polling (every ~400 ms)

- **Core event loop**
  - `tokio::select!` between:
    - periodic "price tick" to evaluate automated orders
    - inbound gRPC stream messages to update indexes

### Subscription filtering (inside `client.rs`)

The subscription request is built with account filters that generally include:

- `owner = adrena_abi::ID` (the Adrena program)
- `memcmp` filter at offset `0` for the Anchor discriminator bytes
- for some accounts, a `datasize` filter to select a specific version (example: `UserProfile` V2)

Additionally, the code adds per-account "close monitoring" subscriptions for known account pubkeys so it can detect deletions/closures.

## Stream processing

### `src/process_stream_message.rs`

Responsibilities:

- Consumes a single `SubscribeUpdate` from the gRPC stream.
- Determines which filter matched (positions create/update, positions close, etc.).
- Updates the corresponding in-memory index:
  - `update_indexed_positions`
  - `update_indexed_limit_order_books`
  - `update_indexed_user_profiles`
- When a create/close event changes the set of relevant accounts, it triggers a **subscription refresh** by re-sending a new `SubscribeRequest` over `subscribe_tx`.

Key concept:

- The keeper keeps an index of accounts and adjusts the gRPC subscription to explicitly track known accounts for close events.

## Automated order evaluation

### `src/evaluate_and_run_automated_orders.rs`

Responsibilities:

- On each "price tick", for each relevant oracle symbol:
  - find the custody associated with that oracle
  - iterate indexed positions (and limit orders) associated with that custody
  - spawn tasks to evaluate and potentially execute:
    - stop loss
    - take profit
    - liquidation
    - limit order execution

It uses:

- `indexed_*` maps for current on-chain state snapshots
- `oracle_price` (last trading price) for trigger evaluation
- `oracle_prices` (batch/attested format) as additional data passed into instructions

## Index management

### `src/update_indexes.rs`

Responsibilities (conceptually):

- Parse raw account data into typed structs from `adrena_abi`.
- Maintain HashMaps keyed by account `Pubkey`.
- Provide helper routines to "ensure custodies are present" for positions and orderbooks.

These functions are called from:

- startup indexing (bulk load)
- stream processing (incremental updates)

## Priority fees

### `src/priority_fees.rs`

Responsibilities:

- Fetch recent prioritization fees via Solana RPC.
- Compute a "mean prioritization fee by percentile" used for transaction prioritization.

The main loop stores these values in a shared `PriorityFees` struct (behind an `Arc<RwLock<...>>`).

## Price API client

### `src/api/last_trading_prices.rs`

Responsibilities:

- Poll `https://datapi.adrena.xyz/last-trading-prices`
- Parse response into:
  - `HashMap<LimitedString, OraclePrice>` (simple price view)
  - `ChaosLabsBatchPrices` (signed/batched payload)

This is used by the automated-order evaluator.

## Instruction building and execution

### `src/handlers/*`

Responsibilities:

- Each file corresponds to a specific keeper action, for example:
  - `liquidate_long.rs` / `liquidate_short.rs`
  - `sl_long.rs` / `sl_short.rs`
  - `tp_long.rs` / `tp_short.rs`
  - `execute_limit_order_long.rs` / `execute_limit_order_short.rs`

They generally:

- check whether conditions are met
- build instruction(s) using data from:
  - position/orderbook structs
  - custody structs
  - pool
  - oracle payloads
- submit transactions through the Anchor `Program` handle

### `src/handlers/create_ixs.rs`

A shared helper module responsible for assembling instruction arguments and account metas used by multiple handlers.

## Utilities

### `src/utils/*`

Common helpers. The most important one conceptually:

- **Anchor discriminator derivation**
  - used to identify account types via the first 8 bytes
  - drives both RPC scanning filters and gRPC subscription memcmp filters

## Data flow cheat sheet

- `client.rs`
  - boot
  - RPC scan to index current state
  - open gRPC stream with filters
  - spawn polling tasks
  - loop:
    - on stream update: `process_stream_message` → update indexes → maybe update subscription
    - on price tick: `evaluate_and_run_automated_orders` → call handlers → send tx

## Practical "Where do I start reading?"

Recommended reading order:

1. `src/client.rs` (startup + main loop)
2. `src/process_stream_message.rs` (how stream updates update state)
3. `src/update_indexes.rs` (how raw accounts become typed state)
4. `src/evaluate_and_run_automated_orders.rs` (trigger evaluation)
5. One handler end-to-end (ex: `handlers/liquidate_long.rs`) + `handlers/create_ixs.rs`
