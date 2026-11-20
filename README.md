# onn MT6768 FRP removal and userdata wipe with mtkclient on Windows

Owner-authorized recovery writeup for clearing the Google/FRP lock on an onn MediaTek tablet and then wiping `userdata`. This records the successful September 2026 session. Failed experiments, later work on other devices, and unrelated research are intentionally excluded.

> **Only do this on a device you own or are explicitly authorized to service.** Erasing `userdata` destroys all local data. Back up anything you need first.

---

## The short version (the exact order, in plain English)

If you only want the checklist, follow these ten steps. The rest of the document explains each one.

1. Install Python 3.12, Git, WinFsp, and UsbDk. Reboot if any installer asks.
2. Make Windows accept the **unsigned/self-signed MediaTek VCOM driver** (Section 4): turn on Test Mode, reboot, create and trust a test certificate, sign the driver's catalog, install the INF with `pnputil`, then verify a COM port appears.
3. Confirm UsbDk can see the tablet in preloader mode (`ID: 0e8d:0003`), not booted Android (`PID_2046`).
4. **Unplug the tablet.** Do not have it connected when you start mtkclient.
5. Start mtkclient from `C:\mtkclient` and wait until it prints `Waiting for PreLoader VCOM`.
6. With the tablet **fully powered off**, **hold Volume Up + Volume Down together** and plug in the USB cable while still holding them. Keep holding about 5–10 seconds. This puts it into **MediaTek BROM / PreLoader VCOM mode** — the mode mtkclient needs. Windows calls the port `MediaTek PreLoader USB VCOM Port (COMxx)`, hardware ID `USB\VID_0E8D&PID_0003`.
7. Run the FRP backup-and-erase script (`r frp frp_backup.bin`, then `e frp`). Keep the backup file local; never publish it.
8. Run the userdata script (`e userdata`). This takes several minutes.
9. Unplug, power on, and finish the Android setup wizard without the Google check.
10. Turn Windows Test Mode back off and reboot (Section 4.8).

### Words you will keep seeing

| Term | Plain-language meaning |
|---|---|
| **BROM** | Boot ROM: the tiny built-in program in MediaTek chips that runs before Android. mtkclient talks to this. |
| **Preloader / VCOM** | The temporary USB serial port the tablet exposes while in BROM. Windows names it `MediaTek PreLoader USB VCOM Port (COMxx)`. |
| **COMxx** | Any COM port number, for example `COM7`. The number changes between cables and USB ports; it does not matter which one you get. |
| **FRP** | Factory Reset Protection — Google's anti-theft check after a factory reset. |
| **DA** | Download Agent: the file mtkclient uploads into the tablet to read/write flash. |
| **GPT** | The partition table — the list of named areas such as `frp` and `userdata`. |
| **UsbDk** | The Windows driver library mtkclient actually uses to talk to the tablet at low level. |

---

## 1. What was accomplished

| Item | Result |
|---|---|
| Toolchain | `mtkclient` V2.1.4 on Windows 11 with Python 3.12 |
| Transport | UsbDk 1.0.22 x64 + a self-signed MediaTek PreLoader VCOM (COM port) driver |
| Device mode | BROM / `MediaTek PreLoader USB VCOM Port`, `USB\VID_0E8D&PID_0003` |
| SoC | MT6768/MT6769 (Helio P65/G85, `k68v1`), HW code `0x707` |
| Security state | SBC enabled; SLA/DAA/SWJTAG disabled; Kamakiri bypass succeeded |
| `frp` partition | Backed up (1 MiB) and erased |
| `userdata` partition | Erased (23,442,980,864 bytes) |
| `nvram` / `proinfo` | Deliberately untouched |

All identifiers from the original logs (ME_ID, SOC_ID, eMMC ID/CID, GPT UUIDs, serials, IMEIs, Wi-Fi details, personal paths, and certificate thumbprints) are omitted or redacted here.

---

## 2. Prerequisites

Install these before touching the tablet:

- **Python 3.12** and **Git**. This machine also had Python 3.9 on `PATH`, so every mtkclient command must use `py -3.12` explicitly. Do not install Python from the Microsoft Store for this workflow.
- **WinFsp** — FUSE support, needed by some mtkclient features.
- **UsbDk 64-bit** — the low-level USB driver mtkclient uses on Windows.
- **A MediaTek preloader COM-port driver** — covered in Section 4. Windows will refuse to install a normal unsigned INF unless Test Mode is enabled, which is why that section exists.
- **Visual Studio Build Tools** — only needed if Python wheels fail to build. They were not needed here; Python 3.12 had prebuilt wheels for every requirement.

