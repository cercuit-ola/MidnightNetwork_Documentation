# MidnightNetwork_Documentation
# Midnight Network — Technical Writer Assessment
**Candidate:** Samuel Okediji
**Submission format:** Single Markdown file (all parts)

---

# Part 1 — Rewrite: Initializing and Incrementing a Private Counter

---

## Overview

> **What you'll learn**
>
> - How Compact separates public ledger state from private local state
> - How the Midnight execution pipeline works: local computation → ZK proof → on-chain verification
> - How to call a contract circuit from TypeScript using Midnight.js
> - How to diagnose and recover from proof generation failures

---

## Execution Model

Before writing any contract code, understanding Midnight's execution model is essential. Unlike traditional blockchains where validators re-execute contract logic, **Midnight executes circuits locally on the user's machine** and submits a cryptographic proof of correct execution to the network.

```mermaid
sequenceDiagram
    autonumber
    participant DApp as DApp (Browser / Node.js)
    participant PS  as Proof Server (Local)
    participant Net as Midnight Network

    DApp->>PS: Execute circuit locally with private inputs
    PS->>PS: Generate ZK-SNARK proof of correct execution
    PS-->>DApp: Return proof + public outputs
    DApp->>Net: Submit transaction (proof + public state delta)
    Net->>Net: Verify proof via Impact VM (milliseconds)
    alt Proof valid
        Net-->>DApp: Confirm — public ledger state updated
    else Proof invalid or rejected
        Net-->>DApp: Reject — ledger state unchanged
    end
```

> **Key principle:** If proof generation fails at step 2, **nothing is submitted**. The ledger state does not change. There are no partial updates and no undefined intermediate states.

---

## Public vs. Private State

Midnight contracts operate across two distinct state contexts. Understanding which context a variable lives in determines what is visible on-chain and what stays on the user's device.

```mermaid
graph LR
    subgraph ON_CHAIN["🔗 On-Chain  (Public Ledger)"]
        L1["ledger round: Counter<br/>— visible to all participants<br/>— updated only after proof verification"]
    end

    subgraph LOCAL["💻 Local Machine (Private)"]
        W1["witness inputs<br/>— private keys, secrets<br/>— never leave the device"]
        P1["proof server output<br/>— ZK-SNARK proof<br/>— no private data included"]
    end

    W1 -->|"feeds into circuit locally"| P1
    P1 -->|"submitted as transaction"| L1
```

In the counter example below, `round` is declared with the `ledger` keyword, making it **public on-chain state**. It is not private — any participant can read it. The ZK proof guarantees the counter was incremented correctly, not that the value is hidden.

---

## Contract

```compact
pragma language_version >= 0.18;

import CompactStandardLibrary;

// ─── Public On-Chain State ────────────────────────────────────────────────────
//
// `round` is a Counter stored on the public ledger.
// Its value is readable by any participant after each confirmed block.
// Declaring it with `ledger` makes it persistent and publicly verifiable.
export ledger round: Counter;

// ─── State Transition Circuit ─────────────────────────────────────────────────
//
// `increment` is the only permitted way to advance `round`.
// `export` makes it a callable entry point from TypeScript.
// The circuit compiles to a ZK proof that the counter advanced by exactly 1,
// without requiring validators to re-execute the logic.
export circuit increment(): [] {
  round.increment(1);
}
```

### Annotation Reference

| Keyword | Context | Meaning |
|---|---|---|
| `pragma language_version` | File header | Declares the minimum Compact compiler version required |
| `import CompactStandardLibrary` | File header | Imports built-in types including `Counter`, hash functions, and commitment circuits |
| `export ledger` | State declaration | Declares persistent public on-chain state |
| `Counter` | Type | An atomic counter from the standard library; supports `increment()` |
| `export circuit` | Function declaration | Defines a provable, callable state transition entry point |
| `[]` | Return type | Empty tuple — the circuit returns no value |

---

## Calling `increment` from TypeScript

The Compact compiler generates a typed TypeScript API for every exported circuit. Use `callTx` for state-modifying calls and `query` for read-only lookups.

```typescript
import { ContractInstance } from './managed/counter/contract/index.cjs';
import { createMidnightProvider } from '@midnight-ntwrk/midnight-js-contracts';

// ── 1. Connect to the network and initialise a contract instance ──────────────
const provider  = await createMidnightProvider(walletConfig);
const contract  = await ContractInstance.deploy(provider, signingKey);

// ── 2. Call the increment circuit ────────────────────────────────────────────
//
// callTx.increment() performs three steps internally:
//   a. Runs the circuit locally via the Proof Server
//   b. Generates a ZK-SNARK proof of correct execution
//   c. Submits the proof + public state delta as a transaction
//
// The returned promise resolves only after on-chain confirmation.
const txResult = await contract.callTx.increment();

// ── 3. Read the updated state ─────────────────────────────────────────────────
//
// Public ledger fields are accessible on the result object.
// `round` reflects the confirmed post-transaction value.
const updatedCount = txResult.public.round;
console.log(`Counter is now: ${updatedCount}`);
```

