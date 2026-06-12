# Making Ender 3 Pro + Klipper on Pi 4 boot in 24s instead of 53s

My Ender 3 Pro runs Klipper on a Pi 4 (Raspberry Pi OS trixie, SD card). After flipping the power switch the stock
LCD stayed dark for about 30 seconds and Mainsail took almost a minute. After this rabbit hole the OS boots in ~9s
and the screen lights up ~24s after power-on. Most of what's left is hardware, not software.

Notes from the journey, mostly so future me remembers why things are configured the way they are.

## Finding the time sinks

Tools used through the whole thing:

```bash
systemd-analyze                  # totals
systemd-analyze critical-chain   # what actually gates what
systemd-analyze blame            # unit durations, NOT the critical path, don't trust it blindly
journalctl -b -o short-monotonic # where the real story lives
sudo vclog --msg                 # the firmware's own timestamped log, this one is gold
```

The trick is that `blame` only shows how long units took, not whether anything waited for them. The
`critical-chain` and the monotonic journal show where the seconds actually go.

## The fixes, in order of discovery

### 1. wlan0 autoconnect (~30s!)

The box is ethernet-only but there was a leftover wifi profile. `NetworkManager-wait-online` (`nm-online -s`)
waited its full timeout for a wlan0 that was never going to connect.

```bash
nmcli connection modify netplan-wlan0-<your-ssid> connection.autoconnect no
```

### 2. Services waiting for network for no reason

klipper, moonraker and nginx all had `After=network-online.target`. None of them need it. Single USB MCU, moonraker
binds `0.0.0.0`. After decoupling them the whole printer stack is ready at ~4s, long before the network exists.

Warning: resetting `After=` with an empty assignment in a drop-in did not work for me, I had to edit the unit files
that actually contain the line.

### 3. cloud-init (~3s)

Not needed on a printer.

```bash
sudo touch /etc/cloud/cloud-init.disabled
```

### 4. netplan inside NetworkManager (~3.5s)

RPi OS trixie ships Ubuntu's netplan integration patches. NM profiles live as `/etc/netplan/90-NM-<uuid>.yaml` and
NM re-syncs them through `netplan generate` on every start. The journal showed `generate` running **four times**
(~1.3s each on SD card) during NM startup.

The fix in words:

1. Copy the rendered keyfiles from `/run/NetworkManager/system-connections/` to
   `/etc/NetworkManager/system-connections/` (same UUIDs, chmod 600)
2. Delete the `90-NM-*.yaml` files from `/etc/netplan/`
3. `sudo netplan generate && sudo nmcli connection reload`

The packaging only routes profiles with the `netplan-` filename prefix through netplan, so with zero netplan-owned
profiles the regenerate storms are gone. NM init went 6.6s to 3.0s.

### 5. Ethernet autonegotiation (~4.3s)

Can't make autoneg faster, but nothing says it has to start late. A tiny oneshot brings the link up at ~2s so the
PHY negotiates while the rest of userspace boots:

```ini
# /etc/systemd/system/eth0-early-up.service
[Unit]
Description=Bring eth0 up early so PHY autonegotiation overlaps boot
DefaultDependencies=no
After=sys-subsystem-net-devices-eth0.device
Wants=sys-subsystem-net-devices-eth0.device
Before=network-pre.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/ip link set dev eth0 up

[Install]
WantedBy=sysinit.target
```

wait-online went from 4.3s to ~0.3s. Nothing got faster, it just stopped being serial. My favourite fix of the lot.

### 6. Bootloader probing USB before SD (~5-10s, invisible)

The screen stayed dark until ~30s but `systemd-analyze` said 11s. The missing time was before the kernel. The Pi 4
EEPROM had `BOOT_ORDER=0xf214` (USB first!) and with the SKR mini E3 sitting on the USB bus the bootloader spent
seconds probing it for boot disks on every cold start. Nibbles read right to left:

```bash
sudo rpi-eeprom-config --edit
# BOOT_ORDER=0xf241  (SD first, then USB, then network, then retry)
```

You only see this phase with `sudo vclog --msg`, it never shows up in any OS-side numbers.

### 7. Hardware the printer doesn't have

No display on the Pi, no wifi (ethernet only), no bluetooth, no audio:

```ini
# /boot/firmware/config.txt
dtparam=audio=off
display_auto_detect=0
#dtoverlay=vc4-kms-v3d
disable_splash=1
dtoverlay=disable-wifi
dtoverlay=disable-bt
```

Kernel time went 4.0s to 2.3s. Bonus: `disable-bt` hands the good PL011 UART back to the serial console.

### 8. Klipper waiting for boot targets it doesn't need

With `DefaultDependencies=no` and `After=-.mount systemd-udevd.service` klipper now execs at ~2s into userspace.
It doesn't even need udev to have settled, klippy retries the serial connect on its own (zero retries seen in
practice anyway).

## Measuring honestly

Two things I learned about the measuring itself:

* `vclog --msg` from a warm reboot understates cold boot by ~4s. Cold SDRAM training is real: ARM handoff at
  ~13.4s cold vs ~9.5s warm. Profile the firmware phase from a true power cycle.
* For end to end numbers I used a ping watcher + a Moonraker `/printer/info` poller, and a Tasmota smart plug as
  the power switch. The plug gives an exact "power on" timestamp (and its energy monitor draws a nice 0W to 9W
  boot curve). Also `uptime -s` right after a cold boot is garbage on an RTC-less Pi, don't trust it.

## Where it landed

| | before | after |
|---|---|---|
| cold power-on to stock LCD alive | ~53s* | **~24s** |
| OS boot (kernel + userspace) | ~19s | **~9s** |
| printer stack (klipper/moonraker/nginx) ready | ~18s | **~4s** |

\* the original 53s also included a slicer/cloud-init mess that was its own story.

What's left and why I stopped: ~8s of cold silicon init (boot ROM + SDRAM training, immovable), ~2.5s of SD reads
in the firmware phase (a faster A2 card would shave 1-2s), NetworkManager's 2.7s daemon init (systemd-networkd
would do it in ~0.2s but it doesn't gate the screen), and klippy's ~5s config parse + MCU handshake, which is just
what Klipper does before it draws on the LCD. The actual waste is gone.
