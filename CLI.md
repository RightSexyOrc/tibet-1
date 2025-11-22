# Tibet CLI

This document describes the `tibet.py` command-line interface.

The CLI groups all commands under a single entry point:

```bash
python3 tibet.py [COMMAND] [OPTIONS]
```

Run any command with `--help` to see all available options and defaults:

```bash
python3 tibet.py --help
python3 tibet.py config-node --help
python3 tibet.py deposit-liquidity --help
```

---

## 1. Quick command overview

### 1.1 Command list

| Command                              | What it does                                                                               |
| ------------------------------------ | ------------------------------------------------------------------------------------------ |
| `config-node`                        | Configure Chia root, network, RPC endpoint, and wallet type (Sage vs reference wallet).    |
| `test-node-config`                   | Check connectivity to the full node and wallet.                                            |
| `launch-router`                      | Launch a v2 router (XCH–CAT) or v2r router (XCH–rCAT) and store its launcher id in config. |
| `set-routers`                        | Manually set router launcher IDs (v2 and/or v2r) in `config.json`.                         |
| `launch-test-token`                  | Launch a test CAT or rCAT token.                                                           |
| `create-pair`                        | Create a new token–XCH or token–rXCH pair.                                                 |
| `sync-pairs`                         | Sync router state and cache known pairs in `config.json`.                                  |
| `get-pair-info`                      | Show reserves, LP asset id, and config for a given pair.                                   |
| `deposit-liquidity`                  | Add liquidity to a pair (you receive LP tokens).                                           |
| `remove-liquidity`                   | Remove liquidity from a pair (you burn LP tokens).                                         |
| `xch-to-token`                       | Swap XCH for tokens using a pair.                                                          |
| `token-to-xch`                       | Swap tokens for XCH using a pair.                                                          |
| `create-pair-with-initial-liquidity` | Create a new pair and deposit initial liquidity in a single transaction.                   |
| `rebase-up`                          | For rCAT pairs only: rebase up the token supply via a special message spend bundle.        |

### 1.2 Typical workflows (very short)

- **Configure and test**

  - `python3 tibet.py config-node [--chia-root ...] [--use-sage/--use-local-node ...]`
  - `python3 tibet.py test-node-config`

- **Launch router + test token + pair**

  - `python3 tibet.py launch-router [--rcat] [--fee ...] [--push-tx]`
  - `python3 tibet.py launch-test-token --amount <CAT_SUPPLY> [--push-tx] [--hidden-puzzle-hash ...]`
  - `python3 tibet.py create-pair --asset-id <TAIL_HASH> [--hidden-puzzle-hash ...] [--inverse-fee ...] [--fee ...] [--push-tx]`

- **Sync and inspect a pair**

  - `python3 tibet.py sync-pairs [--rcat]`
  - `python3 tibet.py get-pair-info --asset-id <TAIL_HASH>`

- **Add liquidity ("deposit liquidity")**

  - Generate offer and bundle:

    ```bash
    python3 tibet.py deposit-liquidity \
      --asset-id <TAIL_HASH> \
      --token-amount <TOKEN_MOJOS> \
      [--xch-amount <XCH_MOJOS>] \
      [--fee <FEE_MOJOS>] \
      [--use-fee-estimate]
    ```

  - Optionally broadcast:

    ```bash
    python3 tibet.py deposit-liquidity \
      --asset-id <TAIL_HASH> \
      --offer offer.txt \
      --push-tx
    ```

- **Remove liquidity**

  - `python3 tibet.py remove-liquidity --asset-id <TAIL_HASH> --liquidity-token-amount <LP_MOJOS> [--use-fee-estimate] [--push-tx]`

- **Swap**

  - XCH → token: `python3 tibet.py xch-to-token --asset-id <TAIL_HASH> --xch-amount <MOJOS> [--use-fee-estimate] [--push-tx]`
  - Token → XCH: `python3 tibet.py token-to-xch --asset-id <TAIL_HASH> --token-amount <MOJOS> [--use-fee-estimate] [--push-tx]`

---

## 2. Workflow-oriented guide

This section explains common tasks step by step using the commands above.

### 2.1 Configure node and wallet

