# Raspberry Pi Zero 2 W

The Zero 2 W uses the same SoC as the Pi 3 (RP3A0: quad Cortex-A53, 64-bit), so the
standard arm64 Stratux image boots on it unchanged. Two things set it apart from the Pi 3/4
Stratux is usually run on: it has **512 MB of RAM** and **one micro-USB OTG port**. This page
covers what the code and image do about that, and how to set one up so it runs well.

## What the image does for the Zero 2 W

| Change | Where | Why |
|---|---|---|
| `gpu_mem=16` under a `[pi02]` filter | `image_build/stage2/10-stratux/files/config.txt` | Stratux is headless. The firmware default of 64 MB for 512 MB boards wastes almost 10% of RAM. Other Pi models are not affected. |
| tmpfs root overlay capped at **min(250 MB, RAM/4)** | `image_build/stage2/10-stratux/files/init-overlay` | The read-only root's writable layer lives in RAM (`/var/log`, `stratux.log`, the SQLite replay log and traces all go there). The fixed 250 MB cap could fill about half of a Zero's memory and push the decoders into OOM. On a Zero 2 W the cap works out to about 110 MB. Boards with ≥1 GB keep 250 MB. |
| `dtoverlay=dwc2,dr_mode=host` (already present) | `config.txt` | Forces the Zero's only USB port into host mode, which the SDRs and GPS need. |
| `arm_64bit=1` (already present) | `config.txt` | The Zero 2 W supports it. The `ogn-rx-eu` prebuilt and the bundled `.deb`s are arm64-only. |

These files ship only with the **SD-card image**, not the `.deb`/OTA update. To apply them to
an existing install, see [Updating an existing install](#updating-an-existing-install).

## Recommended configuration

**Radios: at most two SDRs.** 1090 ES + 978 UAT (US) or 1090 ES + OGN (EU) is the realistic
limit. Each RTL-SDR streams about 4–5 MB/s over the single shared USB 2.0 link, and every
decoder competes for the same four 1 GHz cores. For a third band, put it on the decoder that
has the least work:

- Use an OGN tracker (T-Beam etc., [ogn-ais-receivers.md](ogn-ais-receivers.md)) instead of
  an OGN SDR. `ogn-rx-eu` is the heaviest decoder.
- Use a uAvionix Ping/Pong or a UATRadio, which decode on the device itself.
- Leave AIS (`rtl_ais`) off unless you need it.

**USB: a powered hub, not a passive OTG splitter.** The Zero has one data port, so use a
micro-USB OTG adapter into a hub. Two SDRs plus a GPS draw roughly 600–800 mA, which is more
than the Zero's 5 V rail can comfortably pass through while the CPU is busy. Brown-outs show
up as SDRs dropping off the bus and `dump1090` restarting. Either use a powered hub, or a
hub that back-feeds the Zero from a 5 V / 2.5 A+ supply.

**Cooling: a heatsink is mandatory, a fan is recommended.** The firmware throttles the CPU at
80 °C. A bare Zero 2 W in an enclosed case with the decoders running gets there easily, and
when it throttles, messages are dropped. `fancontrol` works unchanged (PWM on BCM 18).

**Wi-Fi: 2.4 GHz only, single antenna.** This matches what Stratux uses anyway (AP channels
1–11). In AP+Client mode the AP has to follow the client network's channel, because the
CYW43436 chip runs only one channel at a time. Keep the hub and SDRs away from the board
antenna, since USB 3 hubs and SDR spurs both land near 2.4 GHz.

**Settings to leave off in flight:** *Replay Log* (SQLite), *Trace Log* and *AHRS log* all
write into the RAM overlay and use CPU. Turn them on only for debugging, and clear logs
afterwards.

## Building

Don't build on the Zero itself: Go plus the C submodules needs more than 512 MB and takes a
very long time. Cross-build the arm64 `.deb` from any host with Docker:

```sh
git submodule update --init --recursive
make ddpkg
```

then install it via the web UI's update page, or build a full image with
`image_build/build.sh` (see [building.md](../building.md)).

## Updating an existing install

The image-level changes above are not part of the `.deb`. On a Zero 2 W already running
Stratux:

```sh
# /boot/firmware is a normal vfat mount, not part of the overlay
cat >> /boot/firmware/config.txt <<'EOF'

[pi02]
gpu_mem=16
[all]
EOF

# optional: the overlay size cap. With the overlay active, write to the lower layer.
overlayctl unlock
cp init-overlay /overlay/robase/sbin/init-overlay   # from image_build/stage2/10-stratux/files/
overlayctl lock

reboot
```

## Checking it runs well

```sh
vcgencmd get_throttled   # 0x0 = no under-voltage or throttling since boot
vcgencmd measure_temp
free -m                  # "available" should stay well above ~100 MB
df -h /overlay/rwdata    # RAM overlay usage
top                      # dump1090 / stratuxrun / ogn-rx-eu CPU share
```

In the web UI, the status page shows CPU temperature and free "disk" space. On an overlay
system, that "disk" figure is the RAM overlay, so watch it fall if logging is enabled.
