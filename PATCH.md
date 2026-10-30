# Sonalect patch: shared-library copy in `llama-cpp-sys-2`

This fork carries one fix on top of upstream
[utilityai/llama-cpp-rs](https://github.com/utilityai/llama-cpp-rs). This file says what it is
and how to carry it to a new upstream version, without studying the problem again.

## Names

- Fork: [sonalect/llama-cpp-rs](https://github.com/sonalect/llama-cpp-rs), remote `origin`.
- Upstream: [utilityai/llama-cpp-rs](https://github.com/utilityai/llama-cpp-rs), remote
  `upstream` (`git remote add upstream https://github.com/utilityai/llama-cpp-rs.git`).
- Patched branch: `sonalect/<upstream tag>-shared-libs`.
- Patched tag: `<upstream tag>-sonalect.<n>`, what faden depends on.
- Pull request branch: `fix/shared-libs-copy`, the same fix on upstream `main`.

The first one: branch `sonalect/0.1.160-shared-libs`, tag `0.1.160-sonalect.1`.

Two rules: never change a crate version, and never move the `llama-cpp-sys-2/llama.cpp`
submodule away from the commit that the upstream tag pins. faden's tests check token ids
against that exact llama.cpp.

## The problem

With the feature `dynamic-link` (which `dynamic-backends` turns on), the end of `main()` in
`llama-cpp-sys-2/build.rs` (the block `// copy DLLs to target`) hard-links each shared library
into `<target dir>/`, `<target dir>/examples/` and `<target dir>/deps/` with
`std::fs::hard_link(...).unwrap()`. `<target dir>` is the third ancestor of `OUT_DIR`.

- **Cargo, Linux and macOS.** It takes only `libllama.so`, `libggml.so`, ..., which are CMake
  symlinks (`libllama.so -> libllama.so.0 -> libllama.so.0.6.0`). `hard_link` links the symlink
  itself, so `target/debug/libllama.so` points at a `libllama.so.0` that is not there: it
  dangles. On the next run of the build script, `dst.exists()` follows the dangling link and
  says false, `hard_link` fails, and the build stops with

  ```text
  panicked at llama-cpp-sys-2/build.rs:1552:56:
  called `Result::unwrap()` on an `Err` value: Os { code: 17, kind: AlreadyExists, message: "File exists" }
  ```

  Any second run of the build script in the same profile directory does it: other features,
  a changed `LLAMA_LIB_PROFILE`, `cargo clean -p llama-cpp-sys-2`, `cargo clippy`, building an
  example. Upstream issue: utilityai/llama-cpp-rs#896 (PR #980 was closed unmerged).
- **Bazel** (rules_rust `cargo_build_script`). The third ancestor of `OUT_DIR` is
  `bazel-out/<config>/bin`, which has no `deps`: the `deps` link panics with `NotFound`. A
  build script must not write outside `OUT_DIR` there anyway.

## The fix

All in `llama-cpp-sys-2/build.rs`, one commit:

1. Copy only into a Cargo profile directory, that is, when `<target dir>/deps` exists.
   Otherwise skip the whole block (Bazel, any other layout).
2. Take the whole chain: on Linux the glob in `extract_lib_assets()` is `lib*.so*` (the
   variable `versions`). On macOS `lib*.dylib` already matches `libllama.0.dylib` and the
   others.
3. `copy_lib_asset()`: each symlink is recreated as a symlink with the same relative target.
   The real file is hard-linked, or copied with `fs::copy` when hard-linking fails.
4. Idempotent: the destination is looked at with `symlink_metadata` (a dangling link counts
   as present), removed, then created again.
5. Never panic in this block: a failed copy is a
   `cargo:warning=could not copy <src> to <dst>: <error>`, and the build goes on.

Windows changes only through the shared code. The Windows pattern (`*.dll`) is the subject
of upstream PR #1155, which we leave alone. Our lines in `extract_lib_assets()` sit below
the ones that #1155 changes, so the two merge without conflict.

To see the fix: `git show 0.1.160-sonalect.1 -- llama-cpp-sys-2/build.rs`.

## When a new upstream version comes out

Below, `NEW` is the new upstream tag (say `0.1.170`), and `OLD` / `OLD_PATCHED` are the
previous ones (`0.1.160` / `0.1.160-sonalect.1`).

### 1. Is the patch still needed?

```sh
git fetch upstream --tags
git show NEW:llama-cpp-sys-2/build.rs | grep -n 'copy DLLs to target' -A 12
```

If the block still calls `hard_link(...).unwrap()` behind `if !dst.exists()`, the patch is
needed. If upstream merged our pull request, or fixed it another way, faden can go back to
the crates.io version. Run the checks of step 3 on the upstream tag to make sure.

### 2. Branch and apply

```sh
git switch -c sonalect/NEW-shared-libs NEW
git submodule update --init --recursive      # the llama.cpp commit that NEW pins; do not move it
git cherry-pick OLD..OLD_PATCHED             # every commit of ours on top of the old tag
```

This file is in the patch commit, so it comes along.

On a conflict, upstream changed the same lines. Rewrite the block by hand so that it does the
five points of "The fix", taking the code from `OLD_PATCHED` as the model. If upstream moved
the copy elsewhere, or the fix would need more than the copy step, stop and look again before
inventing a workaround.

### 3. Check (Linux, in this repository)

Times on 32 cores: a cold build of llama.cpp takes about 70 s, a rerun about 2 s.

```sh
cargo build -p llama-cpp-2 --features dynamic-backends
LLAMA_LIB_PROFILE=Release cargo build -p llama-cpp-2 --features dynamic-backends
cargo build -p llama-cpp-2 --features dynamic-backends
```

The script watches `LLAMA_LIB_PROFILE`, so the second and third builds rerun it in the same
target directory. All three must succeed. Unpatched, the second one panics as shown above.

```sh
ls -lL target/debug/libllama.so target/debug/deps/libllama.so   # two real files, not "No such file"
find target/debug -maxdepth 2 -xtype l                          # no dangling link: prints nothing
```

Then load the backends. Put this into `llama-cpp-2/examples/load_backends.rs` (a throwaway
file, do not commit it):

```rust
use llama_cpp_2::list_llama_ggml_backend_devices;
use llama_cpp_2::llama_backend::{load_backends, LlamaBackend};

fn main() {
    println!("BACKENDS_DIR = {:?}", llama_cpp_sys_2::BACKENDS_DIR);
    load_backends();
    let backend = LlamaBackend::init().expect("LlamaBackend::init failed");
    for device in list_llama_ggml_backend_devices() {
        println!("device {}: {} ({})", device.index, device.name, device.description);
    }
    drop(backend);
    println!("LlamaBackend::init OK");
}
```

```sh
cargo build -p llama-cpp-2 --features dynamic-backends --example load_backends
cd target/debug && LD_LIBRARY_PATH=. ./examples/load_backends; cd -
rm llama-cpp-2/examples/load_backends.rs
```

Expect `load_backend: loaded CPU backend from .../out/backends/libggml-cpu-<variant>.so`, a
CPU device and `LlamaBackend::init OK`. `LD_LIBRARY_PATH=.` makes the loader take the copies
in `target/debug` (`ldd` shows it). Without it, Linux does not look next to the binary;
`cargo run` and `cargo test` set the path themselves.

Optional, the Bazel layout: run the build script by hand with an `OUT_DIR` whose third
ancestor has no `deps`. It must exit 0 and write nothing into `bin/`. It takes about a
minute, since llama.cpp builds from scratch in the new `OUT_DIR`:

```sh
touch llama-cpp-sys-2/build.rs     # so that -vv prints how cargo runs the script
cargo build -vv -p llama-cpp-2 --features dynamic-backends 2>&1 \
  | grep -o 'Running `[^`]*llama-cpp-sys-2-[0-9a-f]*/build-script-build`' | head -1 \
  | sed 's/^Running `//; s/`$//' > /tmp/bs-cmd.txt
BZ=/tmp/bz/bazel-out/k8-fastbuild/bin/external/llama-cpp-sys-2/_bs.out_dir
rm -rf /tmp/bz && mkdir -p "$BZ"
(cd llama-cpp-sys-2 && bash -c "$(sed "s|OUT_DIR=[^ ]*|OUT_DIR=$BZ|" /tmp/bs-cmd.txt)" > /tmp/bz.log 2>&1)
echo "exit=$?"; ls /tmp/bz/bazel-out/k8-fastbuild/bin    # exit=0, only "external"
```

### 4. Tag and push

```sh
git tag NEW-sonalect.1
git push origin sonalect/NEW-shared-libs NEW-sonalect.1
```

If a later fix goes onto the same upstream version, the next tag is `NEW-sonalect.2`.

### 5. faden

Point faden at the new tag. One way, keeping faden's own version requirement on
`llama-cpp-2`, in faden's root `Cargo.toml`:

```toml
[patch.crates-io]
llama-cpp-2 = { git = "https://github.com/sonalect/llama-cpp-rs", tag = "NEW-sonalect.1" }
llama-cpp-sys-2 = { git = "https://github.com/sonalect/llama-cpp-rs", tag = "NEW-sonalect.1" }
```

Cargo fetches the `llama.cpp` submodule of a git dependency itself. Then refresh the lock
files (Cargo, and Bazel's crate_universe) and run faden's tests. A new llama.cpp may change
token ids: that comes from upstream, not from this patch.

### 6. The upstream pull request

As long as the pull request from `fix/shared-libs-copy` is not merged, keep the branch on
upstream `main`:

```sh
git fetch upstream
git switch fix/shared-libs-copy
git rebase upstream/main
git push --force-with-lease origin fix/shared-libs-copy
```