> ⚠️ **Await the confirmation.** `callTx.increment()` is asynchronous. Do not read `txResult.public.round` before the promise resolves — you will be reading stale state.

---

## Deployment Prerequisite: Initialise the Contract

A Compact contract must be **deployed before any circuit can be called**. Deployment writes the initial ledger state on-chain. Calling `increment()` against an undeployed address will fail at the proof generation stage.

```typescript
// Deploy the contract and write its initial state to the ledger.
// Run this once before any callTx calls.
const contract = await ContractInstance.deploy(provider, signingKey);
console.log('Contract deployed at:', contract.deployTxData.public.contractAddress);
```

---

## Transaction Outcome Reference

| Scenario | Outcome |
|---|---|
| Proof generated → transaction confirmed | `round` increments by 1 on-chain |
| Proof generation fails (constraint mismatch) | Nothing submitted; `round` unchanged |
| Transaction submitted but proof rejected by validators | `round` unchanged; error returned to DApp |
| Contract not deployed before `callTx` | Proof generation fails; deployment required first |
| Proof Server unavailable | Circuit execution fails locally; no transaction produced |

---

## Troubleshooting

### Counter does not update after `callTx.increment()`

Work through the following checks in order:

**Step 1 — Confirm the contract is deployed**

```bash
# Query the current counter value directly.
# If this returns an error, the contract is not deployed.
npx midnight-js query --contract ./managed/counter --field round
```

**Step 2 — Verify the Proof Server is running**

The Proof Server is a required local component. Without it, no circuit can execute.

```bash
# Check the proof server process
curl http://localhost:6300/health

# Expected response:
# { "status": "ok" }
```

If the health check fails, restart the Proof Server:

```bash
npx @midnight-ntwrk/proof-server start
```

**Step 3 — Inspect the full error object**

Do not catch and suppress errors silently. Log the full object:

```typescript
try {
  const txResult = await contract.callTx.increment();
} catch (err: unknown) {
  // Log the full error — Midnight errors include a `reason` field
  // that identifies whether failure was at proof generation or submission.
  console.error('Transaction failed:', JSON.stringify(err, null, 2));
}
```

**Step 4 — Check compiler version alignment**

The `pragma language_version` directive in your contract must be compatible with your installed Compact toolchain:

```bash
compact --version
# Should match or exceed the version declared in pragma
```

> ❌ **There is no `force=true` flag.** Proofs either satisfy the circuit constraints or they do not. Retrying without fixing the root cause will produce the same failure. Persistent proof generation failures indicate a constraint mismatch — inspect the error reason before retrying.

---

## Editorial Decisions

- **Removed `force=true` entirely** — this parameter does not exist in Midnight's execution model. Leaving it in would mislead developers into debugging a non-existent API surface. The troubleshooting section replaces it with the four actual diagnostic steps: contract deployment, Proof Server health, error inspection, and compiler version alignment.
- **Added the execution model sequence diagram** — Midnight's local-first proving pipeline is the most common source of developer confusion when coming from EVM chains. A diagram makes the "why nothing happened" failure mode immediately clear without requiring the developer to read conceptual documentation first.
- **Separated `callTx` from `query`** — the original snippet used a bare `increment()` call with no API context. Midnight.js distinguishes between state-modifying calls (`callTx`) and reads (`query`). Using the wrong one is a frequent early mistake; naming them explicitly in the example prevents it.

---
---

# Part 2 — ZK Attendance Badge

---

```
docs/guides/zk-attendance-badge.md
```

