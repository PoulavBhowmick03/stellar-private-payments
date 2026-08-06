# Stellar Private Payments: a source-grounded technical account

This document describes commit `0d20ca3` as inspected on 2026-08-06. It is deliberately not a generic explanation of privacy pools. Every protocol or implementation claim points to the current repository. “Inference” means the repository supplies evidence but does not state the conclusion. “Not verified” means the repository does not contain enough evidence.

The safety label is non-negotiable: the project calls itself a **work in progress**, **unaudited**, a **reference implementation**, and says not to use it in production or with real assets; it also warns that breaking local storage can make funds inaccessible (`README.md:13-14`, `README.md:119-124`). `SECURITY.md` supplies the private vulnerability-reporting route and warns against public disclosure; it does not replace an audit (`SECURITY.md:6-22`).

## Start here: the high-level explainer

### What this project is about

Stellar Private Payments explores how to make token payments through Stellar while hiding the internal ownership and movement of funds inside a shared pool. It is a client-heavy system: users generate zero-knowledge proofs locally, while Soroban contracts hold the public token balance, maintain a public commitment tree, reject reused nullifiers, enforce the selected ASP policy, and verify proofs (`docs/src/introduction.md:13-27`, `contracts/pool/src/pool.rs:513-645`).

The simplest mental model is a shared safe containing public Stellar tokens. Instead of the contract recording “Alice owns 10” in a public account balance, a wallet holds private **notes**. The chain records only commitments to those notes in a Merkle tree. A note’s commitment is computed from its amount, its owner’s note public key, and a random blinding value (`sdk/prover/src/crypto.rs:97-108`).

The project is not a new asset or a separate blockchain. It is a stack of Circom circuits, Soroban contracts, Rust SDK crates, a CLI, and a browser application that coordinate around existing Stellar tokens (`Cargo.toml:1-22`, `docs/src/introduction.md:75-82`, `app/ARCHITECTURE.md:7-23`).

### What it lets a user do

At the product level, it supports three core actions:

1. **Deposit, or shield:** public tokens move from a Stellar account into the pool, and the transaction creates two commitment slots representing private outputs; unused slots are zero-value padding (`contracts/pool/src/pool.rs:513-536`, `sdk/prover/src/flows.rs:552-593`).
2. **Transfer privately inside the pool:** existing private notes are spent and replaced by new notes encrypted for the recipient. The chain sees the transaction, its timing, two nullifiers, and two new commitments, but the circuit keeps note amounts, keys, blindings, and the input-to-output relationship out of public inputs (`circuits/src/transaction.circom:52-140`, `contracts/pool/src/pool.rs:630-643`).
3. **Withdraw, or unshield:** private notes are spent and a public amount is transferred from the pool to a public Stellar recipient. The withdrawal amount and recipient are public because they are part of `ExtData` (`sdk/prover/src/flows.rs:189-227`, `docs/src/privacy-tradeoffs.md:7-12`).

The project also experiments with compliance controls. A deployment can require allowlist membership, blocklist non-membership, both, or neither; those choices select a specific circuit and verifier (`sdk/types/src/policy_tx.rs:19-64`, `contracts/pool/src/pool.rs:224-280`). Selective disclosure lets a user create an off-chain proof about chosen notes, while Global View Key circuits propose broader auditor visibility but are not yet integrated into the pool contract path (`circuits/src/selectiveDisclosure.circom:16-81`, `docs/src/global_view_key.md:3-18`).

### How it achieves privacy

Privacy comes from separating **private witness data** from **public chain data**. The wallet knows note amounts, note private keys, blindings, and Merkle paths. It gives those values to a Circom witness calculator and Groth16 prover running on the user’s machine or in a browser worker (`sdk/prover/src/flows.rs:607-811`, `sdk/witness/src/lib.rs:24-86`, `sdk/client/src/prover/mod.rs:64-109`).

The resulting proof tells the pool contract, in effect: “I know valid unspent input notes in a recent pool root; I computed their nullifiers and the new commitments correctly; the values balance; and I satisfy this pool’s ASP policy.” It does so without publishing the private witness values (`circuits/src/transaction.circom:52-140`, `contracts/pool/src/pool.rs:561-625`). The contract verifies that proof, marks the public nullifiers as spent, inserts exactly two new commitments, performs any public token transfer, and emits events (`contracts/pool/src/pool.rs:561-645`).

Recipients learn that an output belongs to them through encrypted event payloads. Each output contains an X25519/XSalsa20-Poly1305 ciphertext of the amount and blinding. A wallet scans new commitment events, tries to decrypt each ciphertext, and accepts it only when the decrypted fields recompute the public commitment (`sdk/prover/src/encryption.rs:279-348`, `sdk/prover/src/notes.rs:24-83`, `sdk/state/src/storage.rs:1221-1424`).

### The normal workflow

```text
1. Start/sync wallet
   Derive privacy keys from one wallet signature; replay pool/registry/ASP events.
                              |
2. Register public keys       |  Optional public address-book entry.
                              v
3. Deposit public tokens --> private note commitments in the pool
                              |
4. Wallet discovers and stores its notes from encrypted events
                              |
5. Transfer or withdraw request
   select notes -> build witness -> generate Groth16 proof -> simulate/sign/submit
                              |
6. Pool contract
   check root/nullifiers/ext data/policy/proof -> mutate state -> emit events
                              |
7. Wallet syncs those events and updates its local spendable-note view
```

Key derivation and registration are implemented by the account/client and public-key registry (`sdk/client/src/account.rs:121-149`, `contracts/public-key-registry/src/lib.rs:60-89`). Note selection may create multiple transactions because each proof can consume at most two real notes; the planner consolidates larger selections into a sequence of 2-in/2-out calls (`sdk/tx-planner/src/plan/mod.rs:11-32`, `sdk/tx-planner/src/plan/mod.rs:158-198`). Transaction preparation then builds the witness and proof, simulates Soroban resources, asks the platform wallet to sign, and submits through RPC (`sdk/client/src/pool.rs:350-449`, `sdk/stellar/src/tx_prepare.rs:22-63`, `sdk/stellar/src/submit.rs:8-54`). Finally, the indexer stores emitted events and the state layer turns them into the wallet’s reconstructed local view (`sdk/stellar/src/indexer.rs:158-179`, `sdk/state/src/processor.rs:5-53`).

The browser and CLI execute that same logical workflow through different adapters. The browser uses wasm-bindgen, an OPFS SQLite storage worker, a proving worker, and a JavaScript wallet signer; the CLI uses local SQLite, an in-process prover, and the external `stellar` CLI for signing (`app/ARCHITECTURE.md:33-113`, `cli/src/stellar_cli.rs:1-9`, `sdk/client/src/prover/local.rs:14-51`).

### What this explanation should not imply

“Private” does not mean invisible. Deposits and withdrawals expose public amounts and addresses, every transaction exposes timing, nullifiers and new commitments appear together, and voluntary key registration links a Stellar address to privacy public keys (`docs/src/privacy-tradeoffs.md:7-29`). Recovery is also not wallet-seed-only: the deterministic keys can be recreated, but random note blindings must be recovered from encrypted event history (`sdk/prover/src/encryption.rs:57-98`, `sdk/prover/src/encryption.rs:202-230`).

The correct high-level conclusion is: this repository demonstrates how client-side Groth16 proving, encrypted note discovery, ASP policy circuits, Soroban verification, and local event-derived wallet state can fit together on Stellar. It does **not** establish production readiness: the project is unaudited, current testnet keys are local development keys, event recovery depends on retained or archived history, and GVK is not integrated end to end (`README.md:13-14`, `deployments/testnet/circuit_keys/README.md:21-32`, `README.md:119-124`, `docs/src/global_view_key.md:3-18`).

## 0. Scope boundary

### 0.1 Prior art, project work, and what cannot be attributed from source

The repository makes several ancestry claims explicitly:

| Circuit/code | What the repository supports claiming |
|---|---|
| `transaction`, `keypair`, dense Merkle proof/tree/updater | Their headers say they are based on Tornado Nova and adapted by Nethermind (`circuits/src/transaction.circom:1-3`, `circuits/src/keypair.circom:1-3`, `circuits/src/merkleProof.circom:1-3`, `circuits/src/merkleTree.circom:1-3`, `circuits/src/merkleTreeUpdater.circom:1-3`). |
| Sparse-Merkle verifier/hash | Their headers say they are based on `circomlib` and adapted by Nethermind (`circuits/src/smt/smthash_poseidon2.circom:1-3`, `circuits/src/smt/smtverifier.circom:1-3`, `circuits/src/smt/smtverifierlevel.circom:1-3`). The dependency is pinned by a commit hash (`circuits/circomlib.lock:1`). |
| Poseidon2 Circom constants and Rust permutation | The Circom constants name the HorizenLabs SAGE parameter generator (`circuits/src/poseidon2/poseidon2_const.circom:1-4`); the separate Rust crate identifies itself as the Horizen/IAIK Poseidon2 implementation (`poseidon2/Cargo.toml:1-10`, `poseidon2/src/lib.rs:1-9`). The README preserves the upstream Apache-2.0 attribution (`README.md:149-158`). |
| `policyTransaction*`, `policy_tx_gvk_*`, `aspMembership`, `aspNonMembership`, `globalViewKey` | These files implement the project-specific ASP-policy and GVK composition, but their headers do not make an authorship claim comparable to the Tornado/circomlib headers (`circuits/src/policyTransaction.circom:1-8`, `circuits/src/aspMembership.circom:1-8`, `circuits/src/aspNonMembership.circom:1-8`, `circuits/src/globalViewKey.circom:1-25`). **I am inferring** that this composition is original project work because borrowed files are marked and these are not; verify authorship from Git history and the authors, not from this inference. |

The licensing boundary is also explicit: project circuits are Apache-2.0 while bundled/adapted circomlib components remain LGPL-3.0 (`circuits/LICENSE:16-24`). That establishes licensing and attribution, not novelty.

### 0.2 Native and browser: shared core, different adapters

This is not two complete protocol implementations. `sdk/web` depends directly on the Rust client crate (`sdk/web/Cargo.toml:22-24`), while `sdk/client` depends on the shared prover, witness, state, Stellar, planner, disclosure, and types crates (`sdk/client/Cargo.toml:16-30`). The browser exposes that core through `wasm-bindgen` and adds storage/prover workers and a JavaScript wallet signer (`sdk/web/Cargo.toml:10-20`, `sdk/web/src/lib.rs:1-14`). The app architecture says both native and web paths use the same SDK core and SQLite schema (`app/ARCHITECTURE.md:7-23`).

The behavior is not perfectly identical. Native uses bundled SQLite and enables parallel `ark-circom` plus Wasmer/Cranelift (`sdk/state/Cargo.toml:24-30`, `sdk/witness/Cargo.toml:26-35`); web uses OPFS SQLite and separate workers (`sdk/web/src/workers/storage.rs:107-162`, `sdk/web/src/workers/prover.rs:627-692`). Native `LocalProver` rejects disclosure operations, while the web prover worker implements them (`sdk/client/src/prover/local.rs:61-77`, `sdk/web/src/workers/prover.rs:475-622`). The browser signer calls a SEP-0043-like wallet object; the CLI signer shells out to `stellar` so it never handles the wallet secret (`sdk/web/src/signer.rs:150-169`, `cli/src/stellar_cli.rs:1-9`). These are genuine adapter/feature divergences, not merely target-specific compilation.

### 0.3 Why “reference implementation” is accurate

The source corroborates the README label in four concrete ways:

1. The shipped testnet circuit keys were generated locally with no ceremony, and the repository says not to use them for production or real funds (`deployments/testnet/circuit_keys/README.md:21-32`).
2. Soroban RPC event retention is treated as roughly seven days, so recovery depends on an optional bootnode/archive path (`README.md:119-124`, `docs/src/bootnode.md:1-11`).
3. Browser persistence is OPFS-backed local SQLite; deleting site data can remove the wallet’s local state, and the README warns this can make funds inaccessible (`README.md:119-124`, `sdk/web/src/workers/storage.rs:107-162`).
4. GVK is explicitly described as circuit/type work whose contract integration, event emission, and wallet tooling are follow-ups (`docs/src/global_view_key.md:3-18`). The current pool proof and `ExtData` contain no GVK fields (`contracts/pool/src/pool.rs:78-120`).

The testnet deployment file contains contract identifiers for two named deployments, but that file proves configuration provenance, not that those contracts are live now (`deployments/testnet/deployments.json:1-40`). I did not perform live RPC verification, so I cannot claim current liveness, balances, code hashes, or seeded liquidity.

### 0.4 Sentences to use

