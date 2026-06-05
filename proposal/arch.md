# dApp Ideas — Architecture
> Personal experiments tracker. End-to-end encrypted, with uncompromised UX.

> [!TIP]
> APP_ID = `keccak256("com.dapp.ideas/alpha")` = \
> `0x8a8fbc3f4562a759a7ff2a7bbe961df6a719fd660c4c5dd6e691a4a795b73382`

<details>
  <summary>
    dApp Ideas helps anyone turn personal curiosity into a measurable experiment
  </summary>

  Switched milk types and want to know if your stomach feels better? Saw a new exercise on TikTok claiming to target your rear delts — does it actually work for you? Started a peptide protocol or trying a sleep intervention and want to track what changes? dApp Ideas lets you design the experiment, log observations and data over time, and review your own results.
</details>

We want this to feel like a familiar tracking app — username and password, no seed phrases to memorize, no wallets to set up — while quietly providing the strongest possible guarantee on the data underneath: **it's end-to-end encrypted, only the user can read it, and the user can take it with them at any time.**

## Vision and MVP scope

The long-term picture a full end-to-end pipeline...

1. A personal experiments tracker
2. A licensing layer
3. A marketplace layer
4. A collaboration module
5. An inference layer

> [!NOTE]
> The MVP scope is limited to the first module only (ie, the tracker), which is the important foundation for subsequent modules and layers.

## Architectural requirements

Core requirements from a storage and identity stack: E2EE, data mutability and archival flexibility, ~permissionless access. More specifically:

