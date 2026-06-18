# pnpm-pacquet-prefer-unplugged-permission-sample

Reproduction for a bug where pacquet (pnpm's Rust install engine) installs executable
files with mode `0644` instead of `0755` on Linux **when the import method is `clone`
(reflink)**, making the installed binary impossible to execute (`EACCES` on spawn).

> **Note**
> The `issue-2` branch of this repository is the reproduction environment for a new
> issue, filed as a follow-up to [#12171](https://github.com/pnpm/pnpm/issues/12171) /
> [#12385](https://github.com/pnpm/pnpm/pull/12385). The original issue was about the
> **copy fallback** path (fixed in #12385); this one is about the still-unfixed
> **clone (reflink) success** path.

## Bug

When pacquet imports a package file that is executable in the tarball (mode `0755`) using
the **clone (reflink)** import method, the file is materialized in `node_modules` with mode
`0644`. The CAS store entry is still correct (`0755`, `-exec` suffix), but the reflinked
copy in the virtual store loses the exec bit.

**Example of an affected package:** `@esbuild/linux-x64` — ships its native binary at
`bin/esbuild` with `0755` in the tarball. With `@pnpm/pacquet` and the default `auto`
import method, on a filesystem where reflink succeeds (CircleCI), the binary is installed
as `0644`, so `esbuild` cannot be spawned and `vitest` fails to start.

```
# CAS store — correct
0755  /home/circleci/.local/share/pnpm/store/v11/files/10/1b59d9...-exec

# node_modules — wrong (clone / auto)
0644  node_modules/.pnpm/@esbuild+linux-x64@0.25.12/.../bin/esbuild
```

Same device, **different inode**: the file was reflinked (cloned), not hardlinked, and the
clone did not carry the exec bit over.

https://app.circleci.com/pipelines/github/kimulaco/pnpm-pacquet-prefer-unplugged-permission-sample/13/details?job=13a5ea75-dceb-438c-a869-c4478b4092dd&buildNumber=23&jobType=build&workflowId=084ba982-a9eb-447d-8b0b-5f39c4f8dac3

## Root cause

pacquet's default import method is `Auto`, which tries **clone (reflink) → hardlink → copy**
(`crates/package-manager/src/link_file.rs`). On CircleCI the filesystem supports CoW reflink,
so the very first tier (clone) succeeds and the file is materialized by `reflink_copy::reflink`.

The exec bit is only restored on the **copy** tier (`copy_file`, added in
[#12385](https://github.com/pnpm/pnpm/pull/12385)). The **clone success** path applies no
such restoration:

| Import method | Exec bit | Why |
| --- | --- | --- |
| **Clone (reflink) success** | ❌ lost | see platform note below; no exec-bit restoration in the clone path |
| Hardlink success | ✅ | shares the inode, inheriting the CAS mode (`0755`) |
| Copy (fallback) | ✅ | `copy_file` re-adds the exec bit from the `-exec` suffix (#12385) |

### Why it passes locally (macOS) but fails on Linux CI

The behavior of `reflink_copy::reflink` differs by platform:

- **Linux** (`sys/unix/linux.rs`): creates a fresh target with `create_new` (default mode
  `0666 & ~umask` = `0644`), then `FICLONE` clones **only the data extents** — the source mode
  is **not** inherited. Result: `0644`.
- **macOS** (`sys/unix/macos.rs`): uses `clonefile(2)`, which **does** copy the file mode.
  Result: `0755`.

So the clone path silently produces `0644` on Linux and `0755` on macOS — which is exactly
why this reproduces on CircleCI but not when installing locally on macOS. It also does not
reproduce on GitHub Actions with the same Docker image, because that filesystem does not take
the reflink-success path.

## How to reproduce

Reproduces on CircleCI (`.circleci/config.yml`). The default `test` job fails at `pnpm test`:

```bash
$ vitest run
failed to load config from /home/circleci/project/vitest.config.ts

⎯⎯⎯⎯⎯⎯⎯ Startup Error ⎯⎯⎯⎯⎯⎯⎯⎯
Error: The service was stopped: spawn /home/circleci/project/node_modules/.pnpm/@esbuild+linux-x64@0.25.12/node_modules/@esbuild/linux-x64/bin/esbuild EACCES
    at /home/circleci/project/node_modules/.pnpm/esbuild@0.25.12/node_modules/esbuild/lib/main.js:949:34
    at responseCallbacks.<computed> (/home/circleci/project/node_modules/.pnpm/esbuild@0.25.12/node_modules/esbuild/lib/main.js:603:9)
    at ChildProcess.afterClose (/home/circleci/project/node_modules/.pnpm/esbuild@0.25.12/node_modules/esbuild/lib/main.js:594:28)
    at ChildProcess.emit (node:events:509:28)
    at ChildProcess._handle.onexit (node:internal/child_process:293:12)
    at onErrorNT (node:internal/child_process:508:16)
    at process.processTicksAndRejections (node:internal/process/task_queues:90:21)

[ELIFECYCLE] Test failed. See above for more details.

Exited with code exit status 1
```

### Per-method breakdown (`diagnose-import-methods` workflow)

The `diagnose` job pins `packageImportMethod` and installs fresh (no cache) for each value.
Observed on CircleCI (`cimg/node:24.16`, umask `0022`):

| Matrix cell | install log | binary mode | `pnpm test` |
| --- | --- | --- | --- |
| `diagnose-auto` | `cloned` | **0644** | ❌ EACCES |
| `diagnose-clone` | `cloned` | **0644** | ❌ EACCES |
| `diagnose-hardlink` | `hard linked` | 0755 | ✅ |
| `diagnose-copy` | `copied` | 0755 | ✅ |

Only `clone`/`auto` fail, while `hardlink`/`copy` pass — confirming the clone (reflink)
success path as the cause.

https://app.circleci.com/pipelines/github/kimulaco/pnpm-pacquet-prefer-unplugged-permission-sample/13/details?job=13a5ea75-dceb-438c-a869-c4478b4092dd&buildNumber=23&jobType=build&workflowId=084ba982-a9eb-447d-8b0b-5f39c4f8dac3

## Workaround

Pin the import method to anything other than `clone` in `pnpm-workspace.yaml`:

```yaml
packageImportMethod: copy      # uses the fixed copy_file path (#12385)
# or
packageImportMethod: hardlink  # inherits the CAS mode (0755) via the shared inode
```

## Environment

- pnpm: 11.7.0 with `configDependencies: "@pnpm/pacquet": 0.11.10`
- Node.js: 24.16.0
- OS: Linux (`cimg/node:24.16`)
- Note: `allowBuilds.esbuild: false` is required so esbuild's `postinstall` script does not
  re-chmod the binary to `0755` and mask the bug.
