# TOKENOMICS.md — ZillHa Shards Canonical Spec v2.0

> **Source of truth for all emission, fee, and supply logic.**
> All implementation decisions — in C++, pool software, or game layer — must conform to this document.
> When AI agents disagree, this file wins.

---

## 1 — Coin Identity

| Field            | Value                          |
|------------------|--------------------------------|
| Coin name        | ZillHa Shards                  |
| Ticker           | ZHA                            |
| Atomic unit      | ZEST                           |
| Decimal places   | 13                             |
| ZEST per Shard   | 10,000,000,000,000 (10^13)     |
| Algorithm        | RandomX (CPU-favored, ASIC-resistant, GPU-degraded) |
| Block time       | 30 seconds                     |
| Supply cap       | Infinite (floor-bounded)       |

### Denomination Ladder

| Unit        | Value in Shards       | Value in ZEST          |
|-------------|-----------------------|------------------------|
| Shard       | 1                     | 10,000,000,000,000     |
| Millishard  | 0.001                 | 10,000,000,000         |
| Microshard  | 0.000001              | 10,000,000             |
| Nanoshard   | 0.000000001           | 10,000                 |
| Picoshard   | 0.000000000001        | 10                     |
| ZEST        | 0.0000000000001       | 1 (atomic, indivisible)|

---

## 2 — Block Schedule

```
Blocks per minute : 2
Blocks per hour   : 120
Blocks per day    : 2,880
Blocks per year   : 1,051,200
```

---

## 3 — Emission Schedule

### 3.1 — Year 1: Halvings Every 60 Days

Epoch boundary = 172,800 blocks (60 days × 2,880 blocks/day).

| Epoch | Days      | Block Range               | Base Reward | Epoch Max (×1.2) |
|-------|-----------|---------------------------|-------------|------------------|
| 1     | 0–60      | 0 – 172,799               | 20 ZHA      | 24 ZHA           |
| 2     | 60–120    | 172,800 – 345,599         | 10 ZHA      | 12 ZHA           |
| 3     | 120–180   | 345,600 – 518,399         | 5 ZHA       | 6 ZHA            |
| 4     | 180–240   | 518,400 – 691,199         | 2.5 ZHA     | 3 ZHA            |
| 5     | 240–300   | 691,200 – 864,999         | 1.25 → 3*   | 3 ZHA            |
| 6     | 300–360   | 864,000 – 1,036,799       | 3 ZHA       | 3 ZHA            |

*Floor of 3 ZHA kicks in at epoch 5 — base reward never drops below floor.

### 3.2 — Post Year 1: 10% Decay Every 90 Days

Epoch boundary = 259,200 blocks (90 days × 2,880 blocks/day).

- Base reward decays by 10% each epoch
- Floor of **3 ZHA per block** is permanent — tail emission never stops
- Once floor is reached, base reward holds at 3 ZHA indefinitely

### 3.3 — Daily Emission Reference (Epoch 1, no burn adjustment)

```
Base:      20 ZHA/block × 2,880 blocks/day = 57,600 ZHA/day
Maximum:   24 ZHA/block × 2,880 blocks/day = 69,120 ZHA/day (burn factor 1.2)
Minimum:   16 ZHA/block × 2,880 blocks/day = 46,080 ZHA/day (burn factor 0.8)
```

---

## 4 — Dynamic Emission (Burn-Linked Adjustment)

The block reward floats within each epoch based on real-time burn activity.
All inputs are derived purely from on-chain data — this is consensus-safe.

### 4.1 — Formula

```
burned_ema    = EMA of burned_fees_per_block over last 2,880 blocks (24hr window)
target_burn   = 0.5 × epoch_base_reward
burn_factor   = clamp(0.8, 1.2,  burned_ema / target_burn)
block_reward  = clamp(floor=3, epoch_base × 1.2,  epoch_base × burn_factor)
```

### 4.2 — Equilibrium Logic

