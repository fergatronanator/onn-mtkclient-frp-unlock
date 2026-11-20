# onn MT6768 FRP removal and userdata wipe with mtkclient on Windows

Owner-authorized recovery writeup for clearing the Google/FRP lock on an onn MediaTek tablet and then wiping `userdata`. Everything below is from the successful September 2026 session; failed experiments, later Lenovo Tab M11 rescue work, and unrelated research are intentionally excluded.

> Only run this on devices you own or are explicitly authorized to service. Erasing `userdata` destroys all local data. Back up anything you need first.

## What was accomplished

| Item | Result |
|---|---|
| Toolchain | `mtkclient` V2.1.4 on Windows 11 with Python 3.12 |
| Transport | UsbDk 1.0.22 x64 + Windows MTK serial port / default COM driver |
| Device mode | `MediaTek PreLoader USB VCOM Port`, `USB\VID_0E8D&PID_0003` |
| SoC | MT6768/MT6769 (Helio P65/G85, `k68v1`), HW code `0x707` |
| Security state | SBC enabled; SLA/DAA/SWJTAG disabled; Kamakiri bypass succeeded |
| `frp` partition | Backed up (1 MiB) and erased |
| `userdata` partition | Erased (23,442,980,864 bytes) |
| `nvram` / `proinfo` | Deliberately untouched |

All identifiers from the original logs (ME_ID, SOC_ID, eMMC ID/CID, GPT UUIDs, serials, IMEIs, Wi-Fi details, and personal paths) are omitted or redacted here.

---

## 1. Prerequisites

From the upstream Windows instructions, plus what mattered in practice:

- Python >= 3.9 and Git.
  - This machine had Python 3.9 on `PATH` and Python 3.12 available through the `py` launcher.
  - Use `py -3.12` explicitly for every mtkclient command. Do not rely on plain `python`.
