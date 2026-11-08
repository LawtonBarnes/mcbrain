# McBrain

> **Overview of the whole fleet.** Every app except AIR METAL, IDENTIFY, STRINGS and SCRUTE also runs standalone on a single Pi; see each app's README.

A 6-Raspberry-Pi fleet driving 6 CRTs with retro-styled kiosk apps (test
patterns, an audio spectrum visualizer, a sports/news ticker, a weather
station, an MP3 player, and an ADS-B flight radar), centrally monitored
and controlled from one dashboard machine. This repo is the **hardware
and fleet-architecture overview** -- each app has its own repo with its
own install steps; this one is where they fit together.

![The stack](./img/STACK.jpg)

## What this actually is

Five of the six Pis (`masterofpuppets` + `puppet1`-`4`) sit stacked in
one case as "McBrain," each wired to its own CRT over composite video. A
sixth Pi, `production`, is a standalone unit used for development and
also runs as a full peer in the fleet. One Pi (`masterofpuppets`, or
"MP") is the interactive control point -- it has a USB remote attached
and runs a live dashboard; the other five ("puppets") are headless
appliances with no local input, each running exactly one assigned app
continuously and reporting their health back to MP over the network.

Nobody has to touch a puppet directly to change what it's doing --
MP's dashboard can remotely reassign any puppet to any installed app,
watch its live stats, and relay live input to it, all from one physical
remote control in front of one CRT.

## Hardware

### Per-unit (all 6 machines)

- Raspberry Pi 3B+, Raspberry Pi OS Bookworm (Debian 12), 32-bit
  aarch64, kernel `6.12.93+rpt-rpi-v8`
- MicroSD card (any brand that survives daily power cycling reasonably
  well -- see **Power handling** below on why this matters)
- Composite video out (the RCA jack on the A/V cable) to its own CRT --
  NTSC, 720x480, 16bpp RGB565 framebuffer
