# Reward System Architecture (Bio-Proof-of-Brain Mining)

**Date:** 2026-04-05
**Source:** conversation context

## Key Findings

- The concept is **Bio-Proof-of-Brain Mining** — users earn tokens by achieving measurable bio states via a BCI headband (Neiry/Capsule SDK).
- Hashrate is not binary — it is a **continuous weighted aggregate** of multiple bio signals, each implemented as an independent `HashRateContributor`.
- Emission follows a **Bitcoin-like schedule** (21M tokens, halvings), compressed to 15 years by adjusting initial reward and halving interval to match BNB Smart Chain block rate (~3s/block ≈ 157M blocks in 15 years).
- The smart contract uses a **lazy mint / claim pattern** — the backend Oracle tracks claimable balances in Postgres; the contract only mints when the user calls `claim()`.
- `neiry_kit` (Flutter plugin wrapping Capsule C SDK) already provides all the raw bio signal streams needed — the reward system builds *on top of* it, not inside it.

## Details

### Three Independent Layers

```
[bio signals]  →  HashRate Engine  →  Emission Engine  →  Smart Contract
neiry_kit          (mobile/api)        (backend/Postgres)    (BNB chain)
```

Each layer is decoupled — Emission does not know about EEG; HashRate does not know about tokens.

### HashRate Engine

**`HashRateContributor` interface:**
```
interface HashRateContributor {
  id:      string          // "hrv", "eeg_alpha", "consistency_7d"
  weight:  float           // configurable, not hardcoded
  compute(ctx: BioContext) → {
    score:   float   // 0..1 — how well the user is performing
    quality: float   // 0..1 — signal reliability (artifact presence)
    hint?:   string  // coaching hint for feedback engine
  }
}
```

**`BioContext`** — epoch snapshot of all signals:
- EEG bands: alpha, beta, theta, delta, smr
- Physio: relaxation, concentration, stress, fatigue
- Cardio: HRV (current + baseline), heartRate, kaplanIndex, stressIndex
- Emotions: attention, cognitiveLoad, selfControl
- Behavioral: streak_days, session_count
- Baselines: per-user calibrated norms (Map<string, float>)

**Aggregation formula:**
```
hashrate = Σ(score × weight × quality) / Σ(weight)
final_hashrate = hashrate × streak_bonus(streak_days)
```

Adding a new signal = writing a new `HashRateContributor`. No other code changes.

### Feedback Engine

Lives alongside HashRate Engine. Reads `ContributorResult[]` from the same epoch.

- Priority = `(1 - score) × weight` — worst-performing high-weight contributors surface first.
- Each contributor owns its own `hint` string — Feedback Engine only sorts and formats.
- Output: `FeedbackMessage { priority, target, message, delta }` → shown in mobile UI.

### Emission Engine (backend)

**Schedule parameters** (computed once at deploy):
- `total_supply`: 21,000,000
- `target_years`: 15
- `chain_block_time`: 3s (BNB Smart Chain) → 157M blocks in 15 years
- `initial_reward` and `halving_interval` derived from above

**Epoch**: one block per BNB Smart Chain (~3s). No polling — backend subscribes once via WebSocket `newBlockHeaders`.

**EpochManager** (NestJS, event-driven):
```
web3.eth.subscribe('newBlockHeaders', (block) => {
  reward = EmissionSchedule.rewardAt(block.number)
  distribute(reward, active_users)
  push to clients via gRPC stream
})
```
1. Receive block push from chain
2. Compute `reward = EmissionSchedule.rewardAt(block.number)`
3. Distribute: `user_reward = reward × (user_hashrate / total_hashrate)`
4. Write to `RewardLedger` (Postgres)
5. Push update to active mobile clients via gRPC stream

Mobile client displays what it receives from the stream — single source of truth, no local estimation.

**RewardLedger** (Postgres):
```
userId → claimable_balance   // accumulates
userId → claimed_balance     // already withdrawn to chain
```

### Smart Contract (BNB Smart Chain)

Minimal contract — only `claim()` logic:
```solidity
contract MindToken {
  mapping(address => uint256) public claimable;  // written by Oracle (backend)

  function claim() external {
    uint256 amount = claimable[msg.sender];
    claimable[msg.sender] = 0;
    _mint(msg.sender, amount);
  }

  function setClaim(address user, uint256 amount) external onlyOracle {
    claimable[user] = amount;  // backend batches this ~hourly
  }
}
```

Backend acts as **Oracle** — sends batched `setClaim` transactions periodically (e.g. hourly). Gas is paid only on user-initiated `claim()`.

### Full Data Flow

```
[headband] → neiry_kit → BioContext
                              ↓
                   HashRateContributors[]
                              ↓
                   HashRateAggregator → hashrate: float
                        ↙          ↘
             FeedbackEngine       mind_api
             (→ mobile UI)        (EpochManager, cron)
                                        ↓
                                  RewardLedger (Postgres)
                                        ↓ (batch, ~hourly)
                                  MindToken.setClaim(user, amount)
                                        ↓ (on user request)
                                  claim() → tokens to wallet
```