**Project:** “Stellar Private Payments is an unaudited reference implementation that combines client-generated BN254 Groth16 proofs, a two-input/two-output note pool, optional ASP membership/non-membership policies, Soroban contracts, and native/browser SDK adapters; its shipped testnet keys and recovery infrastructure are development-grade (`README.md:13-18`, `README.md:66-82`, `deployments/testnet/circuit_keys/README.md:21-32`).”

**Your contribution:** “My contribution is _[state only the code, tests, or documentation I personally authored]_; I did not author the inherited Tornado-Nova, circomlib, or Poseidon2 foundations (`circuits/src/transaction.circom:1-3`, `circuits/src/smt/smtverifier.circom:1-3`, `poseidon2/src/lib.rs:1-9`).” I cannot fill the bracket from repository evidence; authorship must come from your commits and review history.

### 0.5 Documentation/source disagreements to remember

- `docs/src/introduction.md` repeats the README’s WIP/reference warning and feature claims (`docs/src/introduction.md:13-27`). The warning agrees with source status; the feature list should not be read as proof of live deployment.
- The README says clearing OPFS is “not critical” and that deterministic keys restore access, while the app architecture says events older than retention cannot be recovered without a bootnode (`README.md:119-124`, `app/ARCHITECTURE.md:209-227`). Random blindings are not deterministic (`sdk/prover/src/encryption.rs:202-230`), so the qualified architecture/source account wins: keys are recoverable, spendable-note state requires replayable ciphertext history.
- The introduction says payments do not reveal transaction amounts and that withdrawal creates no outputs (`docs/src/introduction.md:22-27`, `docs/src/introduction.md:43-48`). Source makes `ext_amount` public and pads every transaction to two outputs/commitment insertions (`contracts/pool/src/pool.rs:400-460`, `sdk/prover/src/flows.rs:583-593`, `contracts/pool/src/merkle_with_history.rs:101-190`). Source wins: internal note amounts are hidden, public deposit/withdrawal value is not, and “no outputs” means no real-valued requested output, not no output slots.
- The encryption module diagram says key-derivation message v2, but the constant and app architecture use v1 (`sdk/prover/src/encryption.rs:16-25`, `sdk/prover/src/encryption.rs:49-54`, `app/ARCHITECTURE.md:195-203`). Compiled Rust uses v1.
- GVK docs omit the current circuit’s salt/stages (`docs/src/global_view_key.md:60-85`, `circuits/src/globalViewKey.circom:62-93`); compiled Circom wins.
- Disclosure docs describe three checks, while current client code adds a separate unspent-nullifier check (`docs/src/disclosure.md:160-175`, `sdk/client/src/disclosure.rs:135-185`); compiled Rust wins.
- App architecture lists `disclose` on the Rust client and web pool surface, but native `LocalProver` returns unsupported while the browser worker implements it (`app/ARCHITECTURE.md:7-15`, `app/ARCHITECTURE.md:87-93`, `sdk/client/src/prover/local.rs:61-77`, `sdk/web/src/workers/prover.rs:475-622`). Treat disclosure as web-worker-supported, not uniformly supported by every client adapter.

## 1. Module map and reading order

### 1.1 Workspace map

“Public surface” below means the surface exposed to another crate, contract caller, binary, or JavaScript consumer—not every `pub` item.

| Layer / component | Responsibility and public surface | Direct dependency shape / why it is separate |
|---|---|---|
| Protocol types — `sdk/types` | Defines amounts, field and key wrappers, policy flags, GVK/disclosure records, correlation IDs, and shared serialized types (`sdk/types/Cargo.toml:13-31`, `sdk/types/src/lib.rs:1-37`). It is the low-dependency vocabulary shared by native and wasm; optional `rusqlite` conversion keeps DB coupling feature-gated (`sdk/types/Cargo.toml:9-31`). |
| State — `sdk/state` | Owns the SQLite schema, migrations, raw-event ingestion, projections, Merkle reconstruction, and note discovery (`sdk/state/Cargo.toml:8-22`, `sdk/state/src/schema.sql:1-9`). It depends on `types` and `stellar`, while target-specific SQLite linkage is isolated here (`sdk/state/Cargo.toml:11-30`). |
| Stellar transport — `sdk/stellar` | Provides RPC, XDR encoding, simulation/assembly/submission, ext-data hashing, and event indexing (`sdk/stellar/Cargo.toml:9-32`, `sdk/stellar/src/lib.rs:1-21`). It isolates Stellar-specific XDR/RPC dependencies from cryptography and planning (`sdk/stellar/Cargo.toml:14-29`). |
| Prover — `sdk/prover` | Implements key/note cryptography, transaction/disclosure circuit inputs, R1CS/Groth16 proving, and proof serialization (`sdk/prover/Cargo.toml:8-35`, `sdk/prover/src/lib.rs:1-24`). It isolates heavy Arkworks, Poseidon2, X25519, and secret-box dependencies and is compiled for native and wasm (`sdk/prover/Cargo.toml:18-43`). |
| Witness — `sdk/witness` | Loads Circom witness WASM through Wasmer, parses JSON inputs, and returns BN254 witness elements (`sdk/witness/src/lib.rs:1-17`, `sdk/witness/src/lib.rs:24-86`). Its target-specific Wasmer engines and native-only parallelism are a real build/runtime boundary (`sdk/witness/Cargo.toml:26-35`). |
| Planner — `sdk/tx-planner` | Selects spendable notes and expands a spend into one or more 2-in/2-out steps (`sdk/tx-planner/src/plan/mod.rs:11-32`, `sdk/tx-planner/src/plan/mod.rs:110-199`). It depends only on `types`, making pure planning independently testable (`sdk/tx-planner/Cargo.toml:10-16`). |
| Disclosure — `sdk/disclosure` | Computes receipt context/key metadata and validates disclosure proofs/roots (`sdk/disclosure/Cargo.toml:8-19`, `sdk/disclosure/src/lib.rs:1-18`). It is a reusable verifier above `prover` and `types`, separate from chain-status checks (`sdk/disclosure/Cargo.toml:10-19`). |
| Client — `sdk/client` | Orchestrates accounts, sync, planning, proving, signing, submission, and native blocking wrappers (`sdk/client/src/lib.rs:1-43`, `sdk/client/src/lib.rs:45-133`). It is the application SDK over every lower SDK crate (`sdk/client/Cargo.toml:16-30`). |
| Web — `sdk/web` | Exports wasm-bindgen handles and supplies JS-wallet, storage-worker, and prover-worker adapters (`sdk/web/Cargo.toml:8-20`, `sdk/web/src/lib.rs:1-14`). It is separate because `cdylib`/worker binaries, JS values, OPFS, and browser APIs do not belong in the portable client (`sdk/web/Cargo.toml:10-44`). |
| SDK tests — `sdk/tests` | Tests the public client surface, planner integration, wallet workflows, and telemetry behavior (`sdk/tests/Cargo.toml:8-25`, `sdk/tests/src/tests/mod.rs:1-3`). It is non-publishable test composition rather than production API (`sdk/tests/Cargo.toml:6-12`). |
| Circuits — `circuits/` | Contains Circom transaction/policy/GVK/disclosure circuits plus a Rust build/test harness (`circuits/Cargo.toml:1-18`, `circuits/Cargo.toml:24-48`). The circuit compiler consumes `.circom`; Rust consumers use generated metadata/test utilities. |
| Key formats — `circuit-keys/` | Converts Groth16 key material among Arkworks proving keys, JSON VKs, Soroban binary VKs, and Rust constants (`circuit-keys/Cargo.toml:9-17`, `circuit-keys/src/lib.rs:1-31`). It exists so build scripts and ceremony tooling share one serialization definition. |
| Poseidon2 — `poseidon2/` | Vendored/upstream Rust Poseidon2 permutation implementation (`poseidon2/src/lib.rs:1-9`). It is a distinct upstream-derived primitive consumed through workspace dependency wiring (`poseidon2/Cargo.toml:1-10`). |
| Fixtures — `testdata/` | Holds per-circuit WASM/R1CS/proving/verifying-key artifacts used by tests and contract builds (`deployments/testnet/circuit_keys/README.md:34-61`). These are generated artifacts, not a runtime service. |
| Pool contract | Verifies policy roots/ext data/proof, rejects spent nullifiers, transfers public tokens, appends exactly two commitments, and emits events (`contracts/pool/src/pool.rs:513-645`). Its public surface is the Soroban contract interface and stable error/event ABI (`contracts/pool/src/pool.rs:28-61`, `contracts/pool/src/pool.rs:187-201`). |
| Groth16 verifier contract | Embeds one VK at build time and verifies an uncompressed 256-byte BN254 proof with Soroban host curve/pairing operations (`contracts/circom-groth16-verifier/src/lib.rs:3-13`, `contracts/circom-groth16-verifier/src/lib.rs:78-140`). Per-circuit deployment is therefore a real contract boundary. |
| ASP membership contract | Maintains an append-only dense tree of authorized membership leaves and exposes roots/proofs through contract state (`contracts/asp-membership/src/lib.rs:1-18`, `contracts/asp-membership/src/lib.rs:69-151`). It shares contract-side hashing through `soroban-utils` (`contracts/asp-membership/Cargo.toml:13-18`). |
| ASP non-membership contract | Maintains a sparse map/tree of blocked note keys and exposes a root used by non-membership proofs (`contracts/asp-non-membership/src/lib.rs:1-18`, `contracts/asp-non-membership/src/lib.rs:126-191`). It is separate because its update/proof semantics are different from append-only membership. |
| Public-key registry | Lets an account authenticate and publish 32-byte note/encryption public keys, then emits them (`contracts/public-key-registry/src/lib.rs:5-37`, `contracts/public-key-registry/src/lib.rs:60-89`). It is public discovery infrastructure, not pool custody. |
| Soroban utilities / contract types | `soroban-utils` supplies field/Poseidon2/Groth16 conversion code; `contract-types` fixes proof-point/error wire types (`contracts/soroban-utils/Cargo.toml:13-21`, `contracts/types/src/lib.rs:1-19`). Both avoid duplicating ABI-critical code across contracts. |
| CLI — `cli/` | Provides `spp`, loads deployments/artifacts/local SQLite, uses the blocking client, and delegates wallet signing to the external Stellar CLI (`cli/Cargo.toml:13-32`, `cli/src/stellar_cli.rs:1-9`). |
| Browser app — `app/` | JavaScript/React UI around the wasm SDK and SEP-0043 wallet interface (`app/ARCHITECTURE.md:7-23`, `app/ARCHITECTURE.md:185-203`). It does not reimplement protocol math. |
| `spp-bob-wallet/` | This is an ignored example/local SQLite wallet artifact, not a workspace product (`sdk/client/examples/SETUP.md:122-122`, `sdk/client/examples/SETUP.md:289-289`). **I could not find a crate manifest or product source in it.** |
| Bootnode — `tools/bootnode` | Archives paginated RPC events in PostgreSQL and serves them until it hands the client back to its own RPC (`tools/bootnode/README.md:1-18`, `docs/src/bootnode.md:13-35`). It is excluded from the root workspace and built separately (`Cargo.toml:1-22`). |
| Ceremony CLI | Imports/exports/contributes Groth16 parameters using the shared key-format crate (`tools/ceremony-cli/Cargo.toml:9-27`). Its existence does not make current keys ceremonial; the testnet key README says they are local (`deployments/testnet/circuit_keys/README.md:21-32`). |
| Deployments | Scripts map policy suffixes to matching verifier artifacts and construct pool/ASP/registry contracts (`deployments/scripts/deploy.sh:139-175`, `deployments/scripts/deploy.sh:523-562`). JSON files are deployment manifests, not live-state proofs (`deployments/testnet/deployments.json:1-40`). |
| E2E tests | Compose real circuits, generated proofs, Soroban test hosts, and planning/coherence checks (`e2e-tests/Cargo.toml:9-32`, `e2e-tests/src/tests/e2e_pool_2_in_2_out.rs:1-37`). |

Some crate splits are architectural and some are organizational. `web`, `witness`, `state`, `stellar`, `prover`, and contract crates isolate target/ABI/heavy-dependency constraints shown above. `types` and `tx-planner` create low-dependency reusable kernels. `disclosure` and `sdk/tests` look primarily like organization/testability boundaries: their manifests show no unique target or ABI constraint beyond reuse (`sdk/disclosure/Cargo.toml:10-21`, `sdk/tests/Cargo.toml:14-28`). **That last classification is an inference from manifests; verify the intended ownership boundary with maintainers.**

### 1.2 Dependency-ordered reading list

