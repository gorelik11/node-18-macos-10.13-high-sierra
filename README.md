# Building Node.js 18 on macOS 10.13 High Sierra

Patches and instructions to compile **Node.js v18.20.4** from source on **macOS 10.13.6 (High Sierra)**.

## Motivation

[Claude Code](https://docs.anthropic.com/en/docs/claude-code) is Anthropic's CLI tool for working with Claude directly from the terminal. While the official installation method uses a standalone binary, it can also be installed via npm:

```bash
npm install -g @anthropic-ai/claude-code
```

This npm installation path requires **Node.js >= 18**. However, the official Node.js 18 prebuilt binaries do not run on macOS 10.13 High Sierra, and the standalone Claude Code binary doesn't support it either.

By compiling Node.js 18 from source with the patches in this repository, you can get a working Node.js 18 on High Sierra and install Claude Code via npm.

## Prerequisites

- **macOS 10.13.6** (High Sierra)
- **Xcode Command Line Tools** (provides clang/clang++)
- **Python 3** (detected by Node's configure script)

> **Note:** Node.js 18 bundles its own OpenSSL (3.0.x) in `deps/openssl/` and compiles it as part of the build. You do **not** need to install OpenSSL separately.

## Build Instructions

### 1. Download Node.js source

```bash
curl -O https://nodejs.org/dist/v18.20.4/node-v18.20.4.tar.gz
tar xzf node-v18.20.4.tar.gz
cd node-v18.20.4
```

### 2. Apply patches

```bash
git apply /path/to/patches/01-ada-charconv-polyfill.patch
git apply /path/to/patches/02-simdutf-disable-simd.patch
git apply /path/to/patches/03-v8-regexp-parser-cpp17-compat.patch
git apply /path/to/patches/04-v8-disable-system-instrumentation.patch
```

Or apply them all at once:

```bash
for p in /path/to/patches/*.patch; do git apply "$p"; done
```

> **Note:** If applying without git, you can use `patch -p0 < patchfile.patch` from inside the node source directory.

### 3. Configure and build

```bash
./configure --prefix=/usr/local --with-intl=full-icu
make -j$(sysctl -n hw.ncpu)
```

> **Important:** Use `--with-intl=full-icu` for full internationalization support. Claude Code requires this.

### 4. Install

```bash
sudo make install
```

### 5. Verify

```bash
node --version
# v18.20.4
```

## Installing Claude Code

Once Node.js 18 is installed:

```bash
npm install -g @anthropic-ai/claude-code
```

Then launch it:

```bash
claude
```

## What the Patches Fix

macOS 10.13 ships with an older clang (Apple LLVM 9.x) and an older libc++ that lack several C++17 features and newer system APIs. The four patches address each incompatibility:

### Patch 01 — `<charconv>` polyfill for the ADA URL parser

**Problem:** The `<charconv>` header (`std::from_chars`, `std::to_chars`) is not available in the libc++ shipped with macOS 10.13.

**Solution:** A polyfill header (`deps/ada/charconv_polyfill.h`) provides minimal implementations of `std::from_chars` and `std::to_chars` using `strtoll`/`snprintf`. Both `deps/ada/ada.cpp` and `deps/ada/ada.h` are patched to include the polyfill instead of `<charconv>`.

<details>
<summary>View diff</summary>

```diff
--- deps/ada/ada.cpp
+++ deps/ada/ada.cpp
@@ -10422,7 +10422,7 @@
 /* begin file src/helpers.cpp */

 #include <algorithm>
-#include <charconv>
+#include "charconv_polyfill.h"
 #include <cstring>
 #include <sstream>
```

```diff
--- deps/ada/ada.h
+++ deps/ada/ada.h
@@ -5143,7 +5143,7 @@

 #include <algorithm>
-#include <charconv>
+#include "charconv_polyfill.h"
 #include <iostream>
 #include <optional>
 #include <string>
```

Plus the new `deps/ada/charconv_polyfill.h` file (85 lines) with `std::from_chars` and `std::to_chars` implementations.

</details>

### Patch 02 — Disable SIMD backends in simdutf

**Problem:** The older clang on macOS 10.13 does not support AVX-512 intrinsics (`_ktestc_mask64_u8`, etc.) required by simdutf's optimized code paths.

**Solution:** Force simdutf to use the portable fallback implementation by disabling all architecture-specific backends.

<details>
<summary>View diff</summary>

```diff
--- deps/simdutf/simdutf.gyp
+++ deps/simdutf/simdutf.gyp
@@ -4,6 +4,14 @@
       'target_name': 'simdutf',
       'type': 'static_library',
       'include_dirs': ['.'],
+      'defines': [
+        'SIMDUTF_IMPLEMENTATION_ICELAKE=0',
+        'SIMDUTF_IMPLEMENTATION_HASWELL=0',
+        'SIMDUTF_IMPLEMENTATION_WESTMERE=0',
+        'SIMDUTF_IMPLEMENTATION_ARM64=0',
+        'SIMDUTF_IMPLEMENTATION_PPC64=0',
+        'SIMDUTF_IMPLEMENTATION_FALLBACK=1',
+      ],
       'direct_dependent_settings': {
         'include_dirs': ['.'],
       },
```

</details>

### Patch 03 — V8 regexp parser C++17 compatibility

**Problem:** Two C++17 constructs in V8's regexp parser fail with the older clang:
1. Aggregate initialization with brace syntax on a class with a private constructor
2. Brace-enclosed return of a template type

**Solution:** Change `private:` to `public:` for the constructor access, and replace brace initialization `{...}` with parenthesized construction `(...)`.

<details>
<summary>View diff</summary>

```diff
--- deps/v8/src/regexp/regexp-parser.cc
+++ deps/v8/src/regexp/regexp-parser.cc
@@ -184,7 +184,7 @@

 template <class CharT>
 class RegExpParserImpl final {
- private:
+ public:
   RegExpParserImpl(const CharT* input, int input_length, RegExpFlags flags,
                    uintptr_t stack_limit, Zone* zone,
                    const DisallowGarbageCollection& no_gc);
@@ -2355,8 +2355,8 @@
                                       RegExpFlags flags,
                                       RegExpCompileData* result,
                                       const DisallowGarbageCollection& no_gc) {
-  return RegExpParserImpl<CharT>{input,       input_length, flags,
-                                 stack_limit, zone,         no_gc}
+  return RegExpParserImpl<CharT>(input,       input_length, flags,
+                                 stack_limit, zone,         no_gc)
       .Parse(result);
 }
```

</details>

### Patch 04 — Disable V8 system instrumentation

**Problem:** V8 enables OS-level tracing on macOS which requires `<os/signpost.h>`, a header only available starting in macOS 10.14 (Mojave).

**Solution:** Disable the `v8_enable_system_instrumentation` flag and remove the `V8_ENABLE_SYSTEM_INSTRUMENTATION` define.

<details>
<summary>View diff</summary>

```diff
--- tools/v8_gypfiles/features.gypi
+++ tools/v8_gypfiles/features.gypi
@@ -64,7 +64,7 @@
       }],
       ['OS == "win" or OS == "mac"', {
         # Sets -DSYSTEM_INSTRUMENTATION. Enables OS-dependent event tracing
-        'v8_enable_system_instrumentation': 1,
+        'v8_enable_system_instrumentation': 0,
       }, {
         'v8_enable_system_instrumentation': 0,
       }],
@@ -465,7 +465,7 @@
         'defines': ['V8_ENABLE_SWISS_NAME_DICTIONARY',],
       }],
       ['v8_enable_system_instrumentation==1', {
-        'defines': ['V8_ENABLE_SYSTEM_INSTRUMENTATION',],
+        'defines': [],
       }],
       ['v8_enable_webassembly==1', {
         'defines': ['V8_ENABLE_WEBASSEMBLY',],
```

</details>

## Lessons Learned

- macOS 10.13's libc++ is missing several C++17 standard library features (`<charconv>`, etc.)
- macOS 10.13 lacks newer system APIs (`<os/signpost.h>` requires 10.14+)
- Apple LLVM 9.x (Xcode on 10.13) doesn't support AVX-512 intrinsics
- Modern Homebrew no longer supports macOS 10.13 — all dependencies must be built from source
- GYP build variables control both preprocessor defines and file inclusion

## License

MIT