| Constraint | Why? | IPFS | Arweave | Sia |
| --- | --- | --- | --- | --- |
| E2EE, user-held keys | sensitive health data | external | external | AppKey |
| E2EE, shareable ephemeral keys | domain-restricted layer-2 access | external | external | derived via HKDF |
| Mutability | growing datasets | new CID | immutable\* | `indexd` [pinning](https://devs.sia.storage/docs/core-concepts/pinning) |
| Persistence | archived datasets | external | native | `indexd` [unpin](https://devs.sia.storage/docs/core-concepts/pinning#unpinning-and-deleting) / `hostd` |
| Cost abstraction | no crypto UX | Filecoin | pay once | Siacoin / Siachain |
| Diverse clients | wide support | abstractable | abstractable | [official SDKs](https://devs.sia.storage/docs/quickstart/index#install-the-sdk) |
| Composability | easy integration, hacker-friendly | | | [app-defined metadata](https://devs.sia.storage/docs/recipes/object-metadata) |
| Open gardens | compete on service | | | |

> \* external implies third-party or self-developed / hosted

Overall, IPFS and Arweave can be utilized with substantial tooling on top, but Sia reduces our work considerably. Specifically, Sia offers a native encryption mechanism, near-comprehensive data flexibility, and ready support across many SDKs.

## High-level architecture

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

Our backend primarily exists as a proxy for the network and a cache layer, to offer ux affordances

## Components

### Client — HTMX driven
> A coping mechanism for React fatigue; own end-to-end experience

Server rendered HTMX templates, along with support for libraries (crypto, sia, etc.) and _user programming_ (via [hyperscript](https://hyperscript.org/)).

**JS libraries**

- `@noble/hashes` — audited, dependency-free. Provides Argon2id and HKDF.
- WebCrypto API — AES-256-GCM via the browser-native crypto.
- `@siafoundation/sia-storage` (client subset) — key derivation, object sealing, request signing.

### Backend — Go with [Echo](https://echo.labstack.com/) / [Chi](https://github.com/go-chi/chi) + `pgx`

Sia is natively written in Go; the Go SDK is the most mature; the ecosystem fit is direct. Concurrency primitives and HTTP performance are strong out of the box, which suits a forwarding proxy with policy enforcement.

> [!IMPORTANT]
> backend never gets to see any plaintext; only holds encrypted keys and signed metadata \
> all crypto operations happen client side and no secrets are shared over the network

| Role | | sia_sdk? |
| --- | --- | --- |
| session mgmt | | |
| key escrow | | |
| caching | | |
| policy enforcement | | |
| indexer forwarding | | |
| usage telemetry | | |

- **Auth + sessions.** Username/password login. Session tokens. Multi-device session management including revocation. Usernames are server-generated, not user-selected.
- **Key escrow.** Stores `wrappedAppKey` (under password-derived key) and `wrappedAppKey_recovery` (under Diceware passphrase-derived key) per user. Stores hash of the user's BIP-39 phrase for the third recovery path. Never sees plaintext keys.
- **Encrypted-metadata cache.** Mirrors the user's pinned-object list and their encrypted metadata blobs for faster startups. Still ciphertext from our side — we cannot decrypt.
- **Policy layer / version cop.** Validates signed payloads, enforces rate limits, performs version checks on supersedes chains before forwarding to the indexer, supports future time/count-bounded write windows.
- **Indexer forwarding.** Uses the Go SDK's HTTP client to forward client-signed requests to the indexer and stream responses back.
- **Per-user telemetry.** Storage counters and consent-based app performance metrics.

### SDK split — what runs where

| Concern | Client (`@siafoundation/sia-storage` JS) | Server (Sia Go SDK) |
| --- | --- | --- |

| BIP-39 → master_seed | 1 | 0 |
| Key derivation (App Key, experiment, data) | 1 | 0 |
| Object sealing (encryption) | 1 | 0 |
| Request signing with App Key | 1 | 0 |

| HTTP client to indexer | 0 | 1 |
| Response parsing | partial (for decryption) | 1 |
| Pin / unpin operations | signed by client | forwarded by server |
| Listing pinned objects | signed by client | forwarded by server |

The App Key never leaves the browser. The server's role is purely transport and policy. SDK versions to be pinned by final MVP ship.

### Storage layer

Primarily managed by the official SDKs that interact with indexers and hosts.

Connected indexer: `https://sia.storage`

- **Indexer:** `https://sia.storage` (Foundation-operated). Recommended in the official Sia developer docs as the default for app developers.
- **Storage providers:** the open Sia host network. The indexer selects and manages contracts based on health and price; we don't pick hosts directly.
- **Erasure coding:** Sia supports tunable durability per object. We use the default redundancy setting unless testing reveals reason to tighten it for specific object types.

Pinning: we pin all objects since the SDK [currently](https://devs.sia.storage/docs/core-concepts/pinning#unpinning-and-deleting) treats unpin and delete as the same operation.

## Identity, keys, and login UX

We follow web2 best practices: generate the `AppKey` on first-time password setup, store encrypted (wrapped) copies for different recovery methods. The base password also seeds HKDF for derived keys.

### Cryptographic primitives

- **BIP-39** — 12-word recovery phrase. The user's master secret per Sia's auth model. We never store the phrase itself; we store only its hash for the third recovery path.
- **Argon2id** — memory-hard password-to-key derivation. Used for password and recovery-passphrase wrapping.
- **HKDF** — derives multiple purpose-specific keys from a seed via domain-separated `info` strings.
- **AES-256-GCM** — symmetric encryption for envelope-wrapping keys.

### Sign-up flow

```
On signup:
  1.  User provides password
  2.  Client generates: salt (16 bytes), recoveryPassphrase (6-word Diceware)
  3.  Client generates: BIP-39 recovery phrase (random entropy, 12 words)
  4.  Client derives App Key from (BIP-39 phrase + our App ID)
        using the SDK's deterministic derivation
  5.  Client connects to indexer (via backend proxy) with the App Key
        — indexer registers the App Key's public key
  6.  Client derives: W = Argon2id(password, salt)
                      R = Argon2id(recoveryPassphrase, salt)
  7.  Client computes:
        wrappedAppKey          = AES-256-GCM(AppKey, W)
        wrappedAppKey_recovery = AES-256-GCM(AppKey, R)
        bip39_hash             = SHA-256(BIP-39 phrase)
  8.  Client sends to backend: username, salt, wrappedAppKey,
                               wrappedAppKey_recovery, bip39_hash
  9.  Backend persists these. Backend never sees the BIP-39 phrase,
        the App Key, or the password.
  10. Client displays to user:
        - The BIP-39 phrase ("this is your master recovery — save it")
        - The recoveryPassphrase ("this is your convenience recovery")
  11. After user confirms they've saved both, the BIP-39 phrase is
        cleared from memory. The App Key remains in session memory.
```

### Login flow

```
On login:
  1. User enters username + password
  2. Backend returns: salt, wrappedAppKey
  3. Client derives W = Argon2id(password, salt)
  4. Client decrypts wrappedAppKey → AppKey
  5. AppKey held in browser memory for the session
  6. Client signs all subsequent requests with AppKey; backend forwards
```

### Password change

```
On password change:
  1. Unwrap AppKey with old password
  2. Generate new salt, derive new W
  3. Re-wrap AppKey → new wrappedAppKey
  4. Send to backend; replace stored copy
  AppKey itself never changes; all data remains accessible.
```

### Recovery paths

| Path | Use case | UX |
| --- | --- | --- |
| **Password / auth methods** | Daily use; future passkey/OAuth support | Familiar; enter credential to log in |
| **Recovery passphrase** (Diceware) | Forgot password, have passphrase | Type 6 words to reset password |
| **BIP-39 phrase** | Forgot password and passphrase, OR want to leave dApp Ideas with full data control | Type 12-word phrase; backend verifies via stored hash; derives App Key fresh; user sets new password |

> [!NOTE]
> If all paths lead to lost secrets, the data is unrecoverable. This is the cost of true user-owned data — responsibility for access lives with the user, not with us. \
> We will support multiple passkeys and additional auth methods over time to reduce the practical likelihood of losing all paths, but the fundamental property does not change.

### Username generation

Usernames are generated server-side from a curated wordlist: `curious-axolotl-2847` style. Eliminates uniqueness collisions, removes a friction point in onboarding, and avoids accidental PII in identifiers. A separate display-name field can be added later for personalization. This pattern is compatible with name service providers on other chains that abstract crypto UX while also enabling easier departures.

### Multi-chain key derivation

### Multikey setup via HKDF's `info`

The BIP-39 phrase is the natural root for HD-wallet-style multi-key identity and for per-resource encryption keys. Via domain-separated HKDF derivations:

```
BIP-39 phrase → master_seed (via BIP-39 / BIP-32)
              → HKDF(info="dapp-ideas:sia:alpha")              → Sia App Key
              → HKDF(info=f"dapp-ideas:experiment:{id}:alpha") → experiment definition key
              → HKDF(info=f"dapp-ideas:manifest:{id}:alpha")   → manifest key
              → HKDF(info=f"dapp-ideas:data:{id}:alpha")       → per-data-point key
              → HKDF(info="dapp-ideas:eth:alpha")              → Eth address (future)
              → HKDF(info="dapp-ideas:auth:alpha")             → app-level signing key (future)
              → HKDF(info="dapp-ideas:[chain]:alpha")          → other chain identities (future)
```

>> note, while maintaining app version in info, we provide some level of redundancy for earlier users.. maybe.. but, earlier users will have more keys

The user has one phrase, but deterministic identities on every chain we support and deterministic keys for every resource they create. Versioned `info` suffixes (currently `alpha`) allow per-purpose key rotation without touching the master.

We persist the `{id}` for each derived resource alongside its storage cost and other metadata, so that on demand we can regenerate the key and share it without holding the key material itself.

## Data model

Sia's object model does the work that a per-user manifest blob would otherwise have to do. Three object types, each with a clear role.

> [!IMPORTANT]
> **Objects are immutable.** Each object's ID is a content-derived hash of its slab layout, so changing the data produces a new object with a new ID. The indexer never updates objects in place. This property shapes every pattern below.

### Object types

**Experiment definition.** One per experiment. Rarely changes. Edits use the supersedes pattern.

```json
{
  "type": "experiment",
  "title": "Methylene Blue + Morning Sunlight",
  "description": "Testing morning MB with UV exposure",
  "tags": ["nootropic", "energy", "circadian"],
  "schema": {
    "fields": [
      {"name": "dose_mg", "type": "number"},
      {"name": "energy_1_10", "type": "number"},
      {"name": "notes", "type": "text"}
    ]
  },
  "started_at": "2026-06-01",
  "ended_at": null
}
```

Encrypted with `experiment_def_key`.

**Data point.** One per logged observation. Immutable, append-only.

```json
{
  "type": "data_point",
  "logged_at": "2026-06-15T08:14:00Z",
  "fields": {
    "dose_mg": 5,
    "energy_1_10": 8,
    "notes": "noticeable bump after 20 min"
  },
  "tags": ["morning", "fasted"]
}
```

Encrypted with its own `data_point_key`. Notice that a data point does *not* hard-bind to a single experiment — the experiment-to-data linkage lives in the manifest. This is the key flexibility that lets the same data point participate in multiple manifests (e.g., a blood panel where one measurement is part of a vitamin D experiment, and the others are independent observations a researcher might license separately).

**Dataset manifest.** A typed listing that aggregates data points and (optionally) an experiment definition. Generated by the user (or by consent from a request) and updated via supersedes as data evolves.

```json
{
  "type": "manifest",
  "manifest_type": "experiment",
  "label": "MB + Sunlight, Spring 2026",
  "experiment_def_id": "<id>",
  "created_at": "<timestamp>",
  "updated_at": "<timestamp>",
  "entries": [
    {"data_point_id": "<id>", "data_point_key": "<base64>"}
  ],
  "policy": null
}
```

Encrypted with `manifest_key`. The `manifest_type` field is the extensibility hook — `"experiment"` for experiment-bound datasets, `"custom"` for ad-hoc user-curated sets, `"lab_panel"` for grouped clinical results, `"request_response"` for researcher-requested datasets, etc. `policy` is null in v1 and becomes a reference to an external policy object in v2.

### Manifest lifecycle

| Manifest type | Created by | Updated when |
| --- | --- | --- |
| `experiment` | Auto-published on experiment creation | New data point added to the experiment |
| `custom` | User-initiated ("here's a set of measurements") | User explicitly edits the set |
| `request_response` | Researcher requests a selection; user consents | User adds matching new data; can also be one-shot |

Experiment manifests are auto-maintained: when the user logs a new data point against an experiment, the client supersedes the existing manifest with a new version that includes the new entry. From the user's perspective, this is invisible — they just log data.

### Edits use a supersedes pattern

Because objects are immutable, an edit isn't an in-place update — it's a new object that points back at its predecessor via a `supersedes` field. The client uploads the new object, pins it, and unpins the predecessor. Following the supersedes chain reconstructs full history. Users can keep prior versions pinned for explicit audit retention if they want.

Manifests, experiment definitions, and edited data points all use the same pattern. The chain is a linked list, not a tree — keeping it flat keeps reads simple.

### Data access flow

1. Client first checks IndexedDB for decrypted metadata and stored config.
2. In parallel, requests backend for encrypted-metadata cache and user preferences.
3. Client decrypts metadata client-side using the appropriate derived keys.
4. Data point bodies are lazy-loaded when the user drills into a specific entry.

## Caching strategy

Two layers of cache, plus the durable source on Sia:

| Layer | What | Latency | Mutability |
| --- | --- | --- | --- |
| **Client IndexedDB** | Decrypted metadata + recently-viewed data points | Instant | Cleared on logout / cache invalidation |
| **Server Postgres** | Encrypted metadata blobs + object listings + per-user telemetry | <50ms typical | Updated on every write |
| **Sia indexer + hosts** | Canonical, durable storage | Higher (first-hit seconds) | Source of truth |

Writes invalidate the relevant entries in both caches. Background sync ensures cross-device updates propagate within a short window. When server-side caching becomes a hot spot (well past MVP scale), the cache can move to an edge layer (e.g., Cloudflare KV); migration is straightforward because the cache is `key → ciphertext`.

## Concurrency

Most writes are appends — logging a new data point creates a new immutable object that can't conflict with anything. The manifest for the affected experiment supersedes its predecessor with the new entry appended; if two devices append concurrently, the version cop catches the second one, the client re-fetches the head, applies its append on top, and tries again. Appends commute, so this is invisible to the user.

For edits to experiment definitions or to existing data points (rare):

1. Every edit references a `supersedes` target — the object it intends to replace.
2. The backend (version cop) checks whether the targeted object is still the current head of its supersedes chain.
3. If yes: forward to the indexer, pin the new object, unpin the predecessor.
4. If no: reject early; return the current head and the conflicting version to the client.
5. On rejection, the client presents a **field-level conflict UI** with "keep mine / keep theirs / keep both as separate entries" options. The user's resolution generates a new edit that supersedes both branches.

The version-cop check in our backend (rather than relying on the indexer to reject) is a UX affordance — failures come back faster and with more context.

## Sharing and key rotation

> [!NOTE]
> The v1 sharing model is deliberately simple: out-of-band, consent-based, no on-chain access control. Programmable policies arrive in v2 with the external key-management layer.

### Sharing a manifest

The unit of sharing is a manifest. Whether it's an experiment, a custom selection, or a researcher request response, sharing means handing over the manifest's ID and its key.

```
On share:
  1. User selects what to share via the UI
       (whole experiment, custom selection, etc.)
  2. App computes a share token:
       {
         manifest_id: "<id>",
         manifest_key: "<base64>",
         indexer: "https://sia.storage",
         shared_by: "<sender's public username>"
       }
  3. App offers delivery options:
       - Copy as string
       - One-time URL to a "view this dataset" landing page
       - (Future) Send via decentralized relayer
  4. Recipient receives the token, opens dApp Ideas in import-only mode,
     pastes the token. App fetches the manifest, follows the supersedes
     chain to its head (for continued access to new data), decrypts the
     manifest with the shared key, then fetches and decrypts each data
     point using the keys listed in the manifest entries.
```

A recipient with a share token can read everything the manifest references, including future data points the original user adds to that manifest (since the manifest supersedes forward). They cannot write, edit, or revoke onward. Nothing technically prevents them from forwarding the token they received; that's the inherent limitation of out-of-band sharing.

### Continued access

Because the manifest supersedes forward as new data points are added, a recipient who walks the supersedes chain to the current head always sees the latest. The same token works indefinitely — no re-sharing needed unless the user rotates keys.

### Key rotation (revoking shared access)

```
On rotation:
  1. App derives a new key with a bumped version suffix:
       new_manifest_key = HKDF(info=f"dapp-ideas:manifest:{id}:beta")
  2. App reads all manifest entries with the old key.
  3. App writes a fresh manifest object encrypted with the new key,
     containing the same entries (data points themselves keep their
     own keys; only the manifest's wrapping changes).
  4. Old manifest is unpinned. Past share tokens (bound to the old key)
     can read old manifest snapshots until host contracts naturally
     expire, but cannot read the new head or anything added after.
```

> [!WARNING]
> **True immediate cryptographic deletion is not a Sia primitive.** Unpinning stops the indexer from maintaining/repairing an object, but its shards remain on hosts until their storage contracts naturally expire (typically months). Sia's storage contracts are protocol-level renter/host agreements; they aren't user-programmable like EVM contracts, so there is no "expire now" mechanism. The clean path to immediate cryptographic invalidation is exactly what an external key-management layer provides: revoke the key, the ciphertext becomes inaccessible regardless of where it physically sits. This is a v2 concern.

## Privacy & threat model

### What's encrypted, where

| Data | Plaintext visible to | Cost of this choice |
| --- | --- | --- |
| Experiment data values | User's browser only | No server-side collab or sync features |
| Experiment metadata (titles, tags, schema) | User's browser only | IndexedDB / server cache stores ciphertext |
| Data point timestamps | Indexer (operational metadata) | Some traffic-analysis risk |
| Object sizes | Indexer (slab layout) | Some traffic-analysis risk |
| Which App Key owns which objects | Indexer (auth model) | Indexer knows our users' public keys |
| Username | Our backend, indexer | Just a generated wordlist string |
| Storage usage counters | Our backend | Operational telemetry only |
| Hash of user's BIP-39 phrase | Our backend | Used only for recovery validation |

The SDK encrypts both the object body **and** the application-defined metadata blob before transmission. Storage providers see only encrypted shards; the indexer sees encrypted blobs, slab layouts, and ownership — but cannot read content.

### Attack surfaces and mitigations

| Attack | Risk | Mitigation |
| --- | --- | --- |
| Compromised user device | Attacker reads everything | Standard E2E limitation. Session revocation lets the user invalidate other sessions remotely. |
| Stolen laptop with active session | Attacker continues as user | Revoke all sessions + force re-auth + password rotation. App Key and data unchanged. |
| Database breach at our backend | Wrapped keys + ciphertext leaked | Attacker still needs password, recovery passphrase, or BIP-39 to decrypt. Argon2id makes brute-forcing expensive. |
| Brute-force on Argon2id | Weak passwords cracked offline | Memory-hard parameters; minimum password strength enforced; auth endpoints rate-limited. |
| Traffic analysis at indexer | Sizes/timing/ownership visible | Accepted limitation. Indexer is trust-but-verify. |
| Lost both password and Diceware | User locked out unless BIP-39 saved | BIP-39 is the canonical recovery; clearly framed at signup. |
| Lost all recovery paths | Data is gone | The cost of true user-owned data. |
| Phishing site mimicking dApp Ideas | User enters credentials, attacker derives keys | Standard web app limitation. Future: domain attestation, passkey support. |
| Stolen share token forwarded onward | Unauthorized recipients gain access | Rotate manifest key to invalidate forward. v2 policy layer makes revocation immediate. |

## Cost and capacity estimation

**Stated approach:** dApp Ideas pays the operational costs of running the service. Users never acquire Siacoin, fund accounts, or interact with crypto economics directly. Future tiers may add subscription-based services for power users; the baseline remains free.

**Where the $2,000 line item goes:** infrastructure costs over the grant period and runway beyond.

| Item | Estimate (monthly) |
| --- | --- |
| Railway hosting (Go service + Postgres) | $20–50 |
| Domain, SSL, basic monitoring | $5–15 |
| Sia storage costs at our data scale | Trivially small — see below |
| Email or other comms (if needed) | $10–20 |
| Buffer for traffic spikes / overages | $20–50 |
| **Estimated total** | **$55–135 / month** |

**Storage costs are negligible at our data scale.** Experiment metadata and data points are mostly text and numbers. A heavy user generating one data point per day with 200 bytes of payload accumulates roughly 73 KB of original data per year — about 220 KB at 3x redundancy. At ~$3/TB/month, a thousand such users for a year would cost a few cents.

**Runway implication.** $2,000 covers 15–36 months of infrastructure at MVP scale; storage cost remains noise relative to compute and bandwidth even at 10,000+ active users.

## Future plans — Departure, layers, and ecosystem

These are deliberately deferred to keep MVP scope honest. Each is a candidate for a follow-on grant or post-grant work.

### IP, licensing, and access layers

- **External key-management layer for programmable access control.** v2's sharing substrate. Per-manifest keys are uploaded to an external policy layer; access conditions are expressed as programmable smart contracts ("anyone holding this license can read," "decryption allowed only inside this attested compute environment," etc.). A decentralized network performs threshold decryption when conditions are met. The Sia storage layer doesn't change — only key management and access control evolve. This is the clean substrate for the researcher layer, brand-claim flows, and any automated data-deal patterns.
- **Researcher discovery and selective sharing.** Built on the key-management layer. Researchers discover datasets matching study criteria; users opt in per-manifest; access is granted programmatically.
- **Consumer brand verifiable claims.** Brands query opt-in pools of user data to substantiate marketing claims with verifiable cryptographic provenance.
- **IP and royalty layer.** Datasets registered as on-chain IP assets; royalties flow to contributors when their data is used in published research or brand claims.

### Departure — user-owned exits

The corollary of user-owned data is user-owned exits. The user can always leave with their data intact. We invest in this explicitly because it's what makes the "user-owned" claim real, not just marketing.

- **Decentralized identity primitives.** Smart-contract-account-style identity built on the multi-chain keys we already derive. Modular validators (passkey-based authentication, name resolution from external naming services, social recovery).
- **Well-formatted exports.** Open, documented export formats (JSON-LD, CSV, signed bundles) that other tools can import. Bring-your-own-archive workflows.
- **Decentralized relayer infrastructure.** Nostr-like relay layer for discoverable experiments and analyses. Lets users publish and discover without relying on our app to be alive.

### Platform and reach

- **Native applications.** PWA is the MVP target. Native iOS/Android via Swift/Kotlin SDKs follows if the product warrants it.
- **Fitness tech integration.** Continuous sync from health platforms (Garmin, Whoop, Oura, CGMs, body composition scales). MVP stretch goal is one one-shot import path; full continuous sync is post-MVP.
- **Offline-first multi-device with CRDTs.** Worth revisiting after the version-cop pattern has been load-tested in practice. CRDTs earn their complexity if multi-device offline editing becomes a frequent pattern.

## Open questions

1. **Storage payment mechanism with the Foundation.** We pay for all storage; the exact billing relationship (pre-funded balance, pass-through billing, sponsored-gas-style pool) needs confirmation from the Foundation. Doesn't change the architecture; affects implementation of the payment flow only.

## Decisions log

>> lets also add numbers and "superceded by ..." \
>> also no "active" marker, easier to spot anomalies

| Date | Decision | Rationale | Status |
| --- | --- | --- | --- |
| 2026-05-12 | Use `https://sia.storage` as indexer | Foundation-recommended public indexer | |
| 2026-05-12 | Skip s3d gateway | Greenfield app; direct SDK is cleaner | |
| 2026-05-15 | Per-user App Key model (Sia-native) | Correcting v1's mistaken multi-tenant assumption | |
| 2026-05-15 | Pin all of a user's data while | Simple MVP; storage cost is negligible anyway | |
| 2026-05-15 | Three recovery paths: password, Diceware, BIP-39 | Convenience for casual users, durability for power users | |
| 2026-05-15 | Client generates BIP-39 phrase; user holds it; backend stores only hash | Sia-spirit-respecting; user remains in full control of canonical recovery | |
| 2026-05-15 | Multi-chain key derivation from BIP-39 (HD-wallet pattern) | Forward path to other chains without separate wallets | |
| 2026-05-15 | HTMX + JS islands for crypto operations | Lighter than React; honest about needing JS for SDK calls | |
| 2026-05-15 | FastAPI on Railway (Python) | Initial choice for dev velocity | **superseded** |
| 2026-05-15 | Backend as cache layer + version cop | Better UX than letting indexer reject stale writes | |
| 2026-05-15 | Two-level cache: Postgres + IndexedDB | Fast session startup, instant repeat reads | |
| 2026-05-15 | Edits via pin-new / unpin-old supersedes pattern | Sia objects are immutable; supersedes via metadata is canonical | |
| 2026-05-15 | App ID generated once at project start, hardcoded forever | Per Sia docs; no Foundation approval flow needed | |
| 2026-05-15 | $2K grant line item is infra runway, not storage subsidy | Storage cost at our data scale is negligible | |
| 2026-05-15 | Conflict UI in MVP scope | Rare case but prevents silent data loss | |
| 2026-05-15 | Session revocation in MVP scope | Required mitigation for lost-device attack surface | |
| 2026-05-19 | Per-experiment encryption key derived via HKDF | One key per experiment makes sharing tractable | **superseded** |
| 2026-05-19 | v1 sharing is out-of-band consent-based token | Simple, no on-chain infra; sufficient for MVP | |
| 2026-05-19 | Key rotation = re-encrypt and migrate via supersedes chain | Required given object immutability | |
| 2026-05-19 | CDR on Story is named v2 sharing substrate | Specific commitment to CDR | **superseded** |
| 2026-05-19 | Go backend on Railway (replaces FastAPI/Python) | Sia's native language; mature Go SDK; better ecosystem fit | |
| 2026-05-19 | HTMX + hyperscript + TS crypto island | Hyperscript for UI behaviors, TS for crypto-heavy code | |
| 2026-05-19 | SDK split: client owns crypto, backend owns transport+policy | Backend becomes real policy enforcement layer without holding keys | |
| 2026-05-19 | Three-object data model: definition, data point, typed manifest | Manifest with `type` field unifies experiment / custom / request-response sharing; data points reusable across manifests | |
| 2026-05-19 | Per-data-point keys, per-manifest keys, separately derived via HKDF | Enables sharing of individual data points outside an experiment context | |
| 2026-05-19 | Manifest supersedes forward for continued access | Recipients walk to head; no re-sharing on new data | |
| 2026-05-19 | v2 sharing substrate left generic (no specific product named) | Keeps options open in public doc; specific choice deferred | |
| 2026-05-19 | Departure framed as first-class product principle | Decentralized ID + exports + relayer infra grouped as "Departure" feature set | |
| 2026-05-19 | Immediate cryptographic deletion is not a Sia primitive | Documented limitation; mitigated by external key layer in v2 | |