1. Read value semantics first: `sdk/types/src/amounts.rs`, `sdk/types/src/policy_tx.rs`, and `sdk/types/src/lib.rs`; later layers assume their signed/unsigned and field encodings (`sdk/types/src/amounts.rs:51-89`, `sdk/types/src/amounts.rs:182-218`, `sdk/types/src/policy_tx.rs:19-64`).
2. Read the exact hashes and encryption: `sdk/prover/src/crypto.rs`, then `encryption.rs` and `notes.rs` (`sdk/prover/src/crypto.rs:33-163`, `sdk/prover/src/encryption.rs:49-199`, `sdk/prover/src/notes.rs:24-83`).
3. Read the constraints: `circuits/src/transaction.circom`, then `policyTransaction.circom`, ASP circuits, disclosure, and GVK (`circuits/src/transaction.circom:52-140`, `circuits/src/policyTransaction.circom:3-8`).
4. Read `sdk/prover/src/flows.rs` to see how Rust creates exactly those signal names and ciphertexts (`sdk/prover/src/flows.rs:607-831`).
5. Read the on-chain mirror: `contracts/soroban-utils/src/poseidon2.rs`, `contracts/pool/src/merkle_with_history.rs`, then `contracts/pool/src/pool.rs` (`contracts/soroban-utils/src/poseidon2.rs:112-130`, `contracts/pool/src/merkle_with_history.rs:101-236`, `contracts/pool/src/pool.rs:513-645`).
6. Read persistence backwards from events: `sdk/stellar/src/indexer.rs`, `sdk/state/src/events_parsers.rs`, `sdk/state/src/processor.rs`, `sdk/state/src/storage.rs` (`sdk/state/src/processor.rs:5-53`, `sdk/state/src/storage.rs:1221-1424`).
7. Read planning and client orchestration: `sdk/tx-planner/src/plan/mod.rs`, `sdk/client/src/pool.rs`, `sdk/client/src/sync.rs` (`sdk/tx-planner/src/plan/mod.rs:110-199`, `sdk/client/src/pool.rs:350-449`, `sdk/client/src/sync.rs:27-52`).
8. Only then read native/web adapters and applications (`sdk/client/src/handle.rs:1-45`, `sdk/web/src/protocol.rs:60-164`, `app/ARCHITECTURE.md:33-113`).

The five highest-conceptual-weight files are: `circuits/src/policyTransaction.circom` (the proof statement and its warning), `sdk/prover/src/flows.rs` (off-chain construction), `contracts/pool/src/pool.rs` (enforcement), `sdk/state/src/storage.rs` (recoverable local model), and `sdk/client/src/pool.rs` (lifecycle orchestration) (`circuits/src/policyTransaction.circom:3-8`, `sdk/prover/src/flows.rs:607-831`, `contracts/pool/src/pool.rs:513-645`, `sdk/state/src/storage.rs:1221-1424`, `sdk/client/src/pool.rs:350-449`).

## 2. Cryptographic derivations

### 2a. Wallet-derived keys

The module’s rationale should be quoted, not generalized:

> “SHA-256 is used (not Poseidon2) because this is an **off-circuit** KDF … standard, well-audited, and available in every target … Poseidon2 is only advantageous inside arithmetic circuits.” (`sdk/prover/src/encryption.rs:8-12`)

It also records the deliberate signature collapse:

> “Earlier versions requested two wallet signatures … The current scheme requests **one** signature … The trade-off is that compromise of that one signature compromises both key families.” (`sdk/prover/src/encryption.rs:16-31`)

Actual code uses the fixed wallet message `Privacy Pool Key Derivation [v1]`, despite the module diagram saying `[v2]`; the compiler obeys the constants and functions, so the diagram is stale (`sdk/prover/src/encryption.rs:16-25`, `sdk/prover/src/encryption.rs:49-54`). Given wallet signature bytes `sig`:

- note seed = `SHA256("privacy-pool/note-key/v1" || sig)`, reduced modulo BN254 Fr to form `NotePrivateKey`; public key is then the protocol Poseidon2 derivation (`sdk/prover/src/encryption.rs:142-179`, `sdk/prover/src/encryption.rs:181-199`).
- X25519 secret seed = `SHA256("privacy-pool/encryption-key/v1" || sig)` and `StaticSecret::from(seed)` applies X25519’s scalar handling; the public key is `PublicKey::from(secret)` (`sdk/prover/src/encryption.rs:100-140`, `sdk/prover/src/encryption.rs:181-199`).
- ASP membership blinding = `SHA256("privacy-pool/asp-secret/v1" || 0x00 || network_id || 0x00 || sig)`, reduced into BN254 Fr (`sdk/prover/src/encryption.rs:49-54`, `sdk/prover/src/encryption.rs:75-98`).

From the same wallet seed, a wallet that reproduces the same signature and network identifier can deterministically restore the note key, X25519 key, and membership blinding (`sdk/prover/src/encryption.rs:57-98`). Per-note blindings are different: they come from fresh randomness and the code says they “must be stored/recovered from encrypted events” (`sdk/prover/src/encryption.rs:202-230`). Therefore seed-only recovery of spendable notes also requires historical ciphertext events; without the local DB and without an archive that reaches those events, the keys recover but the note amounts/blindings do not. **That operational-loss conclusion is an inference from the random blinding and retention code; verify it with the intended recovery UX** (`README.md:119-124`, `sdk/state/src/storage.rs:1221-1424`).

### 2b. Exact in-circuit/off-chain derivations

All field-byte inputs in the Rust hash helpers are interpreted as little-endian BN254 field values (`sdk/prover/src/serialization.rs:41-58`). `H2(a,b,d)` instantiates Poseidon2 width 3 with state `[a,b,d_or_0]`; `H3(a,b,c,d)` uses width 4 with `[a,b,c,d]` (`sdk/prover/src/crypto.rs:33-63`).

| Derivation | Exact preimage/order/domain | Circom/on-chain agreement point | Observer capability; wrong-preimage symptom |
|---|---|---|---|
| Note public key | `H2(private_key, 0, 3)` (`sdk/prover/src/crypto.rs:88-95`, `sdk/prover/src/crypto.rs:165-169`). | `Keypair` hashes `[inPrivateKey, 0]` with domain `3` (`circuits/src/keypair.circom:7-19`). | An observer can compute it only with the private key. A mismatch makes ownership/signature constraints fail, so proving fails loudly; a wallet that derives and registers a consistently wrong key can also create notes that the intended seed cannot spend, a silent recovery failure until spend. |
| Note commitment | `H3(amount, note_public_key, blinding, 1)` in that order (`sdk/prover/src/crypto.rs:97-108`). | Transaction input/output commitments use the same fields/domain (`circuits/src/transaction.circom:52-100`, `circuits/src/transaction.circom:102-128`). | The public sees the commitment but cannot recompute it without amount and blinding (public key may be known through the opt-in registry). A Rust/Circom mismatch fails witness/proof loudly; a ciphertext with wrong amount/blinding is ignored during discovery because the wallet recomputes and compares the commitment (`sdk/prover/src/notes.rs:42-58`). |
| Signature | `H3(private_key, commitment, packed_leaf_index, 4)` (`sdk/prover/src/crypto.rs:110-122`). | `Keypair`’s signature component uses private key, commitment, and `pathIndices` with domain `4` (`circuits/src/keypair.circom:22-34`). | It is a circuit-internal ownership binding, not an Ed25519 signature. An observer cannot compute it without the note private key; wrong index/key/commitment makes the proof fail loudly. |
| Nullifier | `H3(commitment, packed_leaf_index, signature, 2)` (`sdk/prover/src/crypto.rs:124-139`). The private key enters **indirectly through signature**; Merkle siblings and wallet signature bytes do not enter. | Transaction exposes and constrains the nullifier through the input keypair/path (`circuits/src/transaction.circom:52-100`). | The chain sees a nullifier but cannot link it to a commitment without the secret-derived signature. Wrong preimage fails circuit/public-input agreement loudly; if client and circuit agreed on a different formula while old notes used the old formula, double-spend semantics or spendability would change—this requires a coordinated protocol migration. |
| ASP membership leaf | `H2(note_public_key, membership_blinding, 1)` (`sdk/prover/src/crypto.rs:151-163`). | The membership circuit recomputes the leaf and constrains its Merkle root (`circuits/src/aspMembership.circom:19-47`). | The ASP/on-chain observer sees leaves/root, but cannot test a public key without its network-bound blinding. Wrong blinding/path fails the policy proof loudly. |
| Sparse non-membership leaf | `H2(key, value, 1)` for occupied SMT leaves; empty structure uses zero nodes (`sdk/prover/src/sparse_merkle.rs:21-27`, `sdk/prover/src/sparse_merkle.rs:98-118`). | Circom’s `SMTHash1` and the Soroban ASP contract use the same order/domain (`circuits/src/smt/smthash_poseidon2.circom:8-20`, `contracts/asp-non-membership/src/lib.rs:126-145`). | Keys/values and root are public contract state; proof privacy comes from proving absence without revealing a private note key as a public input. Hash/bit-order drift makes roots disagree and policy proofs reject loudly (`sdk/prover/src/sparse_merkle.rs:33-47`, `contracts/asp-non-membership/src/lib.rs:166-191`). |
| `zero_leaf` | `H2(Field("XLM"), 0, 0)`, where the bytes are ASCII `88,76,77` (`sdk/prover/src/crypto.rs:20-31`, `sdk/prover/src/crypto.rs:146-149`). | Contract utility constructs the same value (`contracts/soroban-utils/src/poseidon2.rs:112-130`). | It is public and computable. The nonzero sentinel prevents the empty tree/root from collapsing into literal zero, which the contract treats as invalid (`contracts/pool/src/merkle_with_history.rs:209-213`). A mismatch causes immediate root drift/unknown-root rejection. |
| Poseidon2 domain parameter | It occupies the capacity element: `[a,b,domain]` or `[a,b,c,domain]`; omitted `None` becomes zero (`sdk/prover/src/crypto.rs:33-63`). | Circom’s fixed hash template writes the domain into the last state element (`circuits/src/poseidon2/poseidon2_hash.circom:5-23`). | All inputs and domains are computable when preimages are known. **Inference:** distinct domain bytes prevent the same field tuple from being interpreted across commitment/nullifier/key/signature roles; without them, cross-protocol equality becomes possible. Verify the intended domain registry with the circuit authors. Wrong domains cause proof/root mismatch loudly. |
| External-data hash | Soroban XDR-encodes a map, sorted by symbol key, containing `encrypted_output0`, `encrypted_output1`, `ext_amount`, and `recipient`; it Keccak-256 hashes the bytes and reduces big-endian output modulo BN254 (`sdk/stellar/src/ext_data_hash.rs:10-64`). | Pool rebuilds the same map/hash and compares it with the proof public input (`contracts/pool/src/pool.rs:104-143`, `contracts/pool/src/pool.rs:572-576`). | Every observer can compute it because all four fields are transaction arguments/events. Mutating any bound field without a new proof yields `WrongExtHash`, loudly. The circuit binds only the hash, not plaintext-to-ciphertext correctness (`circuits/src/transaction.circom:102-140`); **inference/security seam:** a malicious prover can bind garbage ciphertext while creating valid commitments, causing silent recipient non-discovery rather than proof rejection (`sdk/prover/src/flows.rs:656-711`, `sdk/prover/src/flows.rs:802-831`). |

The warning atop `policyTransaction.circom` is part of the security model:

> “DO NOT declare `component main` in this file … doing so would add `inPublicKey` to the public signals and leak the sender’s note public key.” (`circuits/src/policyTransaction.circom:3-8`)

### 2c. Encryption and discovery

The plaintext is exactly 48 bytes: 16-byte little-endian `u128` note amount followed by the 32-byte blinding (`sdk/prover/src/encryption.rs:233-276`). Encryption generates an ephemeral X25519 secret/public key, performs Diffie-Hellman with the recipient public key, and uses XSalsa20-Poly1305 with a random nonce; the serialized ciphertext is 120 bytes (`sdk/prover/src/encryption.rs:279-348`). The output ciphertexts are passed in `ExtData`, included in the commitment event, and are not persisted as pool storage fields (`sdk/prover/src/flows.rs:802-831`, `contracts/pool/src/pool.rs:187-201`, `contracts/pool/src/pool.rs:630-643`).

`try_decrypt_and_derive_user_note` attempts decryption, ignores zero-amount padding, derives the wallet public key, recomputes the commitment from decrypted amount/blinding, and only then derives the nullifier/user-note record (`sdk/prover/src/notes.rs:24-83`). State processing scans commitment events in chunks and runs that derivation for each account whose scan high-water mark has not passed the commitment (`sdk/state/src/storage.rs:1221-1424`, `sdk/client/src/core/state.rs:9-40`). There is no recipient tag in the 120-byte encoding or discovery loop (`sdk/prover/src/encryption.rs:279-348`, `sdk/state/src/storage.rs:1288-1421`). The incremental cost is therefore linear in new commitments times local accounts, plus one X25519/secret-box trial per ciphertext slot. That complexity statement is an inference from the nested scan; the repository contains no growth benchmark.

