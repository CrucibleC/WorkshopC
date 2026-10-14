[← Previous: Running WorkshopC](usage.md)

# Configuration

WorkshopC is configured with a YAML file that turns rules on, sets how serious their findings are, and adjusts their options. This page describes the file, the built-in presets, and how to choose and adapt one. The rules themselves and all their options are described in [Rules](rules.md).

## Contents

- [The config file](#the-config-file)
  - [Finding the config](#finding-the-config)
  - [File format](#file-format)
  - [Rule levels](#rule-levels)
  - [Third-party code](#third-party-code)
  - [Angle bracket includes](#angle-bracket-includes)
- [Built-in presets](#built-in-presets)
  - [Choosing a preset](#choosing-a-preset)
  - [The adoption path](#the-adoption-path)
- [Adapting a config](#adapting-a-config)
  - [Starting from a preset](#starting-from-a-preset)
  - [Command-line overrides](#command-line-overrides)
  - [Inspecting the effective config](#inspecting-the-effective-config)
- [Config validation](#config-validation)

## The config file

### Finding the config

WorkshopC uses the first of these that applies:

1. The file or built-in preset given with `--config`, e.g. `--config ci.workshopc.yaml` or `--config default`.
2. A file named `workshopc.yaml` in the current folder, or else in the closest parent folder that has one.

A config that is not called `workshopc.yaml` is named `<name>.workshopc.yaml` by convention, e.g. `ci.workshopc.yaml` for a stricter CI run. Preset names can be written with hyphens or underscores, so `adopt-more` and `adopt_more` are the same preset.

### File format

A config has four parts: the folders that hold third-party code, whether files included with `<>` are third party too, the folder of the compilation database, and the rules. This excerpt shows their shape; a real config writes out all of the rules:

```yaml
third_party_includes:
  - external/
  - vendor/

treat_angle_bracket_includes_as_third_party: false

compile_commands_dir: build

rules:
  enum:
    level: Warning
    allow_enum_typedef: true

  null_check:
    level: Error
    allow_direct_ptr_in_if_statement: false

  no_goto:
    level: Off

  # ... every other rule
```

| Key | Description |
|-----|-------------|
| `third_party_includes` | Folders whose code is never reported, see [third-party code](#third-party-code). |
| `treat_angle_bracket_includes_as_third_party` | `true` or `false`. With `true`, a file included with `<>` is third party too, see [angle bracket includes](#angle-bracket-includes). `false` when left out. |
| `compile_commands_dir` | The folder that holds `compile_commands.json`, relative to the config file. For a built-in preset, it is relative to the current folder. [`-p`](usage.md#config-and-input-options) overrides it. |
| `rules` | One entry for every rule, named by its config key, with a `level` and, unless the rule is `Off`, every one of the rule's options. Every rule and option is described in [Rules](rules.md). |

**Everything is written out.** No rule and no option has a default: every rule needs its `level`, and a rule that is not `Off` needs all of its options, so a config always shows exactly what it enforces. A rule that is `Off` may leave its options out. An empty text option is written as `""`. A config that leaves something out, or contains a rule or option that WorkshopC does not know, e.g. a misspelled one, is [reported as invalid](#config-validation) before anything is analyzed.

WorkshopC reads a simple subset of YAML: nested keys, plain values, `- item` lists, and `#` comments, as used in the presets. [`--dump-config`](#inspecting-the-effective-config) shows the settings exactly as WorkshopC read them.

> [!TIP]
> The default config, `release/configs/default.workshopc.yaml`, lists every rule and option with a one-line description of each rule, which makes it the easiest starting point for a config of your own.

### Rule levels

Every rule has a `level`:

| Level | Effect |
|-------|--------|
| `Off` | The rule does not run. Its options may be left out. |
| `Warning` | Findings are reported as warnings, and set bit `2` of the [exit code](usage.md#exit-codes). |
| `Error` | Findings are reported as errors, and set bit `1` of the exit code. |

Every rule must be given one of these levels; there is no default. `Warning` and `Error` only differ in how a finding is reported. The checks themselves are the same. [`--warnings-as-errors`](usage.md#output-options) reports every warning as an error for a single run, so one config can serve both a relaxed local run and a strict CI run.

### Third-party code

Code that the project does not own, such as vendored libraries and generated SDK code, should not be held to the project's style. A file is third party when its path contains one of the `third_party_includes` entries anywhere, e.g. `external/` matches `project/external/json/json.h`. The match is case sensitive.

- Third-party files and system headers are never reported, but the project's own code that uses them still is.
- When WorkshopC searches a folder for `.c` files, it skips third-party folders.
- Some rules treat third-party functions like the standard library. For example, an array may be passed to a third-party function, and a third-party function has no ownership tags.

The built-in presets list the common names for vendored code: `external/`, `extern/`, `third_party/`, `third-party/`, `thirdparty/`, `ThirdParty/`, `3rdparty/`, `3rdParty/`, `vendor/`, `deps/`, `dependencies/`, `subprojects/` and `vcpkg_installed/`. The `embedded` preset adds `Drivers/`, `CMSIS/` and `Middlewares/`, where vendor HALs and RTOS code usually live (e.g. STM32Cube projects), `managed_components/` for ESP-IDF components and `.pio/libdeps/` for PlatformIO libraries.

The list can be extended or replaced from the command line, see [command-line overrides](#command-line-overrides).

### Angle bracket includes

Many projects include outside code with angle brackets and their own headers with quotes:

```c
#include <json/json.h>     // a library
#include "ui/button.h"     // the project's own header
```

With `treat_angle_bracket_includes_as_third_party: true`, WorkshopC relies on that convention: a file included with `<>` is third party, and so is every file it includes in turn, however that one is written. Libraries are then skipped without listing their folders, e.g. when all of them live in one shared `include/` folder next to the project's own code.

> [!WARNING]
> With this option on, every header included with `<>` is skipped without notice, including the project's own. A project that includes its own headers with `<>`, e.g. `#include <mylib/buffer.h>` from a public `include/` folder, silently loses the checks on them. Only turn it on when the project always includes its own headers with quotes, and [watch the skipped files](usage.md#watching-skipped-files) in CI.

- **Decided per inclusion.** A header included both as `"foo.h"` and as `<foo.h>` is checked through the first and skipped through the second, so it is counted as both checked and skipped. Within one analyzed file an include guard or `#pragma once` makes only the first inclusion count, so a header that every file reaches through `<>` first is only skipped.
- **The list comes first.** A file in one of the `third_party_includes` folders is counted there, however it is included.
- **System headers are unaffected.** They are never checked or counted, whichever way they are included.
- **Everything else stays the same.** Files included with `<>` are third party in every way: rules treat their functions like the standard library, as for the folders in the list.

The summary splits the skipped files by reason, and [`--list-files`](usage.md#listing-files) marks every file that was skipped because of `<>`. The presets turn the option on, except for the [adoption path](#the-adoption-path), where an existing codebase may include its own headers with `<>`, and skipping them would hide exactly the findings the adoption is about.

## Built-in presets

WorkshopC comes with eight presets, built into the executable and selected with `--config <preset>`. They are also available as files in `release/configs`, as a starting point for a config of your own.

| Preset | Intended use |
|--------|--------------|
| `adopt` | A gentle entry point for existing projects. Resolve its findings before moving to `adopt-more`. |
| `adopt-more` | The second adoption step, adding stronger API and pointer checks. Move to `adopt-even-more` when it is clean. |
| `adopt-even-more` | The final adoption step, with explicit resource lifetimes, ownership, and buffer rules. Existing projects can stay here. |
| `default` | Recommended balanced settings for new projects and for users new to WorkshopC. |
| `opinionated` | The default approach with the author's preferred rule severities and CamelCase naming conventions. |
| `strict` | High adherence to WorkshopC's safety principles, ownership model and struct conventions. |
| `embedded` | Firmware-oriented settings: no dynamic allocation by default, explicit state handling, and vendor code exclusions. |
| `nevernull` | For projects that never create or pass null pointers. Results from external libraries still need validation. Also stricter than `default` about pointer and global clarity in general. |

The reasoning behind each preset, and a side-by-side comparison of their settings, is in [Preset rationale](preset-rationale.md).

### Choosing a preset

| Your situation | Start with |
|----------------|------------|
| A new project, or new to WorkshopC | `default` |
| An existing codebase | `adopt`, then follow [the adoption path](#the-adoption-path) |
| Firmware for microcontrollers | `embedded` |
| Safety-critical or audited code that should follow every principle | `strict` |
| A codebase that never passes or stores null pointers | `nevernull` |
| Like `default`, with the author's own preferences | `opinionated` |

A preset is a starting point, not a commitment. Any preset can be [copied into a config file](#starting-from-a-preset) and changed rule by rule.

### The adoption path

Moving an existing codebase to `default` in one step would produce an overwhelming number of findings. The three adoption presets introduce the rules gradually instead, so that each step is a manageable amount of work:

1. **`adopt`** turns on the core checks that find real bugs with little rewriting: null checks, enum safety, complete switch statements and initialization. Most of them are warnings. Rules that need larger changes are off: ownership tags, struct resource management, restricted malloc, single return, namespaces, function discard, typedef requirements and array structs.
2. **`adopt-more`** makes the core checks errors, and adds function pointer typedefs, include guards, function discard and single return as warnings, together with stricter span rules.
3. **`adopt-even-more`** adds ownership tags, struct resource management, restricted malloc and array struct naming as warnings, and makes include guards, function discard and the span rules errors.

`adopt-even-more` is a fine place to stay. `default` is meant for new code, and moving an existing codebase all the way there is a large refactor. Ambitious projects can first make `adopt-even-more` clean with `--warnings-as-errors`.

A practical workflow for each step:

1. Run the next preset locally and fix its findings, one rule or one folder at a time.
2. Where code can not reasonably follow a rule yet, [suppress the section](rules.md#suppressing-rules) with a reason, so that the exception stays visible.
3. When the step is clean, make CI enforce it with `--warnings-as-errors`, so that no new findings creep in while the next step begins.

The [adoption path in Preset rationale](preset-rationale.md#the-adoption-path) lists exactly what changes at each step.

## Adapting a config

### Starting from a preset

To customize a preset, start from a copy of it. Either copy its file from `release/configs`, which keeps the one-line description of every rule, or dump its full settings:

```bash
workshopc --config default --dump-config > workshopc.yaml
```

Then change the rules and options you need. Every rule and option is described in [Rules](rules.md).

### Command-line overrides

Small environment-specific differences can be supplied without copying a preset or editing a config:

| Option | Effect |
|--------|--------|
| `--third-party-include <folder>` | Adds a folder to `third_party_includes`. Repeat it to add several. |
| `--no-config-third-party-includes` | Drops the config's own `third_party_includes`, so that only the folders given with `--third-party-include` are third party. |
| `--prefix-top-dir <folder>` | Overrides `prefix_namespace.top_dir`. |
| `-p`, `--build-path <folder>` | Overrides `compile_commands_dir`. |

For example, when the `vendor/` folder of a preset holds the project's own code, the list can be replaced:

```bash
workshopc --config default --no-config-third-party-includes --third-party-include external/ src/
```

### Inspecting the effective config

`--dump-config` prints the effective config, with the command-line overrides applied and every setting written out, and exits without analyzing anything:

```bash
workshopc --dump-config                                                    # the config found automatically
workshopc --config embedded --prefix-top-dir firmware --dump-config        # a preset with an override
```

The output is itself a complete config, with every rule and every option, an empty text as `""`, so it can be saved and used directly. It is printed even when the config is invalid, so that a broken config can be inspected. A rule or option the config left out still appears in it, so compare it with the config's own file when looking for what is missing.

## Config validation

Before anything is analyzed, WorkshopC checks that the config is complete and consistent. An invalid config stops the run with exit code `65`, and every problem is listed:

```text
Invalid config: /repo/workshopc.yaml
  rules.null_check is missing; every rule must be written out with its level: Off, Warning or Error
  rules.enum.allow_enum_typdef is not a setting of the enum rule
  rules.enum.allow_enum_typedef is missing; every setting of a rule that is not Off must be written out
  rules.struct_resource_management.raii_struct_move_suffix must be set when the rule is not Off
```

The checks are:
- **Only known settings.** Every rule, option and top-level key must be one WorkshopC knows, so a misspelled name is reported instead of being ignored.
- **Every rule is written out with its level.** The level must be `Off`, `Warning` or `Error`.
- **Every option of a rule that is not `Off` is written out.** No option has a default, see [file format](#file-format).
- **Text options that name something are not empty.** Options described as "may not be empty" in [Rules](rules.md), e.g. the suffixes of the struct resource management rule, may not be `""` when their rule is not `Off`. Some only when another option needs them, e.g. `raii_struct_array_destroyer_suffix` with `allow_raii_struct_arrays: true`.
- **Names do not collide.** The function suffixes of the struct resource management rule must be different from each other, and so must the suffixes and names of the interfaces, array struct and span struct rules.
- **Settings that need another rule have it.** E.g. `reference_pointer.disable_null_check_rule_for_reference_pointers` needs the null check rule, `interfaces.interface_must_have_fields_that_are_private_alternative` needs the private alternative rule, `prefix_namespace.accept_pragma_once_instead_of_ifndef` needs `require_ifndef_for_filepath`, and `span_struct.allow_pod_span_to_be_initialized_manually_if_static` needs the struct resource management rule. Some settings exclude each other, e.g. `reference_pointer.allow_direct_ptr_in_if_statement` and `only_allow_reference_pointers_as_function_parameters`.
- **Values make sense.** E.g. `treat_angle_bracket_includes_as_third_party` must be `true` or `false`, `prefix_namespace.stop_at_count` may not be negative, `prefix_namespace.top_dir` may not be one of the `third_party_includes`, and `array_struct.use_naming_rules_on_one_array_field_structs_without_forbidding_public_arrays` needs at least one naming option.

---

[Next: Rules →](rules.md) · [Back to README](../README.md)