1. **Write basic config**

   ```bash
   python3 tibet.py config-node \
     --chia-root ~/.chia/mainnet \
     --network mainnet \
     --use-sage False \
     --use-local-node False
   ```

   This:

   - reads `config.yaml` from your `chia_root`,
   - determines `agg_sig_me_additional_data` from network overrides (if present),
   - writes `chia_root`, `agg_sig_me_additional_data`, `use_sage`, and `rpc_url` (coinset.org or testnet11) to `config.json`.

2. **Test connectivity**

   ```bash
   python3 tibet.py test-node-config
   ```

   This checks:

   - full node health (`healthz()`),
   - wallet connectivity (or just Sage availability if `use_sage` is enabled).

### 2.2 Launch a router

Routers are the on-chain contracts that manage pairs.

```bash
python3 tibet.py launch-router \
  [--rcat] \
  [--fee <FEE_MOJOS>] \
  [--push-tx]
```

- `--rcat` – if set, launch a v2r router for XCH–rCAT pairs. Otherwise, a v2 router for XCH–CAT pairs is launched.
- `--fee` – fee in mojos for the router launch.
- `--push-tx` – if set, the signed spend bundle is sent to the full node and:
  - for v2: `router_launcher_id`, `router_last_processed_id`, and empty `pairs` dictionary are recorded in `config.json`.
  - for v2r: `rcat_router_launcher_id`, `rcat_router_last_processed_id`, and empty `rcat_pairs` dictionary are recorded.

If `--push-tx` is not set, the signed bundle is written to `spend_bundle.json` for manual broadcasting.

### 2.3 Launch a test token (CAT or rCAT)

```bash
python3 tibet.py launch-test-token \
  --amount <TOKEN_SUPPLY> \
  [--push-tx] \
  [--hidden-puzzle-hash <64_HEX>]
```

- `--amount` – number of tokens; 1 CAT = 1000 mojos; the XCH coin used will spend `amount * 1000` mojos.
- `--hidden-puzzle-hash` – optional 32-byte (64 hex chars) hidden puzzle hash to create an rCAT; if omitted, a normal CAT is created.
- `--push-tx` – if set, broadcasts the spend bundle and calls the wallet to add the new asset id (TAIL) as a wallet.

On success it prints:

- the XCH coin used,
- the token asset id (TAIL hash),
- and either pushes the bundle or writes `spend_bundle.json`.

### 2.4 Create a pair

Use `create-pair` when you already have a router and token TAIL.

```bash
python3 tibet.py create-pair \
  --asset-id <TAIL_HASH> \
  [--hidden-puzzle-hash <64_HEX>] \
  [--inverse-fee <INT>] \
  [--fee <FEE_MOJOS>] \
  [--push-tx]
```

- `--asset-id` (required) – token TAIL hash (64 hex characters). A basic length check is performed.
- `--hidden-puzzle-hash` – if supplied, the pair is XCH–rCAT; must be 64 hex characters.
- `--inverse-fee` – only relevant for rCAT pairs; must be between 952 and 999 inclusive when `hidden_puzzle_hash` is set. 999 ≈ 0.1% fee.
- `--fee` – XCH fee; must be at least `ROUTER_MIN_FEE` (42e9 mojos).
- `--push-tx` – broadcast the spend bundle and create the liquidity asset wallet.

The command will:

1. Ensure a router is configured (normal vs rCAT router depending on `hidden_puzzle_hash`).
2. Sync the router and store any new pairs into `pairs` or `rcat_pairs` in `config.json`.
3. Create the pair from a selected XCH coin.
4. Print:
   - the pair launcher id,
   - the liquidity asset id (TAIL of the LP token).
5. Either push the transaction and add the LP CAT wallet, or write `spend_bundle.json`.

### 2.5 Sync pairs

```bash
python3 tibet.py sync-pairs [--rcat]
```

- `--rcat` – syncs rCAT pairs; if not set, syncs normal CAT pairs.

This will:

- load the correct router state (`router_last_processed_id` or `rcat_router_last_processed_id`),
- sync the router coin on-chain,
- update:
  - `pairs` (simple mapping from TAIL → pair launcher id) for CAT pairs, or
  - `rcat_pairs` (TAIL → list of objects with `hidden_puzzle_hash`, `inverse_fee`, `launcher_id`) for rCAT pairs.