Optional Windows Kits tools (`makecat.exe`, `signtool.exe`) are needed only if you build the self-signed driver catalog yourself, as in Section 4.

---

## 3. Install the toolchain

### 3.1 Clone mtkclient and install Python packages

Open **PowerShell** (normal is fine) and run:

```powershell
git clone --depth 1 https://github.com/bkerler/mtkclient C:\mtkclient
py -3.12 -m pip install -r C:\mtkclient\requirements.txt
```

### 3.2 Download the two installers

```powershell
Invoke-WebRequest `
  -Uri "https://github.com/winfsp/winfsp/releases/download/v2.1/winfsp-2.1.25156.msi" `
  -OutFile "C:\Temp\winfsp-2.1.25156.msi" -UseBasicParsing

Invoke-WebRequest `
  -Uri "https://github.com/daynix/UsbDk/releases/download/v1.00-22/UsbDk_1.0.22_x64.msi" `
  -OutFile "C:\Temp\UsbDk_1.0.22_x64.msi" -UseBasicParsing
```

### 3.3 Install them as Administrator

**Two UAC prompts will appear. Click "Yes" on both.**

```powershell
Start-Process msiexec `
  -ArgumentList '/i','C:\Temp\winfsp-2.1.25156.msi','/qn','/norestart' `
  -Verb RunAs -Wait -PassThru

Start-Process msiexec `
  -ArgumentList '/i','C:\Temp\UsbDk_1.0.22_x64.msi','/qn','/norestart' `
  -Verb RunAs -Wait -PassThru
```

Both returned exit code `0`.

### 3.4 Verify

```powershell
py -3.12 C:\mtkclient\mtk.py --help

Test-Path "C:\Program Files (x86)\WinFsp"

Get-ChildItem "C:\Program Files\UsbDk Runtime Library" |
  Select-Object -ExpandProperty Name

Get-CimInstance Win32_SystemDriver |
  Where-Object { $_.Name -like "*UsbDk*" -or $_.DisplayName -like "*UsbDk*" } |
  Select-Object Name, State, StartMode
```

You should see:

- mtkclient banner: `MTK Flash/Exploit Client V2.1.4 (c) B.Kerler 2018-2026`
- WinFsp installed under `C:\Program Files (x86)\WinFsp`
- UsbDk installed under `C:\Program Files\UsbDk Runtime Library`
- `UsbDk` service: `Running`, `Manual`

A sanity check like `import fuse` can fail because `mfusepy` exposes the module as `mfusepy`, not `fuse`. mtkclient does not import `fuse` for this flow, so it is not a blocker.

---

## 4. The unsigned driver, step by step

This is the part that trips people up. mtkclient needs the tablet to show up as a MediaTek **preloader COM port**, but the small custom INF that creates this port is not signed by Microsoft. Windows normally blocks it with an error like:

```text
The third-party INF does not contain digital signature information.
```

The fix used on this machine was: **enable Windows Test Mode, create and trust a self-signed test certificate, sign a catalog for the INF, then install the INF.** Do all of this in an **Administrator PowerShell** (right-click Start → Terminal (Admin) / PowerShell (Admin), then click "Yes" at the UAC prompt).

> **Security note:** Test Mode (test signing) relaxes Windows driver signature checks. Do this only on a PC you control, and turn it back off in Section 4.8 when you are finished.

### 4.1 Turn on Test Mode and reboot

```powershell
bcdedit /set testsigning on
```

Then **restart Windows**. After the reboot you will see a **Test Mode** watermark in the bottom-right corner of the desktop. That watermark means it worked; it is expected.

Confirm before continuing:

```powershell
bcdedit /enum '{current}' | Select-String 'testsigning'
```

You want to see `testsigning Yes`.

### 4.2 Put the driver files in a folder

Make a folder such as `C:\Temp\mtk-driver` containing your MediaTek preloader INF (here called `mtk_preloader_vcom.inf`). The INF must contain a `CatalogFile=mtk_preloader_vcom.cat` line under its `[Version]` section. If that line is missing, add it before continuing.

### 4.3 Create and trust a self-signed test certificate

