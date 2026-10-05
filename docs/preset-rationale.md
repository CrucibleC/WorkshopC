[← Previous: Tag headers](tag-headers.md)

# Preset rationale

WorkshopC's rules can be combined in many ways. The eight built-in presets are the combinations the project recommends for common situations. This page explains the reasoning behind them: the principles they all share, how severities are chosen, why each preset sets its rules the way it does, and how the presets compare.

What each rule does is described in [Rules](rules.md), and how to select and adapt a preset in [Configuration](configuration.md).

## Contents

- [Shared principles](#shared-principles)
- [How severity is chosen](#how-severity-is-chosen)
- [The adoption path](#the-adoption-path)
  - [`adopt`](#adopt)
  - [`adopt-more`](#adopt-more)
  - [`adopt-even-more`](#adopt-even-more)
- [`default`](#default)
- [`opinionated`](#opinionated)
- [`strict`](#strict)
- [`embedded`](#embedded)
- [`nevernull`](#nevernull)
- [Comparison](#comparison)
  - [Rule levels](#rule-levels)
  - [Notable options](#notable-options)

## Shared principles

Some decisions are the same in every preset:

- **Exceptions always need a reason.** The suppression reason rule is an error in every preset, including the first adoption step. A suppression without a reason is an exception that nobody can review later.
- **Opt-in guarantees are errors from day one.** The private alternative and reference pointer rules only check code that uses their tags, so they cost an existing codebase nothing, while code that does opt in gets the full guarantee immediately.
- **Enums are allowed, but typed.** Every preset sets `allow_enum_typedef: true`. Enums are the natural way to express states and modes, and the enum rule removes their danger by requiring typedefs and explicit conversions.
- **Arguments stay what they were given.** `forbid_arg_reassign` and `forbid_mut_arg_pointer` are on everywhere. A function whose arguments never change is easier to follow, and the change is cheap to make.
- **Globals stay in their file.** Every preset requires globals to be `static` and forbids static variables in headers. Globals may still change, since `must_be_const` is off everywhere, but their state stays local to one file.
- **References need no null check.** Every preset exempts reference parameters from the null check rule, since they can never be null.
- **The same lifecycle names.** Every preset uses `_pod`, `_make`, `_copy`, `_move`, `_destroy`, `_return`, `_valid` and `_destroy_array`, and requires raii structs to be moved and destroyed through the address of a variable. Code keeps its names when a project changes preset. Only the free struct suffix differs, see [`opinionated`](#opinionated).
- **Namespaces start at `src`.** Every preset that checks namespaces starts them at `src`, and derives include guards from the file path.
- **Vendor code is left alone.** All presets share the same list of common third-party folder names, and [`embedded`](#embedded) adds the usual vendor and SDK folders.

## How severity is chosen

The level of a rule says how much a finding matters:

- **Error** for rules that prevent defects: null dereferences, leaks, double frees, use after move, unhandled states, ignored failures, name collisions and buffer overruns. Rules that only apply to code that opted in are errors too.
- **Warning** for rules that are about readability and consistency rather than correctness: typedefs, single return, array struct naming, const fields and `goto`. A warning still asks for a fix, but it does not have to block a build, and [`--warnings-as-errors`](usage.md#output-options) can make it block where that is wanted.
- **Off** for rules that a situation does not call for yet, mainly in the early adoption steps, where the structural rules would mean rewriting large parts of a codebase at once.

The presets lean one way or the other from this baseline: the adoption presets start with more warnings, while `opinionated` and `strict` make style rules errors too.

## The adoption path

The three adoption presets bring an existing codebase toward the full WorkshopC style in steps. Each step is meant to be cleared before moving on to the next, and the steps add rules in a deliberate order: first the checks that find real bugs with little rewriting, then the rules that change APIs and naming, and last the rules that change how ownership and resources are expressed.

### `adopt`

> Existing-project starting point: adopt the core checks first, then move to `adopt-more` once these findings are resolved.

| | Rules |
|-|-------|
| **Error** | Suppression reason, private alternative, reference pointer: all of them only apply to code that opts in |
| **Warning** | Enum, private, assignment (without the null bans), null check, interfaces, strict switch, global variable (no naming requirements), span struct (shape and counts only), const field, no goto |
| **Off** | Function pointer, typedef struct, prefix namespace, argument pointer movement, struct resource management, restricted malloc, single return, function discard, array struct |

- **Real bugs first, with small fixes.** Null checks find unsafe dereferences while still allowing short `if (ptr)` checks. The enum rule catches accidental mixing of integers and enums while keeping existing enum types. Strict switch catches missing states and accidental fallthrough. All of them are warnings, so they can be cleared gradually.
- **Intent without upheaval.** The assignment rule makes initialization and argument intent clearer, without yet banning common null patterns. Private fields stay behind a small set of accessors.
- **No large renames yet.** Globals get internal linkage, but naming prefixes are not required in the first step.
- **Groundwork for later steps.** Spans are introduced as sized views before raw arrays are restricted in later presets, and interfaces keep object dispatch consistent while the heavier ownership rules are still off.
- **Visible exceptions.** Suppressions are common during adoption, and a short reason keeps each temporary exception visible in reviews.

### `adopt-more`

> Second adoption step: after `adopt` is clean, strengthen APIs and naming while leaving full resource-lifecycle enforcement for `adopt-even-more`.

Changes from `adopt`:
- **Now errors:** enum, private, assignment, null check, interfaces, strict switch and global variable.
- **New warnings:** function pointer typedefs, prefix namespace (include guards only, names are not checked yet), single return and function discard.
- **Stricter spans:** arrays may only be passed to library functions and spans, and every array needs a span right after it.

The reasoning:
- **Clean stays clean.** With the core checks cleared in `adopt`, they become errors: no implicit mixing of enums and integers, no dereference of a nullable pointer without a check, no unhandled states or fallthrough.
- **Readable APIs.** Callback types are named once, so public signatures stay readable, and returned status and error results are no longer silently ignored.
- **Naming, incrementally.** Module naming starts as a warning and only applies to include guards, where e.g. function names are unaffected because an existing codebase cannot be expected to rename everything.
- **Sizes travel with buffers.** Buffer pointers are paired with their sizes before stricter span use is enforced in the next step.

### `adopt-even-more`

> Final adoption step: explicit ownership, resource lifetimes and sized buffers. Existing projects can stay here.

Changes from `adopt-more`:
- **New warnings:** argument pointer movement (tags on declarations, no call site operators yet), struct resource management, restricted malloc and array struct naming.
- **Now errors:** prefix namespace (still include guards only), function discard and span struct. Static spans may be initialized with braces.

The reasoning:
- **Ownership becomes visible.** Every pointer parameter states whether it takes ownership, writes a result or changes data. Call sites are not marked yet, which keeps the change to existing calls small.
- **Resources get lifetimes.** Construction, transfer and cleanup of owned resources are checked, so that every resource has a clear lifetime, and allocation moves behind a few wrapper functions.
- **Results are handled.** Callers must handle returned status and error information.
- **Raw data stays contained.** Buffer sizes stay attached to their data, and the ways raw data can escape are restricted.

**Why existing projects can stay here.** `default` is meant for new projects. What remains between `adopt-even-more` and `default` (call site operators, namespaced names, global naming conventions, typedefs for every struct, and errors instead of warnings for ownership, resources and allocation) would be a large refactor of an existing codebase for comparatively little extra safety. Ambitious projects can first make `adopt-even-more` clean with `--warnings-as-errors`.

## `default`

> Recommended starting point for new projects: approachable warnings with strong defaults for ownership, pointers and resource safety.

`default` is the preset the rest of the documentation is written around. Every rule is on: rules that protect correctness are errors, and rules about readability are warnings.

| Rule | Level | Settings | Why |
|------|-------|----------|-----|
| [Suppression reason](rules.md#suppression-reason-rule) | Error | | Intentional exceptions must be explainable during review. |
| [Enum](rules.md#enum-rule) | Error | Typedef enums allowed | Enum values stay explicit, with no accidental integer conversions. |
| [Private](rules.md#private-rule) | Error | `_private` fields, `private_` accessors | Implementation details stay behind accessor functions. |
| [Private alternative](rules.md#private-alternative-rule) | Error | | Only applies to tagged fields, which then get the full guarantee. |
| [Function pointer](rules.md#function-pointer-rule) | Warning | | A typedef keeps callback signatures readable, a matter of readability rather than correctness. |
| [Typedef struct](rules.md#typedef-struct-rule) | Warning | | Consistent type names. A missing typedef costs readability, not correctness. |
| [Assignment](rules.md#assignment-rule) | Error | Arguments protected, null allowed | Initialization and the intent behind pointer arguments are deliberate. |
| [Prefix namespace](rules.md#prefix-namespace-rule) | Error | `src`, `_` separator, up to 5 folders, any case | Public names reflect their module path, which prevents name collisions. |
| [Null check](rules.md#null-check-rule) | Error | `if (ptr)` accepted | Dereferences that a null argument can reach are caught. |
| [Argument pointer movement](rules.md#argument-pointer-movement-rule) | Error | `give()` and `overwrite()` required, `lend()` not used | Ownership transfers and outputs stand out at every call site, while ordinary modification stays quiet. |
| [Struct resource management](rules.md#struct-resource-management-rule-raii) | Error | Reverse destruction, standardized destroy functions, no use after destroy | Resource-owning structs get explicit construction and cleanup lifetimes. |
| [Interfaces](rules.md#interfaces-rule) | Error | | Object dispatch layouts stay consistent and inspectable. |
| [Restricted malloc](rules.md#restricted-malloc-rule) | Error | `memory_alloc`, `memory_alloc_array`, `memory_free` | Allocation is centralized behind the project's memory wrappers. |
| [Single return](rules.md#single-return-rule) | Warning | Guard clauses allowed, `return;` optional in `void` functions | One exit point is easier to follow, but guard clauses are often clearer, so they are allowed. |
| [Strict switch](rules.md#strict-switch-rule) | Error | | Every state and branch is handled explicitly. |
| [Global variable](rules.md#global-variable-rule) | Error | Static, `g_` and `s_` prefixes, constants in capitals | Globals stay file-local, and mutable state is easy to spot. |
| [Reference pointer](rules.md#reference-pointer-rule) | Error | Exempt from null checks | Parameters that always hold an object are separated from nullable pointers. |
| [Function discard](rules.md#function-discard-rule) | Error | | Status and error results are never silently ignored. |
| [Array struct](rules.md#array-struct-rule) | Warning | Naming only, arrays allowed anywhere | Spans already guard array sizes, so array structs only get consistent names. |
| [Span struct](rules.md#span-struct-rule) | Error | A span after every array, raw data only to library functions, string literals allowed for `const char*` | Buffer lengths stay coupled to their data, which makes bounds easy to verify. A string literal carries its own terminator and can not be written through `const char*`, so it needs no span. |
| [Const field](rules.md#const-field-rule) | Warning | | Keeps whole-object initialization and assignment possible, a matter of convenience rather than safety. |
| [No goto](rules.md#no-goto-rule) | Warning | | Structured control flow is easier to follow, but a `goto` is not a defect in itself. |

## `opinionated`

> The default preset with the author's preferred severities and naming style: fail early on core safety rules while keeping warnings for selected style checks.

`opinionated` starts from `default` and changes it in three ways:

- **Stricter about style.** Function pointer and struct typedefs, const fields and `goto` are errors, and null checks must be written out as comparisons, so that an unchecked dereference is rejected and every check is explicit.
- **More relaxed about raii details.** The destruction order of raii structs is free, destroy functions are not standardized, and a raii struct may be used after it was destroyed. The private rule is a warning. Lifecycle functions and cleanup are still required for every owned resource, which keeps lifetimes predictable without dictating how a destroy function is written.
- **Its own naming style.** Namespace prefixes have no separator and use at most three folders, which suits CamelCase names such as `UiButtonCreate` for a function declared in `src/ui/button/`. Private accessors contain `pget` and `pset`, interfaces end with `_port` and `_cport`, const spans with `_cspan`, flexible arrays use `flex`, and free structs are created with `_initialize_manually`. Restricted malloc allows `memory_alloc` and `memory_free`, a small, auditable set of wrappers, and spans are not required after every array, which keeps bounds next to the data without restricting every array declaration.

## `strict`

> High-adherence preset: safety rules are errors, and ownership, lifetimes, naming and buffer boundaries are explicit throughout the project.

Changes from `default`:
- **Every rule is an error**, including the style rules, and `void` functions end with `return;`.
- **Every movement is marked.** Call sites mark modify arguments too, with `lend()` or `MUT()`, so move, output and modify semantics are visible at declarations and call sites alike.
- **Null checks are written out** as explicit comparisons.
- **Precise names.** Namespace prefixes are case sensitive, use `__` as separator and up to 20 folders. Constants need the `G_` prefix, and constant static locals `S_` and capital letters, so that mutable state and constants are distinct at a glance.
- **Arrays live in structs.** Arrays may only be declared inside structs, which are named after their element struct and carry their sizes in their names. Array fields are only passed to library functions, spans always hold their whole array, and span data is only passed on in one-statement wrappers, so raw data can not escape reviewed code. String literals get no exception either: they too reach project functions through spans.
- **Tighter resource handling.** Arrays of raii structs are not allowed outside of structs, restricted malloc allows only `memory_alloc` and `memory_free`, and free structs are created with `_initialize_manually`, a name that makes the choice to opt out of the rules deliberate.

Every exception must be justified for audit and review, and the remaining rules follow the same idea: complete resource lifetimes with reverse destruction and standardized cleanup, one final return so that cleanup and exit paths are predictable, structured control flow, and immutable dispatch tables with uniform interface layouts.

## `embedded`

> For firmware on microcontrollers: no dynamic memory, every return value and switch case handled, state kept visible, and the vendor HAL left alone. Much of it follows the spirit of MISRA C.

| Rule | Setting | Why |
|------|---------|-----|
| Restricted malloc | No memory functions anywhere | Fragmentation and allocation failure have no place in firmware. A project that needs a pool allocator adds its functions to the list. |
| Null check | Comparisons written out, e.g. `ptr != NULL` | A null dereference may not fault on a microcontroller: it silently reads address 0. |
| Strict switch | Error | State machines are everywhere, and an unhandled state or a fallthrough is a real bug. |
| Function discard | Error | HAL and driver calls report failures through their return value, which must be handled. |
| Single return | No early returns | One point of exit, which also makes cleanup and timing easier to follow. |
| Enum | Error, typedef enums allowed | Enums are the natural fit for states and modes, but never mixed silently with integers. |
| Assignment | `{0}` allowed | Zeroed static buffers and structs holding pointers are normal in firmware. |
| Struct resource management | Error, free structs created with `_initialize` | Without a heap, raii structs still own peripherals, DMA channels, locks and pool blocks. |
| Interfaces | Error | Static const vtables end up in flash, a good fit for driver interfaces. |
| Global variable | Warning, `g_` and `s_` for mutable state, `G_` and `S_` in capitals for constants | Globals are unavoidable (driver state, data shared with interrupts), so they stay file-local and clearly marked, including state kept in static locals. |
| Prefix namespace | Warning | A flat C namespace across drivers, board support and application code needs prefixes. |
| Function pointer | Warning | Callbacks and handler tables are common, and a typedef keeps them readable. |
| Array struct | Arrays only passed to library functions | Fixed-size buffers carry their size in the type name. |
| Span struct | Span data only passed on in one-statement wrappers, static spans initialized with braces | Buffer overruns are the classic firmware bug, so raw buffer pointers only reach HAL and library calls through small wrappers that are easy to review. Static initializers can not call functions, so static buffers get their spans with braces. |
| Const field | Warning | Make the whole object const instead, so that tables are placed in flash as a whole. |
| Argument pointer movement | Warning, every operator marked | Every movement, including modification, is visible at the call site. |
| No goto | Error | Structured control flow, in the spirit of MISRA C. |
| Suppression reason | Error | Every deviation from the rules needs a written reason for reviews and audits. |

The preset also treats `Drivers/`, `CMSIS/` and `Middlewares/` as third party, where vendor HALs and RTOS code usually live (e.g. STM32Cube projects), together with `managed_components/` for ESP-IDF components and `.pio/libdeps/` for PlatformIO libraries.

## `nevernull`

> For projects that never create or pass null pointers. External libraries may still return null, so validate those boundaries explicitly. Also stricter than `default` about pointer and global clarity in general.

Changes from `default`:
- **No null, anywhere in the project's own code.** The assignment rule forbids initializing or assigning pointers with null, passing null as an argument, and zero-initializing objects that hold pointers with `{0}`. Pointers are initialized with real objects instead.
- **Boundaries stay checked.** The null check rule stays on, so that nullable values from external libraries are treated as unsafe until they are checked.
- **Clearer pointers and globals.** Function pointer typedefs are errors, constants need the `G_` prefix, and span data is only passed on in one-statement wrappers.
- **Allocation in one place**, so that project code can apply its non-null policy consistently. Free structs are created with `_initialize_manually`.

The preset works best together with [reference pointers](rules.md#reference-pointer-rule): parameters that must always receive a real object are tagged as references and need no null check, while normal pointers, and their checks, are kept for the boundaries with external code.

---

[Next: Diagnostic codes →](diagnostic-codes.md) · [Back to README](../README.md)
