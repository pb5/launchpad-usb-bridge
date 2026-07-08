*[日本語](README.ja.md)*

# Launchpad USB Bridge


![The bottom-left pad lit red via the bridge](images/launchpad-lit.jpg)

A direct USB bridge that lets Max (via Node for Max) talk to an original Novation Launchpad
(model `NOVLPD01`, USB Vendor ID `0x1235` / Product ID `0x000e`) without Novation's
discontinued "Automap" software.

## Background / what we found

- The device shows up on the USB bus, but macOS never binds any HID/MIDI class driver to it
  (`bInterfaceClass = 255`, vendor-defined interface, Low-Speed 1.5Mb/s). That's why it's
  invisible to Audio MIDI Setup, Max's `[hid]` object, and `node-hid` (hidapi) alike.
- It's reachable at the raw USB bus level (libusb). Interface 0 exposes two fixed 8-byte
  Interrupt endpoints (IN: `0x81`, OUT: `0x02`).
- The raw packets carried on those endpoints are **plain MIDI bytes** (with running status).
  Automap wasn't doing anything clever -- it was just exposing this nonstandard transport as
  a regular OS MIDI port.
- So the fix isn't "make it look like a HID device" -- it's "read/write the endpoints directly
  with libusb and parse running-status MIDI".

## Runtime gotchas