| Condition                          | burn_factor | Effect                        |
|------------------------------------|-------------|-------------------------------|
| burn = 50% of base reward          | 1.0         | Emission unchanged            |
| burn > 50% of base reward          | > 1.0       | Emission increases (max ×1.2) |
| burn < 50% of base reward          | < 1.0       | Emission decreases (min ×0.8) |
| No burn activity                   | 0.8         | Emission at floor pressure    |

**Goal condition:** `burned_fees ≈ 0.5 × block_reward`
This is achieved when players are spending approximately as fast as coins are minted.

### 4.3 — EMA Parameters

| Parameter         | Value                                   |
|-------------------|-----------------------------------------|
| Window            | 2,880 blocks (exactly 24 hours at 30s)  |
| Metric            | burned fees per block                   |
| Update frequency  | Every block                             |
| Anti-spike        | EMA smoothing inherently dampens spikes |

### 4.4 — Implementation Notes

- `burn_factor` is recalculated every block from chain data
- Every node independently computes the same EMA from the same chain state
- Reward is not fixed per epoch — it floats within `[floor, epoch_base × 1.2]`
- Miners cannot know exact reward until block is being built (by design)

---

## 5 — Emission Speed Factor (Implementation Critical)

> **Do not change `EMISSION_SPEED_FACTOR_PER_MINUTE` from 20.**

The existing CryptoNote formula in `cryptonote_basic_impl.cpp` contains:

```cpp
const int target_minutes = target / 60;
const int emission_speed_factor = EMISSION_SPEED_FACTOR_PER_MINUTE - (target_minutes - 1);
```

At `DIFFICULTY_TARGET_V2 = 30`:
- `target_minutes = 30 / 60 = 0` (integer division)
- `emission_speed_factor = 20 - (0 - 1) = 21`
- Per-block reward = `MONEY_SUPPLY >> 21`
- Blocks per minute = 2
- Per-minute reward = `MONEY_SUPPLY >> 20` ✅ (identical to Monero per-minute rate)

The `- (target_minutes - 1)` term already makes per-minute emission scale-invariant.
Bumping the factor to 22 would make emission **4× slower** — do not do this.

### Known Bugs at 30s Block Time (Must Patch)

**Bug 1 — Compile-time `static_assert` failure:**
```cpp
// cryptonote_basic_impl.cpp line ~84
// Original: static_assert(DIFFICULTY_TARGET_V2 % 60 == 0, ...)
// Fails at compile time for target=30
// Fix: extend assert to allow sub-minute targets
static_assert(DIFFICULTY_TARGET_V2 == 30 || DIFFICULTY_TARGET_V2 % 60 == 0,
              "V2 target must be 30s or a multiple of 60s");
```

**Bug 2 — Tail emission floor zeroes out:**
```cpp
// Original: base_reward = FINAL_SUBSIDY_PER_MINUTE * target_minutes
// At target=30: target_minutes=0, floor becomes 0 — tail emission never kicks in
// Fix: use sub-minute branch in get_block_reward()
```

**Patch (Option A — preserve per-minute invariants):**
```cpp
const int target = version < 2 ? DIFFICULTY_TARGET_V1 : DIFFICULTY_TARGET_V2;

int emission_speed_factor;
uint64_t tail_floor;

if (target >= 60) {
    const int target_minutes = target / 60;
    emission_speed_factor = EMISSION_SPEED_FACTOR_PER_MINUTE - (target_minutes - 1);
    tail_floor = FINAL_SUBSIDY_PER_MINUTE * target_minutes;
} else {
    // Sub-minute: 30s = 2 blocks/min → shift +1, floor halved
    const int blocks_per_minute = 60 / target;
    int k = 0;
    for (int x = blocks_per_minute; x > 1; x >>= 1) ++k;
    emission_speed_factor = EMISSION_SPEED_FACTOR_PER_MINUTE + k;
    tail_floor = FINAL_SUBSIDY_PER_MINUTE / blocks_per_minute;
}

uint64_t base_reward = (MONEY_SUPPLY - already_generated_coins) >> emission_speed_factor;
if (base_reward < tail_floor) base_reward = tail_floor;
```

