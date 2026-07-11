# t2-kernel-patches

Kernel patches for my MacBookPro16,1 on CachyOS. Targets T2 Macs with AMD dGPU, BCM4364 WiFi, Touch Bar. Based on the CachyOS T2 kernel. May not apply or work elsewhere.

## `0001-x86-resume-cached-cpu-bringup.patch`

"Enabling non-boot CPUs" after S3 took 8–97 s (t2linux folklore calls slow smpboot "normal"). Root cause, not Mac-specific: `arch_thaw_secondary_cpus_begin()` defers AP cache/MTRR init to the end of thaw, so every secondary CPU runs its entire serial bringup with reset-state MTRRs (`MTRRdefType=0xc00` — default type UC, no ranges) — all of RAM uncacheable. Measured on the i9-9880H: ALU loops 500–8000× slower, 8 MB memset at 1.7 MB/s, ~1 ms per empty tick at HZ=1000; HT siblings of the boot CPU immune (MTRRs are core-scoped), which is what identified it.

Patch drops the deferral; the thaw path takes the same inline `cache_cpu_init()` rendezvous regular hotplug uses. Validated via a runtime-equivalent module before rebuild: bringup 8–97 s → 37 ms, resume becomes amdgpu-bound (~1 s). Affects all x86 S3 resume upstream; magnitude scales with core count × HZ × pending tick work.

## `0001-brcmfmac-suspend-fix.patch`

Suspend hangs when Broadcom firmware stops responding before the PCI driver finishes suspending.

- `brcmf_pcie_pm_enter_D3` times out waiting 2s for a D3_INFORM ACK and aborts suspend with `-EIO`. Patch lets suspend proceed and sets `devinfo->d3_ack_timeout`.
- `brcmf_pcie_pm_leave_D3` checks the flag on resume and forces the existing remove + re-probe path (firmware reload). INTMASK alone can't detect the hang: firmware that timed out on D3_INFORM can still return nonzero INTMASK, hot-resuming into a zombie (associated, DHCP up, data path dead). Toggling networking in that state can wedge the chip's PCIe interface → whole-machine hard lock.
- `brcmf_msgbuf_delete_flowring` waits for `outstanding_tx` to drain before checking bus state, burning 5–10ms × 10 retries per flowring. Patch checks bus state before and during the wait.

## `0001-touchbar-suspend-resume.patch`

The in-kernel Touch Bar driver `hid-appletb-kbd` (used without userspace `tiny-dfr`) comes back blank after resume.

- Mode restore runs on a 1.5s delayed workqueue, not inline, so HID commands wait for USB to come back.
- Mode-off at suspend is dropped — the device is losing power anyway, and the write raced USB teardown.
- Probe now defers (`-EPROBE_DEFER`) when the backlight device isn't ready instead of continuing without it, removing the optional-backlight case.

## `0001-amdgpu-mclk-override.patch`

Adds `amdgpu.dc_dram_clock_change_latency_ns` to override the DRAM latency the AMD DML uses to decide whether mclk switching is safe.

Navi 14 (Radeon Pro 5300M/5500M in the 2019 16" MBP) hardcodes 404µs in `dcn2_0_nv14_soc` (`dcn20_fpu.c`). The DML can't hide that in the 3072×1920 panel's vblank, marks switching `dm_dram_clock_change_unsupported` (see `MinActiveDRAMClockChangeMargin` in `display_mode_vba_20v2.c`), and pins the dGPU at the top pstate whenever the panel is lit. Hot, loud, bad idle battery.

Add `amdgpu.dc_dram_clock_change_latency_ns=400000` to your kernel options. Tune:

- Flicker/underflow → below GDDR6 retraining time, raise.
- mclk stuck at top → above what DML can hide, lower.

Useful range is below 404000. Ceiling depends on active display timing (vblank per frame), not pixel count or refresh rate alone. Higher-bandwidth modes (4K, high refresh, external + internal) lower it.

`0` or unpatched keeps stock behavior.