- Node for Max's bundled Node.js (`[node.script]`) is often a **different version** from your
  Terminal's `node`. Native modules (`usb`) need to be built/fetched matching Max's bundled
  Node ABI.
  - To check: use `node.script node_version_probe.js` -- the Max console prints
    `Node version` / `ABI (NODE_MODULE_VERSION)` / `Platform/arch`.
  - To match it: temporarily download the matching Node.js tarball, put it first on `PATH`,
    and reinstall `usb` under it (`npm rebuild --target=...` no longer works reliably on
    newer npm, so it's not recommended).
    ```
    curl -O https://nodejs.org/dist/vX.Y.Z/node-vX.Y.Z-darwin-arm64.tar.gz
    tar -xzf node-vX.Y.Z-darwin-arm64.tar.gz
    export PATH="$(pwd)/node-vX.Y.Z-darwin-arm64/bin:$PATH"
    node -v   # sanity check
    rm -rf node_modules/usb && npm install usb
    ```
- A `[node.script ...]` object does nothing just by being created -- **you must explicitly
  send it a `script start` message** before it runs (`launchpad_node_hid.maxpat` sends this
  automatically via a loadbang).

## Using this on another machine

### Intel Mac

You likely don't need this bridge at all. On a tested Intel Mac (Big Sur), the Launchpad
showed up natively in Audio MIDI Setup -> MIDI Studio with zero extra software (Apple's own
USB driver stack still has legacy compatibility for this old low-speed device). Check Audio
MIDI Setup first; if it's there, use it directly from Ableton Live etc. as a normal MIDI
device.

### A different Apple Silicon Mac

Same procedure is needed, but you don't have to start from scratch -- just reuse this repo.

```
git clone https://github.com/pb5/launchpad-usb-bridge.git
cd launchpad-usb-bridge
npm install
```

That Mac's Max may bundle a different Node.js version, so follow the "Runtime gotchas"
section above with `node_version_probe.js` and reinstall `usb` to match if needed.

### Windows

Untested. Windows doesn't auto-assign a generic driver to a "vendor-defined interface" that's
neither HID nor MIDI, so reaching it via libusb will likely require manually assigning a
WinUSB driver to the device (Vendor ID `0x1235` / Product ID `0x000e`) using **Zadig**. Plan
for this separately if you try it.

## If you upgrade Max

Because Node for Max is bundled inside the Max application itself, **upgrading Max can also
change its bundled Node.js version**. That's exactly what broke this bridge originally --
Max's bundled Node (v22.18.0 / ABI 127) didn't match the Terminal's Node (v24.18.0).

If `usb` suddenly stops working after a Max update (stuck on "Node script not ready", or a
`require('usb')` error), do this:

1. Add a `[node.script node_version_probe.js]` object to a patch, send it `script start`,
   and read the new Node.js version from the Max console.
2. Temporarily download that Node.js version, put it on `PATH`, and reinstall `usb` in the
   `launchpad_hid_bridge` folder under it:
   ```
   curl -O https://nodejs.org/dist/vX.Y.Z/node-vX.Y.Z-darwin-arm64.tar.gz
   tar -xzf node-vX.Y.Z-darwin-arm64.tar.gz
   export PATH="$(pwd)/node-vX.Y.Z-darwin-arm64/bin:$PATH"
   node -v
   rm -rf node_modules/usb && npm install usb
   ```
3. Restart Max and reopen the patch.

As long as you stay on one Max version, it keeps working indefinitely -- it only breaks when
Max itself is upgraded.

## Files

- `launchpad_bridge.js` -- the bridge itself. Handles `open` / `close` /
  `send status data1 [data2]` messages. Incoming MIDI is outlet as
  `['midi', status, data1, data2]`.
- `launchpad_node_hid.maxpat` -- the Max patch that uses it (shows decoded MIDI, has LED test
  buttons).
- `usb_probe.js` / `launchpad_raw_read.js` / `node_version_probe.js` -- standalone diagnostic
  scripts used during reverse engineering (run directly with `node xxx.js`, no Max needed).

## MIDI mapping

### Grid (8x8 pads)

`Note = 16 x row + col`. Row 0 is the top row (nearest the round Automap-style buttons), row 7
is the bottom row (nearest the USB connector). All are **Note On/Off (status = 144 / 128,
ch. 1)**.

| row\col | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|---|---|---|---|---|---|---|---|---|
| 0 (top) | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
| 1 | 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 |
| 2 | 32 | 33 | 34 | 35 | 36 | 37 | 38 | 39 |
| 3 | 48 | 49 | 50 | 51 | 52 | 53 | 54 | 55 |
| 4 | 64 | 65 | 66 | 67 | 68 | 69 | 70 | 71 |
| 5 | 80 | 81 | 82 | 83 | 84 | 85 | 86 | 87 |
| 6 | 96 | 97 | 98 | 99 | 100 | 101 | 102 | 103 |
| 7 (bottom) | 112 | 113 | 114 | 115 | 116 | 117 | 118 | 119 |

### Right-side round buttons (scene launch, x8)

The 9th column of the grid. Same formula with col=8: `16 x row + 8`. Also **Note On/Off**.

| row | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|---|---|---|---|---|---|---|---|---|
| Note | 8 | 24 | 40 | 56 | 72 | 88 | 104 | 120 |

### Top round buttons (x8, Automap/Live buttons)

These send **Control Change (status = 176, ch. 1)**, left to right.

| Button | Up | Down | Left | Right | Session | User 1 | User 2 | Mixer |
|---|---|---|---|---|---|---|---|---|
| CC # | 104 | 105 | 106 | 107 | 108 | 109 | 110 | 111 |

Under Ableton Live's native control surface script these may display as
Vol/Pan/SndA/SndB/Stop/TrackOn/Solo/Arm, but the raw MIDI numbers (CC 104-111) are fixed
regardless.

### Velocity/value = LED color (shared)

Grid and scene buttons use Note On, top-row buttons use CC -- both encode color in the
velocity/value byte.

| Value | Color |
|---|---|
| 12 | Off |
| 15 | Red (full) |
| 60 | Green (full) |
| 63 | Amber/yellow (full) |

Red and green each have 2 bits of brightness (0-3), so intermediate brightness levels beyond
these four are also possible (value = 16 x green(0-3) + 8(copy) + 4(clear) + red(0-3)).

## Usage

See `launchpad_node_hid.maxpat`.

### First-time setup (required)

The native module (`usb`) is an environment-specific compiled binary -- git only holds the
source. **Anyone who clones this repo needs to do the following build step once, on their own
machine.** As noted in "Runtime gotchas", Max's bundled Node.js almost never matches your
Terminal's Node.js, so a plain `npm install` alone generally will not work.

1. `npm install`
2. Check Max's bundled Node.js version: add a `[node.script node_version_probe.js]` object to
   a patch, send it `script start`, and read the version from the Max console.
3. Reinstall `usb` for that Node.js version:
   ```
   curl -O https://nodejs.org/dist/vX.Y.Z/node-vX.Y.Z-darwin-arm64.tar.gz
   tar -xzf node-vX.Y.Z-darwin-arm64.tar.gz
   export PATH="$(pwd)/node-vX.Y.Z-darwin-arm64/bin:$PATH"
   node -v   # sanity check
   rm -rf node_modules/usb && npm install usb
   ```

### Everyday usage

1. Open the patch -> a loadbang auto-sends `script start`
2. Click the `open` message
3. Press buttons on the Launchpad -- decoded output appears below `route midi`
4. Use `send <status> <data1> <data2>` messages to control LEDs (e.g. `send 144 0 15` lights
   the top-left pad red)
