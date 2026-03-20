---

## Adversarial Defense & Anti-Spoofing Strategy

> **"We don't verify location — we verify reality."**

Our platform assumes adversaries are sophisticated, coordinated, and adaptive. A 500-user fraud ring faking GPS locations during cyclones isn't theoretical — it's the exact scenario we architect against. Below is our multi-layered defense system, codenamed **Shield Mesh**.

---

### 1. Differentiation Engine — _Proof-of-Presence Protocol (PoP)_

The core challenge: distinguishing a delivery partner genuinely stranded in a monsoon from a spoofer sitting indoors triggering payouts.

**We verify _presence coherence_ across independent, orthogonal signals — not GPS alone.** The system is designed so that successful fraud requires physical presence in the claimed conditions, eliminating the economic advantage of remote spoofing entirely.

#### Behavioral Biometric Modeling

Every genuine user generates a unique **motion fingerprint** — a time-series signature from accelerometer jitter, gyroscope drift, and screen interaction cadence. A rider on a flooded road has a fundamentally different sensor profile than someone stationary indoors.

- **Model**: Sequence classifier producing a 128-dim session embedding, clustered in latent space to flag outliers.
- **Why it works**: Behavioral patterns are high-dimensional and involuntary — orders of magnitude harder to fake than a GPS coordinate.
- **Progressive complexity**: In early deployment, tree-based models approximate sequence signals before transitioning to deep models (LSTM) as labeled data scales.

#### Graph-Based Ring Detection — _Collusion Graph Analysis_

Coordinated rings share infrastructure. We build a **dynamic heterogeneous graph** connecting users through:

| Edge Signal                      | What It Catches                          |
| -------------------------------- | ---------------------------------------- |
| Device fingerprint overlap       | Shared or factory-reset devices          |
| Network co-location (cell/Wi-Fi) | Same physical infrastructure             |
| Temporal claim clustering        | Claims filed within tight time windows   |
| Behavioral embedding similarity  | Suspiciously identical motion signatures |

We run **community detection** (Louvain + temporal PageRank) on the active claims graph. When a cluster exceeds a configurable correlation threshold, the entire subgraph escalates — not individual users. Graph features are computed incrementally, enabling near real-time ring detection without full graph recomputation.

**Why this defeats rings**: Individual spoofers can mimic a single genuine user. But hundreds of users claiming simultaneously from the same cell infrastructure, with identical motion signatures and recently reset devices — that's a topology, not a coincidence. Per-user models miss structural patterns; graph analysis catches them.

#### Temporal Consistency — _Spatiotemporal Narrative_

Every claim is validated against a three-window coherence check:

- **Pre-event**: Did the user have a plausible delivery route leading _into_ the weather zone?
- **During-event**: Does sensor data show weather-consistent patterns (barometric drop, ambient noise, motion)?
- **Post-event**: Did movement resume naturally, or did GPS teleport to a baseline location?

Genuine users have a coherent narrative across all three windows. Spoofers nearly always break it.

---

### 2. Data Intelligence Layer — _Weak Signal Fusion_

No single signal catches a sophisticated spoofer. We fuse **12 orthogonal channels** into a calibrated fraud score, classified by reliability and availability:

#### Signal Classification

| Signal                      | Tier     | Spoofability | Notes                                        |
| --------------------------- | -------- | ------------ | -------------------------------------------- |
| GPS coordinates             | **Core** | Easy         | Baseline; never trusted alone                |
| Cell tower triangulation    | **Core** | Hard         | Cross-validates GPS via carrier API          |
| Delivery platform order log | **Core** | Very Hard    | Was user actually working at claim time?     |
| Historical claim frequency  | **Core** | N/A          | Actuarial baseline per user                  |
| Accelerometer signature     | Optional | Hard         | Micro-vibrations: riding vs. stationary      |
| Barometric pressure         | Optional | Very Hard    | Cross-checked against weather station data   |
| Battery drain rate          | Optional | Very Hard    | Outdoor storm use → measurably higher drain  |
| Screen interaction pattern  | Optional | Hard         | Stress patterns differ during emergencies    |
| Network latency profile     | Optional | Medium       | Degraded latency in flood-affected areas     |
| **Ambient acoustic hash**   | Optional | Very Hard    | Novel — see below                            |
| Wi-Fi BSSID fingerprint     | Fallback | Medium       | Maps BSSIDs to known physical zones          |
| BLE peer proximity mesh     | Fallback | Hard         | Mutual visibility among co-located claimants |

> **Graceful degradation**: The system adapts to device capability and permission availability. A budget phone without a barometer isn't penalized — available signals are weighted proportionally higher. If Bluetooth or microphone permissions are denied, those channels are excluded from scoring. BLE signals in particular are opportunistic and only used when available; absence is never penalized. No single optional signal is a hard requirement.

#### Fusion Architecture

Individual signal classifiers feed into a **stacking ensemble** (gradient-boosted meta-learner) trained on historically adjudicated claims, outputting a calibrated `fraud_score ∈ [0, 1]`:

