# Mirage language and library reference

Read this reference when writing `.mir` code or reviewing module APIs. Confirm uncertain details against the current sources and compiler tests.

## Modules and imports

- A module is a directory. Every `.mir` file directly in it contributes to the same module scope.
- Bare `#import("runtime")` imports expose public declarations as unqualified names; `#import("path") as name` binds a namespace.
- A namespace import preserves qualification:

```mirage
#import("core/collections/list") as list

mut values := list.make_list[i32]()
defer values.destroy()
try values.push(4)
```

- Prefer namespace qualification for generic libraries. Keep the type and constructor qualified from the same namespace.
- Relative sibling imports may use `#import("../sibling")`. Use `--print-module-search` to diagnose resolution.

## Core syntax

```mirage
const immutable := 10
mut inferred := 0
mut explicit: usize = 0
mut zeroed: Thing = default
mut storage: [64]u8 = undefined

pub type Pair = struct {
    left: i32
    right: i32 = 0
}

pub type State = enum(u8) {
    Empty
    Occupied
}

pub fn add[T: type](a: T, b: T) -> T {
    return a + b
}
```

- Use `*T` for pointers, `ptr.*` for dereference, `&value` for address, `[]T` for slices, and `[N]T` for arrays.
- Cast with `cast(value, Type)`. Pointer-to-slice casts may include a length: `cast(ptr, []u8, length)`.
- Methods are declared in `impl Type[...]`; mutating methods conventionally receive `self` and update the referenced object.
- Use `defer resource.destroy()` for owned resources. A deferred conditional is valid where used by existing code.
- `anyptr` is the untyped pointer used by allocators, memory routines, reflection, and generic callbacks.

## Errors

Mirage error unions use enum members as error kinds:

```mirage
pub type Widget_Error = error(Allocator_Error_Kind | Widget_Error_Kind)
pub type Widget_Error_Kind = enum {
    Invalid_Argument
}

pub fn create() -> (Widget, ?Widget_Error) {
    const memory := try allocator.alloc(size)
    return_ok widget
}
```

- Use `return_ok` or `return_ok values...` for success.
- Use `return_err .Variant` for failure.
- `try` propagates the error and unwraps successful value returns.
- A returned error value is truthy on failure and false on success. Capture it when testing expected failure.
- Do not assign a multi-result fallible call to one value unless dropping the other results is explicitly correct; accidental error dropping may panic.
- Include every propagated error-kind enum in the caller's declared error union.

## Collections and ownership

- Constructors generally accept an optional allocator and return an empty, non-owning value until first allocation.
- `nil_allocator()` is the standard deterministic allocation-failure seam.
- `clear` normally retains capacity and releases logical elements; `destroy` frees storage and resets pointer, length/count, and capacity.
- Before growing, check `required > MAX_USIZE - current`. Before byte allocation, check element-count multiplication against `MAX_USIZE / size_of(T)`.
- Copy overlapping ranges with `mem.move`; use `mem.copy` only where overlap is impossible.
- When returning a live slice/view, its lifetime and invalidation follow the owning container.

## Testing details

- `@test` functions take no parameters and return exactly an `error(...)` type.
- Tests in the same module can access module-scope private declarations; prefer public behavior unless private access is needed for a precise invariant.
- `mirage test module` includes tests from the module/import graph. `--all` recursively adds descendant modules.
- Each case runs in a child process. `FAILED` means the test returned an error; `CRASHED` means abnormal termination.
- Slices do not support equality operators. Use a helper:

```mirage
fn equal_bytes(a: []u8, b: []u8) -> bool {
    if len(a) != len(b) {
        return false
    }
    for i in ..len(a) {
        if a[i] != b[i] {
            return false
        }
    }
    return true
}
```

## Sources of truth

- Read core APIs under `core/` in the standard-library repository.
- Read language examples and CLI guidance in the Mirage-Cpp `README.md`.
- Read detailed semantics in Mirage-Cpp `docs/spec.md` and `docs/grammar.md` when present.
- Search compiler regression tests under Mirage-Cpp `tests/` before assuming unsupported syntax is intended.
