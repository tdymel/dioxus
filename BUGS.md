# Local fork - bug tracker

Local clone of DioxusLabs/dioxus. Not checked in to the main repo (see
`.gitignore`) - this is the working index of everything found, with patches
and a reproduction trail, kept up to date as the primary deliverable (no
separate report elsewhere).

`main` is tracked directly (`git fetch origin main`, fast-forwarded locally
as new commits land - currently `e2cc82e63`, version `0.8.0-alpha.1`). We
build and run against `main`, not the `v0.7.10` tag we started this fork
from: `dx` isn't tightly version-locked to the `dioxus` crate a project
depends on (it warns on a mismatch but still builds/serves correctly - see
"Building `docs` against `main`'s `dx`" below), and building against `main`
means any bug already fixed upstream (see Bug 7/9) is just... already
fixed, for free, instead of us carrying a redundant patch for it forever.
Each individual bug's own branch stays on whichever base it was actually
found/fixed against (`v0.7.10` for the earlier ones, since that's what this
fork started as) - only `fix/combined`, the one we actually build, moved.

Branches:

- `upstream-v0.7.10` - clean baseline at the `v0.7.10` tag, no changes. Kept
  for reference/reproduction - several bugs below were found and originally
  fixed against this before we moved to tracking `main`.
- `main` - tracks upstream's real `main`, no changes of our own.
- `fix/growable-table-ifunc` - Bug 1 only, rebased onto `main` (2026-08-29 -
  originally on `upstream-v0.7.10`, kept there safely until now since
  nothing depended on it moving; every bug it fixes is real on `main` too).
- `fix/symbol-index-lookup` - Bug 2 only, rebased onto `main` (2026-08-29,
  same as above).
- `fix/dropped-callgraph-edges` - Bug 4/5 only, rebased onto `main`
  (2026-08-29, same as above).
- `fix/data-symbol-overlap` - Bug 3 only, rebased onto `main` (2026-08-29,
  same as above).
- `verify/against-main` - Bugs 1/2/3/5 stacked together, rebased onto
  `main` - proof they apply cleanly there too, predates this fork moving to
  tracking `main` for real. Refreshed onto current `main` (2026-08-29).
- `enhancement/wasm-inspection-tooling` - exploratory tooling, not a bug fix.
  Rebased onto current `main` (2026-08-29).
- `fix/dangling-wbindgen-describe` - Bug 7 + Bug 8, on `upstream-v0.7.10`.
  Bug 7's half is no longer needed once building against `main` (see Bug 7
  below) - kept as-is since it's still a correct, self-contained fix for
  anyone still building against a `v0.7.10`-based `dx`. Deliberately *not*
  rebased onto `main` - its whole point is staying pinned to `v0.7.10`.
- `fix/hardcoded-ifunc-index-zero` - Bug 8 only, rebased onto `main` (Bug
  7's fix dropped - not needed there, see above).
- `fix/stale-fragment-edge-scan` - Bug 10 only, on `main` directly (found
  and fixed there, independent of everything else this fork carries).
- `fix/lto-callgraph-name-gap` - Bug 11 only, on `main` directly (found and
  fixed there, independent of everything else this fork carries).
- `fix/combined` - **what `docs/justfile`'s `dx` variable actually builds
  and what `just start`/`just start_release`/`just build_release` run
  against.** Bugs 1, 2, 3, 5, 8, 10, 11, rebased onto current `main`
  (`verify/against-main` + `fix/hardcoded-ifunc-index-zero` +
  `fix/stale-fragment-edge-scan` + `fix/lto-callgraph-name-gap`, the last
  two merged in directly rather than rebased). Bug 7 and Bug 9 are *not*
  included - `main` already doesn't have them, so there's nothing to patch.
  Previously based on `upstream-v0.7.10` with six fixes (1, 2, 3, 5, 7, 8) -
  that history is kept at `fix/combined-v0.7.10` in case a `v0.7.10`-based
  build is ever needed again.

## Building `docs` against `main`'s `dx`

Confirmed working, not just theoretically compatible: pointed a `main`-built
`dx` (isolated in a worktree, so it never touched whatever `dx serve` the
user had running at the time) straight at the real `docs`/`libero` project
(which pins `dioxus = "0.7.10"`) with `--hot-patch` - built and rendered
correctly, zero console/page errors (checked with a Playwright probe
capturing every `console`/`pageerror` event, not just eyeballing it).

One real thing to know about, not just a formality: `dx` prints this on
every run against `docs`, and it's correct, not a false positive -

```
🚫 dx and dioxus versions are incompatible!
   • dx version: 0.8.0-alpha.1
   • dioxus versions: [0.7.10]
```

It's a warning, not a hard failure - the build proceeds and the app works,
at least for everything actually exercised so far (a full boot, hot-patch
apply, basic interaction). Not yet stress-tested against the rest of
`libero`'s own feature surface (`--wasm-split` specifically, since that's
the one CLI flag most entangled with exact `dx`/`dioxus` version lockstep -
see the rest of this file). Treat "works" here as "works for what's been
tried," not a blanket guarantee across everything `docs` does.

## Bug 1 - `wasm-split-cli` panics on a growable ifunc table

`expand_ifunc_table_max()` assumed every ifunc table has an explicit
`maximum`. A table built with `--growable-table` (a flag `--wasm-split`
itself requires) has none - every call site unwrapped the `None` and
panicked. Not a rare malformed-input case: it's the table shape every
`--wasm-split` build produces.

Fix branch: `fix/growable-table-ifunc` (commit `a20322c97`). Status: fixed,
verified against a real local build and a standalone repro.

## Bug 2 - relocations resolved by name only, not by index

`get_symbol_dep_node()` resolved every function relocation by looking up the
symbol's name in a map built from walrus's parsed name/debug-info section.
The relocation's own linking-section entry carries an authoritative
`index: u32` right next to that name, and the two aren't guaranteed to
agree - observed in practice for weak/COMDAT-merged functions (shared
`core::fmt` trait impls). The old code discarded the index and silently
dropped the relocation edge on a name-lookup miss.

Fix branch: `fix/symbol-index-lookup` (commit `c4ddd544c`). Status: fixed,
verified against a real local build and a standalone repro (eliminates every
"Could not find function symbol" warning).

## Bug 3 - FIXED - data symbols silently truncated when they overlap a dead symbol's range

**Root cause found and fixed - see "Round 4: root cause found and fixed"
near the end of this section for the full mechanism, the patch, and
end-to-end verification (including with the consumer-side workaround
removed entirely).** The sections below (Round 1 through Round 3) are kept
as the historical investigation trail - several led-astray theories
(the table-index hypothesis, "it's about `Code`'s presence") are kept
un-edited for the record, with pointers to where each was superseded.

Originally: reproduces in any genuine Dioxus web app under `--wasm-split`
that both uses `libero`'s `Code` component *and* has a `format!()` call
shaped like `"...{}...{other_fmt}"` (2+ arguments, the *last* one needing a
formatter other than `Display`) - independent of route/component count
otherwise, and survives both fixes above. A `format!()` call's computed
output (e.g. a hex-formatted suffix) silently comes out truncated/missing
(every piece and argument from the last one onward vanishes, not just gets
garbled), and separately `dioxus_core::diff::VirtualDom::get_mounted_dyn_attr`
traps with `unreachable` on every load - both symptoms are present with or
without `wasm-opt` in the pipeline (tested with it skipped entirely, and
separately with it minimized to `-O0`), and the exact same trigger
reproduces with `wasm-bindgen`/`wasm-split-cli` genuinely involved but
`libero` absent replaced by a synthetic stand-in of matching size, so the
corruption is not attributable to `wasm-opt`, and not to raw scale/data
size alone either - see "Round 3" below for the full bisection trail.

