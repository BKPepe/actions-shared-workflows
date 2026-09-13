# Feeds package test builds on macOS

`feeds-package-test-build-macos.yml` is the macOS counterpart of
`multi-arch-test-build.yml`: it builds the changed packages of a feed on a
macOS runner, with the same steps, artifacts and runtime tests. On Linux the
build runs in the OpenWrt SDK that `openwrt/gh-action-sdk` downloads for every
job. There is no SDK for macOS hosts, so its prebuilt parts are built out of
band by `macos-toolchain.yml` and restored by every pull request job.

| Workflow | Role | Runs | Secrets |
| --- | --- | --- | --- |
| `macos-toolchain.yml` → `reusable_macos-toolchain.yml` | producer: builds the SDK parts on macOS and publishes them | every 6 hours, `workflow_dispatch` | S3 write |
| `feeds-package-test-build-macos.yml` | consumer: restores the SDK parts, builds the changed feed packages on macOS and runtime tests them on Linux | `workflow_call` from feed repositories | none |

## Parity with the Linux workflow

- **Base.** The Linux SDK is the current snapshot of the branch and target.
  The producer follows the same snapshot: its revision (`version.buildinfo`),
  its build configuration (`config.buildinfo`) and its kernel, and the pull
  request job checks that revision out. Other feeds follow their branches on
  both platforms.
- **Prebuilt parts.** The same ones the SDK ships: host tools (including the
  SDK-only ones), the apk host tools, the cross toolchain, and the kernel tree
  kernel modules are built against. The kernel comes from the snapshot SDK
  itself (configuration, `Module.symvers`, modules and vermagic); only its host
  programs are rebuilt for macOS. Host dependencies of packages are built in the
  job, as in the SDK.
- **Package selection and signing.** The job uses the SDK's build configuration,
  its default of enabling every package and of signing each `.apk`, and
  refreshes `.config` after installing each package, as the SDK does before
  every make.
- **Steps.** Per package: `feeds install`, `download`, `check` with `PKG_HASH`
  validation, `refresh` with a dirty-patch check, the `shfmt` check on
  `files/*.init`, then `compile` (stopping at the first failure) and an
  unsigned `package/index`. Packages not available for the architecture are
  skipped with a warning.
- **Inputs, artifacts and runtime tests.** The same `matrix` format with
  `runtime_test`, the same `enable_generic_tests` and `force_generic_tests`
  inputs, the same artifacts and PKG-INFO, and the same runtime tests for each
  architecture whose build succeeded.

Known differences:

- **Runtime tests** cannot run on the macOS runner: GitHub's arm64 macOS
  runners have no Docker and do not support nested virtualization, so neither
  the `openwrt/rootfs` container nor QEMU user emulation is available. A
  follow-up `ubuntu-latest` job runs them instead (see below).
- **BPF toolchain.** The SDK ships a llvm-bpf build; on macOS BPF packages are
  built with the runner's Homebrew `llvm@20` (`CONFIG_BPF_TOOLCHAIN_HOST`).
- **Kernel modules of the core tree.** The SDK does not know the core
  `kernel/linux` package, so a feed package that depends on a kernel module only
  builds that module. The full buildroot packages all core kernel modules then,
  which takes longer but uses the same prebuilt modules.
- **shfmt.** The SDK container has no shfmt, so on Linux the init script check
  only reports the missing command and never fails. shfmt is not installed on
  macOS either, so both behave the same.
- **Target of an architecture.** The Linux SDK container for an architecture
  is built from one target of that architecture (currently `rockchip/armv8`
  for `aarch64_generic` on main); the macOS matrix names its target explicitly.
- **Cores.** Package builds use the 3 cores of the macOS runner (Linux: 4).

## Producer: `macos-toolchain.yml`

