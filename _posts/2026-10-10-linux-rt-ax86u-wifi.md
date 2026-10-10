---
title:  "RT-AX86U with a dead 2.4 GHz radio: chasing wl0 through rc, wl.ko and WFD"
last_modified_at: 2026-10-10T00:00:00+00:00
categories:
  - hardware
tags: linux router meilin reverse-engineering
toc: true
---

## Summary

After about six years of service, the Wi-Fi on my ASUS RT-AX86U (the `RT-AX86U_EPA` variant) stopped working, on both bands.
The root cause turned out to be the 2.4 GHz radio: its PCIe link never comes up at boot.
Losing it also broke 5 GHz, even though the 5 GHz radio itself is alive and still detected by the driver.

The firmware numbers radios in the order they attach.
The surviving 5 GHz radio becomes "radio 0", and ASUSWRT configures radio 0 as 2.4 GHz.
Loading Broadcom's `wl.ko` with `instance_base=1` would leave slot 0 free for the missing radio, but Broadcom's driver requires its unit number to match its WFD (offload) slot.
A lone radio always gets WFD slot 0, so that attempt fails.
This can't be fixed in ASUS's `rc` alone; it needs either kernel-side help or a fix from upstream.

* Router: RT-AX86U (`RT-AX86U_EPA`), BCM4908 SoC
* Firmware: KoolCenter Merlin build `3.0.0.4_388_24231_koolcenter`
* Kernel: 4.1.52, aarch64 (user space is 32-bit ARM)

## 1. Symptom

* Wi-Fi stopped working on both bands.
* The 2.4 GHz network disappeared completely.
* The 5 GHz network did not work as configured either.
* Rebooting and power-cycling the router did not help.
* Everything else kept working: wired clients had network access, and I could still log in to the router.

The firmware's own status page (KoolCenter's ROG toolbox) was confusing too.
It reported the 2.4 GHz radio as running and the 5 GHz radio as offline, even though there was no 2.4 GHz network:

| Item | Shown |
|------|-------|
| Firmware | RT-AX86U `3.0.0.4_388_24231_koolcenter` (built 2024-09-28) |
| Kernel / architecture | 4.1.52 / aarch64 |
| Radio temperature | 2.4G: 49 °C, 5G: offline |
| Transmit power | 2.4G: 26.00 dBm / 398.11 mW, 5G: offline |

The *Wireless → General* settings page didn't add up either.
With the band set to **2.4 GHz**:

* the channel list offered **36, 40 … 161**, which are 5 GHz channels;
* the page reported **current control channel: 157**;
* yet it still showed 2.4 GHz-only options, such as the 20/40 MHz bandwidth and the hint that automatic selection includes channels 12 and 13.

## 2. Root cause analysis

### 2.1 What the router actually sees

The router expects two radios:

| Slot | Band | Linux interface |
|------|------|-----------------|
| `wl0` | 2.4 GHz | `eth6` |
| `wl1` | 5 GHz | `eth7` |

The factory NVRAM still describes both radios: radio 0 (`devid=0x4494`, MAC ending `:10`) and radio 1 (`devid=0x442b`, MAC ending `:14`).

At boot, only one radio shows up, and it is the **5 GHz** one (MAC ending `:14`, i.e. factory radio 1):

* the driver creates `wl0` / `eth6` (Broadcom BCM6710);
* `wl -i eth6 bands` reports only `a`, and it lists only 5 GHz channels;
* `wl -i eth7 status` says `wl driver adapter not found`;
* the 2.4 GHz radio does not appear in the boot log at all.

The PCIe log shows why.
Of the router's three PCIe controllers, only one links up, and the BCM6710 is the only device behind it.
The other two report `link is DOWN` and are powered down.
The 2.4 GHz radio never even appears as a PCI device, so neither `wl.ko` nor `dhd.ko` gets a chance to attach it.

