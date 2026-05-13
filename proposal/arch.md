# dApp Ideas — Architecture

> Personal experiments tracker. Privacy-first. Encrypted at source. Stored on Sia.

---

## What this is

dApp Ideas is a Progressive Web App for running personal n=1 experiments — peptide protocols, sleep interventions, dietary changes, supplement stacks, anything quantifiable. Users design experiments, log observations and biometric data over time, and review their own results.

The defining property: **the actual user-generated data is encrypted on the user's device under keys derived from their password, and stored on the Sia network**. No centralized database of plaintext user health data exists. Users own their data and can take it with them.

The MVP scope is the personal tracker layer. A researcher query layer, fitness-tracker ingestion, and Story Protocol IP rails are part of the longer-term roadmap but explicitly out of scope for the first build.

---

## High-level architecture

```
┌──────────────────────────────────────────────────────────┐
│                   User's Browser (PWA)                    │
│                                                            │
│   Vite + React                                             │
│   ├─ Password → Argon2id → master_seed                     │
│   ├─ master_seed → HKDF → {encrypt_key, sign_key, eth_key} │
│   ├─ Client-side AES-256-GCM encryption                    │
│   └─ Decrypted manifest held in memory only                │
└─────────────────────────┬────────────────────────────────┘
                          │ HTTPS (encrypted payloads)
                          ▼
┌──────────────────────────────────────────────────────────┐
│              Backend — FastAPI on Railway                 │
│                                                            │
│   ├─ Holds dApp Ideas app credential for indexd            │
│   ├─ Postgres: users, manifests (encrypted), versions      │
│   ├─ Per-user quota tracking                               │
│   ├─ Optimistic concurrency control (version numbers)      │
│   └─ Periodic sync: encrypted manifest → Sia               │
└─────────────────────────┬────────────────────────────────┘
                          │ Sia Storage SDK (Foundation indexd)
                          ▼
┌──────────────────────────────────────────────────────────┐
│           Foundation indexd @ app.sia.storage             │
│                                                            │
│   ├─ Erasure-coding (30 slabs, 10 needed for recovery)     │
│   ├─ Contract management with host network                 │
│   └─ Returns opaque metadata handles                       │
└─────────────────────────┬────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────┐
│              Sia Host Network (Layer 1)                   │
│                                                            │
│   Encrypted slabs distributed across many hostd nodes.     │
│   No single host can reconstruct or decrypt data.          │
└──────────────────────────────────────────────────────────┘
```

---

## Components

### Frontend

- **Stack:** Vite + React, deployed as a Progressive Web App.
- **Responsibilities:** All cryptographic operations (key derivation, encryption, decryption). Manifest decryption and editing. UI for experiment creation, data entry, dashboards, conflict resolution.
- **Crypto libraries:** `@noble/hashes` (Argon2id, HKDF), `@noble/curves` (secp256k1 for future EOA work), WebCrypto API (AES-256-GCM).
- **Offline support:** Local IndexedDB caches the in-memory manifest and unsent writes; sync on reconnect.

### Backend

- **Stack:** Python + FastAPI, SQLAlchemy + Alembic, Postgres. Deployed on Railway.
- **Responsibilities:** App-credential custody for `app.sia.storage`. Proxying encrypted blobs to indexd. Per-user quota and usage tracking. Optimistic concurrency control on manifest versions. Async with Sia.
- **Hard rule:** The backend never sees plaintext user data. All payloads are encrypted before they leave the browser.

### Storage (Sia)

- **Indexer:** Foundation-hosted indexd at `app.sia.storage`, accessed with our app credential.
- **Why not self-hosted indexd:** Operational simplicity, eliminates wallet/contract management, aligned with Foundation guidance.
- **Why not s3d:** S3 gateway is for migrating existing S3-native apps. We're greenfield; direct SDK is cleaner and lower-latency.

### Identity & keys

- **Login:** Username + password. Usernames generated server-side from a curated wordlist (`curious-axolotl-2847` style) for friction-free signup.
- **Key derivation:** Password runs through Argon2id once per session to produce a 256-bit master seed. HKDF derives purpose-specific keys from the seed.
- **Recovery:** A 6-word Diceware-style passphrase shown at signup. Wraps a second copy of the master key. Lost password + lost passphrase = lost data, and we say so plainly in onboarding.

>> What if we add a mention of passkeys? Could we not envelope the keys with different login methods? Maybe then track each action with something like `<envelope_id, action>` \
>> This would provide a better user experience, but bring many attack surfaces

---

## Storage architecture

### Manifest model

>> With the proposed structure, what if there are dozens of experiments? Why not just track `experiment_metadata_id` or something?
>> If we are effectively replacing a psql or db layer with sia, and this is a data heavy app, why not mimic a graph of sorts?

Each user has one **encrypted manifest blob**. The plaintext structure is roughly:

```json
{
  "version": 47,
  "experiments": [
    {
      "id": "exp_abc123",
      "title": "Methylene Blue + Sunlight",
      "description": "Testing morning MB protocol with UV exposure",
      "tags": ["nootropic", "sleep", "energy"],
      "schema": {
        "fields": [
          {"name": "dose_mg", "type": "number"},
          {"name": "energy_1_10", "type": "number"},
          {"name": "notes", "type": "text"}
        ]
      },
      "started_at": "2026-06-01",
      "data_point_handles": ["sia_handle_1", "sia_handle_2", ...]
    }
  ]
}
```

- The full manifest, including titles and tags, is **encrypted** before it ever leaves the browser.
- Individual data points are **separately encrypted blobs** on Sia. The manifest holds references (Sia handles) to them, not the data itself.
- This means: a session can load the manifest fast (one fetch, small blob), then lazy-load individual data points as the user drills in.

### Why per-data-point blobs?

- Cheap, granular writes — logging an entry doesn't rewrite the whole manifest's worth of data points.
- Forward-compatible with per-field encryption keys (future researcher layer can re-encrypt just the fields a user opts to share).
- Naturally append-only, which simplifies the merge story.

### Where the manifest physically lives

- **Postgres (primary):** Latest encrypted manifest blob, keyed by user, with a monotonic version number. This is the live operational copy.
- **Sia (durable):** Periodic snapshots of the encrypted manifest pushed via indexd. This is the durability + portability guarantee.
- Data-point blobs live exclusively on Sia.

>> Also the user's logged-in clients


The user can always export their full encrypted dataset from Sia and decrypt it independently of our infrastructure. That's the user-owned guarantee.

>> This could be an attack surface tbh since all data is encrypted using the same key, derived from one password

---

## Authentication & key management

### Envelope encryption

```
On signup:
  1. Generate a random 256-bit master_key K   (this is the long-lived secret)
  2. salt ← random 16 bytes
  3. wrappingKey W = Argon2id(password, salt, m=64MB, t=3, p=1)
  4. wrappedK = AES-256-GCM-Encrypt(K, W)
  5. Server stores: {username, salt, wrappedK, version=0}

  6. recoveryPassphrase ← 6 random words from curated list
  7. recoveryKey = Argon2id(recoveryPassphrase, salt)
  8. wrappedK_recovery = AES-256-GCM-Encrypt(K, recoveryKey)
  9. Server stores wrappedK_recovery alongside wrappedK
 10. Show recoveryPassphrase to user once; user saves it offline.

On login:
  1. Client sends username; server returns {salt, wrappedK}
  2. Client derives W = Argon2id(password, salt)
  3. Client decrypts wrappedK → K
  4. K is now resident in browser memory for the session

On password change:
  1. Unwrap K with old password (as in login)
  2. Generate new salt, derive new W from new password
  3. Re-wrap K with new W → new wrappedK
  4. Send to server, replace stored copy
  5. K never changed; all existing data still decrypts

On recovery:
  1. User has lost password, enters recoveryPassphrase
  2. Client derives recoveryKey, unwraps wrappedK_recovery → K
  3. User sets new password; re-wrap K with new password's W
```

### Key derivation hierarchy