> Note: The ZHA custom `get_block_reward()` replaces the above with epoch-based logic,
> but this patch must be the base before layering epoch halvings on top.

---

## 6 — Fee Handling

Transaction fees are NOT paid to the miner. Miners receive only the block reward.
Fees are split between treasury and burn on every block.

### 6.1 — Fee Split

```
total_fees      = sum of all transaction fees in the block
treasury_share  = total_fees × 50%  → coinbase output[1]
burn_share      = total_fees × 50%  → coinbase output[2] → burn address
```

### 6.2 — Coinbase Structure

Every block's coinbase transaction must contain exactly three outputs:

| Index    | Recipient        | Amount                        |
|----------|------------------|-------------------------------|
| output[0]| Miner address    | full block_reward (ZHA)       |
| output[1]| Treasury address | 50% of total block tx fees    |
| output[2]| Burn address     | 50% of total block tx fees    |

**The block reward itself is never split.** Miners receive 100% of the block reward.
Only fees are split. This preserves mining incentives through the halving schedule.

### 6.3 — Burn Address

| Field     | Value                                              |
|-----------|----------------------------------------------------|
| Type      | Provably unspendable (zero spend key)              |
| Method    | Designated burn address hardcoded in config        |
| Auditable | Yes — all received outputs visible on-chain        |
| EMA input | All outputs to burn address count toward burned_ema|
| Address   | `BURN_ADDRESS_PLACEHOLDER` (set before mainnet)    |

### 6.4 — Treasury Address

| Field     | Value                                              |
|-----------|----------------------------------------------------|
| Type      | Standard wallet address                            |
| Method    | Coinbase output hardcoded in block construction    |
| Address   | `TREASURY_ADDRESS_PLACEHOLDER` (set before mainnet)|

---

## 7 — Difficulty Adjustment

| Parameter  | Value                                         |
|------------|-----------------------------------------------|
| Algorithm  | LWMA-3                                        |
| Window     | 60 blocks (~30 minutes at 30s blocks)         |
| Target     | 30 seconds                                    |
| Rationale  | LWMA-3 handles small networks and variable hashrate well — ideal for game pool with players connecting/disconnecting |

---

## 8 — Decimal Point Implementation

> **Non-standard: 13 decimal places. Requires code changes beyond config.**

### 8.1 — Constants

```cpp
#define CRYPTONOTE_DISPLAY_DECIMAL_POINT  13
#define COIN                              ((uint64_t)10000000000000) // 10^13 ZEST = 1 Shard
```

### 8.2 — `set_default_decimal_point` Validator

File: `src/cryptonote_basic/cryptonote_format_utils.cpp`

The validator whitelist must include 13:
```cpp
case 13:  // ZillHa Shards: shard
case 12:
case 9:
case 6:
case 3:
case 0:
  break;
```

### 8.3 — Unit Name Mapping

```cpp
// get_unit() — decimal point → display name
case 13: return "shard";
case 12: return "millishard";
case 9:  return "microshard";
case 6:  return "nanoshard";
case 3:  return "picoshard";
case 0:  return "zest";
```

### 8.4 — simplewallet `set unit` Parser

```cpp
if      (unit == "shard")       decimal_point = 13;
else if (unit == "millishard")  decimal_point = 12;
else if (unit == "microshard")  decimal_point = 9;
else if (unit == "nanoshard")   decimal_point = 6;
else if (unit == "picoshard")   decimal_point = 3;
else if (unit == "zest")        decimal_point = 0;
```

### 8.5 — `uint64_t` Headroom Check

```
uint64_t max = 18,446,744,073,709,551,615
1 Shard     = 10,000,000,000,000 ZEST
Max Shards  = 18,446,744,073,709,551,615 / 10,000,000,000,000
            = ~1,844,674 Shards representable in uint64_t
```

At 3 ZHA floor emission: ~600+ years before overflow. Safe.

---

## 9 — Network Identity

