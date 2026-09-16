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
  The producer follows the same snapshot: its revision, its build
  configuration (`config.buildinfo`) and its kernel, and the pull request job
  checks that revision out. Other feeds follow their branches on both
  platforms.
- **Prebuilt parts.** The same ones the SDK ships: host tools, the apk host
  tools on branches that use apk, the cross toolchain, and the kernel tree
  kernel modules are built against. Host dependencies of packages are built in
  the job, as in the SDK.
- **Configuration.** The SDK's: the snapshot configuration, every package
  enabled and each `.apk` signed, with `.config` refreshed after installing
  each package.
- **Steps.** Per package: `feeds install`, `download`, `check` with `PKG_HASH`
  validation, `refresh` with a dirty-patch check, the `shfmt` check on
  `files/*.init`, then `compile` (stopping at the first failure) and an
  unsigned `package/index`. Packages not available for the architecture are
  skipped with a warning.
- **Kernel modules.** As in the SDK, a feed package that depends on a kernel
  module builds the core `kernel/linux` package. The full buildroot also makes
  that package depend on `linux-firmware` and `gpio-button-hotplug`, which the
  SDK does not contain; the job builds it without them.
- **Inputs, artifacts and runtime tests.** The same `matrix` format with
  `runtime_test`, the same `enable_generic_tests` and `force_generic_tests`
  inputs, packages and logs artifacts with the same contents and PKG-INFO
  (named `<arch>-macos-PR<n>-<sha>`), and the same runtime tests for each
  architecture with `runtime_test` whose build succeeded.

Known differences:

- **Architectures.** The producer publishes SDK parts for `aarch64_generic`
  and `arm_cortex-a9_vfpv3-d16`, the default matrix of both workflows (Linux:
  10 architectures). A matrix entry for any other architecture fails, because
  no manifest is published for it.
- **Runtime tests** cannot run on the macOS runner: GitHub's arm64 macOS
  runners have no Docker and do not support nested virtualization. A
  follow-up `ubuntu-latest` job runs them instead (see below).
- **BPF toolchain.** The SDK ships a llvm-bpf build; on macOS BPF packages are
  built with the runner's Homebrew `llvm@20` (`CONFIG_BPF_TOOLCHAIN_HOST`).
- **shfmt.** The SDK container has no shfmt, so on Linux the init script check
  never rejects anything. On macOS the check only runs when shfmt is
  installed, and it is not, so both behave the same.
- **Target of an architecture.** The Linux SDK container for an architecture
  is built from one target of that architecture (currently `rockchip/armv8`
  for `aarch64_generic` on main). On macOS it is the target the producer's
  matrix builds for that architecture (`armsr/armv8`); the consumer restores
  those parts whatever target its own matrix names.
- **Cores.** Package builds use the 3 cores of the macOS runner (Linux: 4).
- **Time limit.** The macOS build job is cancelled after 90 minutes per
  architecture, including restoring the SDK parts and updating the feeds
  (Linux: GitHub's default of 360 minutes).
- **Snapshot revision.** The Linux job downloads the SDK of the current
  snapshot when it runs. The macOS job builds at the snapshot revision the
  producer last published, which lags by the producer's schedule and build
  time and stays behind while the producer fails. The job's step summary
  names the revision.

## Producer: `macos-toolchain.yml`

Runs every 6 hours for `master`, `openwrt-25.12` and `openwrt-24.10`, one branch
at a time, only in the `openwrt` organisation; `workflow_dispatch` works
everywhere, with inputs `branch` and `force`, which rebuilds every part even
when its archive is published. For each architecture of the reusable
workflow's `matrix` input (arch/target pairs, by default `aarch64_generic` on
`armsr-armv8` and `arm_cortex-a9_vfpv3-d16` on `mvebu-cortexa9`) it checks out
the snapshot revision, builds the parts below and publishes them at `sdk_url`:

| Archive | Contents |
| --- | --- |
| `macos-tools-<arch>-<branch>-<sha>.tar.zst` | `staging_dir/host` (`make tools/install`) |
| `macos-toolchain-<arch>-<branch>-<sha>.tar.zst` | `staging_dir/toolchain-*`, `staging_dir/target-*` (`make toolchain/install`) |
| `macos-hostpkg-<arch>-<branch>-<sha>.tar.zst` | the apk host tools, their host dependencies, build stamps and source archives (apk branches only) |
| `macos-kernel-<arch>-<branch>-<sha>.tar.zst` | the kernel files `target/sdk/Makefile` puts into the SDK, plus `.vermagic` |

