# Self-hosting the Mirage compiler

This document is the implementation roadmap for rewriting the Mirage compiler in Mirage.
It is intentionally organized as a sequence of independently useful, independently
testable milestones. Do not start a milestone until the previous milestone's exit gate is
green.

The existing C++ compiler is the bootstrap seed and behavioral oracle. The rewrite may use
more Mirage-friendly internal representations, but it must preserve observable language
behavior. The first self-hosted target is x86-64 Linux. WebAssembly follows only after the
native compiler reaches a reproducible bootstrap fixed point.

## Destination architecture

All reusable compiler functionality belongs in the core library. Only the executable
driver remains outside it:

```text
core/mirage/
  source/          Source ownership, spans, line maps, and source providers
  diagnostics/     Structured diagnostics, ordering, rendering, and deduplication
  tokenizer/       Mirage and inline-assembly tokenization
  ast/             Handle-based syntax trees and AST inspection
  parser/          Parser and error recovery
    modules/        Module discovery, import resolution, and source loading
  sema/            Constant evaluation and semantic/body checking
    symbols/        Scopes, declarations, visibility, and namespaces
    types/          Resolved types, interning, layout, and type printing
  asm/             Inline-assembly model and validation
  mir/             MIR model, builder, verifier, printer, and passes
    lowering/       Checked program to MIR lowering
  compiler/        Target-independent compilation facade
    arch/
      x86/          x86-64 encoding, ABI lowering, allocation, and code generation
    elf/            Relocatable ELF object generation

compiler/
  main.mir         CLI, host environment, artifacts, linker, and process execution
```

Indentation is significant in this diagram: nested modules live below the subsystem that
owns them. `parser/modules` consumes syntax and source providers; `sema/symbols` and
`sema/types` are semantic implementation modules; MIR lowering belongs to MIR; and target
and object emission belong to the compiler facade. The directory names describe API
boundaries, not a requirement that every directory contain exactly one Mirage module.
Split a directory when that makes its unit tests or ownership rules clearer.

### Design rules

- Represent recursive compiler structures with flat arenas and typed integer handles such
  as `Source_Id`, `Expr_Id`, `Stmt_Id`, `Decl_Id`, and `Type_Id`. Do not reproduce the C++
  tree of `variant<unique_ptr<...>>` values.
- Use explicit invalid-handle sentinels and checked access at API boundaries. An invalid
  program must produce a diagnostic, not an out-of-bounds access.
- Keep source text alive for every token, AST node, and diagnostic that refers to it.
- Treat source locations as byte offsets internally. Convert to line/column only when
  rendering or serving a tooling request.
- Return structured diagnostics from libraries. Core compiler modules must not print,
  terminate the process, or read CLI globals.
- Make traversal and emission deterministic. Sort filesystem input and never let hash-table
  iteration determine declaration, diagnostic, symbol, MIR, or object order.
- Semantic analysis owns type size, alignment, and field offsets. Backends consume layout;
  they never recompute it.
- MIR values are machine scalars. Aggregates live in addressable memory and move through
  explicit copy operations or caller-owned return storage.
- Preserve both x86 register allocators. The trivial allocator is the permanent correctness
  oracle for the linear allocator, not disposable scaffolding.
- Pass all fallible allocation and I/O through Mirage error values. Never turn recoverable
  failures into `undefined`, a nil dereference, or silent partial output.

## Public library contracts

Stabilize these behavioral contracts as their owning milestone lands. Exact field layout is
private unless another `core/mirage` module requires it.

### Source and diagnostics

- `Source_Manager` owns source bytes, assigns stable source IDs, supports in-memory overlays,
  and maps byte offsets to lines. Loading the same canonical path returns the same source.
- `Source_Provider` abstracts loading and enumerating files. Provide a host filesystem
  implementation and an in-memory implementation for tests and future editor tooling.
- `Source_Span` contains a source ID and half-open byte range. Empty and synthetic spans are
  valid and explicitly represented.
- `Diagnostic_Bag` collects severity, stage, span, message, notes, and related spans. It can
  sort deterministically, deduplicate identical reports, and render without owning stdout.

### Frontend

- Tokenization returns a token arena plus diagnostics. Tokens retain kind, byte span, and
  lexeme span; decoded literal values may live in side tables.
- Parsing returns a partial AST plus diagnostics. Recovery must always consume input or stop
  the current production, so malformed input cannot hang.