### 2d. Three Merkle structures

`sdk/prover/src/merkle.rs` reconstructs the append-only dense note tree and paths for witness creation; its core Merkle functions are re-exported from the Rust `circuits::core::merkle` module rather than copied (`sdk/prover/src/merkle.rs:1-24`). `sdk/prover/src/sparse_merkle.rs` represents keyed ASP blocklist state using LSB-first key bits, while `contracts/pool/src/merkle_with_history.rs` is the authoritative on-chain append-only tree (`sdk/prover/src/sparse_merkle.rs:1-47`, `contracts/pool/src/merkle_with_history.rs:1-18`).

The contract inserts exactly two leaves per transaction and stores a ring of 90 roots (`contracts/pool/src/merkle_with_history.rs:101-190`, `contracts/pool/src/merkle_with_history.rs:193-236`). That is a **90-transaction-state window, not a time window**. It lets a proof generated against a recent root survive intervening transactions; without history, any transaction landing between witness generation and submission would make the proof’s root unknown. The proof remains cryptographically valid but the contract rejects it as stale (`contracts/pool/src/pool.rs:561-571`).

Rust/Circom/Soroban must agree on field endianness, zero nodes, left/right bit order, compression, and insertion order (`sdk/prover/src/merkle.rs:78-132`, `circuits/src/merkleProof.circom:9-35`, `contracts/pool/src/merkle_with_history.rs:101-190`). Drift is loud at proof generation/verification or `UnknownRoot`; discovery drift can also leave a wallet unable to construct a valid path until state reconstruction is fixed.

### 2e. Global View Key

The current status is crucial: GVK types and circuits exist, but the docs call contract integration, event emission, and wallet tooling follow-up work (`docs/src/global_view_key.md:3-18`). The current pool proof/transaction ABI has no GVK public inputs or ciphertext (`contracts/pool/src/pool.rs:78-120`). Therefore GVK is **not yet an end-to-end pool feature** in this source snapshot.

The circuit validates a BabyJubJub public key `D`, derives a per-note scalar `r`, computes `R = rG`, computes shared point `S = rD`, derives a Poseidon2 keystream from `S`, and adds the note payload fields to that stream (`circuits/src/globalViewKey.circom:35-60`, `circuits/src/globalViewKey.circom:62-171`). The exact current nonce chain is:

`h1 = H3(pubkey, amount, blinding, 5)`; `h2 = H3(h1, salt, D.x, 5)`; `h3 = H3(h2, D.y, nonce, 5)`; `r = H3(h3, note_index, 0, 5)` (`circuits/src/globalViewKey.circom:62-93`). The KDF permutes `[S.x,S.y,0,6]`, and ciphertext components add note fields to the resulting elements (`circuits/src/globalViewKey.circom:157-171`). The circuit constrains `r`, curve operations, subgroup/cofactor behavior, and ciphertext equations, so a valid proof certifies that the public ciphertext corresponds to the same private note fields used by the enclosing circuit (`circuits/src/globalViewKey.circom:95-171`).

In `traceable`, ciphertext carries note public key, amount, and blinding; the GVK holder can recompute the commitment and follow that note (`sdk/types/src/gvk.rs:28-51`, `circuits/src/globalViewKey.circom:214-284`). In `viewonly`, it carries amount and blinding but not note public key, so the holder sees values but lacks the field needed to directly recompute the normal commitment (`sdk/types/src/gvk.rs:28-51`, `docs/src/global_view_key.md:22-44`). This describes circuit intent, not deployed behavior.

There is a documentation disagreement. `docs/src/global_view_key.md` shows an older derivation that omits `salt` and does not match the four-stage source chain (`docs/src/global_view_key.md:60-85`). The Circom compiler obeys `globalViewKey.circom`, so its formula above is authoritative for generated artifacts.

`gvk_circuit_stem` maps `(PolicyFlags, PoolGvkMode)` to one of the concrete circuit stems, and `parse_gvk_circuit_stem` reverses that mapping (`sdk/types/src/gvk.rs:173-237`). There are four policy combinations times two GVK modes, hence eight stems (`sdk/types/src/gvk.rs:173-216`). Separate artifacts exist because policy constraints and GVK output shape are compiled into the R1CS/public-signal layout rather than selected dynamically. That reason is an inference from the static stem/R1CS design; verify whether maintainers considered a selector-based universal circuit.

### 2f. Policy flags and verifier selection

`none`, `allowlist`, `blocklist`, and `allowlist-blocklist` are the four valid two-bit combinations; helpers determine which ASP roots/proofs are required (`sdk/types/src/policy_tx.rs:19-64`, `sdk/types/src/policy_tx.rs:68-153`). Client transaction preparation reads the pool flags from chain context and requires/omits corresponding proofs (`sdk/client/src/transact.rs:83-185`). The pool stores immutable policy flags plus addresses of the matching verifier and ASP contracts at initialization (`contracts/pool/src/pool.rs:224-280`, `contracts/pool/src/policy.rs:1-17`). At transact time it fetches current ASP roots, compares them with proof public inputs, and invokes its configured verifier (`contracts/pool/src/pool.rs:585-603`).

Each verifier contract includes one generated VK at build time and checks its expected public-input count (`contracts/circom-groth16-verifier/src/lib.rs:25-26`, `contracts/circom-groth16-verifier/src/lib.rs:78-140`). Deployment scripting maps the policy suffix to the matching `*_vk_soroban.bin`/verifier artifact and passes both flags and verifier address into the pool constructor (`deployments/scripts/deploy.sh:139-175`, `deployments/scripts/deploy.sh:523-562`). This is the exact normal-path barrier against a client choosing a weaker circuit: the client does not supply a verifier address; the pool calls the one fixed at deployment (`contracts/pool/src/pool.rs:224-280`, `contracts/pool/src/pool.rs:585-603`).

There is still a configuration trust seam: the constructor stores flags and a verifier address but does not cryptographically prove that the embedded VK corresponds to those flags (`contracts/pool/src/pool.rs:224-280`). The deploy script establishes that association. **Inference:** an operator who deliberately or accidentally deploys a mismatched verifier can create a malformed policy deployment; an input-count mismatch is likely loud, but a same-shape weaker circuit could undermine policy. Verify same-shape feasibility with circuit/VK inventories and close it by binding a circuit/VK identifier in initialization.

## 3. Implementation ledger

| Concept | Implementation A | Implementation B/C | Why both exist | What keeps them in sync / failure mode |
|---|---|---|---|---|
| Poseidon2 | Rust client hashing in `sdk/prover/src/crypto.rs:33-74`. | Circom permutation/wrappers in `circuits/src/poseidon2/poseidon2_hash.circom:5-23`; Soroban hash in `contracts/soroban-utils/src/poseidon2.rs:1-130`. | Rust constructs witnesses and discovers notes; Circom constrains proofs; Soroban maintains/verifies public state. | Ignored artifact-dependent circuit tests compare Circom results with Rust utilities, while real-proof E2E explicitly compares circuit and pool roots (`circuits/src/test/prove_poseidon2.rs:85-142`, `e2e-tests/src/tests/e2e_pool_2_in_2_out.rs:181-208`). Rust/Circom mismatch stops proving; Circom/contract mismatch rejects proofs or roots. |
| Commitment/nullifier | Rust derivations (`sdk/prover/src/crypto.rs:88-139`). | Circom transaction/keypair (`circuits/src/transaction.circom:52-128`, `circuits/src/keypair.circom:7-34`). Contract does not recompute private commitments/nullifiers; it accepts constrained public inputs and tracks nullifiers (`contracts/pool/src/pool.rs:561-625`). | Privacy requires private preimages stay off chain while the contract enforces proof outputs. | Circuit unit/E2E tests. Wrong Rust preimage fails witness constraints; wrong contract public-input order fails verifier. |
| Dense Merkle | SDK reconstruction reuses Rust `circuits::core::merkle` (`sdk/prover/src/merkle.rs:1-24`). | Circom inclusion (`circuits/src/merkleProof.circom:9-35`); Soroban insertion/history (`contracts/pool/src/merkle_with_history.rs:101-236`). | Client proves historical inclusion; circuit verifies it privately; contract owns canonical append state. | Real-proof E2E explicitly compares circuit and pool roots (`e2e-tests/src/tests/e2e_pool_2_in_2_out.rs:181-208`). Drift appears as witness failure or unknown root. |
| Sparse ASP tree | Rust proof construction (`sdk/prover/src/sparse_merkle.rs:21-118`). | Circom non-membership verifier (`circuits/src/smt/smtverifier.circom:1-80`); Soroban ASP state (`contracts/asp-non-membership/src/lib.rs:126-191`). | Same three-role split as dense Merkle, but keyed sparse semantics. | Sparse circuit tests and full policy E2E exercise the boundary (`circuits/src/test/prove_sparse.rs:100-102`, `e2e-tests/src/tests/e2e_pool_2_in_2_out.rs:141-180`). Bit-order/domain drift yields different roots and loud rejection. |
| Circuit Rust test utilities vs production | Circuit tests calculate expected keypair/transaction values in `circuits/src/test/utils/keypair.rs:1-38` and `circuits/src/test/utils/transaction.rs:1-40`. | Production code is independently located in `sdk/prover/src/crypto.rs:33-163` and `sdk/prover/src/flows.rs:607-831`. | Test utilities feed the compiler/test harness without importing the application prover. | They are separate preimage/wiring implementations but both depend on the workspace `zkhash` primitive (`circuits/Cargo.toml:24-48`, `sdk/prover/Cargo.toml:12-17`). Thus they are a differential oracle for wiring/encoding, not an independent oracle for a shared Poseidon2 bug. |
| Async client vs blocking client | Core client APIs are async and traits use `async_trait(?Send)` (`sdk/client/src/prover/mod.rs:112-130`, `sdk/client/src/storage/mod.rs:113-190`). | `sdk/client/src/blocking/` owns a Tokio runtime and `block_on`s async calls (`sdk/client/src/blocking/runtime.rs:1-22`). | Browser/RPC/worker operations require async; CLI gets synchronous ergonomics. | Same core methods are invoked. Creating the blocking runtime inside an existing Tokio runtime intentionally panics (`sdk/client/src/blocking/runtime.rs:8-17`). |
| Native vs browser storage | `LocalStorage` opens/forks local SQLite connections and uses `RefCell` guards (`sdk/client/src/storage/local.rs:24-79`). | Browser storage worker owns OPFS SQLite and services typed requests (`sdk/web/src/workers/storage.rs:107-162`, `sdk/web/src/protocol.rs:60-164`). | OPFS SQLite must be isolated to a worker; long DB work must not block the UI. | Both call the same `state` functions/schema (`app/ARCHITECTURE.md:7-23`). Messages cross as Serde request/response values; the JS boundary uses `serde-wasm-bindgen` (`sdk/web/src/storage.rs:96-112`). |
| Native vs browser prover | `LocalProver` owns in-process circuit engines (`sdk/client/src/prover/local.rs:14-51`). | Browser worker receives transaction/disclosure params and returns prepared proof/receipt (`sdk/web/src/workers/prover.rs:463-622`). | Browser proving is CPU-heavy and the main/UI thread cannot host it without freezing; Wasmer/browser constraints differ (`sdk/witness/Cargo.toml:26-35`). | Both use `ProverEngine` for transaction proving (`sdk/client/src/prover/mod.rs:64-109`). Divergence: native `LocalProver` does not implement disclosure, browser does (`sdk/client/src/prover/local.rs:61-77`). |
| Native vs browser signer | CLI `AliasSigner` invokes `stellar keys address`, `sign`, and SEP-53 message signing (`cli/src/stellar_cli.rs:81-228`, `cli/src/signer.rs:17-66`). | Browser signer calls wallet `signAuthEntry`/`signTransaction` and validates returned JS shape (`sdk/web/src/signer.rs:50-82`, `sdk/web/src/signer.rs:150-219`). | Secrets remain in the platform wallet on each target. | Both implement the client `Signer` trait (`sdk/client/src/signer/mod.rs:11-29`). Wallet API behavior remains adapter-specific. |
| Correlation context | Native tracing layer stores/looks up correlation IDs (`sdk/types/src/correlation.rs:83-125`). | Browser uses a stack around async/worker calls (`sdk/types/src/correlation.rs:32-81`) and every worker message wraps a `CorrelatedRequest` (`sdk/web/src/protocol.rs:1-9`). | Worker responses can complete out of order; IDs join logs/errors/request-response across boundaries. | The ID is explicitly copied into every request and returned response (`sdk/web/src/protocol.rs:60-164`). Losing it does not alter cryptography but breaks diagnosis and response matching. |
| Unoptimized vs optimized witness WASM | Release compiler artifacts live under `target/circuits-artifacts/release`. | Web build runs Binaryen 131 `wasm-opt -Os` and copies browser artifacts (`sdk/web/scripts/build.sh:114-162`). | Browser delivery needs smaller WASM. | `witness_identity` feeds the same inputs to both and requires byte-identical witness output (`e2e-tests/src/tests/witness_identity.rs:141-180`). It catches optimizer-induced semantic changes or wrong artifact copying, not a shared circuit-logic error. |