### 2.6 Inspect pair info (`get-pair-info`)

```bash
python3 tibet.py get-pair-info --asset-id <TAIL_HASH>
```

- `--asset-id` (required) – TAIL hash of the token in the pair.

This command:

1. Looks up the relevant pair via `config.json` using:
   - `pairs[asset_id]` (normal CAT pair), and/or
   - `rcat_pairs[asset_id]` (one or more rCAT pairs).
2. If multiple pairs exist, it may prompt you to select one and optionally remember your choice.
3. Syncs the pair on-chain and prints:
   - current pair coin id and puzzle hash,
   - liquidity asset id (lp TAIL),
   - for rCAT pairs: hidden puzzle hash and inverse fee (plus implied fee percentage),
   - XCH reserve, token reserve, total liquidity (human-readable units).

### 2.7 Add liquidity (`deposit-liquidity`)

You can either supply an existing offer or let the command generate one for you.

```bash
python3 tibet.py deposit-liquidity \
  --asset-id <TAIL_HASH> \
  [--offer <OFFER_OR_PATH>] \
  [--token-amount <TOKEN_MOJOS>] \
  [--xch-amount <XCH_MOJOS>] \
  [--fee <FEE_MOJOS>] \
  [--use-fee-estimate] \
  [--push-tx]
```

- `--asset-id` (required) – TAIL hash of the token in the pair.
- `--offer` – existing offer (bech32 string or file path). If provided, no new offer is generated.
- `--token-amount` – token amount in mojos for the deposit. Required if generating a new offer (and must be non-zero).
- `--xch-amount` – XCH amount in mojos; only required if the pair has no liquidity yet. If the pair already has liquidity, XCH amount is computed from reserves.
- `--fee` – fee (in mojos) used when generating a new offer.
- `--use-fee-estimate` – if set and a new offer is generated, estimate fee based on the resulting spend.
- `--push-tx` – if set, broadcast the liquidity deposit transaction; otherwise write it to `spend_bundle.json`.

Flow when generating a new offer:

1. Sync the pair, compute LP token TAIL and liquidity token amount.
2. If `--use-fee-estimate` is set, call the fee estimator.
3. For reference wallet (`use_sage = False`):
   - locate token and LP wallets,
   - build an offer where:
     - you give XCH + some LP tokens from the XCH wallet,
     - you give CAT tokens from the token wallet,
     - you receive LP tokens.
4. For Sage: construct an equivalent offer programmatically.
5. Save `offer.txt` and continue to build the on-chain liquidity deposit spend.
6. Either push or write `spend_bundle.json` depending on `--push-tx`.

### 2.8 Remove liquidity (`remove-liquidity`)

```bash
python3 tibet.py remove-liquidity \
  --asset-id <TAIL_HASH> \
  [--offer <OFFER_OR_PATH>] \
  [--liquidity-token-amount <LP_MOJOS>] \
  [--fee <FEE_MOJOS>] \
  [--use-fee-estimate] \
  [--push-tx]
```

- `--asset-id` (required) – TAIL hash of the token in the pair.
- `--offer` – existing offer (string or filepath) to burn LP tokens for XCH + tokens.
- `--liquidity-token-amount` – LP token amount in mojos; required when generating a new offer.
- `--fee` – fee used when generating an offer.
- `--use-fee-estimate` – use estimator when generating the offer.
- `--push-tx` – if set, broadcast the liquidity removal; otherwise write `spend_bundle.json`.

Flow:

1. Sync the pair and compute how many tokens and XCH correspond to the LP amount.
2. Generate or read an offer that burns LP tokens and receives XCH + tokens.
3. Build the on-chain removal spend; optionally aggregate with pending router updates.
4. Either push or write `spend_bundle.json`.

### 2.9 Swap XCH ↔ token

#### 2.9.1 XCH → token (`xch-to-token`)

```bash
python3 tibet.py xch-to-token \
  --asset-id <TAIL_HASH> \
  [--offer <OFFER_OR_PATH>] \
  [--xch-amount <MOJOS>] \
  [--fee <FEE_MOJOS>] \
  [--use-fee-estimate] \
  [--push-tx]
```

