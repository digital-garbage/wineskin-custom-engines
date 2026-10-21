# Wineskin custom engines (DGsCustom)

Wine engines for [Wineskin](https://github.com/Gcenx/WineskinServer) and [Porting Kit](https://www.portingkit.com/) wrappers on macOS, with fixes for running Steam and DirectX 11 games through [DXMT](https://github.com/3Shain/dxmt) (Direct3D 11 on Metal).

Latest: **[WS12_Wine11.0_DXMT_DGsCustom-v0.3](../../releases/latest)**

It is a hobby build: everything was tested on one machine only (see [Test machine](#test-machine)). Other Macs, macOS versions and games are untested.

## Test machine

| | |
|---|---|
| Mac | MacBook Pro 13" M1, 2020 (MacBookPro17,1) |
| Chip / RAM | Apple M1 (8-core CPU, 8-core GPU), 8 GB |
| macOS | 27.0 (26A5421a), Rosetta 2 |
| Display | external 4K (3840x2160) |
| Audio | wired headphones on the 3.5 mm jack, 48 kHz output |
| Mouse | Logitech G PRO X Superlight 2 (up to 8 kHz polling) |
| Wrapper | Wineskin wrapper made by Porting Kit (Steam, 32+64-bit), DXMT enabled |
| Tested | Steam (login, library, store), The Elder Scrolls V: Skyrim Special Edition (about 60 FPS at 1080p low), Counter-Strike: Source |

## What is in v0.3

The base is Porting Kit's Wine 11.0 engine with its DXMT build. On top of it:

| # | Fix | File changed |
|---|-----|--------------|
| 1 | Skyrim SE no longer crashes with "pure virtual function call" | `lib/wine/x86_64-windows/d3d11.dll` (8 bytes) |
| 2 | 5.1 game audio is mixed down to stereo instead of losing the center and rear channels | `lib/wine/x86_64-unix/winecoreaudio.so` |
| 3 | Microsoft's XAudio2 no longer "chops" (helicopter-like flutter) | `lib/wine/x86_64-unix/winecoreaudio.so` |
| 4 | Mac driver rebuilt from plain Wine 11.0 source, with the DXMT surface hooks Steam needs | `lib/wine/x86_64-unix/winemac.so` |

Everything else is byte-identical to the Porting Kit engine. Source patches for 2-4 are in [`patches/`](patches).

### 1. Skyrim "pure virtual function call" crash (DXMT)

**Symptom:** Skyrim SE crashes during loading.

**Cause:** DXMT's `d3d11.dll` defines the C++ handler `__cxa_pure_virtual` as a hard stop (a `ud2` trap instruction). Skyrim hits it during normal operation, and on Windows that call is harmless. Upstream DXMT fixed this in commit [66277b82](https://github.com/3Shain/dxmt/commit/66277b82): the handler logs and returns instead of trapping. Porting Kit's DXMT build predates that commit.

**Fix:** the same change, applied directly to the binary because the source of Porting Kit's DXMT build is not public:
- file offset `0x1ef1d`: `0f 0b` (`ud2`) becomes `eb 28` (jump over the trap)
- file offset `0x1ef47` (padding): `48 83 c4 40 5b c3` (`add rsp,0x40; pop rbx; ret`)

Only the 64-bit `d3d11.dll` is patched (Skyrim SE is 64-bit).

### 2. 5.1 audio on stereo headphones (CoreAudio driver)

**Symptom:** Skyrim NPC voices disappear when you face the speaker. Many sounds are missing on headphones or stereo speakers.

**Cause:** some games (Skyrim among them) always open a 6-channel (5.1) stream. Wine handed all 6 channels to macOS, which played only the first two (front left/right) on a stereo device. Voices, which sit in the center channel, were silently dropped, along with the rear channels.

**Fix:** when a game sends more than 2 channels to a 2-channel device, Wine's CoreAudio driver mixes them down to stereo itself, using the standard ITU-R BS.775 gains: center and surrounds at 0.707 into left and right, LFE dropped, output clamped.

### 3. Microsoft XAudio2 chopping (CoreAudio driver)

**Symptom:** with Microsoft's own `xaudio2_7.dll`, sound has a constant "tu-tu-tu" flutter, like a chopped turbo.

**Cause, measured on the driver's output:** a gap of 56 samples of silence every 3584 samples, which is about 12 times per second. macOS pulls sound from Wine's driver in 512-sample chunks. Microsoft's XAudio2 refills only one 10 ms period (441 samples at 44.1 kHz) per wake-up and keeps the queue nearly empty. Once per cycle, a 512-sample pull finds fewer than 512 samples waiting and the remainder is played as silence. Windows pulls exactly one period at a time, so this never happens there.

**Fix:** Wine's CoreAudio driver asks macOS for 256-frame I/O chunks on playback streams (`kAudioDevicePropertyBufferFrameSize`). A pull is then always smaller than a refill, and the measured output has zero gaps. To change the value, set the environment variable `WINECA_IOFRAMES` (frames per I/O chunk). `WINECA_IOFRAMES=0` keeps the macOS default.

### 4. Mac driver with DXMT surface support (`winemac.so`)

DXMT asks Wine's Mac driver for a Metal layer through a private escape that Porting Kit and CrossOver drivers provide and plain Wine does not. Steam goes one step further: its browser draws from a separate process (`steamwebhelper.exe`) into a window owned by `steam.exe`. That needs a layer that can be shown across processes. Without this, Steam's window stays black or the helper aborts.

This driver is plain Wine 11.0 `winemac.drv` plus:
- `ExtEscape` codes `0x1a86` (get surface: returns the surface and its `CAMetalLayer`) and `0x1a87` (release surface), with `QUERYESCSUPPORT`.
- **Same process:** a Metal view is created in the window's client surface.
- **Other process:** an off-screen `CAMetalLayer` is published through a `CAContext`. The window owner is told the context id through two driver messages and shows it with a `CALayerHost` in the window's content view. Porting Kit's driver uses the same mechanism.

Why rebuild it at all: Porting Kit's driver is closed source. Having the source means mouse and window problems can now be debugged and fixed (see Known issues).

### Recommended: Microsoft XAudio2 for all games

Wine's builtin XAudio2 (based on FAudio) runs games that use XAudio2 2.7 or older, but in Skyrim it sounded muffled and its reverb made voices echo indoors (see also [Wine bug 53108](https://bugs.winehq.org/show_bug.cgi?id=53108)). Microsoft's own DLLs fix both, and with fix 3 they no longer chop. This is a setting of the wrapper's prefix, not of the engine:

1. Make sure the native DLLs are in the prefix (`drive_c/windows/system32` and `syswow64`). Steam installs them with the DirectX redist for many games, or run `winetricks xact`.
2. Import [`extras/native-xaudio2.reg`](extras/native-xaudio2.reg) into the prefix: in Wineskin, use Advanced, then Tools, then Registry Editor, then File, then Import. From a terminal, use `wine regedit native-xaudio2.reg`.

It covers XAudio2 2.0-2.7, X3DAudio and XAPOFX only. Games using XAudio2 2.8/2.9 (Windows 8+ versions, often shipped with the game as `XAudio2_9redist.dll`) are not affected.

## Installing the engine

Download `WS12_Wine11.0_DXMT_DGsCustom-v0.3.tar.7z` from [Releases](../../releases/latest). Do not unpack it; Wineskin reads the `.tar.7z` directly. Put it into:

```
~/Library/Application Support/Wineskin/Engines/
```

(Finder: Go, then Go to Folder, and paste the path. Create the `Engines` folder if it does not exist.)

The engine needs Apple Silicon with Rosetta 2, or an Intel Mac. Games must be Direct3D 11 for DXMT. Point Wine at DXMT's DLLs with the DLL override `*dxgi,*d3d10core,*d3d11,*d3d12=b` if the wrapper does not already do so.

### Plain Wineskin

- **New wrapper:** open **Wineskin Winery**. The engine appears in the engine list once the file is in the Engines folder. Select it and click **Create New Blank Wrapper**, then install your game as usual.
- **Existing wrapper:** right-click the wrapper app, choose **Show Package Contents**, and open **Wineskin.app** inside. Click **Advanced**, open the **Tools** tab, click **Change Engine Used**, and pick `WS12_Wine11.0_DXMT_DGsCustom-v0.3`.

### Porting Kit

Porting Kit ports are Wineskin wrappers and use the same Engines folder.

1. Put the engine file into `~/Library/Application Support/Wineskin/Engines/`.
2. Switch the port to it. If the port app contains `Wineskin.app` (Show Package Contents), use **Change Engine Used** as in the Wineskin steps above. Otherwise, change the engine from the port's settings in Porting Kit.
3. Keep a copy of the port before switching. If something breaks, switch back to the engine it had before.

## Known issues and work in progress

### Mouse camera jitter (WIP, not fixed)

In mouse-look games (Counter-Strike: Source, PowerWash Simulator 2) turning the camera with the mouse is choppy and costs FPS. Turning with the keyboard is perfectly smooth in the same game, so rendering is fine. Only mouse input is affected. What has been found so far:

- **High polling rates hurt.** A pure CPU benchmark (no Wine, no game) runs about 23% slower while an 8 kHz mouse moves. WindowServer and Apple's HID drivers eat the CPU. This part is macOS itself: 1000 Hz behaves better than 8000 Hz, and 125 Hz is the least bad.
- **The camera "zigzag"** comes from how games re-centre the cursor every frame (CS:S does it 50-67 times per second). macOS 27 folds each warp jump into the next movement event, and Wine discards those events. Their sum cancels out, so no real movement is lost, but movement reaches the game unevenly: about half of the frames get no mouse delta during a steady sweep.
- **Cursor clipping state** differed between a "bad" and a "good" run in one test setup. That setup used an instrumented driver, though. With the clean driver in v0.3 the game behaves the same from launch, and alt-tab changes nothing, so this lead is not confirmed.
- **Ruled out by measurement:** Wine's main thread being blocked, window level, display capture, raw input (`m_rawinput 1`), and Wine's cursor-clip event tap.
- **Next steps:** a warp-free relative mouse mode (skip the cursor warp while the game hides and confines the cursor, and feed the game raw movement deltas instead), then measure with the same trace tools.

The Mac driver in v0.3 is built from source (fix 4) mainly so this can be fixed. Porting Kit's driver is closed.

### System cursor over the game (WIP, not included)

In borderless "windowed fullscreen", the macOS cursor can appear over or instead of the game's cursor, for example after touching the Dock area at the bottom of the screen. Alt-tab hides it for a while.

- **Tried:** hiding the Dock and menu bar while a fullscreen Wine window is active (`NSApplicationPresentationHideDock | HideMenuBar`), the way native macOS games do.
- **Result:** with that build, Steam's browser process hung on start ("Failed to launch executable"), so the change was pulled before release and was never tested in a game.
- **Status:** the cause of the hang has not been investigated yet.

### Other

- **Steam window resizing** lags behind, and dropdown menus can leave shadows. This is cosmetic.
- **Skyrim audio** may still sound slightly less crisp than on Windows. Skyrim mixes in 5.1 on every platform (it is hard-coded in the game), so the remaining difference is not the stereo downmix. Unconfirmed lead: Windows "Spatial sound" (Windows Sonic) or PS5 3D audio virtualize headphones, and macOS has no system-wide equivalent for Wine.

## Building from source

The patches apply to the Wine 11.0 release (`git checkout wine-11.0`):

```
git apply patches/wine-11.0-winemac-surface-escape.patch
git apply patches/wine-11.0-coreaudio-downmix-io256.patch
```

Build Wine as x86_64 (under Rosetta on Apple Silicon), then copy `winemac.so` and `winecoreaudio.so` from `dlls/winemac.drv/` and `dlls/winecoreaudio.drv/` into the engine's `wswine.bundle/lib/wine/x86_64-unix/`. The `winemac.so` built from this patch is byte-identical to the one in v0.3.

To repack an engine: put `wswine.bundle/` into `NAME.tar`, compress it with `7zz a -mx=5 NAME.tar.7z NAME.tar`, and set `wswine.bundle/version` to a matching name.

## Credits and license

- [Wine](https://www.winehq.org/), licensed LGPL 2.1 or later. The patches in this repository are under the same license.
- [DXMT](https://github.com/3Shain/dxmt) by 3Shain.
- [Porting Kit](https://www.portingkit.com/) and [Gcenx](https://github.com/Gcenx) for the base engine, the DXMT build and Wineskin.

Made with [Claude Code](https://claude.com/claude-code).
