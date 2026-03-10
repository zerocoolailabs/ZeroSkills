```
 _____  _____  ____    ___     ____  _  __ ___  _      _      ____
|__  / | ____||  _ \  / _ \   / ___|| |/ /|_ _|| |    | |    / ___|
  / /  |  _|  | |_) || | | |  \___ \| ' /  | | | |    | |    \___ \
 / /_  | |___ |  _ < | |_| |   ___) | . \ | | | |___ | |___  ___) |
/____| |_____||_| \_\ \___/   |____/|_|\_\|___||_____||_____||____/
```

# Zero Skills

Zero-shot vulnerability detectors from **Zero Cool**.

Skills that aim to inject information outside the model's training distribution. The vulnerabilities that win contests and matter in production tend to live outside what frontier models already know. A skill is useful when it extends the model's effective knowledge boundary.

## Why these skills exist

Most public security skills restate well-documented patterns. Reentrancy, access control, integer overflow. Frontier models already encode this knowledge from years of public audits, educational material, and vulnerability databases.

Zero Skills target what the model doesn't know:

- **Domain-specific context** learned through experience finding real bugs
- **Protocol-specific invariants** that must hold but aren't in any training set
- **Nuanced implementation details** around storage semantics, upgrade patterns, and state persistence
- **Post-training information** from incidents and patterns discovered after the model's cutoff
- **Edge cases the model won't infer** because they don't appear often enough in public data

## Skills

### Slot Sleuth (`code_sleuth.md`)

EVM storage-safety vulnerability detector. Targets bugs that cause persistent state updates to be lost, overwritten, misdirected, or collide across proxy and upgrade boundaries.

**What it finds:**

- Memory vs storage confusion
- Lost writes (state mutated in memory, never persisted)
- Attacker-influenced storage slot writes
- Storage collisions across proxy/upgrade layouts
- Upgrade layout hazards

The skill runs five detection phases with explicit applicability gates. If the contract doesn't have the relevant storage patterns, it doesn't force findings. It encodes heuristics from human experience finding these bugs in production code.

### Pallet Sleuth (`pallet_sleuth.md`)

Substrate FRAME storage-safety vulnerability detector. Targets bugs that cause pallet state updates to be lost, raw storage writes to hit the wrong namespace, or runtime upgrades and migrations to orphan or corrupt persisted state.

**What it finds:**

- Read-modify-write omissions
- Attacker-influenced raw storage key writes
- Runtime upgrade and storage migration hazards
- Coupled-state desynchronization across maps, counters, and indexes
- Other storage semantics that break persistent state integrity

The skill mirrors the same five-phase, gated review style as Slot Sleuth, but adapts the heuristics to Rust pallets, FRAME storage APIs, raw key access, and SCALE-encoded upgrade paths.

## Contributing

Build a skill that finds real vulnerabilities, the kind that live outside the training distribution, and we'll pay you for it. [Details coming soon.]

## Contact

**Website:** [zerocool.ai](https://zerocool.ai)
**Twitter:** [@ZeroCool_AI](https://twitter.com/ZeroCool_AI)

Built by security researchers. For teams building onchain.