### 9.1 — NETWORK_ID Bytes

All three network IDs encode `"ZILLHASHARDS"` in the first 12 bytes
plus a 4-byte network discriminator. Replaces the Battleground `"BG"` encoding.

```cpp
// Mainnet
{ 'Z','I','L','L','H','A','S','H','A','R','D','S', 0x00,0x00,0x00,0x01 }

// Testnet
{ 'Z','I','L','L','H','A','S','H','A','R','D','S', 0x00,0x00,0x00,0x02 }

// Stagenet
{ 'Z','I','L','L','H','A','S','H','A','R','D','S', 0x00,0x00,0x00,0x03 }
```

### 9.2 — Ports

| Network  | P2P   | RPC   | ZMQ   |
|----------|-------|-------|-------|
| Mainnet  | 19090 | 19091 | 19092 |
| Testnet  | 29090 | 29091 | 29092 |
| Stagenet | 39090 | 39091 | 39092 |

### 9.3 — Address Prefixes

| Network  | Standard | Integrated | Subaddress |
|----------|----------|------------|------------|
| Mainnet  | 66       | 67         | 68         |
| Testnet  | 96       | 97         | 108        |
| Stagenet | 120      | 121        | 132        |

---

## 10 — Binary Names

| Old Name              | New Name                  |
|-----------------------|---------------------------|
| `monerod`             | `zillhad`                 |
| `monero-wallet-cli`   | `zillha-wallet-cli`       |
| `monero-wallet-rpc`   | `zillha-wallet-rpc`       |

---

## 11 — Off-Chain Layer (Pool + Game)

These are NOT consensus — they live in pool software and game server only.

### 11.1 — Pool

| Feature          | Detail                                              |
|------------------|-----------------------------------------------------|
| Distribution     | Configurable (PPLNS recommended)                   |
| Anti-exploit     | Rate limiting, share validation, anomaly detection  |
| Player modifier  | Adjust payout shares by active player count (pool-side only, not consensus) |

### 11.2 — Game Economy Sinks

| Sink              | Type      |
|-------------------|-----------|
| World entry cost  | Primary   |
| Retry costs       | Secondary |
| Branch unlocks    | Secondary |
| Cosmetics         | Secondary |
| Upgrades          | Secondary |

### 11.3 — Analytics (Not Consensus)

Tracked off-chain for game health monitoring only:
- Active players
- Sessions per day
- Tokens spent
- Tokens earned

---

## 12 — Pre-Launch Checklist

- [ ] `CRYPTONOTE_NAME` = `"zillha-shards"`
- [ ] `CRYPTONOTE_DISPLAY_DECIMAL_POINT` = `13`
- [ ] `COIN` = `10000000000000` (10^13)
- [ ] `DIFFICULTY_TARGET_V2` = `30`
- [ ] `DIFFICULTY_TARGET_V1` = `30`
- [ ] `DIFFICULTY_WINDOW` = `2880`
- [ ] `static_assert` patched for 30s block time
- [ ] `get_block_reward()` sub-minute tail floor patched
- [ ] Custom epoch halving logic implemented
- [ ] Burn-linked EMA dynamic adjustment implemented
- [ ] Fee split coinbase (3 outputs) implemented
- [ ] `set_default_decimal_point` validator updated (add 13)
- [ ] `get_unit()` updated with ZHA denomination ladder
- [ ] `simplewallet` `set unit` parser updated
- [ ] NETWORK_ID re-encoded as `ZILLHASHARDS` on all three networks
- [ ] Binary names updated in CMakeLists (`zillhad` etc.)
- [ ] `CRYPTONOTE_NAME` data dir = `".zillha-shards"`
- [ ] Burn address generated and hardcoded
- [ ] Treasury address generated and hardcoded
- [ ] Genesis block generated (stagenet first, then mainnet)
- [ ] All `XMR` / `monero` / `BGD` string literals replaced
- [ ] CI green on all platforms

---

*ZillHa Shards — TOKENOMICS.md v2.0 — April 2026*