```powershell
$drv = "C:\Temp\mtk-driver"
$certName = "MTK Driver Test Sign"

# Create the certificate
$cert = New-SelfSignedCertificate -Type CodeSigningCert `
  -Subject "CN=$certName" `
  -CertStoreLocation Cert:\LocalMachine\My `
  -KeyExportPolicy Exportable `
  -NotAfter (Get-Date).AddYears(5)

# Export it, then trust it
Export-Certificate -Cert $cert -FilePath "$drv\mtk-test.cer" -Force | Out-Null
Import-Certificate -FilePath "$drv\mtk-test.cer" `
  -CertStoreLocation Cert:\LocalMachine\TrustedPublisher | Out-Null
Import-Certificate -FilePath "$drv\mtk-test.cer" `
  -CertStoreLocation Cert:\LocalMachine\Root | Out-Null
```

This creates one test signing certificate and adds it to the machine's trusted publishers. A thumbprint is printed by `New-SelfSignedCertificate`; you do not need to write it down.

### 4.4 Build and sign the driver catalog

`signtool.exe` and `makecat.exe` come with the Windows SDK / Windows Kits. The exact version folder varies by machine; find yours with:

```powershell
Get-ChildItem "C:\Program Files (x86)\Windows Kits\10\bin" -Recurse `
  -Include makecat.exe,signtool.exe -ErrorAction SilentlyContinue |
  Select-Object -ExpandProperty FullName
```

Then, substituting your own Windows Kits version path:

```powershell
$drv = "C:\Temp\mtk-driver"
$certName = "MTK Driver Test Sign"
$makecat = "C:\Program Files (x86)\Windows Kits\10\bin\<version>\x64\makecat.exe"
$signtool = "C:\Program Files (x86)\Windows Kits\10\bin\<version>\x64\signtool.exe"

# Describe the catalog in a .cdf file
@'
[CatalogHeader]
Name=mtk_preloader_vcom.cat
ResultDir=.
PublicVersion=0x0000001
Encoding=0
[CatalogFiles]
<hash>mtk_preloader_vcom.inf=mtk_preloader_vcom.inf
'@ | Set-Content "$drv\mtk_preloader_vcom.cdf" -Encoding ASCII

Push-Location $drv
& $makecat -v mtk_preloader_vcom.cdf
& $signtool sign /fd SHA256 /sm /s My /n $certName /v "$drv\mtk_preloader_vcom.cat"
& $signtool verify /pa "$drv\mtk_preloader_vcom.cat"
Pop-Location
```

You should see `Succeeded` from `makecat`, `Successfully signed` from `signtool`, and `Successfully verified` from the verify command.

### 4.5 Install the INF

Still in the Administrator PowerShell:

```powershell
pnputil /add-driver "C:\Temp\mtk-driver\mtk_preloader_vcom.inf" /install
```

Expected output:

```text
Adding driver package:  mtk_preloader_vcom.inf
Driver package added successfully.
Published Name:         oemXX.inf
Driver package installed on device: USB\VID_0E8D&PID_2000\<redacted>
Driver package installed on device: USB\VID_0E8D&PID_0003\<redacted>
```

The device-specific instance suffixes are redacted here. What matters is that the package installs successfully and covers both preloader IDs (`PID_2000` and `PID_0003`).

### 4.6 Verify the COM port

Put the tablet into BROM/preloader mode (Section 5) and run:

```powershell
Get-PnpDevice -PresentOnly -ErrorAction SilentlyContinue |
  Where-Object { $_.InstanceId -like "USB\VID_0E8D*" } |
  Select-Object Status, Class, FriendlyName, InstanceId |
  Format-List
