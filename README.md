# Grimoire Core Tome

`core` is the bootstrap tome for Grimoire: the managed toolchain and userland that source
builds depend on instead of reaching for host tools. Add it with:

```sh
grm tome add https://github.com/hermetomancy/tome-core --ref main
```

(`grm setup --bootstrap` adds it automatically.)

## Packages

The managed toolchain, in bootstrap order:

- `gmake` — GNU make (bins `gmake` + `make`); the seed package, built first with the host
  compiler boundary.
- `cmake`, `python3-minimal` — build drivers for the LLVM build (`python3-minimal` is a
  static, stdlib-only interpreter that runs LLVM's and rust's build scripts; the full `python3`
  ships in tome-world).
- `llvm` — LLVM + lld, one monorepo build. `clang` is a **split member** carved from the
  same build by its `files` globs, with the compiler-rt runtimes installed inside clang's
  resource directory (`LLVM_ENABLE_RUNTIMES`).
- `toolchain-wrappers` — the managed compiler boundary: `cc`/`c++`/`ar`/`ld`/… wrapper
  scripts baking absolute store paths to clang and llvm (its runtime deps).
- `dash`, `mawk`, `uutils`, `gsed`, `ggrep`, `gtar` — the managed POSIX userland floor that shadows
  the host's shell, awk, coreutils, sed, grep, and tar in builds.
- `gsed` — GNU sed (bins `gsed` + `sed`), declared unconditionally by any build that
  needs GNU sed semantics — no host floor's sed flavor is assumable on any platform.
- `openssl` — TLS/crypto library (libssl/libcrypto, static + shared). Lives in core because
  `rust` links it **statically** into cargo (otherwise cargo's `openssl-sys` falls back to a
  hardcoded Homebrew path on macOS); the `world` tome's curl/git/openssh link it shared,
  cross-tome. TLS comes from here, never the host.
- `musl`, `linux-headers` — Linux libc and kernel headers (Linux targets only).
- `rust-stage0` — official rustc/cargo binaries, repackaged (fixed-output) to seed the
  `rust` source build; conflicts with `rust` for linked installs.
- `rust` — the Rust toolchain built from source with the tome's llvm.
- `build-env` — meta-package: out-of-core packages that compile from source declare
  `build: ["build-env"]` to pull the whole managed toolchain instead of enumerating it.
- `grimoire` — grimoire itself, pinned to a commit tarball; `grm upgrade grimoire` is
  self-update.

Userland software lives in the `world` tome — core is exactly what bootstraps grimoire
and its managed build environment, nothing more.

Naming follows the rule in
[rune-authoring.md](https://github.com/hermetomancy/grimoire/blob/main/docs/rune-authoring.md):
multi-implementation standard utilities are packaged under their implementation name
(`gmake`, `gsed`) and ship both command names; the generic name is a capability.

## Building

Build and register one package in `dist/index.nuon`:

```sh
grm tome build gmake --path .
```

Build every rune (self-bootstrapping: earlier packages are installed store-only so later
runes can use them as build dependencies; split groups build once and register every
member):

```sh
grm tome build --all --path .
```

The generated `dist/` directory is intentionally ignored by git. Publish its contents to
the package host that `tome.rn` advertises, and sign the index
(`minisign -S -m dist/index.nuon`).

## Host floor

The bootstrap leans on the host for exactly: a POSIX userland at `/usr/bin` + `/bin`
(shadowed by dash/mawk/uutils/gsed/ggrep once built), a C compiler boundary (replaced by
toolchain-wrappers once built), and on macOS the SDK (located via `xcrun`). Linking is fully managed —
ld64.lld on macOS, lld on Linux — so no host linker survives past bootstrap.
