# winrecon

Pure PowerShell Windows privilege escalation reconnaissance script. No external tools, no compiled binaries, no admin required. Drop it on a target and run it.

Outputs a timestamped log file alongside the script.

---

## What it does

Twenty sections, single pass, standard user context:

| # | Section | What it collects |
|---|---------|-----------------|
| 1 | System Identity | OS, build, CPU, RAM, BIOS, hotfixes |
| 2 | User & Privilege Context | Token, SID, group memberships, local accounts, `whoami /priv` |
| 3 | UAC & Security Config | UAC consent levels, LUA, virtualization, FilterAdministratorToken |
| 4 | Defender / AV Status | RTP, tamper protection, behavior monitor, engine/sig versions |
| 5 | VBS / HVCI / Secure Boot | Hypervisor-protected code integrity, Credential Guard, Secure Boot state |
| 6 | Network State | Adapters, listening ports + owning process, firewall profiles, ARP table |
| 7 | Services — Privesc Hunt | SYSTEM service binaries with writable exe/dir, unquoted paths, services in user profiles |
| 8 | Service Binary ACLs | `icacls` output for any writable service binaries found in §7 |
| 9 | Scheduled Tasks | SYSTEM/Highest tasks with writable action directories |
| 10 | PATH Analysis | Every PATH dir (machine + user) flagged for writability |
| 11 | KnownDLLs | Full KnownDLLs registry list (hijack-immune set) |
| 12 | Installed Software | Registry-sourced app list with version, publisher, install path |
| 13 | Running Processes | Top 40 by working set with path |
| 14 | Autorun / Persistence | Run/RunOnce keys (HKLM + HKCU), startup folder contents |
| 15 | Writable ProgramData | Directories writable by current user |
| 16 | Interesting Files | Common temp/public dirs, remote access tools (TeamViewer, AnyDesk, VNC, RustDesk), Dropbox |
| 17 | Shares & Remote Access | `net share`, RDP registry state, WinRM key |
| 18 | Privesc Quick Checks | AlwaysInstallElevated, AppInit_DLLs, IFEO debuggers, print monitor DLLs, LSA packages, WMI event subscriptions, dangerous token privileges, named pipes |
| 19 | **Phantom DLL Hunting** | Pure-PS PE import parser — reads every SYSTEM service binary, extracts import table, cross-references against disk. Any DLL in the import table that doesn't exist on disk is a phantom — potentially plantable if a writable PATH dir exists. No procmon required. |
| 20 | Vector Assessment | Scores and ranks four attack vectors (phantom DLL, service binary replacement, PATH hijack, AMSI/ETW bypass viability). Recommends primary and fallback path. |

---

## Usage

```powershell
powershell -ep bypass -File winrecon.ps1
```

No elevation. No dependencies. Writes `RECON_<HOSTNAME>_<TIMESTAMP>.log` to the same directory.

---

## Simulated output

