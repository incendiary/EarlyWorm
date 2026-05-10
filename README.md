# EarlyWorm

A C# proof-of-concept for **Early Bird APC process injection** using dynamic function resolution via [DInvoke](https://github.com/TheWover/DInvoke) to avoid static IAT entries detectable by EDRs.

> **Authorized use only.** This tool was developed under signed scope of work for authorized red team engagements. It is published for educational reference. The author assumes no liability. Users are solely responsible for ensuring they have explicit written authorization before use in any environment.

---

## Technique

The injection sequence:

1. `CreateProcess` — spawns a target process (`notepad.exe`) in a `SUSPENDED` state
2. `VirtualAllocEx` — allocates RW memory in the remote process
3. `WriteProcessMemory` — copies shellcode into that allocation
4. `VirtualProtectEx` — flips protection to `PAGE_EXECUTE_READ`
5. `QueueUserAPC` _(resolved via DInvoke)_ — queues the shellcode as an APC on the suspended thread
6. `ResumeThread` — releases the thread; the queued APC fires before any user code runs

`QueueUserAPC` is resolved at runtime via `DInvoke.DynamicInvoke.Generic.GetLibraryAddress` rather than a static `[DllImport]`, removing the IAT entry that EDR hooks commonly target.

---

## Requirements

- [.NET 8.0 SDK](https://dotnet.microsoft.com/download/dotnet/8.0) (LTS, supported until November 2026)
- Windows (Win32 APIs; will not build or run on Linux/macOS)

---

## Build

```bash
git clone https://github.com/incendiary/EarlyWorm.git
cd EarlyWorm
dotnet build EarlyWorm.sln
```

Output lands in `EarlyWorm/bin/`. NuGet restore (DInvoke 1.0.4) happens automatically.

## Dev Setup (pre-commit hooks)

If you plan to contribute, install the pre-commit hooks to enforce secret scanning and formatting before each commit:

```bash
pip install pre-commit
pre-commit install
```

Hooks run automatically on `git commit`. To run manually against all files:

```bash
pre-commit run --all-files
```

---

## Shellcode

The embedded shellcode is a benign `calc.exe` stub generated with:

```
msfvenom -p windows/x64/exec CMD=calc.exe -f csharp
```

Replace `buf` in `Program.cs` with your own payload. Alternative loading stubs (URL fetch, disk read) are provided as commented examples in the same file.

---

## Project Layout

```
EarlyWorm/
├── EarlyWorm.sln
└── EarlyWorm/
    ├── EarlyWorm.csproj      # net6.0, depends on DInvoke 1.0.4
    ├── Program.cs            # Entry point; attack() holds the injection sequence
    └── Imports/
        └── Imports.cs        # P/Invoke declarations and Win32 structs
```

---

## Roadmap

| # | Status | Description |
|---|--------|-------------|
| [#2](https://github.com/incendiary/EarlyWorm/issues/2) | ✅ Closed | Upgrade target framework from `net6.0` (EOL) to `net8.0` (LTS) |
| [#3](https://github.com/incendiary/EarlyWorm/issues/3) | ✅ Closed | Code quality pass — remove dead code and comment noise |
| [#4](https://github.com/incendiary/EarlyWorm/issues/4) | ✅ Closed | Add `.editorconfig` enforcing Microsoft C# conventions |
| [#5](https://github.com/incendiary/EarlyWorm/issues/5) | ✅ Closed | Add pre-commit hooks (gitleaks + dotnet format) |
| [#6](https://github.com/incendiary/EarlyWorm/issues/6) | ✅ Closed | Enable Dependabot for NuGet dependency updates |
| [#7](https://github.com/incendiary/EarlyWorm/issues/7) | ✅ Closed | Write README.md with setup instructions and roadmap |
| [#8](https://github.com/incendiary/EarlyWorm/issues/8) | ✅ Closed | Enable branch protection and make repository public |
| [#9](https://github.com/incendiary/EarlyWorm/issues/9) | ✅ Closed | Remove residual PDiddyVoke paths from git index |
| [#10](https://github.com/incendiary/EarlyWorm/issues/10) | ✅ Closed | Consider integration test harness — manual smoke test checklist added to README |

---

## Testing

This is a Windows-only PoC. There is no automated test suite — the injection sequence executes real shellcode into a live process, which is not appropriate for a shared CI environment.

**Manual smoke test (on an isolated Windows VM):**

- [ ] Build succeeds: `dotnet build EarlyWorm.sln --configuration Release`
- [ ] Run the binary — `notepad.exe` spawns briefly in a suspended state
- [ ] `calc.exe` launches (confirming the embedded shellcode executed via APC)
- [ ] `notepad.exe` exits cleanly after the APC fires
- [ ] No crashes, no unhandled exceptions in the console

> Always test on an isolated VM. Never run against a production or shared system.

---

## Contributing

PRs and fixes welcome. Things should work, but some of this code predates the public release cleanup and hasn't been fully re-tested in every environment.

---

> *Claude (Anthropic) was used to help uplift this project for public release — reviewing for secrets, refactoring dead code, and setting up tooling. Logic and technique are original.*