**Update:** the real trigger in this codebase has been found and fixed (see
"A real instance, found and fixed" below). A later round traced a specific
instruction/relocation-type mechanism that looked like the explanation (see
"The mechanism, traced to the instruction level") - **that mechanism has
since been directly disproven** (see "Round 3: the table-index hypothesis,
disproven" below) via careful re-verification, and does not actually
explain the corruption. A subsequent round instead precisely characterized
the trigger *shape* (see "Round 3: the trigger, precisely characterized")
and bisected the *minimum program complexity* needed to reproduce it all
the way down from the full 28-route app to a single component (see "Round
3: scale bisection"). **A final round then found the actual root cause and
fixed it - see "Round 4" at the end of this section.**

### `docs_debug` / `docs_debug_min` - the reproduction environments for this bug

`docs_debug/` at the main repo's root (gitignored, a workspace member) is a
copy of `docs/` stripped down for exactly this investigation - kept around
per request rather than cleaned up after each round, so a future session
can pick this back up without re-deriving the setup. It contains the
isolated repro described below (`DiagLayer`/`diag_check` in
`docs_debug/src/main.rs`, logged via a `web-sys` console call from an
`App()` `use_effect`) with the *broken* `format!()` shape left in
deliberately, plus a wider test matrix (`log_pair` calls for cases
A through H - see "Round 3: the trigger, precisely characterized") - `just
build-dx && just start` from inside it reproduces the bug immediately
(check the browser console for `MISMATCH`).

`docs_debug_min/` (same directory level, also gitignored, also a workspace
member) is the result of bisecting *down* from `docs_debug`: no router, no
sidebar, a single real `#[wasm_split]` point, and a `main.rs` that swaps
between candidate causes by editing what `App()` renders (currently left
wired to reproduce - `LiberoProvider` + a single non-highlighting `Code`
block - see "Round 3: scale bisection" for the full progression and what
each step ruled in/out). This is the better starting point for continuing
the investigation - much faster to iterate on (single-digit-second
incremental rebuilds vs. `docs_debug`'s 30-50s) and small enough that the
`diag`-tool-based instruction/relocation techniques below are actually
tractable to apply to it directly.

Both need to stay listed in the root `Cargo.toml`'s workspace `members` to
build - since they're gitignored, a fresh clone (or a `git clean -fdx`)
will break `cargo` commands across the whole workspace until they're
recreated or removed from `members` again.

### The exact trigger, isolated (superseded - see "Round 3: the trigger, precisely characterized")

Both observations below are individually true but the framing turned out
to be incomplete - it's not "the first argument", it's "the *last*
argument", and a preceding call earlier in the list turns out not to
matter at all (see the A-H matrix in Round 3, especially cases G and H).
Kept for the historical record; use the Round 3 section for the actual
current understanding.

`format!()` truncates after the first argument specifically when
constructing that first argument's value needs *any* non-trivial codegen -
either of these alone is sufficient, tested directly against the real docs
app by swapping one expression at a time:

- a preceding function/method call, even one that's fully inlined
  (`format!("{}-{:x}", SomeEnum::X.method(), 0xAAAAu64)` truncates;
  `format!("{}-{:x}", "literal", 0xAAAAu64)` - identical shape, no call -
  does not)
- the first argument itself using `LowerHex` (or presumably any non-`Display`
  formatter), even as a bare literal with zero calls anywhere:
  `format!("{:x}-{:x}", 0xAAAAu64, 0xBBBBu64)` truncates to `"aaaa"`, and
  swapping the second argument's *type* (`0xBBBBu32` instead of `u64`, a
  different monomorphized formatter, different ifunc slot) makes no
  difference - still truncates

A plain `{}` (`Display`) first argument with no preceding call never
truncates, regardless of what the later arguments are.

### Location-independence (the big one)

The corrupting expression was moved out of application code entirely and
placed directly inside `dioxus_web::launch()` itself (`packages/web/src/launch.rs`,
top of the function, printed via `web_sys::console::log_1` so it's observable
without touching any consumer crate) - **it still truncates**, identically,
with zero docs/libero code involved in producing the value. The bug is not
about which crate the `format!()` call lives in; it's about something in the
shape of the compiled binary as a whole.

To get a diagnostic edit into dioxus-core/dioxus-web actually compiling
against the real project (rather than guessing from the outside), the whole
dioxus 0.7.10 family was `[patch.crates-io]`'d to this fork from the main
repo's root `Cargo.toml`. This doesn't work out of the box - every member's
`Cargo.toml` uses `{ workspace = true }` / `foo.workspace = true` inheritance
from *this fork's own* `[workspace.package]` / `[workspace.dependencies]`,
which Cargo can't see when the crate is referenced as a patch path from an
unrelated outer workspace (parse error: `workspace.package.version` /
`workspace.dependencies.X` "was not defined"). `delink_workspace.py` (in this
directory) mechanically rewrites a member's manifest, substituting concrete
values pulled from the fork's own workspace tables (re-anchoring any `path`
dependencies relative to the target crate's own directory) - run it across
every `packages/*/Cargo.toml` and the patch resolves cleanly. Not committed
anywhere (patches the *fork's own* manifests in place, in a directory that's
already gitignored from the main repo) - re-run it fresh next time this is
needed; nothing here depends on keeping the delinked state around.

### A real instance, found and fixed

`libero::context::libero::StylesheetRegistry::stylesheets()` built each
`<style>` tag's reconciliation key with
`format!("{}-{hash:x}", layer.css_name())` - a call (`layer.css_name()`)
followed by a `{:x}`-formatted argument, exactly the isolated trigger
shape. Logging every key produced in a real `--wasm-split` build showed
every single key for a given `CssLayer` collapsing to the *same* string
(`"lsx-framework"`, with the `-{hash:x}` suffix silently missing entirely,
not just truncated) - every stylesheet on that layer got the identical
reconciliation key. `LiberoProvider` renders one `style` element per
registered stylesheet in a keyed `for` loop
(`libero/src/context/libero/mod.rs`), so this fed real, duplicate keys
straight into `dioxus-core`'s keyed diff - the mechanism Bug 6 (below)
already flagged as an unguarded landmine in release builds. This is very
likely *the* concrete path from Bug 3's silent truncation to the separately
observed `unreachable` trap in `get_mounted_dyn_attr`, in this codebase.

Fixed by hex-encoding the hash by hand instead of through `format!`/`core::fmt`
(`push_hex` in `stylesheet_registry.rs`, a plain lookup-table loop with no
`Arguments`/formatter-trait machinery at all) - confirmed by direct A/B
rebuild-and-reload against the real `docs` app, repeatedly, in both
directions: the `format!` version traps on every load, the hand-encoded
version doesn't, and swapping back and forth flips the outcome every time.
`just start` from `docs/` (not just `docs_debug/`) now serves and
navigates all 28 routes cleanly.

This fix does not touch wasm-split-cli - it's a workaround in consumer
code for a codegen shape that miscompiles under the fork's `--wasm-split`
pipeline. The `format!()` call itself remains a latent trigger everywhere
else it might appear; this was the one place it was already firing.

### The minimal isolated reproduction

The same trigger shape, entirely disconnected from `StylesheetRegistry`
and from libero, reproduces identically: a `DiagLayer` enum with one
`const fn name(self) -> &'static str` variant, and

```rust
fn diag_check() -> String {
    let hash: u64 = 0xDEADBEEFu64;
    format!("{}-{hash:x}", DiagLayer::A.name())
}
```

called once from a `use_effect` in `docs_debug`'s `App()`, still comes out
wrong (`"diag-a"` with no suffix at all, or empty, varying slightly run to
run) under the real `--wasm-split` pipeline. This is the smallest
reproduction reached in this whole investigation - 15 lines, no routing,
no libero, no split points anywhere near the corrupting code - and it's
what made the instruction-level tracing below tractable at all: previous
rounds only had either the *whole* real app (too much to trace by hand) or
synthetic crates that never reproduced it in the first place (see "What's
been ruled out"). It's still wired into `docs_debug/src/main.rs`, broken
form, for exactly this purpose.

### The mechanism, traced to the instruction level (superseded - see below)

**This section's conclusion turned out to be wrong** - directly disproven
in "Round 3: the table-index hypothesis, disproven" further down, by
re-running the exact same comparison with `wasm-opt` minimized (`-O0`
instead of the default `-Os`) instead of skipped-after-the-fact. Kept here
un-edited because the *methodology* (the `diag` tool, the relocation-type
distinction, the `InstrLocId`-to-relocation-offset delta) is real,
reusable, and correctly executed - only the conclusion drawn from it was
premature. Read "Round 3" first if picking this up fresh.

Built a throwaway `diag` binary inside `wasm-split-cli` itself
(`packages/wasm-split/wasm-split-cli/src/bin/diag.rs`, registered as a
`[[bin]]` in that crate's `Cargo.toml` - not wired into the real `dx`/
`wasm-split-cli` build, safe to leave or delete) that loads a `.wasm` file
with `walrus` directly and can: dump a function's raw IR instructions by
name/substring match, find what table slot a given function occupies (by
name, in the *pre*-split raw rustc/wasm-bindgen output, where
`--debug-symbols` still leaves real Rust-mangled names in place), and dump
whatever function occupies a given *absolute* table slot in the *post*-
split output (where `--debug-symbols=false` strips names entirely, so this
has to work off content instead - see below).

Compiling `diag_check()` above and dumping the closure it gets inlined
into (`App`'s `use_effect` body) shows the compiled shape directly: the
hex-formatted arg's `Arguments` entry is built by writing a `self` pointer
and a **raw `i32.const` table-index** into a stack-allocated struct, then
calling `alloc::fmt::format::format_inner(out_ptr, pieces_ptr, args_ptr)`.
There is no `call`, `call_indirect`, or `ref.func` anywhere in the
function for the hex formatter - just a bare numeric literal
(`i32.const 301` for this particular build) sitting next to unrelated
constants like buffer sizes and string lengths, indistinguishable from
them by walrus's IR. Looking that slot up in the *pre*-split raw module by
name confirms it: table index 301 is exactly `<u64 as LowerHex>::fmt`.

Looking up the *same absolute slot* (301) in the *post*-split main module
that this exact build actually served: the raw constant in the closure's
code is **byte-for-byte unchanged** (still `301`), but the table itself
has been completely rebuilt - two element segments instead of one,
totalling roughly double the entries - and no function anywhere in the
post-split main module's name-searchable set is `LowerHex`-related at all
(`--debug-symbols=false` strips names in the final artifact, confirmed by
sampling: every one of 3842 functions reports `name=None`). Slot 301 does
hold a real, non-dummy, alive local function (not `wasm-split-cli`'s own
`unreachable`-bodied hole-filler, which has a distinguishably different
type - `[] -> []` vs the formatter shape's `[I32, I32] -> [I32]`), and its
body has the same *skeleton* as `LowerHex::fmt` (same 17-byte scratch
buffer, same shared hex-digit-table constant), consistent with it being
some other integer type's numeric formatter rather than a wildly unrelated
function - but nothing here can fully confirm *which* one, short of a live
in-browser call with a hand-built `Formatter` argument, which wasn't
attempted (too easy to corrupt memory and get a misleading result instead
of a clean answer).

Whichever specific function it is, the constant that's supposed to name it
is asserted, mechanically, to survive the transform unexamined:
`wasmparser`'s own relocation type enum (`RelocationEntry::ty`) already
distinguishes exactly this case - `TableIndexSleb`/`TableIndexI32`
("...Used to refer to the immediate argument of an `i32.const`
instruction, e.g. taking the address of a function") from
`FunctionIndexLeb` (used for actual `call` instructions) - so the
relocation *is* present and typed correctly in the object's `reloc.CODE`
section. `wasm-split-cli` already parses it: `ModuleWithRelocations`
builds a `relocation_map: HashMap<Node, Vec<RelocationEntry>>` covering
every relocation in every function, of every type, keyed by the function
that owns the code. But grep the crate for where `relocation_map` gets
*read* after that: nowhere. It's populated in `build_code_call_graph`/
`build_data_call_graph` and never consulted again - dead infrastructure
that already has exactly the data (`RelocationEntry::ty` +
`RelocationEntry::offset` + the target symbol's `index`) a real fix would
need, just never wired to anything that acts on it.

Wiring it up isn't a small patch, though, and it's worth being specific
about why. `walrus::ir::InstrLocId` *does* preserve each instruction's
original byte offset (`LocalFunction::original_range` and the per-
instruction `InstrLocId` are both populated straight from
`body.original_position()` during parsing) - so byte-offset correlation
between a relocation entry and a specific `Const` instruction is
mechanically possible. But `Splitter::source_module` - the module every
`emit_*` function actually mutates and emits - is parsed from `bindgened`
(the post-`wasm-bindgen` output), not from `original` (the pre-`wasm-
bindgen`, `--emit-relocs` output the relocations actually describe).
`Splitter`'s own doc comment says why in as many words: *"We need the
original around since wasm-bindgen ruins the relocation locations."* A
relocation's byte offset from `original` does not point at the
corresponding instruction in `source_module` - the two files have
different code-section layouts. This is exactly the gap `build_call_graph`
already works around for *reachability* via the `old_to_new` name-based
bridge (matching functions between `original` and `source_module` by
their mangled name, with the `descend()`/`lost_children` fallback Bug 5
fixed for when that lookup misses) - but that machinery only ever produces
a `HashSet<Node>` (which nodes are reachable), never patches an
instruction's operand. Extending it to *value-patching* means: for every
`TableIndex*` relocation in `original`, resolve its target symbol to a
`Node` in `original`'s numbering, bridge that `Node` to `source_module`'s
numbering (already solved, `old_to_new`), separately bridge the *owning
function* the same way, and only then find-and-mutate the matching
`Const` inside `source_module`'s copy of that function - by content/shape
if a `source_module`-native byte offset isn't available (parsed from a
different file, plausibly not from the same relocations at all - not yet
checked whether `bindgened` even carries its own valid `reloc.CODE`
section forward from `wasm-bindgen`, which would be the easier path if it
does), which reintroduces exactly the "how do I know it's *this*
`i32.const` and not some other one with the same value" ambiguity that
made the pure content-matching above inconclusive.

### Round 3: the table-index hypothesis, disproven

The previous round's smoking gun was: the raw `i32.const` table-index
operand for `<u64 as LowerHex>::fmt` was byte-identical (`301`, later `300`
in a slightly different build) between the pre-split object and the final
served module, even though the table itself had been completely rebuilt -
concluding the slot must now hold a *different* function. That comparison
only ever inspected the function's *entry block* (`local.block(entry)`,
never recursing into the `Loop`/`Block`/`IfElse` bodies), and the whole
served build went through `wasm-opt` at its default `-Os` first.

Redone properly this round: `WHAT_IS_AT_SLOT` now walks the full function
via `dfs_in_order` (catching every nested instruction, not just the
top-level ones), and the comparison was re-run against a build with
`wasm-opt` minimized (`Dioxus.toml`'s `[web.wasm_opt] level = "0"`, not the
same thing as skipping it - it still runs, just does far less rewriting -
but enough to remove the doubt). Result: **the function at slot 300 in the
`-O0` build is instruction-for-instruction, constant-for-constant identical
to genuine `<u64 as LowerHex>::fmt` from the pre-split object** - all 52
instructions, including the exact `Const(-1)`/`Const(-2)` forms that the
default `-Os` build had canonicalized into `Sub(1)`/`Sub(2)` (which is what
produced the earlier, misleading "2 instructions missing" observation - a
`wasm-opt` stack-vs-local optimization eliminating a redundant
store-then-immediately-reload, not evidence of anything wasm-split-cli
did). Separately, the same build's `--emit-relocs` object *and* its
post-`wasm-bindgen` copy have this same table slot pointing at the same
function by name, and their `reloc.CODE` custom sections are proven
byte-identical (see the entry below) - table-index preservation holds at
every stage checked, with wasm-opt's own rewriting no longer able to
obscure the comparison.

**And the bug still reproduces at `-O0`.** So the table reference was never
wrong; this whole line of investigation was chasing something that turned
out to be intact. (`wasm-opt` itself was re-confirmed uninvolved in a
different sense too: the corruption is identical whether it runs at `-Os`
or `-O0`, matching the original, even-earlier "skipped entirely" finding.)

### Round 3: the trigger, precisely characterized

Earlier characterizations ("a preceding call", "the first argument uses
`LowerHex`") were both true observations but incomplete - neither is
actually necessary or sufficient. Redone as a controlled matrix, all cases
built into the same binary and run side by side (`docs_debug/src/main.rs`,
functions `diag_a_display_only` through `diag_h_display_hex_display`):

| case | shape | result |
|---|---|---|
| A | `"{}", name_from_call()` | correct |
| B | `"{:x}", literal_u64` | correct |
| C | `"{hash:x}"`, hash from a local (not a call) | correct |
| D | `"{:x}-{:x}", literal, literal` | **wrong** - only first arg's output survives |
| E | `"{}-{}", "a", "b"` | correct |
| F | `"{}-{}-{}", "a", "b", "c"` | correct |
| G | `"{:x}-{}", literal, "tail"` | correct |
| H | `"{}-{:x}-{}", "head", literal, "tail"` | correct |

Any single-argument `format!()` is fine, regardless of formatter trait (A,
B, C). Any number of `Display`-only arguments is fine (E, F). A non-`Display`
argument earlier in the list, with something `Display` after it, is fine
(G, H - the hex value's own output is correct in both). The only shape that
breaks: **two or more arguments where the *last* one needs a formatter
other than `Display`** (D, and the original `diag_check`/`stylesheet_registry`
case, both `Display`-then-`LowerHex` and `LowerHex`-then-`LowerHex`). When
it breaks, everything from the last argument onward - its own formatted
output *and* the trailing pieces around it - is simply absent, not garbled;
the earlier ones print exactly as expected. Not yet tested: whether
`Debug`/`Octal`/`Binary` in the last position behave the same as
`LowerHex` (plausible, given it's clearly not `LowerHex`-specific - D's
first hex argument alone was fine - but not confirmed).

### Round 3: the `--emit-relocs`-alone test

Given `--wasm-split` requires `--emit-relocs` (a linker flag basically
nobody else uses), the next question was whether that flag *by itself* -
no `wasm-bindgen`, no `wasm-split-cli`, no Dioxus, nothing but `rustc`
compiling a tiny crate straight to a `.wasm` file - is enough to reproduce
the shape from the matrix above. Built a ~30-line standalone crate
(`cdylib`, no dependencies, `opt-level = "s"`, `lto = true`, matching the
real build's profile) exporting raw `extern "C"` functions that each run
one of the matrix's shapes and write the result into a fixed-address
buffer, read directly via `WebAssembly.instantiate` in plain Node with zero
JS glue needed. Built twice, once plain and once with
`RUSTFLAGS="-C link-args=--emit-relocs"` (different file sizes confirm the
flag took effect - relocation/linking/name sections add real bytes). Both
builds get every case in the matrix exactly right, including D. `--emit-relocs`
alone, at real-world opt level, on a trivial program, does not reproduce
this - whatever's needed, it needs more than that flag by itself.

### Round 3: scale bisection

This is the one that actually moved the needle. Starting from
`docs_debug_min` (new crate, see above) with nothing but the matrix
functions, one real `#[wasm_split]` point (kept alive across the
single-binary-crate DCE problem the same way as before - manual
`Future::poll()` with a no-op waker), and progressively adding real
project dependencies:

1. **Bare `dioxus` + `wasm-split`, no `libero`.** Every matrix case
   correct, including D. Confirms the *mechanics* of a real `--wasm-split`
   build (real `wasm-bindgen`, real `wasm-split-cli`, a real ifunc table
   with real split-point stubs) aren't sufficient either - matches the
   `--emit-relocs`-alone result, just one layer up the real stack.
2. **`+ libero`, rendering `LiberoProvider { Button { "click me" } }`.**
   Still correct. `libero`'s own theme/stylesheet machinery running (the
   same code the production fix touched) isn't sufficient by itself.
3. **`+ GettingStarted`** (one real docs page, no router, no sidebar -
   `libero::components::{Code, Divider, Flex, Text, Title}` and a couple
   of local `const` strings). **Reproduces** - D-shaped cases now come back
   wrong, exactly like the original `docs_debug` failure. This single page,
   on its own, is enough. dx's own build log shows why it's structurally
   different from step 2: a `code_highlighting` split module gets emitted
   (~1.1MB in the real `docs` app) that doesn't exist without it.
4. **Narrowed the page down to `LiberoProvider { Code { block: true,
   source: SRC, language: "rust" } }` alone** (no `Divider`/`Flex`/`Text`/
   `Title`, no prose). Still reproduces.
5. **Same, with `language` omitted** (so no actual syntax highlighting
   *runs* - `Code`'s highlighting is presumably conditional on having a
   language). Still reproduces, and the `code_highlighting` split module
   still gets emitted regardless. So it's not about highlighting
   executing; something about `Code`'s mere presence in the dependency
   graph is enough.
6. **Control: replaced `Code` with a 1.2MB static `u8` array**
   (`[0x42u8; 1_200_000]`, roughly matching `code_highlighting`'s real
   size, kept alive by summing it into a logged checksum so it can't get
   DCE'd) **instead of** `Code`, keeping everything else at step 2's
   `Button`-only shape. Correct at every case. So it's not raw static data
   size either - something more specific to `Code`/`highlight.rs`/
   `language_catalog.rs` than "a large embedded blob" or "a large function
   count" (the latter already ruled out in the previous round's 12,000-
   function synthetic crate).

Net: the minimum known-sufficient condition is now **`libero`'s `Code`
component (in any form, highlighted or not) present anywhere in the build,
alongside a `format!()` call shaped like case D/H above** - a dramatic
narrowing from "the full 28-route app" but not yet down to a single
top-to-bottom-readable cause. `Code`'s own source
(`libero/src/components/typography/code/{code,highlight,language_catalog,token_theme}.rs`,
~1865 lines total) is the next thing to bisect *within* - none of its own
`format!()` calls are multi-argument (checked directly, they're all
single-`{}`), so whatever it contributes is structural/scale-adjacent to
its presence, not a second instance of the same bug pattern living inside
`Code` itself.

### What's been ruled out, with direct evidence

- wasm-opt / binaryen post-processing (skipped entirely, corruption identical)
- `build_call_graph`'s old-module/new-module name correlation (checked for
  collisions across ~7700/~6300 named functions in the real docs build: zero)
- index-vs-name disagreement in the (already-fixed) relocation resolver
  (checked for mismatches in the real docs build: zero)
- reachability of `App()`'s own direct dependencies (every one of its ~22
  direct call targets and every data symbol it references traced through
  `emit_main_module`'s full pipeline: all correctly retained in main, right
  up to `emit_wasm()`)
- the specific `LowerHex::fmt::<u64>` instantiation `App()`'s relocation
  graph points at: confirmed alive, confirmed at both its original element-
  segment slot *and* its shared-symbols ifunc slot, unchanged through every
  stage of `emit_main_module`; separately confirmed **at actual runtime** by
  hooking `WebAssembly.instantiate` and reading `table.get(219)` directly in
  the browser - it's a real, callable function, matching the shared slot
  exactly
- `LowerHex::fmt::<u64>`'s own callees (`Formatter::pad_integral` + 2 data
  symbols, the digit-lookup tables) - all three also confirmed alive, in main
- raw framework scale, on its own: a synthetic standalone crate with 12,000
  generated functions (~5.5MB, matching the real docs binary's size) mixing
  `Display`/`LowerHex`/`Debug` `format!()` calls, run through the exact same
  patched pipeline via `--target web` in a real headless-Chromium page - the
  known-corrupting `format!("{:x}-{:x}", A, B)` pattern still comes out
  correct. Scale alone isn't sufficient.
- the mere *presence* of real split points, on its own: the same synthetic
  crate, extended with genuine `#[wasm_split]` split points (forced to
  survive dead-code elimination the same way a real, never-visited route
  does - a single binary crate's own DCE runs ahead of the linker and will
  strip an unpolled `#[wasm_split]` async fn despite the macro's export-name
  trick), escalated from 0 -> 1 -> 8 -> 28 (matching the real docs app's
  route count), the last variant with every split point returning a real
  `Element` through a helper function shared with `main` (so `shared_symbols`
  is genuinely non-empty, not the trivial always-empty case) - **still
  correct at all four counts**. Split points are necessary (0 split points
  anywhere, full framework, is the one synthetic configuration that stayed
  correct even for the location-independence test above) but not sufficient
  on their own either.
- multiple weak/COMDAT-merged copies of the same formatter monomorphization
  (a plausible variant of Bug 2's index-vs-name issue, specific to raw
  table-index constants instead of `call`s) - checked directly: exactly one
  `<u64 as LowerHex>::fmt` exists anywhere in the pre-split raw module, no
  duplicates to get confused between
- the reachability/pruning side of `emit_main_module` misclassifying the
  target function as unused and holing it out - ruled out structurally:
  `replace_segments_with_holes` only ever substitutes `make_dummy_func`'s
  distinctly-typed (`[] -> []`) placeholder, and the function actually
  sitting at the slot in question has the real formatter shape
  (`[I32, I32] -> [I32]`, matching buffer-size/digit-table constants) -
  it's alive, not a hole; whatever's wrong is specifically about *which*
  live function ends up at that slot, not about it being missing
- a build-order/runtime-initialization race (shared symbols not yet
  "reported in" by a not-yet-loaded split chunk at the moment the
  corrupting code runs) - the JS runtime glue (`__wasm_split.js`, 63
  lines) only ever *imports* the one shared `WebAssembly.Table` into side
  modules and calls into it by index; there's no JS-side `table.set()`
  patching after instantiation to investigate. Main's own element segments
  are applied once, at main's own instantiation, and nothing in the
  pipeline suggested they'd change afterward - not fully proven with a
  live before/after comparison, but no mechanism was found that would make
  this timing-dependent
- **the table-index staleness theory itself** (Round 2's headline finding)
  - see "Round 3: the table-index hypothesis, disproven" above. Directly
  disproven with a cleaner (`-O0`) comparison than the original: the
  function at the formatter's table slot is instruction-for-instruction
  identical to the genuine one, in the actual served build, and the bug
  still happens anyway
- `--emit-relocs` by itself, with no `wasm-bindgen`/`wasm-split-cli`/Dioxus
  involved at all - a minimal standalone crate at real-world opt level,
  built with and without the flag, gets every case in the trigger matrix
  right either way (see "Round 3: the `--emit-relocs`-alone test`")
- the real `--wasm-split` pipeline (`wasm-bindgen` + `wasm-split-cli`, one
  genuine split point) on a minimal Dioxus app with no `libero` at all -
  correct on every matrix case (see "Round 3: scale bisection", step 1)
- `libero` present and rendering (`LiberoProvider` + `Button`) - correct
  (step 2 of the same); ruled out `libero`'s own theme/stylesheet init
  code (the exact machinery the production fix touched) as *sufficient on
  its own* to trigger the bug elsewhere in the same binary
- raw size of an embedded static data blob, on its own - a 1.2MB `static
  [u8; N]` array standing in for `code_highlighting`'s real ~1.1MB size,
  otherwise identical to the passing `Button`-only build - still correct
  (step 6 of the same). Whatever `Code`'s presence contributes, it isn't
  reducible to "a large data segment exists somewhere in the binary"
- a second instance of the same bug pattern living inside `Code` itself -
  checked directly, every `format!()` call in
  `code.rs`/`highlight.rs`/`token_theme.rs` is single-argument

### Round 4: root cause found and fixed

Started from the one thing Round 3 flagged as never fully nailed down: the
exact ABI of the "pieces"/`Arguments`-shaped constant `format_inner` is
called with. Reading the actual std source shipped with this toolchain
(`~/.rustup/.../lib/rustlib/src/rust/library/core/src/fmt/mod.rs`, this is
`rustc 1.99.0-nightly (1a98b1e13 2026-08-07)`) turned up something Round 2
had no way to know: **this is a brand-new, nightly-only representation of
`fmt::Arguments`**, not the classic `pieces: &[&str]` slice-of-strings
model. Per that file's own header comment:

```
pub struct Arguments<'a> {
    template: NonNull<u8>,
    args: NonNull<rt::Argument<'a>>,
}
```

`template` points at a **NUL-terminated custom bytecode stream** encoding
both literal text and placeholders in one pass: a length-prefixed byte run
(`n < 128`) is a literal string piece of `n` bytes; `n == 128` is the same
for pieces over 127 bytes (a `u16` length follows); a byte with its top two
bits set (`0b11______`, i.e. `0xC0`-`0xFF`) is a placeholder, optionally
followed by flags/width/precision/arg-index fields depending on which of
the low 6 bits are set (`0xC0` alone = a fully default placeholder, no
fields); and **a single `0x00` byte marks the end of the template** - which
trait to use for a given placeholder (`Display` vs `LowerHex` vs `Debug`
etc.) is *not* encoded here at all, only in which formatter function
pointer sits in the corresponding `args` array entry.

Dumping the real compiled bytes at the `template` pointer for the two
corrupting `docs_debug_min` cases confirmed this exactly: address `1048874`
holds `[0xC0, 0x01, 0x2D, 0xC0, 0x00]` in the pre-split object (placeholder,
1-byte literal `-`, placeholder, end - a complete, valid 5-byte template)
but only `[0xC0]` followed by zeroes in the actual served, corrupting
module - **the first placeholder byte survives, and everything after it is
gone**, so `core::fmt::write`'s interpreter reads a premature `0x00` right
after formatting the first argument and stops, matching the "everything
from the last argument onward silently vanishes" symptom precisely (traced
`core::fmt::write` itself, `mod.rs` line ~1636 onward, to confirm the loop
really does terminate on any `0` byte with no distinction from a "real"
end).

Confirmed with a dedicated `SYMBOLS_AT` mode added to `diag` (parses the
`linking` custom section's symbol table directly via `wasmparser`, cross-
referencing each data symbol's declared `(segment_offset, size)` against
walrus's decoded segment addresses to get real virtual addresses) that
**the object's own symbol table has always been correct** - both the
pre-`wasm-bindgen` `original` and the post-`wasm-bindgen` `bindgened`
buffers agree: symbol `.Lanon.523f73e84932f6c50d32c580c1b0533b.87`,
`size=5`, covers exactly `[1048874, 1048879)`. So neither rustc/LLVM's
codegen nor wasm-bindgen's processing is at fault - the size is right at
every stage up through `wasm-split-cli`'s own input.

The actual bug is in `wasm-split-cli` itself, in `prune_main_symbols()`
(`packages/wasm-split/wasm-split-cli/src/lib.rs`) - the function
`emit_main_module()` uses to strip data belonging to symbols that turned
out unreachable in this particular build (as opposed to `clear_data_segments()`,
used only for split/side modules, which takes the opposite - and safe -
approach of *keeping* only bytes belonging to reachable symbols). For each
unreachable `Node::DataSymbol`, it zeroed exactly that symbol's own
declared `segment_offset..segment_offset+symbol_size` range and nothing
else:

```rust
for i in symbol.segment_offset..symbol.segment_offset + symbol.symbol_size {
    data.value[i] = 0;
}
```

This is only safe if data symbols never overlap. They can: LLVM/wasm-ld
tail-merges identical *suffixes* of NUL-terminated byte constants to save
space - exactly the optimization this new `Arguments::template` encoding
is shaped to enable. Confirmed directly in the symbol table: right next to
our own 5-byte, `[1048874, 1048879)` symbol sits an entirely separate,
distinctly-named one (`.Lanon.5e6c5f2443eb9c1abc043f8816c27a66.127`,
`size=4`) covering `[1048875, 1048879)` - a different, shorter template
(presumably for some other, differently-shaped `format!()` call elsewhere
in the crate) whose bytes are simply a suffix of ours, sharing the same
storage. In `docs_debug_min`'s build, *that* shorter symbol turns out
unreachable (nothing in the smaller binary actually uses it) while *our*
longer symbol is very much reachable - so `prune_main_symbols` correctly
leaves our symbol alone, but separately, correctly-by-its-own-logic zeroes
the dead one's `[1048875, 1048879)` range - which silently stomps the last
4 bytes of our still-live template, since the two ranges overlap. The
zeroed range matches the *shorter* dead symbol's own declared bounds
byte-for-byte, not our symbol's - confirmed by checking the actual served
bytes: `1048874` (our symbol's first byte) survives untouched, `1048875`
onward (exactly the dead symbol's range) is zero.

This also fully explains why `libero`'s `Code` component was in the
critical path for reproducing this at all (Round 3's biggest unsolved
question): `Code` doesn't interact with the corrupting `format!()` call in
any direct way - it just pulls in enough additional code and string data
(syntax highlighting tables, theme data, its own `format!()` calls
elsewhere) to shift *which* other, unrelated symbols end up dead-code-
eliminated in a given build. The bug was always latent for *any* multi-
placeholder template that happens to share tail-merged storage with some
other, shorter template that a given build's dead-code elimination
disposes of - `Code`'s presence was never causal, just one of many ways to
change the shape of what's alive vs. dead enough to hit it.

**The fix**: before zeroing any unreachable data symbol's range, compute
which bytes are still claimed by a *live* (reachable) symbol, and skip
those specifically:

```rust
// Data symbols can overlap: wasm-ld tail-merges identical suffixes of byte constants, so a
// short dead blob can be a byte-for-byte suffix of a longer, still-live one and share its
// storage. Zeroing a dead symbol's declared range is only safe where no live symbol also
// claims those bytes, so mark what's live first (segment 0 only - the only one zeroed).
let segment_len = out.data.iter().next().map(|d| d.value.len()).unwrap_or(0);
let mut live_bytes = vec![false; segment_len];
for (id, symbol) in self.data_symbols.iter() {
    if symbol.which_data_segment != 0 || unused_symbols.contains(&Node::DataSymbol(*id)) {
        continue;
    }

    let start = symbol.segment_offset.min(segment_len);
    let end = (symbol.segment_offset + symbol.symbol_size).min(segment_len);
    live_bytes[start..end].fill(true);
}
```

...then, in the existing zeroing loop, skip any byte `live_bytes` marks as
still-owned by a reachable symbol. Small, local, no architectural change -
`prune_main_symbols` already had every piece of data (`self.data_symbols`,
`unused_symbols`) needed to compute this; it just wasn't using it.

**Verification, end to end:**

- `docs_debug_min`'s three-case repro (`diag_check`, `two_hex`,
  `hex_first_display_second`): all three `MATCH` (previously `diag_check`
  and `two_hex` both `MISMATCH`, every single time this was tested across
  three rounds).
- `docs_debug`'s full A-through-H trigger matrix, run live in a browser:
  all 8 cases `MATCH`, plus the original isolated `diag_check` repro also
  `MATCH`.
- The real `docs` app (26 routes), rebuilt with the fixed `dx` and
  navigated end-to-end via Puppeteer, console/page-error-free on every
  route - with the `StylesheetRegistry` workaround (`push_hex`, see "A
  real instance, found and fixed" above) still in place.
- **The same 26-route sweep, repeated with the `StylesheetRegistry`
  workaround fully reverted back to plain `format!("{}-{hash:x}", ...)`**
  (`libero/src/context/libero/stylesheet_registry.rs`, `push_hex` deleted
  entirely) - still clean on every route. This is the real proof: the
  upstream fix alone is sufficient, with no consumer-side workaround
  needed anywhere. The workaround has been removed from `libero` as a
  result (left as an uncommitted change in the main repo, since it's the
  user's production code) - `format!()` is safe to use unconditionally in
  this codebase again, under this fork.

Fix branch: `fix/data-symbol-overlap` (commit `fb7886507`). Not (yet)
upstreamed to `DioxusLabs/wasm-split` - this fix lives on `fix/combined` in
this local fork only.

## Bug 4 - dead code silently disables the shared-chunk optimization

`build_split_chunks()`'s inner loop computes membership (`if
self.main_graph.contains(item) { continue; }`) but never inserts anything
into the map it's building - `funcs_used_by_chunks` is always empty, so
`self.chunks` always holds exactly one empty chunk. The "functions shared
across multiple split points get factored into one common chunk" mechanism
the comment describes has never actually run.

Populating it correctly is a one-line fix, but doing so on its own regresses:
`emit_split_chunk`'s import-handling doesn't account for a non-empty chunk's
own dependencies on `main_graph`/`shared_symbols`, and immediately hits a
walrus panic (`assertion failed: !self.dead.contains(&id)`, a double-delete
inside `emit_wasm()`'s own function arena). Fixing this for real needs
`emit_split_chunk` reworked to compute its own reachability the way
`emit_split_module` does for a real split point, not just reused as-is.

No branch - reverted after confirming the regression, since shipping it
alone trades a silent bug for a hard crash. Documented here so whoever picks
this up next doesn't have to rediscover the interaction.

## Bug 5 (fixed, safe) - dropped call-graph child edges

See "Bug 2"'s sibling: `build_call_graph()`'s old-to-new module correlation
recovers a *parent* node's children when the parent itself has no name match
in the new module (`lost_children`/`descend()`), but had no equivalent
fallback when a single *child* reference inside an otherwise-resolved parent
failed to resolve - that edge was dropped with no trace, silently, one line
above the panic-generating code Bug 2 fixed.

Fix branch: `fix/dropped-callgraph-edges` (commit `bd9259fbe`). Status: fixed,
confirmed safe (A/B tested against bare `fix/combined` on the real docs app:
identical output either way), but does not by itself resolve Bug 3.

## Bug 6 (flagged, not fixed) - dioxus-core silent key-collision corruption

`dioxus-core`'s keyed-list diff (`diff_keyed_middle`) has a `debug_assert!`
for unique list keys that's compiled out in release. A genuine key collision
(e.g. two `<style key="...">` entries that collapse to the same string once
Bug 3 truncates their hash suffix) silently collapses in the `HashMap` build
instead of failing loud, corrupting the diff - which is very likely the
mechanism connecting Bug 3's missing hash suffix to the separately-observed
`unreachable` trap in `get_mounted_dyn_attr`, several frames downstream, in
`dioxus-core` itself rather than in wasm-split-cli. No branch yet - flagged
as an independent robustness gap regardless of what turns out to fully
explain Bug 3.

**Update:** this is no longer just a hypothesis - "A real instance, found
and fixed" (under Bug 3) is exactly this scenario, caught live:
`StylesheetRegistry::stylesheets()`'s `format!`-built keys collapsed to one
string per `CssLayer`, feeding real duplicate keys into precisely this
`LiberoProvider` keyed `for` loop. Still no branch here for `dioxus-core`
itself - the collision no longer happens (fixed at the source, in
`libero`), but the silent `debug_assert!`-only guard is still real and
would just as easily mask a *different* future source of duplicate keys.

## Bug 7 (CONFIRMED FIXED UPSTREAM, in `main`) - hot-patch web builds throw on the very first page load

Unrelated to wasm-split-cli/Bug 1-6 above - this and Bug 8/9 below are about
`--hot-patch` (`BuildMode::Fat`) on the web target, found while chasing down
`Uncaught (in promise) TypeError: import object field
"__wbindgen_placeholder__" is not an Object` in a real browser, on a build
that had *dropped* `--wasm-split` entirely (down to plain `--hot-patch`).
Reproduces on a completely clean `upstream-v0.7.10` checkout too (confirmed
via a worktree, both `dx` and `dioxus`/`subsecond` fully vanilla) - not a
regression from any fix above, and not project-specific: it reproduces on
the official `dioxus/packages/playwright-tests/web-hot-patch` fixture with
zero edits, on the very first `dx serve --hot-patch` page load, no closures
of your own required (`dioxus-web`'s own internals already use one for
event listeners).

**Root cause:** `wasm-bindgen`'s `__wbindgen_describe`-family imports
(module `__wbindgen_placeholder__`) exist purely for its own build-time
interpreter to walk and learn type signatures from, then get stripped from
the shipped module - they're never meant to be called at runtime, and
normally *are* fully removed. On a `BuildMode::Fat` build that strip
doesn't happen - confirmed directly: dumping the compiled `.wasm`'s import
section (`WebAssembly.Module.imports()`) shows a real `__wbindgen_describe`
import under `__wbindgen_placeholder__`, called from thousands of
`WasmDescribe::describe` sites throughout the module, while the JS glue's
own `__wbg_get_imports()` has zero knowledge of that module name at all (no
occurrence of the string anywhere in the generated JS). `WebAssembly.
instantiate`/`new WebAssembly.Instance` throws immediately as a result,
before any hot-patch is ever applied - this is on the *initial* boot, not
the patching mechanism itself.

Why only fat builds: `prepare_wasm_base_module` (which only runs `if
ctx.mode == BuildMode::Fat`, see `build/web.rs`) is presumably what confuses
wasm-bindgen's own internal "describe interpreter" pass into silently
failing/skipping for this module shape - exactly *why* wasn't chased further
(would mean reading wasm-bindgen's own interpreter internals, not vendored
here), since a fix that doesn't require knowing why turned out to be
available (see below).

**Fix:** since these describe functions' return values are never inspected
by anything at runtime, a harmless stub is behaviorally indistinguishable
from the strip that should have happened. `bundle_web` (`build/web.rs`) now
runs a check right after wasm-bindgen finishes: parse the output with
`walrus`, find any imports still declared under module
`__wbindgen_placeholder__`, and if any exist, append a small JS snippet that
wraps `__wbg_get_imports` to also supply a signature-matched no-op stub for
each (return-type-aware: `0n` for an `i64` result specifically, since
`ToBigInt64(undefined)` throws where the other numeric types' `undefined`
coercions don't; `null` for a reference type; `0` otherwise). Only touches
the JS glue, never the wasm module itself - the module can have thousands
of describe call sites, so rewriting *those* would mean reimplementing
wasm-bindgen's own dead-code pass. A no-op (detects nothing, appends
nothing) on every build that doesn't hit this - confirmed via `wasm-split.
spec.js` (a real `--wasm-split --release` build, no hot-patch) still passing
unchanged.

Our own fix - branch `fix/dangling-wbindgen-describe` (commit `4bc20b4e`).
Status: fixed there, verified against the official `web-hot-patch` fixture
(both this fork and a clean `upstream-v0.7.10` worktree) and against the
real `docs` app - both now boot cleanly under `--hot-patch` with zero
console/page errors, confirmed via a Playwright probe capturing every
`console`/`pageerror` event.

**Update: already fixed upstream, independent of our own fix above.**
While rebasing this fork's fixes onto `main` (see the top-level Branches
list), testing an *unpatched* `main` checkout directly - no fixes of ours
at all - showed the official `web-hot-patch` fixture booting clean, zero
edits, zero errors. Bisected to
[#5622, "Fix hotpatching wasm with new v0 symbol mangling in beta linux
builds"](https://github.com/DioxusLabs/dioxus/pull/5622) (`git log
upstream-v0.7.10..main -- packages/cli/src/build/patch.rs` narrows it down
directly) - the same root cause we found independently:
`name_is_bindgen_symbol`'s pattern list only recognized legacy Rust name
mangling, not the `v0` scheme that's been the rustc default since 1.97, so
on a modern toolchain the *thousands* of `WasmDescribe::describe` call
sites throughout a module went unrecognized, got incorrectly promoted into
the ifunc table by `prepare_wasm_base_module`, and that's what pinned the
whole describe machinery in place and kept wasm-bindgen's GC from ever
removing it. Same mechanism, same fix shape (recognize more mangled name
shapes) - upstream just found it first, in a commit that landed after the
`v0.7.10` tag was cut.

Given that, `fix/combined` (which now tracks `main` - see the top-level
Branches list) does **not** include our own fix for this - there's nothing
left to patch. `fix/dangling-wbindgen-describe` is kept as-is, since it's
still a correct, independent fix for anyone building against a
`v0.7.10`-based `dx`.

## Bug 8 (fixed in our fork, still present in `main`) - hot-patched hardcoded ifunc index 0

Found immediately after fixing Bug 7 above, still on the official
`web-hot-patch` fixture: the initial load now works, but applying a real
hot-patch (editing `src/main.rs` while `dx serve --hot-patch` is running)
crashed with a wasm trap, `RuntimeError: null function or function
signature mismatch`, inside `__wbindgen_describe` - reached via an indirect
call (`call_indirect`) from deep inside the patch module's own
`WasmDescribe::describe` call chain. The fixture's own edit (`num += 1` ->
`num += 2`, inside its `onclick` closure body) reproduces it; a
content-only edit elsewhere in the same function (outside any closure
body - see Bug 9's own repro below, which uses exactly that kind of edit
to get *past* this bug) does not. So the trigger looks to be specifically
patching a function that is itself a closure used as an event handler, not
merely touching a component that happens to have one - `dioxus-web`'s own
internal event-listener closures are enough on their own, no explicit
`Closure` of your own required, but only when *their* code is what
changed.

**Root cause:** `create_wasm_jump_table` (`build/patch.rs`) has two call
sites that retarget one of these same `__wbindgen_describe`-family symbols
(the `wbg_funcs` loop, and `env_funcs`' `name_is_bindgen_symbol` branch) by
converting them to an indirect call through the ifunc table - but both
hardcode the table index to `0`, instead of looking one up the way every
*other* case in this function does (via `name_to_ifunc_old`). That's not a
wrong index - it's a nonexistent one: `prepare_wasm_base_module` (see Bug 7)
deliberately never promotes `name_is_bindgen_symbol` names into the ifunc
table in the first place, on the (per Bug 7, false) assumption wasm-bindgen
will already have stripped them - so there was never a real index to look
up. Depending on what happens to occupy slot `0`, this either traps outright
(a null table entry) or - worse - would silently call the wrong function if
slot `0`'s type happened to match.

**Fix:** same reasoning as Bug 7's fix - these functions are dead code at
runtime regardless of what's calling them, so instead of trying to find a
valid index to call through (there isn't one), both call sites now build an
inert local function body directly (a new `stub_out_bindgen_describe_fn`,
mirroring `convert_func_to_ifunc_call`'s structure but emitting
result-shaped zero/null constants instead of a `call_indirect`) - no table
index needed at all. The zero constant is emitted per `ValType`, exhaustively
(`i32`/`i64`/`f32`/`f64`/`v128.const 0`, `ref.null` for a reference type):
falling back to an `i32.const 0` for a result type that isn't `i32` would
emit a type-invalid module rather than an inert one.

**Update: confirmed via direct instrumentation that this is still real,
live, executing code on unpatched `main` today** - not made moot by #5622
(Bug 7's upstream fix) the way Bug 9 turned out to be. Added `eprintln!`
tracing at both `convert_func_to_ifunc_call(..., 0, ...)` call sites in a
`main`-based `dx` build and ran it against several real hot-patch edits:
the hardcoded-`0` path fires every single time, 5 times per closure-body
edit in the smallest test case, including for `__wbindgen_describe` itself.
It just doesn't currently *visibly* misbehave on `main`, in any of the
apps tried (the official fixture, our real `docs` app, two from-scratch
minimal apps) - ifunc slot `0` happens to be occupied by something
type-compatible enough that the wrong `call_indirect` doesn't trap. That
looks like it's a property of typical ifunc table construction order
(describe-shaped functions - trivial, no-argument signatures - seem to
consistently end up early in the table) rather than anything guaranteed,
which is exactly the kind of latent bug worth flagging before an unlucky
build order turns it back into a visible crash.

**Minimal reproduction, isolated from Bug 7's fix:** a from-scratch 15-line
app (`main.rs` below) reliably traps with the exact `RuntimeError: null
function or function signature mismatch` shown above, at the exact
`__wbindgen_describe` call site, when patched with `dx` built from
`upstream-v0.7.10` **plus only Bug 7's fix** (needed just to get past
initial load - unpatched `v0.7.10` can't reach this bug at all, since Bug 7
crashes first). This isolates Bug 8 as an independent defect, not just a
symptom of Bug 7:

```rust
// Cargo.toml deps: dioxus (path dep, features = ["web"]), wasm-bindgen = "0.2.114",
// web-sys = { version = "*", features = ["MouseEvent"] }
use dioxus::prelude::*;

fn app() -> Element {
    let mut num = use_signal(|| 0);

    let _closures = wasm_bindgen::closure::Closure::<dyn FnMut(web_sys::MouseEvent)>::new(
        move |_event: web_sys::MouseEvent| {},
    );

    rsx! {
        button {
            id: "btn",
            onclick: move |_| num += 1,
            "Count: {num}"
        }
    }
}

fn main() {
    dioxus::launch(app);
}
```

Steps: `dx serve --hot-patch`, load the page, then edit `num += 1` to
`num += 2` while the server is running. Trap fires immediately on save.

Branch: `fix/hardcoded-ifunc-index-zero` (commit `df5361cda`), rebased onto `main` (Bug 7's half
of the original combined fix dropped - not needed there). Merged into
`fix/combined`. Status: fixed there, verified three ways: (1) the specific
trap from the minimal repro above is gone when patched against
`fix/combined`'s `dx` instead of a Bug-7-only one, (2) the official
`web-patch.spec.js` still passes end-to-end against `fix/combined`'s `dx`
(no regression from the fix), (3) the `wasm-split.spec.js` regression test
(unrelated `--wasm-split --release` build, no hot-patch at all) also still
passes unchanged.

## Bug 9 (CONFIRMED FIXED UPSTREAM, in `main`) - hot-patched closures: `undefined` externref

Unblocked by Bug 8's fix rather than found independently: with Bug 7/8
fixed, the *same* official `web-hot-patch` fixture's own edit (`num += 1`
-> `num += 2` inside its `onclick` closure body, plus a button-text change
outside it) still fails, later, with a different, non-trapping JS error: `TypeError:
Cannot read properties of undefined (reading '_wbg_cb_unref')`, from
wasm-bindgen's generated `__wbg__wbg_cb_unref_...` shim
(`arg0._wbg_cb_unref()` where `arg0` - an externref argument - is
`undefined`), reached via a `Closure`/`JsClosure` drop-glue chain. Confirmed
this is the last blocker on the official test as of Bug 7/8's fix: without
this, `web-patch.spec.js` fails at its very first `toContainText` assertion
(page never even loads); with Bug 7/8 but not this, it gets past that and
the first click, and fails at the *first real edit's* content-update
assertion instead - i.e. this is the next real thing standing between this
fork and a fully green `web-patch.spec.js`.

**Working theory, not yet confirmed:** `subsecond::apply_patch`'s wasm
branch (`packages/subsecond/subsecond/src/lib.rs`) explicitly grows and
shares two things between the main module and a patch - linear `memory`
(`memory.grow`) and the ifunc/`funcs` table (`funcs.grow`) - so patched code
can address the same memory and call through the same indirect-call table
as the main module. Newer `wasm-bindgen` (reference-types-enabled) output
keeps `externref`s in a *separate* table from the ifunc one (visible in the
generated JS as `wasm.__wbindgen_externrefs`, populated by
`__wbindgen_init_externref_table`). `apply_patch` never touches that second
table at all - it's never grown, shared, or otherwise told about the patch.
If a patch's own externref-table entries land at indices that don't
correspond to anything in the *main* module's externref table (or the
patch gets its own separate table entirely), a `Closure`'s wrapper object -
looked up by table index across that boundary - would resolve to nothing,
exactly matching the observed `undefined`. Not yet verified by reading
`wasm-bindgen`'s own externref-table codegen or instrumenting the actual
table identity/indices at the moment of failure - flagged here rather than
fixed blind, since a wrong fix in this territory (shared mutable table
state between two live wasm instances) risks trading a loud crash for silent
memory corruption.

No fork branch (was never fixed by us). Reproduction, for the record: apply
Bug 7+8's `fix/dangling-wbindgen-describe` (or `fix/combined-v0.7.10`,
the old `v0.7.10`-based combined branch), then run `web-patch.spec.js`
against the official `web-hot-patch` fixture - fails at line 59
(`toContainText("Click button! Count: 1", ...)`) with the error above in
the browser console (dx's own devserver mirrors it server-side too, via
`patch_console.js`'s `monkeyPatchConsole`, as `ERROR wasm-bindgen: imported
JS function that was not marked as \`catch\` threw an error: Cannot read
properties of undefined (reading '_wbg_cb_unref')`). The same from-scratch
15-line app from Bug 8's repro reproduces this independently too, patched
against a `dx` with *both* Bug 7 and Bug 8's fixes applied (so it's not
just Bug 8 wearing a different hat) - confirming this is a real, separate,
third bug, not a downstream symptom of the other two.

**Update: confirmed already fixed upstream, same as Bug 7 - most likely the
same commit.** Testing an unpatched `main` checkout directly: the official
`web-patch.spec.js` passes fully, end to end (`1 passed`), including the
exact edit that reproduces this on `v0.7.10` - no separate fix needed, and
none attempted. The likely mechanism (not separately confirmed by reading
wasm-bindgen's own codegen, but consistent with what changed): `main`'s
`HotpatchModuleCache::new` gained a fallback merge - `symbol_ifunc_map`
now also checks `direct_name_to_ifunc` (built straight from `Function::
name`, i.e. `collect_func_ifuncs`) whenever the usual linking-section
indirection comes up empty. `__wbg_cb_unref`'s own `__saved_wbg_`-renamed
wrapper (built by `prepare_wasm_base_module`, same mechanism as the
describe intrinsics) is exactly the shape this fallback is built to catch,
which would resolve it through the *correct* real ifunc index instead of
either the working-theory externref-table gap above or Bug 8's hardcoded-0
path - both of which this symbol would otherwise be at the mercy of. The
externref-table-sharing theory above was never independently confirmed
either way, and no longer needs to be, now that the symptom it was trying
to explain is gone.

Not included in `fix/combined` (nothing to include - see Bug 7).

## Bug 10 (fixed in our fork) - stale sibling mount panics `dioxus-core`'s placement resolver

Unrelated to Bug 1-9 above (`dioxus-core` itself, not wasm-split-cli or
hot-patch) - found in the real `docs`/`libero` project (not a fork-local
repro to start with), as `index out of bounds: the len is 0 but the index
is 0` panicking at `packages/core/src/mount.rs`'s `Mount::dynamic_slot`,
reliably, on the very first real interaction that portals a component into
existence: `libero`'s `Drawer` (`variant="temporary"`) calls a `use_portal`
hook that writes to a shared signal *during its own render* (registering
its rendered content into a separate `PortalOutlet` elsewhere in the tree)
and then returns `rsx! {}` itself - the same crash also reproduces via a
plain `if open() { Modal { ... } }` toggle with no portal involved at all,
so the portal isn't the trigger, just one of several diff shapes that hit
it.

**Root cause:** `EdgeScan` (`packages/core/src/diff/node.rs`) already
existed specifically for this class of problem - its own doc comment says
`find_first`/`find_last` trust every *live* mount, while placement sibling
scans should read the *committed* component mount view, since a sibling
being placement-resolved can itself be mid-diff with transiently incoherent
live state. In practice the distinction wasn't actually being honored
everywhere it needed to be:

1. Two real placement/sibling-scan call sites -
   `PlacementResolver::resolve_fragment_child_site`'s `following`/`previous`
   closures (`diff/placement.rs`) and `StableFragmentEdges::new`'s
   keyed-reorder edge computation (`diff/iterator.rs`) - called the plain
   `find_first_element`/`find_last_element`, which unconditionally build
   `EdgeScan::live` internally. Only `adjacent_dynamic_sibling_{before,after}
   _in_vnode` (`diff/placement.rs`), a third, unrelated sibling-scan
   function, was already using the committed variant.
2. Even where `EdgeScan::placement` (committed) was correctly selected at
   the top of a walk, `dynamic_anchor_edge_element` threw it away on every
   recursive step: it took a `target_id` (not a full `EdgeScan`) and always
   rebuilt `EdgeScan::live(target_id)` fresh, so descending into a single
   nested `Fragment` inside an otherwise-committed walk silently reverted to
   trusting live (possibly mid-diff) mount state for everything beneath it.
   `dynamic_node_edge_element`'s own recursive calls (its `Fragment` and
   `Component` match arms) had the same problem one level further in: they
   destructured `scan` into a bare `target_id` early and passed only that
   back into `find_element_in_roots`, dropping `committed_component_view`
   on every subsequent level of nesting.
3. Even with (1) and (2) fixed, `mounted_dynamic_component_scope` →
   `Mount::dynamic_slot` still indexed its backing slice unconditionally
   (`self.slots[self.dynamic_offset() + idx]`). A concurrently-mid-diff
   sibling's mount table can legitimately not have that many dynamic slots
   yet (a mount that's about to render nothing, or hasn't had its slots
   populated for this pass yet, both look like zero slots either way) - the
   committed-view fetch (`current_mounted_view`) re-reads the mount's
   *node*, but never re-validates that the specific `idx` the caller is
   about to look up in `Mount`'s *slots* array is actually in range for it,
   so a transiently-behind mount could still be indexed straight past its
   own slots and panic.

None of these are wasm-split-cli/build-pipeline concerns - this is a plain
in-memory `dioxus-core` diffing bug, reachable on desktop and any other
renderer, not just web; it was only *found* here because portaling a
component during render happens to produce exactly the right concurrent
"one scope newly inserted while a sibling scope is also mid-diff in the
same pass" shape to hit it.

**Fix**, all three parts, `packages/core/src/{mount.rs,diff/node.rs,diff/
placement.rs,diff/iterator.rs}`:

1. `VNode` gained `find_first_element_committed`/`find_last_element_committed`
   (same shape as the existing `find_first_element`/`find_last_element`, just
   building `EdgeScan::placement(dom)` instead of `EdgeScan::live(...)`) and
   `resolve_fragment_child_site`/`StableFragmentEdges::new` were switched to
   them. `find_first_element`/`find_last_element` themselves are untouched
   for their own remaining callers (`component.rs`/`suspense/component.rs`'s
   self-referential "where did my own just-replaced content live" lookups,
   and `placement.rs`'s `vnode_first_site`, which is the same self-lookup
   shape one level up) - those read a scope's *own*, not-yet-replaced state
   right before mutating it, not a sibling's, so `EdgeScan::live` remains
   correct there.
2. `find_element_in_roots`, `root_child_edge_element`, and
   `dynamic_anchor_edge_element` now take the full `EdgeScan` (not a bare
   `target_id`) and thread it straight through every recursive call,
   including `dynamic_node_edge_element`'s `Fragment`/`Component` arms -
   `committed_component_view` (and `target_id`, still read off `scan` where
   a bare id is needed) now survives arbitrarily deep fragment/component
   nesting instead of resetting to `live` one level down.
3. `Mount::dynamic_slot` is now bounds-checked
   (`self.slots.get(...).copied().unwrap_or(PackedMountedSlot::empty())`)
   instead of a raw index - `PackedMountedSlot::empty()` is already this
   module's own established "nothing mounted here" value (what a genuinely
   empty component's slot table already returns for `.component_scope()`/
   `.text()`/etc.), so this doesn't introduce a new sentinel, just extends
   the same "not found" case to also cover "this mount doesn't have that
   many slots at all" instead of panicking. All four readers of
   `dynamic_slot` already return `Option` or are otherwise fallible-shaped,
   so nothing has to invent a meaning for the fallback. The write paths
   (`dynamic_slot_mut`, `anchor_slot_mut`) and the anchor read path
   (`anchor_slot`) are untouched, since a bad index there would be a real,
   load-bearing invariant violation worth panicking on, not a best-effort
   edge-scan reaching a sibling that isn't fully populated yet.

   **Caveat, stated honestly:** this part was derived by reading the code
   path, not by re-measuring the panic with (1) and (2) already applied.
   Because the trigger is scheduling-dependent and never reduced to a
   portable repro (see the writeup's "Minimal reproduction" section), there
   is no cheap experiment that separates "still load-bearing" from "now
   redundant belt-and-braces". It is kept because the cost of being wrong
   in that direction is a hard panic in a renderer-agnostic diffing path,
   and the cost of it being redundant is one bounds check on four
   already-fallible reads.

Removing `find_last_element`'s last caller in (1) left the method itself
unused; it is deleted rather than left to warn. `find_first_element` keeps
its own remaining self-lookup callers.

**Verification:**

- The real `docs`/`libero` app's own two repro pages (`/overlay/drawer`,
  `/overlay/modal`), driven end-to-end with Puppeteer (real click, not just
  page load) against `fix/combined`'s rebuilt `dx`: zero panics across 3
  repeated runs each, both before-crash-every-time on the unpatched build
  and after-clean-every-time on the patched one. The Drawer's own
  open/close round trip was also checked functionally, not just
  "didn't crash" - its content renders and un-renders correctly across the
  click.
- A full sweep of all 27 routes in the real `docs` app (fresh page load per
  route) plus targeted interaction on the ones most likely to share this
  diffing machinery (`Image`'s zoom portal, `Tree` keyboard navigation,
  `Select`, `DataList`'s keyed list) - zero console/page errors anywhere,
  confirming the bounds-checked `dynamic_slot` fallback isn't masking
  anything on the paths that were already working.
- The official `wasm-split.spec.js` and `web-patch.spec.js` Playwright
  suites both still pass unchanged against `fix/combined`'s rebuilt `dx` -
  this fix touches shared diffing code those tests also exercise, and
  neither regressed.

Branch: `fix/stale-fragment-edge-scan` (commit `ceef831a1`), based on
`main` (bug found and fixed there directly, independent of everything else
this fork carries). Merged into `fix/combined`. Status: fixed, verified as
above.

## Bug 11 (fixed in our fork) - `wasm-split-cli` deletes a still-called main-module function under fat LTO

Found investigating why the real `docs`/`libero` project's release profile
(`opt-level = "z"`, `lto = true`, `codegen-units = 1`, `strip = true`) -
found by hand, not by us, while chasing a separate bundle-size
investigation - crashes `dx build --wasm-split` every time:
`walrus`'s own internal assertion, `assertion failed:
!self.dead.contains(&id)`, panicking inside `walrus::passes::gc::run`
during `Splitter::emit_main_module`. Reproduces reliably with that exact
profile; a plain `opt-level = "z"` + `strip = true` build (no LTO, default
`codegen-units`) does not hit it.

**Root cause:** `main_roots()` (`wasm-split-cli/src/lib.rs`) - the set of
functions `unused_main_symbols()`/`prune_main_symbols()` treat as
definitely-reachable-from-main - only considers the module's own exports,
its start function, and its imports. `build_call_graph()` separately
computes a fallback for two cases the old-module/new-module name
reconciliation can't line up: edges recovered after being dropped mid-graph
(`recovered_children`), and wasm-bindgen-*synthesized* functions with no
old-module counterpart at all (`new_names` entries missing from
`old_names`) - primarily the per-signature `invoke*` closure-invocation
shims wasm-bindgen generates, which JS calls back into via a function-table
slot `Closure::wrap` captured, never via a traced `call`/`ref.func`
instruction anywhere in the compiled Rust code. Both cases are meant to be
attached as children of `main` - the code's own comments say so ("we're
going to attach the recovered children to the main function" / "attach any
truly new symbols to the main function. Usually these are the shim
functions"). That attachment wrote into `new_call_graph`, a local variable
distinct from `self.call_graph` (the one actually already fully assigned,
earlier in the same function, from the reconciled `original.call_graph`,
and the only one `main_graph`/`shared_symbols`/pruning ever read from).
`new_call_graph` is never read again after being built and never merged
into `self.call_graph` - the entire mechanism was a silent no-op, for both
cases, on every build.

Concretely: an `invoke*` shim with no incoming edge in `self.call_graph`
has no path from any root in the graph reachability is actually computed
from - it's a graph island. It survives pruning only by luck:
`unused_main_symbols()` only considers functions reachable from *some split
point*, so a function nothing in the graph points to is simply never a
deletion candidate in the first place, not proven live by anything. A real,
still-referenced closure body the shim calls, though, *can* be reachable
from a split point's subtree (an unrelated route's own component tree,
also registering event-handler closures) - and with nothing recording that
the shim (unresolvable by name, but very much still compiled in and still
calling it) also keeps it alive from main, `unused_main_symbols()` wrongly
classifies it as deletable. `prune_main_symbols()` deletes it; the shim's
own `call` instruction into it is left untouched (`delete()` doesn't
rewrite references pointing at what it deletes - documented as the
caller's responsibility). `walrus::passes::gc::run`, walking that
still-live `call` to determine reachability during its own pass, then
indexes the already-dead function id and hits its internal
double-free-style assertion.

Fat LTO + a single codegen unit doesn't introduce this gap - it's present
in every build, including ones that never hit it - but makes actually
*triggering* it far more likely: whole-program visibility lets the linker
collapse many originally-distinct closures onto far fewer, far-more-widely-
shared `invoke*` monomorphizations, each with much higher fan-out into real
closure bodies, which raises the odds that at least one such callee also
happens to be reachable from some split point's subtree. A normal
multi-codegen-unit, non-LTO build has the exact same graph-modeling gap; it
just tends to produce many more, much narrower-fan-out shims, so the
odds of a collision in any one build stay low in practice (not zero - this
is a build-shape probability, not something LTO specifically causes).

**Investigation note:** an earlier attempt fixed the symptom by rooting
*every* function referenced by any element (function table) segment, not
just the recovered/truly-new ones `build_call_graph()` already intended to
handle. That's far too broad for a real Dioxus app: `dioxus-core`'s own
component dispatch and rsx! event-handler closures also go through the
function table pervasively, so it silently defeated route-level
`wasm-split` entirely instead of fixing the underlying gap - all 36 of the
real `docs` app's route chunks collapsed to ~1KB stubs each, with
everything pulled back into the (now much larger) main bundle. Not applied
- kept here so nobody re-derives and ships this by accident chasing the
same symptom.

**Fix**, `packages/wasm-split/wasm-split-cli/src/lib.rs`, `build_call_graph()`:
write to `self.call_graph` directly instead of the separate, discarded
`new_call_graph` (which is now removed entirely - the `HashMap` it built up
was never read from anywhere else either).

**Verification:**

- Before the fix: `dx build --platform web --release --debug-symbols=false
  --wasm-split --features libero/wasm-split` against the real `docs` app with
  `opt-level = "z"` + `lto = true` + `codegen-units = 1` + `strip = true`
  in `Cargo.toml`'s `[profile.release]` panics on every run, deterministically
  (confirmed 2 consecutive clean builds, identical panic).
- After the fix: the same build succeeds, deterministically (confirmed 2
  consecutive clean builds from an emptied output directory, byte-identical
  main-bundle hash both times).
- Route-level splitting genuinely still works, not just "didn't crash" -
  all 35 route chunks present with realistic sizes (e.g. `CodePage`
  126,263 bytes, `TreePage` 121,412 bytes - in line with this same app's
  pre-LTO route sizes), totalling 1,764,871 bytes, clearly distinct from
  the "everything collapsed into main" failure mode the discarded earlier
  attempt above produced.
- Main bundle: 1,121,978 bytes - smaller than this app's best prior
  `wasm-split`-only measurement (1,177,438 bytes, no `opt-level`/`lto`/
  `strip` tuning) and smaller than the `opt-level = "z"` + `strip = true`
  (no LTO) measurement that motivated trying LTO in the first place
  (1,150,875 bytes) - i.e. the full profile combination now works and pays
  off, not just compiles.

Note on the sizes above: they are all `--debug-symbols=false` builds. Drop
that flag and every wasm carries its `name` section (+510,153 bytes on main
alone), which is why an otherwise identical build reports ~1.63 MB instead.

Branch: `fix/lto-callgraph-name-gap` (commit `3f6e5a950`), based on `main`
(bug found and fixed there directly). Merged into `fix/combined`. Status:
fixed, verified as above.

## Review pass, 2026-08-29 - all fixes re-read and tightened

Every fix on `fix/combined` was re-read end to end and each branch amended
in place (one commit per fix, as before). No fix was found to be wrong;
these are simplifications, one real defect, and one cleanup. The bug
sections above have been updated to match the code as it now stands.

- **Bug 2 (`fix/symbol-index-lookup`)** re-implemented `parse_module_with_ids`'s
  `Arc<RwLock<_>>`/`on_parse` plumbing inline in `ModuleWithRelocations::new`
  to build its index table. It now calls that helper, which was already
  right there in the same file and already returns exactly this vector.
  60/17 added/removed lines became 29/10.
- **Bug 3 (`fix/data-symbol-overlap`)** built its liveness bitmap correctly
  but verbosely (`.iter().nth(0)`, a redundant `.min(end)` on a range start
  that Rust already treats as empty, an eight-line comment). Same behavior,
  written as a `fill(true)` over a bounded slice.
- **Bug 5 (`fix/dropped-callgraph-edges`)** unchanged apart from comment
  length.
- **Bug 8 (`fix/hardcoded-ifunc-index-zero`)** had a real defect:
  `stub_out_bindgen_describe_fn`'s `match` on the result type handled
  `i64`/`f32`/`f64`/`ref`, then fell through to `i32.const 0` for
  *everything else* - which includes `v128`. A `v128`-returning describe
  intrinsic would have produced a type-invalid module rather than an inert
  one. The match is now exhaustive over `ValType`, with `v128.const 0` for
  the vector case. (No such intrinsic is known to exist, so this was latent,
  not observed.)
- **Bug 10 (`fix/stale-fragment-edge-scan`)** switched the last caller of
  `VNode::find_last_element` to the committed variant and left the method
  behind, so `dioxus-core` built with a `dead_code` warning. The method is
  deleted. Part 3's caveat (see the bug section) is now stated explicitly
  rather than asserted.
- **Bug 11 (`fix/lto-callgraph-name-gap`)** unchanged apart from comment
  length.
- **`feat/vec-children`** detected a `children: Vec<Element>` field by
  comparing the field type against four hardcoded spellings. A field written
  any other way (`std::vec::Vec<dioxus::prelude::Element>`, say) fell
  through to the `children: Element` default and failed to compile with a
  confusing error. It now checks the type structurally, the way
  `type_from_inside_option` next to it already does, and
  `packages/core/tests/vec_children.rs` gained a case that fails to compile
  without it.

**Verification:** `cargo test -p dioxus-core` passes (12 `vec_children`
cases among them, up from 11), `dioxus-core` and `dioxus-cli` build without
warnings, and the real `docs` app's full LTO + `--wasm-split` release build
produces **byte-identical output** to the pre-review `dx`: main bundle
1,121,978 bytes, 35 route chunks totalling 1,764,871 bytes, both matching
Bug 11's recorded figures exactly. The wasm-split-cli changes are therefore
behavior-preserving on a real workload, not just plausibly so.