The key design lesson is that “same math” means **same ordered field elements, domains, encoding, bit order, and public-input order**. Compilation does not prove cross-language equivalence; artifact-dependent circuit tests and real-proof E2E are the executable bridge, while `coherence/policy.rs` only checks SDK/contract policy-bit semantics (`circuits/src/test/prove_poseidon2.rs:85-142`, `e2e-tests/src/tests/e2e_pool_2_in_2_out.rs:181-208`, `e2e-tests/src/tests/coherence/policy.rs:1-35`).

## 4. Communication, end to end

### A. Browser path

1. **App → wasm SDK.** The JavaScript app creates wasm-exposed client/account/pool handles and passes ordinary JS strings/numbers/objects; `serde-wasm-bindgen` converts structured values at the boundary (`app/ARCHITECTURE.md:33-79`, `sdk/web/src/storage.rs:96-112`). Wallet secrets do not enter Rust: the wallet receives auth-entry/transaction XDR and returns signed data (`sdk/web/src/signer.rs:50-82`).
2. **Main wasm → storage worker.** Requests are typed `StorageWorkerRequest` values wrapped in `CorrelatedRequest { correlation_id, request }`; responses carry the same ID (`sdk/web/src/protocol.rs:1-9`, `sdk/web/src/protocol.rs:60-164`). The wallet signature used for deterministic key derivation crosses this boundary, and derived account keys are stored in the worker-owned DB (`sdk/web/src/protocol.rs:71-78`, `sdk/state/src/schema.sql:62-75`). The original wallet seed does not cross.
3. **Storage worker → OPFS SQLite.** One worker opens SQLite through the OPFS VFS and serializes access; lock errors are surfaced rather than opening competing owners (`sdk/web/src/workers/storage.rs:47-52`, `sdk/web/src/workers/storage.rs:107-162`). The same state crate/schema used natively performs event projections (`app/ARCHITECTURE.md:7-23`). Deleting browser site storage deletes this durable local view; the README warns that this can make funds inaccessible (`README.md:119-124`).
4. **Main wasm → prover worker.** Prepared transaction parameters—including the note private key and note witness data—cross to the prover worker; public JS-facing key summaries omit private keys (`sdk/web/src/protocol.rs:39-44`, `sdk/web/src/protocol.rs:139-148`, `sdk/client/src/transact.rs:172-185`). The worker returns proof bytes, public transaction values, and encrypted outputs, with a 30-second request timeout (`sdk/web/src/workers/prover.rs:627-692`).
5. **wasm SDK → RPC/bootnode.** The portable client uses the Stellar RPC indexer for recent events and can use a bootnode for archived pages; handoff is an explicit JSON-RPC error described below (`sdk/client/src/sync.rs:311-449`, `docs/src/bootnode.md:13-35`). RPC receives signed Stellar envelopes and public contract arguments, never plaintext notes (`sdk/stellar/src/submit.rs:8-54`, `sdk/stellar/src/soroban_encode.rs:35-108`).
6. **RPC → contracts.** Soroban invokes the registry/pool/verifier/ASP contracts. The pool sees sender, roots, nullifiers, commitments, public amount, proof, and `ExtData`; it does not receive private note keys, blindings, or Merkle paths (`contracts/pool/src/pool.rs:400-460`, `contracts/pool/src/pool.rs:513-645`).

Correlation IDs exist because worker and network operations are asynchronous and responses/log spans must be associated with the originating action; browser context is explicitly stacked and restored around futures (`sdk/types/src/correlation.rs:32-81`, `sdk/web/src/protocol.rs:1-9`). This is observability/request-routing metadata, not a protocol identifier.

At the worker boundary, the repository defines Serde-serializable Rust enums and delegates their transport serialization to `gloo-worker` (`sdk/web/src/protocol.rs:1-9`, `sdk/web/src/protocol.rs:60-164`, `sdk/web/Cargo.toml:29-35`). I could not find a locally selected JSON/bincode/postcard codec, so I cannot honestly name a byte encoding from this repository alone; the semantic wire format is the typed enum, while the crate version’s transport codec is an external dependency detail. At the public JavaScript boundary, conversion is explicitly `serde-wasm-bindgen` (`sdk/web/src/storage.rs:96-112`).

The wallet integration is only partially standardized. The README calls SEP-0043 Draft, while the web signer still validates wallet-specific JavaScript response objects and rejects absent capabilities (`README.md:22-27`, `sdk/web/src/signer.rs:150-169`).

### B. CLI path

1. `spp` loads the selected deployment, local data directory, circuit artifacts, and a `LocalStorage`/`LocalProver` client session (`cli/src/main.rs:186-268`, `cli/src/artifacts.rs:8-68`, `cli/src/session.rs:1-80`).
2. The blocking facade calls the same async client core using its private Tokio runtime (`sdk/client/src/blocking/runtime.rs:1-22`). Unlike browser OPFS, state is a normal local SQLite file through `LocalStorage` (`sdk/client/src/storage/local.rs:24-79`).
3. Signing is delegated to the official `stellar` executable so `spp` handles an alias and signed XDR, not a mnemonic/private seed (`cli/src/stellar_cli.rs:1-9`, `cli/src/stellar_cli.rs:81-228`). Raw secret aliases are rejected (`cli/src/stellar_cli.rs:304-319`).
4. Circuit WASM/R1CS/proving keys come from configured artifact paths with packaged/testdata fallback logic (`cli/src/artifacts.rs:8-68`). Proof generation is in-process and can use native parallelism (`sdk/witness/Cargo.toml:26-35`).
5. `sdk/stellar` builds XDR, simulates to obtain footprint/resources/fees/auth, assembles, signs, submits, and polls RPC (`sdk/stellar/src/tx_prepare.rs:22-63`, `sdk/stellar/src/tx_assemble.rs:65-144`, `sdk/stellar/src/submit.rs:8-54`).

The apparent path difference is mostly adapters: both use client/planner/prover/state/Stellar core (`sdk/client/Cargo.toml:16-30`, `sdk/web/Cargo.toml:22-24`). Real differences are wallet API, DB backend/worker ownership, browser proof isolation/no native parallel feature, and web-only disclosure support (`sdk/witness/Cargo.toml:26-35`, `sdk/client/src/prover/local.rs:61-77`).

### C. On-chain lifecycle

#### C1. Register keys

1. The account asks its signer for the fixed derivation signature, derives note/encryption public keys, and calls the registry (`sdk/client/src/account.rs:121-149`, `sdk/prover/src/encryption.rs:49-72`).
2. Registry registration requires account authorization, requires both keys to be 32 bytes, rejects replacement, stores them, and emits a public registration event (`contracts/public-key-registry/src/lib.rs:60-89`). It checks length, not curve/canonical validity.

#### C2. Plan deposit, transfer, or withdrawal

1. Planner tiers are: two-note exact, one-note exact, two-note least overshoot, one-note overshoot, then exact `k≥3` and greedy overshoot; search is capped at `TRANSACTION_LIMIT = 10` selected notes (`sdk/tx-planner/src/plan/combination.rs:6-29`, `sdk/tx-planner/src/plan/combination.rs:39-98`, `sdk/tx-planner/src/plan/combination.rs:173-211`).
2. A transaction accepts at most two inputs and produces exactly two output slots. If more than two notes are selected, the plan creates consolidation steps; `n` selected notes require `n-1` calls, with the last step changed into the requested final action (`sdk/tx-planner/src/plan/mod.rs:11-32`, `sdk/tx-planner/src/plan/mod.rs:158-198`). Deposits use dummy inputs; one-input spends pad a dummy; outputs pad zero-amount notes (`sdk/prover/src/flows.rs:552-593`).

#### C3. Build witness and proof

1. The witness contains private note amount/key/blinding, packed leaf index, Merkle siblings, output amount/public key/blinding, and required ASP proofs; roots, nullifiers, commitments, public amount, policy roots, and ext-data hash are public circuit signals (`sdk/prover/src/flows.rs:607-811`, `circuits/src/transaction.circom:52-140`).
2. The Rust builder recomputes every input commitment/signature/nullifier, every output commitment/ciphertext, enforces value conservation, and builds `ExtData` (`sdk/prover/src/flows.rs:516-545`, `sdk/prover/src/flows.rs:615-711`, `sdk/prover/src/flows.rs:802-831`).
3. `WitnessCalculator` loads Circom WASM/R1CS through Wasmer and flattens JSON signal inputs to witness field elements (`sdk/witness/src/lib.rs:24-86`, `sdk/witness/src/lib.rs:89-139`). R1CS parsing checks magic/version/32-byte field size but explicitly skips the prime and assumes BN254 (`sdk/prover/src/r1cs.rs:65-167`).
4. `ProverEngine` calculates a witness, produces a compressed Arkworks Groth16 proof, extracts public inputs, verifies locally, then emits exactly 256 uncompressed proof bytes for Soroban (`sdk/client/src/prover/mod.rs:64-109`). Serialization uses 32-byte little-endian field elements (`sdk/prover/src/serialization.rs:15-59`, `sdk/prover/src/serialization.rs:110-134`).

#### C4. Assemble, sign, submit

1. `soroban_encode` converts the 256-byte proof and public values into contract maps/vectors/XDR values (`sdk/stellar/src/soroban_encode.rs:35-108`).
2. Preparation obtains account sequence, builds the invoke operation, and simulates (`sdk/stellar/src/tx_prepare.rs:22-63`). Assembly rejects bad simulation, applies footprint/resources, adds minimum resource fee, and installs auth entries (`sdk/stellar/src/tx_assemble.rs:65-144`).
3. The platform signer signs the prepared envelope; RPC `sendTransaction` returns a hash, and status polling distinguishes success/failure/pending (`sdk/stellar/src/submit.rs:8-54`).

#### C5. Contract verify and mutate

1. For positive public amount (deposit), sender authorizes and token funds move into the pool; for negative amount, funds later move to the public recipient (`contracts/pool/src/pool.rs:513-536`, `contracts/pool/src/pool.rs:604-614`).
2. Pool checks a recent root, unused/distinct nullifiers, exact ext-data hash, public amount encoding, current ASP roots, and configured Groth16 verifier (`contracts/pool/src/pool.rs:561-603`).
3. Only after verification does it mark nullifiers, transfer withdrawals, insert two commitments, and emit nullifier/commitment events (`contracts/pool/src/pool.rs:604-645`). Soroban transaction atomicity therefore prevents a successful proof from partially applying these contract mutations.

#### C6. Event ingestion and local repair

1. Indexer fetches pages, stores raw events, then stores sync progress (`sdk/stellar/src/indexer.rs:158-179`). Raw event insertions and cursor updates are separate DB transactions, so a crash can repeat a page; event uniqueness/idempotent insertion makes repetition safer than skipping (`sdk/state/src/storage.rs:85-149`).
2. Parsers convert registry/pool/ASP events into typed records; the processor applies derived tables (`sdk/state/src/events_parsers.rs:1-25`, `sdk/state/src/processor.rs:5-53`).
3. Note scanning tries both ciphertexts for each account, verifies plaintext against commitment, inserts discovered notes, and advances the per-account high-water mark in one transaction (`sdk/state/src/storage.rs:1288-1421`).
4. Submission and local projection are not atomic: the chain may succeed before local processing. The next sync re-fetches/processes events, which is the repair mechanism (`sdk/client/src/pool.rs:350-449`, `sdk/state/src/schema.sql:1-9`).

#### C7. Disclosure, separately

1. Client selects one to four owned notes and creates a context hash from domain-separated, length-delimited receipt metadata (`sdk/client/src/disclosure.rs:66-124`, `sdk/disclosure/src/lib.rs:16-69`).
2. The disclosure circuit proves knowledge of each note private key/blinding/path, commitment inclusion at a root, the amount, and the derived nullifier, while binding the external context hash (`circuits/src/selectiveDisclosure.circom:16-81`). No pool transaction or note mutation occurs.
3. Receipt verification checks expected VK/proof, context hash, and known pool roots; the client separately asks the chain whether disclosed nullifiers remain unspent (`sdk/client/src/disclosure.rs:135-185`).

