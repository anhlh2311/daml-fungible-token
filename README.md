# Fungible Token (CIP‑0056) on Canton

This repository contains a minimal, standards‑compliant implementation of the Canton Network token standard CIP‑0056 for fungible tokens, plus a test package and example integration with the Splice Amulet reference.

The code demonstrates both direct (single‑step) transfers and two‑step, pending‑acceptance transfers with pre‑locked inputs, as well as DvP allocations in line with the standard’s allocation APIs.

## Status and compatibility

- **Interfaces implemented**: `HoldingV1`, `TransferInstructionV1` (`TransferFactory` + `TransferInstruction`), `AllocationInstructionV1` (`AllocationFactory`), `AllocationV1`.
- **Two transfer modes**:
  - **Single‑step**: used when sender == receiver (self‑transfer) for simple merge/split operations.
  - **Two‑step**: used when sender != receiver; inputs are aggregated and pre‑locked at instruction creation. Receiver later accepts or rejects.
- **Metadata**: uses the open `MetadataV1` map; callers can set arbitrary token attributes (name, symbol, issued‑at, description, …).
- **Reference**: The test sources include Splice Amulet modules to compare against the richer reference implementation (fees/rewards/etc.).

## Repository layout

- [fungible-token](/fungible-token/) — main package implementing the CIP‑0056 interfaces
  - [daml/Fungible/TokenHolding.daml](/fungible-token/daml/Fungible/TokenHolding.daml)
  - [daml/Fungible/TokenTransferInstruction.daml](/fungible-token/daml/Fungible/TokenTransferInstruction.daml)
  - [daml/Fungible/TwoStepTransferInstruction.daml](/fungible-token/daml/Fungible/TwoStepTransferInstruction.daml)
  - [daml/Fungible/TokenTransferFactory.daml](/fungible-token/daml/Fungible/TokenTransferFactory.daml)
  - [daml/Fungible/TokenAllocation.daml](/fungible-token/daml/Fungible/TokenAllocation.daml)
  - [daml/Fungible/TokenAllocationFactory.daml](/fungible-token/daml/Fungible/TokenAllocationFactory.daml)

- [fungible-token-test](/fungible-token-test) — Daml Script tests and helpers
  - [daml/FungibleTokenTest.daml](/fungible-token-test/daml/FungibleTokenTest.daml)
    - `setupToken`: shared test setup for parties, instrument and initial issuances
    - Tests for two‑step accept/reject/withdraw, expired single‑step, update‑failure, wrong admin, insufficient balance

- [external-test-sources/splice-token-standard-test](/external-test-sources/splice-token-standard-test) — upstream testing utilities and examples (git submodule)

## Prerequisites

- Daml SDK (as per package configs) — e.g. the snapshot in
  - [fungible-token/daml.yaml](/fungible-token/daml.yaml)
  - [fungible-token-test/daml.yaml](/fungible-token-test/daml.yaml)
- Canton or Sandbox for running scripts (tests use in‑memory test runner).
- Git with submodule support (`external-test-sources` are included as a git submodule).

## Getting Started

### Clone with Submodules

When cloning this repository, make sure to include the submodules:

```bash
git clone --recursive https://github.com/AngelhackDev/daml-fungible-token.git
```

If you've already cloned without submodules, initialize them:

```bash
git submodule update --init --recursive
```

## Build

From the repository root (multi-package):

```bash
# First, ensure submodules are initialized
git submodule update --init --recursive

# Then build all packages
daml build --all
```

Or build packages individually:

```bash
cd fungible-token
daml build

cd ../fungible-token-test
daml build
```

## Run tests

- Run tests in the test DAR using the in‑memory test runner:

```bash
cd fungible-token-test
daml test
```

- Narrow to a specific file:

```bash
daml test --files daml/FungibleTokenTest.daml
```

## Concepts

### Instrument