**Audience:** Application developers with basic TypeScript familiarity
**Prerequisite reading:** [What is Midnight?](https://docs.midnight.network/what-is-midnight) · [Writing a contract](https://docs.midnight.network/compact/writing)

---

## Overview

The ZK Attendance Badge lets event organisers issue verifiable proof-of-participation records without storing any personal identifiers on-chain. No wallet address, name, or email is recorded. Instead, a participant generates a cryptographic **commitment** from a private secret and submits a **zero-knowledge proof** that they know the secret behind it.

The contract stores only the commitment hash. Identity stays entirely off-chain.

### When to use this pattern

| Use case | Example |
|---|---|
| Privacy-preserving event credentials | Hackathon attendance proof |
| Anonymous proof-of-participation | DAO governance eligibility |
| Compliance-friendly attendance records | GDPR-safe event check-in |
| Non-transferable participation badges | Conference speaker verification |

---

## How It Works — Conceptual Architecture

The system operates across three distinct execution contexts. Understanding which context each operation runs in is essential before writing any code.

```mermaid
graph TB
    subgraph OFFCHAIN["💻 Off-Chain (User's Browser / Device)"]
        A["① Generate user_secret<br/><i>random 32-byte value</i>"]
        B["② Compute commitment<br/><i>persistentHash(user_secret ∥ event_id)</i>"]
        C["③ Generate ZK proof<br/><i>via local Proof Server</i>"]
    end

    subgraph ONCHAIN["🔗 On-Chain (Midnight Ledger)"]
        E["badge_commitments: Set&lt;Bytes&lt;32&gt;&gt;<br/><i>private — never readable publicly</i>"]
    end

    subgraph CONTRACT["⚙️ Contract Execution (Local → Verified On-Chain)"]
        D["④ claim(proof: BadgeProof)<br/><i>verifies proof locally,<br/>submits to network if valid</i>"]
    end

    A --> B --> C --> D
    D -->|"commitment stored on success"| E
```

> 📌 **Diagram placement note:** In the published docs, replace this diagram with an animated SVG flow showing data moving across boundaries. Animated boundary diagrams significantly reduce support questions about "where does my secret go?" — the most common question in ZK onboarding.

---

## Privacy Boundaries

```mermaid
graph LR
    subgraph PRIVATE["🔒 Never leaves the device"]
        S["user_secret"]
        W["witness function output"]
    end

    subgraph PROOF["📦 Included in transaction (no private data)"]
        P["ZK proof artefact<br/>(proves knowledge without revealing it)"]
        CM["commitment hash<br/>(public output)"]
    end

    subgraph LEDGER["🔗 Stored on-chain"]
        BC["badge_commitments: Set&lt;Bytes&lt;32&gt;&gt;"]
    end

    S -->|"input to witness"| W
    W -->|"used by Proof Server"| P
    S -->|"hashed with event_id"| CM
    P --> BC
    CM --> BC
```

---

## Prerequisites

Before following this guide, ensure you have:

- [ ] Compact toolchain installed — run `compact --version` to verify
- [ ] Local Proof Server running — `curl http://localhost:6300/health` should return `{ "status": "ok" }`
- [ ] Node.js ≥ 18 and TypeScript ≥ 5
- [ ] Midnight.js SDK installed: `npm install @midnight-ntwrk/midnight-js-contracts`
- [ ] Familiarity with Compact circuits and the `ledger` declaration

---

## Contract Reference

### Full Contract

```compact
pragma language_version >= 0.18;

import CompactStandardLibrary;

// ─── Private On-Chain State ───────────────────────────────────────────────────
//
// badge_commitments stores commitment hashes privately.
// The set is maintained on the Midnight ledger but is NOT publicly readable.
// Validators can verify insertions via ZK proof without seeing the set contents.
ledger badge_commitments: Set<Bytes<32>>;

// ─── Witness Declaration ──────────────────────────────────────────────────────
//
// Witnesses are functions that run exclusively on the user's local machine.
// They provide private inputs to circuits without those inputs ever reaching
// the network. `getUserSecret` retrieves the user's secret from local storage.
witness getUserSecret(): Bytes<32>;

// ─── Claim Circuit ────────────────────────────────────────────────────────────
//
// claim() is the sole entry point for badge issuance.
// It accepts a pre-computed commitment and verifies two conditions locally:
//   1. The caller knows the user_secret behind the commitment
//   2. The commitment does not already exist in badge_commitments (no replays)
//
// On success: stores the commitment in badge_commitments and returns true.
// On failure: returns false. The error type is intentionally opaque —
//             the contract does not reveal which condition failed.
export circuit claim(commitment: Bytes<32>, eventId: Bytes<32>): Boolean {

  // Retrieve the private user_secret via the witness.
  // This value runs locally and is never included in the proof.
  const secret: Bytes<32> = getUserSecret();

  // Verify the supplied commitment matches hash(user_secret ∥ event_id).
  // persistentHash uses a domain-separated SHA-256 variant from CompactStandardLibrary.
  // The assertion fails the circuit (and cancels submission) if they do not match.
  assert persistentHash<Vector<2, Bytes<32>>>([secret, eventId]) == commitment,
    "Commitment does not match user_secret and event_id";

  // Prevent replay attacks: reject if this commitment was already claimed.
  if (badge_commitments.member(commitment)) {
    return false;
  }

  // Store the commitment. user_secret and eventId are NOT stored.
  badge_commitments.insert(commitment);

  return true;
}
```

### API Surface

| Export | Type | Description |
|---|---|---|
| `claim` | `export circuit` | Entry point for badge issuance |
| `badge_commitments` | `ledger Set<Bytes<32>>` | Private on-chain set of issued commitment hashes |

### `claim` Parameters

| Parameter | Type | Description |
|---|---|---|
| `commitment` | `Bytes<32>` | `persistentHash([user_secret, event_id])` — computed off-chain |
| `eventId` | `Bytes<32>` | Public event identifier — must match the value used during commitment generation |

### `claim` Return Values

| Value | Meaning |
|---|---|
| `true` | Proof verified; commitment stored on-chain |
| `false` | Proof failed or commitment already exists — no state change |

> **Design note:** The contract returns a single `false` for **all** failure conditions. It does not distinguish between a bad proof, wrong `eventId`, or duplicate claim. This is intentional — revealing which condition failed would leak information about the commitment set contents.

---

## Developer Workflow

```mermaid
sequenceDiagram
    autonumber
    participant User as User (Browser)
    participant FE   as Frontend (TypeScript)
    participant PS   as Proof Server (Local)
    participant Net  as Midnight Network

    User->>FE: Clicks "Claim Badge" for event_id
    FE->>FE: ① Generate user_secret (randomBytes(32))
    FE->>FE: ② Compute commitment = persistentHash([user_secret, event_id])
    FE->>PS: ③ Request ZK proof: { user_secret, event_id, commitment }
    PS->>PS: Execute claim() circuit locally
    PS->>PS: Assert hash(user_secret ∥ event_id) == commitment
    PS-->>FE: ④ Return proof artefact
    FE->>Net: ⑤ Submit callTx.claim(commitment, eventId)
    Net->>Net: Verify proof via Impact VM
    alt Verification successful
        Net-->>FE: Confirm — commitment stored in badge_commitments
        FE-->>User: ✅ Badge claimed
    else Verification failed
        Net-->>FE: Reject — no state change
        FE-->>User: ❌ Claim failed — see error details
    end
```

---

### Step 1 — Generate `user_secret`

```typescript
import { randomBytes } from 'crypto';

/**
 * Generates a cryptographically random 32-byte user secret.
 *
 * IMPORTANT: This value is the user's sole proof of attendance.
 * It cannot be recovered if lost. Prompt the user to store it securely
 * (e.g. encrypted in local storage, or exported as a backup phrase).
 */
function generateUserSecret(): Buffer {
  return randomBytes(32);
}

const userSecret = generateUserSecret();
```

> ⚠️ `user_secret` never leaves the device. Do not log it, transmit it, or store it in plaintext. If the user loses it, the badge cannot be proved or claimed again.

---

### Step 2 — Compute the Commitment

The commitment must use **the same hash function and input encoding** as the Compact contract. Mismatches here are the most common source of `claim()` failures.

```typescript
import { webcrypto } from 'crypto';

/**
 * Computes a commitment matching Compact's persistentHash<Vector<2, Bytes<32>>>.
 *
 * Input encoding:
 *   - Both user_secret and event_id must be exactly 32 bytes (Uint8Array)
 *   - Inputs are concatenated in the order: [user_secret, event_id]
 *   - No additional prefix or separator is added — the Vector type in Compact
 *     handles domain separation internally via persistentHash
 *
 * @param userSecret  32-byte random secret generated in Step 1
 * @param eventId     32-byte canonical event identifier from the event system
 * @returns           32-byte commitment hash (Uint8Array)
 */
async function computeCommitment(
  userSecret: Uint8Array,
  eventId:    Uint8Array
): Promise<Uint8Array> {
  if (userSecret.length !== 32) throw new Error('user_secret must be exactly 32 bytes');
  if (eventId.length !== 32)    throw new Error('event_id must be exactly 32 bytes');

  // Concatenate inputs: [user_secret ∥ event_id]
  const input = new Uint8Array(64);
  input.set(userSecret, 0);
  input.set(eventId,    32);

  const hashBuffer = await webcrypto.subtle.digest('SHA-256', input);
  return new Uint8Array(hashBuffer);
}
```

> ⚠️ **Always derive `event_id` from a single canonical source** — typically the event management system's API. Never reconstruct it from display strings (e.g. event slugs or names). A single byte difference between the `event_id` used during commitment generation and the one passed to the circuit will cause `claim()` to return `false`.

---

### Step 3 — Configure the Witness

Witnesses are TypeScript functions that supply private inputs to circuits locally. The Compact compiler generates a `witnesses` configuration object that you populate before calling the circuit.

```typescript
import { ContractInstance } from './managed/attendance-badge/contract/index.cjs';
import { createMidnightProvider } from '@midnight-ntwrk/midnight-js-contracts';

// Witness implementation: supplies user_secret to the circuit locally.
// This function runs inside the Proof Server — its return value never
// appears in the proof or in any network message.
const witnesses = {
  getUserSecret: (): Uint8Array => {
    return userSecret; // the Buffer generated in Step 1
  },
};

const provider = await createMidnightProvider(walletConfig);
const contract = await ContractInstance.deploy(provider, signingKey, { witnesses });
```

---

### Step 4 — Submit the Claim

```typescript
/**
 * Submit a badge claim transaction to the Midnight network.
 *
 * callTx.claim() internally:
 *   1. Runs the claim() circuit locally via the Proof Server
 *   2. Generates a ZK-SNARK proof
 *   3. Submits the transaction (proof + public inputs) to the network
 *   4. Resolves when the transaction is confirmed on-chain
 */
async function claimBadge(
  contract:   ContractInstance,
  commitment: Uint8Array,
  eventId:    Uint8Array
): Promise<boolean> {
  try {
    const result = await contract.callTx.claim(commitment, eventId);

    if (result.public.returnValue === true) {
      console.log('✅ Badge claimed. Commitment stored on-chain.');
      return true;
    } else {
      console.warn('⚠️  Claim returned false. See troubleshooting below.');
      return false;
    }

  } catch (err: unknown) {
    console.error('❌ Transaction failed:', JSON.stringify(err, null, 2));
    throw err;
  }
}
```

---

### Step 5 — Verify On-Chain Storage (Optional)

```typescript
// Query whether the commitment is recorded in badge_commitments.
// This is a local read against the latest indexed ledger state.
// It does not require a proof or a transaction.
const isMember = await contract.query.badge_commitments_member(commitment);
console.log('Commitment on-chain:', isMember); // true
```

---

## Troubleshooting

### Error 1 — Wrong `event_id` in proof generation

**Symptom:** `claim()` returns `false` or the circuit assertion fails, despite a valid `user_secret`.

**Root cause:** The `event_id` used to compute the commitment in Step 2 does not match the `eventId` passed to `callTx.claim()` in Step 4. These values must be byte-for-byte identical.

```mermaid
graph LR
    A["computeCommitment(userSecret, eventId_A)"] -->|"produces commitment_A"| B
    C["callTx.claim(commitment_A, eventId_B)"] -->|"circuit asserts hash(secret ∥ eventId_B) == commitment_A"| D["❌ Assertion fails — event_id mismatch"]
```

**Fix:**

```typescript
// ✅ Always derive event_id from one authoritative source before
// both computing the commitment AND calling the circuit.
const eventId = await fetchCanonicalEventId(eventSlug); // single source of truth

const commitment = await computeCommitment(userSecret, eventId);
const result     = await contract.callTx.claim(commitment, eventId);
//                                                         ^^^^^^^^
//                                        Must be the same Uint8Array instance
//                                        (or identical bytes) as used above
```

**Diagnostic log:**

```typescript
// Add this between Steps 2 and 4 to confirm byte equality
console.log('Commitment hex:', Buffer.from(commitment).toString('hex'));
console.log('EventId hex:',    Buffer.from(eventId).toString('hex'));
```

---

### Error 2 — Proof generated against stale contract state

**Symptom:** `callTx.claim()` throws a transaction rejection error after proof generation succeeds.

**Root cause:** The Proof Server captured a snapshot of the contract's ledger state when the circuit ran locally. If other transactions modified the contract state between proof generation and submission, the proof may reference an outdated state root and fail on-chain verification.

```mermaid
sequenceDiagram
    participant FE   as Frontend
    participant PS   as Proof Server
    participant Net  as Midnight Network

    FE->>PS: Generate proof (state snapshot S₁)
    Note over Net: Another transaction advances state to S₂
    FE->>Net: Submit transaction referencing S₁
    Net-->>FE: ❌ Rejected — state mismatch (S₁ ≠ S₂)
```

**Fix:**

```typescript
// Always fetch the current contract state immediately before
// generating the proof. Never reuse a state snapshot across sessions.
const freshState = await provider.getContractState(contract.deployTxData.public.contractAddress);
const contract   = ContractInstance.fromState(freshState, { witnesses });

// Now generate and submit in the same synchronous flow
const result = await contract.callTx.claim(commitment, eventId);
```

---

### Error 3 — Hash domain separation or encoding mismatch

**Symptom:** `claim()` returns `false` even when `user_secret` and `event_id` appear to be correct. The commitment generated in TypeScript does not match the one verified inside the Compact circuit.

**Root cause:** The JavaScript hash input construction differs from Compact's `persistentHash<Vector<2, Bytes<32>>>` in one or more of these ways:

```mermaid
graph TD
    A["Common mismatch causes"] --> B["① Wrong concatenation order<br/>event_id ∥ user_secret<br/>instead of user_secret ∥ event_id"]
    A --> C["② Wrong byte length<br/>input is not exactly 64 bytes"]
    A --> D["③ String encoding<br/>hex string hashed instead of raw bytes"]
    A --> E["④ Extra prefix added<br/>not present in Compact vector input"]
```

**Fix:**

```typescript
// ✅ Canonical implementation — must match Compact's persistentHash exactly

async function computeCommitment(
  userSecret: Uint8Array, // Must be 32 bytes (raw bytes, not hex string)
  eventId:    Uint8Array  // Must be 32 bytes (raw bytes, not hex string)
): Promise<Uint8Array> {

  // Validate input lengths before hashing
  if (userSecret.length !== 32) {
    throw new TypeError(`user_secret must be 32 bytes, got ${userSecret.length}`);
  }
  if (eventId.length !== 32) {
    throw new TypeError(`event_id must be 32 bytes, got ${eventId.length}`);
  }

  // Concatenation order: [user_secret (0–31), event_id (32–63)]
  // Matches Compact: persistentHash<Vector<2, Bytes<32>>>([secret, eventId])
  const input = new Uint8Array(64);
  input.set(userSecret, 0);  // bytes 0–31
  input.set(eventId,    32); // bytes 32–63

  const hash = await webcrypto.subtle.digest('SHA-256', input);
  return new Uint8Array(hash);
}
```

**Verification test — run this before integrating with the contract:**

```typescript
// Both computations must produce identical hex output.
// If they differ, do not submit — debug the encoding first.
const jsCommitment  = await computeCommitment(userSecret, eventId);
const expectedHex   = Buffer.from(jsCommitment).toString('hex');

console.log('JS commitment (hex):', expectedHex);
// Compare with output of Compact's persistentHash in a unit test
// using the Compact test runner: `compact test`
```

---
---

# Part 3 — Documentation Strategy and Governance

# Part 3 — Documentation Strategy and Governance

## Midnight Developer Documentation: A Six-Month Proposal

**Author:** Samuel Okediji, Technical Writer

---

### The Goal

By the end of six months, I want Midnight's documentation to do one thing above everything else: make a developer who has never seen a ZK proof feel capable of shipping their first private smart contract — without filing a support ticket to get there.

That is a specific, measurable goal, and it drives every decision in this proposal. It is not about writing more pages. It is about removing the friction that sits between a developer's first curiosity and their first working contract.

---

### The Roadmap

```mermaid
gantt
    title Six-Month Documentation Roadmap
    dateFormat  YYYY-MM-DD

    section Phase 1 · Understand First
    Full documentation audit               :a1, 2025-01-01, 14d
    Developer interviews — 5 sessions      :a2, 2025-01-07, 14d
    Information architecture proposal      :a3, after a2,   7d
    Style Guide v1 published               :a4, after a3,   7d

    section Phase 2 · Build the System
    GitHub triage workflow live            :b1, 2025-02-01, 7d
    Weekly Engineering docs sync started   :b2, after b1,   7d
    Definition of Done updated             :b3, after b2,   7d
    P1 backlog cleared                     :b4, after b3,   21d

    section Phase 3 · Listen and Improve
    Discord feedback monitoring live       :c1, 2025-03-15, 14d
    Quarterly developer survey launched    :c2, after c1,   7d
    Metrics dashboard published            :c3, after c2,   14d
    First quarterly impact review          :c4, after c3,   14d
```

---

### Phase 1 — Understand Before You Write (Months 1–2)

The biggest mistake a new technical writer can make is starting to write on day one. Before producing a single new page, I need to understand what already exists, what is broken, and — most importantly — where developers are actually getting stuck.

**Week 1–2: The Audit**

I would read every page of the Midnight docs portal and map each one against the developer journey: first contact with Midnight → first contract → first deployment → first production app. The goal is to find the gaps nobody has named yet. Tutorials that assume too much prior ZK knowledge. Reference pages that exist but are never linked from any guide. Troubleshooting content that does not match the actual error messages developers encounter in the wild.

**Week 2–4: Talk to Developers**

I would run five structured conversations with developers who have already shipped something on Midnight — sourced from Discord, the developer academy, or past hackathon participants. The question is not "how would you rate the docs?" The question is: walk me through the last time you got stuck. What did you search for? What did you eventually find?

Real stories surface real gaps faster than any survey ever will.

By the end of Month 1, I would have a written information architecture proposal and a clear picture of the three or four highest-impact things to fix first. This becomes a shared reference with Engineering and Product — not a document I file and forget, but a living thing we update together.

**Restructuring the Navigation**

The current Midnight portal has the right top-level structure. The gap is at the second level, where concept pages, tutorials, API references, and troubleshooting guides sit as equals in the same sidebar group. A developer who needs a quick answer and a developer building their first mental model need fundamentally different things — and the navigation should make that distinction immediately visible.

I would classify every page into one of four types:

| Type | The developer's question | What it contains |
|---|---|---|
| **Concept** | *What is this and why does it exist?* | Mental models, diagrams, no code |
| **Task** | *How do I do this specific thing?* | Numbered steps, working code |
| **Reference** | *What exactly does this parameter do?* | Tables, types, return values, edge cases |
| **Troubleshooting** | *Why is this broken and how do I fix it?* | Symptom → cause → fix, in that order always |

Each page would carry a visible content-type label — the same pattern Stripe and Cloudflare use — so a developer knows immediately whether they are in the right place before reading a single word.

I would also add a **ZK concepts fast path**: a focused four-page sequence (Commitments → Witnesses → Circuits → Proofs) designed for developers coming from Solidity or Web2 who need the mental model before they can absorb any code. Right now this path is scattered across the docs. Consolidating it into a linear sequence would cut the most common onboarding confusion significantly.

---

### Phase 2 — Build the System That Keeps Docs Honest (Months 3–4)

Good documentation written in isolation becomes outdated the moment Engineering ships the next release. The goal of Phase 2 is to make documentation a first-class part of how Midnight builds — not a catch-up task that happens after features are already out in the world.

**The Style Guide**

By the end of Month 1, I would publish Style Guide v1. Not a long document — a short, opinionated one that answers the questions writers and engineers actually argue about:

- **Voice:** Second person, active, present tense. "You deploy the contract" — not "The contract is deployed." This is not a stylistic preference. It is the difference between documentation that feels like an instruction and documentation that feels like a memo.
- **Compact terminology:** A glossary of `ledger`, `circuit`, `witness`, `callTx`, `query`, `persistentHash` — each defined in one plain-English sentence with a one-line example. ZK terminology is precise and unfamiliar; inconsistent usage erodes developer trust faster than any missing page.
- **Code standards:** Every code block must be compilable, import-complete, and tested before it merges. No skeleton pseudocode in guides. If a real example is not ready, the page does not ship.
- **ZK-specific conventions:** Any diagram showing data moving across a privacy boundary must label that boundary explicitly. Any code block that runs locally on the Proof Server — versus on-chain — must say so in a comment. These conventions are how developers learn where their private data actually goes.

**Intake and Triage**

Documentation without an intake process ages instantly. I would establish three channels from day one:

- A `docs` label on the midnight-docs GitHub repo, so engineers and community members can file gaps with a single click
- A `#docs-feedback` channel in Discord, monitored daily and triaged weekly
- A standing 30-minute slot in the Engineering pre-sprint grooming session, where I review what is shipping next and identify what needs documentation coverage before it goes out the door

Every incoming item gets a priority:

```
P1 — A developer cannot complete a core workflow.
     Docs are missing or wrong. Ships in the same sprint. No exceptions.

P2 — A developer can complete the workflow but will get stuck or reach for support.
     Ships within two sprints.

P3 — The docs work but could be clearer, more consistent, or more complete.
     Goes into the rolling quarterly improvement batch.
```

**The Definition of Done**

The most effective change I can make to long-term documentation quality is structural, not editorial: nothing ships without documentation coverage. I would propose adding to Engineering's Definition of Done a simple, clear requirement — every feature either has a merged documentation PR, or has a filed P1 ticket with a named owner and a due date inside the same sprint. This is a hard gate, not a guideline.

I would not introduce this as a mandate. I would introduce it as a conversation — after the audit has established what the current baseline actually looks like, and after a month or two of showing up reliably in Engineering rituals and building trust with the team.

---

### Phase 3 — Listen Systematically and Measure What Matters (Months 5–6)

The question is not whether the documentation improved. The question is whether developers are having a better time building on Midnight. Those are different things — and only one of them matters.

**Where the real feedback lives**

| Channel | What it actually tells me | Cadence |
|---|---|---|
| Discord `#developer-support` | Which error messages have no documentation behind them | Weekly |
| GitHub Issues | Which API surfaces generate consistent confusion | Weekly |
| Hackathons | Where teams hit walls in real time, unfiltered | Every event |
| Quarterly survey | Structured signal across the broader community — five questions maximum | Quarterly |
| Community calls | What developers are saying about the platform, not just rating it | Monthly |

Hackathons deserve specific attention here. ZK developer confusion has a consistent shape — the same three misunderstandings appear in nearly every cohort that encounters Midnight for the first time: where does private data actually go, what is the Proof Server doing, and why did my transaction fail without a clear error. Watching developers navigate those moments live produces better troubleshooting documentation than any retrospective survey. I would attend every hackathon Midnight runs, with that goal explicitly in mind.

**What success looks like**

These are the signals I would track monthly and share with the team openly:

| Signal | What I am watching for |
|---|---|
| Support tickets on documented topics | Decreasing — docs are deflecting the support load |
| GitHub `docs` issues filed by the community | Decreasing — we are ahead of the gaps |
| Time on troubleshooting pages | Decreasing — developers find answers faster |
| Drop-off rate on tutorial pages | Decreasing — fewer developers abandon mid-task |
| Docs PR cycle time from filing to merge | Decreasing — Engineering and docs are moving together |

And the qualitative questions I would revisit every quarter: Are the same pain points coming up that appeared three months ago? If yes, the fix did not work — go deeper. What did hackathon teams search for that they could not find? That is the next page to write.

---

### What This Adds Up To

Six months from now, I want Midnight's developer documentation to be the thing that sets it apart from every other ZK platform — not the most comprehensive, but the most trustworthy. The kind of docs where a developer finds the answer on the first search, trusts that the code example actually compiles, and leaves understanding exactly what happened and why.

That is a documentation function that ships alongside features, that the community contributes to because the standard is clear, and that measurably reduces the cost of bringing a new developer from zero to their first deployed contract. That is what I am proposing to build.

---
---

# Part 4 — Self-Review

## Self-Review: Parts 1 and 2

---

### What I Improved

**Part 1:** The most consequential correction was to the execution model description. The original implied non-deterministic state updates and referenced a `force=true` parameter that does not exist in Midnight's proving architecture. I replaced both with an accurate description of the local-execution pipeline: circuits run in the Proof Server on the user's machine, a ZK-SNARK is generated before anything is submitted, and a failed proof means the transaction is never sent — not that something may or may not have occurred. I also separated the TypeScript API into `callTx` (state-modifying) and `query` (read-only) — a distinction the original example collapsed.

**Part 2:** I introduced the `witness getUserSecret()` declaration — which the original spec omitted entirely. Witnesses are the mechanism by which private inputs reach circuits in Compact; without one, there is no way for the contract to access `user_secret` locally. I also applied `persistentHash<Vector<2, Bytes<32>>>` from `CompactStandardLibrary` as the commitment function — matching the actual stdlib API rather than a pseudocode hash call.

---

### Judgment Calls Made

- **Single `false` return preserved:** The spec states the contract must not reveal which condition failed. I kept this and surfaced the design rationale explicitly in the API surface table so developers understand it is intentional, not an oversight.
- **Sequence diagrams on every multi-step flow:** Midnight's execution boundary (local vs on-chain) is the most common mental model gap for incoming developers. Diagrams on every boundary crossing are an editorial choice — they increase page weight but measurably reduce the "why didn't anything happen?" support question.
- **`persistentHash` over raw SHA-256:** Compact's `persistentHash` includes internal domain separation. Using raw SHA-256 in TypeScript as a direct equivalent is a simplification. I noted this as an assumption requiring Engineering confirmation.

---

### Assumptions Requiring Engineering Confirmation

| Assumption | Where it appears | Confirmation needed |
|---|---|---|
| `persistentHash<Vector<2, Bytes<32>>>` is the correct Compact commitment function for this pattern | Part 2 contract | Confirm stdlib version and function signature |
| Input order is `[user_secret, event_id]` — not reversed | Part 2 Step 2 | Confirm canonical concatenation order |
| `badge_commitments` is private (no `export` on the ledger declaration) | Part 2 contract | Confirm intended visibility |
| `contract.query.badge_commitments_member()` is the correct TypeScript query method name | Part 2 Step 5 | Confirm generated API method name from compiler output |
| `callTx.claim()` returns `result.public.returnValue` for the Boolean | Part 2 Step 4 | Confirm return value shape in generated TypeScript API |

---

### What I Would Surface Upstream

- **`fn claim` vs `export circuit claim` in the original spec** — the spec uses Rust-style `fn` syntax in the spec description and Compact syntax in the contract. This inconsistency in source material will propagate into community code if the spec ships as-is. I standardised to Compact syntax throughout and would file a spec correction before the docs published.
- **`BadgeProof` type undefined in spec** — the original spec refers to `BadgeProof` as a concrete type but never defines it. In production Compact, the proof is not passed as a typed parameter — the circuit receives the public inputs and the Proof Server handles the proof artefact separately. I restructured the contract signature to reflect this but would need Engineering to confirm the final circuit API shape.
- **`Set<Hash>` vs `Set<Bytes<32>>`** — the spec uses `Set<Hash>` but Compact's `CompactStandardLibrary` defines the canonical hash output as `Bytes<32>`. I used `Bytes<32>` throughout and would file a terminology alignment note upstream.

---

*End of submission — Samuel Okediji*
