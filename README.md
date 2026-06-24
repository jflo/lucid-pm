# lucid-pm

Project-management hub for **EIP-8184: LUCID — encrypted mempool**.

This repository is the shared, central reference for everyone working on LUCID.
Its job is to answer one question: *where is LUCID represented across the
Ethereum software-development lifecycle (SDLC), and what still needs to be
built?*

If you are picking up work on LUCID, start here. Each section below describes a
**need** of the project, points at **where it lives today** (if it exists), and
flags **what is still missing**.

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

> **Status:** LUCID is **unscheduled** (no target fork). In `execution-specs`
> it is modelled as a standalone fork named `Lucid` with
> `Unscheduled(order_index=4)`.

---

## The needs of the project, and where to find them

LUCID is a coordinated execution-layer **and** consensus-layer change. Below is
each artifact the project requires, in roughly the order it flows through the
Ethereum SDLC. Sibling repositories are checked out alongside this one (see
[Repository layout](#repository-layout)).

### 1. EIP specification — ✅ published (Draft)

The normative specification document.

- **Where it lives:** <https://eips.ethereum.org/EIPS/eip-8184>
  (canonical git path `EIPS/eip-8184.md`; also in the local `eips/` checkout).
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

### 2. Execution-layer reference implementation — ✅ exists

The Python reference (`ethereum/execution-specs`) modelling LUCID as a fork.

- **Where it lives:** `execution-specs/src/ethereum/forks/lucid/`
  — currently a **draft PR**: [jflo/execution-specs#1](https://github.com/jflo/execution-specs/pull/1)
  (branch `eips/unscheduled/eip-8184`), not yet upstreamed to
  `ethereum/execution-specs`.
- **Key files:**
  - `transactions.py` — `SealedTicketTransaction`, type byte `0x05`, signature
    encoding (`SEALED_TICKET_ECDSA_SIGNATURE_ID`).
  - `fork_types.py` — `STCommitment`, `KeyMessage` (decryption-key material).
  - `fork.py` — `SealedTransactionContext`, sealed-ticket ordering and
    validation, TOB-fee computation.
  - `exceptions.py` — `SealedTicketDecryptionError`, `SealedTicketFeeError`, etc.
  - `__init__.py` — fork definition (`Unscheduled(order_index=4)`).
- **Status:** Substantially implemented. Constants of record live in
  `execution-specs/tests/unscheduled/eip8184_lucid/spec.py`:
  `SEALED_TICKET_TX_TYPE = 0x05`, `SLOTNUM_OPCODE = 0x4B`,
  `TOB_GAS_FRACTION_DENOMINATOR = 8`, `TOB_FEE_FRACTION = 128`.
- **Need:** Keep in sync with the EIP once §1 lands.

### 3. Execution-layer reference tests — ⚠️ partial, in-tree only

Tests that pin the reference implementation's behaviour.

- **Where they live:** `execution-specs/tests/unscheduled/eip8184_lucid/`
  - `test_lucid_unit.py` — unit tests (fee math, ordering, validation); run with
    plain `pytest`.
  - `test_lucid.py` — blockchain tests (`SLOTNUM` behaviour); `valid_from("Lucid")`.
  - `helpers.py`, `conftest.py`, `spec.py` — fixtures and the reference-spec stub.
- **Status:** Exists for the implemented surface; coverage will grow with the
  spec.

### 4. Consensus-validity / fixture tests — ❌ not yet present

Client-agnostic conformance fixtures (`ethereum/execution-spec-tests`).

- **Where it belongs:** `execution-spec-tests/tests/<fork>/eip8184_lucid/`
- **Status:** **Missing.** No LUCID test module exists in the
  `execution-spec-tests` checkout.
- **Need:** Port/author fixtures here once the EIP and fork are stable so all
  clients can validate against a shared suite.

### 5. Execution APIs (JSON-RPC / Engine API) — ❌ not yet present

How sealed tickets, commitments, and key messages are submitted and surfaced
over the wire (`ethereum/execution-apis`).

- **Where it belongs:** `execution-apis/src/` (JSON-RPC schemas, Engine API).
- **Status:** **Missing.** No sealed-ticket or commitment methods exist yet.
- **Need:** Define how clients accept sealed-ticket transactions, expose
  commitments, and exchange payloads/keys with the consensus layer across the
  Engine API.

### 6. Consensus-layer specification — ❌ not yet present

LUCID requires CL participation: commitment dissemination, the key-publication
mechanism, and scheduling (`ethereum/consensus-specs`).

- **Where it belongs:** `consensus-specs/specs/<fork>/`
- **Status:** **Missing.** The `KeyMessage` type in `execution-specs` explicitly
  notes that "the CL dissemination mechanism is out of scope for
  execution-specs" — meaning it is *in scope here* and must be specified on the
  consensus side.
- **Need:** Specify key publication, commitment inclusion in beacon blocks
  (`scheduling_beacon_block_root`, `scheduling_slot`, `commit_index`), and the
  key publisher's duties/penalties.

### 7. Client implementations — ⚠️ prerequisite only (Besu)

Production client support. This checkout includes **Besu**.

- **Where it lives:** `besu/`
- **Status / what exists:**
  - `SLOTNUM` opcode (`0x4B`) is already implemented via **EIP-7843**:
    - `besu/evm/src/main/java/org/hyperledger/besu/evm/operation/SlotNumOperation.java`
    - `besu/evm/src/test/java/org/hyperledger/besu/evm/operation/SlotNumOperationTest.java`
    - `besu/acceptance-tests/.../EIP7843SlotNumOpcodeAcceptanceTest.java`
  - No sealed-ticket transaction type (`0x05`), TOB fee market, commitment
    handling, or key-publication support yet.
- **Need:** Implement the sealed-ticket transaction, TOB fee logic, commitment
  processing, and CL coordination in Besu (and track the other clients
  externally).

### 8. Project coordination — ✅ this repo

- **Where it lives:** here (`lucid-pm`).
- **Need:** Keep this README current as the map of who owns what and where each
  artifact lives. As work moves between SDLC stages, update the status markers
  above.

---

## Status at a glance

| # | Need | Repo | Status |
|---|------|------|--------|
| 1 | EIP specification | `eips` | ✅ Published (Draft) |
| 2 | EL reference implementation | `execution-specs` | ✅ Draft PR ([#1](https://github.com/jflo/execution-specs/pull/1)) |
| 3 | EL reference tests | `execution-specs` | ⚠️ Partial (in PR #1) |
| 4 | Conformance fixtures | `execution-spec-tests` | ❌ Missing |
| 5 | Execution APIs (JSON-RPC / Engine) | `execution-apis` | ❌ Missing |
| 6 | Consensus-layer spec | `consensus-specs` | ❌ Missing |
| 7 | Client implementation | `besu` | ⚠️ SLOTNUM only (EIP-7843) |
| 8 | Project coordination | `lucid-pm` | ✅ This repo |

Legend: ✅ exists · ⚠️ partial · ❌ not yet present.

---

## Repository layout

These repositories are expected to be checked out as siblings (the maintainer's
layout is `~/src/lucid/`):

```
lucid/
├── lucid-pm/              ← you are here (project coordination)
├── eips/                  ← EIP-8184 spec (published, Draft)
├── execution-specs/       ← EL reference impl + tests (forks/lucid)
├── execution-spec-tests/  ← client-agnostic conformance fixtures
├── execution-apis/        ← JSON-RPC / Engine API definitions
├── consensus-specs/       ← CL spec (key publication, scheduling)
└── besu/                  ← Besu client implementation
```

## Key constants (reference)

From `execution-specs/tests/unscheduled/eip8184_lucid/spec.py`:

| Name | Value | Meaning |
|------|-------|---------|
| `SEALED_TICKET_TX_TYPE` | `0x05` | EIP-2718 type byte for sealed-ticket txs |
| `SLOTNUM_OPCODE` | `0x4B` | Opcode returning the commitment slot |
| `TOB_GAS_FRACTION_DENOMINATOR` | `8` | Top-of-block gas fraction denominator |
| `TOB_FEE_FRACTION` | `128` | Top-of-block fee fraction |

---

## Contributing

When you create, move, or complete a LUCID artifact in any of the sibling
repositories, update the relevant section and the **Status at a glance** table
above so this remains an accurate map of the project.