```

Successful detection looks like this:

```text
Status       : OK
Class        : Ports
FriendlyName : MediaTek PreLoader USB VCOM Port (COMxx)
InstanceId   : USB\VID_0E8D&PID_0003\<redacted>
```

`Status : OK` with no yellow warning icon is the goal. If you see `PID_2046` with a `MIDI function`, Android booted normally instead — go back to Section 5.

Also confirm UsbDk sees the same device:

```powershell
& "C:\Program Files\UsbDk Runtime Library\UsbDkController.exe" -n
```

The relevant line is `ID: 0e8d:0003`.

### 4.7 Optional WinUSB variant (tried, then reverted)

On this machine a registry-based experiment also forced the same MediaTek USB entries onto WinUSB instead of the serial driver. It was **reverted about six minutes later** and was **not needed for the successful run**. Include it only as a fallback if mtkclient cannot see the device via the normal VCOM/UsbDk path.

The WinUSB values are:

| Setting | Serial (normal) | WinUSB (experimental) |
|---|---|---|
| `Service` | `usbser` | `WinUSB` |
| `ClassGUID` | `{4d36e978-e325-11ce-bfc1-08002be10318}` | `{146bf252-9f25-4209-a6dd-c45a1180abc4}` |

Set it, replacing `<device-instance-id>` with the actual registry key for your device (found under `HKLM:\SYSTEM\CurrentControlSet\Enum\USB\VID_0E8D&PID_0003`):

```powershell
$target = "HKLM:\SYSTEM\CurrentControlSet\Enum\USB\VID_0E8D&PID_0003\<device-instance-id>"
Set-ItemProperty $target -Name Service -Value 'WinUSB'
Set-ItemProperty $target -Name ClassGUID -Value '{146bf252-9f25-4209-a6dd-c45a1180abc4}'
Set-ItemProperty $target -Name Driver -Value '{146bf252-9f25-4209-a6dd-c45a1180abc4}\0000'
```

Revert it by restoring the serial values and removing the `Driver` override:

```powershell
Set-ItemProperty $target -Name Service -Value 'usbser'
Set-ItemProperty $target -Name ClassGUID -Value '{4d36e978-e325-11ce-bfc1-08002be10318}'
Remove-ItemProperty $target -Name Driver -ErrorAction SilentlyContinue
```

Do not copy hardware-specific instance suffixes from anyone else's guide; yours will be different.

### 4.8 Turn Test Mode back off (cleanup)

After all flashing is done:

```powershell
bcdedit /set testsigning off
```

Reboot. The Test Mode watermark should disappear.

---

## 5. The exact mode and the exact button combo

### What the mode is called

People use several names for the same state. For this tablet they all mean: **the tablet is sitting in MediaTek's boot ROM, presenting a preloader USB serial port.**

| Name you may see | Where you see it |
|---|---|
| **BROM mode** | What mtkclient calls it when it says `Device is in BROM-Mode. Bypassing security.` |
| **Preloader mode / PreLoader VCOM** | What Windows calls the COM port: `MediaTek PreLoader USB VCOM Port (COMxx)` |
| **`USB\VID_0E8D&PID_0003`** | The hardware ID in Device Manager / UsbDk (`ID: 0e8d:0003`) |
| **`PID_2000`** | The other preloader ID sometimes seen before the port switches |

This is **not** normal Android mode. A booted tablet shows `VID_0E8D&PID_2046` with a `MIDI function`, and mtkclient cannot use it.

### Exactly which keys to hold, and when

mtkclient prints this hint itself:

```text
Power off the phone before connecting.
For brom mode, press and hold vol up, vol dwn, or all hw buttons and connect usb.
For preloader mode, don't press any hw button and connect usb.
```

For the successful onn run, the instruction was the volume-button (BROM) variant. Do it like this:

1. **Unplug** the USB cable.
2. Make sure the tablet is **completely powered off** — hold the power button until it shuts down, and wait for the screen to go black.
3. Start mtkclient first and wait for `Waiting for PreLoader VCOM` (see Section 6).
4. **Hold Volume Up and Volume Down at the same time.** Do **not** hold the power button; plugging in the USB cable powers the tablet in this mode.
5. While still holding both buttons, **plug the USB cable into the tablet**.
6. Keep holding for about **5–10 seconds**, then let go. mtkclient should move past `Waiting for PreLoader VCOM` to a handshake.
7. Confirm Windows shows `MediaTek PreLoader USB VCOM Port (COMxx)` / `PID_0003` and UsbDk shows `0e8d:0003`.

If it boots into Android instead (`PID_2046` / MIDI function), or you see a COM port named `MediaTek USB Port`, unplug, power off completely, and repeat from step 4. If holding both buttons does not work, mtkclient's hint says Volume Up alone, Volume Down alone, or all hardware buttons are also accepted — try one of those variants.

### Why the order matters

If the tablet is already connected when mtkclient starts, it can try to resume an old download-agent session and stop with:

```text
DaHandler - Please disconnect, start mtkclient and reconnect.
```

So the rule is always: **unplug → start mtkclient → wait for `Waiting for PreLoader VCOM` → plug in with the volume buttons held.**

---

## 6. Watcher script (unplug → start → wait → plug in)

A PowerShell watcher automated the timing above. This is the final version with the 30-minute completion window used for the large `userdata` erase. For the FRP pass, line 22 pointed to the FRP script instead.

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

What the script does, in plain English:

1. Waits until you unplug the tablet.
2. Starts `mtk.py` from the right directory (`C:\mtkclient`).
3. Watches its output until it says `Waiting for PreLoader VCOM`, then tells you to plug in. **That is your cue to do the Volume Up + Volume Down plug-in from Section 5.**
4. Waits for the command to finish and writes everything to a log.

Launch it hidden:

```powershell
Start-Process powershell `
  -ArgumentList '-NoProfile','-ExecutionPolicy','Bypass','-File','C:\Temp\mtk_watch.ps1' `
  -WindowStyle Hidden
