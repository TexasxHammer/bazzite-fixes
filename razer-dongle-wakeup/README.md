# Wake Bazzite from Sleep with the Razer Wolverine V3 Pro Wireless Dongle

By default, Linux doesn't let the Razer Wolverine V3 Pro's wireless dongle wake the PC from sleep. When the controller is wired, it's the USB device itself and can wake the system. When it's wireless, the dongle is the USB device, and its wakeup setting is `disabled`. This guide turns it on.

## Device details (this machine)

| Field | Value |
|---|---|
| Device | Razer Wolverine V3 Pro for Xbox 2.4 (wireless dongle) |
| USB ID | `1532:0a4c` |
| sysfs path | `/sys/bus/usb/devices/1-4.1` (may change if you use a different port) |
| Default wakeup | `disabled` |
| Sleep mode | S3 `deep` |

## 1. Find the dongle

`lsusb` isn't installed on Bazzite by default, so this reads the information directly from sysfs:

```bash
for d in /sys/bus/usb/devices/*; do [ -f "$d/idVendor" ] && grep -qi 1532 "$d/idVendor" && echo "$(basename $d) $(cat $d/idVendor):$(cat $d/idProduct) $(cat $d/product 2>/dev/null) wakeup=$(cat $d/power/wakeup)"; done
```

Example output:

```
1-4.1 1532:0a4c Razer Wolverine V3 Pro for Xbox 2.4 wakeup=disabled
```

## 2. Test it for this boot only

Replace `1-4.1` with the path from step 1 if yours is different:

```bash
echo enabled | sudo tee /sys/bus/usb/devices/1-4.1/power/wakeup
```

Put the PC to sleep, then press the controller's **Xbox/home button**. The controller drops its connection during sleep, and the home button reconnects it to the dongle, which then wakes the PC. Pressing a face button may not work.

## 3. Make it permanent (udev rule)

On Bazzite, `/etc` is writable and keeps its contents across updates, so this rule survives them:

```bash
echo 'ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="1532", ATTR{idProduct}=="0a4c", ATTR{power/wakeup}="enabled"' | sudo tee /etc/udev/rules.d/90-razer-dongle-wakeup.rules
```

```bash
sudo udevadm control --reload && sudo udevadm trigger
```

The rule matches the dongle by its USB ID, so it still applies if you move the dongle to another port.

To remove it:

```bash
sudo rm /etc/udev/rules.d/90-razer-dongle-wakeup.rules
```

## 4. If it still doesn't wake the PC

### Check your BIOS settings
- Turn on **Wake on USB** / **USB wake from S3**.
- Turn off **ErP / EuP Ready**. It cuts power to the USB ports during sleep.
- If your board has an option like "USB power in S3/S4/S5", set it to always on.
- Use a USB port on the motherboard's back panel, not a front-panel port or an unpowered hub.

### Enable wakeup on the parent hub
The dongle is on hub `1-4`. A Logitech receiver on the same hub can already wake the PC while the hub's wakeup is `disabled`, so this usually isn't needed. If you want to try it anyway:

```bash
echo enabled | sudo tee /sys/bus/usb/devices/1-4/power/wakeup
```

### Try s2idle instead of deep sleep
In S3 deep sleep, some boards cut power to USB devices. s2idle usually handles USB wake better, but it uses slightly more power while asleep:

```bash
rpm-ostree kargs --append-if-missing=mem_sleep_default=s2idle
```

Reboot afterward. To undo:

```bash
rpm-ostree kargs --delete=mem_sleep_default=s2idle
```

## Possible hard limit

Some wireless dongles can't send a wake signal at all, because their firmware doesn't support it. If none of the steps above work, only a firmware update could fix it, and you'd need Razer Synapse on Windows to install one. Until then, you can:
- Use the controller wired when you need it to wake the PC.
- Wake the PC with a keyboard, mouse or the power button, then use the controller.