- An instrument identifies a token type as `{ admin : Party, id : Text }` (see [HoldingV1](https://github.com/hyperledger-labs/splice/blob/main/token-standard/splice-api-token-holding-v1/daml/Splice/Api/Token/HoldingV1.daml)).
- From the same code, multiple tokens (e.g., “USDC”, “USDT”) are created by using different `InstrumentId`s and admin parties.

### Metadata

- `MetadataV1` is an open key→value map (TextMap). You can attach business attributes such as name, symbol, issued‑at, description, etc., on factories, holdings, and results.
- Example (caller‑provided):

```haskell
import Splice.Api.Token.MetadataV1 as M
import DA.TextMap as TM

let tokenMeta = M.Metadata with
      values = TM.fromList
        [ ("token.example.org/name", "MyToken")
        , ("token.example.org/symbol", "MTK")
        , ("token.example.org/issued-at", "2025-08-21T12:34:56Z")
        ]
```

### Transfers

- Single‑step (self‑transfers): [TokenTransferInstruction.daml](/fungible-token/daml/Fungible/TokenTransferInstruction.daml)
  - Inputs are validated and archived at accept time; sender change and receiver holdings are created immediately.
- Two‑step (pending acceptance, sender ≠ receiver): [TwoStepTransferInstruction.daml](/fungible-token/daml/Fungible/TwoStepTransferInstruction.daml)
  - Inputs are aggregated, archived, and converted into a single locked holding at instruction creation; receiver later accepts (consumes lock, creates receiver holding) or rejects/withdraws (returns funds to sender).

### DvP allocations

- Lock funds into an allocation and settle/cancel/withdraw via [TokenAllocationFactory.daml](/fungible-token/daml/Fungible/TokenAllocationFactory.daml) and [TokenAllocation.daml](/fungible-token/daml/Fungible/TokenAllocation.daml).
- **Purpose**: Reserve funds for multi‑party or cross‑asset trades so they can be settled atomically later (Delivery‑vs‑Payment). This splits a trade into a prepare phase (allocate/lock) and a settle phase (execute/cancel/withdraw).
- **Flow in this implementation**:
  - **AllocationFactory** aggregates specified input holdings, archives them, returns any change to the sender, and creates a single locked holding for the leg amount. The lock is held by the settlement executor and expires at `settleBefore`.
  - An **Allocation** references that locked holding and encodes the leg (`sender`, `receiver`, `amount`, `instrument`, `timing`, `executor`).
  - **ExecuteTransfer** consumes the locked holding and creates the receiver holding(s).
  - **Withdraw/Cancel** consumes the locked holding and returns an unlocked holding to the sender.
- **Benefits**: Ensures funds are pre‑funded and cannot be double‑spent while a trade is being coordinated; interoperable with CIP‑0056 `AllocationInstructionV1`/`AllocationV1` wallets and backends.

## Usage examples

### Create a transfer instruction (factory call)

```haskell
let extraArgs = MetaV1.ExtraArgs with
      meta = tokenMeta
      context = MetaV1.emptyChoiceContext

exerciseCmd (toInterfaceContractId @TransferInstrV1.TransferFactory transferFactoryCid)
  TransferInstrV1.TransferFactory_Transfer with
    expectedAdmin = admin
    transfer = TransferInstrV1.Transfer with
      sender
      receiver
      amount
      instrumentId
      requestedAt
      executeBefore
      inputHoldingCids
      meta = tokenMeta
    extraArgs
```

### Accept or reject a pending transfer

```haskell
exerciseCmd instrCid TransferInstrV1.TransferInstruction_Accept with extraArgs
-- or
exerciseCmd instrCid TransferInstrV1.TransferInstruction_Reject with extraArgs
```

## Sequence diagrams

### Two‑step transfer

```mermaid
sequenceDiagram
  participant S as Sender
  participant R as Receiver
  participant F as TransferFactory
  participant L as LockedHolding

  S->>F: TransferFactory_Transfer (sender≠receiver, inputs)
  activate F
  F->>F: Validate and archive inputs, then return change
  F->>L: Create locked holding (amount, expiresAt=executeBefore)
  F-->>S: Pending + senderChangeCids
  deactivate F

  R->>F: TransferInstruction_Accept
  activate F
  F->>L: Archive locked holding
  F-->>R: Create receiver holding
  F-->>R: Completed
  deactivate F
```

### Allocation (DvP) settlement

```mermaid
sequenceDiagram
  participant E as Executor
  participant AF as AllocationFactory
  participant A as Allocation
  participant L as LockedHolding
  participant R as Receiver

  E->>AF: AllocationFactory_Allocate (transferLeg, inputs)
  AF->>AF: Validate and archive inputs, then return change
  AF->>L: Create locked holding
  AF-->>E: AllocationInstructionResult(Completed, allocationCid)

  E->>A: Allocation_ExecuteTransfer
  A->>L: Archive lock
  A-->>R: Create receiver holding
  A-->>E: Allocation_ExecuteTransferResult
```

## Development notes

- Two‑step instruction creation lives in [TokenTransferFactory.daml](/fungible-token/daml/Fungible/TokenTransferFactory.daml). The factory decides between single‑step and two‑step:
  - sender == receiver ⇒ single‑step (`TokenTransferInstruction`)
  - sender ≠ receiver ⇒ two‑step (`TokenTwoStepTransferInstruction`), with a pre‑locked holding

- Tests are organized in [FungibleTokenTest.daml](/fungible-token-test/daml/FungibleTokenTest.daml). Use `setupToken` to initialize parties, an instrument and initial balances for each test.

## External Test Sources

The external test sources (`external-test-sources/`) are now managed as a separate repository and included as a git submodule. This provides better dependency management and allows the test utilities to be reused across multiple projects.

### Quick Submodule Commands

```bash
# Update external test sources to latest version
git submodule update --remote external-test-sources

# Initialize submodules (if not already done)
git submodule update --init --recursive
```

## License

The implementation builds on interfaces and test sources that are Apache‑2.0 licensed. See headers in the referenced files for details.
