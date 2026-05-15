# dApp Ideas — Architecture

> Personal experiments tracker. Privacy-first. User-owned data, stored on Sia.

**Status:** Draft v2 (2026-05-13). Significant revision after auditing v1 against current Sia developer documentation. Open for inline review.

---

## What this is

dApp Ideas is a Progressive Web App for running personal n=1 experiments — peptide protocols, sleep interventions, dietary changes, supplement stacks, anything quantifiable*. Users design experiments, log observations and biometric data over time, and review their own results.

>> * we mention that we are also marketplace? "we help you make experiments quantifiably measurable"

The defining property: **user-generated data is encrypted by the Sia SDK before it leaves the browser and stored on the Sia network under per-user keys**. No centralized database of plaintext user health data exists. Users own their data and can take it with them.

>> defining property should be general; ie, something like "user-generated data is end to end encrypted with uncompromised ux; or the like.. Later we lead upto "Why Sia" from the ground up.

The MVP scope is the personal tracker layer. A researcher query layer, fitness-tracker ingestion, and Story Protocol IP rails are part of the longer-term roadmap but explicitly out of scope for the first build.

>> We again here, show the future and scope it down to MVP; lets them know the gravity of the MVP, and eventual large scale benefit; maybe tie in alignment with Sia Foundation goals at the end as a btw or something

---

## High-level architecture

>> for browser, i want to explore htmx; partly because i want to try it, partly because not many people use it, partly because im tired of react, etc. rest of the flow looks great!
>> why htmx? eventually have this be embedded in contract data somehow and actually enable decentralized marketplace (user-ownability); htmx is much lighter than react and the pattern is often locality of behaviour vs separation of concerns (ofc libs still exist); potentially easier to embed logic in a contract with final data layer being the LOB component.. 

>> for backend, wouldnt the indexer be run through us so we can maintain a cache layer?

```
┌──────────────────────────────────────────────────────────────┐
│                 User's Browser (Vite + React PWA)            │
│                                                                │
│   ├─ Username + password login                                 │
│   ├─ Password unlocks an envelope-wrapped App Key              │
│   │  (the wrapped blob is fetched from our backend)            │
│   ├─ Sia Storage SDK (@siafoundation/sia-storage) signs        │
│   │  indexer requests with the unwrapped App Key               │
│   └─ SDK encrypts objects + metadata before transmission       │
└─────────────┬────────────────────────────┬───────────────────┘
              │                            │
              │ password auth +            │ signed requests +
              │ wrapped-key retrieval      │ pre-encrypted payloads
              ▼                            ▼
┌───────────────────────────────┐   ┌─────────────────────────────┐
│  Our Backend                  │   │  Sia Indexer @ sia.storage  │
│  FastAPI on Railway           │   │  (Foundation-operated)      │
│                               │   │                             │
│  ├─ User auth                 │   │  ├─ Authenticates via       │
│  ├─ Stores: wrappedAppKey,    │   │  │  App Key signatures      │
│  │  salt, recovery wrappers   │   │  ├─ Tracks pinned objects   │
│  ├─ Per-user analytics        │   │  │  per public key          │
│  ├─ Storage billing relay     │   │  ├─ Coordinates with        │
│  │  (infra costs)             │   │  │  storage providers       │
│  └─ NO storage proxy:         │   │  ├─ Manages slab health,    │
│     bytes go browser→indexer  │   │  │  initiates repairs       │
│>> why not we also maintain a  │   │  │  initiates repairs       │
│>> cache layer?                │   │  │  initiates repairs       │
│                               │   │  └─ Never sees plaintext    │
└───────────────────────────────┘   └──────────────┬──────────────┘
                                                   │
                                                   ▼
                                    ┌──────────────────────────────┐
                                    │  Sia Storage Provider Network│
                                    │                              │
                                    │  Encrypted shards across     │
                                    │  many hostd nodes. No single │
                                    │  host can reconstruct data.  │
                                    │  Hosts never see plaintext   │
                                    │  or app-level semantics.     │
                                    └──────────────────────────────┘
```

---

## How Sia actually works (and why we're using it)

>> we should reformat this to be more technical; maybe a table of sia-services (`hostd`, `indexd`, etc.), where they are used, and why
>> also we should lead from breaking down our arch requirements, then build up Sia from the ground up, leading to why Sia;

