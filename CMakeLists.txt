# Vendored z8lua — PICO-8's fixed-point Lua 5.2 dialect. See UPSTREAM_SHA.txt and
# LOCAL_PATCHES.md.
#
# The Lua core ships as upstream .c, but LUA_NUMBER is z8::fix32 (a C++ type), so
# these units MUST compile as C++. We do that in the build only -- the sources keep
# their upstream .c names (no renames). Upstream's own makefile does the same by
# setting CC=g++.

# Compile the core. Skip the units that pull host OS / dlopen deps or are test-only
# and are never opened by linit -- they stay in the tree (least-destructive) but are
# simply not registered as sources.
file(GLOB Z8_SRCS "${CMAKE_CURRENT_LIST_DIR}/*.c")
list(FILTER Z8_SRCS EXCLUDE REGEX "/(liolib|loslib|loadlib|ltests)[.]c$")

idf_component_register(SRCS ${Z8_SRCS} INCLUDE_DIRS ".")

# Force C++ on the .c sources (LUA_NUMBER = z8::fix32). No file is renamed.
set_source_files_properties(${Z8_SRCS} PROPERTIES LANGUAGE CXX)

# setjmp/longjmp error handling so C++ exceptions can stay disabled (ESP-IDF default).
target_compile_definitions(${COMPONENT_LIB} PUBLIC LUA_USE_LONGJMP)

# Third-party code: silence its warnings so a -Werror project build still passes.
target_compile_options(${COMPONENT_LIB} PRIVATE -w)

# ESP-IDF compiles with -fno-jump-tables, which turns luaV_execute's ~40-way opcode
# switch into a linear compare-chain — every bytecode dispatch walks ~20 branches
# (measured: the dominant per-instruction cost on the S3). Re-enable jump tables +
# switch conversion for the interpreter so opcode dispatch is O(1). (Later flags win.)
target_compile_options(${COMPONENT_LIB} PRIVATE -fjump-tables -ftree-switch-conversion)