### Sync and retention

`SyncMode::Inline` makes a freshness check/catch-up part of the user operation; in `Background`, ordinary reads/actions only kick the long-running loop and return rather than awaiting catch-up (`sdk/client/src/sync.rs:27-52`, `sdk/client/src/sync.rs:91-105`, `sdk/client/src/sync.rs:187-296`). `Client::sync` and `Account::sync` are equivalent explicit deployment-wide `catch_up` calls using their shared RPC/storage/config/bootnode (`sdk/client/src/client.rs:80-92`, `sdk/client/src/account.rs:60-69`). Catch-up fetches raw deployment-scoped events, projects them, scans ciphertexts for accounts, and rebuilds the wallet’s spendable view; an offline wallet resumes from its saved cursor (`sdk/client/src/sync.rs:451-474`, `sdk/state/src/processor.rs:5-53`).

Recent RPC history is insufficient after roughly seven days, so the optional bootnode stores deployment-namespaced pages in PostgreSQL (`README.md:119-124`, `tools/bootnode/README.md:1-18`, `tools/bootnode/src/storage/postgres.rs:149-204`, `tools/bootnode/src/storage/postgres.rs:333-373`). When the archive reaches the handoff point it returns JSON-RPC error `-32002`, telling the client to continue against its own RPC (`docs/src/bootnode.md:13-35`, `sdk/client/src/sync.rs:311-449`). A user with no bootnode can sync recent events and use already indexed notes, but cannot reconstruct older encrypted note history after losing local state. **That last consequence is an inference from retention plus random blindings; verify exact provider retention and recovery UX before presenting a numeric guarantee.**

The processor says broken events “must not be deleted” (`sdk/state/src/processor.rs:18-20`). Unprocessed selection is based on absence from derived event tables, so retaining the raw row lets a later parser fix retry it; deleting or cursor-skipping it would make the state gap permanent (`sdk/state/src/storage.rs:1134-1169`).

## 5. How correctness is established

| Test family | Unique thing it catches | What it does not establish |
|---|---|---|
| `e2e_pool_2_in_2_out` | Generates real proofs and runs the 2-in/2-out contract path, catching public-input order, VK, proof encoding, pool state, and token-flow integration mismatches (`e2e-tests/src/tests/e2e_pool_2_in_2_out.rs:1-37`). | Browser/real RPC/wallet behavior and production key trust. |
| `e2e_pool_2tx_plan` | Exercises planner consolidation followed by final spend, so it catches intermediate-note resolution and multi-call state transitions absent from a single transaction (`e2e-tests/src/tests/e2e_pool_2tx_plan.rs:342-440`). | Plans beyond covered fixtures, concurrency, crash recovery. |
| `coherence/` | Currently checks only that SDK policy-bit encoding and `requires_*` semantics match the pool policy module (`e2e-tests/src/tests/coherence/policy.rs:1-35`). | Hash/root/proof equivalence, a shared primitive bug, or semantic privacy claims. |
| `witness_identity` | Requires optimized browser witness WASM and release artifact WASM to produce byte-identical witnesses across policy and disclosure variants (`e2e-tests/src/tests/witness_identity.rs:141-180`, `e2e-tests/src/tests/witness_identity.rs:495-613`). | Proof correctness if both artifacts share wrong circuit logic. |
| `sdk/tests/plan.rs` | Public-SDK planning/step behavior and error mapping (`sdk/tests/src/tests/plan.rs:1-40`). | Contract/proof execution. |
| `sdk/tests/wallet.rs` | Account/wallet state flows against SQLite/client fixtures (`sdk/tests/src/tests/wallet.rs:1-40`). | Browser OPFS and external wallet compatibility. |
| `sdk/tests/telemetry.rs` | Correlation/logging and secret-redaction expectations (`sdk/tests/src/tests/telemetry.rs:1-40`). | Cryptographic privacy against chain observers. |
| Pool contract unit tests | Exercise contract errors, roots, token movement, and events with a dummy verifier/proof (`contracts/pool/src/test.rs:48-52`). | Real Groth16 verification; E2E supplies that missing oracle. |
| Circuit tests | Compare compiled-circuit witnesses/constraints with Rust test utilities (`circuits/src/test/utils/keypair.rs:1-38`, `circuits/src/test/utils/transaction.rs:1-40`). | Fully independent primitive validation because both use workspace `zkhash` (`circuits/Cargo.toml:24-48`). |

There are no tracked snapshot files under `e2e-tests/test_snapshots/` in this commit, and pool tests explicitly disable Soroban host snapshot-file writing (`contracts/pool/src/test.rs:258-266`). Therefore nothing is currently pinned there and a “snapshot diff” has no current meaning. This is a source-tree absence, not a claim from documentation.

`testdata` key triplets represent one circuit in four consumers: proving key for Arkworks, JSON VK as interchange input, Soroban binary VK embedded by verifier builds, and generated Rust constants (`deployments/testnet/circuit_keys/README.md:34-61`). `circuit-keys` shares conversions; `ceremony-cli` can manipulate/contribute parameters (`circuit-keys/src/lib.rs:1-31`, `tools/ceremony-cli/Cargo.toml:13-27`). Current shipped testnet keys are directly documented as locally generated development keys with no trusted setup ceremony (`deployments/testnet/circuit_keys/README.md:21-32`).

Not covered by current tests, as far as I could find:

- live testnet liveness/code hash/config provenance; local manifests are not live proof (`deployments/testnet/deployments.json:1-40`);
- malicious/omitting bootnode behavior, archive completeness, or long-offline recovery; docs explicitly make the bootnode untrusted (`docs/src/bootnode.md:37-60`);
- full browser wallet + OPFS deletion/recovery + real worker lifecycle;
- constructor misbinding of policy flags to a same-shape weaker verifier (`contracts/pool/src/pool.rs:224-280`);
- crash fault injection between chain confirmation, raw-event storage, projection, and note scan;
- discovery scaling across many commitments/accounts (`sdk/state/src/storage.rs:1221-1424`);
- cryptographic binding between an output commitment’s plaintext and its encrypted `ExtData` ciphertext (`circuits/src/transaction.circom:102-140`);
- end-to-end GVK, because the docs identify it as follow-up work (`docs/src/global_view_key.md:3-18`);
- a real multi-party trusted setup for shipped keys (`deployments/testnet/circuit_keys/README.md:21-32`);
- native selective-disclosure proving (`sdk/client/src/prover/local.rs:61-77`).

Absence claims are based on this commit’s test inventory; maintainers should confirm external/private CI before treating them as definitive.

## 6. Rust and Soroban engineering decisions

### Handles, async, and target constraints

`Prover` and `Signer` are trait objects stored through `Handle`, which is `Arc` natively and `Rc` on wasm (`sdk/client/src/handle.rs:1-45`, `sdk/client/src/prover/mod.rs:112-130`, `sdk/client/src/signer/mod.rs:11-29`). This lets the same client swap an in-process native prover/signer for worker/JS implementations without making every public client type generic. Storage is different: `Client<S>` remains generic over `Storage`; it is not a `Handle<dyn Storage>` (`sdk/client/src/storage/mod.rs:113-190`, `sdk/client/src/lib.rs:45-88`). **Inference:** this preserves static storage typing while runtime-polymorphic peripherals vary more often; verify that design intent with maintainers.

Traits use `async_trait(?Send)`, and wasm `Handle` is `Rc`, so futures need not be `Send` (`sdk/client/src/prover/mod.rs:112-130`, `sdk/client/src/handle.rs:1-8`). This accommodates browser-local wasm futures. The cost is that even native callers cannot freely spawn those trait futures onto arbitrary multithreaded executors without an owning wrapper. The blocking CLI creates its own multithread Tokio runtime and panics if invoked inside an existing Tokio runtime (`sdk/client/src/blocking/runtime.rs:1-22`).

### Load-bearing borrows/lifetimes

`DeriveNoteFn<'a>` is a borrowed `FnMut` callback, so note scanning can borrow account/decryption context for only the storage operation rather than require `'static` ownership (`sdk/state/src/storage.rs:63-64`). Rows and account keys are passed by reference through the scan, avoiding copies of secret material while SQLite transactions bound mutation lifetime (`sdk/state/src/storage.rs:1288-1421`). If this were `'static`, browser/native callers would need owned/cloned state or global storage; if the DB transaction/borrow escaped, Rust would reject use after connection/transaction end.

`LocalStorage` wraps its connection in `RefCell` and returns `Ref`/`RefMut` guards (`sdk/client/src/storage/local.rs:24-47`). That moves exclusive-borrow checking to runtime so trait methods can take `&self`; overlapping mutable/reentrant borrows panic instead of becoming a data race. Worker isolation avoids sharing this non-thread-safe connection across browser tasks (`sdk/web/src/workers/storage.rs:107-162`).

Witness builders borrow input slices while constructing owned `CircuitInputs`, but worker-bound `TransactParams` own keys, vectors, and strings (`sdk/prover/src/flows.rs:112-153`, `sdk/web/src/protocol.rs:139-148`). The ownership is load-bearing: an async worker message can outlive the JS/Rust stack frame that requested it.

### FFI and type-safety boundaries

- `wasm-bindgen`/`serde-wasm-bindgen` is a dynamic JS boundary; Rust validates missing/wrong wallet return fields at runtime (`sdk/web/src/signer.rs:50-82`, `sdk/web/src/signer.rs:150-169`). Rust types begin only after conversion succeeds.
- Witness calculation loads compiler-generated WASM in Wasmer and accepts JSON/R1CS bytes; magic/version/field-size and witness lengths are checked, but the R1CS prime is assumed BN254 rather than verified (`sdk/witness/src/lib.rs:24-86`, `sdk/prover/src/r1cs.rs:65-167`, `sdk/prover/src/serialization.rs:110-134`).
- Contract calls cross Stellar XDR/Soroban host types. Proof size/type conversions are checked before BN254 host operations, and verifier errors are explicit (`contracts/types/src/lib.rs:8-19`, `contracts/circom-groth16-verifier/src/lib.rs:78-140`). Rust cannot guarantee that caller-provided bytes encode a valid proof; the host/verifier establishes that at runtime.

Workspace lints deny unsafe code in project crates (`Cargo.toml:110-113`). The unsafe-adjacent surface is therefore dependency/FFI code (Wasmer, SQLite, browser/Soroban hosts), not explicit repository `unsafe` blocks.

Every contract is `#![no_std]` and uses Soroban SDK host-backed types/storage (`contracts/pool/src/lib.rs:1-8`, `contracts/public-key-registry/src/lib.rs:1-8`). That rules out OS files, sockets, threads, normal `std` runtime services, and native database/prover dependencies inside contracts; those responsibilities must remain client-side.

### Errors, state, and invariants

Errors are separated by recovery/ABI layer: client errors include orchestration and partial-plan completion (`sdk/client/src/error.rs:6-80`); planner errors describe pure selection/invariant failures (`sdk/tx-planner/src/plan/error.rs:1-45`); execution errors represent state-machine/argument failures (`sdk/tx-planner/src/execute/error.rs:1-16`); pool errors are stable Soroban numeric ABI variants (`contracts/pool/src/pool.rs:28-61`); proof-point conversion has contract-type errors (`contracts/types/src/lib.rs:8-19`). One enum would couple pure offline planning, retryable RPC/client failures, and irreversible public contract ABI.

Native and browser use the same transactional schema, but persistence cannot be atomic with chain submission (`app/ARCHITECTURE.md:7-23`, `sdk/stellar/src/indexer.rs:158-179`). Chain state can temporarily lead local state after a crash; idempotent raw-event ingestion and later sync repair it (`sdk/state/src/storage.rs:85-149`, `sdk/state/src/processor.rs:5-53`). Per-account scan inserts and high-water updates are transactional, preventing a crash from advancing beyond an undiscovered note (`sdk/state/src/storage.rs:1288-1421`).

Newtypes help but do not make every invariant unrepresentable. `NoteAmount` is unsigned `u128` while `ExtAmount` models signed public flow (`sdk/types/src/amounts.rs:51-89`, `sdk/types/src/amounts.rs:182-218`). Field constructors validate values below the BN254 modulus, but `Field(pub U256)` exposes its inner field, so callers can bypass constructors (`sdk/types/src/amounts.rs:320-414`). Key wrappers fix 32-byte length and private-key wrappers zeroize/redact debug output, but they do not prove curve membership/canonicality (`sdk/types/src/lib.rs:239-286`). The registry likewise checks only length (`contracts/public-key-registry/src/lib.rs:60-89`). Say “type-guided and validated on normal constructors,” not “all invalid states are impossible.”

