# SLEEP FIX: Fans, pump and RGB stay on while Bazzite is asleep

**Symptom:** Bazzite suspends (screen off, session paused), but case fans, the CPU cooler or AIO pump, and RGB lighting keep running. A key press or mouse click wakes the system normally.

**Applies to:** any Bazzite image (desktop, Deck, GNOME, KDE; stable or testing) on Intel or AMD desktops. Laptops are covered with a caveat in [Laptops](#laptops).

A Word version of this guide is in [`SLEEP FIX.docx`](SLEEP%20FIX.docx).

---

## Why it happens

Linux supports two kinds of suspend-to-RAM:

| Mode | Linux name | What the hardware does |
|---|---|---|
| **S0ix / Modern Standby** | `s2idle` | The board stays in the "on" (S0) state while the CPU and devices idle. The PSU's 12V rail stays live, so **fans, pump and RGB keep running**. |
| **S3 / Suspend-to-RAM** | `deep` | The PSU turns off everything except 5V standby power, which keeps RAM alive. **Fans, pump and most RGB go dark.** |

Many recent boards and firmware versions advertise s2idle, and the kernel uses it by default whenever the firmware offers it. On a desktop that usually means the system "sleeps" with everything still spinning.

The fix is to get the system into **S3 (`deep`)**. That takes the right setting in Linux, and sometimes a BIOS/UEFI change so the firmware offers S3 at all.

---

## Step 1: Check which sleep mode you are using

```bash
cat /sys/power/mem_sleep
```

The mode in brackets is the current default.

| Output | Meaning | Next step |
|---|---|---|
| `[s2idle] deep` | S3 is available but not the default | Step 2 |
| `s2idle [deep]` | Already using S3 | Skip to [Things that may still stay lit](#things-that-may-still-stay-lit-in-s3) |
| `[s2idle]` | The firmware is not offering S3 | Step 3 first, then Step 2 |

After a suspend and resume, confirm the mode that was actually used:

```bash
journalctl -b | grep -i "PM: suspend entry"
```

It prints `PM: suspend entry (s2idle)` or `PM: suspend entry (deep)`.

---

## Step 2: Switch Linux to deep sleep

### Test it first (temporary, until reboot)

```bash
echo deep | sudo tee /sys/power/mem_sleep
systemctl suspend
```

Fans, pump and RGB should switch off. Wake the system with the keyboard or power button and check that everything works: display, network, audio, USB devices. Do not make the change permanent until this test passes.

### Make it permanent

Bazzite is image-based (rpm-ostree), so kernel boot options are changed with `rpm-ostree kargs`, not by editing GRUB:

```bash
rpm-ostree kargs --append-if-missing=mem_sleep_default=deep
systemctl reboot
```

To undo:

```bash
rpm-ostree kargs --delete=mem_sleep_default=deep
systemctl reboot
```

### Alternative: systemd setting

On systemd 256 or newer, you can set the mode in a systemd config file instead of a kernel argument. `/etc` is writable on Bazzite.

```bash
sudo mkdir -p /etc/systemd/sleep.conf.d
printf '[Sleep]\nMemorySleepMode=deep\n' | sudo tee /etc/systemd/sleep.conf.d/deep.conf
```

To undo, delete `/etc/systemd/sleep.conf.d/deep.conf`.

---

## Step 3: BIOS/UEFI settings

Update the BIOS to the latest version first; sleep fixes are common in firmware updates. Then look for the settings below. Names and locations vary by vendor and BIOS version, so use the BIOS search function (often F3 or Ctrl+F) if your board has one.

### What to change

| Setting (common names) | Set to | Why |
|---|---|---|
| **Suspend to RAM**, **S3 support**, **ACPI Sleep State: S3** | Enabled / Auto / S3 | Makes the firmware offer S3 to the OS |
| **Low Power S0 Idle Capability**, **Modern Standby**, **S0ix** | Disabled | On some boards, S3 is hidden while this is enabled |
| **LED in sleep / hibernate / soft-off state** | Off | Turns off the motherboard's own RGB and ARGB headers while asleep |
| **ErP / ErP Ready / EuP** | Enabled, then test | Cuts standby power. On some boards this also disables USB wake, so turn it off again if the keyboard can't wake the PC |
| **Wake on USB**, **Power On By Keyboard**, **Resume by USB device** | Enabled | Lets the keyboard or mouse wake the system from S3 |
| **USB power in S3/S4/S5**, **USB standby power** | Disabled (optional) | Turns off USB-powered RGB and accessories in sleep. Disabling it can also stop USB wake |
| **Wake on LAN**, **Resume by PCI-E device** | Disabled, unless you use them | Prevents unwanted wakeups |

### Where to look by vendor

These are typical locations. Your board may differ.

| Vendor | Sleep / ErP / wake | LED in sleep |
|---|---|---|
| **ASUS** | Advanced > APM Configuration (ErP Ready, Power On By PCI-E); Advanced > ACPI Configuration | Advanced > Onboard Devices Configuration > LED lighting > "When system is in sleep, hibernate or soft off states" |
| **MSI** | Settings > Advanced > Power Management Setup (ErP Ready); Settings > Advanced > Wake Up Event Setup | Settings > Advanced > "EZ LED Control" or the LED options in the same area |
| **Gigabyte / AORUS** | Settings > Platform Power (ErP, Power On By Keyboard, wake options) | Settings > Miscellaneous > "LEDs in Sleep, Hibernation, and Soft Off States" |
| **ASRock** | Advanced > ACPI Configuration (Suspend to RAM, PCIE Devices Power On, USB wake) | Advanced > Onboard Devices or Tool > RGB LED options |
| **Laptops / OEM desktops** | Power or Sleep menus; some have an explicit option (for example "Sleep State: Linux" on some Lenovo models) | Usually not adjustable |

---

## Step 4: Make keyboard and mouse wake work from S3

S3 powers down more of the system, so a device that woke the PC from s2idle may not wake it from S3.

Check which devices are allowed to wake the system:

```bash
cat /proc/acpi/wakeup
```

The USB controllers (usually named `XHC`, `XHC0`, `XHCI` or similar) should show `*enabled`. To enable one for the current boot, use its exact name from the list:

```bash
echo XHC | sudo tee /proc/acpi/wakeup
```

That command **toggles** the setting, so run it only once and check the list again.

To keep it enabled after reboots, add a udev rule for the specific keyboard. Find its vendor and product IDs with `lsusb`, then create `/etc/udev/rules.d/90-usb-wake.rules`:

```
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="xxxx", ATTR{idProduct}=="yyyy", ATTR{power/wakeup}="enabled"
```

Replace `xxxx` and `yyyy` with the IDs from `lsusb`, then reboot. The power button always works as a fallback.

---

## Things that may still stay lit in S3

### USB-powered RGB, fans and AIO pumps

RGB hubs, fan controllers and AIO pumps that connect over USB (for example Corsair iCUE, NZXT, Lian Li and Razer devices) can draw USB standby power and stay lit even in S3. Either disable USB standby power in the BIOS (see Step 3), or turn them off with a sleep hook.

Create `/etc/systemd/system-sleep/rgb-off`:

```sh
#!/bin/sh
case "$1" in
  pre)  openrgb --noautoconnect -p off.orp 2>/dev/null || openrgb -c 000000 ;;
  post) openrgb -p default.orp ;;
esac
```

Then make it executable:

```bash
sudo chmod +x /etc/systemd/system-sleep/rgb-off
```

This assumes [OpenRGB](https://openrgb.org/) is installed and you have saved profiles named `off` and `default` in it. For AIO pumps and fan controllers, [liquidctl](https://github.com/liquidctl/liquidctl) commands can go in the same hook.

### GPU

Moving from s2idle to S3 changes how the graphics card is powered down. If resume gives a black screen or a frozen display, check the GPU first.

- **NVIDIA:** the suspend helper services must be enabled. Bazzite's NVIDIA images normally set this up. Check with:
  ```bash
  systemctl status nvidia-suspend nvidia-resume nvidia-hibernate
  ```
- **AMD and Intel graphics:** usually work without changes. If resume fails, check `journalctl -b -1` after the next reboot for `amdgpu` or `i915`/`xe` errors.

---

## Laptops

Many laptops from about 2020 onward support **only** s2idle; the firmware has no working S3. On those machines, forcing `deep` can cause failed resumes or a dead battery. On a laptop:

- Only use `mem_sleep_default=deep` if `deep` appears in `/sys/power/mem_sleep` **and** the Step 2 test passes several times in a row.
- Fans and lights staying on in s2idle on a laptop often means the system is not reaching its lowest idle state. That is a different problem; the s2idle debugging tools are the place to start (for AMD systems, the `amd-s2idle` tool from the amd-debug-tools project).

---

## If S3 is not available at all

If `deep` never appears, even after BIOS updates and the Step 3 changes, the firmware does not support S3 and you are limited to s2idle. The best you can do then:

- Use the sleep hook above to turn off RGB, and lower fan and pump speeds with liquidctl or your fan control tool before sleep, then restore them on wake.
- Use hibernate or a full shutdown when you want everything off.

---

## Quick summary

1. `cat /sys/power/mem_sleep`: if `deep` is listed, test it with `echo deep | sudo tee /sys/power/mem_sleep` and `systemctl suspend`.
2. If it works, make it permanent: `rpm-ostree kargs --append-if-missing=mem_sleep_default=deep`.
3. If `deep` is not listed, enable S3 or disable Modern Standby in the BIOS.
4. Set the BIOS "LED in sleep" option to Off, and make sure USB wake is enabled.
5. For USB-powered RGB, add the OpenRGB sleep hook.
