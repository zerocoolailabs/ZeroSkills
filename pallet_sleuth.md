Analyze Substrate FRAME pallets and runtime storage code for **storage-safety vulnerabilities** that can cause persistent on-chain state updates to be lost, overwritten, misdirected, orphaned, or to become undecodable across runtime upgrades and migrations.

Focus on bugs involving:

- read-modify-write omissions (`get`, `take`, clone, or local copy mutated but never persisted)
- raw or attacker-influenced storage key writes
- storage layout, prefix, or encoding hazards across runtime upgrades and migrations
- partial updates that desynchronize primary storage from indexes, counters, totals, or ownership records
- other storage semantics that can corrupt or invalidate persistent state

Only report issues that have a **clear persistence or state integrity impact**.

---

## Applicability gates

Only proceed if BOTH conditions are true:

1. The pallet or runtime contains **state-mutating logic** intended to update persistent storage (`#[pallet::call]`, hooks, admin functions, migration paths, issuance/accounting updates, role/config changes, queues, registries, position maps, etc.).

2. At least one of the following storage-risk signals is present:

- writes to `StorageValue`, `StorageMap`, `StorageDoubleMap`, `StorageNMap`, or `CountedStorageMap`
- read-modify-write flows using `get`, `try_get`, `take`, cloning, or temporary decoded values
- raw storage access such as `sp_io::storage::*`, `frame_support::storage::unhashed::*`, or child storage APIs
- custom migrations, `OnRuntimeUpgrade`, `VersionedMigration`, `StorageVersion`, or manual translation logic
- coupled state such as map plus counter, reverse index, aggregate total, queue length, approval record, or ownership list

If these gates are not satisfied, do not force findings.

---

## Workflow

### 1) Inventory storage and storage writers

Map the pallet's persistent state surface before looking for bugs.

- Identify key storage items such as balances, positions, total issuance, configuration, roles, queues, escrows, registries, counters, reverse indexes, and migration/version flags.
- Enumerate functions and hooks that mutate them:
    - `#[pallet::call]` dispatchables
    - internal helpers reachable from those dispatchables
    - `on_initialize`, `on_finalize`, `on_idle`, `on_runtime_upgrade`, and privileged admin paths
- Identify manual key usage such as:
    - `sp_io::storage::set`, `clear`, `append`, or child storage writes
    - `frame_support::storage::unhashed::*`
    - manual key construction, prefix concatenation, or custom encoding before storage access

Use this inventory to anchor later findings to **specific storage items and write paths**.

---

### 2) Lost write / read-modify-write omission

Look for cases where storage-backed data is loaded into an owned temporary value, mutated, but never written back to storage.

Typical pattern:

- A storage value (struct, map entry, queue item, or config object) is loaded with `get`, `try_get`, `take`, or a clone into a local variable.
- The local value is mutated.
- The mutation is never persisted with `insert`, `set`, `put`, `append`, `mutate`, `try_mutate`, `mutate_exists`, or an equivalent write path.

Report a finding only if **all** of the following are true:

- A storage-backed value is copied or decoded into a non-storage temporary.
- That temporary is mutated (field assignment, push/pop, arithmetic update, flag flip, map insertion, etc.).
- On all reachable paths, the mutated value is **not written back** to the original storage location.
- The surrounding logic indicates that persistence was expected.

Indicators that persistence is expected:

- function or event name suggests an update
- emitted events imply a state change
- later code reads the storage assuming it was updated
- callers rely on the update for accounting, authorization, queue progress, or migration completion

Do not report if:

- the code intentionally constructs a derived copy for return, validation, or comparison
- mutation occurs through a storage closure such as `mutate`, `try_mutate`, `mutate_exists`, or `translate`
- the value is written back through a helper function on all paths
- the temporary is only used for dry-run logic, benchmarking, or weight estimation

---

### 3) Attacker-influenced storage key writes

Look for cases where **untrusted input affects which storage key is written**.

This usually occurs in:

- raw storage APIs (`sp_io::storage::*`)
- unhashed storage helpers
- child trie storage
- manual pallet/storage prefix construction
- custom encoding or hashing before a write

Report a finding only if **all** of the following are true:

- attacker-controlled input influences the final storage key, child key, or namespace used in a write
- that key space is not strictly constrained by typed storage APIs, allowlists, origin checks, or provenance checks
- the resulting write can corrupt unrelated or sensitive state

Examples of realistic overwrite targets:

- owner, admin, or role flags
- configuration parameters and operational switches
- balances, ledgers, reservations, locks, or accounting totals
- migration flags, `StorageVersion`, or next-id counters
- another pallet's namespace or a sensitive child trie

Do not report if:

- the write goes through a declared typed storage item with a fixed pallet/storage prefix and supported hasher
- the raw key is a fixed constant used in a clearly scoped standard pattern
- strong access control and namespace validation prevent untrusted writes outside the intended domain

---

### 4) Runtime upgrade and storage migration hazards

If the pallet participates in runtime upgrades or storage migrations, examine storage layout and decode safety.

Look for situations where persisted state from older runtimes can become unreadable, misplaced, or partially migrated.

High-risk signals include:

- changing a storage item's key type, value type, hasher, query kind, or prefix without a migration
- changing struct fields, enum variants, field order, compact encoding, or wrapper types in ways that break old SCALE data
- renaming pallets or storage items, splitting or merging maps, or changing ownership/index structures without backfilling state
- migrations that are non-idempotent, partially applied, weight-truncated, version-gated incorrectly, or missing `StorageVersion` updates
- manual decode/translate logic that drops entries, defaults corrupted values silently, or skips failure cases

Report layout issues carefully:

- If you can demonstrate an **actual decode failure, orphaned state set, or corrupted migrated state**, report it as a concrete vulnerability.
- If there is only a credible risk during the next upgrade, report it as an **upgrade-safety risk**.

Do not report if:

- the pallet has no upgrade or migration path relevant to the changed storage
- a migration explicitly handles the old layout, is version-gated correctly, and updates version markers
- the change is backward-compatible and the read/write paths preserve prior state semantics

---

### 5) Other storage semantics (only when security-relevant)

Look for storage behaviors that can corrupt or desynchronize state.

Examples include:

- deleting or overwriting a primary record while leaving reverse indexes, approvals, counters, totals, or queue metadata unchanged
- using `take`, `mutate_exists`, `kill`, `clear_prefix`, or `drain` without repairing dependent state
- incrementing or decrementing totals separately from per-account or per-position storage so accounting drifts
- treating decode failure or missing storage as `Default` in a way that resets privileges, limits, or balances
- marking a migration, settlement, or queue item as complete before all dependent writes have occurred

Only report issues where the result could cause:

- privilege escalation
- incorrect accounting
- corrupted configuration
- inaccessible or misaccounted funds/assets
- broken runtime or protocol invariants
