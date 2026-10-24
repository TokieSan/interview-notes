# Privacy-First Design (Apple-style)

At Apple, privacy is an engineering constraint stated during requirements, not a closing paragraph. Bolted-on privacy is the most-cited Apple design failure mode.

This is the half most candidates under-prepare. **Your opening for any Apple design prompt:**

> "Before I design, let me get the requirements and the data model straight — and I want to include the privacy model in requirements rather than bolt it on. So: what data are we handling, what's the classification, who legitimately needs to read it, and what would we prefer never to be able to read at all?"

That last clause — *what would we prefer never to be able to read* — is the Apple-shaped question. It's the one that leads to end-to-end encryption, on-device processing, and blinded/oblivious server designs.

### 1.3 The privacy toolkit — know these by name and by trade-off

| Technique                                 | What it buys                                          | What it costs                                                                                  |
| ----------------------------------------- | ----------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| **On-device processing**                  | Server never sees raw data at all                     | Device CPU/battery/memory; model size limits; harder to iterate                                |
| **End-to-end encryption**                 | Server stores ciphertext it cannot read               | No server-side search, ranking, or dedup; key escrow/recovery becomes the hard problem         |
| **Differential privacy**                  | Aggregate statistics with a formal per-user guarantee | Noise budget (ε) is finite and spends down; unusable for small cohorts                         |
| **Local DP / randomised response**        | Privacy before data leaves the device                 | Much worse signal-to-noise; needs large populations                                            |
| **Federated learning**                    | Model improves without raw data leaving               | Complex orchestration, straggler handling, and gradients still leak without secure aggregation |
| **Private set intersection / PIR**        | Query without revealing the query                     | Expensive; usually bounded to specific flows                                                   |
| **Data minimisation & retention limits**  | The cheapest privacy technique that exists            | Requires discipline and deletion machinery that actually works                                 |
| **Tokenisation / pseudonymisation**       | Decouples identity from behaviour                     | Re-identification risk via joins; rotate the tokens                                            |
| **Secure Enclave / hardware-backed keys** | Keys never leave the chip                             | Only works on-device; recovery story needed                                                    |

**Also required, and often forgotten:** authentication vs authorisation as separate concerns; per-request authz checks rather than at the gateway only; audit logging that itself doesn't become a privacy hole; data deletion that propagates through caches, replicas, backups, and derived indexes (this last one is the question behind "is deletion a hard requirement?").

### 1.4 Design prompts to run, with the Apple angle

| Prompt | The privacy turn |
|---|---|
| **Settings/preferences sync across a user's devices** | CRDTs merged on-device so the server stores opaque blobs; per-device keys; conflict resolution without server-side reads (C3.8) |
| **Typeahead/search suggestions** | What query data is logged at all? On-device index for personal results, server for global; DP for aggregate popularity |
| **Crash/telemetry reporting pipeline** | Opt-in, DP-noised aggregates, no stable device identifier, retention floor, and stripping payloads on-device before upload |
| **Build artifact cache / CI system for a monorepo** *(closest to Developer Foundations' actual domain)* | Access control per repo and per branch; secrets never in cache keys; reproducibility vs cache-poisoning threat model |
| **Migrate an auth system with zero downtime** | Active sessions during transition, token format change, rollback with tokens already issued, authz policy enforcement mid-migration |
| **Distributed rate limiter for a public API** | Can't log full request content for abuse analysis — so how do you detect abuse? (Hashed features, DP counts, on-device signals) |
| **Feature flag / config distribution to millions of devices** | Flag values themselves can leak roadmap; targeting rules can leak user attributes back to the device |
