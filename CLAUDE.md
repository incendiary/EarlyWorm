# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

**EarlyWorm** is a C# proof-of-concept for Early Bird APC process injection. It creates a suspended process, writes shellcode into its memory, then queues an APC before resuming the thread so the payload executes in the target process's context. `QueueUserAPC` is resolved dynamically via **DInvoke** rather than imported directly, which avoids a static IAT entry for that function.

## Build

```bash
dotnet build EarlyWorm.sln
dotnet build EarlyWorm/EarlyWorm.csproj
```

Requires .NET 6.0 SDK. NuGet restore happens automatically. Output lands in `EarlyWorm/bin/`.

## Project Layout

```
EarlyWorm/
  Program.cs          # Entry point; attack() holds the full injection sequence
  Imports/Imports.cs  # All P/Invoke declarations and Win32 structs
  EarlyWorm.csproj    # Targets net6.0; depends on DInvoke 1.0.4
```

There are no tests and no additional projects in the solution.

## Architecture

The injection sequence in `Program.cs`:
1. `CreateProcess` → spawns target (`notepad.exe`) in a `SUSPENDED` state
2. `VirtualAllocEx` → allocates RW memory in the remote process
3. `WriteProcessMemory` → copies shellcode into that allocation
4. `VirtualProtectEx` → flips protection to `PAGE_EXECUTE_READ`
5. `QueueUserAPC` (resolved via DInvoke's `GetLibraryAddress`) → queues the shellcode as an APC on the suspended thread
6. `ResumeThread` → releases the thread; the queued APC fires before user code runs

**DInvoke** (`DInvoke.DynamicInvoke.Generic.GetLibraryAddress`) is used exclusively for `QueueUserAPC` resolution. All other Win32 calls use standard `[DllImport]` P/Invoke defined in `Imports/Imports.cs`.

## Pre-Publication Checklist

- [x] **Project renamed**: `PDiddyVoke` → `EarlyWorm` across all files, directories, namespaces, and GitHub
- [x] **Namespace mismatch fixed**: unified to `EarlyWorm` throughout
- [x] **Engagement artifacts sanitised**: private IP and operator username path in comments replaced with generic placeholders
- [x] **Git history**: contains `192.168.1.117` (RFC 1918 test IP, non-sensitive) — accepted, no action needed
- [ ] `.editorconfig` — not yet added
- [ ] `.pre-commit-config.yaml` — not yet added
- [ ] `README.md` — not yet added