A quick grounding because some of this is non-obvious if you've only worked with IPFS or S3-style storage.

### The pieces

- **Storage providers** are independent operators running `hostd`. They rent space; they're paid in Siacoin via on-chain storage contracts. They only ever see encrypted shards.
- **Indexers** sit between apps and storage providers. They track which objects live where, coordinate uploads and downloads, monitor slab health, and repair data when redundancy drops below threshold. The recommended hosted indexer is `https://sia.storage`, operated by the Foundation.
- **The Sia network** uses erasure coding to fragment each object into redundant shards distributed across providers, so the loss of multiple hosts doesn't lose your data. Current median network pricing is around **$2.97/TB/month at 3x redundancy** (~$0.01/TB upload, ~$1.55/TB download).
- **Apps** (us) are software identities with a stable 32-byte App ID, chosen once and never changed.

### The per-user model

>> this section should be reshaped as "classic crypto security with ux affordances" and highlight the envelope encryption.. also should be more like a technical doc than conversational

This is the part that changed from v1 of this doc. Sia's auth model is **per-user**, not per-app:

- Each user has their own 12-word BIP-39 recovery phrase
- Combined with our stable App ID, the phrase deterministically derives a per-user **App Key** (a keypair)
- The user approves our app via the indexer once; the indexer registers the user's App Key public key
- All subsequent requests are signed with the App Key; the indexer authorizes by signature
- The indexer associates each uploaded object with the public key that signed for it
- The indexer enforces that only that public key can list, fetch, or delete those objects

Apps are not multi-tenant credentials. We never "speak for" all our users with one master key. Each user's data is bound to their own cryptographic identity, even though our app facilitates the flow.

### Pinning (not what you think)

>> got it; so something like active experiments are pinned and past ones are not? are objects re-pinnable?

In Sia, "pinning" is not the IPFS concept of "don't garbage-collect this from a node's cache." Sia pinning is the operation that **registers an object with the indexer for ongoing health management** — once pinned, the indexer will detect failing shards and initiate repairs to maintain redundancy.

Upload alone erasure-codes and distributes data; pinning makes it listable, syncable, and eligible for repair. We pin everything we want to persist for users.

### Why Sia, not IPFS or Arweave

>> we could again build this up as we need this, ipfs doesnt provide, ipfs out; then arweave out, finally sia fits..

| | IPFS | Arweave | Sia |
|---|---|---|---|
| Default privacy | Public via CID | Public unless you encrypt | Encrypted by SDK |
| Persistence model | Cache-and-hope; need pinning services | Pay-once, store-forever | Contract-based, repaired by indexer |
| Mutability | New CID per change | Immutable | First-class object updates |
| Pricing | Pay each pinning service | One-time large fee | ~$3/TB/mo, market-priced |
| Access control | None (CID = bearer token) | App-layer only | Per-app indexer-enforced |
| Built-in redundancy | You orchestrate it | Yes | Yes (erasure coding) |

For evolving personal health/biometric data — encrypted, mutable, regularly updated — Sia is the natural fit. IPFS lacks encryption and predictable persistence. Arweave's permanent-immutable model is wrong for data that changes daily.

---

## Components

### Frontend

>> 2 things: 1, what is @noble/hashes? ;; 2, we dont need last line of reduntant AES layer

- **Stack:** Vite + React, deployed as a Progressive Web App. Installable on mobile, offline-capable for data entry with sync on reconnect.
- **Sia integration:** `@siafoundation/sia-storage` SDK. SDK handles object sealing (encryption), upload, pin, download, and signed indexer requests.
- **Crypto we add on top:** Argon2id for password-to-key derivation, AES-256-GCM for envelope-wrapping the App Key. Both via `@noble/hashes` / WebCrypto API.
- **No additional encryption layer over user data.** The SDK already encrypts objects and metadata under keys derived from the App Key. Adding our own AES layer would be redundant.

### Backend

>> Should we choose something other than python? go or rust? or sia is on sui, which is move-vm right?
>> We should be a small storage proxy for cache, and maybe concurrency, with a note of v2 moving cache elsewhere to something like cloudflare
>> agreed on what we never store; basically nothing plaintext

