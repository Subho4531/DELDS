# D-SHIELD — Build Plan (NIDS Core)

Decentralized Self-Healing Intelligent Event Learning Defense.
Research build plan for a 3-tier PQC-secured federated NIDS.

## 1. Topology

```
                        ┌────────────┐
                        │  Central   │
                        │   Node     │
                        └─────┬──────┘
        ┌───────────┬─────────┼─────────┬───────────┐
        │            │        │          │           │
   ┌────▼───┐   ┌────▼───┐┌───▼────┐┌────▼───┐   (4 submains)
   │Submain1│   │Submain2││Submain3││Submain4│
   └─┬┬┬┬───┘   └─┬┬┬┬───┘└─┬┬┬┬───┘└─┬┬┬┬───┘
     ││││          ││││       ││││       ││││
     C C C C        C C C C    C C C C    C C C C   (4 children each = 16 leaves)
```

- **1 central node** — global model aggregator + consensus coordinator.
- **4 submain nodes** — regional aggregators; each owns 4 child nodes.
- **16 child nodes** — traffic sources / local sensors, each runs a local model.
- **Total: 21 Mininet hosts** (1 + 4 + 16), plus 1+ attacker host injected at the leaf or submain layer depending on scenario.

## 2. Security model

PQC (ML-KEM for key exchange, ML-DSA for signatures) secures **every** link:

- **Child ↔ Submain**: each child does a PQC handshake with its submain before sending traffic features or model updates. Submain authenticates each child via ML-DSA signature on a registered public key.
- **Submain ↔ Central**: same handshake pattern one layer up. This is the higher-value link (carries aggregated FL updates for all 4 children), so it's the one to get right first per your original instinct — build and test this pair before replicating child↔submain.
- Build order: get **1 submain ↔ 1 child** working end-to-end, then **1 submain ↔ central**, then scale out to the full 4×4 tree. Don't build all 21 nodes' crypto at once — the pairwise protocol is identical, only fan-out changes.

## 3. Two-tier FL flow

```
Child (local model, local traffic)
   │  train on local tshark features
   ▼
Local model update (weights/gradients)
   │  PQC-encrypted + signed
   ▼
Submain: aggregates 4 children's updates (FedAvg or similar)
   │  PQC-encrypted + signed
   ▼
Central: aggregates 4 submains' updates → global model
   │
   ▼
Global model broadcast back down (submain → child) for next round
```

Each child also runs **local inference** continuously between FL rounds — FL improves the shared model over time, but detection/response (Section 5) runs locally in real time and doesn't wait for a round to complete.

## 4. Attack scenarios to simulate (per earlier attack diagram)

| Attack | Injected at | Detection signal |
|---|---|---|
| Node compromise | child or submain | node behavior deviates from FL update norms (e.g. outlier gradients), or drops/modifies forwarded traffic |
| Replay | any link | ML-KEM session freshness / nonce reuse check fails |
| MITM | child↔submain or submain↔central | PQC signature/identity mismatch, unexpected intermediate hop |
| DoS / flooding | any leaf | packet-rate spike, same-source flood, feature extraction shows abnormal volume |

## 5. Local detect → consensus → act pipeline (per node)

```
Traffic capture (tshark) → Feature extraction → ML inference (local model)
   │
   ├─ No alert → continue monitoring
   └─ Alert → Validate severity → Local action: block
                 → Broadcast alert to peer submains / central
                 → Consensus: if 2f+1 of {4 submains, central} agree → mark node malicious
                 → Global action: isolate / revoke / trigger recovery path
```

Consensus quorum is computed over the **5 aggregator-tier nodes** (4 submains + central) — children don't vote, they only report. This keeps the BFT quorum small and tractable (f=1, need 3 of 5 to agree) instead of trying to get consensus across 21 nodes.

## 6. Build phases

### Phase 0 — Environment
- Mininet topology script: 1 central + 4 submain + 16 children, custom tree topology (not default Mininet tree, since fan-out differs at each level).
- tshark capture + filtering + CSV export pipeline running on each host.

### Phase 1 — PQC channel (submain ↔ child, single pair first)
- Integrate `liboqs`/`oqs-provider` for ML-KEM handshake + ML-DSA signing.
- One child, one submain: handshake → encrypted session → basic message exchange.
- Test: replay captured handshake, confirm it's rejected (freshness/nonce check).

### Phase 2 — Scale security to full tree
- Replicate Phase 1 pairwise protocol: all 16 child↔submain pairs, all 4 submain↔central pairs.
- Latency benchmark: classical (plaintext/TLS) vs PQC handshake, at each tier — this directly answers your "performance comparison" research question.

### Phase 3 — Local ML detection
- Feature extraction from tshark CSVs (packet rate, size, MAC/IP pairs, timing).
- Train Random Forest per-child on labeled traffic (normal + injected attacks from Section 4).
- Local inference loop wired to the detect→block pipeline (Section 5), latency-checked (RF, not LSTM, for the live path).

### Phase 4 — Federated learning loop
- Implement FedAvg-style aggregation at submain (4 children) and central (4 submains).
- PQC-encrypt/sign every model update in transit.
- Sanity-bound aggregation inputs (reject outlier updates) so a compromised child can't poison the global model — this is the guard for the feedback-loop risk flagged earlier.

### Phase 5 — Consensus + response
- Implement 2f+1 voting among the 5 aggregator-tier nodes.
- Wire consensus result → isolate/revoke action.
- Define re-trust path: how an isolated node re-registers (new PQC keypair, probation period with reduced trust weight in future votes).

### Phase 6 — Attack simulation + evaluation
- Script each attack from Section 4 against the full tree.
- Collect metrics: detection accuracy / false-positive rate, time-to-detect, time-to-heal, PQC vs classical latency overhead, FL convergence with/without malicious updates.
- Generate report graphs (Pandas + Matplotlib) — matches your original reporting pipeline.

## 7. Open items to resolve during build (not blocking start)
- Exact FedAvg weighting (equal per child vs weighted by traffic volume).
- Whether re-trust after isolation requires central approval or submain-local approval.
- Attack injection tooling — custom scripts vs Scapy vs existing Mininet attack modules.

## 8. Suggested starting point

Start at **Phase 0 + Phase 1**: stand up the Mininet tree topology, then get exactly one child↔submain PQC handshake working before anything else. Everything downstream (FL, consensus, ML) assumes the secure channel exists — building it first, narrow, and correct avoids rework.