- Module resolution accepts search roots, forced modules, and a `Source_Provider`; it returns
  a deterministic module graph and search trace.
- Semantic checking returns the checked program, interned layouts, expression side tables,
  link directives, test metadata, and diagnostics. Partial data is allowed after errors for
  tooling, but code generation requires an error-free result.

### Middle end, backend, and facade

- `Mir_Module` is target-independent and can be printed, verified, and optimized without
  filesystem or process access.
- Every MIR transformation returns either verified MIR or a structured internal-error
  diagnostic. Verification also runs immediately before a backend consumes a module.
- `Compile_Request` contains the root module, target, source provider, module roots, forced
  modules, compile-time options, RTTI/init settings, optimization choice, and allocator mode.
- `Compile_Result` contains diagnostics, object or standalone-module bytes, link directives,
  compilation statistics, and success state. It does not invoke a linker.
- The external driver writes artifacts atomically, invokes the linker with an argument
  vector rather than a shell command, handles signals and `EINTR`, implements `build`, `run`,
  and `test`, and owns cleanup of temporary files.

## Core-library work required first

The existing `List`, `Hash_Map`, `String`, allocator, file, formatting, and testing modules
are useful foundations, but they are not yet a sufficient compiler platform. Finish the
following work before porting compiler logic.

Complete proposed Mirage listings for these prerequisites are indexed in
[`CORE_LIBRARY_IMPLEMENTATION.md`](CORE_LIBRARY_IMPLEMENTATION.md). They are kept as
documentation so each replacement or addition can be reviewed and applied manually.

### 1. Harden containers

- Fix probing and tombstone behavior in `Hash_Map`; in particular, `remove` must advance its
  probe index when an occupied key does not match. Correct tombstone accounting when a slot
  is reused.
- Detect integer overflow in capacity growth, `count * size_of(T)`, and string-length
  arithmetic before allocating.
- Define and test ownership for `clone`, `clear`, `remove`, and `destroy`, especially for
  values that themselves own allocations.
- Add deterministic iteration APIs for `List`, `Hash_Map`, and a new `Hash_Set`.
- Add `get_or_put`, checked reserve, and entry iteration so symbol tables and interners do not
  perform redundant probes.
- Test empty structures, collision chains, wraparound probes, tombstones, repeated growth,
  allocation failure, and destruction after partial initialization.

### 2. Add arenas and interning

- Implement a generic arena that returns stable typed integer handles and stores values in
  append-only chunks or lists. Arena growth must not invalidate handles.
- Support checked lookup, iteration in insertion order, reset, and destruction.
- Implement byte/string interning on top of an arena and hash map. Equal byte strings must
  return the same ID; IDs must remain stable until interner destruction.
- Add stress tests for growth, deduplication, embedded NUL bytes, empty strings, hash
  collisions, and allocation failure.

### 3. Complete strings and byte buffers

- Use an explicit borrowed byte view for source and symbol text, distinct by naming from the
  owning `String` type even if both currently use slices internally.
- Add searching, splitting, trimming, prefix/suffix removal, byte comparison, integer
  parsing with overflow detection, and escape decoding.
- Add `Byte_Buffer`/`Byte_Writer` with checked growth, patch-at-offset, alignment padding,
  and little-endian `u8/u16/u32/u64` emission. ELF and x86 code must share it.
- Document which APIs preserve arbitrary bytes and which require valid UTF-8. The compiler
  frontend must preserve arbitrary source bytes and diagnose invalid sequences rather than
  corrupting them.

### 4. Complete filesystem and path support

- Add read-entire-file, write-all, atomic replace, metadata/file-size queries, and directory
  enumeration.
- Add path joining, normalization, canonicalization, parent/name/extension queries, and a
  containment check that is aware of path components rather than string prefixes.
- Sort directory entries before returning compiler input.
- Add secure temporary-file creation and recursive temporary-directory test fixtures.
- Test missing files, permission errors, short reads/writes, CRLF files, `..` traversal,
  symlinks, empty paths, and paths containing spaces.

### 5. Add process support

- Spawn programs from an argument vector without shell expansion.
- Report normal exit, signal termination, spawn failure, and wait failure distinctly; retry
  interrupted waits.
- Add environment lookup and monotonic time utilities required by the driver.
- Test arguments containing spaces and metacharacters, missing executables, non-zero exits,
  and signal termination.

### 6. Strengthen testing support