- `--asset-id` – TAIL hash of the token.
- `--offer` – existing offer; if provided, no new offer is generated.
- `--xch-amount` – amount of XCH in mojos to swap; must be non-zero when generating a new offer.
- `--fee`, `--use-fee-estimate`, `--push-tx` – same role as above.

The command:

1. Syncs the pair.
2. If generating a new offer:
   - computes expected token output using the AMM formula and `inverse_fee`,
   - prints the expected token amount (human readable),
   - builds and saves an offer.
3. Builds the swap on-chain and either pushes it or writes `spend_bundle.json`.

#### 2.9.2 Token → XCH (`token-to-xch`)

```bash
python3 tibet.py token-to-xch \
  --asset-id <TAIL_HASH> \
  [--offer <OFFER_OR_PATH>] \
  [--token-amount <MOJOS>] \
  [--fee <FEE_MOJOS>] \
  [--use-fee-estimate] \
  [--push-tx]
```

- `--asset-id` – token TAIL hash.
- `--offer` – existing offer or file path.
- `--token-amount` – number of token mojos to swap; required when generating an offer.
- `--fee`, `--use-fee-estimate`, `--push-tx` – same as above.

The command:

1. Syncs the pair.
2. If generating a new offer:
   - computes expected XCH output with the AMM formula and `inverse_fee`,
   - prints the expected XCH amount,
   - builds and saves an offer.
3. Builds the swap on-chain, possibly aggregates additional spends, and either pushes or writes `spend_bundle.json`.

### 2.10 Create pair with initial liquidity

Use this when you want to create the pair and deposit initial liquidity in a single transaction.

```bash
python3 tibet.py create-pair-with-initial-liquidity \
  --asset-id <TAIL_HASH> \
  --offer <OFFER_OR_PATH> \
  --token-amount <TOKEN_MOJOS> \
  --xch-amount <XCH_MOJOS> \
  --liquidity-destination-address <XCH_ADDRESS> \
  [--hidden-puzzle-hash <64_HEX>] \
  [--inverse-fee <INT>] \
  [--push-tx]
```

- `--asset-id` – TAIL hash of the token.
- `--offer` – an existing offer that contributes the XCH and CAT liquidity plus some routing fees (see testing docs for precise amounts).
- `--token-amount` – token liquidity (mojos) to deposit initially.
- `--xch-amount` – XCH liquidity (mojos) to deposit initially.
- `--liquidity-destination-address` – address that will receive the LP tokens.
- `--hidden-puzzle-hash` – if provided, creates an XCH–rCAT pair.
- `--inverse-fee` – optional; must be 993 for normal XCH–CAT pairs; for rCAT pairs, other values are allowed.
- `--push-tx` – if set, pushes the combined deploy+liquidity transaction; otherwise writes `spend_bundle.json`.

The command:

1. Ensures a router is configured (normal or rCAT based on `hidden_puzzle_hash`).
2. Syncs the router and updates `pairs`/`rcat_pairs` in `config.json`.
3. Uses `create_pair_with_liquidity` to construct a single spend that both creates the pair and deposits liquidity to the destination address.
4. Either pushes it or writes `spend_bundle.json`.

### 2.11 Rebase up (`rebase-up`)

For **rCAT pairs only**. This adjusts the effective token supply via an external spend bundle that spends the hidden puzzle hash.

```bash
python3 tibet.py rebase-up \
  --asset-id <TAIL_HASH> \
  --other-sb <OTHER_SB_JSON_PATH> \
  [--offer <OFFER_OR_PATH>] \
  [--token-amount <TOKEN_MOJOS>] \
  [--fee <FEE_MOJOS>] \
  [--use-fee-estimate] \
  [--push-tx]
```

- `--asset-id` – token TAIL hash; must correspond to an rCAT pair.
- `--other-sb` – JSON file holding a spend bundle that spends the hidden puzzle hash and carries the rebase message.
- `--offer` – existing offer; if omitted, one is generated.
- `--token-amount` – amount of tokens to _add_ to the reserve (mojos); required when generating an offer.
- `--fee`, `--use-fee-estimate`, `--push-tx` – same semantics as in other commands.