Each archive is named after a hash of the OpenWrt files and configuration it
is built from, so a part is only rebuilt when those change or its archive is
missing, and a run whose parts are all published stops early. Finished
archives are uploaded even if a later part fails, so the next run reuses them.
The manifest `macos-sdk-<branch>-<arch>.json` is uploaded last, only when
every part but the kernel succeeded and every archive it names can be read at
`sdk_url`. The runner image and its tools are not part of the hash; the
manifest records the image version.

The kernel tree is the snapshot revision's patched kernel source, prepared on
macOS with the snapshot SDK's kernel configuration, whose `Module.symvers`,
modules and vermagic (checked against `profiles.json`) are copied in; only
the kbuild host programs and generated headers are built on macOS. A kernel
that cannot be assembled does not fail the run: the manifest is published
without a kernel. It is retried once; after a second failure it is not
retried until its inputs change or `force` is set.

The build configuration is the snapshot's `config.buildinfo` with three
changes: `CONFIG_BUILDBOT` is dropped, because in a full buildroot its
toolchain version check deletes the target and toolchain directories whenever
a restored toolchain meets a newer checkout; BPF packages use the runner's
LLVM; and `CONFIG_SIGN_EACH_PACKAGE` is enabled, as the SDK's own `Config.in`
does.

The uploads only run for `schedule` and `workflow_dispatch` events. The S3
credentials are only given to the upload steps, never to a step that builds
anything, and the producer runs no code from feed pull requests. Without S3
credentials the producer builds every part, which makes it usable as a smoke
test, but publishes nothing. Old archives are never deleted. Archives that no
current manifest names can be pruned by hand; deleting one a manifest names
fails every consumer job for that branch and architecture at the restore step
until the producer's next run rebuilds it.

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
  "kernel_failures": 0,
  "vermagic": "cc7c3d3eae998f6650a571d971ffe247",
  "snapshot_vermagic": "cc7c3d3eae998f6650a571d971ffe247",
  "config": "CONFIG_TARGET_armsr=y\n…"
}
```

### Secrets and variables

The trigger workflow maps the repository/organisation secrets to the reusable
workflow, accepting either the `ccache_s3_*` names used by `packages.yml` and
`kernel.yml` or plain `s3_*`:

| Secret | Description |
| --- | --- |
| `ccache_s3_endpoint` / `s3_endpoint` | S3 API endpoint, e.g. `https://<account-id>.r2.cloudflarestorage.com`; a bucket or path in it is ignored |
| `ccache_s3_bucket` / `s3_bucket` | bucket name |
| `ccache_s3_access_key` / `s3_access_key` | access key ID |
| `ccache_s3_secret_key` / `s3_secret_key` | secret access key |

Whitespace around the values, such as a newline from pasting, is ignored.

The repository variable `MACOS_SDK_URL` is the public URL of that bucket, used
to reuse published archives and to check the uploaded ones. It defaults to
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
must be publicly readable at `sdk_url`. The job has no step that builds the
SDK parts. It fails before building any package, with a pointer to the
producer, when no manifest is published for the branch and architecture, when
the configuration needs a different set of host tools than the published
ones, when the restored toolchain or apk host tools are incomplete, or when
the restored kernel's vermagic is not the snapshot's. It warns when a package
build rebuilt a restored host build stamp. When no kernel is published,
packages that need the kernel tree are skipped with a warning.

Pull requests against a branch the producer does not publish (for example an
end-of-life release) fail the same way; callers can limit the job with
`if: github.base_ref == 'master'`. Dispatching the producer for another branch
only works while downloads.openwrt.org still has snapshots for it, and the
schedule does not refresh that branch afterwards.

The runtime tests run in a follow-up `ubuntu-latest` job, with the steps of
`multi-arch-test-build.yml` unchanged, for every matrix entry with
`runtime_test: true` whose macOS build succeeded: it downloads the packages the
macOS job uploaded, checks the feed out, and runs `test_entrypoint.sh` in the
`openwrt/rootfs` container under QEMU. Packages built on macOS record the
absolute path of the feed as their origin (the SDK records `/feed/…`), so the
feed is also mounted at that path inside `/ci`, where the script looks for
`test.sh`, `pre-test.sh` and `test-version.sh`. With `feed_repository`, the
test job checks out the feed commit the packages were built from.

## Self-test from this repository

`manual-test-feeds-macos.yml` (`workflow_dispatch` only) runs the consumer
against `openwrt/packages` – or any `feed_repository` – with an optional
package list. Set the repository variable `MACOS_SDK_URL` to point the self-test
at your own bucket; otherwise it uses the default `sdk_url`.
