# repro-pacquet-prefer-unplugged-permission

Reproduction for a bug where pacquet (pnpm's Rust install engine) installs files from packages with `preferUnplugged: true` without preserving the executable bit on Linux.

## Bug

When pacquet installs a package that has `preferUnplugged: true` and no `bin` field, files that are executable in the tarball (mode `0755`) are written to `node_modules` with mode `0644`, making them impossible to execute.

**Example of an affected package:** `@esbuild/linux-x64` — ships its native binary at `bin/esbuild` with `0755` in the tarball, but pacquet installs it with `0644`. Any package with `preferUnplugged: true` and no `bin` field that contains executable files may be affected.

https://github.com/evanw/esbuild/blob/v0.28.0/npm/%40esbuild/linux-x64/package.json

```
# CAS store — correct
Mode: 0755  /home/circleci/.local/share/pnpm/store/v11/files/10/1b59d9...-exec

# node_modules — wrong
Mode: 0644  node_modules/.pnpm/@esbuild+linux-x64@0.25.12/.../bin/esbuild
```

Same device, different inode: pacquet is copying the file instead of hardlinking, and the copy does not preserve the exec bit.

## Root cause (suspected)

`@esbuild/linux-x64/package.json` declares `preferUnplugged: true` with no `bin` field. pacquet applies `ensure_executable_bits` only to files listed in `bin`, so the native binary falls through without the correct mode being set.

## How to reproduce

Reproduces on CircleCI (`.circleci/config.yml`). The `Test` step fails with:

https://app.circleci.com/jobs/github/kimulaco/pnpm-pacquet-esbuild-sample/7

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

The `Debug esbuild binary permissions` step shows `0644` on the binary and `0755` on the CAS entry.

https://github.com/kimulaco/pnpm-pacquet-esbuild-sample/actions/runs/26889393149

Does not reproduce on GitHub Actions (`.github/workflows/test.yml`) with the same Docker image (`cimg/node:24.16`). The difference is likely the filesystem setup: on CircleCI, both the pnpm CAS store and `node_modules` reside on the same overlayfs device, triggering a same-device copy path in pacquet that does not preserve the exec bit.

## Environment

- pnpm: 11.5.1 with `configDependencies: "@pnpm/pacquet": 0.2.13`
- Node.js: 24.16.0
- OS: Linux (`cimg/node:24.16`, overlayfs)
