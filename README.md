# lucid-pm

Project-management hub for **EIP-8184: LUCID — encrypted mempool**.

This repository is the shared, central reference for everyone working on LUCID.
Its job is to answer one question: *where is LUCID represented across the
Ethereum software-development lifecycle (SDLC), and what still needs to be
built?*

If you are picking up work on LUCID, start here. Each section below describes a
**need** of the project, points at **where it lives today** (if it exists), and
flags **what is still missing**.

## Status at a glance

| # | Need | Repo | Status |
|---|------|------|--------|
| 1 | EIP specification | [ethereum/EIPs](https://github.com/ethereum/EIPs) | ✅ Published (Draft) |
| 2 | EL reference implementation | [ethereum/execution-specs](https://github.com/ethereum/execution-specs) | ✅ Draft PR ([jflo#1](https://github.com/jflo/execution-specs/pull/1)) |
| 3 | EL reference tests | [ethereum/execution-specs](https://github.com/ethereum/execution-specs) | ⚠️ Partial (in PR #1) |
| 4 | Conformance fixtures | [ethereum/execution-spec-tests](https://github.com/ethereum/execution-spec-tests) | ❌ Missing |
| 5 | Execution APIs (JSON-RPC / Engine) | [ethereum/execution-apis](https://github.com/ethereum/execution-apis) | ❌ Missing |
| 6 | Consensus-layer spec | [ethereum/consensus-specs](https://github.com/ethereum/consensus-specs) | ❌ Missing |
| 7 | Client implementation | [hyperledger/besu](https://github.com/hyperledger/besu) | ⚠️ SLOTNUM only (EIP-7843) |
| 8 | Project coordination | [jflo/lucid-pm](https://github.com/jflo/lucid-pm) | ✅ This repo |

Legend: ✅ exists · ⚠️ partial · ❌ not yet present.

---

## What is LUCID?

LUCID (EIP-8184) introduces an **encrypted mempool** to Ethereum using a
commit/reveal scheme, so that transaction contents are hidden from block
producers until ordering is fixed — mitigating MEV and front-running.

The core mechanisms are:

- **Sealed-ticket transaction (type `0x05`)** — a new
  [EIP-2718](https://eips.ethereum.org/EIPS/eip-2718) transaction type carrying
  a commitment to an encrypted payload, fee parameters, and key-publication
  metadata. The next unused type after EIP-7702 (`0x04`).
- **`SLOTNUM` opcode (`0x4B`)** — exposes the commitment slot of the executing
  transaction (returns `0` for ordinary transactions). The opcode itself is
  defined by [EIP-7843](https://eips.ethereum.org/EIPS/eip-7843) and reused by
  LUCID.
- **ST commitments** — sealed-transaction commitments included in a scheduling
  block (single tickets or bundles), with an associated gas obligation.
- **Key publication** — after the commitment deadline, a key publisher releases
  the decryption-key material so sealed payloads can be revealed and executed.
- **Top-of-block (TOB) fee market** — a separate fee (`max_tob_fee`) for
  priority placement at the top of the block.

> **Status:** LUCID is **unscheduled** (no target fork). In execution-specs it
> is modelled as a standalone fork named `Lucid` with
> `Unscheduled(order_index=4)`.

---

## The needs of the project, and where to find them

LUCID is a coordinated execution-layer **and** consensus-layer change. Below is
each artifact the project requires, in roughly the order it flows through the
Ethereum SDLC.

### 1. EIP specification — ✅ published (Draft)

The normative specification document.

- **Where it lives:** <https://eips.ethereum.org/EIPS/eip-8184> · source:
  [ethereum/EIPs `EIPS/eip-8184.md`](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-8184.md)
- **Discussion:**
  <https://ethereum-magicians.org/t/eip-8184-lucid-encrypted-mempool/28017>
- **Metadata:** Standards Track · Core · **Draft** · created 2026-03-04 ·
  requires [EIP-2718](https://eips.ethereum.org/EIPS/eip-2718) and
  [EIP-7805](https://eips.ethereum.org/EIPS/eip-7805) (FOCIL inclusion lists).
- **Authors:** Anders Elowsson (@anderselowsson), Justin Florentine (@jflo),
  Julian Ma (@ma-julian).
- **Abstract:** LUCID lets encrypted transactions travel through Ethereum's
  public inclusion pipeline while remaining concealed until after scheduling
  decisions are finalized — protecting MEV-sensitive order flow, limiting
  probabilistic front-running, and broadening the censorship resistance of
  inclusion lists, while staying agnostic to any specific encryption standard.
- **Pinned in execution-specs:** the reference-spec stub
  (`tests/unscheduled/eip8184_lucid/spec.py`) pins commit
  `e2c08597b73aa471687ddf0f443cbf4d80cceb75` ("Update EIP-8184: minor fix",
  2026-03-25). Bump this whenever the EIP text changes.
- **Need:** Drive the EIP from Draft toward Review/Last Call as the
  implementation and tests mature.

### 2. Execution-layer reference implementation — ✅ exists (draft PR)

The Python reference modelling LUCID as a fork.

- **Where it lives:** [jflo/execution-specs#1](https://github.com/jflo/execution-specs/pull/1)
  (branch `eips/unscheduled/eip-8184`), under
  [`src/ethereum/forks/lucid/`](https://github.com/jflo/execution-specs/tree/eips/unscheduled/eip-8184/src/ethereum/forks/lucid).
  Not yet upstreamed to [ethereum/execution-specs](https://github.com/ethereum/execution-specs).
- **Key files:**
  - `transactions.py` — `SealedTicketTransaction`, type byte `0x05`, signature
    encoding (`SEALED_TICKET_ECDSA_SIGNATURE_ID`).
  - `fork_types.py` — `STCommitment`, `KeyMessage` (decryption-key material).
  - `fork.py` — `SealedTransactionContext`, sealed-ticket ordering and
    validation, TOB-fee computation.
  - `exceptions.py` — `SealedTicketDecryptionError`, `SealedTicketFeeError`, etc.
  - `__init__.py` — fork definition (`Unscheduled(order_index=4)`).
- **Status:** Substantially implemented. Constants of record live in
  `tests/unscheduled/eip8184_lucid/spec.py`: `SEALED_TICKET_TX_TYPE = 0x05`,
  `SLOTNUM_OPCODE = 0x4B`, `TOB_GAS_FRACTION_DENOMINATOR = 8`,
  `TOB_FEE_FRACTION = 128`.
- **Need:** Keep in sync with the EIP once §1 lands, and upstream the PR.

### 3. Execution-layer reference tests — ⚠️ partial, in the draft PR

Tests that pin the reference implementation's behaviour.

- **Where they live:** [jflo/execution-specs#1](https://github.com/jflo/execution-specs/pull/1),
  under [`tests/unscheduled/eip8184_lucid/`](https://github.com/jflo/execution-specs/tree/eips/unscheduled/eip-8184/tests/unscheduled/eip8184_lucid).
  - `test_lucid_unit.py` — unit tests (fee math, ordering, validation); run with
    plain `pytest`.
  - `test_lucid.py` — blockchain tests (`SLOTNUM` behaviour); `valid_from("Lucid")`.
  - `helpers.py`, `conftest.py`, `spec.py` — fixtures and the reference-spec stub.
- **Status:** Exists for the implemented surface; coverage will grow with the
  spec.

### 4. Consensus-validity / fixture tests — ❌ not yet present

Client-agnostic conformance fixtures.

- **Where it belongs:** [ethereum/execution-spec-tests](https://github.com/ethereum/execution-spec-tests),
  under `tests/<fork>/eip8184_lucid/`.
- **Status:** **Missing.** No LUCID test module exists yet.
- **Need:** Port/author fixtures here once the EIP and fork are stable so all
  clients can validate against a shared suite.

### 5. Execution APIs (JSON-RPC / Engine API) — ❌ not yet present

How sealed tickets, commitments, and key messages are submitted and surfaced
over the wire.

- **Where it belongs:** [ethereum/execution-apis](https://github.com/ethereum/execution-apis)
  (JSON-RPC schemas and the Engine API under `src/`).
- **Status:** **Missing.** No sealed-ticket or commitment methods exist yet.
- **Need:** Define how clients accept sealed-ticket transactions, expose
  commitments, and exchange payloads/keys with the consensus layer across the
  Engine API.

### 6. Consensus-layer specification — ❌ not yet present

LUCID requires CL participation: commitment dissemination, the key-publication
mechanism, and scheduling.

- **Where it belongs:** [ethereum/consensus-specs](https://github.com/ethereum/consensus-specs),
  under `specs/<fork>/`.
- **Status:** **Missing.** The `KeyMessage` type in execution-specs explicitly
  notes that "the CL dissemination mechanism is out of scope for
  execution-specs" — meaning it is *in scope here* and must be specified on the
  consensus side.
- **Need:** Specify key publication, commitment inclusion in beacon blocks
  (`scheduling_beacon_block_root`, `scheduling_slot`, `commit_index`), and the
  key publisher's duties/penalties.

### 7. Client implementations — ⚠️ prerequisite only (Besu)

Production client support.

- **Where it lives:** [hyperledger/besu](https://github.com/hyperledger/besu)
- **Status / what exists:**
  - `SLOTNUM` opcode (`0x4B`) is already implemented via **EIP-7843**:
    - [`SlotNumOperation.java`](https://github.com/hyperledger/besu/blob/main/evm/src/main/java/org/hyperledger/besu/evm/operation/SlotNumOperation.java)
    - [`SlotNumOperationTest.java`](https://github.com/hyperledger/besu/blob/main/evm/src/test/java/org/hyperledger/besu/evm/operation/SlotNumOperationTest.java)
    - `EIP7843SlotNumOpcodeAcceptanceTest.java` (acceptance-tests)
  - No sealed-ticket transaction type (`0x05`), TOB fee market, commitment
    handling, or key-publication support yet.
- **Need:** Implement the sealed-ticket transaction, TOB fee logic, commitment
  processing, and CL coordination in Besu (and track the other clients
  externally).

### 8. Project coordination — ✅ this repo

- **Where it lives:** [jflo/lucid-pm](https://github.com/jflo/lucid-pm).
- **Need:** Keep this README current as the map of who owns what and where each
  artifact lives. As work moves between SDLC stages, update the **Status at a
  glance** table.
