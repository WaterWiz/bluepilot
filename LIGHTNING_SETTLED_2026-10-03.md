# Comma Four (Lightning) - Settled Setup Restore, 2026-10-03

What this restores: BluePilot bp-7.0 (e1d051d) on the comma four with
- comma openpilot v0.11.2 big-model runtime (Lebowski 880M) on the chestnut eGPU
- speed-scaled steering look-ahead (BP curvature mode)
- Ford EV regen/brake handoff fix + longitudinalActuatorDelay 0.25
- AGNOS 19.7 kernel + abl on an AGNOS 18.5 system (USB3 fixes for chestnut)

Results and test history: `/data/Lightning Fork/Lateral Tuning Log.md`. Git: WaterWiz/bluepilot branch `lightning-settled-2026-10-03`.

## When to use
After a BluePilot update, reinstall, or reset wiped the local changes (symptoms: dock not used / UsbGpuPresent=0,
steering ping-pong back, speed surging).

## 1. Kernel (only if AGNOS was reflashed - `uname -v` shows Jul 8 2026)
Images: `/data/Lightning Fork/agnos-19.7/` (boot-19.7.img, abl-19.7.img). Copy to the comma, then on the comma:
    sudo dd if=boot-19.7.img of=/dev/disk/by-partlabel/boot_$(slot) bs=1M conv=fsync
    sudo dd if=abl-19.7.img  of=/dev/disk/by-partlabel/abl_$(slot)  bs=1M conv=fsync
Slot = `cat /proc/cmdline | grep -o slot_suffix=_[ab]`. Verify sha256 (first N bytes) against agnos_19.7.json before reboot.

## 2. openpilot changes (BluePilot must still be bp-7.0 e1d051d; otherwise rebase the git branch instead)
    rsync -a files/ comma@<ip>:/data/openpilot/          # the 11 changed files
    tar xzf tinygrad_repo-v0.11.2.tar.gz -C /tmp && rsync -a --delete --exclude=.git /tmp/tinygrad_repo/ comma@<ip>:/data/openpilot/tinygrad_repo/
Models (1.8 GB, not in this folder): `/data/Lightning Fork/op-chestnut/openpilot/selfdrive/modeld/models/` ->
`/data/openpilot/selfdrive/modeld/models/` (delete the old *.onnx there first; write tg_input_devices.json with
"selfdrive.modeld.modeld" keys - see project notes).

## 3. Settings (BluePilot UI or Params)
custom_profile_curv=1, pc_blend_ratio_low_C_UI_curv=0.4, pc_blend_ratio_high_C_UI_curv=0.4, FordPrefLateralControl=0,
AlphaLongitudinalEnabled=1 (reboot), then ExperimentalMode=1, SpeedLimitMode=2. Full snapshot: params-snapshot.txt.

## 4. Reboot parked, confirm
`big model active: True` in swaglog, UsbGpuPresent=1, CarParams longitudinalActuatorDelay=0.25.