- **Stack:** FastAPI + SQLAlchemy + Alembic, Postgres. Deployed on Railway.
- **Role:** Auth, key escrow (envelope-wrapped App Keys), analytics, and billing relay. **Not a storage proxy** — the SDK talks directly from browser to indexer.
- **What it stores per user:** username, salt, wrappedAppKey (under password-derived key), wrappedAppKey_recovery (under recovery passphrase), encrypted-metadata cache for fast session startup, usage counters.
- **What it never stores:** the BIP-39 recovery phrase, plaintext App Keys, plaintext user data, plaintext experiment metadata.

### Storage layer

- **Indexer:** `https://sia.storage` (Foundation-operated). Recommended in the official Sia developer docs as the default for app developers.
- **Storage providers:** the open Sia host network. We don't choose them directly — the indexer does, based on health and price.
- **App identity:** dApp Ideas registers a single 32-byte App ID with the Foundation (likely via an approval/API-key-like flow — see open questions). The App ID is hardcoded in our app for its entire lifetime. Changing it would invalidate every user's data.

>> we should settle app identity layer for the final doc; we cant leave open questions like this.. 

---

## Identity, keys, and login UX

### Design goal

The user experience is a familiar **username + password** flow with an optional **recovery passphrase** as backup. The user never sees, types, or has to save a 12-word BIP-39 seed phrase. The seed phrase exists as an internal implementation detail to bridge to Sia's auth model.

>> we should never take custody of the seed phrase; only the wrapped seed phrase, with auth-method / recovery phrase
>> users still need to save the BIP-39 phrase; we could also store a hash of it for account recovery via seed phrase;

This is a deliberate departure from Sia's recommended UX (which expects users to save the BIP-39 phrase). The justification: our target audience is biohackers and self-experimenters, not crypto-natives, and the seed-phrase ceremony would tank adoption. The recovery passphrase preserves the spirit of user-controlled recovery in a less alienating form.

>> we should structure as "our goal is to move beyond niche communities and enable anyone to track little experiments" even things like switch type of milk, or switched brands, or saw this new exercise on tiktok for my targetted muscle, does it work?, etc.
>> So, providing affordances with ux is very important
>> we do also consider security risks and potential attack surfaces, see below (should add if does not exist)

### Sign-up flow

>> on step 3, why not let client generate ephemeral BIP-39?
>> step 11 is correct
>> maybe a step 12, where we store hash of BIP-39; if user ever returns with only sia recovery phrase

```
On signup:
  1. User provides password
  2. Backend generates: salt, recoveryPassphrase (6-word Diceware)
  3. Backend generates: ephemeral BIP-39 recovery phrase (random entropy)
  4. Client derives App Key from (ephemeralPhrase + our App ID)
       — using the SDK's deterministic derivation
  5. Client goes through indexer approval flow with the new App Key
       — registers App Key public key with the indexer
  6. Client derives: wrappingKey W = Argon2id(password, salt)
  7. Client derives: recoveryKey R = Argon2id(recoveryPassphrase, salt)
  8. Client computes: wrappedAppKey = AES-256-GCM(AppKey, W)
                      wrappedAppKey_recovery = AES-256-GCM(AppKey, R)
  9. Client sends both wrapped blobs to backend; backend persists them
 10. Client displays recoveryPassphrase to user with strong "save this" guidance
 11. Ephemeral BIP-39 phrase is discarded everywhere — never persisted
```

After signup, the ephemeral BIP-39 phrase is gone. The App Key (now wrapped two ways) is the durable secret.

### Login flow

```
On login:
  1. User enters username + password
  2. Backend returns: salt, wrappedAppKey
  3. Client derives W = Argon2id(password, salt)
  4. Client decrypts wrappedAppKey → AppKey
  5. AppKey is held in browser memory for the session
  6. SDK signs all subsequent indexer requests with AppKey
```

### Password change

```
On password change:
  1. Unwrap AppKey with old password
  2. Generate new salt, derive new W from new password
  3. Re-wrap AppKey with new W
  4. Send new wrappedAppKey to backend; replace stored copy
  5. AppKey itself never changes; all user data still accessible
```

### Lost password recovery

