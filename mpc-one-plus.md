# MPC One+ under the hood

Unofficial notes on how the Akai MPC One+ works inside: the Linux system, the MPC app, audio, the controller's MIDI, the display, storage and USB. Everything here was measured on a real unit or read from inMusic's own open-source release, and each fact says which.

- **Tested unit:** MPC One+, MPC firmware 2.14, with TheKikGen's SSH-enabled image (see [Getting a shell](#getting-a-shell))
- **Last updated:** 2026-11-12 (see [Changelog](#changelog))
- **Not affiliated with Akai Professional or inMusic.** MPC, MPC One+ and Akai Professional are their trademarks. Opening a shell on your MPC, stopping its app or changing its system is at your own risk and may void your warranty.

## How to read this

Each fact carries a tag:

| Tag | Meaning |
| --- | --- |
| **[measured]** | Seen or measured on the tested One+ (2.14) |
| **[source]** | Read in inMusic's open-source release (kernel, device tree, drivers); not necessarily checked on the device |
| **[reported]** | Published by someone else (linked in [Credits](#credits-and-related-work)), usually for another MPC model or firmware; not yet confirmed on the One+ |
| **[unverified]** | Expected, not yet confirmed |

Unit-to-unit differences are possible, especially in anything analog (levels, knob positions).

**Other MPCs:** inMusic's source splits its standalone products into a 32-bit generation (`mpc-g1`, Rockchip RK3288, which includes the One+) and a 64-bit one (`mpc-g2`, newer models). Part 1 most likely applies to other first-generation units in broad strokes; per-board details in Part 2 (MIDI note numbers, codec, ports) are One+ facts until someone checks another model. Reports from other models are very welcome ([README](README.md#contributing)).

## Contents

- [Part 1: How the stock MPC works](#part-1-how-the-stock-mpc-works)
- [Part 2: Tech specs](#part-2-tech-specs)
  - [System](#system) · [Storage](#storage) · [Services](#services) · [Audio](#audio) · [Controller MIDI](#controller-midi) · [LEDs](#leds) · [Display](#display) · [Touchscreen and power button](#touchscreen-and-power-button) · [USB-B port](#usb-b-port-device-side) · [CV/Gate outputs](#cvgate-outputs) · [Kernel features](#kernel-features) · [Performance headroom](#performance-headroom)
- [Part 3: Lessons learned and good practice](#part-3-lessons-learned-and-good-practice)
- [Part 4: Open questions](#part-4-open-questions)
- [Changelog](#changelog)

---

## Part 1: How the stock MPC works

The MPC One+ is a small Linux computer with a music controller built in. Everything you see and hear comes from one program, the MPC app, running on an ordinary (if trimmed-down) Linux system.

**The computer.** A Rockchip RK3288: four 32-bit ARM cores (Cortex-A17 class) with 2 GB of RAM, a Mali GPU and an internal eMMC drive. On firmware 2.14 it runs a Buildroot-built Linux 5.15 with the PREEMPT_RT real-time patches, glibc 2.32 and a busybox userland. [measured]

**The MPC app.** systemd starts it at boot as `inmusic-mpc.service` (which runs `/usr/bin/az01-launch-MPC`) and restarts it if it exits. It owns the screen, the touchscreen, the audio codec and the controller while it runs. Thread names such as "JUCE Message Th" suggest it is built with the JUCE framework. [measured] A few helper services run beside it (see [Services](#services)).

**The controller is a separate board on internal USB.** Pads, Q-Links, the main encoder, the buttons and their LEDs are not wired to the processor directly. They sit on a controller board that the Linux side sees as a USB MIDI device ("MPC One +", USB ID `09e8:0056`) with three raw MIDI ports. The app reads pads and knobs as MIDI notes and CCs and lights the LEDs by sending MIDI back. The DIN MIDI jacks are the third port of the same device. [measured] The CV/Gate outputs are very likely driven through this board too, since the main board has no DAC for them. [source] On the MPC Live that board is an STM32F401 microcontroller whose firmware ships with the system and is flashed over USB DFU; the One+'s is probably similar. [reported]

**Audio is one stereo codec with analog knobs.** The audio inputs and outputs are a plain I2S codec (ALSA card "ACVA"): one stereo playback stream and one stereo capture stream, 44.1 kHz only, and no software mixer controls at all. The MAIN VOLUME and input gain knobs are analog, and the headphone jack carries the same signal as the main outs. [measured, source]

**The screen is a portrait panel mounted sideways.** The display is an 800x1280 portrait DSI panel turned to landscape in the case; software rotates its picture by 270 degrees. The MPC app draws through the DRM/KMS interface as "DRM master" rather than through the legacy framebuffer. The touchscreen is an ILI2117 on a normal Linux input device. [measured]

**Storage is split three ways.** The system partition is read-only and full. User data, settings overlays and the factory sound content live on an internal ext4 partition (`/media/az01-internal`). The SD card is exFAT at `/media/MPC SDC`. [measured]

**Firmware 2.x and 3.x are different operating systems.** MPC 3.10's open-source release shows a Yocto build with Linux 6.18, glibc 2.39 and systemd 255, against 2.14's Buildroot / Linux 5.15 RT / glibc 2.32. [source] Anything that depends on the system (the SSH image, binaries, paths) should be rechecked after an update. On 3.x the app's service is `acvs.service` and external drives are mounted `noexec`. [reported]

**What inMusic publishes.** inMusic releases the GPL/LGPL parts of the firmware (kernel, device trees, drivers, system libraries) at [github.com/inmusic-software/az0x-open-source](https://github.com/inmusic-software/az0x-open-source) (branch `mpc-g1/3.10` for this generation; the tarballs are Git LFS objects). The MPC app, the controller board's firmware and its pad/LED/CV protocol are not included. [source]

### Getting a shell

All of Parts 2-3 assume SSH access. These notes were made with **TheKikGen's SSH-enabled MPC 2.14 image** from the [MPC-LiveXplore](https://github.com/TheKikGen/MPC-LiveXplore) project; follow that project's own instructions and warnings rather than any summary here. As a root shell you can stop the MPC app, run your own programs, and also break things, so read [Part 3](#part-3-lessons-learned-and-good-practice) first.

---

## Part 2: Tech specs

### System

| Item | Value | |
| --- | --- | --- |
| SoC | Rockchip RK3288, 4 x ARMv7 (CPU part 0xc0d, Cortex-A17 class) | [measured] |
| CPU features | NEON, VFPv3/VFPv4, vfpd32, hardware integer divide (idiva/idivt), LPAE | [measured] |
| RAM | 2 GB (2018 MB reported); about 1.5 GB free with the MPC app running; no swap | [measured] |
| Kernel (2.14) | `5.15.118-inmusic-2023-06-21-p2-rt64`, SMP PREEMPT_RT, armv7l | [measured] |
| Build system (2.14) | Buildroot 2021.02.12 | [measured] |
| C library (2.14) | glibc 2.32 | [measured] |
| Userland | busybox; no gcc, python or ldd; no `pkill` | [measured] |
| Kernel config | `/proc/config.gz` is present: `zcat /proc/config.gz \| grep ...` answers kernel-option questions | [measured] |
| Product code | `ACVA2` (the One+; the original One is `ACVA`). Programs can read it from `/sys/firmware/devicetree/base/inmusic,product-code`; the `compatible` string is `inmusic,acva2` | [source, reported] |
| Device tree (One+) | `rk3288-az05-acva2.dts` ("inmusic,acva2", base `rk3288-az05.dtsi`); the original MPC One is `rk3288-az01-acva.dts` | [source] |
| Other inMusic drivers | `az01-power` (power supply), `az01-fan` (hwmon) | [source] |
| Network | Reaches the internet when connected (apt worked from a chroot) | [measured] |

### Storage

| Mount | Device | Type | Notes | |
| --- | --- | --- | --- | --- |
| `/` | mmcblk0p6 | ext4, **read-only** | 99% full (a few MB free). Never install anything here | [measured] |
| `/media/az01-internal` | mmcblk0p8 | ext4, read-write | 9.8 GB, about 7 GB free on the tested unit. Holds the `/etc` and `/var` overlay upper layers and `system/acvs-content.img`, an ext4 image loop-mounted (loop0) and overlaid on the factory content partition (mmcblk0p7) | [measured] |
| `/media/MPC SDC` | mmcblk1p1 | exFAT | The SD card. exFAT has no symlinks or Unix permissions | [measured] |
| `/tmp` | RAM | | Cleared at reboot | [measured] |

The internal drive is the place for your own files: use a subfolder of your own and back up first if you ever plan to clear factory content.

### Services

| Service / process | What it does | |
| --- | --- | --- |
| `inmusic-mpc.service` | Runs `/usr/bin/az01-launch-MPC` (the MPC app) with `Restart=always`. `systemctl stop inmusic-mpc` stops it cleanly; `systemctl start inmusic-mpc` brings it back | [measured] |
| `az01-script-runner` | Akai's autorun mechanism | [measured] |
| `DeviceControlServer.bin` | Creates the virtual MIDI ports | [measured] |
| `crashpad_handler` | Uploads MPC app crash reports (Sentry) | [measured] |
| Telemetry in the MPC app | Usage analytics (PostHog) besides Sentry crash reports; the "Collect Usage Statistics" preference does not cover crash reporting (analysis of firmware 3.9.1) | [reported] |
| MPC app memory limit | The app refuses commands with "Not enough memory to perform the command" once system RAM use reaches 80% (3.9.1). Anything else you run beside the app eats into that | [reported] |

The MPC app also routes the DIN MIDI port through an ALSA sequencer client (client 16) on its side. [measured]

### Audio

**Devices**

| Item | Value | |
| --- | --- | --- |
| ALSA card | card 1, id `ACVA`, device `hw:1,0`: 1 playback + 1 capture | [measured] |
| Driver | machine `sound/soc/rockchip/rockchip_inmusic_jp07.c` + codec `sound/soc/codecs/inmusic-jp07.c` (GPL v2). A plain I2S codec controlled by three GPIOs (enable, reset, mute); MCLK = 256 x fs | [source] |
| Sample rate | **44,100 Hz only** | [measured, source] |
| Channels | 2 out, 2 in | [measured, source] |
| Formats | S16_LE, S24_LE, S32_LE; RW or MMAP interleaved | [measured] |
| Period / periods | 32 to 65,536 frames; 2 or more periods | [measured] |
| Mixer controls | **None** (`/dev/snd/controlC1` lists 0 controls). No input gain, output volume or headphone route in software | [measured, source] |
| Headphones | Same stream as the main outs; the MAIN VOLUME knob sets both | [source] |
| Power-up | Codec enable, ~500 ms wait, reset release; held muted until a stream runs; kept biased between streams (`idle_bias_on`) | [source] |
| Real-time | SCHED_FIFO threads allowed as root on the RT kernel | [measured] |
| Card 0 | The controller's USB-Audio device ("One", "MPC One +"), MIDI only for our purposes | [measured] |

`plughw:1,0` works too, but asking it for 48 kHz makes ALSA resample to 44.1 kHz. Open `hw:1,0` at 44,100 Hz to avoid that. [measured]

**Round-trip latency** (cable from MAIN OUT L/R to INPUT L/R, chirp pings found with a matched filter; tool: [MPDJ's latency probe](https://github.com/longcutter/mpdj/blob/mpc-port/platforms/mpc/source/audio/LatencyDetect.cpp)) [measured]

| Setup | Round trip | Notes |
| --- | --- | --- |
| Raw ALSA, 3 x 128 frames, capture open, 2 periods prefilled | **297 frames (6.73 ms)**, same on L and R, identical over 8 pings | 256 frames of buffer + **~41 frames (~0.9 ms) of converter delay**. A later run gave 296: allow +-1 frame between runs |
| JACK 1.9.21, `-p 128 -n 3` | 553 frames (12.54 ms) | JACK reports capture 128 + playback 384; + the same ~41 converter frames |
| Through a reverb pedal (analog dry path) | 296 frames | The pedal's dry path added nothing measurable |

The ~41-frame converter delay is the useful hardware number: add it to whatever buffering your own program uses.

**Loopback level vs the knobs** (probe ping at -12 dBFS, cable loopback; knob positions are approximate) [measured]

| MAIN VOLUME | Input gain | Return vs sent |
| --- | --- | --- |
| very low | 1/2 | -49 dB |
| 1/4 | 1/2 | -30 dB |
| 3/4 | 1/2 | **-6 dB** |
| 3/4 | 3/4 | +5.5 dB (a full-scale signal would clip) |

Both knobs are steep: 35 dB separates the last two rows. MAIN 3/4 with input 1/2 is a good starting point for re-amping through pedals (the loudest peaks come back around -6 dBFS). Turn MAIN back down before listening: it also sets the headphone level.

### Controller MIDI

ALSA card 0, USB-Audio "MPC One +" (USB `09e8:0056`), on an internal USB host controller (`ff540000.usb`) at full speed. [measured]

| Raw MIDI port | Name | Use |
| --- | --- | --- |
| `hw:0,0,0` (`/dev/snd/midiC0D0`, subdevice 0) | MPC Studio Live Public | Sent nothing during capture |
| `hw:0,0,1` (`/dev/snd/midiC0D0`, subdevice 1) | MPC Studio Live **Private** | **All pads, knobs and buttons arrive here; LEDs are written here** |
| `hw:0,0,2` (`/dev/snd/midiC0D0`, subdevice 2) | MPC Studio Live MIDI Port | The DIN MIDI jacks |

The port names say "MPC Studio Live" on the One+, the same as the original MPC One; the Live, Live II, X and Key 61 use "MPC Public" / "MPC Private" / "MPC MIDI Port A-D". [measured, reported] The Public port carries SysEx in both directions (pad colours, identity); buttons, pads and button LEDs use Private. [reported]

**Identity and product ID.** The controller answers a universal identity request (`F0 7E 00 06 01 F7`) with Akai's ID 0x47, a product ID and its serial number. Product IDs used in Akai's SysEx: 0x3A MPC X, 0x3B Live, 0x40 Force, 0x46 One, 0x47 Live II, 0x4B Key 61, 0x52 XL, **0x56 One+ (ACVA2)**. [reported]

**Pads: channel 10.** Note on `0x99` with velocity, note off `0x89`, pressure as poly aftertouch `0xA9` on the same note. Bank A layout (pad 1 = bottom left): [measured]

| Pad | Note | | Pad | Note |
| --- | --- | --- | --- | --- |
| 1 | 37 (0x25) | | 9 | 48 (0x30) |
| 2 | 36 (0x24) | | 10 | 47 (0x2F) |
| 3 | 42 (0x2A) | | 11 | 45 (0x2D) |
| 4 | 82 (0x52) | | 12 | 43 (0x2B) |
| 5 | 40 (0x28) | | 13 | 49 (0x31) |
| 6 | 38 (0x26) | | 14 | 55 (0x37) |
| 7 | 46 (0x2E) | | 15 | 51 (0x33) |
| 8 | 44 (0x2C) | | 16 | 53 (0x35) |

The same numbering is in TheKikGen's code for the MPC Live, so it is shared across models. [measured, reported]

The bank buttons A-D are ordinary buttons (below). Whether the pads send other notes on banks B-D, or the app does the bank switching itself, hasn't been checked. [unverified]

**Q-Links: channel 1, relative encoders, touch-sensitive** (Q-Link 1 = top knob, 4 = bottom) [measured]

| Q-Link | Touch note | Turn CC |
| --- | --- | --- |
| 1 | 84 (0x54) | 16 (0x10) |
| 2 | 85 (0x55) | 17 (0x11) |
| 3 | 86 (0x56) | 18 (0x12) |
| 4 | 87 (0x57) | 19 (0x13) |

- Touch = note on `0x90` / note off `0x80`
- Turn value is two's complement: `0x01-0x3F` = steps one way, `0x41-0x7F` = steps the other way. A slow turn sends 1; faster turns send 2-3 per message
- The detent-free knobs send many messages per turn: dividing by about 6-12 messages per step feels right for menu navigation (MPDJ uses 6-12 depending on the job)

**Main encoder: channel 1.** Turn = CC 100 (`0x64`), relative, same encoding as the Q-Links. Push = note 111 (`0x6F`). [measured]

**Buttons: channel 1 notes, velocity `0x7F` = press, `0x00` = release** [measured]

| Button | Note | | Button | Note |
| --- | --- | --- | --- | --- |
| Q-LINK | 0 | | SHIFT | 49 |
| PROGRAM EDIT | 2 | | BROWSE | 50 |
| SAMPLE EDIT | 6 | | MAIN | 52 |
| ERASE | 9 | | TAP TEMPO | 53 |
| NOTE REPEAT | 11 | | + | 54 |
| TC | 12 | | - | 55 |
| A | 35 | | UNDO | 67 |
| B | 36 | | REC | 73 |
| C | 37 | | OVER DUB | 80 |
| D | 38 | | STOP | 81 |
| FULL LEVEL | 39 | | PLAY | 82 |
| 16 LEVELS | 40 | | PLAY START | 83 |
| STEP SEQ | 41 | | TRACK MIX | 116 |
| NEXT SEQ | 42 | | COPY | 122 |
| TRACK MUTE | 43 | | MENU | 123 |
| | | | SAMPLER | 125 |

Button notes overlap pad notes (e.g. PLAY and pad 4 are both note 82); the channel tells them apart. [measured]

### LEDs

LEDs are lit by sending MIDI **back to the Private port** (open `/dev/snd/midiC0D0` read-write on subdevice 1, the same handle you read input from). Value 127 = on, 0 = off. Mapped with [MPDJ's LED probe](https://github.com/longcutter/mpdj/blob/mpc-port/platforms/mpc/source/input/LedProbe.cpp). [measured]

| LEDs | Message | |
| --- | --- | --- |
| Q-Link mode LEDs 1-4 (above the Q-LINK button) | CC 90, 91, 92, 93 on channel 1 | [measured] |
| Button LEDs | CC on channel 1 whose number **equals the button's note number** (table above) | Confirmed for PLAY 82 (green), OVER DUB 80 (red), REC 73 (red), UNDO 67, TAP TEMPO 53; others expected to follow the same rule [unverified] |
| Button LED values | 0 = off, 1 = dim, 3 = bright; the MPC app also sends 2 to some buttons (meaning not documented). It blinks TAP TEMPO by alternating 1 and 3. 127 lights them on the One+ | [reported; 0 and 127 measured] |
| Pad RGB LEDs | SysEx **on the Public port** (`hw:0,0,0`): `F0 47 7F <product ID> 65 00 04 <pad> <R> <G> <B> F7`, R/G/B 0-127. On the One+: `F0 47 7F 56 65 00 04 ...`. Pad 0 = pad 1 (bottom left), counting left to right then up: pad index = MPC pad number - 1 | [reported] |

Untested on the One+ so far: the pad SysEx and the 1/2/3 values.

### Display

| Item | Value | |
| --- | --- | --- |
| Panel | DSI (`card0-DSI-1`), native 800x1280 **portrait**, mounted as 1280x800 landscape | [measured] |
| Rotation | 270 degrees: logical landscape `(x, y)` -> panel `(px, py) = (y, 1279 - x)` | [measured] |
| DRM | `/dev/dri/card0`, `DRM_ROCKCHIP=y` | [measured] |
| fbdev | `/dev/fb0` (rockchipdrmfb emulation), 32 bpp, stride 3,200 bytes, virtual 800x3840 = 3 pages for page-flipping by panning | [measured] |
| What the MPC app uses | DRM directly, as DRM master ("MPC Main Thread"), scanning out an XR24 800x1280 buffer on plane 0, unrotated | [measured] |
| GPU | Mali with the vendor kbase driver (`/dev/mali0`, root only); vendor `libmali.so.14` (built against glibc 2.32) bind-mounted into `/usr/lib`; libEGL, libGLESv2, libdrm and libfreetype present; **no SDL** | [measured] |
| Backlight | `/sys/class/backlight/mipi-backlight` | [measured] |

Who holds the display and what it shows: `/sys/kernel/debug/dri/0/clients` and `/sys/kernel/debug/dri/0/state` (debugfs). [measured]

### Touchscreen and power button

| Device | What | |
| --- | --- | --- |
| `/dev/input/event0` | ILI2117 touchscreen, multi-touch protocol; axes 0..2048 on both X and Y. Panel axes match the framebuffer (no swap or invert needed on the tested unit) | [measured] |
| `/dev/input/event1` | gpio-keys: the power button (`KEY_POWER`) | [measured, source] |

Touch to landscape screen: scale raw values to panel pixels (x * 800 / 2048, y * 1280 / 2048), then undo the 270-degree rotation above.

### USB-B port (device side)

| Item | Value | |
| --- | --- | --- |
| Controller | `ff580000.usb`, DesignWare dwc2 OTG, `CONFIG_USB_DWC2_DUAL_ROLE=y`, `maximum_speed` high-speed; `dr_mode` peripheral in the device tree | [measured, source] |
| Gadget support (2.14) | Built into the kernel (no modules to load): `USB_CONFIGFS` with UAC2 audio, MIDI, HID, FunctionFS, mass storage, serial/ACM, NCM/ECM/RNDIS/EEM, printer, loopback. UAC1 off. inMusic's own `uac2_az01` function exists but is off | [measured] |
| configfs | Mounted at `/sys/kernel/config`; `usb_gadget` is empty | [measured] |
| Standalone mode | **The MPC app builds no gadget.** With the app running and the port cabled to a Mac, the UDC stayed "not attached" and the Mac saw nothing. A program of your own can create its own gadget through configfs without tearing anything down | [measured] |
| Controller mode | Builds a UAC2 gadget (reported by MPC-LiveXplore users; not checked here) | [unverified] |
| Clock steering | Mainline Linux 5.15 (including 5.15.118) has only the **"Capture Pitch 1000000"** control (it steers the computer -> MPC direction through the feedback endpoint); **"Playback Pitch" arrived in 5.16**. So on 2.14, MPC -> computer audio can't be steered by the gadget and needs resampling on the MPC, unless inMusic backported it (not checked). 6.18 (3.10) has both | [source] |

inMusic's `f_uac2_az01.c` (in the 3.10 source) can lock the USB audio stream to the codec's period timing via `sound/core/pcm_sync_stream.c`; it's GPL kernel code, worth reading for ideas. [source]

### CV/Gate outputs

| Item | Value | |
| --- | --- | --- |
| Jacks | 8 outputs on 4 x 3.5 mm TRS: tip/ring = 1/5, 2/6, 3/7, 4/8. A mono TS cable gets the tip only | [measured] |
| How they're driven | Not through the audio codec (one stereo playback stream only). The One+ main board's device tree has no DAC, SPI, IIO or CV-named GPIO, so almost certainly through the controller board over USB | [source] |
| Message format, voltage range, resolution | Not yet probed. Reports for the MPC X say 0-5 V unipolar | [unverified] |

### Kernel features

From `/proc/config.gz` and `/dev` on 2.14. [measured]

| Feature | Present? |
| --- | --- |
| PREEMPT_RT | Yes |
| Filesystems | ext2/3/4, squashfs, overlay, vfat, exfat, fuse, hfsplus, iso9660, udf |
| Loop devices | Yes, built in; `/dev/loop0-7` (loop0 used by inMusic's content image); `losetup`, `chroot`, `mount` in busybox |
| Namespaces, cgroups | Yes (cgroups v1 and v2) |
| ALSA sequencer (`SND_SEQ`) | **No** (raw MIDI only) |
| ALSA loopback (`SND_ALOOP`) | **No** |
| `BINFMT_MISC` | No |

### Performance headroom

Rough numbers to plan with, all at 44.1 kHz, 128-frame blocks (2,902 us each). [measured]

| Workload | Result |
| --- | --- |
| A full tracker engine (Polyend Tracker CE port, sample playback, 8 tracks, effects) | ~110-200 us average per block (~5% of one core); max ~510 us |
| SuperCollider 3.13 (scsynth on JACK 128 x 3) | Sine 2.5% of one core; 8 voices of ~18 UGens + 2 looping buffer players 12%; ~1.06% per extra voice; 72 voices at 81.7% with 0 xruns over 300 s |
| JACK 1.9.21 at 128 x 3 on `hw:1,0` | 0 xruns over 300 s with that load; DSP load 7-9% with a tracker client |
| A second process feeding audio through shared memory + futex wake per block | 0 underruns at 128 frames with the producer at SCHED_FIFO 60; wake latency avg 11 us, max 40 us |
| Same producer at normal priority under heavy load | Occasional underruns from pre-emption: real-time producers need SCHED_FIFO |
| Streaming a multi-minute song from the SD card | Imports in ~5 s, plays clean |

Three cores are essentially idle while a single-threaded audio program runs.

---

## Part 3: Lessons learned and good practice

### Working on the device

- **Stop the MPC app before using the screen, controls or audio:** `systemctl stop inmusic-mpc`, and `systemctl start inmusic-mpc` when you're done. While it runs it is DRM master (anything you draw doesn't show), and it holds the codec and the controller port
- **Never write to `/`.** It's read-only and full anyway. Put your files in a folder of your own on `/media/az01-internal`
- **busybox, not a full userland:** there's no `pkill` (use `kill $(pidof <name>)`), no compiler, no python, no `ldd`. Check a tool exists before a script relies on it
- **A crash can look like a freeze.** If your program segfaults while drawing, the last frame stays on screen and the controls stop answering; the terminal may not print anything. Debug crashes on a host (QEMU user mode + gdb works well) rather than on the device
- **Copying files from a Mac:** `tar` and `cp` leave `._name` AppleDouble files on exFAT; set `COPYFILE_DISABLE=1` before `tar`. Convert audio to 16-bit 44.1 kHz WAV first (`afconvert -f WAVE -d LEI16@44100 in.aif out.wav`). Check the file size after copying: a mistyped source path makes an empty file
- **Don't send the controller unknown SysEx.** One command (0x72) puts the controller board into DFU firmware-update mode (reported for the Live); stick to the documented messages
- **Firmware updates:** 3.x is a different OS from 2.x. Don't update a modded unit until the mod you rely on is known to work on the new version

### Building programs for it

- **Cross-compile.** Target armv7 hard-float (`arm-linux-gnueabihf`, `-mcpu=cortex-a17` or a15). Either link against **glibc 2.31 or older** (2.14 has 2.32; Debian bullseye's toolchain or `zig cc -target arm-linux-gnueabihf.2.31` work) or build **fully static against musl**
- **zig as the cross-compiler** worked well from macOS and Linux with no sysroot to manage
- **musl gotcha:** in static musl programs `sched_setscheduler` always fails (ENOSYS). Use `pthread_setschedparam(pthread_self(), SCHED_FIFO, ...)` instead
- **No libasound needed.** Raw ALSA ioctls on `/dev/snd/pcmC1D0p` / `pcmC1D0c` and `/dev/snd/midiC0D0` are enough, which keeps binaries self-contained
- **A full Debian userland works beside the stock system.** A Debian 12 (bookworm) armhf root in a plain folder on the internal drive, entered with `chroot` after bind-mounting `/dev`, `/dev/pts`, `/dev/shm`, `/proc` and `/sys`, ran `apt` natively on the MPC (about 660 MB with JACK and SuperCollider installed). Keep it off the SD card (exFAT can't hold a Linux root). Notes for that root: JACK needs `JACK_NO_AUDIO_RESERVATION=1` (no D-Bus); SuperCollider's `sclang` as root needs `QT_QPA_PLATFORM=offscreen` and `QTWEBENGINE_DISABLE_SANDBOX=1`

### Audio

- Open **`hw:1,0` at 44,100 Hz** directly; there is no other rate and `plughw` would resample
- **3 periods of 128 frames** ran with 0 xruns over long sessions with a tracker engine at SCHED_FIFO 70. With capture open and the capture read pacing the loop, prefilling 2 periods of silence gives a 297-frame round trip
- 2 periods of 128 also play clean in steady state, but our implementation got one xrun at start-up; not solved yet
- **JACK at 128 x 3 is clean** on this kernel; it costs 256 frames (5.8 ms) more round trip than a raw ALSA loop at the same period size
- **There are no mixer controls.** Any gain, volume or input level a program shows has to be done in software; the user still has to set the analog knobs. Measure the return level and guide them to the knobs (see the [level table](#audio))
- **The headphones carry the main outs.** You can't send something to the headphones only, so never mix a re-amped return into the outs while printing through pedals, or it feeds back

### Controller MIDI

- **Select the Private subdevice before opening the port.** Without alsa-lib: open `/dev/snd/controlC0`, `ioctl(SNDRV_CTL_IOCTL_RAWMIDI_PREFER_SUBDEVICE, &(int){1})`, then open `/dev/snd/midiC0D0` read-write on the same thread. That is how alsa-lib picks a subdevice
- **Use a running-status MIDI parser** and tell pads (channel 10) from buttons (channel 1) by channel, since their note numbers overlap
- **No ALSA sequencer** on this kernel, so programs that expect one (SuperCollider's `MIDIIn`, JACK's seq bridge) need raw MIDI, OSC or a relay instead
- When handing the controller from one program to another, a relay that keeps the port open and forwards bytes over a pipe made hand-overs instant and let the parent watch for an escape combo even if the child hung

### Display

- **`/dev/fb0` works** once nothing else is DRM master: write the 270-degree-rotated image and pan `yoffset` to flip pages
- **Only redraw what changed.** Copying a full 800x1280 frame costs real time; comparing frames in 32x32 tiles and writing only changed tiles kept a busy UI at ~30 fps with the main loop free. Compare rows in memory order (tile-major comparison was 3x slower on the device)
- **Plain DRM works** too: `SET_MASTER`, `SETCRTC` with dumb buffers, CPU drawing, `PAGE_FLIP`, including padded pitches
- **SDL2's KMSDRM backend does not draw** with Debian bookworm's Mesa (22.3, software GL): SDL becomes master and the panel shows its buffer, but the buffer stays black. Untested ways round it: an X server on `/dev/fb0`, or a newer Mesa
- **Check who owns the display** with `/sys/kernel/debug/dri/0/clients` before assuming your program's drawing is wrong

### USB

- **The USB-B port is free in standalone mode.** Your program can build a configfs gadget (e.g. UAC2 + MIDI) on `ff580000.usb` and should remove it on exit, so the MPC app starts clean afterwards

---

## Part 4: Open questions

Help welcome on any of these, from One+ owners or other models.

- Confirm the pad colour SysEx with product ID 0x56 on the One+, and what button LED value 2 does (second colour on two-colour buttons?)
- The identity reply's product ID on the One+ (expected 0x56)
- CV/Gate: transport and message format, voltage range, resolution, whether outputs hold when the app exits
- Whether inMusic's 5.15 kernel backported "Playback Pitch" (mainline 5.15 has only "Capture Pitch"); link speed once bound; how macOS and Windows enumerate a gadget
- Input level: are the inputs usable for guitar at instrument level, or line only
- Polyphonic aftertouch resolution and the pads' velocity curve
- The same tables for MPC One, Live, Live II, X, Key 61 and other first-generation units: which note numbers and devices differ

## Credits and related work

- **TheKikGen**, [MPC-LiveXplore](https://github.com/TheKikGen/MPC-LiveXplore) and its [wiki](https://github.com/TheKikGen/MPC-LiveXplore/wiki): the SSH image these notes depend on; the controller's port layout, identity request, pad colour SysEx and DFU mode ([MPC internal controller, MIDI aspects](https://github.com/TheKikGen/MPC-LiveXplore/wiki/MPC-internal-controller---Midi-aspects), which credits Lenore Byron for decoding SysEx commands); [inMusic device IDs](https://github.com/TheKikGen/MPC-LiveXplore/wiki/Akai--InMusic-device-Ids); [MPC 3 first analysis](https://github.com/TheKikGen/MPC-LiveXplore/wiki/MPC-3-first-analysis); the firmware 3.9.1 privacy and memory-limit analyses on the same wiki
- **TheKikGen**, [MPCLiveXplore-libs](https://github.com/TheKikGen/MPCLiveXplore-libs) (CC0): the product code and SysEx ID table including the One+ (ACVA2, 0x56), the pad colour message and pad numbering, button LED values
- **bcrowe306**, [MPC-One-Control-Mode-MIDI-Protocol](https://github.com/bcrowe306/MPC-One-Control-Mode-MIDI-Protocol): the MIDI map the MPC One uses in *controller mode* toward a computer, which is different from the internal map above
- **inMusic**, [az0x-open-source](https://github.com/inmusic-software/az0x-open-source): kernel, device trees and drivers
- **Linux kernel** source ([torvalds/linux](https://github.com/torvalds/linux), [gregkh/linux](https://github.com/gregkh/linux) for 5.15.118): which versions have the UAC2 Pitch controls

## Changelog

- **2026-11-12 (later):** Added findings from other projects, tagged [reported] and credited: pad colour SysEx and the One+'s product ID, button LED values, identity request, per-model port names, controller MCU and DFU warning, MPC app memory limit and telemetry, 3.x service and mount changes. UAC2 Pitch controls answered from kernel source. New Credits section

- **2026-11-12:** First version, from testing done while building [MPDJ](https://github.com/longcutter/mpdj) (2026-11-05 to 2026-11-12)