## 7. Where Stellar/Soroban fought the design

### On-chain proof resources

The verifier deliberately uses Soroban’s native BN254 operations, including multi-scalar multiplication and pairing, rather than implementing curve arithmetic in contract WASM (`contracts/circom-groth16-verifier/src/lib.rs:3-13`, `contracts/circom-groth16-verifier/src/lib.rs:99-139`). The client must simulate every invocation to obtain footprint, resource limits, and `minResourceFee`, then assemble those values into the transaction (`sdk/stellar/src/tx_prepare.rs:22-63`, `sdk/stellar/src/tx_assemble.rs:65-144`).

I could not find a source-backed fixed CPU instruction limit, memory limit, budget assertion, benchmark, or stable “one transact costs X” figure in this commit. The cost is network/configuration dependent and simulation-derived in the implementation. Do not quote a number without a fresh RPC simulation. What would help: checked-in budget regression tests and reproducible per-circuit simulation reports. That recommendation is an inference from the absence of such evidence.

### Event retention and archival state

The roughly seven-day RPC event window creates the bootnode subsystem: PostgreSQL page caching, deployment namespacing, trust documentation, and explicit handoff (`README.md:119-124`, `tools/bootnode/README.md:1-18`, `docs/src/bootnode.md:13-60`). This is not a peer-to-peer bootnode in the networking sense; it is an untrusted event archive/cache. What would help is a durable, standardized archival event API with completeness proofs or checkpoints, so recovery does not depend on a project-operated cache. This is a recommendation inferred from the current trust/retention design.

### Browser proving

Browser proving requires wasm-specific non-parallel Ark-Circom/Wasmer JS features, separate workers, artifact optimization, and a timeout (`sdk/witness/Cargo.toml:26-35`, `sdk/web/src/workers/prover.rs:627-692`, `sdk/web/scripts/build.sh:114-162`). The workspace even pins a no-parallel Ark-Circom branch (`Cargo.toml:31-38`). I found no checked-in source benchmark for proof time or source-backed byte-size table, so do not claim either. What would help is browser-safe parallel witness/proof execution, streaming/cacheable artifact distribution, and measured performance budgets; this is an inference from the worker/build structure.

### Wallet integration and browser storage

SEP-0043 is marked Draft and the adapter still performs wallet-specific JavaScript shape/capability checks (`README.md:22-27`, `sdk/web/src/signer.rs:150-169`). OPFS supplies SQLite but clearing site data destroys it, while random note blindings require either that DB or replayable ciphertext history (`README.md:119-124`, `sdk/prover/src/encryption.rs:202-230`). What would help is a stable wallet authorization/signing interface plus a portable encrypted state export/checkpoint format. This is an inference; the repository does not specify such a standard.

### Deployment/build workarounds

Deployment scripts select one verifier artifact per policy circuit and wire its address into a pool, reflecting Groth16’s circuit-specific VK and static public-input layout (`deployments/scripts/deploy.sh:139-175`, `deployments/scripts/deploy.sh:523-562`). Web CI builds/optimizes multiple circuit artifacts and checks witness identity because browser optimization is a separate correctness boundary (`.github/workflows/wasm-build.yml:38-71`, `e2e-tests/src/tests/witness_identity.rs:141-180`). A universal or updatable verifier interface could reduce deployment combinatorics, but the repository gives no documented reason for choosing Groth16/Circom over a universal setup; treat that as a design question, not an established criticism.

## 8. Security and privacy, without marketing

### What the chain reveals

| Action | Public observation |
|---|---|
| Deposit | Positive `ext_amount`, authenticated sender/transaction context, pool recipient, two new commitments/ciphertexts and their indices/timing are public (`contracts/pool/src/pool.rs:513-536`, `contracts/pool/src/pool.rs:630-643`, `docs/src/privacy-tradeoffs.md:7-12`). The internal output split is encrypted, but total deposited amount is not. |
| Private transfer | `ext_amount = 0`, caller/transaction timing, two nullifiers, and two new commitments/ciphertexts are co-located in one invocation (`contracts/pool/src/pool.rs:561-645`, `docs/src/privacy-tradeoffs.md:7-12`). The chain does not learn the input-commitment link or output amounts from the proof, but it sees the transaction bundle and timing. |
| Withdrawal | Negative `ext_amount`/absolute public withdrawal value, public recipient, timing, two nullifiers and two new commitments are visible (`sdk/prover/src/flows.rs:189-227`, `contracts/pool/src/pool.rs:604-643`). The docs explicitly warn that public addresses correlate with new commitments (`docs/src/privacy-tradeoffs.md:9-12`). |

The registry voluntarily publishes a Stellar address alongside note and encryption public keys (`docs/src/privacy-tradeoffs.md:13-18`, `contracts/public-key-registry/src/lib.rs:21-37`). The blocklist publishes unblinded note public keys, while membership publishes blinded leaves (`docs/src/privacy-tradeoffs.md:22-29`).

The practical anonymity set is the set of plausible unspent commitments consistent with the observer’s policy knowledge and side information. **This is an inference** from the contract/event surface, not a metric computed by the repository. Deposits/withdrawals with public amounts and addresses, synchronous nullifier/commitment events, timing, registry links, policy-set membership/blocklist information, and user behavior shrink it (`docs/src/privacy-tradeoffs.md:7-29`).

### ASP trust

The membership administrator can change the admin, toggle who may insert leaves, and—by default—authorize additions to the allowlist tree (`contracts/asp-membership/src/lib.rs:112-143`, `contracts/asp-membership/src/lib.rs:180-201`). Turning “Admin-Only Leaf Insert” off means any caller can add a correctly formed leaf; then a user can self-admit and the allowlist no longer represents ASP approval (`README.md:59-62`, `contracts/asp-membership/src/lib.rs:128-143`). It does not break Merkle correctness; it breaks the policy meaning.

The non-membership admin alone can insert/delete blocked keys (`contracts/asp-non-membership/src/lib.rs:361-364`, `contracts/asp-non-membership/src/lib.rs:516-519`). ASP admins can include/exclude keys and cause censorship/denial under the selected policy, but the contracts do not receive note private keys, amounts, or blindings (`circuits/src/aspMembership.circom:19-47`, `circuits/src/aspNonMembership.circom:1-53`). They cannot spend a note merely by controlling a root.

### GVK trust and visibility

If integrated, the holder of private scalar corresponding to `D` can decrypt GVK ciphertexts: traceable reveals public key+amount+blinding; view-only reveals amount+blinding (`sdk/types/src/gvk.rs:28-51`, `circuits/src/globalViewKey.circom:214-284`). The circuit proves ciphertext consistency with note fields (`circuits/src/globalViewKey.circom:95-171`). Today there is no pool ABI/configuration that binds `D`, publishes these ciphertexts, or lets a user inspect deployed GVK mode (`contracts/pool/src/pool.rs:78-120`, `docs/src/global_view_key.md:3-18`). Honest answer: **not yet**; contract binding/event/tooling would close it.

### Selective disclosure

A receipt’s Groth16 proof establishes knowledge of note secret/blinding/path for stated commitments, amounts, nullifiers, roots, and context hash (`circuits/src/selectiveDisclosure.circom:16-81`, `sdk/types/src/disclosure.rs:46-84`). It does not establish the real-world identity or truthfulness of the human-readable authority/purpose; those strings are assertions whose exact bytes are merely bound by the context hash (`sdk/disclosure/src/lib.rs:16-69`). The expected verifying-key hash must also come from an independently trusted source (`sdk/client/src/disclosure.rs:135-185`).

`is_cryptographically_valid` combines proof, context, and known-root checks; `is_valid_and_unspent` additionally requires chain nullifier checks (`sdk/types/src/disclosure.rs:242-280`, `sdk/client/src/disclosure.rs:135-185`). They differ because ownership/existence at a historical root is a Groth16 fact, while present unspentness is mutable chain state. The docs’ “three checks” section omits the current separate unspent check, so current Rust is authoritative (`docs/src/disclosure.md:160-175`).

### Compromised secrets

- Wallet derivation signature: derives note private key, X25519 private key, and network membership blinding, so an attacker can decrypt recoverable history and obtain spend authority once note data/paths are available (`sdk/prover/src/encryption.rs:57-98`, `sdk/prover/src/encryption.rs:100-179`). This is why collapsing two signatures concentrates risk (`sdk/prover/src/encryption.rs:16-31`).
- X25519 private key only: decrypts addressed ciphertexts to amount/blinding, but cannot compute the note-key signature/nullifier required to spend (`sdk/prover/src/encryption.rs:351-406`, `sdk/prover/src/crypto.rs:110-139`).
- Note private key only: derives note public key/signatures/nullifiers, but a spend witness also needs amount, blinding, leaf index, and Merkle siblings (`circuits/src/transaction.circom:52-100`). It is spend authority combined with recoverable note state, not a self-contained wallet backup.

Assertions that remain outside cryptographic proof include deployment-to-VK policy correctness, archive completeness/honesty, wallet identity behind a registered address, human meaning of disclosure context, ciphertext/plaintext consistency for ordinary note encryption, and current deployment liveness (`contracts/pool/src/pool.rs:224-280`, `docs/src/bootnode.md:37-60`, `circuits/src/transaction.circom:102-140`). These are “not yet” or operational trust assumptions, not privacy guarantees.

The documented logging model classifies keys, seeds, signatures, witnesses, and membership blindings as never-log secrets, while addresses, amounts, commitments, and nullifiers are redacted by default (`docs/src/security.md:24-30`). It warns that debug configuration can reveal the second class and that such logs must not be shared (`docs/src/security.md:32-33`). The telemetry test verifies a sample `Sensitive<NoteAmount>` redaction, not every call site or every Tier-0 secret (`sdk/tests/src/tests/telemetry.rs:34-64`); the global “never logged” rule is therefore a policy plus partial regression coverage, not a proof of absence.

## 9. Hostile Q&A