- Add equality assertions for bytes, strings, slices, and structured values.
- Add table-driven test helpers, golden-file comparison, temporary filesystem fixtures, and
  deterministic diagnostic comparison.
- Where practical, add an allocator wrapper that fails on the Nth allocation so error paths
  are exercised instead of merely inspected.

**Prerequisite exit gate:** all new facilities have focused `@test` coverage compiled and
run by the C++ compiler. Container randomized tests agree with a simple reference model,
and all tests pass under both trivial and linear register allocation.

## Rewrite milestones

### Milestone 0: freeze the reference baseline

Record the C++ compiler revision, standard-library revision, supported targets, CLI options,
positive corpus outputs, negative corpus diagnostics, and normalized MIR output. Add a
differential harness capable of running the same fixture through two compiler executables.
Normalize only unstable presentation such as absolute temporary paths and elapsed time; do
not normalize token kinds, source spans, diagnostic messages, MIR operations, exit codes,
stdout, or generated object bytes.

**Exit gate:** the complete C++ suite is green; running the C++ executable against itself
through the new differential harness reports no differences.

### Milestone 1: source management and diagnostics

Implement `core/mirage/source` and `core/mirage/diagnostics`, including source overlays,
line maps, structured notes, deterministic sorting, deduplication, and text rendering. Port
the C++ diagnostic-engine unit cases before using diagnostics elsewhere.

**Exit gate:** golden output matches representative C++ diagnostics for LF, CRLF, tabs,
empty files, end-of-file spans, malformed bytes, warnings, notes, and multiple files. Every
span lookup is bounds checked.

### Milestone 2: tokenizer

Port token kinds and the normal tokenizer. Cover identifiers, keywords, numeric bases and
underscores, strings/chars and escapes, comments, attributes, directives, operators,
automatic semicolon insertion, and raw inline-assembly capture. Keep token text as borrowed
source spans and store decoded data only when needed.

Build a token-dump test executable using only `core/mirage/source`, diagnostics, and
tokenizer. Compare its canonical token stream with a C++ token-dump mode or fixture oracle.

**Exit gate:** token kind, lexeme bytes, and start/end offsets match for every repository
`.mir` file. Existing lexer robustness cases, truncated literals, arbitrary byte input, and
random mutations terminate without crashes or hangs.

### Milestone 3: AST and parser

Define handle-based AST arenas before porting parser procedures. Port in dependency order:
types and primary expressions; postfix/unary/binary expressions; initializers; statements
and blocks; declarations; generics, traits, attributes, macros, and inline assembly. Add a
canonical structural AST printer that contains kinds, important payloads, and spans but no
pointer values.

Port recovery guards explicitly. Every repetition must prove that it consumed a token or
must leave the production with a diagnostic.

**Exit gate:** canonical ASTs match on all valid fixtures. Negative parser fixtures produce
the expected primary diagnostics without hanging or unbounded cascades. Mutation tests over
representative generic, trait, error, and assembly sources always terminate.

### Milestone 4: module resolution

Port deterministic `.mir` file discovery, imports, bare imports, forced modules, module
ordering, search roots, standard-library override behavior, containment rules, cycles, and
search tracing. Keep loading behind `Source_Provider` so tests need no real filesystem and
future editor buffers can override disk sources.

**Exit gate:** module graphs, module order, canonical source paths, and search traces match
the C++ compiler for every module-resolution fixture. Escapes, symlinks, missing roots,
cycles, duplicate discovery, and empty modules have explicit tests.

### Milestone 5: declarations and symbols

Port module symbol tables, scopes, visibility, imports/namespaces, duplicate detection,
functions, globals, types, impl collection, trait impl collection, and attribute
prevalidation. Intern symbol names and use insertion-order arenas for stable diagnostics.
Expose a deterministic declaration snapshot for differential tests.

**Exit gate:** declaration snapshots and declaration-stage diagnostics match C++ over the
corpus. No later type/body checking is required to exercise this module.

### Milestone 6: resolved types and layout

Port primitive and indexed types, aliases, pointers, arrays, slices, functions, structs,
enums, unions, tagged unions, bitsets, traits, errors, generic substitutions, type interning,
and type printing. Then port size, alignment, field offsets, recursion detection, and the
target pointer-width option.

Type and layout arenas are the single source of truth for every later phase. A backend must
never inspect AST types to derive layout.

**Exit gate:** canonical resolved-type and layout snapshots match C++ for x86-64, including
recursive pointers, rejected by-value recursion, packed structs, tagged payloads, generic
instances, oversized arrays, and error unions.

