---
name: mirage
description: Write, review, test, and debug Mirage programming-language code and Mirage core-library modules. Use for `.mir` files, Mirage module/import design, allocator-aware containers and strings, `error(...)` handling, `@test` suites, compiler diagnostics, crashes or wrong-code investigation, MIR inspection, module-resolution problems, and work spanning the Mirage standard-library and Mirage-Cpp compiler repositories.
---

# Mirage Development

Treat the checked-out compiler and library sources as authoritative. Mirage is evolving; do not substitute syntax or semantics remembered from another language.

## Establish the workspace

1. Locate the standard-library root containing `core/` and the compiler checkout containing `src/` and `build/mirage`.
2. Prefer a `mirage` executable on `PATH`; otherwise use the repository build, commonly a sibling `Mirage-Cpp/build/mirage`.
3. Run commands from the standard-library root when compiling core modules. Pass `--std=<library-root>` when the module root is otherwise ambiguous.
4. Read the target module, its tests, imported APIs, and relevant call sites before editing. Preserve unrelated dirty-worktree changes.
5. Check the current CLI with `mirage --help`; flags may evolve.

Use `references/language-and-library.md` when writing or reviewing Mirage syntax, APIs, ownership, imports, or tests. Use `references/debugging.md` when diagnosing compiler errors, crashes, wrong code, module resolution, MIR, or backend behavior.

## Choose the workflow

### Implement or change Mirage code

1. Reproduce the current behavior with the narrowest relevant command.
2. Inspect adjacent modules for established naming, error, allocator, and ownership conventions.
3. Make the smallest coherent change. Do not silently change public contracts to accommodate an implementation bug.
4. Add tests for success, boundaries, error propagation, allocator failure, state preservation, cleanup, and prior regressions.
5. Run the focused module tests, then the recursive core suite when the change can affect consumers.

```sh
mirage test core/path
mirage test core --all
```

### Add tests

- Put `@test` functions in a `.mir` file in the module directory; all `.mir` files in that directory form one module.
- Test observable behavior first. Inspect public state only when it represents a documented invariant or is required to verify allocator/ownership behavior.
- Use deterministic collision/hash helpers for containers and `nil_allocator()` for allocation-failure paths.
- Cover exact boundaries: empty state, first element, exact capacity, growth threshold, last valid index, first invalid index, overflow, clear/reuse, and destroy/reset.
- Keep tests independent. Each test owns and destroys its resources with `defer` where appropriate.
- A failing or crashing test may reveal a real implementation defect. Confirm the expected contract before weakening the assertion.

Minimal pattern:

```mirage
pub type Example_Test_Error = enum(i32) {
    Failed = 1
}

pub type Example_Test_Result = error(
    Example_Test_Error | Allocator_Error_Kind | Example_Error_Kind
)

@test
fn operation_preserves_invariants() -> Example_Test_Result {
    mut value := make_example()
    defer value.destroy()

    try value.operation()
    if !value.expected_condition() {
        return_err .Failed
    }
    return_ok
}
```

### Diagnose a failure

1. Run the exact failing command and capture the first root diagnostic.
2. Separate parser, semantic, MIR/codegen, link, runtime, and test-harness failures.
3. Reduce to one module or a temporary minimal module without modifying user code.
4. Compare inferred and explicit types, especially generic types imported through namespaces.
5. Inspect generated structure with `--dump-ast`, module lookup with `--print-module-search`, and lowering with `--emit-mir`.
6. For suspected optimization or allocation faults, compare `--mir-opt`, `--regalloc=trivial`, `--nortti`, and `--no-eager-generic-check` as relevant.
7. Diagnose only when asked to diagnose. Modify code only when the request authorizes a fix.

## Verification ladder

Run checks in proportion to the change:

```sh
# One module and its imported tests
mirage test core/collections/list

# A module tree recursively
mirage test core --all

# Compile/run an application module
mirage build path/to/module
mirage run path/to/module

# Inspect compiler stages
mirage build path/to/module --dump-ast
mirage build path/to/module --emit-mir
mirage build path/to/module --emit-mir --mir-opt
mirage build path/to/module --print-module-search
```

When changing Mirage-Cpp itself, run the repository battery from the compiler checkout:

```sh
just test
```

Set `MIRAGE_STD=/path/to/Mirage` if the standard library is not the expected sibling checkout.

## Guardrails

- Do not assume a file is a module. A directory is a module; its `.mir` files share module scope.
- Prefer namespace imports (`#import("core/collections/list") as list`) for reusable modules and generic types. Qualify their types and constructors consistently.
- Do not compare slices with `==` or `!=`; compare lengths and elements.
- Do not discard a fallible result accidentally. Use `try`, capture the error return, or deliberately handle it.
- Maintain allocator invariants: on failure, initialized length/count and owned pointers must remain valid; `destroy` must reset owned state.
- Check unsigned addition before computing required capacity. Exercise exact-capacity and overflow cases.
- `@test` cases run in forked children: a crash is isolated but remains a real defect.
- Never report success from compilation alone when runtime behavior or tests were requested.
- Report the exact commands and pass/fail totals. Distinguish changes made by the agent from pre-existing user edits.