```
On recovery:
  1. User enters username + recoveryPassphrase
  2. Backend returns: salt, wrappedAppKey_recovery
  3. Client derives R = Argon2id(recoveryPassphrase, salt)
  4. Client decrypts wrappedAppKey_recovery → AppKey
  5. User sets a new password; re-wrap and persist
```

If both password and recoveryPassphrase are lost, the user's data is unrecoverable. This is the unavoidable cost of true end-to-end encryption with no third-party dependency. We say this plainly in onboarding.

>> except for BIP-39 phrase; basically a third secret recovery phrase

### Username generation

Usernames are generated server-side from a curated wordlist: `curious-axolotl-2847` style. This eliminates uniqueness collisions, removes a friction point in onboarding, and avoids accidental PII in identifiers. Users can change their displayed name later if we add a profile feature; the underlying ID stays stable.

### Future: deterministic Eth address

>> we should maybe frame as deterministic multi-chain addresses from Sia AppKey (or BIP-39 actually)

The App Key derivation produces a keypair we can use for additional purposes. In a future version, we'll derive an Eth address from a sibling HKDF context (using a domain-separated `info` string) so users have a deterministic Ethereum identity tied to their dApp Ideas account, without separate wallet onboarding. This becomes the foundation for Story Protocol integration in v2.

```
appKeySeed → HKDF(info="dapp-ideas:sia:v1")  → Sia App Key (live)
           → HKDF(info="dapp-ideas:eth:v1")  → Eth EOA (future)
           → HKDF(info="dapp-ideas:auth:v1") → app-level signing key (future)
```

Versioned `info` strings let us rotate individual purpose-keys without disturbing others.

---

## Data model

### Objects, not manifests

In v1 of this doc we proposed a per-user manifest blob. We're dropping that — Sia's object model already does the job. We use two object types, both pinned with the user's App Key, both carrying encrypted application-defined metadata:

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

The object body can be small or empty; the metadata blob is the meaningful payload. Experiment objects get rewritten when the user edits the definition (rare).

**Data-point objects.** One per logged observation. Metadata references the parent experiment by object ID.

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

These are append-only — new logged entry creates a new object. No editing existing data points (preserves audit trail). If a user wants to correct, they can mark an older data point as superseded via a future schema field.

>> why should we not allow edits?

### Listing and grouping

To render the user's dashboard:

1. SDK asks the indexer to list all pinned objects for the user's App Key public key
2. Indexer returns object IDs + encrypted metadata blobs
3. SDK decrypts metadata client-side
4. Client groups by type (experiment vs data point) and by `experiment_id`
5. Data-point bodies are lazy-loaded only when the user drills into a specific entry

This is fast: even thousands of objects produce small metadata blobs that decrypt quickly. The actual data bytes only load on demand.

>> See, we can cache all of this and use Sia as backup; lets let our backend be more beefy

### Caching encrypted metadata

>> 2 levels of cache, our backend and client browser (IndexedDB)

To avoid hitting the indexer on every session start, our backend optionally caches the (object_id, encrypted_metadata) tuples per user. It's just opaque ciphertext — the backend cannot decrypt — but it makes session loads instant. Background sync keeps the cache current.

### Why not a manifest blob

>> largely agree; but can remove from doc

We removed the manifest design from v1 because:

- The indexer already provides "list all objects for this user" — no need for us to maintain that mapping
- Per-object metadata is encrypted and app-defined, so we put structured data right where it belongs
- Append-only data points eliminate the concurrency-conflict problem entirely (new objects never collide with existing ones)
- The architecture becomes Sia-native rather than working around its model

---

## Concurrency

>> looks largely correct, but we can let our server be a cop checking version numbers before sending off to Sia (ux affordance)

Because data points are append-only objects (each entry = a new pinned object), there is no last-write-wins risk for ordinary logging. Two devices logging entries to the same experiment simultaneously create two independent objects; both show up in the next list operation.

Concurrency only matters for editing **experiment definitions** (rare: changing schema, title, tags) and **superseding data points** (also rare). For these:

- Object version included in updates
- Indexer's update endpoint accepts a base-version parameter; rejects stale writes
- On rejection, client re-fetches, presents field-level conflict UI (side-by-side diff with "keep mine / keep theirs / keep both" options)