The command:

1. Reads `other_sb` and loads the rCAT pair information from config.
2. Verifies that the pair uses a hidden puzzle hash (rCAT); fails for normal CAT pairs.
3. Syncs the pair.
4. Generates or reads an offer to pay the rebase cost.
5. Builds a combined spend bundle that includes pair logic and the external message spend (`other_sb`).
6. Either pushes or writes `spend_bundle.json`.

---

## 3. Quick reference (by command)

This section mirrors the overview but in a compact bullet form for copy-paste.

- **`config-node`**

  ```bash
  python3 tibet.py config-node \
    [--chia-root PATH] \
    [--use-sage / --use-sage False] \
    [--network mainnet|testnet] \
    [--use-local-node]
  ```

- **`test-node-config`**

  ```bash
  python3 tibet.py test-node-config
  ```

- **`launch-router`**

  ```bash
  python3 tibet.py launch-router \
    [--rcat] \
    [--fee FEE_MOJOS] \
    [--push-tx]
  ```

- **`set-routers`**

  ```bash
  python3 tibet.py set-routers \
    [--launcher-id HEX] \
    [--rcat-launcher-id HEX]
  ```

- **`launch-test-token`**

  ```bash
  python3 tibet.py launch-test-token \
    --amount TOKEN_SUPPLY \
    [--push-tx] \
    [--hidden-puzzle-hash 64_HEX]
  ```

- **`create-pair`**

  ```bash
  python3 tibet.py create-pair \
    --asset-id TAIL_HASH \
    [--hidden-puzzle-hash 64_HEX] \
    [--inverse-fee INT] \
    [--fee FEE_MOJOS] \
    [--push-tx]
  ```

- **`sync-pairs`**

  ```bash
  python3 tibet.py sync-pairs [--rcat]
  ```

- **`get-pair-info`**

  ```bash
  python3 tibet.py get-pair-info --asset-id TAIL_HASH
  ```

- **`deposit-liquidity`**

  ```bash
  python3 tibet.py deposit-liquidity \
    --asset-id TAIL_HASH \
    [--offer OFFER_OR_PATH] \
    [--token-amount TOKEN_MOJOS] \
    [--xch-amount XCH_MOJOS] \
    [--fee FEE_MOJOS] \
    [--use-fee-estimate] \
    [--push-tx]
  ```

- **`remove-liquidity`**

  ```bash
  python3 tibet.py remove-liquidity \
    --asset-id TAIL_HASH \
    [--offer OFFER_OR_PATH] \
    [--liquidity-token-amount LP_MOJOS] \
    [--fee FEE_MOJOS] \
    [--use-fee-estimate] \
    [--push-tx]
  ```

- **`xch-to-token`**

  ```bash
  python3 tibet.py xch-to-token \
    --asset-id TAIL_HASH \
    [--offer OFFER_OR_PATH] \
    [--xch-amount MOJOS] \
    [--fee FEE_MOJOS] \
    [--use-fee-estimate] \
    [--push-tx]
  ```

- **`token-to-xch`**

  ```bash
  python3 tibet.py token-to-xch \
    --asset-id TAIL_HASH \
    [--offer OFFER_OR_PATH] \
    [--token-amount MOJOS] \
    [--fee FEE_MOJOS] \
    [--use-fee-estimate] \
    [--push-tx]
  ```

- **`create-pair-with-initial-liquidity`**

  ```bash
  python3 tibet.py create-pair-with-initial-liquidity \
    --asset-id TAIL_HASH \
    --offer OFFER_OR_PATH \
    --token-amount TOKEN_MOJOS \
    --xch-amount XCH_MOJOS \
    --liquidity-destination-address XCH_ADDRESS \
    [--hidden-puzzle-hash 64_HEX] \
    [--inverse-fee INT] \
    [--push-tx]
  ```

- **`rebase-up`**

  ```bash
  python3 tibet.py rebase-up \
    --asset-id TAIL_HASH \
    --other-sb OTHER_SB_JSON \
    [--offer OFFER_OR_PATH] \
    [--token-amount TOKEN_MOJOS] \
    [--fee FEE_MOJOS] \
    [--use-fee-estimate] \
    [--push-tx]
  ```