Runs every 6 hours for `master`, `openwrt-25.12` and `openwrt-24.10`, one branch
at a time, only in the `openwrt` organisation; `workflow_dispatch` (inputs
`branch` and `force`) works everywhere. For each matrix entry it:

1. downloads `version.buildinfo`, `config.buildinfo`, `profiles.json` and
   `sha256sums` of the snapshot, checks they belong to the same upload, and
   checks out the snapshot revision;
2. hashes the inputs of each part and looks for the archives at `sdk_url`; if
   the published manifest already names this snapshot and all archives, it
   stops;
3. builds each part whose archive is missing, reusing the rest:

   | Archive | Built with | Contents |
   | --- | --- | --- |
   | `macos-tools-<arch>-<branch>-<sha>.tar.zst` | `make tools/install` | `staging_dir/host` |
   | `macos-toolchain-<arch>-<branch>-<sha>.tar.zst` | `make toolchain/install` | `staging_dir/toolchain-*`, `staging_dir/target-*` |
   | `macos-hostpkg-<arch>-<branch>-<sha>.tar.zst` | `make package/system/apk/host/compile` (apk branches only) | the files it installs and its build stamps |
   | `macos-kernel-<arch>-<branch>-<sha>.tar.zst` | see below | the kernel files `target/sdk/Makefile` puts into the SDK, plus `.vermagic` |

4. uploads the new archives, then the manifest `macos-sdk-<branch>-<arch>.json`.

The kernel is assembled from the snapshot SDK: `make target/linux/prepare`
unpacks and patches the kernel source; the SDK tarball is verified against
`sha256sums`; its kernel `.config` is used to build the kbuild host programs
natively (`olddefconfig`, `modules_prepare`), which must not change the
configuration apart from the compiler identification; then the SDK's
`Module.symvers`, `modules.builtin` and modules are copied in, and its vermagic,
which must equal the one in `profiles.json`, is written to `.vermagic`. A kernel
that cannot be assembled does not fail the run: the manifest is published
without a kernel, and it is not retried until its inputs change or `force` is
set.

The build configuration is the snapshot's `config.buildinfo`, with three
changes: `CONFIG_BUILDBOT` is dropped, because in a full buildroot it deletes
`staging_dir` and `build_dir` whenever a restored toolchain meets a newer
checkout (the SDK does not have that code, and everything else it implies is
set explicitly in `config.buildinfo`); BPF packages use the runner's LLVM; and
`CONFIG_SIGN_EACH_PACKAGE` is enabled, as the SDK's own `Config.in` does. Tools,
toolchain and apk host tools are built with the configuration the pull request
job uses, so their build stamps match there, and the name of the host tools
stamp is recorded in the manifest.

The upload only runs for `schedule` and `workflow_dispatch` events. The S3
credentials are written to a temporary `mc` configuration that only exists
while uploading, never while anything builds, and the producer runs no code from
feed pull requests. Without S3 credentials the producer builds every part,
which makes it usable as a smoke test, but publishes nothing.

### Manifest

```json
{
  "schema": 2,
  "revision": "r36227-36d3bea3a9",
  "openwrt_commit": "36d3bea3a96348d0657e2503da55d23743cb3e55",
  "snapshot": "https://downloads.openwrt.org/snapshots/targets/armsr/armv8",
  "sdk_file": "openwrt-sdk-armsr-armv8_gcc-14.4.0_musl.Linux-x86_64.tar.zst",
  "sdk_sha256": "fa33a7bc…",
  "target": "armsr-armv8",
  "image_version": "20260831.0337.3",
  "llvm_formula": "llvm@20",
  "tools": "macos-tools-aarch64_generic-master-….tar.zst",
  "tools_stamp": "staging_dir/host/stamp/.tools_compile_nyyy…",
  "toolchain": "macos-toolchain-aarch64_generic-master-….tar.zst",
  "hostpkg": "macos-hostpkg-aarch64_generic-master-….tar.zst",
  "kernel": "macos-kernel-aarch64_generic-master-….tar.zst",
  "kernel_failed": "",
  "vermagic": "cc7c3d3eae998f6650a571d971ffe247",
  "snapshot_vermagic": "cc7c3d3eae998f6650a571d971ffe247",
  "config": "CONFIG_TARGET_armsr=y\n…"
}
```

