# Mirage debugging reference

Read this reference when code does not compile, crashes, produces wrong output, or resolves the wrong module.

## Classify the failure

| Symptom | Start here |
|---|---|
| Token/parse diagnostic | `--dump-ast`, nearby grammar tests |
| Unknown name/type or mismatch | imports, qualification, generic instantiation, declared error union |
| Cannot resolve module | `--print-module-search`, working directory, `--std` |
| MIR verification/codegen error | `--emit-mir`, then `--mir-opt` comparison |
| Link failure | `#link` directives, `-l`, target, linker driver |
| Runtime crash | smallest test, bounds/ownership, allocator state, pointer lifetime |
| Wrong code | optimized/unoptimized MIR, linear/trivial register allocation, minimal reproducer |
| Test is `FAILED` | assertion or returned error path |
| Test is `CRASHED` | panic, signal, out-of-bounds access, unhandled error |

## Diagnostic commands

```sh
mirage test path/to/module
mirage build path/to/module --dump-ast
mirage build path/to/module --print-module-search
mirage build path/to/module --emit-mir
mirage build path/to/module --emit-mir --mir-opt
mirage build path/to/module --regalloc=trivial
mirage build path/to/module --nortti
mirage build path/to/module --no-eager-generic-check
mirage build path/to/module --print-link-directives
```

Use one variable at a time. A mode that makes the problem disappear narrows the responsible stage; it does not by itself prove the root cause.

## Common failure patterns

### Cascading diagnostics

Start with the earliest type or call error. Unknown identifiers after a failed destructuring call are usually cascades.

### Generic import identity

If an imported generic value is incompatible with an apparently identical explicit type, use one namespace consistently:

```mirage
#import("core/collections/list") as list
type Visited = list.List[anyptr]

mut visited := list.make_list[anyptr]()
```

### Error accidentally dropped

This may compile but panic on failure:

```mirage
mut value := fallible_constructor()
```

Propagate or capture it:

```mirage
mut value := try fallible_constructor()
const value, err := fallible_constructor()
```

### Capacity arithmetic

Required capacity is normally `length + additional`, guarded before addition. Test empty insertion, exact-full insertion, multi-element append, overflow, clear then reuse, and failure-state preservation.

### Off-by-one stacks and cursors

For a stack depth representing the number of live entries, pop with `depth--; entries[depth]`. Test depth zero, one, nested entries, and repeated parent/backtracking operations.

### Ownership after errors

When a later operation fails after an earlier allocation, confirm whether temporary storage is destroyed. Test with `nil_allocator()` where possible and inspect pointer/length/capacity state after the call.

### Unsafe self-aliasing

Appending a slice that points into a buffer may become invalid if growth reallocates before copying. Determine whether self-append is part of the API contract; if so, test both with and without growth.

## Compiler-level regression workflow

When the compiler is likely at fault:

1. Create the smallest module directory reproducing the failure.
2. Confirm it against the current compiler binary.
3. Search `tests/` and `examples/` for a related regression.
4. Inspect the stage corresponding to the failure: parser/AST, sema/type resolution, MIR generation/verification, optimization, register allocation, backend, linker driver, or test harness.
5. Add a regression fixture before or with the compiler fix.
6. Run the focused test and `just test` from Mirage-Cpp.

Do not edit the standard library merely to conceal a compiler type-identity, lowering, or backend defect unless the library code itself violates its intended contract.