| Threshold     | Action                               |
| ------------- | ------------------------------------ |
| `< 0.15`      | Auto-approve                         |
| `0.15 – 0.65` | Soft verification (single challenge) |
| `0.65 – 0.90` | Manual review queue                  |
| `> 0.90`      | Auto-hold + ring analysis trigger    |

The system **minimizes false positives at baseline fraud volumes** and dynamically tightens thresholds when coordinated attack patterns are detected. Thresholds are continuously tuned based on live fraud pressure and payout exposure.

#### Novel Signal: Ambient Acoustic Hash

During claim submission, we request a **3-second ambient audio capture** — our standout unconventional signal.

**How it works**:

1. Compute mel-spectrogram on-device → extract a 64-dim spectral embedding via environmental sound classifier (ESC-50 backbone).
2. Compare against expected acoustic profile for the claimed weather event and location type.
3. A user claiming cyclone exposure whose embedding matches "indoor silence" receives a significant fraud score increase.

**Privacy**: All processing happens **on-device**. Raw audio never leaves the phone — only the compact embedding is transmitted. No speech content is captured or stored.

**Robustness**:

- Capture timing is **randomized** — not predictable by adversaries.
- **Liveness validation** via noise entropy: pre-recorded audio exhibits detectable spectral regularity.
- Signal is **optional, not mandatory** — permission denial simply removes this channel from scoring.

---

### 3. UX and Fairness Layer — _Trust Scoring_

Fraud detection is useless if it creates a hostile experience for honest users. Our system ensures the vast majority of legitimate claimants face **zero friction**, while flagged claims get a fair, transparent process.

#### Tiered Claim Flow

```
  CLAIM SUBMITTED
        │
   Shield Mesh Scoring (<200ms)
        │
   ┌────┼────────────────┐
   ▼    ▼                ▼
 GREEN   AMBER            RED
 Auto-pay  Soft Challenge    Manual Review
 <5 min    (single step)     Hold + Notify
```

#### Soft Challenge (AMBER)

Instead of blocking, we issue a **single low-friction challenge** — randomized from 15+ variants to prevent adversarial memorization:

- Quick photo of surroundings (EXIF metadata + weather correlation validated)
- Last 3 delivery order IDs (cross-referenced via partner API)
- Tap-to-confirm (response latency is itself a signal — automated scripts respond too fast)

#### Persistent Trust Score

Every user carries a `trust_score` (0–100) that evolves over time:

| Event                       | Impact        |
| --------------------------- | ------------- |
| Verified legitimate claim   | +3            |
| Passed soft challenge       | +5            |
| Consistent delivery history | +1/month      |
| Failed challenge            | −15           |
| Linked to flagged ring      | −30           |
| Successful appeal           | +10 (restore) |

Users with `trust_score > 75` **skip soft challenges entirely** — rewarding long-term honest behavior and creating natural incentive alignment.

#### Edge Case: Poor Connectivity

Extreme weather degrades networks. We handle this without penalizing users:

- **Offline claim queuing**: App caches claim data locally (GPS, sensors, timestamp), cryptographically signed with device key + timestamp to prevent post-hoc tampering. Submits when connectivity resumes.
- **Grace window**: Claims within 6 hours of a verified weather event accepted with relaxed thresholds.
- **Degraded mode**: Incomplete sensor data causes available signals to be weighted higher — claims are never rejected due to missing optional data.

#### Appeal System

Every held or denied claim triggers:

1. **Plain-language explanation**: "Your claim was flagged because [signal summary]."
2. **One-tap appeal** with option to submit additional evidence (photos, delivery app screenshots).
3. **4-hour SLA** for human review with full signal context.
4. **Ombudsman escalation** for persistent disputes.

---

### 4. Privacy and Compliance

Collecting device signals demands rigorous privacy discipline. Our architecture is **privacy-by-design**:

- **On-device feature extraction**: Audio, accelerometer, and sensor data are processed locally. Raw data **never leaves the device** — only derived feature vectors are transmitted.
- **No raw data storage**: Only computed embeddings and scores are persisted server-side.
- **Explicit consent gating**: Each signal source (audio, BLE, sensors) requires granular user permission. Denial removes that channel — it never blocks claims.
- **Data retention limits**: Feature vectors auto-expire after 90 days. Adjudicated claim metadata retained for model training with anonymization.
- **Audit transparency**: All sensitive computations are designed to be verifiable via audit logs without exposing raw user data.
- **Compliance-ready**: Architecture aligns with DPDPA (India) and GDPR data minimization principles.

---

### 5. System Architecture — _Shield Mesh Pipeline_

```
 INGEST              EXTRACT              MODEL               DECIDE            ACT
 ───────            ─────────            ──────              ───────           ─────
 GPS, Sensors  ──▶  Motion embeddings ──▶ Per-user scorer  ──▶ GREEN/AMBER/ ──▶ Auto-pay
 Network data       Audio hash            (Ensemble)          RED routing      Challenge
 Delivery logs      Graph features        Ring detector                        Hold
 Audio embedding    Temporal features     (GNN + Louvain)     Trust score      Escalate
                                          Temporal model      integration
                                               │
                                    ┌──────────▼──────────┐
                                    │    FEEDBACK LOOP     │
                                    │  Adjudicated claims  │
                                    │  retrain weekly +    │
                                    │  drift detection     │
                                    └─────────────────────┘
```

