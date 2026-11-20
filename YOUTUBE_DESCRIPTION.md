# YouTube description — onn MT6768 FRP removal + userdata wipe with mtkclient

Suggested title: **onn MT6768 FRP Removal + Userdata Wipe with mtkclient (Owner-Authorized)**

Copy/paste the text below into the YouTube Description field. Adjust the chapter timestamps to match your video.

```text
Owner-authorized recovery demo: clearing the Google/FRP check on an onn MT6768 tablet with mtkclient on Windows, then wiping userdata.

⚠️ Only use this on a device you own or are explicitly authorized to service. Erasing userdata permanently deletes local data. This is not for stolen devices.

This is not really about the tablet's model year. The BROM exploit used here depends on the MediaTek chip, not whether the tablet was sold in 2024, 2025, or 2026. This unit reports MT6768/MT6769 (k68v1), HW code 0x707, with SLA/DAA disabled. Check your device first; other chips may need a different flow.

Brief steps:
1. Install Python 3.12, Git, WinFsp, and UsbDk on Windows.
2. Enable Windows Test Mode, sign/install the MediaTek PreLoader VCOM driver, reboot, and confirm UsbDk sees ID 0e8d:0003.
3. Unplug the tablet. Start mtkclient and wait for "Waiting for PreLoader VCOM".
4. Power the tablet off, hold Volume Up + Volume Down, plug in USB while holding, and keep holding 5-10 seconds. Windows should show "MediaTek PreLoader USB VCOM Port (COMxx)" / USB\VID_0E8D&PID_0003.
5. Run the FRP backup-and-erase script: r frp frp_backup.bin, then e frp. Keep the backup local and private; do not upload it.
6. Run the userdata script: e userdata. Wait several minutes for the wipe to finish.
7. Unplug, power on, and complete Android setup without the Google check.
8. Turn Windows Test Mode back off and reboot.

Full written guide with driver details, troubleshooting, and sanitized logs:
https://github.com/fergatronanator/onn-mtkclient-frp-unlock

mtkclient: https://github.com/bkerler/mtkclient

Chapters (adjust timestamps to match your video):
00:00 What this does / safety
00:30 Which chips are affected
01:00 Windows tools and drivers
02:00 Unsigned driver / Test Mode
03:00 Entering MediaTek BROM mode
04:00 FRP backup and erase
05:00 Userdata wipe
06:00 Verification and cleanup

#mtkclient #MediaTek #FRP #Android #onn
```