The conflict UI is included in MVP scope because it's small additional work, prevents silent data loss, and signals product quality.

---

## Privacy & threat model

### What's encrypted, where

| Data | Plaintext visible to | Cost |
|---|---|---|
| Experiment data values | User's browser only | no collab / sync |
| Experiment metadata (titles, tags, schema) | User's browser only | IndexedDB storage |
| Data point timestamps | Indexer (operational metadata) | |
| Object sizes | Indexer (slab layout) | |
| Which App Key owns which objects | Indexer (auth model) | |
| Username | Our backend, indexer | |
| Storage usage counters | Our backend | |

The SDK encrypts both the object body **and** the application-defined metadata blob before transmission. Storage providers see only encrypted shards. The indexer sees object IDs, encrypted metadata blobs, slab layouts, and which public key owns what — but cannot read any of it.

### What we (the app operator) can see

- Username (a generated wordlist string)
- Wrapped App Key blobs (cannot unwrap without password or recoveryPassphrase)
- Operational telemetry (which user uploaded how many objects, total bytes consumed)
- Optionally, the encrypted-metadata cache (still ciphertext)

What we cannot see:
- The user's password or master keys
- The plaintext of any experiment data, observation, or biometric value
- The plaintext of any experiment metadata (titles, tags, schemas)
- The user's Sia recovery phrase (it was ephemeral; we never had it)

>> we see the hash of BIP-39 tho

### What we don't protect against

- **Compromised user device.** Anyone with the user's password or unlocked browser sees everything. Same as every E2E system.
- **Sophisticated traffic analysis** at the indexer level. Object sizes, timing, and ownership-by-public-key are visible to the indexer operator.
- **Loss of both password and recovery passphrase.** Data is gone. Documented in onboarding.

>> if providing multiple login methods, we also allow lost phone == make unauthorized additions; need a way to revoke / terminate session (and wrapped key) as well

---

## Storage payment model

**Stated approach:** dApp Ideas pays for all user storage. Users never need to acquire Siacoin, fund accounts, or interact with crypto economics.

**Mechanism (TBD with Foundation):** the exact billing relationship between our App and the `sia.storage` indexer isn't fully documented publicly. Likely options:

- App-level pre-funded balance on the indexer drawn down by all our users' usage
- Pass-through billing back to us based on per-app-ID usage
- Free tier for grant projects during the development period

The $2,000 user-storage subsidy line in the grant proposal funds this regardless of which mechanism turns out to be canonical. With network median pricing at ~$2.97/TB/month at 3x redundancy and experiment data being small (mostly text and numbers), $2,000 covers many hundreds of users for the duration of the grant period and beyond.

>> 2 notes,
>> 1, we figure out if its like gas and contracts, at which point it just becomes sponsored gas; I can try to find this out as well via discord
>> 2, we relabel 2k for infra costs.. and maybe estimate and reach a year with max number of users.. 

---

## Out of scope for MVP

>> we restrucutre this as future plans / roadmap; and also avoid specific mentions to any product.. only generic concepts like nft based royalty / ip layer, smart contract accounts (so not explicitly mentioning 7579)

>> fitness ingestion should be renamed to something like "fitness tech integration" or something

>> regarding crdt, mostly agreed, but just want to take a look again in light of the proposed changes / feedback

>> just native apps, not (just) mobile.. 

>> the object sharing part can be folded into nft / ip layer

>> we should also mention the nostr like service, even if as an exploration

These are deliberately deferred to keep the 8-week build honest. Each is a candidate for a future grant or post-grant work.

- **Researcher query and discovery layer.** Privacy-preserving access for scientists to run aggregate studies on consenting pools of users' data. Requires selective-disclosure encryption schemes; significant scope.
- **Story Protocol integration.** User data points as licensable IP assets. The Eth address derivation in MVP lays the groundwork; actual integration is v2+.
- **ERC-7579 modular smart contract accounts.** Future identity upgrade path. Eth EOA signs SCA deployment when ready. khaaliNames module and WebAuthn module both candidates.
- **Real-time fitness ingestion** (Garmin, Whoop, Oura, CGM). MVP stretch goal is one one-shot import path (Garmin Connect export or Apple Health export). Continuous sync is v2.
- **CRDT-based offline merging.** Current concurrency model is per-object versioning with auto-merge for the append-only case. True offline-first multi-device editing is v2.
- **Mobile-native apps.** PWA covers iOS and Android. Native apps (Swift/Kotlin SDKs exist) come later if the product justifies it.
- **Object sharing via indexer share URLs.** Will be needed for researcher layer; not relevant for personal tracker MVP.

