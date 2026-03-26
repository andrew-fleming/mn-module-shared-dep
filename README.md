# mn-module-shared-dep

When two modules are imported into a contract with separate prefixes,
each module's own ledger state is correctly isolated into independent slots.
However, if both modules import the same third module (a shared transitive dependency),
the compiler deduplicates that dependency's ledger state into a single shared instance rather than isolating it per importing module.
This means two modules that are intended to be independent end up sharing mutable state from their common dependency.

This behavior appears to be determined by the importing module's directory context:
when two modules in the same directory both import a shared dependency,
the compiler deduplicates that dependency's ledger state into a single instance,
even when the dependency itself resides in a separate directory.
When the importing modules reside in different directories,
the same dependency gets separate state per module, despite resolving to the same file.

## File Layout

```text
src/
  common/
    Common.compact    # Shared dependency with `isInitialized: Boolean`
  dir1/
    A.compact         # Imports ../common/Common
    B.compact         # Imports ../common/Common
  dir2/
    C.compact         # Imports ../common/Common
  MyContract.compact  # Imports `A`, `B`, `C` with prefixes
```

`A` and `B` are in the same directory.
`C` has identical code but resides in a different directory.
All three import the same `Common` module.

## Steps to reproduce

1. Clone this repo
2. Compile the contract (compact version: 0.30.0):

```bash
compact compile src/MyContract.compact ./artifacts
```

3. Inspect `artifacts/contract/index.js` and search for `_isInitialized`.

## Expected behavior

**Option A (isolation):** Each prefixed module import receives its own
independent copy of the transitive dependency's ledger state.
`A`, `B`, and `C` each get their own `_isInitialized` slot.
This is consistent with the expectation that prefixed imports create
fully independent namespaces.

**Option B (shared state):** Transitive dependencies that resolve to
the same file share ledger state across all importing modules.
`A`, `B`, and `C` all share a single `_isInitialized` slot.
This would be a valid design choice if documented and consistent.

## Actual behavior

Neither. `A` and `B` share slot 0, but `C` gets its own slot 1 which seems to be determined
by directory structure rather than any semantic property of the modules.

In the compiled artifact, the three generated functions show this directly:

- `_isInitialized_0` (A) → reads slot 0
- `_isInitialized_1` (B) → reads slot 0
- `_isInitialized_2` (C) → reads slot 1
