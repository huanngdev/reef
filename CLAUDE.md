# Reef

**Decentralized bug bounty marketplace for Sui Move contracts**

## Overview

Reef is a trust-minimized marketplace where security researchers and Sui Move package maintainers trade vulnerability disclosures. Two flows are supported: a Researcher discovers a bug and lists it for sale, or a Maintainer creates a bounty to invite submissions. All sensitive bug details are encrypted with Seal and stored on Walrus; payouts are governed by on-chain escrow logic with timeout protection and an Arbiter panel for disputes.

The platform exists because traditional bug bounty services (Immunefi, HackerOne) act as centralized trust anchors. Reef replaces that trust with verifiable on-chain logic, cryptographic gating of bug details, and an economically-incentivized dispute mechanism.

The MVP focuses on Sui Move smart contract bugs only. Frontend and SDK bug categories are deferred to post-hackathon roadmap.

All protocol parameters are hardcoded as Move constants in the smart contract. There is no administrative role, no governance mechanism, no fee adjustment, and no emergency pause. Future iterations of the protocol require deploying new contracts; users migrate at their own discretion. This trade-off maximizes trust minimization at the cost of operational flexibility — once deployed, the rules cannot change.

## Actors

### Researcher

Security researchers who discover bugs in deployed Move packages. Can list a bug for sale, or submit to an active Bounty. Requires only a Sui wallet to participate. Posts a small anti-spam stake when submitting to Bounties, refunded if the submission is not flagged.

### Maintainer

Owner of a Move package who wants to secure their code. Operates in two modes:

- **Reactive** — funds an existing Finding posted by a Researcher
- **Proactive** — creates a Bounty with locked prize pool, sets scoring rules and deadlines

In MVP, no challenge deposit is required when disputing. Post-MVP, the Maintainer stakes a percentage of escrow when challenging a Finding, slashed if the Arbiter panel rules against them.

### Arbiter

Independent reviewer who stakes SUI to join the dispute resolution pool. Randomly selected from the pool when a Dispute is opened, weighted by stake and reputation. Earns rewards for voting with majority verdict; slashed for minority votes or failure to reveal. Reputation is tracked on-chain and influences future selection probability.

### Observer

Anyone browsing the platform without participating. Sees public listings with teasers only. Can read fully-disclosed Findings after the disclosure period. No wallet required unless transitioning to an active role.

## Core Functions

### Researcher actions

- **List finding** — create a Finding with public teaser and reference to encrypted blob on Walrus
- **Withdraw listing** — remove a Finding that has not yet been funded
- **Claim timeout payout** — claim escrow funds when the Maintainer fails to respond within the confirm window
- **Submit to bounty** — post a Submission to an open Bounty, with anti-spam stake
- **Claim bounty winnings** — withdraw prize after the Maintainer judges submissions

### Maintainer actions

- **Fund finding** — escrow SUI against a Researcher's Finding
- **Confirm valid** — release escrow to the Researcher upon validation (happy path)
- **Challenge finding** — open a Dispute against a funded Finding
- **Create bounty** — lock a prize pool, define target package, scoring config, and deadlines
- **Judge submissions** — rank winning Submissions according to the committed scoring config
- **Cancel bounty** — cancel a Bounty before its submission window opens

### Arbiter actions

- **Stake to join panel** — deposit SUI and create an Arbiter profile to become eligible for selection
- **Commit vote** — submit a hashed verdict during the commit phase of a Dispute
- **Reveal vote** — disclose the verdict and salt during the reveal phase
- **Claim arbiter reward** — collect reward and reputation update after Dispute resolution
- **Withdraw stake** — exit the panel after a cooldown period

## MVP Scope

### In MVP — required for end-to-end demo

- Flow A happy path: list → fund → confirm → paid
- Flow A timeout path: list → fund → (silence) → Researcher auto-claim
- Flow B happy path: create Bounty → submissions → judge → distribute prizes
- Seal encryption gating based on escrow state for Flow A, and based on deadline for Flow B
- Walrus blob upload and retrieval
- Listing browse and detail UI
- Wallet connection via Sui Wallet Kit
- Platform fee collection on successful payouts

### Out of MVP — mocked or skipped

- Dispute mechanism entirely — in MVP, Maintainer can only confirm or let timeout expire; challenging is not exposed
- Arbiter role, panel selection, commit-reveal voting, and reputation system
- Disclosure period enforcement via post-disclosure Seal policy
- Bounty timeout auto-distribution — Maintainer must judge for prize release
- Severity validation — trust the Researcher's claim; Maintainer evaluates manually

### Roadmap post-hackathon

- Real Arbiter panel with full commit-reveal flow and reputation tracking
- Nautilus TEE integration for automated PoC verification
- Disclosure timeline enforcement on-chain
- Multi-category support (frontend bugs, SDK bugs)
- Insurance pool for false-positive protection

## Data Flow

### Flow A — Researcher-initiated finding

1. Researcher writes the vulnerability writeup and proof-of-concept locally.
2. Researcher encrypts the artifact bundle with Seal client-side.
3. Researcher uploads the encrypted blob to Walrus and receives a blob identifier.
4. Researcher submits an on-chain transaction to create a Finding with status `Listed`.
5. Maintainer browses the marketplace and reviews the public teaser.
6. Maintainer funds the Finding, creating an Escrow object with status `Funded`; a funding event is emitted.
7. Maintainer queries Seal key servers; the funding state proves access rights and the decryption key is released.
8. Maintainer fetches the encrypted blob from Walrus and decrypts locally to read the bug details.

From here, three outcomes are possible:

