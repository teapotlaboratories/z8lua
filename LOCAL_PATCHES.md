# z8lua — vendoring notes & local patches

Upstream: [`jtothebell/z8lua`](https://github.com/jtothebell/z8lua), branch
`z8lua-3ds-switch`, @ the commit in [`UPSTREAM_SHA.txt`](UPSTREAM_SHA.txt). This is
PICO-8's fixed-point Lua 5.2 dialect (`LUA_NUMBER = z8::fix32`).

Vendoring here is **least-destructive**: the source tree is **byte-identical to upstream
except one header patch**. Nothing is renamed, and nothing is deleted. All the
"integrate into ESP-IDF" work lives in the build files, not in upstream source.

## The only source change: `fix32.h`

On **xtensa-esp-elf** (ESP-IDF), `int32_t` is `long`, so plain `int` has no exact-match
`fix32` constructor and `int → fix32` is *ambiguous* — a hard compile error under C++.
(On x86 hosts `int32_t == int`, so it compiles there and the bug is hidden.)

Added an implicit `fix32(T)` for `T` = `int` / `unsigned int`, guarded by `enable_if` so it
only exists when they are *distinct* from `int32_t` / `uint32_t` (a no-op on x86). It mirrors
upstream's own `long` / `unsigned long` template right above it, and is more complete than
upstream's `#ifdef __3DS__` block (which covers `int` only). Worth upstreaming.

## Everything else is build-only (no source edits)

`fix32` is a C++ type, so the Lua core must compile **as C++**. We do this in the build,
**not** by renaming files:

- **ESP-IDF** ([`CMakeLists.txt`](CMakeLists.txt)) registers the core `.c` files and marks
  them `set_source_files_properties(… PROPERTIES LANGUAGE CXX)`, so CMake invokes the C++
  compiler on them. `LUA_USE_LONGJMP` keeps setjmp/longjmp error handling (C++ exceptions
  disabled). This mirrors upstream's own makefile, which sets `CC = g++ -std=c++17`.
- **Host** benchmark build compiles the same `.c` files with `g++ -x c++`.

**Excluded units** — `liolib` / `loslib` / `loadlib` (host OS / `dlopen` deps that don't
apply on the MCU, never opened by `linit`) and `ltests` (Lua's internal test harness) — are
**kept in the tree** but simply not registered as sources. No `#define` guard and no
deletion needed.

## Re-pulling upstream

A clean rebase onto a newer `z8lua-3ds-switch` commit: re-apply only the one `fix32.h`
hunk. The build files (`CMakeLists.txt`, this file, `UPSTREAM_SHA.txt`) are additions that
never touch upstream source, so they don't conflict.