Old archives are never deleted; superseded ones can be pruned by hand.

### Secrets and variables

The trigger workflow maps the repository/organisation secrets to the reusable
workflow, accepting either the `ccache_s3_*` names used by `packages.yml` and
`kernel.yml` or plain `s3_*`:

| Secret | Description |
| --- | --- |
| `ccache_s3_endpoint` / `s3_endpoint` | S3 API endpoint, without bucket or path, e.g. `https://<account-id>.r2.cloudflarestorage.com` |
| `ccache_s3_bucket` / `s3_bucket` | bucket name |
| `ccache_s3_access_key` / `s3_access_key` | access key ID |
| `ccache_s3_secret_key` / `s3_secret_key` | secret access key |

The repository variable `MACOS_SDK_URL` is the public URL of that bucket, used
to find archives that can be reused. It defaults to
`https://s3-ccache.openwrt-ci.ansuel.com`.

## Consumer: `feeds-package-test-build-macos.yml`

Reusable workflow for feed repositories, mirroring the inputs and the artifacts
of `multi-arch-test-build.yml`. It can be called from the same workflow as the
Linux job; its concurrency group is distinct.

```yaml
jobs:
  test-macos:
    uses: openwrt/actions-shared-workflows/.github/workflows/feeds-package-test-build-macos.yml@main
```

| Input | Default | Description |
| --- | --- | --- |
| `enable_generic_tests` | `true` | run the generic runtime tests, as in the Linux workflow |
| `force_generic_tests` | `true` | run the generic runtime tests even for packages with a `test.sh` |
| `matrix` | `aarch64_generic` (runtime tested), `arm_cortex-a9_vfpv3-d16` | JSON build matrix, same format and `runtime_test` key as the Linux workflow |
| `packages` | changed packages, else the feed's default set | explicit space-separated package list |
| `feed_repository` | the checked-out workspace | `owner/name` of a feed to clone instead (self-tests) |
| `branch` | derived from the pull request base branch | OpenWrt branch |
| `sdk_url` | `https://s3-ccache.openwrt-ci.ansuel.com` | base URL that serves the manifests and archives (public read) |

The archives are fetched anonymously, so the bucket (or a CDN in front of it)
must be publicly readable at `sdk_url`. The job never builds the SDK parts
itself. It fails immediately, with a pointer to the producer, when no manifest
is published for the branch and architecture, when the configuration needs a
different set of host tools than the published ones, or when the restored host
tools or toolchain get rebuilt anyway. Pull requests against a branch the
producer does not publish (for example an end-of-life release) fail the same
way; callers can limit the job with `if: github.base_ref == 'master'` or
dispatch the producer for that branch.

The runtime tests run in a follow-up `ubuntu-latest` job, with the steps of
`multi-arch-test-build.yml` unchanged, for every matrix entry with
`runtime_test: true` whose macOS build succeeded: it downloads the packages the
macOS job uploaded, checks the feed out, and runs `test_entrypoint.sh` in the
`openwrt/rootfs` container under QEMU. Packages built on macOS record the
absolute path of the feed as their origin (the SDK records `/feed/…`), so the
feed is also mounted at that path inside `/ci`, where the script looks for
`test.sh`, `pre-test.sh` and `test-version.sh`.

## Self-test from this repository

`manual-test-feeds-macos.yml` (`workflow_dispatch` only) runs the consumer
against `openwrt/packages` – or any `feed_repository` – with an optional
package list. Set the repository variable `MACOS_SDK_URL` to point the self-test
at your own bucket; otherwise it uses the default `sdk_url`.
