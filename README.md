# Kindle Scribe DIY: jailbreak, custom UI, pen input

Notes for technical users who want to replace the Kindle Scribe's stock interface with their own, using KOReader as the base. Everything here was verified on a **Kindle Scribe (1st gen), firmware 5.20.1.0.2, KOReader v2026.07.1**. Other models and versions will differ in the details.

This is the groundwork only. The full UI built on it ("Kindle OS": home grid, PDF pen annotation, notebook, split screen) will be open-sourced separately later.

> You do this at your own risk. A jailbreak doesn't touch the bootloader, but editing system files afterwards can brick the device. Don't do it on a Kindle you can't afford to lose.

---

## 1. Jailbreak

Jailbreak methods are tied to firmware ranges, and Amazon closes them with updates. **Always check [kindlemodding.org](https://kindlemodding.org) (Jailbreak Wizard)** for the method that matches your exact firmware (Settings → Device Options → Device Info). On 5.20.1 the method is **Véra**.

### Block updates first

An update arriving mid-jailbreak can undo the jailbreak or close the hole it uses. The simplest block is to **fill the storage** until less than about 100 MB is free. That is too little for an update download (hundreds of MB) and enough for the jailbreak's two small books.

- Copy large dummy files into the Kindle's root over USB, e.g. with OpenMTP. Newer firmware uses MTP, not USB mass storage.
- Avoid `libmtp` command-line tools: each session close can drop the Kindle out of MTP mode until you replug it.
- Delete any `update*.bin` or `update.bin.tmp.partial` file you find in the root.

### Run Véra (summary; follow the site for the current steps)

1. On the Kindle, open the browser at `kindlemodding.org/vera` and pick your model and firmware.
2. Download the *Font Calibration* book and the device-specific *Véra* book.
3. Open Font Calibration, then set **Aa**:
   - Font: Bookerly, bold 0, max size.
   - Line spacing: as the page says.
   - Save the settings as a theme.
4. Open the Véra book and wait for **"RESTARTING GUI"**. Ignore "application error" popups.
5. Check that it worked: a `JAILBROKEN.txt` marker appears in the root, and `;kpm` in the search bar now does something.
6. Delete the filler files and both books. Véra blocks updates from here on.

### Install KOReader and get a shell

Turn Wi-Fi on and type these in the home search bar:

```
;kpm update
;kpm install koreader
```

Open KOReader, then go to **Settings → Network → SSH server**:

- Turn on "Login without password" just for the first connection.
- Start the server.
- It runs on **port 2222** of the Kindle's Wi-Fi IP.

Then install a key and turn passwordless login off again:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/kindle -N ""
ssh -p 2222 root@<kindle-ip> 'mkdir -p /mnt/us/koreader/settings/SSH && cat >> /mnt/us/koreader/settings/SSH/authorized_keys' < ~/.ssh/kindle.pub
```

**SSH and scp gotchas:**

- The SSH server lives inside KOReader. **Restarting KOReader kills your session**, so make deploy scripts tolerate the drop.
- KOReader's dropbear only speaks legacy scp, so use **`scp -O`**.
- It drops the connection when a destination filename is longer than about 100 characters. Upload under a short temp name, then `mv` it over ssh.
- Streaming with `ssh ... 'cat > file' < file` is the most reliable way to copy files.

---

## 2. Custom UI as a KOReader plugin

KOReader is the base: it already handles the e-ink driver, input, the document engines (MuPDF, crengine) and the reader. A plugin can take over everything the user sees.

### Where things live

| Path on device | What |
|---|---|
| `/mnt/us/koreader/plugins/<name>.koplugin/` | your plugin (`_meta.lua` + `main.lua`) |
| `/mnt/us/koreader/patches/` | user patches (`<prio>-name.lua`), applied at startup |
| `/mnt/us/koreader/settings.reader.lua` | `G_reader_settings` |
| `/mnt/us/koreader/crash.log` | log (`logger.info/warn`) |
| `/mnt/us/documents/` | books |

To restart KOReader after a deploy, detach from the SSH session, since it dies with KOReader:

```bash
ssh -p 2222 root@<kindle-ip> 'setsid sh -c "pkill -f [r]eader.lua; sleep 4; /mnt/us/koreader/koreader.sh" </dev/null >/dev/null 2>&1 &'
```

### Minimal plugin

```lua
-- _meta.lua
local _ = require("gettext")
return { fullname = _("My UI"), description = _("Custom shell") }
```

```lua
-- main.lua
local WidgetContainer = require("ui/widget/container/widgetcontainer")
local Home = require("home")          -- must be required HERE, see gotcha below
local MyUI = WidgetContainer:extend{ name = "myui", is_doc_only = false }

function MyUI:init()
    -- self.ui is the FileManager or the ReaderUI; plugins are instantiated for both.
    if self.ui.document then return end   -- reader instance
    UIManager:nextTick(function() Home.show() end)
end

return MyUI
```

**Plugin gotchas:**

- **Your plugin folder is on `package.path` only while `main.lua` loads.** Any `require("home")` evaluated later, inside a callback, fails. Require every internal module at the top of `main.lua` (or of files `main.lua` requires).
- **Taking over Home:** KOReader's "start with" setting (file browser / last file / history) wins over your screen. Set it in your plugin, then show your own full-screen widget over the FileManager.
- **Wrap `paintTo` in `pcall`.** An error during paint takes the whole of KOReader down, since the paint loop has no protection.
- **Comparing an FFI color with `== nil` calls its `__eq` and crashes.** Use `rawequal(c, nil)`.
- **`Blitbuffer.gray(v)` is inverted** (0 = white). Use `Blitbuffer.Color8(v)` for a gray level.

### Drawing and refreshing on e-ink

- A widget is a table with `getSize()` and `paintTo(bb, x, y)`. Show it with `UIManager:show(w, refresh_type, region)`.
- **Always pass a refresh region** to `show`, `close` and `setDirty`. Without one, the framebuffer gets painted but **the panel never updates**: your screenshot of `/dev/fb0` looks fine while the screen shows nothing.
- Refresh types: `"ui"` for chrome, `"partial"` for content, `"full"` for a flashing clean-up, `"fast"` (DU, black and white only) for pen ink and animation.
- **Never block in an input callback.** A synchronous refresh or wait takes hundreds of ms, and evdev drops events meanwhile. For the pen, that shows as the start of a stroke drawn as one straight line. Defer the work with `UIManager:nextTick`.

### Scaling

The Scribe is 1860×2480 at 300 ppi. KOReader's `Screen:scaleBySize` targets smaller screens and makes everything tiny. A level-1 patch can rescale all UI:

```lua
-- patches/1-uiscale.lua  (early: before any widget is built)
local Screen = require("device").screen
local base = math.min(Screen:getWidth(), Screen:getHeight()) / 930
Screen.scaleBySize = function(_, px) return math.floor(px * base + 0.5) end
```

The patch also changes book text sizes. Compensate in `CreDocument:setFontSize` if you want book text sizes unchanged.

Patch priorities:

| Prefix | When it runs |
|---|---|
| `0-` | once, after an update |
| `1-` | early, always |
| `2-` | after UIManager is ready |
| `8-` / `9-` | on exit |

Use patches for global behaviour (scaling, the boot splash). Use the plugin for everything else.

### Gestures

- Gestures reach widgets through `registerTouchZones` with `ges` = `tap`, `hold`, `swipe`, `pan`, `pan_release` and so on.
- `overrides = { "tap_forward", ... }` puts your zone before KOReader's own. A handler that returns `nil` lets the gesture fall through to the next zone.
- `swipe.pos` is the **start** point. `pan` events carry `start_pos`, `pos` and `relative`.
- For edge swipes (control centre, home), wrap `UIManager.sendEvent` and look at `Gesture` events before any widget sees them.
- KOReader's gesture detector can throw on a second ("buddy") contact without coordinates. Wrap its `feedEvent` in `pcall` and drop the contacts on error.

### Rotation

| Setting | Effect |
|---|---|
| `input_ignore_gsensor = false` | auto-rotate on |
| `input_lock_gsensor` (via `Device:lockGSensor`) | rotation lock |
| `lock_rotation = true` | books keep the current orientation instead of their saved one |

Rotation arrives as a `SetRotationMode` event. Rebuild your layouts there.

---

## 3. Pen input

### Devices

| Device | What it is |
|---|---|
| `event3` WacomDigitizer | raw pen: 15624×20832 units, pressure 0..4095 |
| `event6` stylus-custom | pen re-sent by Amazon's WetInkHandler, already in screen px; **this is what KOReader reads** |
| `event4` pt_mt | touchscreen (multitouch) |
| `event1` bma4xy_acc | accelerometer |

The touchscreen emits **only `ABS_MT_*`** codes. The pen emits **only `ABS_X`/`ABS_Y`/`ABS_PRESSURE`/`ABS_DISTANCE`/tilt and `BTN_*`** codes. Every raw event's source is therefore unambiguous.

### Getting pen events in KOReader

KOReader puts the pen in its own slot (`input.pen_slot`) and calls a stylus callback on every `SYN_REPORT`, **before** gesture detection:

```lua
local input = require("device").input
input:registerStylusCallback(function(input, slot)
    -- slot = { slot, id, x, y, tool, timev }; id == -1 means lift. x/y are native (unrotated) px.
    -- tool: 1 = pen, 2 = eraser end. Return true to keep this contact away from the gesture detector.
    return handle(slot)
end)
```

- **Pressure isn't stored in the slot.** Wrap `input.eventAdjustHook` and capture `EV_ABS/ABS_PRESSURE` there.
  - Swallow the event as well: the Kindle's gyro transform otherwise reads pressure values 15..18 as rotation events.
  - Get the pressure range with `EVIOCGABS(ABS_PRESSURE)` (default 4095).
- **Rotation:** map native coordinates with `Screen:getTouchRotation()`, the same way `GestureDetector:translateCoordinates` does.

### Palm and finger rejection

KOReader shares **one `cur_slot` across all input devices**, so a finger frame in the middle of a stroke steals the pen's `ABS_X`/`ABS_Y` updates. To fix it, in your `eventAdjustHook`:

1. Route every pen event (`BTN_TOUCH`, `ABS_X`/`ABS_Y`) explicitly to `pen_slot`, and give finger frames back their own slot.
2. Treat the pen as "in use" for 800 ms after **any** pen event. Ignore a finger contact that *starts* inside that window **for its whole life**, lift included, so no finger gets stuck down.
   - Don't key this on `BTN_TOOL_PEN` proximity: stylus-custom doesn't reliably send the leave-proximity event, so such a flag gets stuck.
3. **Force `ABS_MT_TOOL_TYPE` to 0 (finger) for touch.** The firmware reports palms as `MT_TOOL_PALM`, which code mapping tool types to pen ends reads as the **eraser**. The result is a hand resting on the screen erasing ink.

### Fast ink (stylus mode)

The Scribe's MediaTek "hwtcon" display controller has a low-latency pen mode:

```lua
local ffi = require("ffi")
local C = ffi.C
local MXCFB_SET_STYLUS_MODE = 0x4004464C           -- _IOW('F', 0x4C, uint32_t)
local arg = ffi.new("uint32_t[1]", 1)               -- 1 = EPDC_STYLUS_MODE_WITH_NO_TPS, 0 = off
C.ioctl(Screen.fd, MXCFB_SET_STYLUS_MODE, arg)
```

**Rules:**

- **While stylus mode is on, every update must be DU** (waveform 1, or DUNM 12 in night mode). Any other update must first turn it off.
  - Enforce this by wrapping `Screen.mech_refresh`: if the update is flashing or not DU, turn stylus mode off before passing it on.
  - Then KOReader's own refreshes can't collide with it.
- Draw each segment into `Screen.bb`, then call `Screen:refreshFast(x, y, w, h)` on its bounding box.
  - DU is black and white only, so render gray ink later with a normal refresh after the pen lifts.
- **Turn it on while the pen hovers**, not at pen-down. The ioctl waits for pending updates, and blocking at contact loses the first points of the stroke.
- **Turn it off on every exit path:** close, suspend, error, and when a dialog opens. If a crash leaves it on, clear it once at startup.

With all of this, latency is close to stock. Draw strokes as stamped discs along the segment, with the radius taken from pressure.

### Storing strokes

- Store points normalised to the page (0..1) with pressure: `{ x, y, p, ... }`. They then survive zoom, rotation and reflow of the page geometry.
- Store per document in `doc_settings`, or in your own JSON per notebook.
- Debounce saves to a few seconds after the last stroke, and flush on close and suspend.

---

## 4. Testing without touching the device

You can drive the Kindle entirely over SSH:

- **Screenshots:** read `/dev/fb0` (1860×2480, 8-bit gray) and convert it on the Mac.
  - This shows the framebuffer, **not the panel**, so it can't reveal a missing refresh.
  - Wait for animations and auto-hide timers before grabbing.
- **Synthetic touch:** write `ABS_MT_*` events to `/dev/input/event4` with a small LuaJIT script (`./luajit` lives in `/mnt/us/koreader`). Touch axes are in screen pixels.
  - Tap = down + up.
  - Drag = a series of moves over about 1.5 s. That produces `pan` + `pan_release`; a fast one becomes a `swipe`.
- **Synthetic pen:** the same idea on the stylus device, with `ABS_X`/`ABS_Y`/`ABS_PRESSURE` + `BTN_TOUCH`.
- **Logs:** `tail -f /mnt/us/koreader/crash.log`.

## 5. Useful references

- KindleModding: https://kindlemodding.org
- KOReader source (`frontend/device/input.lua`, `gesturedetector.lua`, `ffi/framebuffer_mxcfb.lua`, `frontend/userpatch.lua`): https://github.com/koreader/koreader
- FBInk (MTK/hwtcon refresh constraints documented in its sources): https://github.com/NiLuJe/FBInk

## License

This guide, including its code snippets, is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). See [LICENSE](LICENSE). You may share and adapt it, including commercially, as long as you give credit.