From the master key K (or from master_seed, depending on chosen design — we'll likely derive K from a master_seed via HKDF so we can derive sibling keys too):

```
master_seed (256 bits, never leaves browser)
  │
  ├─ HKDF(info="dapp-ideas:encrypt:v1") → AES-256 key for data encryption
  ├─ HKDF(info="dapp-ideas:sign:v1")    → ed25519 key for auth challenges
  └─ HKDF(info="dapp-ideas:eth:v1")     → secp256k1 key → Eth address (EOA)
```

The EOA is derived but **not used** in MVP. The address is computed client-side and stored in our Postgres as part of the user record. When v2 ships Story Protocol integration, the user already has a deterministic Eth address that's been theirs since signup — no separate wallet onboarding.

Versioning the `info` strings means any individual key can be rotated in the future without touching the master seed.

>> What does that last line mean? If a key is rotated, so will the eth eoa, so until theres a smart contract account, that would mean transferring all assets to new eoa \

---

## Privacy & threat model

### What we protect

- **User-generated experiment data** (observations, biometrics, dosages, notes): encrypted client-side under user-derived keys. Server, indexer, and host network all see only ciphertext.
- **Experiment metadata** (titles, tags, schemas, dates): encrypted as part of the manifest. Server stores the encrypted blob; cannot read titles or categorization.

### What we don't claim to protect against

- **A compromised user device.** Anyone with the user's password (or their unlocked browser) sees everything. Same as every other E2E system.
- **Backend memory dump during a non-active session.** We don't see plaintext at any point, so there's nothing to dump on the server. We do see ciphertext flowing through; an attacker with full backend access could log ciphertext for offline attack, but they cannot break the encryption.
- **Sophisticated traffic analysis.** Sizes and timing of uploads are visible to anyone with network access at our backend. Acceptable for the MVP.

### What our backend knows

- Username (a generated wordlist string)
- Encrypted manifest blob, keyed by username, versioned
- Bytes uploaded/downloaded per user (for quota)
- Eth address (derived, not used in MVP)
- Timestamps of operations

### What our backend does not know

- The user's password
- The user's master seed or any derived key
- Any plaintext content of any experiment or data point
- Experiment titles, tags, descriptions, or schemas
- Any biometric or observation value

---

## Concurrency model

### Versioning

Every manifest write includes the version number the client started from. Server uses optimistic concurrency control:

```
client.put_manifest(encrypted_blob, base_version=47)

server:
  if current_version == 47:
    accept; current_version = 48
  else:
    reject with current_version
```

### Auto-merge for additive changes

Most edits in this app are **append a new data point** to an existing experiment. These are trivially mergeable:

```
on reject:
  1. Client downloads current encrypted manifest from server
  2. Client decrypts both: its local version and the new server version
  3. Client diffs at the JSON-structural level:
     - new experiments on either side → merge (union)
     - new data points appended to existing experiments → merge (union)
     - same field of same experiment edited on both sides → CONFLICT
  4. If no conflicts: re-encrypt merged, retry write
  5. If conflicts: invoke conflict UI
```

### Conflict UI

For true conflicts (same field edited on both sides — rare for a solo-user n=1 tracker but possible across phone + desktop):

- Side-by-side diff at the field level
- Per-conflict options: "Keep mine," "Keep theirs," "Keep both as separate entries"
- Resolution commits a new merged manifest version

The conflict UI is included in MVP scope because it's a small protection that prevents silent data loss and signals quality.

---

## Out of scope for MVP

>> One thing for a future variation would be to implement nostr like behaviour using sia; \
>> ie, users can subscribe to each other's data streams, and sia becomes a decentralized relayer
>> side note, the 30/10 erasure coding also kinda makes this at least like 3 relayers (maybe more?)

These are deliberately deferred to keep the 8-week budget honest. Each is a candidate for a future grant or post-grant work.

- **Researcher query and discovery layer.** Privacy-preserving access for scientists to run aggregate studies on consenting pools of users' data. Requires selective-disclosure encryption schemes; significant scope.
- **Story Protocol integration.** User data points as licensable IP assets. The EOA derivation in MVP lays the groundwork; actual integration is v2+.
- **7579 modular smart contract accounts.** Future identity upgrade path. EOA signs SCA deployment when ready. khaaliNames module and WebAuthn module both candidates for inclusion.
- **Real-time fitness ingestion** (Garmin, Whoop, Oura, CGM). MVP stretch goal is *one* one-shot import path (Garmin Connect export or Apple Health export). Continuous sync is v2.
- **CRDT-based offline merging.** Current concurrency model is optimistic locking with auto-merge for adds. True offline-first multi-device editing requires CRDTs and is future work.
- **Mobile-native apps.** PWA covers iOS and Android. Native apps (Swift/Kotlin SDKs exist for Sia) come later if the product justifies it.

---

## Open questions for the Foundation

These need resolution from the Sia Foundation (likely via Matt) before locking the proposal:

1. **Cost model for `app.sia.storage`.** Is there a free quota for grant projects, pass-through billing, or pre-funded account model? This determines how the $2K user-storage subsidy line item flows mechanically.
2. **Per-app usage telemetry.** Does the indexer publish per-app usage stats via API, or only via the indexer's own dashboard? Affects backend quota implementation.
3. **App credential model.** One credential per application (as expected) or are per-user sub-credentials possible? Affects whether Pattern A (full proxy) or Pattern B (signed URLs) is even feasible in future.
4. **Rate limits and quotas.** Per-second throughput limits, max-file-size limits, or per-app monthly caps on the Foundation's hosted instance.

---

## Decisions log

>> Okay, i like the idea of a decision log, but not every date can be the same.. either group under date, or edit table

| Date | Decision | Rationale |
|---|---|---|
| 2026-05-12 | Use Foundation-hosted indexd at `app.sia.storage` | Matt's suggestion; reduces operational scope |
| 2026-05-12 | Skip s3d | We're greenfield; direct SDK is cleaner |
| 2026-05-12 | Pattern A (full proxy) for app credential | Simpler for MVP; signed URLs are v2 |
| 2026-05-12 | Encrypted manifest + encrypted data points on Sia | Strong privacy claim; needed for "user-owned data" mission alignment |
| 2026-05-12 | Postgres holds latest encrypted manifest as live cache | Latency; Sia is the durable backup |
| 2026-05-12 | Username + password, envelope encryption, no embedded wallet | No third-party dependency; standard pattern |
| 2026-05-12 | Diceware passphrase for recovery | Memorable, not seed-phrase-coded |
| 2026-05-12 | EOA derived via HKDF, not used in MVP | Lays groundwork for Story Protocol future without adding scope |
| 2026-05-12 | Conflict UI in MVP scope | Quality signal; small additional work over reject-with-refresh |
| 2026-05-12 | FastAPI on Railway | Lightweight, fits team skills, eliminates ops complexity |