### Milestone 7: constant and value resolution

Port integer/float/string literals, enum and bitset constants, `iota`, compile-time options,
environment values, constant expressions, globals, default field values, macros, and
compile-only conditions. Make overflow and narrowing decisions explicit by target type.

**Exit gate:** constant-value snapshots and diagnostics match C++ for all constant-related
fixtures, including boundary values and target-dependent options. This milestone does not
yet check function bodies.

### Milestone 8: signatures, traits, and generics

Port function and method signatures, default parameters, generic constraints and
instantiation, impl coherence/orphan checks, trait composition, trait dispatch metadata,
calling conventions, imports/exports, initialization dependencies, and test discovery.

**Exit gate:** signature/type metadata, generic instance sets, vtable plans, initialization
order, test lists, link directives, and diagnostics match C++ without MIR generation.

### Milestone 9: body semantic checking

Port local scopes, expression typing, calls, coercions, lvalues, assignments, control flow,
returns, error-state narrowing, `try`, `match`/`switch`, loops, `defer`, optional-error
handling, RTTI expressions, and inline-assembly validation. Store codegen decisions in
explicit expression side tables as the C++ compiler does; do not re-derive them in lowering.

**Exit gate:** the self-hosted frontend accepts every positive corpus fixture and rejects
every negative fixture with matching stage, severity, primary span, and stable message
substring. Canonical semantic snapshots match for representative complex programs.

### Milestone 10: MIR model, builder, verifier, and printer

Transliterate the deliberately Mirage-friendly MIR: flat vectors, `u32` handles, block
parameters, scalar values, memory-resident aggregates, explicit slots and relocations. Port
the builder, verifier, printer, signature interning, and construction-time invariants.

**Exit gate:** all hand-built MIR unit cases pass. For every verifier rule, at least one
invalid module is rejected with a deterministic error. Printing the same module twice is
byte-identical.

### Milestone 11: semantic-to-MIR lowering

Port lowering feature-by-feature but merge it as one completed milestone. Use the checked
program's types, layouts, and expression side tables. Preserve the C++ rules for aggregate
storage, caller-owned returns, trait handles/vtables, error unions, generics, RTTI, tests,
initializers, and inline assembly. Unsupported syntax must produce a named diagnostic and
must never emit a partial module as successful output.

**Exit gate:** normalized unoptimized MIR matches the C++ compiler across the complete
positive corpus, and every emitted module passes both implementations' verifier.

### Milestone 12: MIR optimization

Port slot promotion and peephole passes. Keep each pass independently selectable and verify
before and after every pass in debug/test builds.

**Exit gate:** optimized MIR is deterministic and valid; optimized and unoptimized programs
have identical corpus exit codes and stdout; pass statistics agree on pinned unit fixtures.

### Milestone 13: x86 encoder and ELF writer

Port the byte encoder bottom-up, followed by relocations, symbols, string tables, sections,
and the relocatable ELF writer. Use shared checked byte-writing utilities. Do not begin full
instruction selection until these components are independently trustworthy.

**Exit gate:** every supported instruction matches GNU `as` semantically and, where the
encoding form is fixed, byte-for-byte. Generated objects pass `readelf` inspection, link
successfully, and correctly resolve local/global function and data relocations.

### Milestone 14: x86 backend with trivial allocation

Port ABI lowering, instruction selection, frames, calls, globals, aggregate operations,
control flow, and emission using the spill-every-value trivial allocator. This isolates
backend correctness from register-allocation correctness.

**Exit gate:** all positive native fixtures link and match the C++ compiler's exit code and
stdout under trivial allocation. ABI fixtures pass in both Mirage-to-Mirage and C interop
directions.

### Milestone 15: linear allocation and inline assembly

Port interval construction, GPR/XMM classes, fixed-register kill ranges, caller/callee-save
rules, spilling, save-around-call behavior, allocation verification, and inline-assembly
emission. Preserve `--regalloc=trivial` in the public driver.

**Exit gate:** trivial and linear outputs behave identically across the full corpus. The
machine verifier rejects synthetic interference and kill-range violations. Inline-assembly
and encoder differential suites pass.

### Milestone 16: compiler facade and driver parity