```
================================================================
  SKYWALKER RECON -- 22DIV / george wu
  Target: WORKSTATION-01
  Date: 2026-06-19 14:32:07
  User: jsmith
  Script: C:\temp\winrecon.ps1
================================================================

=== SECTION 1: SYSTEM IDENTITY ===

  Hostname:      WORKSTATION-01
  OS:            Microsoft Windows 11 Home 10.0.26200.2961
  Build:         26200
  Architecture:  64-bit
  Domain:        WORKGROUP
  Manufacturer:  GIGABYTE
  Model:         G7 GD
  CPU:           Intel(R) Core(TM) i5-11400H @ 2.70GHz
  RAM:           15.8 GB
  BIOS:          FB05
  Install Date:  01/14/2025 09:12:44
  Last Boot:     06/19/2026 08:03:21

  Installed Hotfixes:
    KB5058481  06/10/2026  Security Update
    KB5055523  05/13/2026  Security Update
    KB5050094  04/08/2026  Update

=== SECTION 2: USER & PRIVILEGE CONTEXT ===

  Username:      WORKSTATION-01\jsmith
  SID:           S-1-5-21-3842734961-1284142592-428019729-1001
  Auth Type:     Negotiate
  Is Admin:      False

  Group Memberships:
    WORKSTATION-01\jsmith (S-1-5-21-...-1001)
    Everyone (S-1-1-0)
    BUILTIN\Users (S-1-5-32-545)
    NT AUTHORITY\INTERACTIVE (S-1-5-4)
    NT AUTHORITY\Authenticated Users (S-1-5-11)
    NT AUTHORITY\This Organization (S-1-5-15)
    LOCAL (S-1-2-0)
    NT AUTHORITY\NTLM Authentication (S-1-5-64-10)
    Mandatory Label\Medium Mandatory Level (S-1-16-8192)

  Token Privileges:
    Privilege Name                Description                          State
    ============================= ==================================== ========
    SeShutdownPrivilege           Shut down the system                 Disabled
    SeChangeNotifyPrivilege       Bypass traverse checking             Enabled
    SeUndockPrivilege             Remove computer from docking station Disabled
    SeIncreaseWorkingSetPrivilege Increase a process working set       Disabled
    SeTimeZonePrivilege           Change the time zone                 Disabled

  Local User Accounts:
    Administrator  SID:S-1-5-21-...-500  Disabled:True [ADMIN]
    jsmith         SID:S-1-5-21-...-1001  Disabled:False
    DefaultAccount SID:S-1-5-21-...-503  Disabled:True
    WDAGUtilityAccount SID:S-1-5-21-...-504  Disabled:True

=== SECTION 3: UAC & SECURITY CONFIG ===

  ConsentPromptBehaviorAdmin = 5
  ConsentPromptBehaviorUser = 3
  EnableLUA = 1
  PromptOnSecureDesktop = 1
  EnableVirtualization = 1
  FilterAdministratorToken = 0

=== SECTION 4: DEFENDER / AV STATUS ===

  AMRunningMode:           Normal
  RealTimeProtection:      True
  BehaviorMonitor:         True
  IoavProtection:          True
  OnAccessProtection:      True
  AntivirusEnabled:        True
  AntispywareEnabled:      True
  IsTamperProtected:       False
  NISEnabled:              True
  AMServiceEnabled:        True
  AMProductVersion:        4.18.25050.0
  AMEngineVersion:         1.1.25050.2
  AntispywareSignatureAge: 0 days

  [CRITICAL] TAMPER_PROTECT_OFF -- Tamper Protection disabled -- Defender can be stopped programmatically post-SYSTEM
  [MEDIUM] RTP_ACTIVE -- RealTime Protection active -- binaries scanned on write, TOCTOU race requires this

=== SECTION 5: VBS / HVCI / SECURE BOOT ===

  DeviceGuard\EnableVirtualizationBasedSecurity = 0
  DeviceGuard\RequirePlatformSecurityFeatures = 0
  HVCI Enabled = 0
  SecureBoot = 1

=== SECTION 6: NETWORK STATE ===

  --- Adapters ---
  Ethernet: 192.168.1.105/24 (Up)
  Wi-Fi: 192.168.1.108/24 (Up)

  Default Gateway: 192.168.1.1
  DNS Servers: 192.168.1.1, 8.8.8.8

  --- Listening Ports ---
  0.0.0.0:135    PID:1256  svchost
  0.0.0.0:445    PID:4     System
  0.0.0.0:5040   PID:7332  svchost
  0.0.0.0:7680   PID:3948  svchost
  0.0.0.0:49664  PID:1044  lsass
  127.0.0.1:1042 PID:6812  OfficeClickToRun

  --- Firewall Profiles ---
  Domain Profile Settings:
  State                                 ON
  Private Profile Settings:
  State                                 ON
  Public Profile Settings:
  State                                 ON

=== SECTION 7: SYSTEM SERVICES -- PRIVESC HUNT ===

  Total privileged services: 187

  [*] No writable SYSTEM service binaries or directories found

  --- Services in User Profiles ---
  (none)

  --- Unquoted Service Paths ---
  [*] No unquoted service paths found

=== SECTION 9: SCHEDULED TASKS (SYSTEM/HIGHEST) ===

  SYSTEM/Highest tasks checked: 43, writable: 0

=== SECTION 10: PATH VARIABLE ===

  --- SYSTEM PATH ---
  C:\Windows\system32
  C:\Windows
  C:\Windows\System32\Wbem
  C:\Windows\System32\WindowsPowerShell\v1.0\
  C:\Windows\System32\OpenSSH\
  C:\Program Files\Git\cmd
  C:\Users\jsmith\.local\bin [WRITABLE]

  [HIGH] PATH_WRITABLE -- SYSTEM PATH writable: C:\Users\jsmith\.local\bin

  --- USER PATH ---
  C:\Users\jsmith\AppData\Local\Microsoft\WindowsApps

=== SECTION 11: KnownDLLs ===

  Count: 47
  ntdll.dll, kernel32.dll, kernelbase.dll, advapi32.dll, user32.dll, gdi32.dll, ...

=== SECTION 18: PRIVESC QUICK CHECKS ===

  AlwaysInstallElevated (HKLM): NOT SET (secure)
  AlwaysInstallElevated (HKCU): NOT SET (secure)

  LoadAppInit_DLLs: 0 (0=disabled, 1=enabled)
  AppInit_DLLs: (empty)

  --- IFEO Debugger Entries ---
  [*] No IFEO debugger entries (clean)

  --- Print Monitor DLLs ---
  Monitor: Local Port  Driver: localspl.dll  (IN System32)
  Monitor: Standard TCP/IP Port  Driver: tcpmon.dll  (IN System32)

  --- LSA Auth Packages ---
  Auth Packages: msv1_0
  Security Packages: kerberos, msv1_0, schannel, wdigest, tspkg, pku2u

  --- WMI Event Subscriptions ---
  [*] No WMI event consumers (clean)

  --- Token Privileges (Exploitable) ---
  [*] No dangerous privileges (standard user token)

=== SECTION 19: PHANTOM DLL HUNTING ===

  Scanning SYSTEM service PE imports for phantom DLLs...
  (DLLs in import table that don't exist on disk = plantable)

  KnownDLLs count: 47
  Writable PATH dirs: 1
    C:\Users\jsmith\.local\bin

  Scanning 187 SYSTEM services...

  [CRITICAL] PHANTOM_DLL -- Service 'ClickToRunSvc' (LocalSystem) NORMAL-imports 'osppc.dll' -- NOT ON DISK -- PLANTABLE via writable PATH
  [HIGH] PHANTOM_DLL -- Service 'DiagTrack' (LocalSystem) DELAY-imports 'diagtrack_win.dll' -- NOT ON DISK
  [HIGH] PHANTOM_DLL -- Service 'WpnService' (LocalSystem) NORMAL-imports 'wpncore.dll' -- NOT ON DISK

  Services scanned (unique binaries): 94
  Phantom DLLs found: 3

  --- PHANTOM DLL DETAIL ---
  Service:  ClickToRunSvc (Microsoft Office Click-to-Run Service)
  Account:  LocalSystem
  Binary:   C:\Program Files\Common Files\microsoft shared\ClickToRun\OfficeClickToRun.exe
  Phantom:  osppc.dll (NORMAL-load)
  Plantable: True
  ---
  Service:  DiagTrack (Connected User Experiences and Telemetry)
  Account:  LocalSystem
  Binary:   C:\Windows\System32\svchost.exe
  Phantom:  diagtrack_win.dll (DELAY-load)
  Plantable: False
  ---

=== SECTION 20: SKYWALKER VECTOR ASSESSMENT ===

  ┌─────────────────────────────────────────────────┐
  │  V7 GOLF (phantom_dll)    Score:  95/100     │
  └─────────────────────────────────────────────────┘
  [+] Office ClickToRunSvc detected
  [+] osppc.dll confirmed PHANTOM (not on disk)
  [+] Writable PATH dir available for DLL plant

  ┌─────────────────────────────────────────────────┐
  │  V4 DELTA (svc_replace)   Score:   0/100     │
  └─────────────────────────────────────────────────┘
  [-] No writable SYSTEM service binaries

  ┌─────────────────────────────────────────────────┐
  │  V6 FOXTROT (path_hijack) Score:  70/100     │
  └─────────────────────────────────────────────────┘
  [+] 1 writable dir(s) in machine PATH
  [+] Phantom DLL targets available for PATH plant

  ┌─────────────────────────────────────────────────┐
  │  ECLIPSE (AMSI+ETW)     Score:  80/100     │
  └─────────────────────────────────────────────────┘
  [+] HWBP bypass requires no elevation (standard user)
  [+] SetThreadContext on own threads -- no behavioral signature
  [~] RTP active -- eclipse needed before payload execution
  [+] Tamper Protection OFF -- Defender can be stopped after achieving SYSTEM

  ═══════════════════════════════════════════════════
  RECOMMENDED ATTACK PATH:
    PRIMARY:  V7 GOLF (score 95)
    FALLBACK: V6 FOXTROT (score 70)
    ECLIPSE: VIABLE

  DEPLOY COMMAND:
    python deploy.py --pentest --skip-recon
  ═══════════════════════════════════════════════════

================================================================
  RECON COMPLETE
  Output: C:\temp\RECON_WORKSTATION-01_20260619_143209.log
  Hostname: WORKSTATION-01
  Time: 2026-06-19 14:32:09
  Sections: 20
  Phantom DLLs: 3
  Recommended: V7 GOLF (score 95)
================================================================

  Log written to: C:\temp\RECON_WORKSTATION-01_20260619_143209.log
```