```

Then watch `C:\Temp\mtk_watch.log` for `PLUG IN NOW`.

---

## 7. Read-only diagnostic pass

Before modifying anything, a read-only handshake and GPT dump was run from `C:\mtkclient`:

```powershell
cd C:\mtkclient
py -3.12 C:\mtkclient\mtk.py --socid printgpt
```

Important: `--socid` is a flag, not a standalone command. Running only `mtk.py --socid` fails with:

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

The GPT confirmed the two partitions of interest:

```text
frp:       Offset 0x0000000001508000, Length 0x0000000000100000
userdata:  Offset 0x00000001cf000000, Length 0x00000005754f8000
```

That is a 1 MiB `frp` partition and an approximately 23.4 GB `userdata` partition.

---

## 8. FRP backup and erase

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

**Do not publish or commit that backup.** It contains device-specific data.

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

## 9. Userdata wipe

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

## 10. Verification and final state

| Partition | Status |
|---|---|
| `frp` | Erased; 1,048,576-byte backup retained locally, not published |
| `userdata` | Erased; 23,442,980,864 bytes formatted |
| `nvram` | Untouched |
| `proinfo` | Untouched |

Unplug the tablet, power it on, and it should boot directly to the Android setup wizard without Google account verification.

`nvram` and `proinfo` were deliberately not erased. On MediaTek devices they can contain IMEI/radio-related data, and wiping them can break connectivity. Leave them alone.

---

## 11. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `The third-party INF does not contain digital signature information.` | Test Mode is off, or the catalog was not created/signed | Enable test signing, reboot, create and sign the `.cat` (Sections 4.1, 4.4), then retry `pnputil` |
| `testsigning Yes` but the COM port still has a warning icon | Driver package not installed for this USB ID | Re-run `pnputil /add-driver ... /install` and check `USB\VID_0E8D&PID_0003` is covered |
| `the following arguments are required: command` | `--socid` used by itself | Use `py -3.12 mtk.py --socid printgpt` |
| `DaHandler - Please disconnect, start mtkclient and reconnect.` | Device already connected when mtkclient started, so it tried to resume a DA session | Unplug; start mtkclient; wait for `Waiting for PreLoader VCOM`; then plug in with Volume Up + Volume Down held |
| `Status: Handshake failed, retrying...` | Device not in BROM/preloader state or reconnect timing was off | Power off fully, hold Volume Up + Volume Down while connecting, and retry after mtkclient is waiting |
| Device shows `VID_0E8D&PID_2046` / `MIDI function` | Android booted normally | Re-enter BROM mode with the volume buttons and confirm `PID_0003` |
| `unrecognized arguments: frp frp_backup.bin; e frp` | PowerShell `Start-Process` mangled the quoted `multi` argument | Use `script` with a text file containing one command per line |
| `PermissionError` writing `hwparam.json` | mtkclient launched from the wrong working directory | Run from `C:\mtkclient` |
| `ModuleNotFoundError: No module named 'fuse'` | `mfusepy` installs as `mfusepy`, not `fuse` | Ignore for this flow; the FUSE mount command is not used |
| `Auth file is required. Use --auth option.` | Seen on a different, later device with SLA/DAA enabled | Out of scope for this onn procedure; do not apply this writeup blindly to that device |

---

## 12. Artifacts referenced

These paths were used locally and should not be treated as portable:

- `C:\mtkclient` — cloned mtkclient repository and working directory.
- `C:\Temp\mtk-driver` — folder holding the MediaTek preloader INF and its signed catalog.
- `C:\Temp\mtk_watch.ps1` — unplug/start/wait watcher.
- `C:\Temp\mtk_watch.log` — watcher log.
- `C:\Temp\mtk_run.log` — redirected mtkclient stdout.
- `C:\Temp\frp_script.txt` — `r frp frp_backup.bin` and `e frp`.
- `C:\Temp\userdata_script.txt` — `e userdata`.
- `C:\mtkclient\frp_backup.bin` — 1 MiB FRP backup; sensitive, do not publish.

## Scope note

This document records the confirmed onn/MT6768 FRP wipe and `userdata` erase only. Other devices, tools, and later experiments are not part of this procedure.