- **Confirm path** — Maintainer confirms the bug is valid. Escrow releases to the Researcher; status becomes `Paid`.
- **Challenge path** — Maintainer challenges the Finding. A Dispute is created. Arbiters are selected, decrypt the bug detail via dispute-scoped Seal policy, vote through commit-reveal, and majority verdict determines whether funds release to Researcher (`Paid`) or refund to Maintainer (`Refunded`).
- **Timeout path** — Maintainer remains silent past the confirm window. Researcher calls the timeout-claim function; escrow releases to Researcher; status becomes `Paid`.

### Flow B — Maintainer-initiated bounty

1. Maintainer submits a transaction to create the Bounty, locking the prize pool and setting target package, scoring config, submission window, and judging window.
2. One or more Researchers prepare their reports, encrypt with Seal, and upload to Walrus.
3. Each Researcher submits to the Bounty with an anti-spam stake; a Submission is created and attached to the Bounty.
4. When the submission window closes, the Bounty auto-transitions to `Judging` status.
5. The Seal policy for Submissions now permits Maintainer decryption based on time-elapsed condition.
6. Maintainer reads all Submissions and judges, ranking winners per the committed scoring config. The prize pool distributes accordingly; valid (non-spam) stakes refund.

### Encryption and storage flow

A Researcher's bug artifact begins as plaintext on their machine. Seal encrypts the bundle with a policy that binds decryption rights to specific on-chain conditions:

- For Flow A — decryption is permitted when the escrow is funded and the requester is the buyer
- For Flow B — decryption is permitted after the submission window closes for the Bounty's Maintainer
- For Disputes — decryption is permitted for Arbiters listed in the active Dispute object

The encrypted blob is uploaded to Walrus, which returns a blob identifier. This identifier is stored on-chain in the Finding or Submission object alongside the Seal policy identifier.

When a Maintainer or Arbiter is entitled to view the bug, they query Seal key servers with proof of their role. Key servers verify the on-chain state via Move policy evaluation and return decryption keys. The recipient then fetches the encrypted blob from Walrus and decrypts locally.

Once decryption occurs, the platform cannot prevent the decrypted content from being copied. The protection model relies on gating decryption access combined with reputation and slashing for bad-faith Maintainers who decrypt then falsely claim invalid.

## Configuration Parameters

### Time windows

| Parameter                | Default  | Set by                                    |
| ------------------------ | -------- | ----------------------------------------- |
| Confirm window           | 7 days   | Researcher when listing (range 3–14 days) |
| Disclosure period        | 90 days  | Hardcoded                                 |
| Bounty submission window | 14 days  | Maintainer when creating Bounty           |
| Bounty judging window    | 7 days   | Maintainer when creating Bounty           |
| Dispute commit phase     | 48 hours | Hardcoded                                 |
| Dispute reveal phase     | 24 hours | Hardcoded                                 |
| Arbiter stake cooldown   | 14 days  | Hardcoded                                 |

### Economic parameters

| Parameter                    | Default      | Notes                                                    |
| ---------------------------- | ------------ | -------------------------------------------------------- |
| Platform fee                 | 2.5%         | Applied to successful payouts only, not refunds          |
| Minimum asking price         | 10 SUI       | Anti-spam floor for listings                             |
| Minimum bounty pool          | 100 SUI      | Ensures Bounties are worth participating in              |
| Anti-spam submission stake   | 5 SUI        | Refunded if the Submission is not flagged as spam        |
| Arbiter minimum stake        | 100 SUI      | Enough commitment to incentivize honest voting           |
| Dispute opening fee          | 5 SUI        | Paid by the disputer, contributes to Arbiter reward pool |
| Minority Arbiter slash       | 20% of stake | Redistributed to majority voters                         |
| Maintainer challenge deposit | 5% of escrow | Post-MVP; not enforced in MVP                            |

### Panel parameters

| Parameter           | Value                                | Reasoning                                                                  |
| ------------------- | ------------------------------------ | -------------------------------------------------------------------------- |
| Panel size          | 5 Arbiters                           | Odd number prevents ties; small enough for cheap on-chain random selection |
| Quorum              | 3 of 5 reveals                       | Below quorum escalates and slashes non-revealers                           |
| Majority threshold  | 3 of 5 same verdict                  | Standard simple majority                                                   |
| Selection weighting | sqrt(stake) × (1 + reputation / 100) | Anti-whale bias plus reputation reward                                     |

### Reputation deltas

| Event                          | Delta | Notes                              |
| ------------------------------ | ----- | ---------------------------------- |
| Vote with majority             | +5    | Standard correct-vote reward       |
| Vote with minority             | −10   | Penalty for incorrect call         |
| Failure to reveal after commit | −20   | Strong penalty to prevent griefing |
| Starting reputation            | 0     | New Arbiters begin neutral         |

### Severity tiers (advisory)

These ranges guide Researchers in pricing their Findings. Not enforced on-chain.

| Severity | Suggested price range | Typical scenario                         |
| -------- | --------------------- | ---------------------------------------- |
| Critical | 500+ SUI              | Drain user funds; full contract takeover |
| High     | 100–500 SUI           | Privilege escalation; denial of service  |
| Medium   | 25–100 SUI            | Logic bug with exploit path              |
| Low      | 10–25 SUI             | Best practice; gas optimization          |

### Bounty scoring configuration

The Maintainer provides an array of `(rank, percentage_basis_points)` tuples when creating a Bounty. Example configurations:

- **Three winners, weighted split** — 50% / 30% / 20% of the pool
- **Five winners, flat split** — each receives 20%
- **Winner takes all** — single winner receives 100%

The sum of all percentages must equal 10,000 basis points (100%). This is validated on-chain at Bounty creation.