---

## Key feature: phantom DLL detection (Section 19)

Section 19 is a pure-PowerShell PE binary parser. It:

1. Enumerates all services running as `LocalSystem` / `SYSTEM`
2. For each unique service binary, opens the file and walks the PE format manually — DOS header → PE signature → optional header → data directory → import descriptor table
3. Extracts every DLL name from both the normal import table and the delay-load import table
4. Skips `api-ms-win-*` and `ext-ms-win-*` API set forwarders (not real DLLs)
5. Skips anything in the KnownDLLs registry key (protected from hijack)
6. Checks each remaining import against `System32`, `SysWOW64`, `Windows\`, and the service's own directory
7. Any import not found on disk = **phantom DLL** — the service will trigger a DLL search-order walk at runtime looking for something that isn't there
8. Cross-references against writable directories in the machine PATH — if a writable dir exists, the phantom is potentially plantable without admin

No Procmon, no Sysinternals, no external tools.

---

## Notes

- Runs as standard user. Some WMI queries may return partial results on hardened systems.
- Section 19 reads PE files from disk — it does not load or execute them.
- The write-test in Section 19 uses a randomly-named `.skywalker_write_test_*` temp file that is deleted immediately.
- `$ErrorActionPreference = "SilentlyContinue"` throughout — failed queries are skipped silently.
- Tested on Windows 10 21H2, Windows 11 22H2/24H2.

---

## Bug fixes

### Bug #1 — Section 6: all listening ports resolved to the recon script's own process

**Symptom:** Every listening port showed the same PID and process name — the PowerShell process running the script — regardless of which process actually owned the port.

**Root cause:** `$pid` is a read-only automatic variable in PowerShell (`$PID`) that always holds the current process ID. Attempting to assign `$pid = $Matches[4]` silently fails when `$ErrorActionPreference = "SilentlyContinue"` is set — PowerShell swallows the `WriteError` and leaves `$pid` unchanged. Every subsequent `Get-Process -Id $pid` call then queries the script's own PID, returning "powershell" for every port.

**Fix:** Renamed the local variable to `$portPid` throughout the Section 6 netstat block. Added a `(system)` fallback for PIDs where `Get-Process` returns nothing (SYSTEM-owned processes that deny standard-user access).

```powershell
# Before (broken)
$pid = $Matches[4]
$proc = (Get-Process -Id $pid -ErrorAction SilentlyContinue).ProcessName