The physical 5 GHz radio is now driver instance 0, and the firmware treats it as the **2.4 GHz slot** (`wl0_nband=2`, `wl0_ifname=eth6`), while `wl1_ifname` stays empty.
The web UI's "2.4 GHz" page controls the 5 GHz radio and the "5 GHz" page controls nothing, which is why neither band works as configured.
It also explains the ROG toolbox: it reads the radio in slot `wl0` and labels it "2.4G", so the numbers it shows as 2.4G are really the 5 GHz radio's.
The transmit power confirms it: `wl -i eth6 txpwr_target_max` reports 26.00 dBm on channel 157, the same figure the toolbox shows as "2.4G".
The same goes for the settings page: the "2.4 GHz" page configures slot `wl0`, and its channel list comes from the radio in that slot, so it offers 5 GHz channels and reports channel 157.
The full log and NVRAM excerpts are in [Appendix A](#a-logs).

I want to be careful about one thing: I can't prove the 2.4 GHz silicon is dead.
The failure is below any driver: the PCIe link to the radio never comes up.
A dead chip would do that, but so would a power, clock or reset fault, or a problem with the link itself.

### 2.2 How ASUSWRT decides which radio is wl0

ASUSWRT asks the driver for each interface's unit number with the `WLC_GET_INSTANCE` ioctl and builds the NVRAM prefix `wl%d_` from it.
So the important number is the driver's runtime unit, not the position of `eth6` in `wl_ifnames`.
`rc` alone has 38 call sites that use `WLC_GET_INSTANCE`.

### 2.3 How rc loads the driver: `load_wl()`

`/sbin/rc` is the ASUS init/service binary (`/sbin/init` is a symlink to it).
It is a stripped 32-bit ARM executable; [Appendix D](#d-tools-and-method) describes how I found the function.
`load_wl()` starts at `0xc6140`. Reconstructed:

```c
void load_wl(void)                                   // 0xc6140
{
    add_to_list("wl",  modules);                     // order is fixed:
    add_to_list("dhd", modules);                     // "wl dhd"
    system("insmod .../cfg80211.ko");

    maxunit = -1;                                    // set ONCE, before the loop
    foreach (module, modules) {
        for (i = 1; i <= 32; i++) {                  // scan eth1..eth32
            if (wl_probe("eth%d") == 0 &&
                wl_ioctl(ifname, WLC_GET_INSTANCE, &unit) == 0 &&
                unit > maxunit)
                maxunit = unit;
        }
        base = maxunit + 1;
        if (module is "dhd")
            insmod dhd "instance_base=<base> dhd_msg_level=<nvram>";
        else
            insmod wl  "instance_base=<base> [msglevel=...]";
    }
    wl_driver_mode_update();                         // sets wlradio_dmode_<unit>=NIC/DGL
}
```

What this means:

* `wl` is always loaded first. At that point no Broadcom device exists, so `wl` always gets `instance_base=0`.
* `dhd` is loaded second and sees whatever `wl` created. On my router `wl` created unit 0, so `dhd` gets `instance_base=1`, which matches `/sys/module/dhd/parameters/instance_base`.
* There is **no** model-specific branch here and **no** NVRAM variable that controls `instance_base`; the only NVRAM inputs are debug message levels.

The source in [asuswrt-merlin.ng](https://github.com/RMerl/asuswrt-merlin.ng) (`release/src/router/rc/sysdeps/init-broadcom.c`, branch `3004.388`) matches the binary line for line.
ASUS does hard-code `instance_base` for a few models; BT10, for example, loads `wl` with `instance_base=1`.
RT-AX86U just uses the generic path.

In `sysinit()`, `init_wl()` (which calls `load_wl()`) runs at `0x3d2dc`, and the user hook `run_custom_script("init-start")` runs later at `0x3d474`.
So no boot script can run before `wl.ko` is loaded, but `init-start` does run before the radios are configured and bridged.

### 2.4 How wl.ko numbers radios

`wl.ko` is not stripped.
In `wl_attach` it computes the unit the way the public Broadcom sources say: `unit = instance_base + wl_found`, where `wl_found` counts radios that attached successfully ([Appendix C](#c-disassembly-excerpts)).
With only one radio attaching, it always gets unit `instance_base`, which is 0 under `rc`'s default.
So far the fix looked simple: load `wl.ko` with `instance_base=1`.

### 2.5 The blocker: WFD

WFD (WLAN Forwarding Driver, `wfd.ko`) is Broadcom's glue between the Wi-Fi driver and the hardware packet offload engine.
Every radio must register with it, and the two modules impose a combined rule:

* `wfd_bind()` hands out the **first free slot** (up to 4). With a single radio, that is always slot 0. Its own log shows this: it is told `wl_radio_idx 1`, and still assigns `wfd_idx 0`.
* `wl.ko`'s `wl_wfd_bind` requires **`unit == wfd_idx`**. If they differ, it prints `mismatch` and the attach fails.

Therefore **a lone radio can only be unit 0**.
Neither module has a parameter to change this ([Appendix E](#e-module-parameters)), the WFD bind in `wl_attach` is unconditional, and nothing else in the firmware binds a WFD slot early (`dhd.ko` only does so for dongle radios, and this router has none).

### 2.6 Why rc can't fix it

`rc` only chooses which modules to load, in what order, and with which `insmod` arguments.
The constraint lives inside two Broadcom kernel modules:

| Component | Owner | License |
|-----------|-------|---------|
| `wl.ko` | Broadcom | Proprietary |
| `wfd.ko` | Broadcom | GPL |
| `rc` / `load_wl()` | ASUS (Merlin modifies it) | GPL |

## 3. Solutions and test results

### 3.1 Swap the NVRAM band settings: failed

`nvram set wl0_nband=1` and `wl1_nband=2`, then `service restart_wireless`.
Both values are regenerated to their defaults.
Whatever rewrites them isn't in `rc`: I found no writer for `wlN_nband` there, so it should be in `wlconf` or `libshared.so`.

### 3.2 Change the module load order: ruled out by analysis

If `dhd` were loaded first, it would get `instance_base=0` and attach nothing; `wl` would then still find no devices and also get 0.
Order only matters when `dhd` actually creates an interface.

### 3.3 Reload wl.ko with instance_base=1 from init-start: failed

Merlin-based firmware runs `/jffs/scripts/init-start` early in boot when *Enable JFFS custom scripts and configs* is on (`jffs2_scripts=1`).
I first checked that the hook ran at all with a script that only writes a file to `/tmp`.
`logger` output from `init-start` is lost, because `syslogd` is not running yet, so I logged to a file under `/jffs` instead.

The test script ([Appendix B](#b-test-script)) unloads `wl` and loads it again with `instance_base=1`.
It only acts once, when a flag file exists, so a bad boot can be fixed by power-cycling.
My first version compared `wl -i eth6 bands` with `"a"`, but the command prints `a ` with a trailing space, so the script skipped itself.

The second attempt looked promising at first.
`rmmod wl` worked (use count 0), and the driver did attach the radio as **unit 1** (`wl1-kthrd`, `phy_radio_attach`).
Then WFD gave it slot 0, `wl_wfd_bind` reported `wl1 wfd_idx 0 mismatch`, and the driver tore the radio down again.
After that boot the router had no Wi-Fi at all until the next (normal) reboot.
This is the test that exposed the WFD rule in section 2.5.

### 3.4 Remaining options

For the 5 GHz radio to be unit 1, **something must already hold WFD slot 0** when it attaches.
It does not have to be a radio, or even a `wl` device.
In other words, a dead 2.4 GHz radio would need a placeholder.

1. **A placeholder kernel module** that calls `wfd_bind()` on a dummy device and keeps slot 0. It would be loaded from `/jffs` in `init-start` between `rmmod wl` and `insmod wl instance_base=1`. It doesn't modify any Broadcom code (`wfd_bind` is an exported symbol and `wfd.ko` is GPL), but it does mean building a module against this exact Broadcom 4.1.52 kernel and getting the WFD structures right. A mistake here crashes the kernel.
2. **Patching `wl.ko`** to ignore the mismatch. The check probably exists because the offload path routes packets by slot number, so this is risky, and it means modifying Broadcom's proprietary driver. Not something I want to do.
3. **Leaving the radio as wl0 and making ASUSWRT treat wl0 as 5 GHz.** This needs finding what regenerates `wl0_nband=2` (see 3.1). I haven't traced it yet.
4. **Upstream.** Only Broadcom can relax the rule, and only ASUS can ask them. For a six-year-old router I don't expect a firmware change.

There is also no GitHub repository for reporting issues to ASUS: ASUS only publishes GPL tarballs on its support site, and Asuswrt-Merlin has GitHub issues disabled (support goes through SNBForums).

For now the Wi-Fi is still broken: the 5 GHz radio is detected, but sits in the 2.4 GHz slot.
If I get to the `wlconf`/`libshared` side, or anyone has built a WFD placeholder for HND routers, I'll update this post.

## 4. Appendix

### A. Logs

Factory NVRAM (MACs partly masked):

```text
0:devid=0x4494   0:devpath0=sb/1/   devpath0=pcie/1/1/   0:macaddr=3C:7C:3F:xx:xx:10
1:devid=0x442b   1:devpath1=sb/1/   devpath1=pcie/0/1/   1:macaddr=3C:7C:3F:xx:xx:14
```

PCIe bring-up (only one of three links comes up):

```text
bcm963xx-pcie: found port [0] GEN2 Core Rev [3.04] with 2 Lanes
bcm963xx-pcie: core [0] link is DOWN
bcm963xx-pcie: core [0] powered down
bcm963xx-pcie: found port [1] GEN2 Core Rev [3.04] with 1 Lanes
bcm963xx-pcie: core [1] Link UP - [1] lanes, [GEN2] speed
pci 0000:00:00.0: Checking PCIe ASPM for vendor 14e4 device 6710
bcm963xx-pcie: found port [2] GEN2 Core Rev [3.04] with 1 Lanes
bcm963xx-pcie: core [2] link is DOWN
bcm963xx-pcie: core [2] powered down
```

PCI devices (normal boot):

```text
0000:00:00.0 vendor=0x14e4 device=0x4908 class=0x060400 driver=pcieport
0000:01:00.0 vendor=0x14e4 device=0x6710 class=0x028000 driver=wl
```

Normal boot (only one radio attaches):

```text
wl0: creating kthread wl0-kthrd
wl0: phy_radio_attach: RF Band Cap: 2G:1 5G:1
wfd_registerdevice Successfully registered dev eth6 ifidx 0 wfd_idx 0
eth6: Broadcom BCM6710 802.11 Wireless Controller 17.10.157.2809 (r801046)
```

Resulting NVRAM mapping:

```text
wl0_ifname=eth6   wl0_unit=0   wl0_nband=2   wl0_hwaddr=3C:7C:3F:xx:xx:14
wl1_ifname=       wl1_unit=0   wl1_nband=1   wl1_hwaddr=
```

Test boot with `instance_base=1` (section 3.3):

```text
dhd                   999536  0
wl                   7549641  0
insmod rc=0
...
kthread_should_stop detected on wl0
wl1: creating kthread wl1-kthrd
wl1: phy_radio_attach: RF Band Cap: 2G:1 5G:1
wl1 taf_do_enable: TAF is enabled
wfd_bind: wfd0-thrd initialized pktlists: radio 1 nodes 1032 xfer wl_pktfwd_xfer_callback+0x0/0x440 [wl]
wfd_bind: Dev eth%d wfd_idx 0 wl_radio_idx 1 Type skb:sll configured WFD thread wfd0-thrd minQId/maxQId (8/9), status (0) qmask 0x3
wl_wfd_bind: wl1 wfd_idx 0 success
wl_wfd_bind: wl1 wfd_idx 0 mismatch
wfd_unregisterdevice Error Incorrect wfd_idx -1 passed
kthread_should_stop detected on wl1
```

### B. Test script

`/jffs/scripts/init-start`, enabled for one boot with `touch /jffs/wlib.enable`:

```sh
#!/bin/sh
LOG=/jffs/wlib.log
KO=/lib/modules/$(uname -r)/extra/wl.ko
{
echo "=== uptime $(cut -d' ' -f1 /proc/uptime) init-start"
[ -f /jffs/wlib.enable ] || { echo "no flag, skip"; exit 0; }
rm -f /jffs/wlib.enable                       # one-shot: next boot is normal
wl -i eth7 bands >/dev/null 2>&1 && { echo "eth7 exists, skip"; exit 0; }
B=" $(wl -i eth6 bands 2>/dev/null | tr -s ' \t\n' ' ') "
case "$B" in *" b "*) echo "eth6 has 2.4G band, skip"; exit 0;; esac
case "$B" in *" a "*) ;; *) echo "eth6 not 5G, skip"; exit 0;; esac
lsmod | grep -E '^(wl|dhd) '
rmmod wl || { echo "rmmod wl failed"; exit 0; }
insmod "$KO" instance_base=1; echo "insmod rc=$?"
dmesg | grep -E 'wl[0-9]|eth6' | tail -20
} >> $LOG 2>&1
```

### C. Disassembly excerpts

`wl.ko`, `wl_attach` (unit computation):

```text
205118: ldp  w0, w21, [x20, #0x8]   ; w0 = instance_base, w21 = wl_found
205128: adds w21, w0, w21           ; unit = instance_base + wl_found
205180: str  w21, [x26]             ; wl->unit = unit
```

`wl.ko`, `wl_wfd_bind` (the mismatch check):

```text
4230: ldr  w19, [x20]               ; w19 = wl->unit  (1)
4260: bl   wfd_bind                 ; returns the WFD slot  (0)
4284: cmp  w19, w21                 ; unit == wfd_idx ?
4288: b.ne 42e8                     ; no -> "mismatch", return -1 -> attach fails
```

`rc` strings referenced from `load_wl()`:

| String | Used at |
|--------|---------|
| `load_wl(): starting...` | `0xc6190` |
| `instance_base=%d dhd_msg_level=%d` | `0xc63a8` |
| `instance_base=%d %s` | `0xc6488` |
| `load_wl(): insmod %s %s.` | `0xc6274` |
| `load_wl(): end.` | `0xc6584` |

Other useful `rc` addresses (388_24231 KoolCenter build): `init_wl()` at `0xc661c`, `wl_driver_mode_update()` at `0xc5ebc`.

### D. Tools and method

* **`rc`** is a stripped 32-bit ARM executable (`ET_EXEC`, not PIE, base `0x10000`). Searching for string addresses finds nothing, because the code builds them PC-relatively:

  ```text
  ldr   r0, [pc, #imm]    ; literal = offset
  add   r0, pc, r0        ; address = offset + (this instruction + 8)
  ```

  macOS's `/usr/bin/objdump` is LLVM and disassembles ARM and AArch64 fine. A short Python script that resolves every `ldr`/`add pc` pair and annotates the listing with the target string was enough; no Ghidra needed.
* **Firmware image:** the `pureubi` `.w` file is a raw UBI image whose root filesystem is **UBIFS** (not squashfs). `ubi_reader` (`ubireader_extract_files`) extracted it in a Python venv. The extracted `/sbin/rc` was byte-identical to the one on the router.
* **Safety:** a one-shot flag file for anything that reloads drivers at boot is worth the extra two lines.

### E. Module parameters

| Module | Parameters |
|--------|------------|
| `wl.ko` | `ctdma intf_name nompc passive_channel_skip instance_base piomode oneonly wl_txq_bound wl_txq_thresh txworkq passivemode` |
| `wfd.ko` | `wifi_prefix num_packets_to_read` |

None of them chooses or reserves a WFD slot.
