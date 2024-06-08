# Core WebAssembly Module Build Targets Explainer

For any given WIT [`world`], the Component Model specification defines multiple
**Core WebAssembly module build targets** that all logically represent the same
world. The different module build targets provide Core WebAssembly producer
toolchains different low-level representation choices that don't change the
high-level runtime semantics of the `world`.

## Motivation

While the Component Model's [binary format] provides more [linking] and
optimization options than the module build targets listed below and is thus
strictly more expressive, there are several reasons for having the Component
Model define Core WebAssembly module build targets that use the existing
standard [Core WebAssembly Binary Format]:
* It allows *interface authors* (e.g., in [WASI]) to write their interface
  just once (in WIT) for both Core WebAssembly and Component runtimes.
* It allows existing *Core WebAssembly producer toolchains* to more-easily
  incrementally target the Component Model. Once a producer toolchain supports
  one module build target, they're much closer to being able to take advantage
  of the full Component Model.
* It allows existing *Core WebAssembly runtimes* to partially support the
  Component Model and WIT-defined interfaces (like [WASI]) by natively
  implementing a module build targets. While this restricts the set of content
  that the runtime can execute, this can help understand and motivate the
  additional work required to support the full Component Model.

Importantly, any WebAssembly module targeting a Component Model build target
can be trivially wrapped (e.g., by [`wasm-tools component`]) to become a
semantically-equivalent component. Thus, these modules can be considered
"simple components", which is why these build targets are defined in the
Component Model spec even though they are just Core WebAssembly modules.

## List of Build Targets

Currently, there is only one module build target defined:
* `wasm32-wasip2`: uses one 32-bit linear memory with the Canonical ABI
  as defined in the [WASI Preview 2] stability milestone.

Other build targets can be added as needed in the future. In particular, the
following additional build targets are anticipated:
* `wasm64-wasip2`: uses a 64-bit linear memory (based on [memory64]).
* `wasmgc-wasip2`: uses managed memory (based on [wasm-gc]).
* `wasm(32|64|gc)-wasip3`: offers async variants of all imports and exports and
  enables newer WIT features like `stream`.

When the [shared-everything-threads] proposal stabilizes, it could either be
added backwards-compatibly to existing build targets or added as a separate
build target.

## Target World

The rest of this document assumes a single, fixed "target world". This target
world determinse a fixed list of import and export names, each with associated
component-level types. For example, if a runtime wants to support "all of WASI
Preview 2", the target world would (currently) be:
```wit
world all-of-WASI {
  include wasi:http/proxy@0.2.0;
  include wasi:cli/command@0.2.0;
}
```
However, a runtime can also choose to support just `wasi:http/proxy` or
`wasi:cli/command` or a subset of one of these worlds: this is a choice every
producer and consumer gets to make independently. WASI and the Component Model
only establish a fixed set of names that *MAY* appear in imports and exports
and, when present, what the agreed-upon semantics *MUST* be.

TODO: mention pre-processed resolution of `use` and `include` and WIT gates
and that really we're talking about WAT here.

## Top-Level