Assemble the phases behind `core/mirage/compiler` without adding process concerns to the
library. Replace the placeholder `compiler/main.mir` with the thin external driver. Match
the current options and actions needed for bootstrap: `build`, `run`, `test`, recursive
`test --all`, output path, libraries, standard-library root, target, compile-time options,
forced modules, init/RTTI, MIR emission/optimization, and allocator choice.

Artifact writes must detect write and close failures. Native linking and executed programs
must report spawn, exit, signal, and wait failures correctly and clean temporary artifacts.

**Exit gate:** run the existing CLI and integration suite once with the C++ driver and once
with the Mirage driver. Results, diagnostics under test, link directives, exit codes, and
stdout match.

### Milestone 17: bootstrap to a fixed point

Use clean directories and pin the source revision, standard-library revision, target,
options, environment, linker, and linker flags.

1. Build compiler generation 1 with the C++ seed compiler.
2. Use generation 1 to compile the full Mirage compiler and its required core modules into
   generation 2, retaining the pre-link object bytes.
3. Run the complete native test suite with generation 2.
4. Use generation 2 with identical inputs to produce generation 3 and its pre-link object.
5. Require generation-2 and generation-3 compiler object bytes to be exactly identical.
6. Run the complete native test suite with generation 3.
7. Repeat the bootstrap from a clean checkout using only the archived seed compiler,
   documented host tools, and pinned sources.

Compare pre-link objects because an external linker may add host-dependent metadata. If
compiler output is split across multiple objects, compare a sorted manifest containing each
relative name and cryptographic hash. A behavioral match without an object fixed point is a
candidate compiler, not a completed self-host.

**Exit gate:** generations 2 and 3 reach the object-byte fixed point, both pass all tests,
and the clean-room bootstrap is reproducible.

### Milestone 18: transition and retire the C++ implementation

Make the Mirage compiler the default only after a soak period in which both compilers run
the differential suite. Archive a known-good seed binary, its source revision, hashes, build
instructions, and minimum host requirements. Keep the C++ repository available until the
archived-seed bootstrap has been independently repeated.

**Exit gate:** normal development and CI build the compiler with the Mirage compiler; the
C++ seed is no longer required after generation 1; documented recovery from a clean machine
has been demonstrated.

## Post-bootstrap work

After x86-64 self-hosting is stable, port targets in this order:

1. Standalone `wasm32-unknown-unknown` using the existing MIR dispatch-loop design.
2. WASI host bindings and execution tests.
3. Relocatable WebAssembly and Emscripten linking.
4. Structured-CFG emission only as an optimization after the dispatch-loop backend is the
   correctness oracle.

For every target, compare source acceptance and MIR with the native compiler first, then
compare observable execution where a host runtime exists. Do not make Wasm parity a blocker
for declaring the x86-64 compiler self-hosted.

The LSP is deliberately outside this roadmap. The source provider, partial frontend
results, stable handles, and structured diagnostics are designed so a later LSP can consume
`core/mirage` without embedding driver behavior or duplicating compiler logic.

## Working discipline

- Port behavior, not C++ syntax. Read the C++ implementation and its tests together.
- Commit one milestone or one clearly bounded sub-feature at a time, with its differential
  test in the same change.
- Keep canonical dump formats deliberately boring and stable; they are migration tools and
  future debugging interfaces.
- When C++ and Mirage differ, reduce the case before changing either implementation. Decide
  whether the difference is a Mirage bug, a C++ bug, or intentionally unspecified behavior,
  then add a regression test documenting the decision.
- Never weaken a comparison merely to make a milestone green. Normalize only values proven
  irrelevant to semantics.
- Run malformed-input and allocation-failure tests throughout the rewrite. A compiler that
  compiles itself but crashes on user mistakes is not ready to replace its seed.
- Track peak memory and compilation time at every completed milestone, but prioritize
  correctness and determinism until the fixed-point bootstrap is achieved.

## Definition of done

The compiler is fully self-hosted on x86-64 Linux when all of the following are true:

- Reusable implementation lives under `core/mirage/*`; only the driver lives under
  `compiler/`.
- The Mirage frontend, semantic analysis, MIR, x86 backend, ELF writer, and driver pass their
  unit and differential suites.
- The compiler builds the complete standard library and positive corpus and correctly
  rejects the negative corpus.
- A Mirage-built compiler rebuilds itself to byte-identical pre-link object output.
- Both fixed-point generations pass the complete suite.
- A clean bootstrap succeeds from the archived seed and documented host prerequisites.
- CI and ordinary development use the Mirage compiler as the default compiler.
