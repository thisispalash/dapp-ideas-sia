# dApp Ideas
> Turn your personal curiosity into a measurable experiment

> Sia Small Grant (2026-05)

## Repo Structure

## Architecture
> for detailed version please see [arch.md](proposal/arch.md)

<details>
  <summary>High Level Diagram</summary>

  ```
  ┌─────────────────────────────────────────────────────────────────┐
  │              User's Browser (HTMX + hyperscript + JS)            │
  │                                                                   │
  │   HTMX-driven HTML UI; hyperscript for UI behaviors               │
  │   JS module (crypto island):                                      │
  │     ├─ Argon2id + AES-GCM for envelope operations                 │
  │     ├─ HKDF for derived keys                                      │
  │     ├─ Sia SDK client subset:                                     │
  │     │    – BIP-39 → master_seed                                   │
  │     │    – key derivation (App Key, experiment keys, data keys)   │
  │     │    – object sealing (encrypt + erasure-coding planning)     │
  │     │    – request signing with App Key                           │
  │     └─ IndexedDB cache (decrypted metadata for this session)      │
  └────────────┬─────────────────────────────────┬──────────────────┘
              │                                 │
              │ HTML fragments / auth /         │ Pre-signed +
              │ cached metadata reads           │ pre-sealed payloads
              ▼                                 ▼
  ┌────────────────────────────────────────────────────────────────┐
  │            Our Backend — Go (Chi or Echo) on Railway             │
  │                                                                   │
  │   ├─ Auth + session management                                    │
  │   ├─ HTMX fragment rendering (html/template)                      │
  │   ├─ Sia SDK server subset:                                       │
  │   │    – HTTP client to indexer                                   │
  │   │    – Forwards signed requests, parses responses               │
  │   │    – Does NOT hold App Keys, does NOT sign                    │
  │   ├─ Encrypted-metadata cache (Postgres; still ciphertext)        │
  │   ├─ Policy layer: rate limits, version cop, time/count bounds    │
  │   └─ Per-user telemetry                                           │
  └────────────────────────────────┬─────────────────────────────────┘
                                  │ Signed HTTP requests
                                  ▼
                    ┌─────────────────────────────────┐
                    │  Sia Indexer @ sia.storage      │
                    │  (Foundation-operated)          │
                    │                                 │
                    │  Authenticates via App Key      │
                    │  signatures. Tracks pinned      │
                    │  objects per public key.        │
                    │  Coordinates hosts. Never sees  │
                    │  plaintext.                     │
                    └────────────────┬────────────────┘
                                    │
                                    ▼
                    ┌─────────────────────────────────┐
                    │  Sia Storage Provider Network   │
                    │                                 │
                    │  Encrypted shards across many   │
                    │  hostd nodes. No single host    │
                    │  can reconstruct data.          │
                    └─────────────────────────────────┘
  ```
</details>

### Client

### Backend

### Sia Network

## Milestones

## Progress Reporting

## LICENSE
