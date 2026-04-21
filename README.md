# t3ra1n

**palera1n from rooted Android. No PC. No Mac. No Linux box.**

t3ra1n is a Termux-native wrapper for [palera1n](https://github.com/palera1n/palera1n) that runs the full checkm8 jailbreak chain directly from a rooted Android phone. It handles USB HAL coexistence, kernel authorization lockdown, and process lifecycle, so you just pick an option and plug in your iPhone.

This is a **beta**. It works. It has bugs. It has only been tested on one device configuration. Read the whole README before you start.

-----

## How it works

Android’s USB stack actively interferes with palera1n’s libusb — the HAL reclaims the device during checkm8’s reconnect window, breaking the exploit. t3ra1n solves this with a Magisk module (`s3ra1n-usb`) that:

- Stops USB HAL services before palera1n runs
- Forces DWC3 controller to host mode
- Runs a 20ms watchdog that re-authorizes Apple devices across all PID transitions (`1227 DFU → 1227 PWNed → 4141 PongoOS`)
- Restores full Android USB state after palera1n exits

The result: checkm8 hits reliably, the full jailbreak chain completes, and your Android phone goes back to normal.

-----

## Tested on

|Component    |Details                          |
|-------------|---------------------------------|
|Host device  |Sony Xperia XZ (Kagura, F8331)   |
|SoC          |Qualcomm Snapdragon 820 (MSM8996)|
|Android      |AOSP 13 GSI                      |
|Root         |Magisk                           |
|Target iPhone|iPhone X (A11, iOS 16.7.10)      |
|palera1n     |v2.2.1 linux-arm64               |

**This is the only configuration t3ra1n has been confirmed working on.** Other Qualcomm devices with DWC3 USB controllers will likely work (Pixel 3/4, OnePlus 6/7, Xperia XZ1/XZ Premium, various Xiaomi/Redmi), but there are no guarantees. Samsung devices with the Exynos USB stack or different HAL service names may need adjustments.

-----

## Requirements

- Android phone with **root** (Magisk or KernelSU)
- `s3ra1n-usb` Magisk module installed (included in this repo)
- Python 3 in Termux (`pkg install python`)
- `palera1n-linux-arm64` binary in Documents or Downloads
- USB OTG cable
- iPhone with a checkm8-compatible chip (A8–A11), iOS 15+

-----

## Setup

**1. Install the Magisk module**

Flash `s3ra1n-usb-v0.1.zip` via the Magisk app → Install from storage. Reboot.

**2. Install Python in Termux**

```bash
pkg update && pkg install python
```

**3. Download palera1n**

```bash
curl -L https://github.com/palera1n/palera1n/releases/download/v2.2.1/palera1n-linux-arm64 \
  -o ~/storage/downloads/palera1n-linux-arm64
```

Or download it manually and place it in your Downloads or Documents folder. t3ra1n will find it automatically.

**4. Download t3ra1n**

```bash
curl -L https://github.com/NoxProductions/s3ra1n/releases/latest/download/t3ra1n.py \
  -o ~/storage/downloads/t3ra1n.py
```

**5. Run**

```bash
python3 ~/storage/downloads/t3ra1n.py
```

-----

## Usage

```
        .:'
    __ :'__
 .'`__`-'__``.    t3ra1n  v1.0
:__________.-'    An experimental Ra1n Project
:_________:       github.com/NoxProductions/s3ra1n
 :_________`-;    Requires: Magisk + s3ra1n-usb module
  `.___.-.__.'

   1.  Jailbreak      Rootful               (-f)
   2.  Jailbreak      Rootless              (-l)
   3.  Setup          Create fakefs         (-f -B)
   4.  Setup          Partial fakefs        (-f --setup-partial-fakefs)
   5.  Boot           PongoOS shell         (--pongo-shell)
   6.  Boot           Verbose boot          (-f --verbose-boot)
   7.  Boot           Safe mode             (-f --safe-mode)
   8.  Recovery       Enter recovery        (--enter-recovery)
   9.  Recovery       Exit recovery         (--exit-recovery)
  10.  Info           Device info           (--device-info)
  11.  Danger         Force revert          (--force-revert)

   0.  Exit
  DB.  USB Diagnostic
       Append DB for debug output  e.g. 1DB
```

Append `DB` to any option number to see full phase output. Example: `1DB` runs rootful jailbreak with verbose logging. `DB` alone opens the USB diagnostic dump.

-----

## Critical usage notes

These are not suggestions. Ignoring them will waste your time.

**Do not plug in the OTG cable before the script tells you to.**
t3ra1n needs to lock down the USB stack before the iPhone connects. If the iPhone is already plugged in when the script starts, Android will have already claimed it and the jailbreak will fail. Wait until Phase 5 prompts you.

**Do not unplug the OTG cable while palera1n is running.**
Unplugging mid-exploit puts palera1n into a kernel D-state — it becomes unkillable even with `kill -9`, even as root. The only recovery is a full Android reboot. Wait until t3ra1n’s Phase 7 completes and says “USB state restored” before removing the cable.

**Do not close Termux during Phase 7.**
Phase 7 restores Android’s USB stack. It takes up to 2 minutes. If you kill Termux during this phase, your USB stack will be in a partially-locked state until you reboot.

**First-time rootful jailbreak: use option 3 first, then option 1.**
Option 3 (`-f -B`) creates the fakefs partition. This only needs to run once. After that, use option 1 (`-f`) for every subsequent boot.

**iPhone X and iPhone 8 series: enter Recovery mode, not DFU.**
palera1n handles the DFU transition from Recovery automatically for A11 chips. If you try to enter raw DFU on an iPhone X, t3ra1n will detect it and tell you to switch to Recovery. The script blocks this case explicitly.

-----

## Known bugs

**USB charging stops working after multiple jailbreak attempts**
After several back-to-back runs, the DWC3 controller can get into a state where USB charging no longer works. A full Android reboot clears this. If your phone stops charging via USB, reboot it.

**Occasional “Resource busy” on second attempt without reboot**
If palera1n fails mid-run and you immediately retry, the previous process may still hold the USB interface. t3ra1n attempts to clear this automatically, but if it fails, reboot Android before retrying. The zombie detection at startup will tell you if this is the case.

**2-minute Phase 7 unlock time**
Restoring the USB stack takes longer than it should on some configurations due to HAL service restart timing. This is a known issue and will be improved in a future version. Do not close Termux.

**DWC3 mode path hardcoded to Qualcomm devices**
The USB controller path detection covers MSM8996, MSM8998, SDM845, SM8150, SM8250, MSM8953, and SDM660. Devices with MediaTek USB controllers or non-standard sysfs paths are not supported yet.

-----

## Architecture

```
t3ra1n.py  (Termux menu + lifecycle)
    │
    ├── s3ra1n-usb (Magisk module)
    │   ├── Stops vendor.usb-hal-1-0
    │   ├── Forces DWC3 → host mode
    │   ├── Sets authorized_default=0 on root hubs
    │   └── 20ms watchdog: re-authorizes Apple devices + chmod 666 devnodes
    │
    └── palera1n-linux-arm64
        ├── checkm8 UaF exploit
        ├── PongoOS boot
        ├── KPF kernel patch
        └── fakefs / rootless bootstrap
```

-----

## Relationship to s3ra1n

t3ra1n is the Android-native branch of [s3ra1n](https://github.com/NoxProductions/s3ra1n). s3ra1n was the original proof of concept that ran palera1n on a Samsung Galaxy S3 running postmarketOS — real Linux, no Android HAL to fight. t3ra1n solves the harder version of the same problem: running the same exploit from inside Android, where the USB stack actively works against you.

Both tools live in the same repository. s3ra1n targets postmarketOS and Linux. t3ra1n targets rooted Android with Termux.

-----

## Contributing

If you get t3ra1n working on a device not listed in the tested section, open an issue or PR with your device details (SoC, Android version, root method, DWC3 sysfs path). The goal is to build a device compatibility table.

If it doesn’t work, `python3 t3ra1n.py --diag` with your iPhone plugged in gives a full USB state dump. Include that in your issue.

-----

## Credits

- [palera1n team](https://github.com/palera1n/palera1n) — the actual jailbreak
- [checkra1n team](https://checkra.in) — checkm8
- Built by [Noxbit](https://github.com/NoxProductions)

-----

## Disclaimer

This tool is for educational and research purposes. Use it on devices you own. The authors take no responsibility for bricked devices, data loss, or warranty voids. checkm8 is a hardware vulnerability — it cannot be patched by Apple on affected devices, but that doesn’t mean it’s risk-free to exploit.

-----

*Part of the s3ra1n project — hardware that has no business doing this in 2026.*