- Line-level analog audio out (same A/V cable's audio RCA) into an
  external amp -- volume is controlled externally, not by any app
- `metalshop` user, passwordless `sudo`, one shared SSH key across the
  fleet

### Machine-specific extras

| Machine | Extra hardware | Why |
|---|---|---|
| `masterofpuppets` (MP) | USB remote receiver (2.4GHz, HID) | The one interactive control point |
| `puppet1`-`4` (P1-P4) | USB microphone ("USB PnP Sound Device" class) | LOUDNESS (audio spectrum visualizer) |
| `puppet4` (P4) specifically | RTL-SDR USB dongle (RTL2832U) + antenna | JOAN JET (ADS-B flight radar) |
| `production` | (none required) | Dev/build machine; also runs as a real fleet peer |

### The remote control

A cheap 2.4GHz wireless remote (identifies over USB as "XING WEI 2.4G
USB" / similar Nordic Semiconductor-chipset units are common under this
description). It enumerates as several separate HID sub-devices (a
plain keyboard interface, a Consumer Control interface for Home/Volume,
a System Control interface for Power) rather than one -- every app's
input-handling code already accounts for this by grabbing every
`/dev/input` device with an `EV_KEY` capability rather than assuming a
single device, so a different remote should mostly just work as long as
you re-map its actual keycodes (capture them with a live `evdev`
listener -- don't guess from the box).

### Physical stack + cooling

MP/P1-P4 sit stacked top-to-bottom in one open-sided case. **Stacking 5
boards with heatsinks in a closed vertical run creates a real
middle-of-the-stack thermal problem** -- the top board vents freely and
the bottom draws cool ambient air from below, but boards in the middle
are sandwiched and accumulate heat from both directions, hitting ARM
throttling under load. A single shared USB fan blowing across the whole
stack was enough to drop every board 15-20°C and clear all throttling
-- if you're building a similar stack, budget for active cooling from
day one rather than discovering this after the fact. (This project is
mid-upgrade from one shared fan to a built-in fan on each layer, for a
cleaner and more even result.)

### Custom boot splash

All 6 machines share a custom Plymouth boot splash (replacing the stock
Raspberry Pi rainbow-square theme) via the `pix` theme's
`/usr/share/plymouth/themes/pix/splash.png`, rebuilt into both
initramfs images with `sudo plymouth-set-default-theme -R pix` after
replacing the image. Purely cosmetic, but a nice touch for a kiosk fleet
where the boot sequence is visible on the actual CRT.

## Network

All 6 machines sit on the same LAN, addressed by hostname via a shared
`~/.ssh/config` (short aliases `mp`, `p1`-`p4`, `pr`) on whatever
machine is doing fleet administration (this repo's tooling assumes SSH
key auth + `sudo` access to all 6). **Static IPs or DHCP reservations
are strongly recommended** -- most of this fleet's software (the
puppet-address table in `scrutinizer.py`/`sync-fleet.sh`, described
below) refers to puppets by IP, not hostname/mDNS, so an IP that drifts
after a router reset or power event silently breaks fleet-wide
monitoring and control for whichever puppet moved, without erroring
loudly. If a puppet ever goes silent, sweep the LAN's ARP table for the
Raspberry Pi Foundation's `b8:27:eb` MAC prefix and confirm identity via
`ssh <ip> hostname` before assuming a machine is actually down.

## Software architecture

Two roles, one shared HTTP protocol between them:

- **STRINGS** (`github.com/LawtonBarnes/strings`) runs on every puppet
  (P1-P4, and `production`) as a `systemd` service -- it renders
  nothing itself, just supervises whichever app is currently assigned
  to that machine (relaunching it on crash, on boot, or on a fleet sync
  picking up new code) and exposes an HTTP API on port 8420 for status
  and remote control. A puppet's assignment persists locally
  (`state.json`), so it comes back running the same app after a power
  cycle even if MP is briefly unreachable.
- **CENTRAL SCRUTINIZER / SCRUTE** (`github.com/LawtonBarnes/scrutinizer`)
  runs only on MP, launched straight from console autologin (`tty1`) so
  MP boots directly into a live dashboard with zero manual steps. It's
  the one thing a person actually interacts with: cycle through
  LOCAL/each puppet's live stats, remotely reassign any puppet to any
  installed app, relay the physical remote's input live to whichever
  puppet is selected, and trigger a fleet-wide power action.

### The apps (each assignable to any puppet via STRINGS)

| App | Repo | What it needs |
|---|---|---|
| BARS | `github.com/LawtonBarnes/bars` | Nothing extra |
| LOUDNESS | `github.com/LawtonBarnes/loudness` | A USB mic (P1-P4 have one each) |
| CHANNEL 38 | `github.com/LawtonBarnes/channel38` | Internet access |
| WEATHERSTAR 4000 | (not a separate repo -- built on [ws4kp](https://github.com/netbymatt/ws4kp)) | Internet access, more RAM/CPU headroom (runs Chromium) |
| bebop | `github.com/LawtonBarnes/bebop` | An MPD server + music library |
| JOAN JET | `github.com/LawtonBarnes/joanjett` | An RTL-SDR dongle (P4 only) |

Each repo's own README covers install steps for that app specifically.
STRINGS validates assignment requests against an allow-list of what's
actually installed on that machine, so a puppet missing an app's
dependencies just reports it as unavailable rather than crashing.

## Fleet operations

- **`sync-fleet.sh`** (in the `scrutinizer` repo): SSH's into every
  machine, `git pull --ff-only`s whatever apps are installed there
  (not just the currently-running one), and restarts via STRINGS's
  `/restart` only if the pulled repo matches the currently-assigned
  app. Run this from whatever machine holds fleet-wide SSH access --
  it shells out to `ssh`/`curl`, it isn't itself deployed to the fleet.
- **Deploy keys**: each repo pushes via its own repo-scoped GitHub
  deploy key, held by whichever single machine is the "edit here" host
  for that repo, rather than one broad personal access token shared
  everywhere.
- **Power handling**: a graceful `sudo reboot`/`shutdown -r` is
  **unreliable** on this fleet -- it can hang indefinitely on Plymouth's
  reboot-transition splash, sometimes needing a physical power cycle to
  recover. A **soft poweroff** (`shutdown -h now`, or the remote's
  Power-button dialog) is reliable across every machine tested. The
  standing nightly routine is: soft-poweroff all 6, confirm they've
  gone dark, *then* cut power at the strip -- never cut live power on a
  running Pi if it can be helped, since that carries real (if usually
  small) SD-card filesystem-corruption risk.
- **Mic calibration**: LOUDNESS's `calibration/` folder documents a
  repeatable tone-sweep procedure for measuring and correcting
  per-mic/per-puppet frequency-response differences in cheap USB mics
  -- worth rerunning any time a mic is swapped or the physical stack is
  rebuilt.

## Building your own

Roughly, in order: flash Raspberry Pi OS Bookworm on however many Pis
you want, wire each to a CRT over composite (see any app repo's
"Installing on a fresh Pi" section for the `vc4-fkms-v3d` overlay steps
-- full KMS does not reliably support composite output on a Pi 3B+),
set up passwordless `sudo` + SSH key auth for a shared control user,
clone and install STRINGS on every machine you want remotely supervised,
clone and install CENTRAL SCRUTINIZER on whichever one machine will be
your control point (with a USB remote attached), then install whichever
of the app repos you actually want available and let STRINGS/SCRUTE
handle assignment. Every individual app works standalone too, if you'd
rather run a single Pi with no fleet management at all -- just install
that one repo directly and skip STRINGS/SCRUTE entirely.