Every build target defines the following set of imports (defined below):
 1. [WIT-derived function imports](#wit-derived-imports)

and the union of the following sets of exports (also defined below):
 1. [WIT-derived function exports](#wit-derived-exports)
 2. [Memory exports](#memory-exports)
 3. [Initialization export](#initialization-export)

TODO: for a given target world, reject if import outside this set, but accept
if export outside this set.

The names of WIT-derived function imports and exports are automatically
synthesized from the names as they appear in WIT. To avoid clashes between the
open-ended set of WIT names, a `_` prefix is used (below) for all built-in
function names, since WIT names cannot and will not contain `_`.

The runtime behavior of the WIT-derived imports and exports is entirely defined
by the [Canonical ABI], as explained in detail below.

The only additional global runtime behavioral rule that does not strictly
follow from the Canonical ABI is:
 * Modules *MUST NOT* call any imports during the Core WebAssembly [`start`]
   function that access memory (as defined by [`needs_memory`], given the
   import's type). Hosts *MUST* trap eagerly (at the start of the import call)
   in this case.

This rule allows modules to be run in a wide variety of core runtimes
(including browsers) which do not expose memory until after the `start`
function returns. This matches the general Core WebAssembly toolchain
convention that `start` functions should only be used for module-internal
initialization. General-purpose initialization code that may call imports
may instead run in the [`_initialize` function](#initialization-export).

Two other runtime behaviors that are captured by the Canonical ABI that are
worth stating explicitly here are:
 * The host may not reenter guest wasm code on the same callstack (i.e., once
   a wasm module calls a host import, the host may not recursively call back
   into the calling wasm module instance).
 * Once a module instance traps, the host *MUST NOT* execute any code with
   that module instance in the future.
 * Any WIT-derived export *MAY* be called repeatedly and in any order by the
   host. Producer toolchains *MUST* handle this case without trapping or
   triggering Undefined Behavior (but *MAY* eagerly return an error value
   according to the declared return type).

The following subsections specify the sets of imports and exports enumerated
above:

### Name Canonicalization

Given an [`importname`] or [`exportname`] `n`, the **canonicalization** of `n`
is given by:
* If `n` is a [`plainname`], the canonicalized name is `<n>_p2`.
* Otherwise, if `n` is an [`interfacename`] with no trailing `@<version>`, the
  canonicalized name is also `<n>_p2`.
* Otherwise:
  * the canonicalized name is `<base>@<canonicalized-version>_p2` where:
    * `n` is split into `<base>@<version>`,
    * `version` is split into `<major>.<minor>.<patch>` followed by the optional
      `-<prerelease>` and `+<build>` fields, as defined by [SemVer 2.0], and
    * `canonicalized-version` is:
      * if the optional `prerelease` field is present:
        * `<major>.<minor>.<patch>-<prerelease>`
      * otherwise, if `major` and `minor` are `0`:
        * `0.0.<patch>`
      * otherwise, if `major` is `0`:
        * `0.<minor>`
      * otherwise:
        * `<major>`.

TODO: depnames

For example:

| `importname` or `exportname` | Canonicalized name       |
| ---------------------------- | ------------------------ |
| `a`                          | `a_p2`                   |
| `a:b/c`                      | `a:b/c_p2`               |
| `a:b/c@1.2.3+alpha`          | `a:b/c@1_p2`             |
| `a:b/c@0.1.2+alpha`          | `a:b/c@0.1_p2`           |
| `a:b/c@0.0.1+alpha`          | `a:b/c@0.0.1_p2`         |
| `a:b/c@1.2.3-nightly+alpha`  | `a:b/c@1.2.3-nightly_p2` |

The reason for this version canonicalization is to avoid requiring every
runtime to implement this fuzzy matching so that all imports of names in the
left column (of the table above) match if they have the same right column.
Instead, producer toolchains do this at build time so that host import
resolution can use a simple string table of imports, as is usual for Core
WebAssembly runtimes.

TODO: explain why `_p2`

### Memory Exports

If [`needs_memory`] is true for the type of any import or export in the target
world, then the following export *MUST* be present:

* `(export "_memory_p2" (memory 0))`

This exported linear memory is used as the `memory` field of [`canonopt`] in
the Canonical ABI.

If [`needs_realloc`] is true for the type of any import or export in the target
world, then the following export *MUST* be present:

* `(export "_realloc_p2" (func (param i32 i32 i32 i32) (result i32)))`

This exported allocation function is used as the `realloc` field of
[`canonopt`] in the Canonical ABI.

### Build Target ABI Options

The Canonical ABI is parameterized by a small set of ABI options
([`canonopt`]). The `memory` and `realloc` options are already defined
[above](#memory-exports).

Additionally:
* The `string-encoding` is fixed to `"utf8"`.
* The (unstable Preview 3) `async` field is fixed to `True`.
* The `post-return` option for a WIT function named `fn` is set to be the
  exported `_post_*` function described [below](#wit-derived-function-exports),
  if present (otherwise `None`).

When `gc` and `memory64` fields are added to `canonopt`, they would be
mentioned here and configured by the build target.

### Implicit Component Instance and Task

The Canonical ABI maintains per-component-instance spec-level state that
affects the lifting and lowering of parameters and results in imports and
exports. As described above, Core WebAssembly build targets treat each Core
WebAssembly module as a simple component, and thus there is always an
**implicit component instance** created for each Core WebAssembly module
instance that is implicitly passed to each Canonical ABI import.

Canonical ABI imports also take an **implicit task** which contains per-call
spec-level state used to dynamically enforce the rules for reentrance,
`borrow`, and, in a Preview 3 timeframe, async calls). Until Preview 3 async is
enabled for a module build target, there is only ever *at most one* task per
instance (corresponding to an active synchronous host-to-wasm call, noting that
host → wasm → host → wasm reentrance is disallowed) and thus the **implicit
component instance** mentioned above also implies the **implicit task**.

### WIT-derived Function Imports

The Core WebAssembly function *imports* derived from the target world are
defined as follows.

* For each import in the target world with [canonicalized] name `n` and type `t`:
  * If `t` is a WIT interface:
    * For each function in `t` with name `fn` and type `ft`, the build target includes:
      * `(import "<n>" "<fn>" <flat-type>)`, where:
        * `flat-type` is [`flatten_functype`]`(ft, 'lower')` and
        * the runtime behavior is defined by [`canon_lower`], given the
          [build target's ABI options] and the [implicit component instance and task].
    * For each *original* resource type in `t` with name `rn`, the build target includes:
      * `(import "<n>" "_drop_<rn>" (func (param i32)))`, where:
        * the runtime behavior is defined by [`canon_resource_drop`], given the
          [build target's ABI options] and the [implicit component instance and task].
  * Otherwise, if `t` is a function type, the build target includes:
    * `(import "" "<n>" <flat-type>)`, where:
      * `flat-type` and the runtime behavior are defined as in the above
        interface case.
* For each export in the target world with [canonicalized] name `n` and type `t`:
  * If `t` is a WIT interface:
    * For each *original* resource type in `t` with name `rn`, the build target includes:
      * `(import "<n>" "_drop_<rn>" (func (param i32)))`,
      * `(import "<n>" "_new_<rn>" (func (param i32) (result i32)))`, and
      * `(import "<n>" "_rep_<rn>" (func (param i32) (result i32)))`, where:
        * the runtime behavior is defined by [`canon_resource_drop`],
          [`canon_resource_new`] and [`canon_resource_rep`], resp., all given
          the [build target's ABI options] and the [implicit component instance and task].

Above, the word "original" in *original* resource type means a resource type
that isn't simply definitionally equal to another resource type (using `type
foo = bar` in WIT).

### WIT-derived Function Exports

The Core WebAssembly function *exports* derived from the target world are
defined as follows. These exports are not *required* to exist (if they are
absent, they simply won't be called), but if an export *is* present with the
given name, it *MUST* have the given type.

* For each export in the target world with [canonicalized] name `n` and type `t`:
  * If `t` is a WIT interface:
    * For each function in `t` with name `fn` and type `ft`, the build target includes:
      * `(export "<n>_<fn>" <flat-type>)`
      * `(export "_post_<n>_<fn>" (func (params <flat-params>)))`, where:
        * `flat-type` is [`flatten_functype`]`(ft, 'lift')`;
        * `flat-params` is [`flatten_functype`]`(ft, 'lift').results`; and
        * the runtime behavior is defined by [`canon_lift`], given the
          [build target's ABI options], the [implicit component instance],
          `None` as the `caller`, a no-op `on_block` callback, an `on_start`
          callback that provides the export's arguments and an `on_return`
          callback that accepts the export's results.
    * For each *original* resource type in `t` with name `rn`, the build target includes:
      * `(export "_dtor_<n>_<rn>" (func (param i32)))`, where:
        * this destructor is called by the host to allow the guest to release
          any private allocations associated with an owned handle previously
          returned by the host; and
        * the `i32` parameter is the `i32` "rep" value passed to the
          `_new_<rn>` import when creating the resource that is now being
          destroyed.
  * Otherwise, if `t` is a function type, the build target includes:
    * `(export "<n>" <flat-type>)`
    * `(export "_post_<n>" (func (params <flat-params>)))`, where:
      * `flat-type`, `flat-params` and the runtime behavior are defined as
        in the above interface case.

As mentioned above, producer toolchains *MUST* allow its exported non-`_post`
functions to be called 0..N times in any order. (A `_post_<f>` function *MUST*
only be called immediately after a preceding `f` call.)

### Initialization Export

*Every* module build target includes the following optional Core WebAssembly
function export which, if present, will be called:

* `(export "_initialize_p2" (func))`

A producer toolchain can rely on its initialization function being called *at
some point in time* before any other export call. As mentioned above, unlike
during the Core WebAssembly `start` function, producer toolchains *MAY* call
any import during `_initialize_p2`.

## Example

Given the following world:
```wit
package ns:pkg@0.2.1;

interface i {
  resource r {
    constructor(s: string);
    m: func() -> string;
  }
  frob: func(in: r) -> r;
}

interface j {
  use i.{r};
  resource s {
    constructor(i: u32);
    m: func() -> u32;
  }
  frob: func(in: s) -> s;
}

world w {
  import i;
  import f: func() -> string;
  export j;
  export g: func() -> string;
}
```
the `wasm32-wasip2` build target includes the following imports and exports:
```wat
(module
  (import "ns:pkg/i@0.2_p2" "[constructor]r" (func (param i32 i32)))
  (import "ns:pkg/i@0.2_p2" "[method]r.m" (func (param i32)))
  (import "ns:pkg/i@0.2_p2" "frob" (func (result i32) (result i32)))
  (import "ns:pkg/i@0.2_p2" "_drop_r" (func (param i32)))
  (import "" "f_p2" (func (param i32)))
  (import "ns:pkg/j@0.2_p2" "_drop_s" (func (param i32)))
  (import "ns:pkg/j@0.2_p2" "_new_s" (func (param i32) (result i32)))
  (import "ns:pkg/j@0.2_p2" "_rep_s" (func (param i32) (result i32)))
  (export "_memory_p2" (memory 0))
  (export "_realloc_p2" (func (param i32 i32 i32 i32) (result i32)))
  (export "ns:pkg/j@0.2_[constructor]s_p2" (func (param i32)))
  (export "ns:pkg/j@0.2_[method]s.m_p2" (func (result i32)))
  (export "ns:pkg/j@0.2_frob_p2" (func (result i32) (result i32)))
  (export "_dtor_ns:pkg/j@1_s_p2" (func (param i32)))
  (export "g_p2" (func (result i32)))
  (export "_post_g_p2" (func (param i32)))
  (export "_initialize_p2" (func))
)
```

## Relation to compiler flags

TODO:
* wasi-libc-based:
  * `--target=wasm32-wasip2`
  * Controls generation of core wasm, doesn't yet fix module-vs-component
  * Component-aware toolchains emit components by default (e.g., invoking wasm-component-ld)
    * But can disable via linker flag:
      * e.g.: `-Wl,--emit-module` in wasi-sdk or `-Clink-arg=--emit-module` in Rust
* Go:
  * GOARCH = wasm32
  * GOOS = wasip2
  * `-buildmode=module|component`


[Canonicalized]: #name-canonicalization
[Build Target's ABI Options]: #build-target-abi-options
[Implicit Component Instance]: #implicit-component-instance-and-task
[Implicit Component Instance and Task]: #implicit-component-instance-and-task

[Binary Format]: Binary.md
[Canonical ABI]: CanonoicalABI.md
[`needs_memory`]: CanonicalABI.md#TODO
[`needs_realloc`]: CanonicalABI.md#TODO
[`flatten_functype`]: CanonicalABI.md#flattening
[`canon_lower`]: CanonicalABI.md#canon-lower
[`canon_resource_drop`]: CanonicalABI.md#canon-resourcedrop
[`canon_resource_new`]: CanonicalABI.md#canon-resourcenew
[`canon_resource_rep`]: CanonicalABI.md#canon-resourcerep
[AST Explaiern]: Explainer.md
[`canonopt`]: Explainer.md#canonical-abi
[Lifting and Lowering Definitions]: Explainer.md#canonical-abi
[`importname`]: Explainer.md#import-and-export-definitions
[`exportname`]: Explainer.md#import-and-export-definitions
[`plainname`]: Explainer.md#import-and-export-definitions
[`interfacename`]: Explainer.md#import-and-export-definitions
[`canonopt`]: Explainer.md#canonical-abi
[Linking]: Linking.md
[`world`]: WIT.md#wit-worlds

[Core WebAssembly Binary Format]: https://webassembly.github.io/spec/core/binary/index.html
[`start`]: https://webassembly.github.io/spec/core/syntax/modules.html#start-function
[`import`]: https://webassembly.github.io/spec/core/syntax/modules.html#imports
[`export`]: https://webassembly.github.io/spec/core/syntax/modules.html#exports
[`name`]: https://webassembly.github.io/spec/core/syntax/values.html#syntax-name

[Memory64]: https://github.com/webAssembly/memory64
[wasm-GC]: https://github.com/WebAssembly/gc
[shared-everything-threads]: https://github.com/webAssembly/shared-everything-threads
[WASI]: https://github.com/webAssembly/wasi
[WASI Preview 2]: https://github.com/WebAssembly/WASI/tree/main/preview2
[`wasi:cli/run`]: https://github.com/WebAssembly/wasi-cli/blob/main/wit/run.wit

[`wasm-tools`]: https://github.com/bytecodealliance/wasm-tools
[`wasm-tools component`]: https://github.com/bytecodealliance/wasm-tools#tools-included

[SemVer 2.0]: https://semver.org/spec/v2.0.0.html
