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

---

## Vision and MVP scope

The long-term picture is three connected modules:

1. **A personal experiments tracker** where individuals run their own n=1 studies in private. (This MVP.)
2. **A researcher layer** that lets scientists discover and run privacy-preserving aggregate studies across pools of consenting users — turning thousands of personal experiments into population-scale insight without ever centralizing the data.
2b, **Consumer Brands** that lets companies make verifiable claims (eg, this workout gets results 30% faster, or this new "health snack" increases digestible fiber by x%, etc.)
3. **An IP and royalty layer** so the users whose data contributes to published research and studies can be recognized and rewarded.

This grant funds only the first layer. Without solid storage and identity primitives, the rest doesn't work. The MVP delivers a working tracker that users would actually use, on a storage layer that's architecturally ready for the layers above.

> The Sia Foundation's mission of *user-owned data* maps directly onto what we're trying to build. Personal health, biometric, and behavioral data is the most sensitive personal data category that exists, and today it lives in walled gardens. We want to invert that — and Sia is the substrate that makes the inversion technically real, not just a slogan.

## Architectural requirements

core requirements from a storage and identity stack: E2EE, data mutability and archival flexibility, ~permissionless access; more specifically,

| Contstraint | Why? | IPFS | Arweave | Sia |
| ----------- | ---- | ---- | ------- | --- |
| E2EE, user-held keys | sensitive health data | external | external | AppKey |
| E2EE, sharable ephemeral keys | domain restricted layer-2 access | external | external | |
| Mutability | growing datasets | new CID | immutable* | `indexd` "[pinning](https://devs.sia.storage/docs/core-concepts/pinning)" |
| Persistence | archived datasets | external | native | `indexd` ["unpin"](https://devs.sia.storage/docs/core-concepts/pinning#unpinning-and-deleting) / `hostd` |
| Cost Abstraction | no crypto ux | filecoin | pay once | siacoin / siachain |
| Diverse clients | wide support | abstractable | abstractable | [official sdks](https://devs.sia.storage/docs/quickstart/index#install-the-sdk) |
| Composability | easy integration, hacker friendly | | | [app-defined metadata](https://devs.sia.storage/docs/recipes/object-metadata) |
| Open Gardens | compete on service | | | |

> \* external implies third party or self developed / hosted

Overall, IPFS and Arweave can be utilized with a bunch of tooling on top, but with Sia, we reduce our work considerably. Specifically, Sia offers an encryption mechanism, near comprehensive data flexibility, and ready support.

## High-level architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              User's Browser (HTMX + JS islands)                  │
│                                                                   │
│   HTMX-driven HTML UI                                             │
│   JS layer:                                                       │
│     ├─ Argon2id + AES-GCM for envelope ops                        │
│     ├─ HKDF for derived keys                                      │
│     ├─ Sia Storage SDK (@siafoundation/sia-storage)               │
│     └─ IndexedDB cache (decrypted metadata for this session)      │
└────────────┬─────────────────────────────────┬──────────────────┘
             │                                 │
             │ HTML fragments / auth /         │ Signed requests +
             │ cached metadata reads           │ pre-sealed payloads
             ▼                                 ▼
┌────────────────────────────────┐   ┌─────────────────────────────┐
│  Our Backend                   │   │  Sia Indexer @ sia.storage  │
│  FastAPI on Railway            │   │  (Foundation-operated)      │
│                                │   │                             │
│  ├─ Auth + session mgmt        │   │  ├─ Authenticates via       │
│  ├─ Renders HTMX fragments     │   │  │  App Key signatures      │
│  ├─ Stores wrapped App Keys    │   │  ├─ Tracks pinned objects   │
│  ├─ Encrypted-metadata CACHE   │   │  │  per public key          │
│  │  (still ciphertext to us)   │   │  ├─ Coordinates with hosts  │
│  ├─ Version cop: validates     │   │  ├─ Manages slab health,    │
│  │  base-version before each   │   │  │  initiates repairs       │
│  │  indexer write              │   │  └─ Never sees plaintext    │
│  └─ Per-user telemetry         │   │                             │
└────────────────────────────────┘   └──────────────┬──────────────┘
                                                    │
                                                    ▼
                                    ┌──────────────────────────────┐
                                    │  Sia Storage Provider Network│
                                    │                              │
                                    │  Encrypted shards across     │
                                    │  many hostd nodes. No single │
                                    │  host can reconstruct data.  │
                                    │  Hosts never see plaintext.  │
                                    └──────────────────────────────┘
```

Notable: the SDK still talks **directly** to the indexer for actual storage operations (upload/download bytes don't tunnel through our backend), but our backend holds an encrypted-metadata cache and acts as a version validator before writes, both of which significantly improve UX.

>> No, here can we not split the sdk into two? on the client they do the key gen and sign / encrypt; and we on the backend do version control and time / count bound updates to the indexer.. is this not possible?

>> Also, how about Go as the backend? Definitely not JS, kinda don't want to do Python.. So, Go or Rust? Both have servers with templates support for htmx rendering

---

## Components

### Frontend — HTMX with JS islands
> A coping mechanism for react fatigue 

**Stack:** HTMX-driven HTML rendered by FastAPI, with a JavaScript module for cryptographic operations and SDK calls. Alpine.js or vanilla JS for any client-side state beyond what HTMX handles natively. Installable PWA.

>> for js support, i want to use hyperscript ~ https://hyperscript.org/

**The crypto-island pattern.** Pure HTMX can't drive encryption — the SDK and our key operations live in the browser. The pattern:

1. Server renders HTML with form fields and HTMX attributes for navigation and pure-data updates.
2. Submit events that involve crypto get intercepted by a JS handler (`hx-on:submit` or equivalent).
3. JS handler: derives keys, calls SDK to encrypt + upload, then triggers HTMX to refresh the affected fragment from the server's cache.
4. Pure-render operations (listing experiments, loading dashboards) flow normally through HTMX, with the JS layer providing decrypted content from IndexedDB.

This is honest about the hybrid: HTMX where it makes sense, JS where it must.

**Crypto libraries.**

- `@noble/hashes` — an audited, dependency-free crypto library. Provides Argon2id (password-to-key derivation) and HKDF (deriving multiple purpose-specific keys from a single seed). Chosen for being small, auditable, and not pulling in a sprawling dependency tree.
- WebCrypto API (built into browsers) — for AES-256-GCM symmetric encryption used in envelope-wrapping the App Key.
- `@siafoundation/sia-storage` — the JS SDK that handles object sealing and indexer communication.

### Backend — FastAPI on Railway, with cache role

**Stack:** Python + FastAPI + SQLAlchemy + Alembic + Postgres, deployed on Railway. Renders HTMX fragments and handles auth.

>> okay yeah stack looks pretty glued together, lets switch to some framework in Go or Rust (Go because sia is natively in go, rust because been meaning to try it).. tbh if rust's feature is data and program integrity, and go's feature is amazing concurrency, rust seems the better choice here.. forcing better decisions for a critical foundation.. thoughts?

**Roles:** general ux improvement, [encrypted] key escrow, data versioning and cache, some usage stats, connectors

- **Auth + sessions.** Username/password login. Session tokens. Multi-device session management including revocation. Note, username is server generated, not user selected.
- **Key escrow.** Stores `wrappedAppKey` (under password-derived key) and `wrappedAppKey_recovery` (under Diceware passphrase-derived key) per user. Stores hash of the user's BIP-39 phrase for the third recovery path. Never sees plaintext keys.
- **Encrypted-metadata cache.** Mirrors the user's pinned-object list and their encrypted metadata blobs for faster start-ups.
- **Version cop.** Manages versioning of data, eventually CRDTs
- **Per-user telemetry.** User level storage counter and consent base app performance metrics

### Storage layer

Primarily managed by the Official SDKs that interact with indexers and hosts.

Connected indexer :: `https://sia.storage`

- **Indexer:** `https://sia.storage` (Foundation-operated). Recommended in the official Sia developer docs as the default for app developers.
- **Storage providers:** the open Sia host network. The indexer selects and manages contracts based on health and price; we don't pick hosts directly.
- **Erasure coding:** Sia supports tunable durability per object. We'll choose the default redundancy setting unless testing reveals reason to tighten it for specific object types.

> [!TIP]
> APP_ID = `keccak256("com.dapp.ideas/alpha")` = \
> `0x8a8fbc3f4562a759a7ff2a7bbe961df6a719fd660c4c5dd6e691a4a795b73382`

Pinning :: We pin all since [currently](https://devs.sia.storage/docs/core-concepts/pinning#unpinning-and-deleting) the sdk treats "unpin" and "delete" as the same operation

## Identity, keys, and login UX

We follow web2 best practices and generate `AppKey` on first time password setup, and then store encrypted versions for different auth methods. We also use the base password for HKDF.

### User Seed

_todo_

### Cryptographic primitives

- **BIP-39** — 12-word recovery phrase. The user's master secret per Sia's auth model. We never store the phrase itself; the user holds it.
- **Argon2id** — memory-hard password-to-key derivation. Used for password and recovery-passphrase unwrapping.
- **HKDF** — derives multiple purpose-specific keys from a seed via domain-separated `info` strings.
- **AES-256-GCM** — symmetric encryption for envelope-wrapping the App Key.

### Sign-up flow

```
On signup:
  1.  User provides password
  2.  Client generates: salt (16 bytes), recoveryPassphrase (6-word Diceware)
  3.  Client generates: BIP-39 recovery phrase (random entropy, 12 words)
  4.  Client derives App Key from (BIP-39 phrase + our App ID)
        using the SDK's deterministic derivation
  5.  Client connects to indexer with the new App Key
        — indexer registers the App Key's public key
  6.  Client derives: wrappingKey W = Argon2id(password, salt)
                      recoveryKey R = Argon2id(recoveryPassphrase, salt)
  7.  Client computes:
        wrappedAppKey          = AES-256-GCM(AppKey, W)
        wrappedAppKey_recovery = AES-256-GCM(AppKey, R)
        bip39_hash             = SHA-256(BIP-39 phrase)
  8.  Client sends to backend: username, salt, wrappedAppKey,
                               wrappedAppKey_recovery, bip39_hash
  9.  Backend persists these. Backend never sees the BIP-39 phrase,
        the App Key, or the password.
  10. Client displays to user:
        - The BIP-39 phrase, with strong "save this" guidance
          (this is your master recovery — write it down)
        - The recoveryPassphrase, with "save this too" guidance
          (this is your convenience recovery)
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
  6. SDK signs all subsequent indexer requests with AppKey
```

### Password change

```
On password change:
  1. Unwrap AppKey with old password
  2. Generate new salt, derive new W
  3. Re-wrap AppKey → new wrappedAppKey
  4. Send to backend; replace stored copy
  AppKey itself never changes; all data still accessible.
```

### The three recovery paths

| Path | Use case | UX |
|---|---|---|
| **Password / Auth Methods ** | Daily use | Familiar; enter password to log in |
| **Recovery passphrase** (Diceware) | Forgot password, have passphrase | Type 6 words to reset password |
| **BIP-39 phrase** | Forgot password *and* passphrase, OR want to leave dApp Ideas with full data control | Type 12-word phrase; backend verifies via stored hash; derives App Key fresh; user sets new password |

> [!NOTE]
> If all three paths lead to lost secrets, data will be unrecoverable \
> unless we store plaintext; or attempt redundant storage and keys

### Username generation

Usernames are generated server-side from a curated wordlist: `curious-axolotl-2847` style. Eliminates uniqueness collisions, removes a friction point in onboarding, and avoids accidental PII in identifiers. A separate display-name field can be added later for personalization.

Additionally, this is very compatible with name service providers on other chains that abstract crypto ux while also enabling easier departures

### Multi-chain key derivation (future)

>> we could do `"dapp-ideas:experiment:{id}:alpha"` and `"dapp-ideas:data:{id}:alpha"` and store the `{id}` for both and their overall storage costs and other relevant metadata.. then, while sharing, we can generate these specific keys and share them

The BIP-39 phrase is the natural root for HD-wallet-style multi-chain identity and for per-resource encryption keys. Via domain-separated HKDF derivations:

```
BIP-39 phrase → master_seed (via BIP-39 / BIP-32)
              → HKDF(info="dapp-ideas:sia:alpha")             → Sia App Key
              → HKDF(info=f"dapp-ideas:experiment:{id}:v1") → per-experiment encryption key
              → HKDF(info="dapp-ideas:eth:alpha")             → Eth address (future)
              → HKDF(info="dapp-ideas:auth:alpha")            → app-level signing key (future)
              → HKDF(info="dapp-ideas:[chain]:alpha")          → other chain identities (future)
```

The user has one phrase, but a deterministic identity on every chain we ever support and a deterministic key for every experiment they create. Versioned `info` strings allow per-purpose key rotation without touching the master. The `experiment_id` itself serves as the nonce input, so distinct experiments yield uncorrelated keys.

## Data model

Sia's object model does the work that a per-user manifest blob would otherwise have to do. We use two object types, both pinned with the user's App Key, both carrying encrypted application-defined metadata.

**Objects are immutable.** Each object's ID is a content-derived hash of its slab layout, so changing the data produces a new object with a new ID. This is by design — the indexer never updates objects in place. This property shapes how we handle edits below.

**Objects within an experiment share a key.** Every object that belongs to a given experiment — the experiment object itself and all of its data points — is encrypted with the same `experiment_key`, derived deterministically from `(BIP-39 phrase, experiment_id)` via HKDF. This is the design choice that makes sharing tractable: granting access to an experiment means sharing a single key, not re-wrapping one key per data point. Distinct experiments use uncorrelated keys, so leaking one experiment's key doesn't compromise any others.

>> The reason we want to avoid this is because some data, independent of experiment, may be useful.. for example, consider blood work done for a vitamin-d experiment, but the lab returns some standard measures as well (eg, other vitamins); then this raw data should licensible, and needs its own key

>> one thing we could do is upload twice.. once with the data key and once with the experiment key, then experiment can be updated to include a reference to its id.. but wait, objects are immutable, so thats an issue.. is this where the App layer (us) comes in and shines?

**Experiment objects.** One per experiment. Metadata holds the experiment definition.

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

**Data-point objects.** One per logged observation. References the parent experiment by object ID.

```json
{
  "type": "data_point",
  "experiment_id": "<experiment_object_id>",
  "logged_at": "2026-06-15T08:14:00Z",
  "fields": {
    "dose_mg": 5,
    "energy_1_10": 8,
    "notes": "noticeable bump after 20 min"
  }
}
```

**Edits use a supersedes pattern.** Because objects are immutable, an edit isn't an in-place update — it's a new object that points back at its predecessor. The client uploads a new object containing the updated content plus a `supersedes` field referencing the previous object's ID, pins the new object, and unpins the old one. Unpinning releases the old object from active health management; its shards remain readable until host contracts naturally expire. Following the supersedes chain reconstructs the full history of a logical entry. Users can optionally keep prior versions pinned if they want explicit audit retention.

>> If edits use a supersede pattern, should we explore tree options? or maybe merkle? might simplify our types as well, but complicate reads and reconstruction; maybe not..

### Data Access

1. Client first checks IndexedDB (because, web and pwa) for any decrypted metadata and stored configs
2. Requests backend for encrypted cache, and other user preferences
3. SDK encrypts/decrypts any metadata and handles key-gen client side
4. Experiment data lazy loaded when requested via UI

## Caching strategy

>> overall seems fine, but might remove section in final version

Two layers of cache, plus the durable source on Sia:

| Layer | What | Latency | Mutability |
|---|---|---|---|
| **Client IndexedDB** | Decrypted experiment metadata + recently-viewed data points | Instant | Cleared on logout / cache invalidation |
| **Server Postgres** | Encrypted metadata blobs + object listings + per-user telemetry | <50ms typical | Updated on every write (server is the version cop) |
| **Sia indexer + hosts** | Canonical, durable storage | Higher (single-digit seconds first hit) | Source of truth |

**Cache invalidation.** Writes invalidate the relevant entries in both caches. Background sync ensures cross-device updates propagate within a short window.

**v2 evolution.** The server cache moves to an edge layer (Cloudflare KV/R2 or similar) once we have enough users that Postgres becomes a hot spot. The migration is straightforward because the cache is just key→ciphertext.

## Concurrency

>> i strongly think we should atleast some form of tree structure to track instead. maybe for experiment and data linking, or even dataset linking; for simple data point version control we can follow the supersede pattern.. unless we want to support "view edits" (w/o a loop)

Most writes are appends — logging a new data point is a brand-new object that can't conflict with anything. For appends, no concurrency machinery is needed.

For **edits** to experiment definitions or to existing data points (rare but allowed), concurrency is checked via the supersedes chain:

1. Every object the client edits references a `supersedes` target — the object it intends to replace.
2. Our backend (version cop) checks whether the targeted object is still the current head of its supersedes chain — i.e., that no other edit has superseded it in the meantime.
3. If yes: accept, forward the new object to the indexer for upload + pin, and unpin the predecessor.
4. If no (some other device already edited this object): reject early, return the current head and the conflicting version.
5. On rejection, the client presents a **field-level conflict UI** (side-by-side diff with options: "keep mine," "keep theirs," "keep both as separate entries"). User's resolution generates a new edit that supersedes both branches.

Putting the chain-head check in our backend rather than relying on the indexer is a UX affordance — failures come back faster, and we can present them with more context.

## Sharing and key rotation (v1)

Because every object in an experiment is encrypted under the same `experiment_key`, sharing is a one-key operation. The v1 flow is deliberately simple — there is no on-chain access control yet; that's a v2 concern (see Future plans).

>> i think there is merit in having per datapoint keys as well; can be derived from AppKey, that's fine

### Sharing an experiment

```
On share:
  1. User selects "Share this experiment" in the UI
  2. App computes a share token:
       {
         experiment_id: "<id>",
         experiment_key: "<base64>",
         indexer: "https://sia.storage",
         shared_by: "<sender's public username>"
       }
  3. App offers the user a few ways to deliver the token:
       - Copy as a string
       - Generate a one-time URL to a "view this experiment" landing page
       - (Future) Send via a sharing-capable channel
  4. Recipient receives the token, opens dApp Ideas in import-only mode,
     pastes the token. App fetches the experiment + all data points by id,
     decrypts client-side with the shared key, renders read-only.
```

>> overall flow looks ok, might revisit later

A recipient with a share token can read everything in that experiment, including future data points the original user adds (since they're encrypted under the same key). They cannot write, edit, or share onward in v1 — though nothing stops them from forwarding the token they received, which is the inherent limitation of any out-of-band sharing model.

>> on continued access, if objects are immutable and generate a new id on updates, doesn't that mean the recepient also needs the reference? if so, then we could do continual access by sharing updated references

### Key rotation (revoking shared access)

Because objects are immutable, "rotating an experiment key" really means migrating an experiment to a new key. The flow:

```
On rotation:
  1. App derives a new key using a bumped version suffix:
       new_key = HKDF(info=f"dapp-ideas:experiment:{experiment_id}:v2")
       (or any monotonic counter)
  2. App reads all pinned objects of the experiment with the old key
  3. App re-encrypts each object's payload with the new key,
     uploads as new objects with supersedes pointers to their predecessors,
     pins the new ones, unpins the old ones
  4. Past share tokens (still bound to the old key) can read old objects
     until host contracts expire, but cannot read anything created after
     rotation
```

>> can we immediately expire host contracts?

### What v1 doesn't do

>> idk how much of this section i want to really keep vs incorporate into language elsewhere

- No on-chain access conditions. Sharing is consent-based and out-of-band.
- No granular per-data-point sharing within an experiment. The unit of sharing is the experiment.
- No automatic licensing or royalty flows. Recipients can read; what they do next is governed by trust, not contracts.

All three are addressed in the CDR-mediated v2 design.

## Privacy & threat model

>> will read this in the future version

### What's encrypted, where, at what cost

| Data | Plaintext visible to | Cost of this choice |
|---|---|---|
| Experiment data values | User's browser only | No server-side collab or sync features |
| Experiment metadata (titles, tags, schema) | User's browser only | IndexedDB / server cache stores ciphertext |
| Data point timestamps | Indexer (operational metadata) | Some traffic-analysis risk |
| Object sizes | Indexer (slab layout) | Some traffic-analysis risk |
| Which App Key owns which objects | Indexer (auth model) | Indexer knows our users' public keys |
| Username | Our backend, indexer | Just a generated wordlist string |
| Storage usage counters | Our backend | Operational telemetry only |
| Hash of user's BIP-39 phrase | Our backend | Used only for recovery validation |

The SDK encrypts both the object body **and** the application-defined metadata blob before transmission. Storage providers see only encrypted shards; the indexer sees encrypted metadata blobs and slab layouts but cannot read any of it.

### What we (the operator) can see

- Username (a generated wordlist string)
- Wrapped App Key blobs (cannot unwrap without password, recovery passphrase, or BIP-39 phrase)
- Hash of the BIP-39 phrase (cannot reverse to phrase)
- Operational telemetry (how many objects, total bytes)
- Encrypted-metadata cache (still ciphertext)

### What we cannot see

- The user's password
- The user's BIP-39 phrase (only its hash)
- Plaintext App Keys
- Any plaintext experiment data, observation, or biometric value
- Any plaintext experiment metadata (titles, tags, schemas)

### Attack surfaces and mitigations

| Attack | Risk | Mitigation |
|---|---|---|
| Compromised user device | Attacker reads everything | Standard E2E limitation. Session revocation lets the user invalidate other sessions remotely. |
| Stolen laptop with active session | Attacker continues as user | "Revoke all sessions" + force re-auth + password rotation. App Key + data unchanged. |
| Database breach at our backend | Wrapped keys + ciphertext leaked | Attacker still needs password, recovery passphrase, or BIP-39 to decrypt. Argon2id makes brute-forcing expensive. |
| Brute-force on Argon2id | Weak passwords cracked offline | Memory-hard parameters; enforce minimum password strength; rate-limit auth endpoints. |
| Traffic analysis at indexer | Object sizes/timing/ownership visible | Accepted limitation. The indexer is a trust-but-verify component. |
| Lost both password and Diceware | User locked out unless BIP-39 saved | The BIP-39 phrase is the canonical recovery; clearly framed at signup. |
| Lost all three recovery paths | Data is gone | Documented honestly. The cost of real E2E. |
| Phishing site mimicking dApp Ideas | User enters credentials, attacker derives keys | Standard web app limitation. Future: domain attestation, passkey support. |

## Cost and capacity estimation

>> similarly will check this out when the architecture is locked down

**Stated approach:** dApp Ideas pays the operational costs of running the service. Users never acquire Siacoin, fund accounts, or interact with crypto economics.

**Where the $2,000 line item goes:** infrastructure costs over the grant period and runway beyond.

| Item | Estimate (monthly) |
|---|---|
| Railway hosting (FastAPI + Postgres) | $20–50 |
| Domain, SSL, basic monitoring | $5–15 |
| Sia storage costs at our data scale | Trivially small — see below |
| Email or other comms (if needed) | $10–20 |
| Buffer for traffic spikes / overages | $20–50 |
| **Estimated total** | **$55–135 / month** |

**Storage costs are negligible at our data scale.** Experiment metadata and data points are mostly text and numbers. A heavy user generating one data point per day with 200 bytes of payload accumulates ~73 KB per year of original data — about 220 KB at 3x redundancy. At ~$3/TB/month, a thousand such users for a year would cost a few cents.

**Runway implication.** $2,000 covers 15–36 months of infrastructure at MVP scale, with storage cost remaining noise relative to compute and bandwidth. Even at 10,000+ active users (well past MVP scope), storage cost remains a small fraction of infra cost.

**On storage payment mechanism with the Foundation.** We're following up with the Foundation on whether app-level storage works like sponsored gas — we pay on behalf of users via a pre-funded balance, pass-through billing, or some other model. This affects the *mechanics* of the payment flow but not the overall capacity story.

## Future plans (roadmap)

These are deliberately deferred to keep MVP scope honest. Each is a candidate for a follow-on grant or post-grant work.

>> we should rewrite this to be generic; ie, ip registration and licensing layer and not story or cdr

- **Confidential Data Rails (CDR) integration on Story.** The v2 sharing layer. Per-experiment keys are uploaded to CDR vaults on Story; access conditions are expressed as programmable smart contracts ("anyone holding this license NFT can read," "decryption is allowed only inside a TEE running this attested model-training binary," etc.). A decentralized network of TEEs performs threshold decryption when conditions are met. The Sia storage layer doesn't change at all — only the key-management and access-control surface evolves. This is the architecturally clean substrate for the researcher query layer, royalty flows, and any future automated data-deal patterns.
- **Researcher discovery and selective sharing.** Built on CDR. Researchers discover experiments matching study criteria; users opt in per-experiment; access is granted programmatically; royalties flow back. Combines naturally with Story's existing IP and licensing primitives.
- **IP and royalty layer on Story.** Experiments and data points registered as on-chain IP assets via Story Protocol, with royalties flowing to contributors when their data is used in published research. Builds on the CDR-mediated sharing infrastructure above.

>> "departure" as a feature (or something catchy like this): decentralized id, well formatted exports, nostr-esque relayers

- **Decentralized identity primitives.** Smart-contract-account-style identity built on the multi-chain keys we already derive. Modular validators (passkey-based authentication, name resolution from external naming services, social recovery, etc.).
- **Real-time discovery infrastructure.** A Nostr-like relay layer for discoverable experiments and aggregate analyses across users who opt in. Worth exploring as an alternative or complement to indexer-based discovery.

>> Wide device support

- **Native applications.** PWA is the MVP target, covering web and mobile installs. Truly native applications via Swift/Kotlin SDKs once the product justifies it.
- **Fitness tech integration.** Continuous sync from health platforms (Garmin, Whoop, Oura, CGMs, body composition scales). MVP stretch goal is one one-shot import path
- **Offline-first multi-device with CRDTs.** Worth revisiting in light of the version-cop concurrency model — it may be sufficient, or CRDTs may still earn their complexity if multi-device offline editing becomes a frequent pattern.

## Open questions

The genuinely unresolved items at this point in the design:

1. **Storage payment mechanism.** Pre-funded balance? Pass-through billing? Sponsored-gas-style pool? Awaiting clarification from the Foundation (we'll ask in Discord). Doesn't change the architecture; affects implementation of the payment flow.

>> we pay for all storage; in the future we might do plans and subscription based service

2. **Per-app usage telemetry from the indexer.** Whether the indexer exposes per-App-ID usage stats via API or only via its dashboard. Affects whether we need to track byte-level usage ourselves on the backend or can pull it from the indexer. Doesn't block MVP.

>> no like i said, lets sit in the middle

---

## Decisions log

>> will likely remove from final version, but lets keep this to track

| Date | Decision | Rationale |
|---|---|---|
| 2026-05-12 | Use `https://sia.storage` as indexer | Foundation-recommended public indexer |
| 2026-05-12 | Skip s3d gateway | Greenfield app; direct SDK is cleaner |
| 2026-05-15 | Per-user App Key model (Sia-native) | Correcting v1's mistaken multi-tenant assumption |
| 2026-05-15 | Pin all of a user's data while active; future archive flow can unpin | Simple MVP; storage cost is negligible anyway |
| 2026-05-15 | Three recovery paths: password, Diceware, BIP-39 | Convenience for casual users, durability for power users |
| 2026-05-15 | Client generates BIP-39 phrase; user holds it; backend stores only hash | Sia-spirit-respecting; user remains in full control of canonical recovery |
| 2026-05-15 | Multi-chain key derivation from BIP-39 (HD-wallet pattern) | Forward path to other chains without separate wallets |
| 2026-05-15 | HTMX + JS islands for crypto operations | Lighter than React; honest about needing JS for SDK calls |
| 2026-05-15 | FastAPI on Railway (Python) | Dev velocity for MVP; Go is the natural future migration |
| 2026-05-15 | Backend as cache layer + version cop | Better UX than letting indexer reject stale writes |
| 2026-05-15 | Two-level cache: Postgres server-side + IndexedDB client-side | Fast session startup, instant repeat reads |
| 2026-05-15 | Edits via pin-new / unpin-old supersedes pattern | Sia objects are immutable by design (object ID = content hash); supersedes via metadata field is the canonical pattern |
| 2026-05-19 | Per-experiment encryption key derived via HKDF from BIP-39 phrase | One key per experiment makes sharing tractable (one operation, not N); HKDF with experiment_id as nonce keeps keys uncorrelated |
| 2026-05-19 | v1 sharing is out-of-band consent-based token | Simple, no on-chain infra; sufficient for MVP; clear v2 upgrade path |
| 2026-05-19 | Key rotation = re-encrypt and migrate via supersedes chain | Required for revocation given object immutability; rare operation; cleaner with CDR in v2 |
| 2026-05-19 | CDR on Story is the named v2 sharing substrate | Concrete architectural commitment for the researcher and IP layers; CDR is purpose-built for this exact pattern |
| 2026-05-15 | App ID generated once at project start, hardcoded forever | Per Sia docs; no Foundation approval flow needed |
| 2026-05-15 | $2K grant line item is infra runway, not storage subsidy | Storage cost at our data scale is negligible; infra is the real cost |
| 2026-05-15 | Conflict UI in MVP scope | Rare case but prevents silent data loss |
| 2026-05-15 | Session revocation in MVP scope | Required mitigation for lost-device attack surface |