- [WinFsp](https://winfsp.dev/rel/) for FUSE support.
- [UsbDk 64-bit](https://github.com/daynix/UsbDk/releases).
- Normal MediaTek serial port driver, or the default Windows COM port driver, with no warning icon in Device Manager.
- Visual Studio Build Tools are only needed if Python wheels fail to build. They were not needed here; Python 3.12 had prebuilt wheels for all requirements.

Do not install Python from the Microsoft Store for this workflow.

---

## 2. Setup commands

Clone and install Python dependencies:

```powershell
git clone --depth 1 https://github.com/bkerler/mtkclient C:\mtkclient
py -3.12 -m pip install -r C:\mtkclient\requirements.txt
```

Download the two installers:

```powershell
Invoke-WebRequest `
  -Uri "https://github.com/winfsp/winfsp/releases/download/v2.1/winfsp-2.1.25156.msi" `
  -OutFile "C:\Temp\winfsp-2.1.25156.msi" -UseBasicParsing

Invoke-WebRequest `
  -Uri "https://github.com/daynix/UsbDk/releases/download/v1.00-22/UsbDk_1.0.22_x64.msi" `
  -OutFile "C:\Temp\UsbDk_1.0.22_x64.msi" -UseBasicParsing
```

Install them with elevation. Two UAC prompts were accepted:

```powershell
Start-Process msiexec `
  -ArgumentList '/i','C:\Temp\winfsp-2.1.25156.msi','/qn','/norestart' `
  -Verb RunAs -Wait -PassThru

Start-Process msiexec `
  -ArgumentList '/i','C:\Temp\UsbDk_1.0.22_x64.msi','/qn','/norestart' `
  -Verb RunAs -Wait -PassThru
```

Both installers returned exit code `0`.

Verify the install:

```powershell
py -3.12 C:\mtkclient\mtk.py --help

Test-Path "C:\Program Files (x86)\WinFsp"

Get-ChildItem "C:\Program Files\UsbDk Runtime Library" |
  Select-Object -ExpandProperty Name

Get-CimInstance Win32_SystemDriver |
  Where-Object { $_.Name -like "*UsbDk*" -or $_.DisplayName -like "*UsbDk*" } |
  Select-Object Name, State, StartMode
```

Observed:

- mtkclient banner: `MTK Flash/Exploit Client V2.1.4 (c) B.Kerler 2018-2026`
- WinFsp installed at `C:\Program Files (x86)\WinFsp`
- UsbDk installed at `C:\Program Files\UsbDk Runtime Library`
- `UsbDk` service: `Running`, `Manual`

A sanity check like `import fuse` may fail because `mfusepy` exposes the module as `mfusepy`, not `fuse`. mtkclient does not import `fuse` for this flow, so it is not a blocker.

---

## 3. Device detection and timing

Check that Windows sees the device in the correct mode:

```powershell
Get-PnpDevice -PresentOnly -ErrorAction SilentlyContinue |
  Where-Object { $_.InstanceId -like "USB\VID_0E8D*" } |
  Select-Object Status, Class, FriendlyName, InstanceId |
  Format-List
```

Successful onn detection looked like this, with the instance suffix redacted:

```text
Status       : OK
Class        : Ports
FriendlyName : MediaTek PreLoader USB VCOM Port (COMxx)
InstanceId   : USB\VID_0E8D&PID_0003\<redacted>
```

Confirm UsbDk sees the same USB device:

```powershell
& "C:\Program Files\UsbDk Runtime Library\UsbDkController.exe" -n
```

The relevant line was:

```text
ID: 0e8d:0003
```

A booted Android system may instead appear as `VID_0E8D&PID_2046` with a `MIDI function`. That is normal gadget mode and is not usable for this mtkclient flow. mtkclient needs the `PID_0003` preloader/BROM VCOM port.

### Timing is critical

Start mtkclient first, then plug in the device. If the device is already connected and stable when mtkclient starts, it may take the DA-resume path and immediately print:

```text
DaHandler - Please disconnect, start mtkclient and reconnect.
```

The reliable sequence was:

1. Unplug the device.
2. Start mtkclient and wait until it prints `Waiting for PreLoader VCOM`.
3. Plug the device in while holding the appropriate hardware button for BROM/preloader entry, or power it off and reconnect as prompted.
4. Let mtkclient finish.

---

## 4. Watcher script

A PowerShell watcher automated the unplug → start mtkclient → wait → plug-in sequence. This is the final version with the longer 30-minute completion window used for the large `userdata` erase. For the FRP pass, line 22 pointed to `C:\Temp\frp_script.txt` instead.

`C:\Temp\mtk_watch.ps1`:

```powershell
$ErrorActionPreference = 'Continue'
$log = "C:\Temp\mtk_watch.log"
Remove-Item $log -ErrorAction SilentlyContinue
function Log($msg) { Add-Content -Path $log -Value $msg }

# Phase 1: wait for device to be unplugged
Log "[1] Waiting for device to be UNPLUGGED (up to 5 min)..."
$deadline = (Get-Date).AddMinutes(5)
$unplugged = $false
while ((Get-Date) -lt $deadline) {
    $dev = Get-PnpDevice -PresentOnly -ErrorAction SilentlyContinue |
        Where-Object { $_.InstanceId -like "USB\VID_0E8D*" }
    if (-not $dev) { $unplugged = $true; break }
    Start-Sleep -Seconds 2
}
if (-not $unplugged) { Log "[1] TIMEOUT waiting for unplug"; exit 1 }
Log "[1] Device unplugged."

# Phase 2: start mtk.py from the mtkclient directory
Log "[2] Starting mtk.py..."
$outFile = "C:\Temp\mtk_run.log"
Remove-Item $outFile, "$outFile.err" -ErrorAction SilentlyContinue
$proc = Start-Process py `
    -ArgumentList '-3.12','C:\mtkclient\mtk.py','script','C:\Temp\userdata_script.txt' `
    -WorkingDirectory 'C:\mtkclient' `
    -RedirectStandardOutput $outFile `
    -RedirectStandardError "$outFile.err" `
    -PassThru -NoNewWindow
Log "[2] mtk.py started (PID $($proc.Id))."

# Phase 3: wait for mtk.py to reach waiting state
$deadline = (Get-Date).AddMinutes(2)
$waiting = $false
while ((Get-Date) -lt $deadline) {
    if (Test-Path $outFile) {
        $content = Get-Content $outFile -Raw -ErrorAction SilentlyContinue
        if ($content -match "Waiting for PreLoader VCOM") { $waiting = $true; break }
    }
    if ($proc.HasExited) { Log "[3] mtk.py exited early."; exit 1 }
    Start-Sleep -Seconds 1
}
if (-not $waiting) { Log "[3] mtk.py did not reach waiting state."; exit 1 }
Log "[3] mtk.py is WAITING for device. PLUG IN NOW."

# Phase 4: wait for mtk.py to finish
$deadline = (Get-Date).AddMinutes(30)
while (-not $proc.HasExited -and (Get-Date) -lt $deadline) { Start-Sleep -Seconds 2 }
if (-not $proc.HasExited) {
    Log "[4] TIMEOUT. Killing mtk.py."
    Stop-Process -Id $proc.Id -Force
    Start-Sleep -Seconds 2
}
Log "[4] mtk.py finished (exit $($proc.ExitCode))."
Log "[4] --- stdout ---"
Get-Content $outFile -ErrorAction SilentlyContinue | ForEach-Object { Log $_ }
Log "[4] --- stderr ---"
Get-Content "$outFile.err" -ErrorAction SilentlyContinue | ForEach-Object { Log $_ }
Log "[4] DONE"
```

Launch it hidden:

```powershell
Start-Process powershell `
  -ArgumentList '-NoProfile','-ExecutionPolicy','Bypass','-File','C:\Temp\mtk_watch.ps1' `
  -WindowStyle Hidden
```

---

## 5. Read-only diagnostic pass

Before modifying anything, a read-only handshake and GPT dump was run from `C:\mtkclient`:

```powershell
cd C:\mtkclient
py -3.12 C:\mtkclient\mtk.py --socid printgpt
```

Important: in this mtkclient version, `--socid` is a flag, not a standalone command. Running only `mtk.py --socid` fails with:

```text
the following arguments are required: command
```

Pair it with a command such as `printgpt`.

Relevant sanitized output:

```text
Preloader - CPU:            MT6768/MT6769(Helio P65/G85 k68v1)
Preloader - HW code:        0x707
Preloader - Target config:  0xe1
Preloader - SBC enabled:    True
Preloader - SLA enabled:    False
Preloader - DAA enabled:    False
Preloader - SWJTAG enabled: False
Preloader - ME_ID:          <redacted>
Preloader - SOC_ID:         <redacted>

DaHandler - Device is protected.
DaHandler - Device is in BROM-Mode. Bypassing security.
PLTools - Loading payload from mt6768_payload.bin, 0x264 bytes
Exploitation - Kamakiri Run
Exploitation - Done sending payload...
PLTools - Successfully sent payload: ...\mtkclient\payloads\mt6768_payload.bin

DAXFlash - Successfully uploaded stage 1, jumping ..
DAXFlash - Successfully received DA sync
DAXFlash - DRAM setup passed.
DAXFlash - Successfully uploaded stage 2
DAXFlash - DA SLA is disabled
```

The GPT confirmed a dedicated `frp` partition:

```text
frp:       Offset 0x0000000001508000, Length 0x0000000000100000
userdata:  Offset 0x00000001cf000000, Length 0x00000005754f8000
```

That is a 1 MiB `frp` partition and an approximately 23.4 GB `userdata` partition.

---

## 6. FRP backup and erase

`C:\Temp\frp_script.txt` contained:

```text
r frp frp_backup.bin
e frp
```

This reads the FRP partition to a backup file, then erases it in the same mtkclient connection.

The watcher effectively ran:

```powershell
py -3.12 C:\mtkclient\mtk.py script C:\Temp\frp_script.txt
```

from `C:\mtkclient`.

Result:

```text
Successsfully formatted addr 0x1508000 with length 1048576
```

The misspelling is from mtkclient's own output. The backup was verified:

```powershell
Get-Item C:\mtkclient\frp_backup.bin | Select-Object Name, Length, LastWriteTime
```

```text
Name            Length
----            ------
frp_backup.bin  1048576
```

Do not publish or commit that backup; it contains device-specific data.

### Why `script` instead of `multi`

The direct one-liner is:

```powershell
py -3.12 mtk.py multi "r frp frp_backup.bin; e frp"
```

but when passed through PowerShell `Start-Process -ArgumentList`, the quoted argument was flattened. mtkclient saw the semicolon-separated string as extra arguments and failed with:

```text
mtk.py: error: unrecognized arguments: frp frp_backup.bin; e frp
```

The script-file form avoids all quoting problems and was the command that actually succeeded.

---

## 7. Userdata wipe

After explicit approval, `userdata` was wiped as a factory reset.

`C:\Temp\userdata_script.txt` contained:

```text
e userdata
```

The watcher was pointed at that script and ran:

```powershell
py -3.12 C:\mtkclient\mtk.py script C:\Temp\userdata_script.txt
```

from `C:\mtkclient`. The completion timeout was increased to 30 minutes because the partition is about 23 GB.

Result:

```text
Successfully formatted addr 0x1cf000000 with length 23442980864
```

That is 23,442,980,864 bytes, or about 21.8 GiB.

---

## 8. Verification and final state

After the runs:

| Partition | Status |
|---|---|
| `frp` | Erased; 1,048,576-byte backup retained locally |
| `userdata` | Erased; 23,442,980,864 bytes formatted |
| `nvram` | Untouched |
| `proinfo` | Untouched |

The device should then be unplugged, powered on, and boot directly to the Android setup wizard without Google account verification.

`nvram` and `proinfo` were deliberately not erased. On MediaTek devices they can contain IMEI/radio-related data, and wiping them can break connectivity.

---

## 9. Troubleshooting notes

| Symptom | Cause | Fix |
|---|---|---|
| `the following arguments are required: command` | `--socid` used by itself | Use `py -3.12 mtk.py --socid printgpt` |
| `DaHandler - Please disconnect, start mtkclient and reconnect.` | Device was already connected when mtkclient started, so it tried to resume a DA session | Unplug first; start mtkclient; wait for `Waiting for PreLoader VCOM`; then plug in |
| `Status: Handshake failed, retrying...` | Device not in the expected BROM/preloader state or reconnect timing was off | Power off, hold volume up/down while connecting, and retry after mtkclient is waiting |
| `unrecognized arguments: frp frp_backup.bin; e frp` | PowerShell `Start-Process` mangled the quoted `multi` argument | Use `script` with a text file containing one command per line |
| `PermissionError` writing `hwparam.json` | mtkclient launched from the wrong working directory | Run from `C:\mtkclient` |
| `ModuleNotFoundError: No module named 'fuse'` | `mfusepy` installs as `mfusepy`, not `fuse` | Ignore for this flow; the FUSE mount command is not used |
| Device shows `VID_0E8D&PID_2046` / `MIDI function` | Android is booted normally | Re-enter preloader/BROM mode and confirm `PID_0003` |
| `Auth file is required. Use --auth option.` | Seen on a different, later device with SLA/DAA enabled | Out of scope for this onn procedure; do not apply this writeup blindly to that device |

---

## 10. Artifacts referenced

These paths were used locally and should not be treated as portable:

- `C:\mtkclient` — cloned mtkclient repository and working directory.
- `C:\Temp\mtk_watch.ps1` — unplug/start/wait watcher.
- `C:\Temp\mtk_watch.log` — watcher log.
- `C:\Temp\mtk_run.log` — redirected mtkclient stdout.
- `C:\Temp\frp_script.txt` — `r frp frp_backup.bin` and `e frp`.
- `C:\Temp\userdata_script.txt` — `e userdata`.
- `C:\mtkclient\frp_backup.bin` — 1 MiB FRP backup; sensitive, do not publish.

## Scope note

This document records the confirmed onn/MT6768 FRP wipe and `userdata` erase only. Later Lenovo Tab M11 attempts, SP Flash Tool rescue work, ADB/debloat scripts, and unrelated device research are not part of this procedure.
