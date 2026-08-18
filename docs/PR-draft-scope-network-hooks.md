# PR 草稿：scope Winsock/IOCP hooks to network subprocesses only

> 用途：随时可推到上游 yuaotian/antigravity-proxy。基于 fork 分支 `main`（提交 cf6310c）。
> 拟稿：2026-08-18 思思；验收：本机 Antigravity v2.8.1 实测通过（2026-08-18）。

## Title

fix: scope Winsock/IOCP hooks to network subprocesses only (fix local TLS loopback breakage)

## Body

### Problem

`version.dll` is auto-loaded by the Antigravity main process and its
identically-named Electron child processes (DLL search order). The v2.2 proxy
installed full Winsock/IOCP hooks in **every** process, including the UI host
processes. This breaks the loopback HTTPS/gRPC connection between the UI and
the language server (self-signed cert), producing:

```
tls: unknown certificate
Lost connection to the language server. Agent features may not work.
```

The SOCKS5 tunnel itself succeeds (logs show `隧道建立成功`), but the app UI
cannot talk to the language server, so the proxy appears dead even though
traffic is being intercepted.

### Fix

Classify processes and scope hooks accordingly:

- `Antigravity.exe` / `Antigravity IDE.exe` (host processes): install
  **only** `CreateProcessA/W` hooks (needed to keep injecting child
  processes); skip network hooks entirely.
- `language_server.exe` / `node.exe` (network subprocesses): full network
  hooks → SOCKS5 proxy, so Google API traffic goes through the proxy while the
  UI↔language-server loopback stays untouched.

Changes:

- `src/hooks/ProcessName.hpp`: add `IsAntigravityHostProcessName()`
- `src/main.cpp`: pass process name / network-hook flag to `Hooks::Install()`,
  log injector mode
- `src/hooks/Hooks.cpp`: `Install(bool enableNetworkHooks)` — injector mode
  creates only CreateProcess hooks
- `tests/test_process_name.cpp`: classification tests for main process, IDE,
  language server, node
- `CMakeLists.txt`: LLVM-MinGW `.def` link compat + static CRT options
  (Windows x64 build chain)

### Verification

Built with LLVM-MinGW (clang x64, Release, tests ON):

- `antigravity_tests.exe` / `process_name_tests.exe`: pass
- Deployed DLL hash matches baseline, proxy log shows:
  - host process: `注入器模式`（injector mode, network hooks skipped）
  - language server: `所有 API Hook 安装成功 (Phase 1-3)`
  - `SOCKS5: 隧道建立成功` to `daily-cloudcode-pa.googleapis.com:443`
- language server log: `Auth succeeded`、`initialized server successfully`,
  no more `tls: unknown certificate` / `i/o timeout` errors

Verified live on Antigravity **v2.8.1** (2026-08-18).

### Notes for maintainer

The fix is additive and backwards compatible: non-Antigravity hosts (where
only one process loads the DLL and network hooks are always desired) behave
exactly as before, since `IsAntigravityHostProcessName()` only matches
`antigravity.exe` / `antigravity ide.exe`.

If the app renames its host processes in a future release, only
`IsAntigravityHostProcessName()` needs updating (see `target_processes` in
config.json for the current process names).

## 推送命令备忘

```bash
# 分支名保持一致（避免节外生枝）：
git push origin main
# 或单独开分支：
git checkout -b fix/scope-network-hooks-20260818
git push origin fix/scope-network-hooks-20260818
# 然后 gh pr create --repo yuaotian/antigravity-proxy --head SilenceSik:fix/scope-network-hooks-20260818 --title "..." --body "..."
```

> 注意：PR 是**对外公开**操作，推之前先给主人过目。