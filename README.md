# t2-kernel-patches

Kernel patches for my MacBookPro16,1 on CachyOS. Targets T2 Macs with AMD dGPU, BCM4364 WiFi, Touch Bar. Based on the CachyOS T2 kernel. May not apply or work elsewhere.

## `0001-x86-resume-cached-cpu-bringup.patch`

"Enabling non-boot CPUs" after S3 took 8–97 s (t2linux folklore calls slow smpboot "normal"). Root cause, not Mac-specific: `arch_thaw_secondary_cpus_begin()` defers AP cache/MTRR init to the end of thaw, so every secondary CPU runs its entire serial bringup with reset-state MTRRs (`MTRRdefType=0xc00` — default type UC, no ranges) — all of RAM uncacheable. Measured on the i9-9880H: ALU loops 500–8000× slower, 8 MB memset at 1.7 MB/s, ~1 ms per empty tick at HZ=1000; HT siblings of the boot CPU immune (MTRRs are core-scoped), which is what identified it.

Patch drops the deferral; the thaw path takes the same inline `cache_cpu_init()` rendezvous regular hotplug uses. Validated via a runtime-equivalent module before rebuild: bringup 8–97 s → 37 ms, resume becomes amdgpu-bound (~1 s). Affects all x86 S3 resume upstream; magnitude scales with core count × HZ × pending tick work.

## `0001-macsmc-hwmon-resume-null-deref.patch`

Hard freeze on every S3 resume with the cachyos-7.1.3-1 tree. `macsmc_hwmon_probe()` never calls `platform_set_drvdata()` — the hwmon pointer only goes to `devm_hwmon_device_register_with_info()`, which sets drvdata on the hwmon class device. `macsmc_hwmon_resume()` reads `dev_get_drvdata(&pdev->dev)` (always NULL) and dereferences `hwmon->smc` → oops at 0x8 in `dpm_resume()`, killing systemd-sleep with user.slice still frozen. Latent until this tree: the `is_acpi` Intel-Mac binding is new; DT-only builds never bound on T2 Macs. One line: set platform drvdata in probe.

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

## `0001-xhci-pci-t2-titan-ridge-no-runtime-pm.patch`

External USB-C ports detect nothing once the Thunderbolt xHCI runtime suspends. The JHL7540 (`8086:15ec`) never signals PME on port connect: in D3hot a plug gets no enumeration, nothing in dmesg, `wakeup_count` stays 0. No trust prompt on an iPhone/iPad, no `usbmuxd`, no storage.

Root hub wakeup is not the missing piece. Enabling it arms the hardware (`lspci` goes `PME-Enable-` → `PME-Enable+` in D3hot), but a plug still produces nothing. Forcing D0, same cable and port, enumerates instantly. The PME is never sent.

Drops `XHCI_DEFAULT_PM_RUNTIME_ALLOW` for Titan Ridge on Apple systems, cleared after the `hci_version >= 0x120` rule so that rule can't set it again. Upstream sets it deliberately ([patchwork 10599433](https://patchwork.kernel.org/patch/10599433/)): an awake xHCI keeps the whole Thunderbolt controller awake. Cost is ~0.34 W idle on a MacBookPro16,1, inside sample noise on battery.

Userspace equivalent, if you don't want to rebuild: `ATTR{vendor}=="0x8086", ATTR{device}=="0x15ec", ATTR{power/control}="on"` in a udev rule.

## `0001-amdgpu-navi14-athub-pg.patch`

Navi 14 is the only navi1x without `AMD_PG_SUPPORT_ATHUB` in `nv_common_early_init()` (`nv.c`); Navi 10 and Navi 12 both set it. The flag gates `FEATURE_ATHUB_PG_BIT` in `navi10_init_allowed_features()` (`navi10_ppt.c`), so PMFW never power-gates ATHUB on Navi 14. Patch adds the flag to the Navi 14 `pg_flags`.

## `0001-amdgpu-navi14-dfll-pll-shutdown.patch`

`navi10_append_powerplay_table()` sets `DPM_OVERRIDE_DISABLE_DFLL_PLL_SHUTDOWN` whenever `PP_GFXOFF_MASK` is set (`TODO: remove it once SMU fw fix it`, no firmware version gate), so the gfx DFLL PLL stays powered through every GFXOFF cycle on all navi1x. Patch skips the override when MP1 is IP version 11.0.5 (Navi 14). Navi 10 and Navi 12 unchanged.

## `0001-amdgpu-mclk-override.patch`

Adds `amdgpu.dc_dram_clock_change_latency_ns` to override the DRAM latency the AMD DML uses to decide whether mclk switching is safe.

Navi 14 (Radeon Pro 5300M/5500M in the 2019 16" MBP) hardcodes 404µs in `dcn2_0_nv14_soc` (`dcn20_fpu.c`). The DML can't hide that in the 3072×1920 panel's vblank, marks switching `dm_dram_clock_change_unsupported` (see `MinActiveDRAMClockChangeMargin` in `display_mode_vba_20v2.c`), and pins the dGPU at the top pstate whenever the panel is lit. Hot, loud, bad idle battery.

Add `amdgpu.dc_dram_clock_change_latency_ns=400000` to your kernel options. Tune:

- Flicker/underflow → below GDDR6 retraining time, raise.
- mclk stuck at top → above what DML can hide, lower.

Useful range is below 404000. Ceiling depends on active display timing (vblank per frame), not pixel count or refresh rate alone. Higher-bandwidth modes (4K, high refresh, external + internal) lower it.

`0` or unpatched keeps stock behavior.