---

## Open questions

Need resolution from Matt or the Foundation before the architecture is final:

1. **App ID registration flow.** Is this a self-serve generate-and-go, or does it go through an approval/API-key-like process with the Foundation? The docs imply per-app registration with the indexer but don't show developer-facing workflow.

>> lets just rewrite doc in more general terms.. also flow seems like a chosen 32 byte hex string that ships with the app ~ https://devs.sia.storage/docs/core-concepts/apps#app-id

2. **Storage payment mechanism.** How does a third-party app pay for its users' storage at `sia.storage`? Pre-funded balance? Pass-through billing? Free tier for grants? This determines the mechanics of our $2K subsidy.

>> not sure yet; going to be sponsoring tier for users; or ideally some kind of enterprise deal..

3. **Approval flow programmatic-vs-manual.** The "Connect to an Indexer" docs describe an approval URL the user opens to grant the app access. For our flow (where we generate the recovery phrase ephemerally and the user never sees it), is there a programmatic path that handles this without a user-facing redirect?

>> yes we create a BIP-39 on the client, which has our App ID; we ask user to save the BIP-39; client derives App Key, hashes BIP-39, wraps App Key with auth method, and sended username, salt*, BIP-39 hash, wrappedKey_auth_method; and some other convenience things like version and the like

4. **Erasure-coding configuration at sia.storage.** What's the actual redundancy ratio? Is it configurable per object/app, or fixed?

>> seems like it is editable ~ https://devs.sia.storage/docs/core-concepts/erasure-coding#tunable-durability

5. **Per-app usage telemetry.** Does the indexer publish per-App-ID usage stats via API for our backend to consume, or only via the indexer's own dashboard?

>> tbh not sure; also not clear here ~ https://devs.sia.storage/docs/core-concepts/indexers

6. **Spirit-of-the-guidance check on hiding the recovery phrase.** Our design generates the BIP-39 phrase ephemerally and discards it after deriving the App Key, replacing the user-facing recovery with a Diceware passphrase that wraps a second copy of the App Key. Functionally sound, but worth a sanity check: is this an acceptable interpretation of "the recovery phrase must never be stored by the app, but instead stored securely by the user"? We're respecting the letter (no storage of the phrase) but bending the spirit (user doesn't hold the canonical Sia recovery).

>> see all other comments about new proposed design

---

## Decisions log

>> not reviewing this, will review this at the absolute end, lets just keep appending to this as we move through versions

| Date | Decision | Rationale |
|---|---|---|
| 2026-05-12 | Use `https://sia.storage` as indexer | Foundation-recommended public indexer |
| 2026-05-12 | Skip s3d | Greenfield app; direct SDK is cleaner |
| 2026-05-12 | FastAPI on Railway | Lightweight, fits team skills |
| 2026-05-13 | Per-user App Key model (Sia-native) | Correcting v1's mistaken multi-tenant assumption |
| 2026-05-13 | No additional encryption layer over user data | SDK already encrypts; redundant |
| 2026-05-13 | No storage proxy in backend | SDK talks browser→indexer directly |
| 2026-05-13 | Hide BIP-39 phrase from users | Target audience is non-crypto; envelope-wrap App Key under password + Diceware recovery |
| 2026-05-13 | Objects with encrypted metadata, no separate manifest | Sia-native; eliminates concurrency problem for append-only data |
| 2026-05-13 | Append-only data point objects | Audit trail; trivial concurrency |
| 2026-05-13 | Conflict UI for experiment-definition edits | Rare case but worth protecting against silent data loss |
| 2026-05-13 | We pay for storage on behalf of users | UX: users never deal with Siacoin or storage costs |
| 2026-05-13 | Derive Eth EOA from same key seed (future use) | Forward path to Story Protocol without separate wallet |