1. **[WEAKNESS] Why Circom/Groth16 and per-circuit trusted setup instead of a universal setup?** The repository does not document that choice, so I cannot defend its original rationale. What is verifiable is that each verifier embeds one VK and deployments select per-policy artifacts; current keys are local development keys (`contracts/circom-groth16-verifier/src/lib.rs:25-26`, `deployments/scripts/deploy.sh:139-175`, `deployments/testnet/circuit_keys/README.md:21-32`).
2. **[WEAKNESS] What is actually deployed on testnet versus mocked or test-keyed?** The manifest names two testnet deployments and contract IDs, but I did not verify them live (`deployments/testnet/deployments.json:1-40`). Their keys are explicitly local/non-ceremonial, and contract unit tests often use a dummy verifier while E2E uses real proofs (`deployments/testnet/circuit_keys/README.md:21-32`, `contracts/pool/src/test.rs:48-52`).
3. **How do you know Rust hashes match Circom and Soroban?** There are three runtime-language implementations; artifact-dependent circuit tests compare Circom with Rust utilities, and real-proof E2E compares the circuit root to the pool root (`sdk/prover/src/crypto.rs:33-163`, `circuits/src/test/prove_poseidon2.rs:85-142`, `e2e-tests/src/tests/e2e_pool_2_in_2_out.rs:181-208`). Some Rust utilities share the same `zkhash` dependency and SDK dense-Merkle code reuses `circuits::core`, so this is regression evidence, not fully independent or formal equivalence (`circuits/Cargo.toml:24-48`, `sdk/prover/src/merkle.rs:1-24`).
4. **[WEAKNESS] What happens if the bootnode disappears?** Already indexed notes remain local, and recent RPC events remain available, but a fresh/recovered client cannot replay history older than provider retention (`README.md:119-124`, `docs/src/bootnode.md:1-35`). Funds remain on chain; operational recovery can fail because random blindings live in ciphertext history (`sdk/prover/src/encryption.rs:202-230`).
5. **Why 2-in/2-out?** The circuit and contract are fixed at two inputs/two outputs, padding unused slots and inserting exactly two leaves (`sdk/prover/src/flows.rs:552-593`, `contracts/pool/src/merkle_with_history.rs:101-190`). The repository does not state the original rationale; the visible cost is consolidation calls when more than two notes fund a payment (`sdk/tx-planner/src/plan/mod.rs:158-198`).
6. **Who can deanonymize a user today, and at what cost?** No source-backed deanonymization cost is measured. Observers can correlate public amounts, addresses, timing, registry links, and synchronous events; ASP blocklists expose keys (`docs/src/privacy-tradeoffs.md:7-29`). GVK deanonymization is circuit work, not currently wired into the pool (`docs/src/global_view_key.md:3-18`).
7. **[WEAKNESS] What must change before real money?** At minimum: audit/hardening, ceremony-derived production keys, durable recovery/archive guarantees, live deployment proof, browser recovery testing, and closure of documented/uncovered seams (`README.md:13-14`, `deployments/testnet/circuit_keys/README.md:21-32`). The project itself says not to use real assets (`README.md:13-14`).
8. **What did you actually build?** Only name commits/features you authored; repository ancestry shows substantial inherited foundations (`circuits/src/transaction.circom:1-3`, `circuits/src/smt/smtverifier.circom:1-3`, `poseidon2/src/lib.rs:1-9`). I cannot infer personal authorship from the working tree.
9. **Can a client choose `none` against an allowlist pool?** Not in the normal path: pool flags/verifier are fixed at construction and current ASP roots are checked before verification (`contracts/pool/src/pool.rs:224-280`, `contracts/pool/src/pool.rs:585-603`). **Weak configuration seam:** the constructor does not itself attest that verifier bytecode/VK matches flags.
10. **[WEAKNESS] Does encrypted output provably match the created note?** No ordinary transaction circuit constraint links ciphertext plaintext to output amount/key/blinding; it binds only the ext-data hash (`circuits/src/transaction.circom:102-140`). Honest clients construct both together, but malicious garbage ciphertext can make a valid note undiscoverable (`sdk/prover/src/flows.rs:656-711`).
11. **Why is the zero leaf nonzero?** It is a domain-separated hash of `XLM` and zero, shared by client/contract, and contract root validation rejects literal zero (`sdk/prover/src/crypto.rs:20-31`, `contracts/soroban-utils/src/poseidon2.rs:112-130`, `contracts/pool/src/merkle_with_history.rs:209-213`). It prevents empty/sentinel state from being confused with the field zero.
12. **How stale can a generated proof be?** The contract recognizes a ring of 90 roots, meaning 90 tree updates/transactions, not 90 ledgers or minutes (`contracts/pool/src/merkle_with_history.rs:1-18`, `contracts/pool/src/merkle_with_history.rs:193-236`). Beyond that, it rejects `UnknownRoot` and the client must resync/reprove.
13. **Can the ASP steal funds?** Root control can admit, block, or censor users, but spend constraints still require note secrets/path (`circuits/src/transaction.circom:52-100`). Disabling admin-only membership lets anyone self-admit and destroys allowlist meaning (`contracts/asp-membership/src/lib.rs:128-201`).
14. **[WEAKNESS] Is wallet-seed recovery sufficient?** It recovers deterministic keys, not random note blindings; ciphertext history is additionally required (`sdk/prover/src/encryption.rs:57-98`, `sdk/prover/src/encryption.rs:202-230`). The README’s storage warning is therefore substantive (`README.md:119-124`).
15. **Why one wallet signature for both key families?** The module says it reduced two prompts to one and explicitly accepted correlated compromise (`sdk/prover/src/encryption.rs:16-31`). SHA-256 domains still separate derived byte strings (`sdk/prover/src/encryption.rs:49-54`, `sdk/prover/src/encryption.rs:181-199`).
16. **[WEAKNESS] Is GVK available to auditors now?** No. Circuit/type definitions exist, but contract binding, events, and wallet tooling are listed as follow-ups (`docs/src/global_view_key.md:3-18`, `contracts/pool/src/pool.rs:78-120`).
17. **Does disclosure prove the authority statement is true?** No; it proves note ownership/inclusion and binds exact context bytes, while authority/purpose semantics are asserted (`circuits/src/selectiveDisclosure.circom:16-81`, `sdk/disclosure/src/lib.rs:16-69`). Current unspentness is a separate RPC/contract-state check (`sdk/client/src/disclosure.rs:135-185`).
18. **What if browser storage is cleared immediately after submission?** Chain success is unaffected, but local projection and random note data may be lost until event replay reconstructs them (`sdk/client/src/pool.rs:350-449`, `sdk/prover/src/encryption.rs:202-230`). Recovery past RPC retention needs an archive/bootnode (`README.md:119-124`).
19. **[WEAKNESS] What does one transact cost?** The repository supplies no stable number or budget benchmark. The client simulates and uses returned resource/fee values, so quote only a fresh measured deployment result (`sdk/stellar/src/tx_prepare.rs:22-63`, `sdk/stellar/src/tx_assemble.rs:65-144`).
20. **Are the docs trustworthy when they disagree with code?** They are useful design context, but the compiler obeys source: current key constants are v1 despite a v2 diagram, GVK source includes salt absent from docs, and current disclosure has a fourth unspent check (`sdk/prover/src/encryption.rs:16-25`, `sdk/prover/src/encryption.rs:49-54`, `circuits/src/globalViewKey.circom:62-93`, `docs/src/disclosure.md:160-175`). Present the disagreement explicitly.

## 10. Explainer layer

### Glossary

| Term | Plain meaning in this repository |
|---|---|
| Note / UTXO | A private spendable record: amount, recipient note key, random blinding, commitment, and position/path (`sdk/prover/src/notes.rs:24-83`). |
| Commitment | `H(amount, note public key, blinding; domain 1)`: public tree leaf hiding those fields (`sdk/prover/src/crypto.rs:97-108`). |
| Nullifier | One-time public spend marker derived from commitment, leaf index, and secret-key-derived signature (`sdk/prover/src/crypto.rs:110-139`). |
| Blinding | Random field element that hides otherwise guessable commitment plaintext; it is recovered from encrypted events (`sdk/prover/src/encryption.rs:202-230`). |
| Merkle root history | Ring of the 90 most recent note-tree roots accepted for proofs (`contracts/pool/src/merkle_with_history.rs:1-18`, `contracts/pool/src/merkle_with_history.rs:193-236`). |
| ASP | Association Set Provider: administrator/state whose membership or blocklist root is constrained by the policy circuit (`README.md:16-18`, `contracts/pool/src/pool.rs:585-603`). |
| Membership / non-membership | Proof that a blinded key leaf is in an allowlist tree / note public key is absent from a blocklist sparse tree (`circuits/src/aspMembership.circom:19-47`, `circuits/src/aspNonMembership.circom:1-53`). |
| Sparse Merkle tree | Key-directed tree that stores only occupied paths and supports absence proofs (`contracts/asp-non-membership/src/lib.rs:1-23`). |
| Policy flags | Two bits selecting none, allowlist, blocklist, or both and therefore a static circuit/VK (`sdk/types/src/policy_tx.rs:19-64`). |
| Global view key | Proposed BabyJubJub auditor key/ciphertext constrained inside GVK circuit variants; not yet pool-integrated (`docs/src/global_view_key.md:3-18`). |
| Traceable / view-only | GVK modes revealing key+amount+blinding / amount+blinding to the GVK holder (`sdk/types/src/gvk.rs:28-51`). |
| Selective disclosure | Off-chain Groth16 receipt proving ownership/inclusion/amounts for chosen notes and binding a context (`circuits/src/selectiveDisclosure.circom:16-81`). |
| Ext data | Public recipient, signed public amount, and two encrypted outputs whose XDR hash is a proof input (`sdk/stellar/src/ext_data_hash.rs:10-64`). |
| Shielding | Deposit: public tokens enter pool while private output commitments are created (`contracts/pool/src/pool.rs:513-536`). |
| Bootnode | Untrusted PostgreSQL-backed event archive that fills the RPC retention gap, then hands off to normal RPC (`docs/src/bootnode.md:1-60`). |
| Deployment | One configured set of network/asset/pool/verifier/ASP/registry addresses; manifest presence is not live proof (`deployments/testnet/deployments.json:1-40`). |

### Process paths

```text
Browser                                      CLI
JS app                                       spp command
  | JS values; wallet stays in extension       | alias/config/artifact paths
  v                                             v
sdk/web wasm handles                         blocking sdk/client
  | CorrelatedRequest                           | block_on same async core
  +--> storage worker --> OPFS SQLite            +--> local SQLite
  |    signature in; derived keys/state          |
  +--> prover worker (private witness)            +--> in-process prover
  |                                               +--> external stellar CLI signs
  +--------------------+--------------------------+
                       v
                sdk/stellar RPC/XDR
                       |
              recent RPC / archive bootnode
                       v
        registry + pool + ASP + verifier contracts
```

The boundary facts are implemented in `sdk/web/src/protocol.rs:60-164`, `sdk/web/src/workers/storage.rs:107-162`, `cli/src/stellar_cli.rs:1-9`, and `sdk/stellar/src/tx_prepare.rs:22-63`.

```text
spendable-note rows
      |
      v
planner: choose <=10 notes --> n-1 two-input steps
      |
      v
state: roots + paths + ASP proofs
      |
      v
flows.rs: private witness + ciphertext + public inputs
      |
      v
Circom WASM witness --> Groth16 proof --> local verification
      |
      v
XDR encode --> simulate/resources --> wallet sign --> submit
      |
      v
pool: root/nullifier/ext/policy/proof checks
      |
      +--> token transfer / mark nullifiers / insert 2 commitments
      v
events --> raw SQLite --> typed projection --> trial decrypt --> spendable notes
```

The sequence is anchored in `sdk/tx-planner/src/plan/mod.rs:110-199`, `sdk/prover/src/flows.rs:607-831`, `sdk/client/src/prover/mod.rs:64-109`, `contracts/pool/src/pool.rs:561-645`, and `sdk/state/src/storage.rs:1221-1424`.

### Five-minute spoken version

“This is an unaudited reference implementation of private payments on Stellar, not a production mixer and not a finished compliance product. A user’s wallet signs one fixed message. The client hashes that signature into two separate key families: a BN254 note key for proving ownership and an X25519 key for decrypting incoming notes. Individual note blindings remain random, so recovery needs encrypted event history as well as the wallet seed (`README.md:13-14`, `sdk/prover/src/encryption.rs:49-72`, `sdk/prover/src/encryption.rs:202-230`).

“The private state is a two-input, two-output note model. A commitment hides amount, note public key, and blinding. Spending proves a recent Merkle inclusion and secret-key ownership, then publishes a nullifier so the same note cannot be spent again. Deposits and withdrawals still expose the public amount and address, and all calls expose timing plus the bundle of nullifiers and new commitments (`sdk/prover/src/crypto.rs:97-139`, `circuits/src/transaction.circom:52-140`, `docs/src/privacy-tradeoffs.md:7-12`).

“Compliance policy is circuit-selected. A pool can require a blinded allowlist membership proof, a public-key blocklist non-membership proof, both, or neither. The pool fixes a verifier at deployment and checks current ASP roots before calling it. The important operational caveat is that deployment tooling—not the constructor itself—ensures that the verifier’s embedded key matches the declared policy (`sdk/types/src/policy_tx.rs:19-64`, `contracts/pool/src/pool.rs:224-280`, `contracts/pool/src/pool.rs:585-603`).

“Most work happens off chain. Native CLI and browser share the Rust client, planner, state, Stellar, witness, and prover crates. The browser adds OPFS SQLite and separate storage/prover workers; the CLI uses a local DB and asks the external Stellar CLI to sign. The chain only receives public inputs, proof, and encrypted output payloads (`sdk/client/Cargo.toml:16-30`, `sdk/web/Cargo.toml:22-44`, `cli/src/stellar_cli.rs:1-9`).

“Correctness crosses three languages: Rust constructs hashes and witnesses, Circom constrains them, and Soroban repeats public hashing/tree logic and verifies Groth16. Artifact-dependent circuit tests and real-proof E2E detect drift, but shared dependencies mean they are not a formal independent equivalence proof (`circuits/src/test/prove_poseidon2.rs:85-142`, `e2e-tests/src/tests/e2e_pool_2_in_2_out.rs:181-208`, `sdk/prover/src/merkle.rs:1-24`).

“There are two compliance-adjacent disclosure ideas. Selective disclosure works off chain now: it proves ownership, inclusion, amounts, and context binding, while current unspentness remains a separate chain query. Global view keys have circuit and type designs for traceable or view-only audit ciphertexts, but they are not wired into the pool contract or deployed event path yet (`circuits/src/selectiveDisclosure.circom:16-81`, `sdk/client/src/disclosure.rs:135-185`, `docs/src/global_view_key.md:3-18`).

“The largest production gaps are explicit: no audit, locally generated trusted-setup keys, fragile browser-local recovery, an event-retention dependency that created an untrusted bootnode archive, no checked-in cost/performance budget, and untested integration seams. Also, ordinary output ciphertext is hash-bound to the proof but not proven to encrypt the same plaintext as the commitment. So the honest conclusion is that this demonstrates the architecture and integration; it is not ready for real money (`README.md:13-14`, `deployments/testnet/circuit_keys/README.md:21-32`, `README.md:119-124`, `circuits/src/transaction.circom:102-140`).”