### Architecture Decisions

| Question | Decision | Rationale |
|---|---|---|
| Where is hashrate computed? | Mobile client, sent to server | Raw bio data doesn't leave the device |
| Where is emission tracked? | Postgres (backend) | Cheap, reliable, flexible |
| When are tokens minted? | Only on user `claim()` | No gas cost for passive accumulation |
| How to extend with new signals? | New `HashRateContributor` | Zero impact on existing code |
| Where does HashRateEngine live? | Backend — mobile streams low-frequency time series (~1s intervals, aligned with block time), server computes hashrate | Formulas stay server-side (easy to update without app release), server sees artifacts and makes its own decisions |
| How to verify headband data? | PAT + signed session | Can't fake hashrate without device |

### Where neiry_kit Fits

`neiry_kit` is the **leaf** — it provides raw streams. The reward system does not modify it.

Available streams from `neiry_kit` that feed into `BioContext`:
- `NfbClassifier.stateStream` → alpha, beta, theta, delta, smr
- `PhysioClassifier.stateStream` → relaxation, concentration, fatigue, stress
- `EmotionsClassifier.stateStream` → attention, cognitiveLoad, selfControl
- `CardioClassifier.stateStream` → heartRate, stressIndex, kaplanIndex
- `ProductivityClassifier.indexesStream` + `metricsStream`

Current status of `neiry_kit`: Dart API complete, iOS bridges partially done (DeviceLocator ready, Device/Classifiers pending), Android bridges not started. The reward system can be designed now; hardware integration follows when bridges are ready.

## Resolved Decisions

### Anti-fraud
Full prevention is impossible without trusted hardware. Practical approach for MVP:
- Send a **low-resolution time series** (e.g. values every 10 seconds) rather than aggregates — a realistic EEG/HRV time series is hard to fabricate convincingly (noise, artifacts, inter-channel correlations, non-constant alpha).
- **Server-side anomaly scoring** — flag statistical outliers across the user population (e.g. alpha=0.95 constant for 60 min, or physiologically impossible transitions).
- Explore whether Capsule SDK can **cryptographically sign sessions** from firmware — would be strong hardware-level verification if available.
- Accept that some fraud will exist; raise its cost rather than eliminate it.

### Baseline Calibration
Baseline is not a one-time calibration — it is a **living user model** (`UserBioProfile`):
```
UserBioProfile {
  hrv_baseline:        rolling average (last 30 days)
  eeg_alpha_baseline:  individual norm (NFBCalibrator + accumulation)
  resting_heart_rate:  adaptive norm
  progress_curve:      how metrics change over time
}
```
- Initial calibration: first 3–5 sessions using Capsule SDK's `NFBCalibrator`
- Ongoing: rolling window — baselines drift with user's progress
- `HashRateContributor` compares `(current - baseline) / baseline`, not absolute values

**Coaching Engine** is a separate system above `UserBioProfile`: selects exercises, analyzes video (pose quality), and updates the profile. It feeds progress back into hashrate organically — better technique → better baseline → higher hashrate ceiling. This is a large independent feature, well-isolated from the core reward system.

### Mobile Wallet Binding
Simple flow: user provides their blockchain address, backend mints and sends tokens to it.
- Add **proof of ownership**: ask user to sign a standard message with their wallet (`eth_sign`) before saving the address. Prevents accidentally entering someone else's address.
- No in-app wallet required — external wallet (MetaMask, Trust Wallet, etc.) is sufficient.
- In-app wallet is a separate product decision (lowers onboarding barrier) but not architecturally required.

### Difficulty Adjustment
Pure halvings create **reward dilution** at scale — more users = less reward per person, which breaks long-term retention.

Chosen approach: **Dynamic Difficulty Ceiling**
- `max_daily_hashrate` is not a fixed number — it is derived from the emission schedule and target audience size:
  ```
  max_daily_hashrate = total_epoch_reward / target_active_users
  ```
- As user count grows, the cap scales down proportionally — so the absolute reward for an average honest user stays stable.
- A user's share is determined by quality, not hours-online. Prevents "meditate 24/7 to dominate" scenarios.
- Combined with halvings: emission shrinks on schedule, cap adjusts to maintain per-user predictability.

A **reward floor** (guaranteed minimum per active session) may be added later to protect retention at scale, but is not required for MVP.

## Open Questions

1. **Trusted sensor pipeline**: Each signal source has a different trust level — BCI headband (may support firmware signing), heart rate monitor (likely no signing), video (self-evidencing but deepfakeable), phone (rootable). This is a broad research topic that cannot be resolved without deep investigation during development. No decision yet.
