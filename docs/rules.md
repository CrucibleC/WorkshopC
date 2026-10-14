[← Previous: Configuration](configuration.md)

# Rules

WorkshopC's rules describe a style of C in which ownership, lifetimes, nullability and buffer sizes are visible in the code and checked on every run. This page is the complete reference: what each rule enforces and why, its options, and examples of code it accepts and reports.

If you are new to WorkshopC, [the tour](tour.md) introduces the most important rules in a few minutes, and [Configuration](configuration.md) explains how rules are turned on.

## Contents

- [Reading this page](#reading-this-page)
- [Rules at a glance](#rules-at-a-glance)
- [Enum rule](#enum-rule)
- [Private rule](#private-rule)
- [Private alternative rule](#private-alternative-rule)
- [Function pointer rule](#function-pointer-rule)
- [Typedef struct rule](#typedef-struct-rule)
- [Assignment rule](#assignment-rule)
- [Prefix namespace rule](#prefix-namespace-rule)
- [Null check rule](#null-check-rule)
- [Argument pointer movement rule](#argument-pointer-movement-rule)
- [Struct resource management rule (RAII)](#struct-resource-management-rule-raii)
- [Interfaces rule](#interfaces-rule)
- [Restricted malloc rule](#restricted-malloc-rule)
- [Single return rule](#single-return-rule)
- [Strict switch rule](#strict-switch-rule)
- [Global variable rule](#global-variable-rule)
- [Reference pointer rule](#reference-pointer-rule)
- [Function discard rule](#function-discard-rule)
- [Array struct rule](#array-struct-rule)
- [Span struct rule](#span-struct-rule)
- [Const field rule](#const-field-rule)
- [No goto rule](#no-goto-rule)
- [Typedef number rule](#typedef-number-rule)
- [Suppressing rules](#suppressing-rules)
- [Adjusting code for analysis](#adjusting-code-for-analysis)

## Reading this page

- Each rule starts with its **config key** (its name in the config file) and the [diagnostic codes](diagnostic-codes.md) it can report, followed by an example config and a table of its options.
- A config must give every rule a `level`: `Off`, `Warning` or `Error`. A rule that is not `Off` must have every option in its table written out, since no option has a default. An option described as "may not be empty" also needs a value other than `""`. Otherwise the config is [reported as invalid](configuration.md#config-validation).
- In the examples, `// OK` marks code the rule accepts and `// Reported` marks code it reports, usually followed by the reason. The examples only show the lines that matter, so declarations they rely on are sometimes left out.
- Code in system headers and in `third_party_includes` folders is never reported. The project's own code that uses it still is.
- Code that a macro generates belongs to the file the macro is defined in, so a vendor macro used in the project's code is not reported. A name the macro builds with `##`, e.g. `type##_list` in a generic container macro, belongs to the file the macro is used in, since it is written nowhere else.
- Tags such as `PRIVATE`, `REF` and `receives` are macros from the [tag headers](tag-headers.md). The examples use both naming styles.

## Rules at a glance

| Rule | Config key | What it enforces | Codes |
|------|------------|------------------|-------|
| [Enum](#enum-rule) | `enum` | No enums, or typedef enums that only take their own members | `CCW01xx` |
| [Private](#private-rule) | `private` | Fields with a private name are only reached through static accessors | `CCW02xx` |
| [Private alternative](#private-alternative-rule) | `private_alternative` | Tagged fields are only reached through their own struct's functions | `CCW03xx` |
| [Function pointer](#function-pointer-rule) | `function_pointer` | Every function pointer is written with a typedef | `CCW04xx` |
| [Typedef struct](#typedef-struct-rule) | `typedef_struct` | Every struct has a typedef | `CCW05xx` |
| [Assignment](#assignment-rule) | `assignment` | Variables are initialized at declaration, optionally no null and no argument changes | `CCW06xx` |
| [Prefix namespace](#prefix-namespace-rule) | `prefix_namespace` | Names in headers carry a prefix built from their folder path | `CCW07xx` |
| [Null check](#null-check-rule) | `null_check` | Pointer parameters are checked for null before they are dereferenced | `CCW08xx` |
| [Argument pointer movement](#argument-pointer-movement-rule) | `argument_pointer_movement` | Pointer ownership is tagged on parameters and marked at call sites | `CCW09xx` |
| [Struct resource management](#struct-resource-management-rule-raii) | `struct_resource_management` | Pod, raii and free structs with checked creation and cleanup | `CCW10xx`–`CCW13xx` |
| [Interfaces](#interfaces-rule) | `interfaces` | One standard layout for vtables and interfaces | `CCW14xx`–`CCW15xx` |
| [Restricted malloc](#restricted-malloc-rule) | `restricted_malloc` | Memory functions are only used inside a few wrapper functions | `CCW16xx` |
| [Single return](#single-return-rule) | `single_return` | Functions have one final return, optionally after guard clauses | `CCW17xx` |
| [Strict switch](#strict-switch-rule) | `strict_switch` | Every switch has a default, and no case falls through | `CCW18xx` |
| [Global variable](#global-variable-rule) | `global_variable` | Linkage, const and naming conventions for globals and static locals | `CCW19xx` |
| [Reference pointer](#reference-pointer-rule) | `reference_pointer` | Tagged pointer parameters always point to a real object | `CCW20xx` |
| [Function discard](#function-discard-rule) | `function_discard` | Return values are used, or discarded explicitly with `(void)` | `CCW21xx` |
| [Array struct](#array-struct-rule) | `array_struct` | Arrays live inside structs, and array structs follow naming conventions | `CCW22xx` |
| [Span struct](#span-struct-rule) | `span_struct` | Pointer and size travel together in standard span structs | `CCW23xx` |
| [Const field](#const-field-rule) | `const_field` | Struct fields are never const themselves | `CCW24xx` |
| [No goto](#no-goto-rule) | `no_goto` | No goto statements | `CCW25xx` |
| [Typedef number](#typedef-number-rule) | `typedef_number` | Built-in number types are only used through a typedef, e.g. `int32_t` | `CCW26xx` |
| [Suppression reason](#suppression-reason-rule) | `suppression_reason_rule` | Every suppression gives a reason | `CCW0002` |

## Enum rule

An `enum` in C is just an integer: an enum variable accepts any integer, and enum values mix freely with arithmetic, which easily creates bugs. This rule either forbids enums completely or, with `allow_enum_typedef: true`, allows them under strict rules.

**Config key:** `enum` · **Codes:** `CCW0101`–`CCW0106`

```yaml
rules:
  enum:
    level: Warning
    allow_enum_typedef: false
```

| Option | Description |
|--------|-------------|
| `allow_enum_typedef` | Allow enums that have a typedef, under the requirements in [Allow typedef enums](#allow-typedef-enums-allow_enum_typedef-true). When `false`, every enum definition is reported. |

### Forbid enums (`allow_enum_typedef: false`)

Every `enum` definition is reported.

```c
enum Color {            // Reported: enums are not allowed
    RED,
    GREEN,
    BLUE
};
```

Instead, a set of static or extern struct objects can stand in for the values, or a struct can be declared and a pointer to it typedef'd as the "enum":

```c
// One way to mimic enums with type safety:
// the functions return pointers to static ColorTag objects
struct ColorTag;
typedef const struct ColorTag* Color;
Color color_red(void);
Color color_green(void);
Color color_blue(void);

// These defines make the call sites read like enum usage
#define RED color_red()
#define GREEN color_green()
#define BLUE color_blue()
```

### Allow typedef enums (`allow_enum_typedef: true`)

Enums are allowed, but two requirements apply.

**1. Every enum must have a typedef.** The typedef may come before or after the definition, and an anonymous enum is fine if it is defined together with its typedef.

```c
typedef enum Color { RED, GREEN, BLUE } Color;      // OK
typedef enum { SMALL, MEDIUM, LARGE } Size;         // OK: anonymous with a typedef

enum Direction { NORTH, SOUTH };
typedef enum Direction Direction;                   // OK: typedef after the definition

enum Shape { CIRCLE, SQUARE };                      // Reported: no typedef
```

**2. A variable, assignment target or function argument of a typedef enum type may only be given a member of that enum.** Copying a value that already has the enum type (a variable, or the result of a function) is fine, and so is an explicit cast, which is the way to deliberately convert an integer. A conditional expression is fine if both branches are members.

```c
Color a = RED;                  // OK
Color b = a;                    // OK: already a Color
Color c = (Color)1;             // OK: explicit cast
Color d = flag ? RED : BLUE;    // OK: both branches are members

Color e = 0;                    // Reported: plain integer
Color f = i;                    // Reported: integer variable
Color g = SMALL;                // Reported: member of a different enum
Color h = RED | GREEN;          // Reported: arithmetic

a = 2;                          // Reported
a = (Color)i;                   // OK: explicit cast
a++;                            // Reported: arithmetic
a |= GREEN;                     // Reported: arithmetic

void takes_color(Color color);
takes_color(GREEN);             // OK
takes_color(2);                 // Reported
takes_color((Color)2);          // OK: explicit cast
```

Details:
- Initializations, assignments (including through struct fields and pointers) and call arguments are checked. Compound assignments (`+=`, `|=`, ...), `++` and `--` are always reported, since they can produce values outside of the enum.
- A parameter that is a plain `int` accepts enum members without complaint, since it is not an enum type.
- Enums defined in system headers or `third_party_includes` folders are not checked, neither their definitions nor the code using them.
- The compiler can report its own warning when a member of one enum is used as a different enum (`-Wimplicit-enum-enum-cast`). WorkshopC's warning comes in addition to it.

## Private rule

This rule makes a struct field private by its name: every field named `private_field` (e.g. `_private`), in any struct, may only be accessed from a few accessor functions in a source file. Keeping private data behind a handful of functions means that the rest of the code can not put the struct in an inconsistent state, and that changing the private layout only affects one file.

**Config key:** `private` · **Codes:** `CCW0201`–`CCW0202`

```yaml
rules:
  private:
    level: Error
    private_field: _private
    setter_contains: private_set
    getter_contains: private_get
```

| Option | Description |
|--------|-------------|
| `private_field` | The exact field name that makes a field private, e.g. `_private`. May not be empty. |
| `getter_contains` | Text that the name of a getter contains, anywhere in the name. May not be empty. |
| `setter_contains` | Text that the name of a setter contains, anywhere in the name. It may be the same as `getter_contains`, e.g. `private_` for both. May not be empty. |

With this config, the following is reported:

```c
struct Color
{
  int weight;
  struct {
      int r, g, b;
  } _private;
};
typedef struct Color Color;

void access_color_members(Color* color)
{
  int weight = color->weight; // OK: not private
  int r = color->_private.r;  // Reported: not an accessor
}
```

Instead the private field must be accessed through getter and setter functions. An accessor must meet all three requirements:
- it is `static`,
- it is defined in a `.c` file (a static function in a header is copied into every file that includes it, so it is reported there),
- its name contains `getter_contains` or `setter_contains`, anywhere in the name, e.g. `private_get_red` or `color_private_set_red`.

```c
// color.c
static int private_get_red(const Color* const color)
{
  return color->_private.r;       // OK
}

static void private_set_red(Color* const color, int new_value)
{
  color->_private.r = new_value;  // OK
}

int color_red(const Color* const color)
{
  return private_get_red(color);  // OK: the public function goes through the accessor
}
```

If the private field is a named struct, two accessors are enough, since they can hand out a pointer to the whole private part.

```c
struct ColorPrivate
{
  int r, g, b;
};
typedef struct ColorPrivate ColorPrivate;

struct Color
{
  int weight;
  ColorPrivate _private;
};
typedef struct Color Color;

static const ColorPrivate* private_get(const Color* const color)
{
  return &color->_private;
}

static ColorPrivate* private_set(Color* const color)
{
  return &color->_private;
}

void color_set_red(Color* const color, int red)
{
  private_set(color)->r = red;    // OK: r is a field of ColorPrivate, which is not private itself
}
```

Details:
- Only the field with exactly the configured name is private. Its own fields are reached through it, so `color->_private.r` counts as an access of `_private`.
- An access outside of any function, e.g. in the initializer of a global variable (`static int* g_red = &g_color._private.r;`), is always reported, even inside `sizeof`.
- Accesses in system headers and `third_party_includes` folders are not checked.
- Private fields can also be marked with a tag instead of a name, see the [private alternative rule](#private-alternative-rule). The two rules find private fields in different ways and can be enabled together, as they are in the built-in presets.

## Private alternative rule

This rule is an alternative to the [private rule](#private-rule) that, instead of matching a field by name, matches any field tagged with the privacy tag. A [premade tag header](tag-headers.md) can be used as is. The tag is a macro that adds an annotation while WorkshopC analyzes the code and expands to nothing otherwise, so its name can be changed freely.

**Config key:** `private_alternative` · **Codes:** `CCW0301`–`CCW0302`

```yaml
rules:
  private_alternative:
    level: Error
```

This rule has no options besides `level`.

A tagged field may only be accessed from a function whose name starts with the owning struct's own name, i.e. the struct tag (`Color` for `struct Color`), not a typedef alias of it. The function must also reach the field in one of two ways:
- through its first parameter, which must be a pointer to the owning struct named `self` (const or not, typedef or not), e.g. `self->r`,
- or with `.` directly on a variable or parameter whose type is the owning struct itself (not a pointer to it), e.g. `color.r` on a local `Color_t color`. This is what lets the [pod and raii creator functions](#struct-resource-management-rule-raii) set up the struct they return.

> [!TIP]
> This rule works especially well with [struct resource management](#struct-resource-management-rule-raii), where the creator functions can also access the private fields.

This example uses the macro `PRIVATE` for the privacy tag.

```c
#include "private_tag.h"

struct Color
{
  int weight;
  PRIVATE int r;
  PRIVATE int g;
  PRIVATE int b;
};
typedef struct Color Color_t;

void access_color_members(Color_t* color)
{
  int weight = color->weight; // OK: not tagged
  int r = color->r;           // Reported: not one of Color's functions
}
```

Instead, the accessor functions' names start with the struct's own name, and their first parameter is a pointer named `self`.

```c
int Color_get_r(const Color_t* self)
{
  return self->r;             // OK
}

void Color_set_r(Color_t* self, int value)
{
  self->r = value;            // OK
}

Color_t Color_pod(int r, int g, int b)
{
  Color_t color = {0};
  color.r = r;                // OK: a local value of the struct itself, in a function named after it
  color.g = g;
  color.b = b;
  return color;
}

int Color_get_other_r(const Color_t* other)
{
  return other->r;            // Reported: the pointer is not named self
}

int get_r(const Color_t* self)
{
  return self->r;             // Reported: the name does not start with Color
}
```

Reaching into a *different* struct's tagged field through a field of the current struct still requires that other struct's own accessor, even from inside a function that is otherwise a valid accessor for the current struct.

```c
struct Wrapper
{
  PRIVATE Color_t color;
};
typedef struct Wrapper Wrapper_t;

void Wrapper_bad(Wrapper_t* self)
{
  self->color.weight = 5; // OK: color is Wrapper's own field, and weight is not tagged
  self->color.r = 5;      // Reported: r belongs to Color, not Wrapper
}
```

As with the private rule, an access outside of any function is always reported, and accesses in system headers and `third_party_includes` folders are not checked.

## Function pointer rule

This rule requires every function pointer to be written with a typedef, whether it is a variable, a parameter, a struct field or the return type of a function. A typedef makes function signatures readable, keeps a signature in one place when it changes, and makes it harder to mistype.

**Config key:** `function_pointer` · **Codes:** `CCW0401`

```yaml
rules:
  function_pointer:
    level: Warning
```

This rule has no options besides `level`.

```c
typedef int (*math_function)(int, int);

// Reported: the parameter is spelled out without a typedef
void bad_perform_math(int (*math_func)(int, int), int a, int b, int* out_res)
{
  *out_res = math_func(a, b);
}

// OK
void perform_math(math_function math_func, int a, int b, int* out_res)
{
  *out_res = math_func(a, b);
}

int add(int a, int b);

void foo(void)
{
  int (*raw)(int, int) = add;                 // Reported
  math_function typed = add;                  // OK
  int (*table[2])(int, int) = {add, add};     // Reported: an array of function pointers
  math_function typed_table[2] = {add, add};  // OK
}

struct calculator
{
  int (*raw_operation)(int, int);             // Reported
  math_function operation;                    // OK
};

int (*get_raw_operation(void))(int, int);     // Reported: the return type
math_function get_operation(void);            // OK
```

Details:
- Global, static and local variables, function parameters, struct fields and return types are checked, also when the function pointer is the element of an array or is pointed to by another pointer (`int (**)(int)`).
- Any typedef works, whatever its name, as long as the function pointer itself is written with it. A typedef used only for the return or parameter types of the function (`size_t (*f)(int)`) does not count.
- Declarations in system headers and `third_party_includes` folders are not checked, and a macro that expands to a function pointer declaration is reported with `(macro expansion)` added to the message.

## Typedef struct rule

This rule requires every struct definition to have a typedef. Typedefs keep the codebase consistent, remove the need to write `struct` every time the type is used, and give every type one uniform way to be named.

**Config key:** `typedef_struct` · **Codes:** `CCW0501`

```yaml
rules:
  typedef_struct:
    level: Warning
```

This rule has no options besides `level`.

```c
struct Position                 // Reported: no typedef
{
  int x, y, z;
};

typedef struct Size             // OK
{
  int width, height;
} Size;

struct Point                    // OK: the typedef can come later in the file
{
  int x, y;
};
typedef struct Point Point_t;   // Any name works

typedef struct                  // OK: anonymous struct with a typedef
{
  int r, g, b;
} Color;

typedef struct Line             // OK
{
  struct Segment                // Reported: a named nested struct needs its own typedef
  {
    int start, end;
  } segment;
} Line;
```

Details:
- Only struct definitions are checked. A forward declaration (`struct Position;`) alone is not reported.
- The typedef may come before or after the definition, anywhere at file scope in the same translation unit, and may have any name.
- Anonymous structs are never reported, since they can not be named anyway (e.g. an anonymous nested struct or a single `struct { ... } instance;`).
- A struct defined by a macro is checked where the macro is used, even when the macro itself comes from a third party header. Structs defined in system headers and `third_party_includes` folders are not checked.

## Assignment rule

This rule requires every variable of a basic type (e.g. `int`, `float`, `char`, `_Bool`) or pointer type to be initialized in the statement that declares it, and the same goes for arrays of them. Initialization at declaration prevents use-before-initialization bugs and makes intent clear: a variable that exists is ready to use. Structs and unions are not covered by this check, see [struct resource management](#struct-resource-management-rule-raii) for them.

Optional settings go further: they can forbid null pointers in assignments and arguments, and forbid changing a function's arguments.

**Config key:** `assignment` · **Codes:** `CCW0601`–`CCW0612`

```yaml
rules:
  assignment:
    level: Warning
    forbid_null_assign: true
    forbid_null_as_arg: true
    forbid_zero_init_for_objects_with_pointers: true
    forbid_arg_reassign: true
    forbid_mut_arg_pointer: true
```

| Option | Description |
|--------|-------------|
| `forbid_null_assign` | Pointers, pointer fields and arrays of pointers may not be set to null. See [Null assignment](#null-assignment). |
| `forbid_null_as_arg` | Null may not be passed as an argument to any function. |
| `forbid_zero_init_for_objects_with_pointers` | An object that contains pointers may not be initialized with `{0}`. |
| `forbid_arg_reassign` | Arguments, and the fields of arguments passed by value, may not be changed. See [Argument modification](#argument-modification). |
| `forbid_mut_arg_pointer` | The address of an argument may only be taken as a pointer to const. |

```c
static int g_count;         // Reported: globals and statics too, even though C sets them to 0
int a;                      // Reported
int b = 5;                  // OK
int* ptr;                   // Reported
int values[4];              // Reported
int values_2[4] = {0};      // OK
Position position;          // OK: structs are not checked by this rule
```

### Null assignment

The first three settings support codebases that follow a strict "nothing may be null" policy, so that no null dereference can occur. `NULL`, `0` and casts of `0` such as `(void*)0` all count as null.

```c
struct Setting
{
  const char* name;
  int x, y, z;
};
typedef struct Setting Setting;

void example(int* a, int* b)
{
  int* ptr = a;
  ptr = NULL;                                 // Reported with forbid_null_assign
  int* empty = NULL;                          // Reported with forbid_null_assign
  foo(NULL, 3, 7);                            // Reported with forbid_null_as_arg

  Setting setting = {0};                      // Reported with forbid_zero_init_for_objects_with_pointers
  Setting named = { .name = NULL, .x = 1 };   // Reported with forbid_null_assign: NULL in a pointer field
  named.name = 0;                             // Reported with forbid_null_assign

  int* pair[2] = { a, b };                    // OK
  int* partial[3] = { a, b };                 // Reported with forbid_null_assign: the third element is left null
  int* explicit_null[2] = { a, NULL };        // Reported with forbid_null_assign
}
```

- `forbid_null_assign`: a pointer may not be initialized with null, a pointer or a pointer field may not be assigned null, a pointer field in an initializer list may not be given null, and an array of pointers must explicitly initialize every element (at every level of an array of arrays) with a non-null value, since elements left out become null pointers.
- `forbid_null_as_arg`: null may not be passed as an argument to any function.
- `forbid_zero_init_for_objects_with_pointers`: an object that contains a pointer, directly or in a field of a field, may not be initialized with `{0}`, since that silently makes every pointer in it null. Objects without pointers may still use `{0}`.

### Argument modification

Const correctness for arguments is often skipped, but an argument that changes halfway through a function makes the function harder to follow. Instead of requiring `const` on every parameter, these settings simply forbid changing arguments. They do not affect the data a pointer argument points to.

> [!NOTE]
> For const correctness across all variables, a tool such as clang-tidy can be used alongside WorkshopC.

```c
typedef struct RGB
{
  int r, g, b;
} RGB;

void foo(const char* name, int x, RGB color, RGB* rgb)
{
  name = "New string";    // Reported with forbid_arg_reassign
  x = 5;                  // Reported with forbid_arg_reassign
  x += 1;                 // Reported with forbid_arg_reassign
  x++;                    // Reported with forbid_arg_reassign
  color.r = 0;            // Reported with forbid_arg_reassign: a field of a struct passed by value
  int* x_ptr = &x;        // Reported with forbid_mut_arg_pointer
  set_int(&x);            // Reported with forbid_mut_arg_pointer (void set_int(int* out))

  const int* x_view = &x; // OK: the argument can not be modified through it
  print_int(&x);          // OK (void print_int(const int* value))
  int y = x;              // OK
  rgb->r = x;             // OK: writing through a pointer argument is fine
}
```

- `forbid_arg_reassign`: an argument may not be assigned a new value (`=`, `+=`, `|=`, ...) or be changed with `++` or `--`, and neither may the fields of a struct passed by value, since the change would only affect the local copy.
- `forbid_mut_arg_pointer`: the address of an argument, or of a field of a by-value argument, may only be taken as a pointer to const, wherever it is taken: in a declaration, an assignment, a function call or a return. A cast decides the type, so `(const int*)&x` is OK while `(int*)&x` is not. Without this setting, `forbid_arg_reassign` could be worked around by writing through a pointer to the argument.

## Prefix namespace rule

This rule requires the names declared in a header to start with a prefix built from the header's folder path, which mimics namespaces in languages such as C++. It prevents naming collisions in large projects and makes it obvious at a glance which module a name belongs to. Optionally, the header must also have an include guard built from its path.

**Config key:** `prefix_namespace` · **Codes:** `CCW0701`–`CCW0702`

```yaml
rules:
  prefix_namespace:
    level: Warning
    top_dir: src
    work_from_top: true
    stop_at_count: 10
    use_separator: true
    separator: __
    apply_to_functions: true
    apply_to_structs: true
    apply_to_typedefs: true
    require_ifndef_for_filepath: true
    accept_pragma_once_instead_of_ifndef: false
    case_insensitive: false
```

| Option | Description |
|--------|-------------|
| `top_dir` | The folder the namespaces start in, e.g. `src`. The prefix is built from the folders after the first folder with this name in the header's path, not including it. A header that is not inside such a folder is not checked. Can be overridden with [`--prefix-top-dir`](usage.md#config-and-input-options). May not be empty. |
| `stop_at_count` | The most folders used in the prefix. |
| `work_from_top` | With `true`, the folders closest to `top_dir` are used, otherwise the folders closest to the file. |
| `use_separator` | Write `separator` between the folder names and after the last one. Without it the folder names are joined directly, e.g. `appchrono`. |
| `separator` | The separator, e.g. `__`. May not be empty, even when `use_separator` is `false`. |
| `apply_to_functions` | Function names in headers must have the prefix, static functions included, since every file that includes the header sees them. |
| `apply_to_structs` | Struct names must have the prefix. |
| `apply_to_typedefs` | Typedef names must have the prefix. |
| `require_ifndef_for_filepath` | The header must have an include guard built from its path, see below. |
| `accept_pragma_once_instead_of_ifndef` | Accept `#pragma once` instead of the include guard. Needs `require_ifndef_for_filepath: true`, or the config is invalid. |
| `case_insensitive` | A name only needs the same letters as the prefix, in upper or lower case. |

Details:
- Folder names keep their case in the prefix, and every character that is not a letter or digit becomes `_`, so a folder `My-Lib` becomes `My_Lib` in the prefix (e.g. `My_Lib__open`) and `MY_LIB` in the include guard. Finding `top_dir` in the path does not depend on case.
- With `case_insensitive: true`, for the folders `App/Chrono` both `App__Chrono__start` and `app__chrono__start` are fine. Without it the case must match the folders exactly. The include guard is not affected, it is always in capital letters.
- With `require_ifndef_for_filepath: true`, the first `#ifndef` in the first 10 lines of the header must be the path after `top_dir`, including the file name, in capital letters and joined with `_`, whatever the separator setting. With `accept_pragma_once_instead_of_ifndef: true` too, a `#pragma once` in the first 10 lines is accepted instead, and then the `#ifndef`, if any, is not checked.
- Only headers are checked, since their names are the ones other files see, and headers in `third_party_includes` folders are skipped.
- `top_dir` may not be one of the `third_party_includes`, and `stop_at_count` may not be negative.

Here is what the settings above expect from a header:

```c
// This file is in c/workspace/repos/project/src/app/chrono/timer.h

#ifndef APP_CHRONO_TIMER_H
#define APP_CHRONO_TIMER_H

struct app__chrono__timer;
typedef struct app__chrono__timer app__chrono__timer_t;
typedef int (*app__chrono__callback)(void);

void app__chrono__start_timer_with_callback(app__chrono__timer_t* timer,
                                            app__chrono__callback callback,
                                            int seconds);

void start_timer(app__chrono__timer_t* timer);      // Reported: missing the app__chrono__ prefix

#endif
```

With `work_from_top: false` and `stop_at_count: 1`, only the folder closest to the file is used, so the prefix for the same header would be `chrono__`.

## Null check rule

Every pointer parameter must be checked for null before it is dereferenced. A function can then never crash on a null argument, and the check documents that null is an input the function handles.

**Config key:** `null_check` · **Codes:** `CCW0801`

```yaml
rules:
  null_check:
    level: Error
    allow_direct_ptr_in_if_statement: true
```

| Option | Description |
|--------|-------------|
| `allow_direct_ptr_in_if_statement` | Accept a plain `ptr` or `!ptr` as a null check. When `false`, the comparison must be written out, e.g. `ptr != NULL`. |

```c
int dereference_without_check(int* i_ptr)
{
  return *i_ptr;  // Reported
}

int dereference_safely(int* i_ptr)
{
  if (i_ptr == NULL)
  {
    return -1;
  }
  return *i_ptr;  // OK
}
```

`*i_ptr`, `i_ptr->field` and `i_ptr[index]` all count as dereferences. A dereference is only allowed where the pointer is known to be non-null on every path of the function that reaches it, so the rule follows the control flow of the function rather than just looking for a check somewhere before it. A pointer is known to be non-null:
- In the branch where a condition says so: the `if` branch of `if (i_ptr != NULL)`, the `else` branch of `if (i_ptr == NULL)`, the right side of `i_ptr && *i_ptr` or `!i_ptr || *i_ptr`, the matching branch of `?:`, and a loop body behind such a loop condition.
- After a check whose null branch always leaves: `return`, `break`, `continue`, `goto`, or a call to a function that never returns, such as `abort()` or `exit()`. `assert(i_ptr != NULL)` works the same way, since a failed assert aborts.

These conditions count as null checks:
- A comparison with null, e.g. `i_ptr == NULL`, `NULL != i_ptr`, or a comparison inside a macro such as `IS_NULL(i_ptr)`. It can also be stored in a variable first: `bool valid = i_ptr != NULL; if (valid) ...`.
- With `allow_direct_ptr_in_if_statement: true`, a plain `i_ptr` or `!i_ptr`, e.g. `if (!i_ptr) return;` or `return i_ptr ? *i_ptr : 0;`.

```c
int sum(int* a, int* b, int flag)
{
  if (a == NULL)
    return 0;

  if (flag)
  {
    if (!b)
      return *a;      // OK: a is not null on every path here
    return *a + *b;   // OK: b is not null in this branch
  }

  return *a + *b;     // Reported for b: it is only checked when flag is set
}

int log_only(int* value)
{
  if (value == NULL)
    log_error("null");  // execution goes on with a null pointer

  return *value;        // Reported: the null branch does not leave
}
```

Where paths come together, e.g. after an `if`/`else`, a loop or a `switch`, or at a label reached by a `goto`, the pointer is only known to be non-null if it is on all of them. So a check in one branch of an `if` does not count after it unless the other branches check too, and a check inside a loop does not count after the loop, since the loop may not run at all.

Giving a parameter a new value (`i_ptr = other;`) forgets that it was checked. Passing the pointer on to another function is not a check, and neither is a function's result, e.g. `if (is_valid(i_ptr))`, since the rule can not know what the function checks. Each parameter is checked on its own and reported at most once, at its first unchecked dereference.

Parameters of the [reference pointer rule](#reference-pointer-rule) can never be null, so they can be exempted from this rule with `disable_null_check_rule_for_reference_pointers`.

## Argument pointer movement rule

This rule makes the ownership of pointers visible. Every non-const pointer parameter must be tagged with how the function uses it, and call sites can be required to mark the argument with a matching operator, so both the function and the call site show what happens to the pointer. A [premade tag header](tag-headers.md) can be used as is, and all the names can be changed.

**Config key:** `argument_pointer_movement` · **Codes:** `CCW0901`–`CCW0912`

There are three tags:
- **move**: the function takes ownership of the pointer, e.g. it stores it or frees it. The caller may not use the pointer again until it has been given a new value.
- **out**: the function writes a result through the pointer, typically a `Data**` it points at newly created data, or a value it fills in.
- **modify**: the function changes the data the pointer points to, but the caller keeps ownership.

A pointer to const needs no tag, since the function can only read through it.

```yaml
rules:
  argument_pointer_movement:
    level: Warning
    require_operator_for_move_callsite: true
    require_operator_for_out_callsite: true
    require_operator_for_modify_callsite: false
```

| Option | Description |
|--------|-------------|
| `require_operator_for_move_callsite` | Arguments for move parameters are wrapped in the move operator. |
| `require_operator_for_out_callsite` | Arguments for out parameters are wrapped in the out operator. |
| `require_operator_for_modify_callsite` | Arguments for modify parameters are wrapped in the modify operator. |

Each setting decides whether that kind of argument is marked at the call site. When it is `true`, the argument must be wrapped in the operator. When it is `false`, the operator may not be used at all, so that every call site in the project looks the same.

With the settings above and the [lowercase tag header](tag-headers.md#the-premade-headers), whose tags are `receives` (move), `initializes` (out) and `borrows` (modify) and whose operators are `give()`, `overwrite()` and `lend()`, the code looks like this:

```c
struct Data
{
  int i, j, k;
};
typedef struct Data Data;

int get_data_i(const Data* data);                           // OK: const, no tag needed
void initialize_data(initializes Data** data);
void edit_data(borrows Data* data, int i, int j, int k);
void give_data_to_other_section(receives Data* data);
void no_tag(Data* data);                                    // Reported: non-const pointer without a tag

void example(void)
{
  Data* data = NULL;
  initialize_data(overwrite(&data));                        // OK
  edit_data(data, 3, 5, 7);                                 // OK: the modify operator is disabled
  int i = get_data_i(data);                                 // OK
  give_data_to_other_section(give(data));                   // OK

  Data* other = NULL;
  initialize_data(&other);                                  // Reported: missing overwrite()
  edit_data(lend(other), 1, 2, 3);                          // Reported: the modify operator is disabled
  initialize_data(give(&other));                            // Reported: wrong operator
  get_data_i(give(other));                                  // Reported: the parameter has no tag
}
```

Const pointers need no tags, and in this config the `lend()` operator is disabled. This is a good middle ground: ownership transfers and outputs stand out at every call site, while ordinary modification stays quiet. The code documents itself, and when a parameter later changes to or from `receives` or `initializes`, every call site that was not updated is reported.

A function declared more than once (e.g. in a header and in the source file) must use the same tags every time. Functions declared in system headers and `third_party_includes` folders have no tags, so passing a pointer to them needs no operator, and using one is reported.

### Use after move

Once a pointer has been moved, the called function owns it, so the pointer may not be used again (read, moved again, compared, passed on, its address taken) until it has been reassigned, either with `=` or by declaring it again.

Passing its address to an `out` parameter does not count as reassigning it, since the called function is not guaranteed to write a new value, for example when it fails and returns an error code early. Reassign it explicitly first, e.g. `data = NULL;`.

The check follows the control flow of the function: a use is reported if the pointer may have been moved on any path that reaches it, so a move inside one branch of an `if` counts after the `if`, unless that branch returns, and a move inside a loop counts in the next iteration. Only plain pointer variables are followed, not struct fields or array elements.

```c
void example(void)
{
  Data* data = NULL;
  initialize_data(overwrite(&data));
  give_data_to_other_section(give(data));
  edit_data(data, 1, 2, 3);                 // Reported: data was moved
  initialize_data(overwrite(&data));        // Reported: out may not write a new value
  data = NULL;                              // OK: data is reassigned
  initialize_data(overwrite(&data));        // OK
  edit_data(data, 1, 2, 3);                 // OK
}
```

### Moving a copied pointer

A pointer that has been copied to or from another pointer may not be moved, since the other pointer would still point at what the called function now owns. This applies to both pointers. A copy is made when a pointer variable or struct field is given the value of another one: by a declaration (`Data* alias = data;`), an assignment (`holder.data = data;`), an initializer (`Holder holder = { data };`), or by copying a whole struct, which copies its pointer fields (`Holder copy = holder;`).

The copy ends when either pointer is given a new value, e.g. `alias = NULL;`. Then the other one may be moved. Like use after move, the check follows the control flow, so a copy in one branch of an `if` counts after the `if`.

```c
void example(void)
{
  Data* data = NULL;
  initialize_data(overwrite(&data));
  Data* alias = data;
  give_data_to_other_section(give(data));   // Reported: alias still points at data

  Data* owner = NULL;
  initialize_data(overwrite(&owner));
  Data* handle = owner;
  give_data_to_other_section(give(handle)); // Reported: owner still points at it

  Data* first = NULL;
  initialize_data(overwrite(&first));
  Data* second = first;
  first = NULL;
  give_data_to_other_section(give(second)); // OK: first was given a new value
}
```

### Moving a struct field

A pointer field that is moved, through `.` or `->`, must be given a new value right after the move, so that the struct never holds a pointer to memory it no longer owns. The statement right after the one with the move must assign the field with `=`. Usually that is `NULL`, but any value other than the field itself will do, e.g. the next element of a list. When one statement moves several fields, of one or more structs, their assignments follow one after another, in any order. Passing the field's address to an `out` parameter does not count, since the function is not guaranteed to write a new value.

A move in a `return` statement, or as the last statement of its block, can not be followed by its assignment, and is reported. To move a field out of a function, first take it into a local variable and set the field to `NULL`, then move the local variable. Setting the field to `NULL` also ends the [copy](#moving-a-copied-pointer) into the local variable.

```c
void example(borrows List* list, borrows Pair* pair, receives Data* spare)
{
  give_data_to_other_section(give(list->head));                   // OK
  list->head = NULL;

  give_two_to_other_section(give(pair->first), give(list->tail)); // OK: in any order
  list->tail = NULL;
  pair->first = spare;                                            // OK: any new value

  give_data_to_other_section(give(pair->second));                 // Reported: another statement comes first
  edit_data(pair->first, 1, 2, 3);
  pair->second = NULL;
}
```

### Borrowed pointers

A parameter tagged `borrows` or `initializes` is only borrowed by the function, so it may not be moved away. The same goes for what an `initializes` parameter points to.

```c
void process(borrows Data* data)
{
  give_data_to_other_section(give(data));   // Reported: the function does not own data
}
```

The movement tags may only be written on parameters, of function declarations and definitions and of function pointer types, not on variables, struct fields, functions or typedefs.

### Pod creators

When the [struct resource management rule](#struct-resource-management-rule-raii) is on too, the creator function of a [pod struct](#pod-structs) may not have a parameter tagged `receives` (move). A pod struct owns no resources, so its creator has nothing to take ownership of a pointer for. It is reported at the level of this rule. The other tags, and the creators of raii and free structs, are not affected.

```c
position position_pod(int x, int y);                 // OK
buffer buffer_pod(receives Data* data, int size);    // Reported: a pod creator may not take ownership
marker marker_pod(const Data* source);               // OK
```

### Tags on function pointers

A function pointer type carries the tags on its parameters, just like a function, and a non-const pointer parameter of it needs a tag too. A function, or another function pointer, that is assigned or passed to a function pointer must have the same tags, parameter by parameter. This is checked wherever a function pointer gets its value: initializations (also of arrays and struct fields), compound literals such as `(handlers){ .on_event = function }`, assignments, arguments and return statements. A cast does not hide a mismatch.

```c
typedef void (*consumer_t)(receives Data* data);

void give_data_to_other_section(receives Data* data);
void edit_data(borrows Data* data, int i, int j, int k);
void borrow_data(borrows Data* data);

consumer_t consumer = give_data_to_other_section;  // OK: the same tags
consumer_t wrong = borrow_data;                    // Reported: borrows instead of receives
void register_consumer(consumer_t consumer);
register_consumer(borrow_data);                    // Reported: same
```

Calls through a function pointer follow the tags of its type, exactly like calls to a function: the call site operators, the use of a pointer after it was moved, and the borrowed parameters that may not be moved. The function pointer can be a variable, a parameter, a struct field, an array element or a conditional expression. A call through a conditional expression, e.g. `(flag ? first : second)(give(data))`, follows the tags its branches agree on, and is reported when they tag a parameter differently, since the call site can only follow one set of tags.

```c
void example(consumer_t consumer)
{
  Data* data = NULL;
  initialize_data(overwrite(&data));
  consumer(give(data));                     // OK: the operator of consumer_t's receives parameter
  edit_data(data, 1, 2, 3);                 // Reported: data was moved to consumer
}
```

A function pointer type from a system header or a `third_party_includes` folder has no tags, so any function may be assigned or passed to it.

The tags and operators are ordinary macros, so any names can be used. See [writing your own tags](tag-headers.md#writing-your-own-tags).

## Struct resource management rule (RAII)

This rule brings RAII (Resource Acquisition Is Initialization) to C. Every struct of the project belongs to one of three categories, decided by its creator function: a function named after the struct plus a suffix from the config. A struct must have exactly one creator function.

| Category | Creator function | What the rule guarantees |
|----------|------------------|--------------------------|
| [Pod struct](#pod-structs) | `<struct>_pod` | Every value comes from a function or another value, never from an uninitialized or hand-built struct |
| [Raii struct](#raii-structs) | `<struct>_make` | Exactly one owner, a complete set of lifecycle functions, and destruction before every scope exit |
| [Free struct](#free-structs) | `<struct>_init` | No restrictions, for code that needs full control |

The suffixes in the table are the ones used by the `default` preset. A struct's creator and its other lifecycle functions are exempt from the struct's rules, since that is where the struct is actually built: they may, for example, set fields directly or use brace initializers.

**Config key:** `struct_resource_management` · **Codes:** `CCW1001`–`CCW1006`, `CCW1101`–`CCW1112`, `CCW1201`–`CCW1211`, `CCW1301`–`CCW1305`

```yaml
rules:
  struct_resource_management:
    level: Error
    pod_struct_creator_suffix: _pod
    raii_struct_creator_suffix: _make
    raii_struct_destroyer_suffix: _destroy
    raii_may_only_destroy_value_ref: true
    raii_struct_copy_suffix: _copy
    raii_struct_move_suffix: _move
    raii_may_only_move_value_ref: true
    raii_struct_return_suffix: _return
    raii_struct_valid_suffix: _valid
    raii_use_after_destroy: false
    raii_struct_array_destroyer_suffix: _destroy_array
    allow_raii_struct_arrays: true
    raii_destroy_in_reverse_order: true
    raii_standardized_destroy_definitions: true
    free_struct_creator_suffix: _init
```

| Option | Description |
|--------|-------------|
| `pod_struct_creator_suffix` | Suffix of the creator function of a [pod struct](#pod-structs), e.g. `_pod`. May not be empty. |
| `raii_struct_creator_suffix` | Suffix of the make function of a [raii struct](#raii-structs), e.g. `_make`. May not be empty. |
| `raii_struct_copy_suffix` | Suffix of the copy function, e.g. `_copy`. May not be empty. |
| `raii_struct_move_suffix` | Suffix of the move function, e.g. `_move`. May not be empty. |
| `raii_struct_destroyer_suffix` | Suffix of the destroy function, e.g. `_destroy`. May not be empty. |
| `raii_struct_return_suffix` | Suffix of the return function, e.g. `_return`. May not be empty. |
| `raii_struct_valid_suffix` | Suffix of the validation function, e.g. `_valid`. May not be empty. |
| `free_struct_creator_suffix` | Suffix of the init function of a [free struct](#free-structs), e.g. `_init`. May not be empty. |
| `allow_raii_struct_arrays` | Allow arrays of raii structs outside of structs, see [arrays of raii structs](#arrays-of-raii-structs). When `false`, they are only allowed inside structs. |
| `raii_struct_array_destroyer_suffix` | Suffix of the array destroy function, e.g. `_destroy_array`. May not be empty when `allow_raii_struct_arrays` is `true`. |
| `raii_use_after_destroy` | When `false`, a raii struct may not be used after it has been destroyed. See [lifetime settings](#lifetime-settings). |
| `raii_destroy_in_reverse_order` | Raii structs must be destroyed in reverse declaration order. |
| `raii_may_only_move_value_ref` | The move function may only be given the address of a variable. |
| `raii_may_only_destroy_value_ref` | The destroy function may only be given the address of a variable. |
| `raii_standardized_destroy_definitions` | A destroy function destroys each of its raii fields exactly once, side by side in one block. See [standardized destroy functions](#standardized-destroy-functions). |

All the function suffixes must be different from each other.

### Pod structs

Plain Old Data (pod) structs come with only one rule: they must always be initialized correctly. A pod struct has a creator function of the form `struct <name> <name>_pod(...)`. Every value of a pod struct must come from a function return value, another struct variable or a field, whether it initializes a variable, is passed as an argument or is returned, so `{0}` and brace literals are only allowed in the `_pod` function. This rules out uninitialized and half-initialized structs.

```c
typedef struct position
{
  int x, y, z;
} position_t;

static inline position_t position_pod(int x, int y, int z)
{
  return (position_t){x, y, z};                   // OK: brace literals are allowed in the _pod function
}

static inline position_t position_default(void)
{
  return position_pod(0, 0, 0);                   // Builds on the creator function above
}

int position_sum(position_t position);

typedef struct path
{
  position_t start, end;
} path_t;

path_t path_default(void);

void foo(void)
{
  position_t no_init_pos;                         // Reported: not initialized
  position_t zero_pos = {0};                      // Reported: not from a function or variable
  position_t pos = position_default();            // OK
  pos = position_pod(1, 2, 3);                    // OK: pod structs may be reassigned
  position_t pos_2 = pos;                         // OK: copied from another variable
  position_t* pos_ptr = &pos;                     // OK
  position_t pos_3 = *pos_ptr;                    // OK: copied through a pointer
  path_t path = path_default();                   // OK
  position_t start = path.start;                  // OK: copied from a field
  int start_sum = position_sum(path.start);       // OK: a field as an argument
  pos_2.y = 10;                                   // OK
  int sum = position_sum((position_t){1, 2, 3});  // Reported: brace literal as an argument
}
```

A pod struct may only contain other pod structs, not raii or free structs, since a copy of a pod struct is just a copy of its bytes.

Arrays of pod structs are allowed outside of structs when every element, at every level of an array of arrays, is explicitly initialized from a function return value, another struct variable or a field, e.g. `position_t line[2] = { position_pod(0, 0, 0), pos };`, so `{0}`, brace literals and missing elements are not allowed. Arrays of raii structs follow their own rules, see [arrays of raii structs](#arrays-of-raii-structs).

### Raii structs

Raii structs own resources, such as memory, files or locks, and come with many more rules. A raii struct variable is initialized *once*, from a function call, and is never reassigned, although its contents can still be changed. No scope may be exited without either destroying or returning it. A raii struct needs a full set of lifecycle functions, which must be declared, while defining them is up to the project:

| Function | Example | Purpose |
|----------|---------|---------|
| make | `dynamic_string_make` | Creates the struct and initializes every member, like a constructor. |
| copy | `dynamic_string_copy` | Creates an independent copy of another struct, like a copy constructor. |
| move | `dynamic_string_move` | Creates a new struct and transfers the resources to it, like a move constructor. Good practice is to leave the source valid but empty. |
| destroy | `dynamic_string_destroy` | Cleans up and frees every resource, like a destructor. |
| return | `dynamic_string_return` | Moves ownership out of a function in a return statement: in effect a copy and a destroy in one, which is easy for the compiler to optimize. |
| valid | `dynamic_string_valid` | Tells whether the struct is in a valid, usable state, much like catching an exception from a constructor. A raii struct may be valid or invalid, but never in an illegal state. |

Every function except make must take a pointer to the struct, named `self`, as its first parameter. WorkshopC checks that all of the functions exist. Here is a simple example:

```c
struct dynamic_string
{
    char* data;
    size_t size;
    size_t capacity;
};
typedef struct dynamic_string d_str;

// These functions are mandatory
d_str dynamic_string_make(const char* c_str);   // Allocates and copies c_str
d_str dynamic_string_copy(const d_str* self);   // Allocates a new copy of self
d_str dynamic_string_move(d_str* self);         // Moves the data to a new object and empties self
void  dynamic_string_destroy(d_str* self);      // Frees the data, leaving self empty
d_str dynamic_string_return(d_str* self);       // Moves the data out of a function
_Bool dynamic_string_valid(d_str* self);        // Checks the internal state, e.g. that size <= capacity

// Any other functions are up to the struct
const char* dynamic_string_data(const d_str* self);
void dynamic_string_add(d_str* self, const char* c_str);
void dynamic_string_concat(d_str* self, const d_str* addition);
void dynamic_string_reset(d_str* self);
```

With the raii struct set up, here is how it is used:

```c
// Sets up a dynamic string and moves its ownership to a print function
void take_dynamic_string_and_print(d_str string_as_value);

void print_big_greeting(void)
{
  d_str greeting = dynamic_string_make("Hello, world!");
  if (!dynamic_string_valid(&greeting))
  {
    dynamic_string_destroy(&greeting);
    return;
  }

  dynamic_string_add(&greeting, " Hello, everyone!");
  take_dynamic_string_and_print(dynamic_string_move(&greeting));

  dynamic_string_destroy(&greeting);
}
```

A raii struct owns its resources, so every value of it must have exactly one owner, and the rules follow from that:
- A variable must be initialized from a function return value, e.g. the make, copy or move function, never from another variable (`d_str b = a;` would make two owners of the same memory).
- It may not be reassigned once initialized, not even through a pointer, a field or an array element, since the old value would leak.
- It may only be passed by value as a function return value, e.g. `take(dynamic_string_copy(&s))` or `take(dynamic_string_move(&s))`, so that the called function gets its own value, which it then must destroy.
- Every variable, and every parameter passed by value, must be destroyed or returned before every scope exit (the end of its block, `return`, `break`, `continue` or `goto`). A `goto` exits the blocks that do not contain its label, so the common `goto cleanup;` pattern works as long as the label is in the same block as the variables it destroys.
- A function returning a raii struct must return a function call, and a local variable is returned with the return function.
- A raii struct returned by a function may not be discarded or have a field read directly on the returned value, since it could then never be destroyed.

```c
d_str make_greeting(const char* name)
{
  d_str greeting = dynamic_string_make("Hello, ");
  dynamic_string_add(&greeting, name);
  return dynamic_string_return(&greeting);          // OK: ownership moves to the caller
}

void use_greetings(void)
{
  d_str a = make_greeting("world");                 // OK
  d_str b = a;                                      // Reported: two owners of the same memory
  a = dynamic_string_make("again");                 // Reported: the old value would leak
  make_greeting("world");                           // Reported: the result can never be destroyed
  size_t size = make_greeting("world").size;        // Reported: same
  d_str c = dynamic_string_return(&a);              // Reported: return functions only belong in return statements

  dynamic_string_destroy(&a);
}                                                   // Reported: b and c are never destroyed
```

> [!TIP]
> Raii structs work best together with the [private rule](#private-rule) or the [private alternative rule](#private-alternative-rule): a major point of a raii struct is that its internal state is always under the control of its own functions.

#### Deleting copy, move or return

Not every raii struct can be copied or moved. A struct that owns a mutex, a file handle or a hardware peripheral may not be copyable, and a function that is not needed yet does not have to be written yet. WorkshopC still requires the copy, move and return functions to be declared, but a macro can declare them only while WorkshopC analyzes the code, with [`WORKSHOPC_PARSING`](#adjusting-code-for-analysis). The real build then has no such function, so a call to it fails to compile, which is better than a function that is declared but never defined:

```c
#ifdef WORKSHOPC_PARSING
#define DELETE_RAII_COPY(name)   struct name name##_copy(const struct name* self)
#define DELETE_RAII_MOVE(name)   struct name name##_move(struct name* self)
#define DELETE_RAII_RETURN(name) struct name name##_return(struct name* self)
#else
#define DELETE_RAII_COPY(name)   _Static_assert(1, #name "_copy is deleted")
#define DELETE_RAII_MOVE(name)   _Static_assert(1, #name "_move is deleted")
#define DELETE_RAII_RETURN(name) _Static_assert(1, #name "_return is deleted")
#endif
```

```c
typedef struct file_handle
{
  int descriptor;
} file_handle;

file_handle file_handle_make(const char* path);
void file_handle_destroy(file_handle* self);
file_handle file_handle_return(file_handle* self);
_Bool file_handle_valid(const file_handle* self);

DELETE_RAII_COPY(file_handle);    // OK: a file handle can not be copied
DELETE_RAII_MOVE(file_handle);    // OK: not needed yet
```

- The macros take the struct's own name, not a typedef, so the struct must have one, e.g. `typedef struct file_handle { ... } file_handle;`.
- The `;` after the macro belongs to the code that uses it. In the real build the macro becomes an always true `_Static_assert`, which needs C11, so that the `;` does not stand on its own.
- Only copy, move and return can be deleted. The make, destroy and valid functions must exist, since the ownership rules depend on them.
- WorkshopC sees the declaration, so it does not report a call to a deleted function. The compiler does, in the real build.

> [!NOTE]
> WorkshopC comes with no premade header for these macros, since the function suffixes are set in the config and differ between presets, e.g. `_copy` in `default` and `Copy` in `opinionated`. Write the macros once for your project, with the suffixes from your config's `raii_struct_copy_suffix`, `raii_struct_move_suffix` and `raii_struct_return_suffix`.

#### Raii structs with raii fields

A raii struct can hold other raii structs as fields. Since a raii value may not be reassigned, such a field is set in the initializer of the value that the make, copy or move function builds, and not assigned afterwards. Only these lifecycle functions may initialize the struct with a brace initializer.

```c
typedef struct message
{
  d_str text;
  int priority;
} message;

message message_make(const char* text, int priority)
{
  message self = { .text = dynamic_string_make(text), .priority = priority };  // OK
  return self;
}

message message_copy(const message* self)
{
  return (message){ .text = dynamic_string_copy(&self->text), .priority = self->priority };  // OK
}

message message_make_later(const char* text)
{
  message self = message_make("", 0);
  self.text = dynamic_string_make(text);            // Reported: the old text would leak
  return message_return(&self);
}
```

If building a field needs steps first, e.g. checking that it is valid, do them with a local variable and move it into the initializer: `message self = { .text = dynamic_string_move(&text), .priority = 1 };`.

#### Arrays of raii structs

With `allow_raii_struct_arrays: true` and `raii_struct_array_destroyer_suffix` set (e.g. `_destroy_array`), one-dimensional arrays of raii structs are allowed outside of structs. Every element must be initialized from a function return value, and the array must be destroyed before every scope exit with the array destroy function, whose first argument must be the array itself and whose second argument must be the size of the array. The function is not required for a raii struct in general, only once an array of it is declared, and it must have the signature `void <struct name><suffix>(<struct name>* self, size_t n)`.

```c
void dynamic_string_destroy_array(d_str* self, size_t n);

void foo(void)
{
  d_str names[2] = { dynamic_string_make("a"), dynamic_string_make("b") };   // OK

  dynamic_string_destroy_array(names, sizeof(names) / sizeof(names[0]));     // OK: any expression with the right value works, e.g. 2
}
```

An array destroyed element by element, with the wrong size, or through a pointer is reported, as is an array of arrays of raii structs. With the span struct rule's `require_span_immediately_after_array: true`, the array destroy function is still given the array itself, not its span.

#### Lifetime settings

These settings tighten how raii structs are moved, destroyed and used afterwards.

- `raii_use_after_destroy: false`: a raii struct that has been destroyed with its destroy function may not be referenced again, neither passed to a function nor accessed through a field. The same goes for a span or const span initialized from a destroyed [array of raii structs](#arrays-of-raii-structs), e.g. `vectors_span.data[0]` after `int_vector_destroy_array(vectors, 2)`. Only the span declared from the array is followed, not copies of it.
- `raii_destroy_in_reverse_order: true`: raii structs must be destroyed in reverse declaration order. If `a`, `b` and `c` are initialized in that order, they must be destroyed as `c`, `b`, then `a`. When `false`, the destruction order is unrestricted.
- `raii_may_only_move_value_ref: true`: a raii move function may only be given the address of a variable, e.g. `dynamic_string_move(&name)`, so that only an object owned by the calling scope can be moved from. A pointer (`dynamic_string_move(name_ptr)`), a struct field (`&holder.name`, `&holder->name`) or anything behind a pointer (`&*name_ptr`) is reported. The lifecycle functions of any struct may still work through their `self` pointer, e.g. a struct's own return function moving `self`, or the move function of a struct with a raii field moving `&self->field`.
- `raii_may_only_destroy_value_ref: true`: the same for the destroy function, e.g. `dynamic_string_destroy(&name)`. The destroy function of a struct with raii fields may still destroy them with `&self->field`. Fields of [free structs](#free-structs) are not affected and may be destroyed from anywhere, e.g. `dynamic_string_destroy(&holder.name)` or `dynamic_string_destroy(&holder->name)`, since free structs come with no rules.

#### Standardized destroy functions

With `raii_standardized_destroy_definitions: true`, the destroy function of a raii struct must destroy each of its raii fields exactly once. A field is destroyed with `<field struct>_destroy(&self->field)`, or `<field struct>_destroy_array(self->field, count)` for an array field. All the field destroys are statements side by side in one block, either:
- the body of the destroy function, or
- the then-block of an `if` statement directly in the body. The condition can be anything.

Destroys in a nested `if`, an `else` branch, a nested `{ }` block, a loop or a `switch` are reported, and so are destroys split between two blocks. A `return` or `goto` before all the destroys, e.g. an early return when `self` is null, skips the struct as a whole and is allowed by this rule, though e.g. the [single return rule](#single-return-rule) may not allow it. A `return` or `goto` between the destroys is reported, and so is a label between them: both would leave the struct partly destroyed. Other statements, e.g. freeing a buffer, may come anywhere. With `raii_destroy_in_reverse_order`, the fields must be destroyed in reverse declaration order. Pod, free and pointer fields are not owned raii fields and are not affected.

A raii struct is therefore always either fully alive or fully destroyed, never partly. Its make function brings every raii field to life, and its destroy function destroys every raii field together. A condition in the destroy function decides whether the struct as a whole is destroyed, e.g. when it is null or has been moved from, never which of its fields are. When every raii type has a valid empty state, a moved-from struct usually needs no condition at all: its fields are already empty, so destroying them does nothing.

```c
struct owner { first_t first; char* buffer; second_t second; bool moved; };

void owner_destroy(owner_t* self)
{
    if (self != NULL && !self->moved)       // OK: any condition
    {
        second_destroy(&self->second);      // OK: reverse declaration order
        memory_free(self->buffer);          // OK: other statements may come in between
        first_destroy(&self->first);        // OK
    }
}

void split_destroy(split_t* self)
{
    second_destroy(&self->second);

    if (self->buffer != NULL)
    {
        first_destroy(&self->first);        // Reported: not in the same block as 'second'
    }
}

void early_return_destroy(early_return_t* self)
{
    if (self == NULL)
        return;                             // OK: skips the struct as a whole

    second_destroy(&self->second);          // OK
    first_destroy(&self->first);            // OK
}

void return_between_destroy(return_between_t* self)
{
    second_destroy(&self->second);

    if (self->moved)
        return;                             // Reported: 'first' would be left alive

    first_destroy(&self->first);
}
```

### Free structs

Finally, free structs come with no rules at all for how they are used. Pod and raii structs follow a safety-first approach and trust the compiler to handle copy elision and `static inline` functions effectively. A free struct instead gives the programmer complete freedom, for example to optimize where it is needed. The only requirement is an init function of the form `<any return type> <struct name>_init(<struct name>* self, ...)`:

```c
struct graphics_renderer
{
  // Fields
};
typedef struct graphics_renderer graphics_renderer;
typedef const char* graphics_error_msg;
int graphics_renderer_init(graphics_renderer* self, int arg1, float arg2, graphics_error_msg* result_msg);
```

The suffix can be made deliberately long to make users stop and think, e.g. `free_struct_creator_suffix: _init_manual_management_struct`.

The init function sets up the free struct itself, so it may assign the struct's own members freely, even when a member is a raii struct. Normally reassigning a raii struct is forbidden since the old value would leak, but in the init function there is no old value yet.

### Standard library and third-party structs

Structs from the standard library or third-party libraries are not affected by this rule. To make sure they are always initialized and their resources released, they can be wrapped in pod and raii structs.

## Interfaces rule

This rule standardizes how interfaces and their vtables are written. A vtable is a set of functions, and an interface pairs an object with the vtable of functions that work on it. With one standard layout, dynamic dispatch looks the same everywhere in the project and is easy to inspect.

**Config key:** `interfaces` · **Codes:** `CCW1401`–`CCW1406`, `CCW1501`–`CCW1504`

```yaml
rules:
  interfaces:
    level: Error
    vtable_suffix: _vtable
    interface_suffix: _interface
    const_interface_suffix: _const_interface
    interface_must_have_fields_that_are_private_alternative: true
```

| Option | Description |
|--------|-------------|
| `vtable_suffix` | Structs whose names end with this suffix are [vtables](#vtables). May not be empty. |
| `interface_suffix` | Structs whose names end with this suffix are [interfaces](#interfaces). May not be empty. |
| `const_interface_suffix` | Structs whose names end with this suffix are const interfaces. It is checked first, so it may end with `interface_suffix`. May not be empty. |
| `interface_must_have_fields_that_are_private_alternative` | Both interface fields must be private for the [private alternative rule](#private-alternative-rule), which must then be on. |

The suffixes must be different from each other, and from the suffixes and names set for the array struct and span struct rules.

### Vtables

A vtable struct is a struct whose name ends with `vtable_suffix`. It may only hold function pointers, directly or through a typedef. Every function pointer takes a `void*` or `const void*` as its first parameter: the object the function works on.

A vtable variable must be `static const`, and it must be initialized at declaration with braces, with an element for every function pointer. Designated elements and `&function` are fine. Every element must be an actual function, never `NULL` or `0`. A vtable is used through a pointer to it, so a vtable parameter passed by value is reported too.

When [struct resource management](#struct-resource-management-rule-raii) is on, a vtable struct without a creator function (no make, pod or init function) is a free struct instead of an invalid one.

```c
struct shape_vtable
{
  double (*area)(const void* self);             // OK
  void (*scale)(void* self, double factor);     // OK
  int count;                                    // Reported: not a function pointer
  void (*reset)(void);                          // Reported: no void* or const void* first
};

static const shape_vtable_t circle_vtable = { circle_area, circle_scale };           // OK
static const shape_vtable_t square_vtable = { .area = square_area, .scale = NULL };  // Reported: NULL is not a function
const shape_vtable_t shared_vtable = { circle_area, circle_scale };                  // Reported: not static
static const shape_vtable_t partial_vtable = { circle_area };                        // Reported: 'scale' is missing
```

### Interfaces

An interface struct is a struct whose name ends with `interface_suffix`. It holds exactly two fields, in this order:
- a `void*` named `object`: the object the functions work on;
- a pointer to a const vtable struct named `vtable`, e.g. `const shape_vtable_t* vtable`. Any vtable struct will do.

A const interface, whose name ends with `const_interface_suffix`, gives read-only access to its object, so it holds a `const void* object` instead. The const suffix is checked first, so it may end with the interface suffix, as `_const_interface` ends with `_interface`.

With `interface_must_have_fields_that_are_private_alternative: true` and the [private alternative rule](#private-alternative-rule) on, both fields must be marked private. Only the interface's own functions can then reach the object and the vtable.

```c
struct shape_interface
{
  PRIVATE void* object;                   // OK
  PRIVATE const shape_vtable_t* vtable;   // OK
};

struct shape_const_interface
{
  PRIVATE const void* object;             // OK: a const interface holds a const void*
  PRIVATE const shape_vtable_t* vtable;   // OK
};

struct broken_interface
{
  PRIVATE void* self;                     // Reported: must be named object
  PRIVATE shape_vtable_t* vtable;         // Reported: the vtable must be const
  PRIVATE int count;                      // Reported: only two fields are allowed
};
```

## Restricted malloc rule

This rule keeps all dynamic memory handling in one place. The memory functions `malloc`, `calloc`, `realloc`, `free`, `strdup`, `strndup`, `asprintf`, `getline` and `realpath` may only be used inside the functions listed in the config. All of them either allocate or free memory, some of them without it being obvious (`getline` grows the buffer it is given and `realpath` allocates when given `NULL`). With allocation behind a few wrappers, a project can add tracking, pools or failure handling in one place.

**Config key:** `restricted_malloc` · **Codes:** `CCW1601`

```yaml
rules:
  restricted_malloc:
    level: Error
    list_of_allowed_malloc_functions:
      - memory_alloc
      - memory_alloc_array
      - memory_free
```

| Option | Description |
|--------|-------------|
| `list_of_allowed_malloc_functions` | The functions in which the memory functions may be used. With an empty list, they may not be used anywhere. |

Only the name of the enclosing function is checked. It must match a list entry exactly (case sensitive), and there are no requirements on its return type or parameters.

```c
void* memory_alloc(size_t size)
{
    return malloc(size);                        // OK: inside an allowed function
}

void memory_free(void* memory)
{
    free(memory);                               // OK
}

void foo(void)
{
    char* name = strdup("name");                // Reported
    void* (*allocator)(size_t) = malloc;        // Reported: taking the function as a pointer is a use too
    void* memory = memory_alloc(16);            // OK: calls the wrapper
    memory_free(memory);                        // OK
}
```

A macro that expands to one of the functions counts as using it where the macro is used. Uses inside system headers and `third_party_includes` folders are not checked.

## Single return rule

This rule requires every function to have a single return statement, which must be the last statement of the function. A function with one exit point is easier to follow, and cleanup code placed before the return can never be skipped.

**Config key:** `single_return` · **Codes:** `CCW1701`–`CCW1702`

```yaml
rules:
  single_return:
    level: Warning
    allow_early_return: true
    require_return_for_void: true
```

| Option | Description |
|--------|-------------|
| `allow_early_return` | Allow guard clauses: returns from an `if` without an `else` directly in the function's top-level block. |
| `require_return_for_void` | `void` functions must also end with `return;`. |

```c
int sum(const int* values, int count)
{
  int result = 0;

  for (int i = 0; i < count; ++i)
  {
    if (values[i] < 0)
      return -1;      // Reported: return inside a loop

    result += values[i];
  }

  return result;      // OK: the final return
}
```

With `allow_early_return: true`, guard clauses are allowed as well: an `if` without an `else`, placed directly in the function's top-level block, may return from its branch, either as its only statement or as the last statement of its block. Guard clauses do not have to come before all other code. A macro used as a statement at the top level, such as `RETURN_IF_NULL(ptr);`, counts as an early return too.

```c
int get_value(const int* value)
{
  if (!value)
    return -1;        // OK with allow_early_return

  if (*value > 100)
  {
    log_error("too large");
    return -2;        // OK with allow_early_return
  }

  return *value;      // OK: the final return
}

int pick(int flag)
{
  if (flag)
    return 1;         // Reported: an if with an else is not a guard clause
  else
    return 2;         // Reported
}                     // Reported: the function does not end with a return
```

With `require_return_for_void: true`, `void` functions must also end with `return;`. When it is `false`, the final return is optional for them, but any other return still has to follow the rules above.

## Strict switch rule

This rule requires every `switch` statement to have a `default` case, and every case (including `default`) to end with a `break` or `return`, so that a missing `break` can never silently fall through into the next case, and no value is left unhandled.

**Config key:** `strict_switch` · **Codes:** `CCW1801`–`CCW1802`

```yaml
rules:
  strict_switch:
    level: Warning
```

This rule has no options besides `level`.

```c
switch (value)
{
  case 1:
  case 2:             // OK: stacked labels share one body
    result = 1;
    break;
  case 3:             // Reported: falls through into case 4
    result = 3;
  case 4:
    result += 4;
    break;
}                     // Reported: no default case
```

A case also ends correctly with `continue` or `goto`, with a call to a function that never returns (such as `abort()` or `exit()`), with a block whose last statement ends the case, or with an `if`/`else` where both branches end the case. A nested `switch` as the last statement of a case is reported, since its `break` only leaves the inner switch.

## Global variable rule

This rule enforces conventions for global (file scope) variables and static local variables, which makes them easy to spot and limits who can change them. Each option can be enabled separately.

**Config key:** `global_variable` · **Codes:** `CCW1901`–`CCW1905`

```yaml
rules:
  global_variable:
    level: Warning
    must_be_static: true
    must_be_const: false
    forbid_static_in_header: true
    mutable_require_prefix: true
    mutable_prefix: g_
    mutable_must_be_caps: false
    const_require_prefix: false
    const_prefix: G_
    const_must_be_caps: true
    mutable_local_static_require_prefix: true
    mutable_local_static_prefix: s_
    mutable_local_static_must_be_caps: false
    const_local_static_require_prefix: true
    const_local_static_prefix: s_
    const_local_static_must_be_caps: false
```

| Option | Description |
|--------|-------------|
| `must_be_static` | File scope variables must have internal linkage through `static`. An `extern` declaration is therefore reported too. |
| `must_be_const` | File scope variables must be deeply const, see below. Does not apply to static locals. |
| `forbid_static_in_header` | No static globals, or static locals in functions, defined in a header, since every including file would get its own copy. |
| `mutable_require_prefix` | Mutable globals must start with `mutable_prefix`. |
| `mutable_prefix` | The prefix of mutable globals, e.g. `g_`. May not be empty when `mutable_require_prefix` is `true`. |
| `mutable_must_be_caps` | Mutable globals are written in capital letters. |
| `const_require_prefix` | Deeply const globals must start with `const_prefix`. |
| `const_prefix` | The prefix of deeply const globals, e.g. `G_`. May not be empty when `const_require_prefix` is `true`. |
| `const_must_be_caps` | Deeply const globals are written in capital letters. |
| `mutable_local_static_require_prefix` | Mutable static locals must start with `mutable_local_static_prefix`. |
| `mutable_local_static_prefix` | The prefix of mutable static locals, e.g. `s_`. May not be empty when `mutable_local_static_require_prefix` is `true`. |
| `mutable_local_static_must_be_caps` | Mutable static locals are written in capital letters. |
| `const_local_static_require_prefix` | Deeply const static locals must start with `const_local_static_prefix`. |
| `const_local_static_prefix` | The prefix of deeply const static locals. May not be empty when `const_local_static_require_prefix` is `true`. |
| `const_local_static_must_be_caps` | Deeply const static locals are written in capital letters. |

Details:
- **Deeply const** means that the variable and everything it points to is const: a pointer must be const on every level, both the pointer itself and what it points to, e.g. `const int* const`. An array is classified by its elements. A variable that is not deeply const is mutable.
- Naming is chosen separately for four kinds of variables: mutable globals (`mutable_*`), deeply const globals (`const_*`), mutable static locals (`mutable_local_static_*`) and deeply const static locals (`const_local_static_*`). Static locals do not inherit the settings for globals.
- When a required prefix is present, capitalization is only checked after the prefix. A prefix that is not required may be empty (`""`), but is still written out, like every option.
- Local variables that are not static are never checked. A variable declared more than once (e.g. `extern` in a header and the definition in the source file) is only checked once, at its definition.

Two common setups are internal state and constants:

```c
// mutable_require_prefix: true, mutable_prefix: g_, must_be_static: true
static int g_counter = 0;               // OK
static int counter = 0;                 // Reported: missing the mutable prefix

// const_must_be_caps: true, const_require_prefix: false
const int MAX_SIZE = 10;                // OK
const char* const APP_NAME = "app";     // OK
const int lower_case = 1;               // Reported: not capitalized
const char* NAME_POINTER = "name";      // Reported: the pointer itself is mutable, so it needs the g_ prefix
```

## Reference pointer rule

A reference pointer is a pointer parameter that always points to a real object, so it can never be null. It is marked with a tag, a macro that WorkshopC sees as an annotation. A [premade tag header](tag-headers.md) can be used as is, and the macro name can be changed freely. References make the difference between "may be null" and "is always an object" part of a function's signature, and the call sites are checked against it.

**Config key:** `reference_pointer` · **Codes:** `CCW2001`–`CCW2012`

```yaml
rules:
  reference_pointer:
    level: Warning
    disable_null_check_rule_for_reference_pointers: true
    allow_direct_ptr_in_if_statement: false
    only_allow_reference_pointers_as_function_parameters: false
```

| Option | Description |
|--------|-------------|
| `disable_null_check_rule_for_reference_pointers` | The [null check rule](#null-check-rule), which must then be on, does not require a null check for reference parameters, since they can never be null. |
| `allow_direct_ptr_in_if_statement` | A plain pointer in an `if` condition, e.g. `if (pointer)` or `if (first && second)`, also checks it for null, so that its first use in the then branch may make it a reference, see [pointers checked for null](#pointers-checked-for-null). May not be `true` together with `only_allow_reference_pointers_as_function_parameters`. |
| `only_allow_reference_pointers_as_function_parameters` | The tag may only be used on function parameters, also of function pointer types, not on [variables and struct fields](#reference-variables-and-fields) or on [functions and function pointer typedefs returning a reference](#functions-returning-a-reference). May not be `true` together with `allow_direct_ptr_in_if_statement`. |

```c
void set_value(REF int* value, int new_value);

void example(REF int* reference, int* pointer)
{
  int local = 1;
  int numbers[3] = {1, 2, 3};

  set_value(&local, 2);         // OK: the address of a variable
  set_value(&numbers[1], 2);    // OK: the address of an array element
  set_value(reference, 2);      // OK: a reference passed on

  set_value(pointer, 2);        // Reported: a normal pointer may be null
  set_value(NULL, 2);           // Reported

  reference = pointer;          // Reported: a reference may not be reassigned
  *reference = 5;               // OK: writing through a reference is fine
}
```

The argument for a reference parameter must be the address of an object: a variable, a field of a variable (`&point.x`), an array element (`&numbers[1]`) or a field reached through another reference (`&reference->field`). An array or a string literal is accepted too, since it decays to a pointer to its first element, and so is another reference passed on: a reference parameter, variable or field. A pointer [right after a null check](#pointers-checked-for-null) is accepted as well. A pointer variable, `NULL`, a pointer returned from a function, or anything reached through a normal pointer is reported. A reference may not be reassigned (`=`, `+=`, `++`, ...), and the tag may only be used on pointers.

A function pointer can be a reference too, so the function can call it without a null check. Its argument must then be a function, or another reference:

```c
typedef int (*math_func_t)(int a, int b);

int apply(REF math_func_t func, int a, int b)
{
  return func(a, b);            // OK: a reference is never null
}

apply(add, 1, 2);               // OK: a function is never null
apply(&add, 1, 2);              // OK
apply(NULL, 1, 2);              // Reported, and so are 0, (void*)0 and (math_func_t)0
apply(maybe_null, 1, 2);        // Reported: a normal function pointer may be null
```

The reference tag may be written on parameters, of function declarations and definitions and of function pointer types, on pointer variables and pointer struct fields, and on functions and function pointer typedefs returning a pointer, but not on other typedefs or anything else.

### Functions returning a reference

A function that returns a pointer can return a reference, by writing the tag in front of the function. Its result is then a reference for the caller: it can initialize a reference variable or be passed to a reference parameter. With `only_allow_reference_pointers_as_function_parameters: true`, the tag may not be used on functions.
- Every declaration and the definition must carry the tag, or none of them.
- The function may only return what a reference accepts: the address of an object, another reference (a parameter, variable or field, or the result of another function returning a reference), or a pointer [right after a null check](#pointers-checked-for-null).
- It may not return the address of a local variable or parameter, since it no longer exists once the function has returned. Global variables and `static` locals are fine.

```c
static int g_total = 0;

REF int* total(void)
{
  return &g_total;                          // OK: a global variable
}

REF int* find(Data* pointer)
{
  int local = 0;

  if (pointer != NULL)
    return &pointer->count;                 // Reported: a field reached through a normal pointer

  return &local;                            // Reported: local no longer exists after the return
}

void use(void)
{
  set_value(total(), 1);                    // OK: the result is a reference
}
```

A function pointer type can return a reference too, when the tag is written on its typedef, e.g. for a getter in a vtable. The tag has to be on a typedef: on a variable, field or parameter of a function pointer type, it keeps meaning that the function pointer itself is never null.
- Only a function that returns a reference, or another function pointer of such a type, may be assigned or passed to it, also in initializer lists. A function returning a normal pointer would break the guarantee. The other way around is fine: a function returning a reference may be used as a normal function pointer, which only drops the guarantee.
- A call through it gives a reference, also through a struct field of the type and through a typedef of the typedef.
- With `only_allow_reference_pointers_as_function_parameters: true`, the tag may not be used on typedefs.

```c
typedef REF Data* (*getter_t)(const Store* store);   // getter_t returns a reference
typedef Data* (*plain_getter_t)(const Store* store);

getter_t get = store_first;                 // OK: store_first returns a reference
getter_t bad = store_find;                  // Reported: store_find returns a normal pointer
plain_getter_t loose = store_first;         // OK: the guarantee is only dropped

edit_data(get(store), 1, 2, 3);             // OK: the result is a reference
edit_data(loose(store), 1, 2, 3);           // Reported: a normal function pointer's result may be null

REF getter_t never_null = get;              // the tag on a variable: the function pointer is never null
```

### Reference variables and fields

A pointer variable or struct field with the tag is a reference too: it always points to a real object, so it can be passed to a reference parameter, and it may never be reassigned. With `only_allow_reference_pointers_as_function_parameters: true`, the tag may not be used on them at all.
- A reference variable must be initialized when it is declared, with the same values a reference parameter accepts, or with a pointer [right after a null check](#pointers-checked-for-null).
- A struct with reference fields must give every one of them such a value in its initializer list, when it is declared or written as a compound literal. Leaving a reference field out, giving it `NULL` or `0`, or declaring the struct without an initializer list is reported, also for structs nested in other structs and for the elements of arrays. A copy of another struct value is fine, since its references are set already.
- A reference field may not be assigned later, since it only gets its value in the initializer list.

```c
typedef struct Holder
{
  REF Data* data;
  int count;
} Holder;

void example(Data* pointer)
{
  Data local = make_data();

  REF Data* reference = &local;             // OK
  REF Data* nullable = pointer;             // Reported: a normal pointer may be null
  Holder holder = { &local, 1 };            // OK
  Holder missing = { .count = 1 };          // Reported: data is left out
  Holder copy = holder;                     // OK: a copy of a struct whose references are set

  edit_data(reference, 1, 2, 3);            // OK: a reference variable as an argument
  edit_data(holder.data, 1, 2, 3);          // OK: a reference field too
  reference = &other;                       // Reported: a reference may not be reassigned
  holder.data = &other;                     // Reported
}
```

### Pointers checked for null

A normal pointer may become a reference right after an `if` statement has checked that it is not null: in the then branch, its first use may be the value of a reference variable or field, or the argument of a reference parameter. The check is written `pointer != NULL`, `pointer != 0`, `pointer != (void*)0` or `pointer != (Type*)0`, with the null on either side, and may be one of several conditions joined by `&&`. With `allow_direct_ptr_in_if_statement: true`, a plain pointer counts as a check too, e.g. `if (pointer)` or `if (first && second)`, but `if (!pointer)` does not. With `||` anywhere in the condition, the pointer may still be null, so it does not count. Struct fields can be checked the same way, e.g. `item->data != NULL`.

```c
void example(Data* first, Data* second)
{
  if (first != NULL && second != NULL)
  {
    REF Data* checked = first;              // OK: the first use after the check
    edit_data(second, 1, 2, 3);             // OK
    edit_data(first, 1, 2, 3);              // Reported: no longer the first use
  }

  if (first != NULL || second != NULL)
    edit_data(first, 1, 2, 3);              // Reported: with '||', first may be null
}
```

Only the first use counts, since later code may change the pointer or what it points to. Only the then branch counts: code after `if (pointer == NULL) return;` is not recognized.

Every declaration of the same function must use the tag on the same parameters. This is checked across headers and source files, and between declarations and definitions in one source file. When a function has several reference parameters, each position is checked on its own.

A function pointer type follows the same rules as for the [movement tags](#tags-on-function-pointers): a function assigned or passed to a function pointer must have the reference tag on the same parameters as the function pointer type, and the arguments of a call through a function pointer must be valid for its reference parameters.

```c
typedef void (*reader_t)(REF const Data* data);

void read(REF const Data* data);
void read_nullable(const Data* data);

reader_t reader = read;                     // OK
reader_t nullable = read_nullable;          // Reported: read_nullable does not take a reference
reader(NULL);                               // Reported: a reference can never be null
```

The tag works side by side with the [argument pointer movement rule](#argument-pointer-movement-rule), in any order, e.g. `void scale(MUTABLE REF float* value, float factor);` called as `scale(MUT(&value), 2.0f);`, or `void scale(borrows massive float* value, float factor);` called as `scale(lend(&value), 2.0f);` with the lowercase headers. The reference tag is not a movement tag, so a non-const reference still needs one of the movement tags when that rule is on.

A reference only borrows the object it points to, so while the argument pointer movement rule is on, it can not take or give away ownership. Both are reported at the level of this rule:
- A parameter may not be both a reference and tagged move, on functions and on function pointer types alike.
- A reference parameter may not be given to a move parameter. Pass on the object it points to by modify or as a const pointer instead.

```c
void consume(receives Data* data);
void take(receives massive Data* data);     // Reported: a reference can not take ownership

void process(borrows massive Data* data)
{
  edit_data(lend(data), 1, 2, 3);           // OK: a reference may be passed on to modify
  consume(give(data));                      // Reported: a reference does not own the object
}
```

## Function discard rule

This rule requires the result of every call to a non-void function to be used. Ignored results are a common source of bugs, since many functions report failure through their return value. An explicit cast to `void` documents that discarding the result is intentional, and is allowed.

**Config key:** `function_discard` · **Codes:** `CCW2101`

```yaml
rules:
  function_discard:
    level: Warning
```

This rule has no options besides `level`.

```c
int get_value(void);
typedef int (*value_function)(void);

void example(value_function function)
{
  get_value();                  // Reported
  (void)get_value();            // OK: an intentional discard
  int value = get_value();      // OK: the result is used
  if (get_value())              // OK: used as a condition
    value = 1;

  function();                   // Reported: calls through function pointers are checked too
  (get_value(), get_value());   // Reported twice: both results are discarded
  size_t size = sizeof(get_value());  // OK: never evaluated
}
```

Calls through function pointers are checked like direct calls, whether the pointer is a variable, a struct field, a vtable entry or an array element. Functions that return a struct by value are not checked by this rule. Discarding a returned [raii struct](#raii-structs) is reported by the struct resource management rule instead, even with a `(void)` cast, since such a value must always be destroyed.

## Array struct rule

This rule requires arrays to be declared only as fields inside structs. It helps prevent passing the wrong array size through C's array-to-pointer decay: once an array becomes a pointer, a receiving function can not know its length unless the caller also supplies it correctly. A struct keeps the array and its size together, even when it is passed around.

**Config key:** `array_struct` · **Codes:** `CCW2201`–`CCW2204`

```yaml
rules:
  array_struct:
    level: Warning
    only_allow_array_passing_to_library_functions: true
    enforce_size_suffix_for_array_structs: true
    size_suffix_with_underscore: true
    flexible_size_name: flexible
    enforce_suffix_for_array_structs: true
    array_struct_suffix: _array
    struct_name_as_prefix: false
    use_naming_rules_on_one_array_field_structs_without_forbidding_public_arrays: false
```

| Option | Description |
|--------|-------------|
| `only_allow_array_passing_to_library_functions` | Array fields may only be passed directly to standard library and third-party functions. |
| `enforce_suffix_for_array_structs` | The name of an array struct ends with `array_struct_suffix`. |
| `array_struct_suffix` | The array struct suffix, e.g. `_array`. May not be empty. |
| `enforce_size_suffix_for_array_structs` | The name of an array struct ends with the element count of every dimension. |
| `size_suffix_with_underscore` | Separate the counts with underscores, e.g. `_5_10` rather than `510`. |
| `flexible_size_name` | Used instead of a count for a flexible array member, e.g. `flexible`. May not be empty. |
| `struct_name_as_prefix` | The name of an array of project structs starts with the element struct's name. |
| `use_naming_rules_on_one_array_field_structs_without_forbidding_public_arrays` | Allow arrays anywhere, and only apply the naming options. See [naming rules without forbidding arrays](#naming-rules-without-forbidding-arrays). |

With `only_allow_array_passing_to_library_functions: false`, the rule only checks that arrays are struct fields. With it set to `true`, an array field may only be passed directly to a function declared by the standard library or in a `third_party_includes` path. Calls to project functions are reported, so project code must expose a safer wrapper or another deliberate interface instead.

```c
struct image
{
  unsigned char pixels[1024];
};
typedef struct image image;

void project_process(unsigned char* pixels, size_t count);

void use_image(image* image)
{
  project_process(image->pixels, 1024);                         // Reported with only_allow_array_passing_to_library_functions
  memcpy(image->pixels, image->pixels, sizeof(image->pixels));  // OK: standard library
}

int loose_buffer[4];                                            // Reported: not a struct field
```

Standard library functions are recognized from system headers. Third-party functions are recognized when their declarations come from a path listed in `third_party_includes`.

### Naming array structs

The naming options only apply to a struct whose single field is an array.

- `enforce_suffix_for_array_structs`: the name must end with `array_struct_suffix`.
- `enforce_size_suffix_for_array_structs`: the name must end with the element count of every dimension, e.g. `_5_10_15` for `int values[5][10][15]`. A flexible array, with no count, uses `flexible_size_name` instead, e.g. `_flexible` for `unsigned char bytes[]`. With `size_suffix_with_underscore: false`, the counts get no underscores, e.g. `51015`.
- When both are on, the array struct suffix comes first, e.g. `_array_5`.
- `struct_name_as_prefix`: when the elements are structs of the project, the name must start with the element struct's name, e.g. `point` for `point_t points[7]`. An anonymous element struct is named by its typedef. Elements that are structs from the standard library or a third party, or that are not structs at all, need no prefix.

An anonymous array struct is checked by its typedef name. With every option on:

```c
struct int_array_5 { int values[5]; };                  // OK
struct float_array_5_10 { float values[5][10]; };       // OK
struct bytes_array_flexible { unsigned char bytes[]; }; // OK
struct point_array_7 { point_t points[7]; };            // OK
struct time_array_2 { struct tm times[2]; };            // OK: struct tm is from the standard library
struct polygon { point_t points[4]; size_t count; };    // OK: more than one field

struct int_array_6 { int values[5]; };                  // Reported: must end with '_array_5'
struct int_5_array { int values[5]; };                  // Reported: the suffix comes before the count
struct path_array_7 { point_t points[7]; };             // Reported: must start with 'point'
```

### Naming rules without forbidding arrays

With `use_naming_rules_on_one_array_field_structs_without_forbidding_public_arrays: true`, arrays may be declared anywhere, e.g. as globals or local variables, and only the naming options above apply. This fits a project that wants consistent names for its array structs without moving every array into a struct. `only_allow_array_passing_to_library_functions` still applies to array fields. At least one of the naming options must be on, or the config is reported as invalid, since the rule would then check nothing.

```c
int loose_buffer[4];                                    // OK with the option
struct int_values { int values[5]; };                   // Reported: must end with '_array_5'
```

## Span struct rule

This rule standardizes span types, which pair a pointer with an element count. Any struct whose name ends with `span_struct_suffix` or `const_span_struct_suffix` must contain exactly two fields: a pointer named `data` and a `size_t` named `size`. Optional settings then make spans the standard way to pass arrays and buffers around, so that a pointer never travels without its size.

**Config key:** `span_struct` · **Codes:** `CCW2301`–`CCW2309`

```yaml
rules:
  span_struct:
    level: Warning
    only_allow_array_passing_to_library_functions_and_spans: true
    allow_string_literals_as_const_char_arguments: true
    span_struct_suffix: _span
    const_span_struct_suffix: _const_span
    allow_spans_to_be_given_fewer_elements_than_their_size: false
    require_span_immediately_after_array: false
    only_allow_span_data_passing_in_one_line_static_functions: false
    allow_pod_span_to_be_initialized_manually_if_static: false
```

| Option | Description |
|--------|-------------|
| `span_struct_suffix` | Structs whose names end with this suffix are spans, e.g. `_span`. May not be empty. |
| `const_span_struct_suffix` | Structs whose names end with this suffix are const spans, e.g. `_const_span`. It is checked first, so it may end with `span_struct_suffix`. May not be empty. |
| `only_allow_array_passing_to_library_functions_and_spans` | Arrays, and the data of spans, may only be passed to standard library and third-party functions, span pod creators and struct initializers. See [passing arrays and span data](#passing-arrays-and-span-data). |
| `allow_string_literals_as_const_char_arguments` | With `only_allow_array_passing_to_library_functions_and_spans`, a string literal may also be passed to a pointer to const characters, e.g. a `const char*` parameter. See [string literals](#string-literals). |
| `allow_spans_to_be_given_fewer_elements_than_their_size` | A span may cover only the start of an array. |
| `require_span_immediately_after_array` | Every array outside of a struct is followed right away by a span holding it, which is then the only way to reach the array. |
| `only_allow_span_data_passing_in_one_line_static_functions` | The data of a span may only be passed on inside one-statement static wrapper functions. |
| `allow_pod_span_to_be_initialized_manually_if_static` | A span with static storage duration may be initialized with braces. Needs the [struct resource management rule](#struct-resource-management-rule-raii) to be on. |

### Spans and const spans

A const span is a read-only view: it works exactly like a span, but holds a pointer to const data and is named with `const_span_struct_suffix`. A span may change the data it points to, so its pointer may not point to const data, and a const span may not, so its pointer must. A const span may still view data that can change, e.g. a normal array. The const suffix is checked first, so it may end with the span suffix, as `_const_span` ends with `_span`.

```c
struct int_span { int* data; size_t size; };                    // OK
struct int_const_span { const int* data; size_t size; };        // OK
struct const_data_span { const int* data; size_t size; };       // Reported: const data needs a const span
struct mutable_data_const_span { int* data; size_t size; };     // Reported: a const span needs const data

const int constants[2] = { 4, 5 };
int_const_span_t view = { constants, 2 };                       // OK
```

Only the data decides the kind of span: a const pointer to non-const data, `int* const data`, is a normal span.

### Initializing spans

A span variable must be initialized at declaration. When an array is used in its initializer, the `size` value must be a constant expression equal to the array's element count. When struct resource management is enabled, the same count is required when an array is passed to the span's pod creator: a function named `<span struct name><pod struct creator suffix>` that returns the span, e.g. `int_span_t int_span_pod(int* data, size_t size)`. A function that only has a name like it, e.g. one ending with the pod suffix for a span that does not exist, or one that returns something else, is a normal project function.

With `allow_spans_to_be_given_fewer_elements_than_their_size: true`, the count may also be smaller than the array's element count, so a span can cover only the start of an array, e.g. `{ values, 2 }` for a 3 element array. It may never be larger, and it must still be a constant so it can be checked.

A span can also start inside an array, at `&array[index]` or `array + index`. Its count is then checked against the elements left from that index, so `{ &values[3], 7 }` is right for a 10 element array. The index must be a constant within the array, and an index past its end is reported.

```c
int values[10];
int_span_t tail = { &values[3], 7 };           // OK: the 7 elements left from index 3
int_span_t rest = { values + 3, 7 };           // OK: the same
int_span_t wrong = { &values[3], 10 };         // Reported: only 7 elements are left
int_span_t outside = { &values[12], 1 };       // Reported: index 12 is outside of the array
```

When [struct resource management](#struct-resource-management-rule-raii) is enabled and a span has a pod creator, the span is a pod struct, and a pod struct may only be initialized from a function, another struct variable or a field. A span with static storage duration can not call its pod creator, though: a global or `static` initializer must be a constant expression. With `allow_pod_span_to_be_initialized_manually_if_static: true`, a span or const span with static storage duration may therefore be initialized with braces. That covers globals, `static` globals and `static` locals. The braces are still checked like any span initializer, e.g. the count of the array. Other spans, other pod structs, and brace-built spans passed as arguments or returned are still reported.

```c
int g_values[4];
static int_span_t s_span = { g_values, 4 };      // OK with the option: static storage duration
static int_span_t s_long = { g_values, 5 };      // Reported: the count is still checked
static pos_t s_origin = { 0, 0 };                // Reported: not a span

void example(void)
{
  static int_span_t local = { g_values, 4 };     // OK with the option: a static local
  int_span_t span = { g_values, 4 };             // Reported: not static, use int_span_pod
}
```

Here is a complete example of a span with a pod creator:

```c
struct integer_span
{
  int* data;
  size_t size;
};
typedef struct integer_span integer_span_t;

integer_span_t integer_span_pod(int* data, size_t size)
{
  return (integer_span_t){ data, size };
}

void foo(void)
{
  int array[110] = { 0 };
  integer_span_t array_span = integer_span_pod(array, 110);    // OK: the count matches the array
  use_integers(&array_span);
}
```

### Requiring a span after every array

With `require_span_immediately_after_array: true`, every array outside of a struct, local, static or global, must be followed right away by a span or const span variable holding the whole array: for a local array in the next statement, for a global in the next declaration of the file. The span must start at the beginning of the array, `values` or `&values[0]`, and hold its full size, with an initializer or the span's pod creator.

After that, the span is the one way to reach the array. The array may not be named again, except:
- in the span's initializer,
- in code that is never evaluated: `sizeof`, `_Alignof` and `__typeof__`,
- as the first argument of the array destroy function of an [array of raii structs](#arrays-of-raii-structs), e.g. `int_vector_destroy_array(vectors, 2)`, when that function has the exact signature `void <struct name><suffix>(<struct name>* self, size_t n)` and the struct is a raii struct with its raii creator declared before the call, since the struct resource management rule tracks the array itself.

Elements are reached through the span, e.g. `values_span.data[1]`, or through the span's functions, and a matrix through its span of rows, e.g. `matrix_span.data[1][2]`. An array without its span only gets the missing span reported, not every use of it.

```c
int values[3] = { 1, 2, 3 };
int_span_t values_span = { values, 3 };          // OK

int a[2], b[3];
int_span_t a_span = { a, 2 }, b_span = { b, 3 }; // OK: one declaration for both

int lonely[3];                                   // Reported: no span follows
int count = 0;

values_span.data[1] = 4;                         // OK: through the span
size_t bytes = sizeof(values);                   // OK: never evaluated
values[1] = 4;                                   // Reported: use values_span instead
int* pointer = values;                           // Reported: same
```

### Passing arrays and span data

With `only_allow_array_passing_to_library_functions_and_spans: false`, only the span shape and the initialization rules above apply. With it set to `true`, arrays may only be passed directly to standard library functions, third-party functions, span pod creators, or struct initializers. This prevents array-to-pointer decay from hiding the array length and helps avoid passing an incorrect size to a function.

The data of a span or const span is the array it views, so it is restricted too: `span.data`, `span->data` and pointer arithmetic on it such as `span.data + 1` may only be passed to standard library and third-party functions. A project function takes the span itself instead of its data and size separately, and not even a pod creator may get the data, since the size given with it could not be checked. The same goes for the pointer returned by a function named after the span, e.g. a getter `int* int_span_data(const int_span_t* self)`, when it is passed on directly. A pointer to a single element, e.g. `&span.data[i]`, reading or writing elements, and a pointer to the span itself are not restricted.

```c
void process_ints(int* data, size_t size);
void process_span(int_span_t span);

process_span(span);                                     // OK
memcpy(span.data, other, 3 * sizeof(int));              // OK: standard library
edit_value(&span.data[0]);                              // OK: a single element
process_ints(span.data, span.size);                     // Reported: pass the span instead
process_ints(int_span_data(&span), 3);                  // Reported: the getter returns the span's data
int_span_t rest = int_span_pod(span.data + 1, 2);       // Reported: the size can not be checked
```

Making a smaller span from a span belongs in the span's own functions, where the data can be combined with a checked size, e.g. with an initializer. Together with the [private alternative rule](#private-alternative-rule) and private `data` and `size` fields, only those functions can reach them at all.

#### String literals

A string literal is an array too, so with `only_allow_array_passing_to_library_functions_and_spans: true` it may only be passed to the same functions as other arrays. A literal is a special case, though: it always ends with its terminator, and it may never be written to. A function that can only read it, through a pointer to const characters such as `const char*`, can therefore not run past its end. With `allow_string_literals_as_const_char_arguments: true`, a string literal may be passed to such a parameter, also through a function pointer.

```c
void log_message(const char* message);
void overwrite_message(char* message);
void log_count(int count, ...);

log_message("hello");             // OK with the option: read-only parameter
overwrite_message("hello");       // Reported: the literal could be written to
log_count(1, "hello");            // Reported: a variadic argument has no parameter type to check

char copy[] = "hello";
log_message(copy);                // Reported: an array, not a literal, so its size could be lost
```

Without the option, a literal is handled like any other array, e.g. by copying it into an array with a span: `char greeting[] = "hello";` followed by a `char_const_span` holding it.

### One-statement wrappers

With `only_allow_span_data_passing_in_one_line_static_functions: true`, the data of a span, including a pointer returned by a function named after it, may only be passed on inside a static function whose body is a single statement: one expression or one `return`, and not a comma expression. Every place that hands raw data to an unsafe function, e.g. from the standard library, is then a small wrapper that takes spans and is easy to review on its own, and all other code passes the spans themselves. Combined with `only_allow_array_passing_to_library_functions_and_spans`, a wrapper may still only hand the data to standard library and third-party functions.

```c
static void int_span_copy(int_span_t destination, int_span_t source)
{
  memcpy(destination.data, source.data, source.size * sizeof(int));   // OK: a one-statement static wrapper
}

void example(int_span_t a, int_span_t b)
{
  int_span_copy(a, b);                                                // OK: spans all the way
  memcpy(a.data, b.data, 3 * sizeof(int));                            // Reported: raw data outside a wrapper
}
```

## Const field rule

This rule forbids struct and union fields that are const themselves. A const field makes the whole struct impossible to assign, or to set up in place after its declaration: for example, a make function can not fill in such a struct through a pointer. A pointer field may still point to const data, since the field itself can still be assigned. To make data read-only, make the whole object const instead.

**Config key:** `const_field` · **Codes:** `CCW2401`

```yaml
rules:
  const_field:
    level: Warning
```

This rule has no options besides `level`.

```c
struct values
{
  const int* pointer_to_const;      // OK: only the data it points to is const
  const char* name;                 // OK
  int* const* to_const_pointer;     // OK: the field itself is not const

  const int constant;               // Reported
  int* const const_pointer;         // Reported: only the data it points to may be const
  const int* const both;            // Reported
  const int constants[3];           // Reported: an array of const elements
  const_int_t typedef_constant;     // Reported: const through a typedef
};
```

## No goto rule

This rule forbids `goto` statements. That covers `goto label;` and the computed `goto *address;` of GNU C, also when hidden in a macro. Loops, `break`, `continue` and `return` cover the same needs in a structured way. Labels on their own are not reported, only the jumps to them.

**Config key:** `no_goto` · **Codes:** `CCW2501`

```yaml
rules:
  no_goto:
    level: Error
```

This rule has no options besides `level`.

```c
int scale(int value)
{
  if (value < 0)
    goto done;        // Reported

  value = value * 2;

done:                 // OK: a label is not a jump
  return value;
}
```

## Typedef number rule

This rule requires the built-in number types (`short`, `int`, `long`, `long long` and their `unsigned` versions) to be used through a typedef. That can be a standard one such as `int32_t` and `uint64_t` from `<stdint.h>`, or one of the project's own such as `i32`. The size of `int` and `long` differs between platforms, while a typedef states the size the code relies on, and it can be changed in one place.

**Config key:** `typedef_number` · **Codes:** `CCW2601`–`CCW2603`

```yaml
rules:
  typedef_number:
    level: Warning
    forbid_number_literals_in_plain_char: true
    require_typedef_for_signed_and_unsigned_char: true
    require_typedef_for_float_and_double: true
    require_static_assert_for_number_typedefs: true
```

| Option | Description |
|--------|-------------|
| `forbid_number_literals_in_plain_char` | A number literal may not become the value of a plain `char`, in an initializer or a plain assignment, also through a cast or a minus sign: `char c = 10;`, `c = 65;` and `text[i] = 0;` are reported. A character literal (`'a'`, `'\n'`, `'\0'`) is fine, and so is a value that is already a `char`, `++`, `--`, `+=`, arithmetic on characters such as `c - '0'` or `'a' + offset`, and the zero initializer `{ 0 }`. |
| `require_typedef_for_signed_and_unsigned_char` | `signed char` and `unsigned char` must be used through a typedef too, e.g. `int8_t` and `uint8_t`. |
| `require_typedef_for_float_and_double` | `float`, `double` and `long double` must be used through a typedef too, e.g. a project's own `f32` and `f64`. |
| `require_static_assert_for_number_typedefs` | A typedef that names a built-in number type the rule covers must be followed directly by a static assert on its size, except a character type, whose size is always 1, see [static asserts on number typedefs](#static-asserts-on-number-typedefs). Only checked for C11 and newer. |

```c
typedef int i32;                  // OK: a typedef that names a number type may be written with one
typedef i32 count_t;              // OK: so may a typedef of a typedef
typedef int (*callback_t)(void);  // Reported: only uses int, write i32 (*callback_t)(void)
typedef unsigned* counts_ptr;     // Reported

int counter = 0;                  // Reported: use e.g. int32_t or i32
i32 typed = 0;                    // OK
uint64_t total = 0;               // OK
unsigned long length(void);       // Reported: the return type
void set(short value);            // Reported: the parameter
size_t size = sizeof(int);        // OK: expressions such as sizeof and casts are not checked

struct header
{
  unsigned flags : 3;             // Reported
  uint32_t checked : 1;           // OK
};

char name[16];                    // OK: plain char is text, not a number
_Bool visible;                    // OK
double ratio;                     // Reported with require_typedef_for_float_and_double
unsigned char byte;               // Reported with require_typedef_for_signed_and_unsigned_char
```

Only declarations in the project's own code are checked: variables, parameters, struct and union fields, function return types and typedefs, also the return and parameter types of function pointers. The one place a built-in number type may be written is a typedef that names it, optionally `const`, such as `typedef int i32;` or `typedef const double cf64;`. A typedef that only uses one, of a function pointer, pointer or array, is checked like any other declaration. Expressions, literals and integer promotions are not, so `(int)x`, `sizeof(long)` and `x + 1` are fine. Plain `char` is never reported, since it holds text, and neither is `_Bool`. System headers and `third_party_includes` folders are skipped, as always.

`main` is an exception, since the C standard fixes its signature: `int main(void)` and `int main(int argc, char** argv)` are fine. The variables in its body are still checked.

Other signatures dictated by a library are not exceptions, e.g. a `qsort` comparator returning `int` or a signal handler taking an `int`. Wrap them in a [suppression](#suppressing-rules) with a reason:

```c
// WorkshopC off
// Reason: qsort requires a comparator that returns int
int compare_counts(const void* left, const void* right)
{
  const i32 a = *(const i32*)left;
  const i32 b = *(const i32*)right;
  return (a > b) - (a < b);
}
// WorkshopC on
```

> [!TIP]
> To print a typedef'd number portably, use the format macros of `<inttypes.h>`, e.g. `printf("%" PRId32 "\n", value);` for an `int32_t`, or the project's own print wrappers, instead of `%d` and `%ld`, whose sizes are those of `int` and `long`.

### Number literals in plain char

With `forbid_number_literals_in_plain_char`, a plain `char` only gets characters written as characters. `char c = 10;` hides that the value is a newline, while `char c = '\n';` says so. A byte value belongs in `uint8_t` or `int8_t` instead.

```c
char letter = 'a';                // OK
char newline = 10;                // Reported: write '\n'
char negative = -1;               // Reported
char cast = (char)65;             // Reported: write 'A'
char copy = text[0];              // OK: already a char

letter = 66;                      // Reported
letter++;                         // OK: e.g. iterating over the letters
letter += 2;                      // OK
text[i] = 0;                      // Reported: write '\0'
i32 digit = letter - '0';         // OK: arithmetic on characters

char word[3] = { 'h', 105, '\0' }; // Reported: 105

char buffer[64] = { 0 };          // OK: { 0 } zero initializes any object
key zeroed = { 0 };               // OK, also when the first field is a plain char
char pair[2] = { 0, 0 };          // Reported: only a single 0 is the zero initializer
```

### Static asserts on number typedefs

With `require_static_assert_for_number_typedefs`, a typedef that names a built-in number type must be followed directly by a static assert whose condition uses `sizeof(<typedef>)`. Then the build fails on a platform where the type does not have the size the name promises. Any condition is accepted, and so are `_Static_assert`, C23's `static_assert` and the `static_assert` macro of `<assert.h>`.

```c
typedef int i32;
_Static_assert(sizeof(i32) == 4, "i32 must be 4 bytes");    // OK

typedef unsigned short u16;
_Static_assert(sizeof(u16) * 8 == 16, "u16 must be 16 bits"); // OK: any condition with sizeof(u16)

typedef unsigned char u8;                                  // OK: needs none, see below

typedef short i16;                                         // Reported: the assert does not use sizeof(i16)
_Static_assert(sizeof(short) == 2, "short must be 2 bytes");

typedef double f64;                                        // Reported: no static assert after it
typedef i32 count_t;                                       // OK: a typedef of a typedef needs none
typedef int32_t i32_alias;                                 // OK
```

Only typedefs of types the rule covers need one. A `float` or `double` typedef needs one only with `require_typedef_for_float_and_double` on. A typedef of `signed char`, `unsigned char` or plain `char` never does: `sizeof` of a character type is 1 by definition, so such an assert could never fail. To rely on 8-bit bytes, use `uint8_t` and `int8_t`, which only exist where a byte has 8 bits. Static asserts came with C11, so files compiled as C99 or older are not checked.

## Suppressing rules

Sometimes a rule has to give way, e.g. for code dictated by a legacy protocol or a hardware register layout. Every rule can be turned off for a section of a file with a comment saying `// WorkshopC off`, and back on with `// WorkshopC on`. Nothing is reported for the lines in between.

```c
// WorkshopC off
// Reason: The enum values are dictated by a legacy protocol
enum Color {            // Not reported
  RED,
  GREEN,
  BLUE
};
// WorkshopC on
```

The markers are case sensitive and are recognized anywhere on a line, so write them as comments on lines of their own.

A suppression only applies to the file it is written in, so a header that turns WorkshopC off does not affect the files that include it. Every `// WorkshopC off` must be turned back on with `// WorkshopC on` in the same file, in headers and source files alike, since a forgotten `on` would silently disable every rule for the rest of the file. The following are always checked and reported as errors, whatever the config says:

| Code | Reported when |
|------|---------------|
| `CCW0001` | A `WorkshopC off` is never turned back on in the same file |
| `CCW0003` | A `WorkshopC on` has no preceding `WorkshopC off` |
| `CCW0004` | A `WorkshopC off` comes while already turned off: suppressions can not be nested, and the first `on` ends the suppression |

Every run counts the suppressed sections and the lines of code they suppress, in the [summary](usage.md#reading-the-output), and [`--list-suppressions`](usage.md#listing-suppressions) lists each section with its reason, so that the exceptions stay visible.

> [!TIP]
> Code that WorkshopC should not see at all, e.g. a compiler extension clang does not understand, can be hidden from it instead. See [Adjusting code for analysis](#adjusting-code-for-analysis).

### Suppression reason rule

Every suppression should say why it exists. When this rule is on, each `// WorkshopC off` comment must be followed **on the very next line** by a comment that starts with `Reason: ` and gives a non-empty reason. A `/* Reason: ... */` block comment is also accepted. The reasons then show up in code review, and every exception in the codebase can be found and explained later.

**Config key:** `suppression_reason_rule` · **Codes:** `CCW0002`

```yaml
rules:
  suppression_reason_rule:
    level: Error
```

This rule has no options besides `level`.

```c
// WorkshopC off
// Reason: The enum values are dictated by a legacy protocol
enum Color { RED, GREEN, BLUE };    // OK: the suppression has a reason
// WorkshopC on

// WorkshopC off
enum Shape { CIRCLE, SQUARE };      // Reported on the 'WorkshopC off' line: no reason given
// WorkshopC on
```

Details:
- `Reason: ` is case sensitive, and a blank line between the two comments does not count.
- Only the `WorkshopC off` that opens a suppressed range needs a reason.
- The check covers the analyzed file and every project header it includes, but not system headers or `third_party_includes` folders.
- The rule reports on the `WorkshopC off` line itself, so it can not be silenced by the suppression it is checking.

## Adjusting code for analysis

WorkshopC analyzes code with the macro `WORKSHOPC_PARSING` defined, which makes it possible to adjust code for the analysis with `#ifdef` and `#ifndef`. The [tag headers](tag-headers.md) use it to turn the tags into annotations only while WorkshopC analyzes the code, so that a normal build is unaffected:

```c
#ifdef WORKSHOPC_PARSING
#define PRIVATE __attribute__((annotate("workshopc_private_field")))
#else
#define PRIVATE
#endif
```

It can also hide code that clang can not handle, e.g. an extension of another compiler:

```c
#ifndef WORKSHOPC_PARSING
#pragma some_vendor_specific_pragma
#endif
```

Code hidden this way is not checked at all, so to only silence a rule, prefer [suppressing it](#suppressing-rules) with a reason.

The other way around, a declaration can be shown to WorkshopC only, e.g. a raii struct's copy function that the real build deliberately does not have, see [deleting copy, move or return](#deleting-copy-move-or-return).

---

[Next: Tag headers →](tag-headers.md) · [Back to README](../README.md)