**Key design decisions**:

- **Streaming ingestion** (Kafka): Signals arrive asynchronously; the system scores progressively as data becomes available.
- **Feature store** (Redis + PostgreSQL): Precomputed user history served at sub-100ms latency.
- **Model serving** (ONNX Runtime): All models run inference in **<200ms per claim**.
- **Feedback loop**: Every adjudicated claim feeds the training set. Weekly retraining with automated KL-divergence drift detection to catch distribution shifts early.

---

### 6. Adversarial Thinking — _Red Team Playbook_

We maintain an internal **Red Team Protocol** — simulating adversary evolution and patching defenses preemptively.

#### Attack Evolution Matrix

| Gen | Attack                             | Our Defense                                                            |
| --- | ---------------------------------- | ---------------------------------------------------------------------- |
| 1   | Basic GPS spoofing                 | Cell tower triangulation + GPS consistency                             |
| 2   | Spoofing + fake accelerometer      | Barometric cross-validation + acoustic hash                            |
| 3   | Mule drops phone in storm zone     | Behavioral biometrics (interaction pattern doesn't match device owner) |
| 4   | Distributed ring with real devices | Graph-based ring detection + BLE mesh                                  |
| 5   | Adversarial ML (crafted inputs)    | Ensemble diversity + model rotation + perturbation detection           |

#### Adaptive Defenses

- **Model rotation**: Multiple model variants trained on different feature subsets, rotated unpredictably — adversaries cannot optimize against a static target.
- **Honeypot signals**: Synthetic "easy exploit" patterns injected periodically. Users who consistently trigger them reveal adversarial intent.
- **Canary claims**: Fake profiles seeded into fraud channels. If canary identities start filing claims, we know system details have been leaked to adversaries.
- **Drift detection**: KL-divergence monitoring on feature distributions triggers automated retraining and human review when input patterns shift unexpectedly.

---

### Case Study: Detecting _Operation Monsoon Phantom_

> **Scenario**: 47 users file weather-impact claims within a 12-minute window during Cyclone Kedar, all claiming to be stranded in South Mumbai.
>
> | Signal         | Finding                                                                                                                 |
> | -------------- | ----------------------------------------------------------------------------------------------------------------------- |
> | **Graph**      | 43 of 47 share Wi-Fi BSSIDs traced to a co-working space in Andheri — 30 km from claimed location                       |
> | **Temporal**   | Zero delivery route history in South Mumbai in the preceding 24 hours                                                   |
> | **Acoustic**   | 39 audio hashes cluster around an "indoor office" profile — inconsistent with outdoor cyclone conditions                |
> | **BLE Mesh**   | Despite claiming co-location in a flood zone, zero mutual Bluetooth visibility across all claimant pairs                |
> | **Behavioral** | Accelerometer data shows stationary, low-jitter patterns — consistent with phones on desks, not held by stranded riders |
>
> **Result**: Ring flagged in **3 minutes**. All 47 claims routed to RED. 4 legitimate users who happened to claim in the same window were auto-cleared by high trust scores (82, 91, 77, 88) and consistent pre-event delivery trajectories.
>
> **Impact**: **Rs. 2.3L in fraudulent payouts prevented. Zero legitimate users impacted.**

---

### Target Performance Metrics

| Metric                        | Target     | Rationale                                    |
| ----------------------------- | ---------- | -------------------------------------------- |
| Fraud detection rate (recall) | > 95%      | Catch nearly all coordinated ring activity   |
| False positive rate           | < 2%       | Honest users must not be blocked             |
| Claim scoring latency         | < 200ms    | Real-time UX with no perceived delay         |
| Ring detection latency        | < 5 min    | Resolution window before payouts trigger     |
| Cost per claim evaluation     | < Rs. 0.50 | Sustainable unit economics at scale          |
| Appeal resolution SLA         | < 4 hours  | Maintain user trust and platform credibility |

---

### Deployment Phases

| Phase                      | Scope                                                                                  | Timeline  |
| -------------------------- | -------------------------------------------------------------------------------------- | --------- |
| **Phase 1 — MVP**          | GPS + cell tower triangulation + delivery log cross-ref + rule-based anomaly detection | Week 1–4  |
| **Phase 2 — Intelligence** | Behavioral modeling + graph-based ring detection + trust scoring                       | Month 2–3 |
| **Phase 3 — Advanced**     | Acoustic hash + BLE mesh + adversarial model rotation + full feedback loop             | Month 4–6 |

Each phase is independently deployable and delivers incremental fraud reduction. Phase 1 alone addresses the majority of unsophisticated spoofing attempts. In early phases, simpler tree-based models approximate sequence and graph signals before transitioning to deep architectures as labeled data accumulates.

---

> **Our system doesn't try to detect fraud — it makes large-scale fraud coordination economically unviable and detectable with high confidence.**

---
