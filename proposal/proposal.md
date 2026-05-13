# dApp Ideas: Personal Experiments Tracker

## Introduction

### Project Name

dApp Ideas: Personal Experiments Tracker

### Name of the organization or individual submitting the proposal

Palash A. / khaaliDimaag

I am a builder in the Web3 space, building for the past 5 years, though I first discovered Bitcoin over a decade ago through a class on cryptography. Since then I have experimented with a lot of protocols and technologies, and mainly participated in hackathons as I ideated on longer term web3-native projects. dApp Ideas is one such project. *khaaliDimaag* is the umbrella for my projects exploring the idea of a *sovereign individual and network states* ([https://thenetworkstate.com/](https://thenetworkstate.com/)).

Recent projects ::

- Battle Trade (Paper Trading Perps to get people comfortable with trading) :: [https://battle.fyi](https://battle.fyi)  
- khaaliNames (onchain user management) :: [https://github.com/khaaliDimaag/khaaliNames](https://github.com/khaaliDimaag/khaaliNames)

### Describe your project

dApp Ideas is a privacy-first web app, built as an installable Progressive Web App, for running personal n=1 experiments. Users design protocols such as  testing peptide stacks, sleep interventions, dietary changes, supplement regimens, or anything quantifiable, log observations and biometric data across the experiment window, and review their own results over time.

Every data point is encrypted on the user's device before it ever leaves the browser, and stored on Sia via the indexd SDK. Users hold their own keys; the app holds nothing readable. There is no centralized database of plaintext user health data.

The MVP funded by this grant covers experiment creation and logging, manual data entry (text observations plus structured numerical and biometric fields), encrypted client-side upload to Sia, retrieval and visualization, and user-controlled key management. If schedule permits, a stretch goal adds a single fitness-data import path \~ most likely Garmin Connect or Apple Health export, decided based on early-user feedback.

The longer-term vision, outside this grant's scope, is a researcher query layer that lets scientists run privacy-preserving large-scale studies against pools of consenting users' data, with IP and licensing rails via Story Protocol. The current grant funds only the personal storage and tracker layer that the rest of the roadmap depends on.

### How does the projected outcome serve the Foundation's mission of user-owned data? What problem does your project solve?

Health and biometric data is the most sensitive personal data category that exists, and today it sits in walled gardens (Apple Health, Whoop, Oura, MyFitnessPal, Strava) where users have no real ownership, no portability, and limited recourse if a platform changes its terms or shuts down. The quantified-self and personal-experimentation communities are growing rapidly (peptide protocols, longevity stacks, biohacking, sleep optimization) but have nowhere private to store the data they generate.

dApp Ideas inverts the model. Data is encrypted before it leaves the user's device and stored on Sia under keys only the user controls. The app cannot read user data; neither can a future researcher, neither can a future acquirer. Users can take their data with them, fork the app, or self-host the frontend at any time.

This is the literal expression of "user-owned data." It also positions Sia as the natural storage layer for the emerging DeSci (decentralized science) ecosystem — a category where Sia's architecture is uniquely suited compared to its peers. IPFS lacks native encryption and requires recurring pinning fees for persistence, and content-addressing makes streaming biometric data awkward because every update mints a new CID. Arweave is pay-once-immutable, the wrong economic model for evolving experiment data that updates daily. Sia's mutable, contract-based, natively encrypted storage is the right primitive for personal scientific data.

### Are you a resident of any jurisdiction on the FATF / OFAC sanctions list?

No

### Will your payment bank account be located in any jurisdiction on the FATF / OFAC sanctions list? 

No

## 

## Grant Specifics

### Amount of money requested and justification with a reasonable breakdown of expenses

**Total: $10,000 USD**

| Line item | Amount | Notes |
| :---- | :---- | :---- |
| Development (2 months) | $8,000 | Full-stack web build, indexd SDK integration, encryption design and implementation, key-management UX, PWA configuration, testing, documentation |
| User storage subsidy | $2,000 | Seeds the hosted indexd allowance and contract formation costs so early testers onboard with zero friction, ie, crypto layer fully abstracted away |

The development line covers two months of focused work at a rate consistent with a small-grant project. The $2,000 subsidy is intentional: at Sia's median rate of roughly $3/TB/month, and given the small footprint of structured experiment data (mostly text and numerical fields), $2,000 comfortably covers more than one hundred early users for the duration of the grant period and well beyond. Removing the cryptocurrency-onboarding step is essential for reaching the actual target audience, biohackers and self-experimenters, not Web3 natives.

### What is the high-level architecture overview for the grant? What security best practices are you following?

The frontend is a Vite \+ React Progressive Web App, installable on mobile and offline-capable for data entry. Storage runs through indexd, integrated via Sia's official SDK. The grant funds a hosted indexd instance with a shared storage allowance for early users, while client-side AES-GCM encryption (keyed from a user-held BIP-39 seed phrase\*) ensures that even the hosted indexd cannot read user data. There are no app-side accounts; users authenticate by signing with their seed-derived keypair, and no plaintext user health data is ever stored anywhere outside the user's own device. The data model is a versioned JSON schema for experiment objects (protocol metadata, timeline, observations, biometric entries), designed for forward compatibility with the future researcher query layer.

Security practices follow the [Sia Foundation grants-guide development guidelines](https://github.com/SiaFoundation/grants-guide/tree/master): end-to-end encryption with the user as the only key holder, full open-source code for community audit, no third-party trackers or analytics SDKs, strict Content Security Policy, no key custody on our side, and reproducible builds for the PWA. The seed-phrase generation and recovery flow follows mature wallet patterns to reduce footguns for non-crypto users.

### What are the goals of this small grant? Please provide a general timeline for completion

The singular goal is to ship a working, publicly available MVP of the personal experiments tracker within eight weeks.

| Week | Milestone |
| :---- | :---- |
| 1 | Schema design, architecture finalization, indexd SDK integration spike |
| 2 | UI shell, experiment creation flow, key generation and recovery UX |
| 3 | Encrypted upload-and-retrieval round-trip working end-to-end against testnet indexd |
| 4 | Experiment logging interface, basic dashboard, PWA configuration. **Month-1 progress report filed.** |
| 5 | Mainnet integration, user-subsidy mechanism wired up, onboarding flow polished |
| 6 | Stretch goal: single fitness-data import (Garmin Connect or Apple Health export, chosen based on week-5 user feedback) |
| 7 | External testing with 25–50 invited users from self-experimentation communities; bug fixes |
| 8 | Public launch, documentation, retrospective. **Month-2 progress report filed.** |

### Who is the target user for your project?

Self-experimenters from the quantified-self, biohacking, longevity, and peptide-using communities — the audience of /r/Nootropics, /r/Peptides, /r/QuantifiedSelf, Huberman Lab listeners, and people who currently track personal protocols across a fragmented mix of spreadsheets, Notion templates, and platform-specific apps. The common thread is that these users already care about their data, already collect it, and already understand the value of n=1 experimentation; they simply lack a private, portable, ownership-respecting place to keep it.

### What are your plans for this project following the grant?

Beyond this MVP, the roadmap moves in three stages. First, broader health-data ingestion across multiple platforms (Garmin, Whoop, Oura, CGMs, body-composition scales) so the tracker becomes the user's single source of truth. Second, a researcher query and discovery layer that allows scientists to run privacy-preserving aggregate studies against pools of consenting users' data, turning thousands of n=1 experiments into population-scale insight without ever centralizing the underlying data. Third, a Story Protocol integration so users' data points become licensable IP assets they can earn from when used in published research. We anticipate applying for a larger Sia grant to fund the subsequent layers once the storage MVP is proven and adopted.

### Potential risks that will affect the outcome of the project

The largest risk is key-management UX for users unfamiliar with seed phrases. We mitigate this with a well-tested onboarding pattern modeled on mature wallets (MetaMask, Phantom) and an optional encrypted-seed backup to a user-chosen channel (iCloud, Google Drive), explicitly labeled as a convenience trade-off rather than a default. Sia's first-retrieval latency, which is slower than centralized cloud, is mitigated by aggressive local caching of recently-decrypted data and clear onboarding expectations. Cold-start adoption is partly addressed by the storage subsidy that removes financial friction and primarily addressed by launching directly into self-experimentation communities where I am already active. Scope is strictly ringfenced: the fitness-data integration remains a stretch goal rather than a delivery commitment, and the researcher query layer is explicitly out of scope for this grant.

## 

## **Development Information**

### Will all of your project's code be open-source?

Yes. The full codebase will be released under the Apache 2.0 license. No closed-source components are currently planned. The hosted indexd configuration and deployment scripts will also be published so anyone can self-host their own instance.

### Leave a link where code will be accessible for review

[https://github.com/thisispalash/dapp-ideas-sia](https://github.com/thisispalash/dapp-ideas-sia)

### Do you agree to submit monthly progress reports?

Yes. Progress reports will be filed monthly to the Sia forum per program requirements, summarizing milestones completed, blockers, and any scope adjustments.

## 

## Additional context

A note that didn't fit the form template: I've been trying to build this project for a couple of years now, and the missing piece was always the storage primitive. Before discovering Sia, the working plan was a desktop application with an embedded IPFS node, each user's install acting as both a pinning layer and a node in a peer-to-peer network with other users. Workable on paper, painful in practice, and a heavy ask of non-technical biohackers and self-experimenters. Sia uncomplicates the whole thing: native client-side encryption, mutable contracts, and a real economic layer for hosts means I can ship an ordinary web app and still give users genuine ownership of their data. No install. No node. No friction.

I came across Sia at Consensus Miami, talking with Matt at the Sia booth. That conversation is the reason this proposal exists; what had stalled for years suddenly had a clear path forward. Grateful for the work the Foundation is doing, and excited to build on top of it.

## Contact info

### Email

 [thisispalash@kdio.xyz](mailto:thisispalash@kdio.xyz)

### Twitter / X

[@theprimefibber](https://x.com/theprimefibber)
