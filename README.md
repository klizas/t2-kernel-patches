# t2-kernel-patches

Kernel patches for my MacBookPro16,1 on CachyOS. Targets T2 Macs with AMD dGPU, BCM4364 WiFi, Touch Bar. May not apply or work elsewhere.

## `0001-brcmfmac-suspend-fix.patch`

Suspend hangs when Broadcom firmware stops responding before the PCI driver finishes suspending.

- `brcmf_pcie_pm_enter_D3` times out waiting 2s for a D3_INFORM ACK and aborts suspend with `-EIO`. Patch lets suspend proceed; on resume, `intmask == 0` triggers the existing detach + re-probe path.
- `brcmf_msgbuf_delete_flowring` waits for `outstanding_tx` to drain before checking bus state, burning 5–10ms × 10 retries per flowring. Patch checks bus state before and during the wait.

## `0001-touchbar-suspend-resume.patch`

The in-kernel Touch Bar driver `hid-appletb-kbd` (used without userspace `tiny-dfr`) comes back blank after resume.

- Mode restore runs on a 1.5s delayed workqueue, not inline, so HID commands wait for USB to come back.
- Mode-off at suspend is dropped — the device is losing power anyway, and the write raced USB teardown.
- Probe returns `-EPROBE_DEFER` when the backlight device isn't ready, removing the optional-backlight case.

## `0001-amdgpu-mclk-override.patch`

Adds `amdgpu.dc_dram_clock_change_latency_ns` to override the DRAM latency the AMD DML uses to decide whether mclk switching is safe.

Navi 14 (Radeon Pro 5300M/5500M in the 2019 16" MBP) hardcodes 404µs in `dcn2_0_nv14_soc` (`dcn20_fpu.c`). The DML can't hide that in the 3072×1920 panel's vblank, marks switching `dm_dram_clock_change_unsupported` (see `MinActiveDRAMClockChangeMargin` in `display_mode_vba_20v2.c`), and pins the dGPU at the top pstate whenever the panel is lit. Hot, loud, bad idle battery.

`300000` works for me. Tune:

- Flicker/underflow → below GDDR6 retraining time, raise.
- mclk stuck at top → above what DML can hide, lower.

Useful range is below 404000. Ceiling depends on active display timing (vblank per frame), not pixel count or refresh rate alone. Higher-bandwidth modes (4K, high refresh, external + internal) lower it.

`0` or unpatched keeps stock behavior.

Add `amdgpu.dc_dram_clock_change_latency_ns=300000` to your kernel options.