# After (fixed)
$portPid = $Matches[4]
$proc = (Get-Process -Id $portPid -ErrorAction SilentlyContinue).ProcessName
if (-not $proc) { $proc = "(system)" }
```

**Verified:** Tested against live `netstat -ano` output — ports now resolve to correct owning processes (svchost, System, Dropbox, etc.).

---

### Bug #2 — Section 20: box-drawing characters rendered as garbage (`â"â`)

**Symptom:** Section 20 vector assessment boxes displayed as sequences of `â"â•` instead of `┌─┐│└┘═` in both the terminal and the log file on Windows 11 with default locale settings.

**Root cause:** PowerShell 5.1 on Windows initialises `[Console]::OutputEncoding` to the system OEM codepage (typically CP437 or CP1252 depending on locale). When `Write-Host` emits a UTF-8 string containing box-drawing characters (U+2500–U+257F), the console interprets the multi-byte UTF-8 sequences as individual codepage characters, producing the `â"â` mojibake. The `Out-File -Encoding utf8` call in the `W` function was already correct for the log file, but the console stream was mismatched.

**Fix:** Added two lines immediately after `$ErrorActionPreference`:

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

`[Console]::OutputEncoding` controls the byte encoding used by `Write-Host` / stdout. `$OutputEncoding` controls what PowerShell uses when piping to native executables. Setting both ensures the full output pipeline is UTF-8 consistent.

**Verified:** Box-drawing characters render correctly before and after the fix call — confirmed on Windows 11 26200 with default system locale.

---

## TODO — Release Blackops

_Automated read-only assessment — what a full public-release pass would do for this repo. Suggestions only; nothing above has been changed or removed._

- [ ] Audit git history for AI/Claude attribution; scrub if any is found.
- [ ] Add a `LICENSE` file (MIT or your choice + holder).
- [ ] Add discovery topics for SEO (`gh repo edit --add-topic ...`, up to 20).
- [ ] Cut a tagged release (`v1.0.0`); attach a build artifact if this ships a binary/app.
- [ ] Add a screenshot or diagram to the README if there's a GUI or visual output.
- [ ] Verify a clean from-scratch build/run against the README quick start (produce a real artifact, don't trust the docs).

<sub>Workflow: https://github.com/rainfantry/release-blackops-skill</sub>
