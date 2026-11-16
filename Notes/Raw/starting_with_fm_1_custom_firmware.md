# Starting with fm-1 custom firmware

**User:** @RobinSchmidt

## @RobinSchmidt

What is the easiest way to get started with custom firmware development for the M-VAVE FM-1 for someone who is familiar with audio plugin development using JUCE and the VST and CLAP SDKs?

## Copilot

The easiest route is to fork **Dinghy** and build it with GitHub Actions, so you don't have to set up a toolchain first.

**Steps**

1. **Fork `zvenson/dinghy`.** It's a GitHub template. Its description says it is "the smallest firmware… the plumbing done and documented (updates, USB rescue, USB-MIDI, USB audio, panel, screen, settings), a sine on top". You only write the DSP and leave the hardware plumbing alone. It has a `.github/workflows` folder, so the build can probably run in CI. I only skimmed the repo page and didn't read the README, so confirm that.
2. **Read the docs first.**
   - `m-vave/fm1_open` is the official SDK and app. It has `sdk/`, `app/`, `tools/` and `web/` folders. I only saw its file listing, not its README.
   - `sbegueria/fm1-dev-wiki`, especially Hardware-Overview.md and AI-Skills.md.
   - `aroum/fm1-custom-fw`, a hub of community firmwares and the MIDI SysEx docs.
3. **Install the Unbricker before you flash anything:** `Quixotic7/MvaveFM1Unbricker`. Dinghy also advertises a "USB rescue" path.
4. **Change the sine to your own voice.** Once that works, try `charlesvestal/fm1-base` (the platform layer) or read Felucca's source (`hugelton/Felucca`) for a real synth.

**Differences from JUCE/VST/CLAP**

- **No plugin host.** There is no `processBlock` and no GUI toolkit. You write a small audio callback and handle the panel and screen yourself.
- **Tiny CPU and RAM.** The chip is a Jieli part, which is not an ARM Cortex-M. Avoid `float` heavy math, `std::` containers, heap allocation and `exp`/`sin`/`tanh` in the audio path. Use wavetables, lookup tables, fixed-point or cheap approximations, and render polyphony with a voice-count cap.
- **Check how floats are handled.** Whether the chip has an FPU matters a lot for your DSP design. The Hardware-Overview.md wiki page should say.
- **A crash can brick the device.** Test flashing with the recovery path in place.

**Testing without hardware:** my notes list `open.m-vave.com/fm1-workbench`, a browser emulator. I haven't checked whether it runs custom firmware. Dinghy has a `tests/` folder, so you can probably unit-test your DSP on the desktop, as you would in JUCE.

I can read Dinghy's README and docs and give you the exact build and flash commands if you want.

## @RobinSchmidt

I do not have the hardware unit yet but I want to get started with trying my first attempts with emulators anyway, so as to not take any risks of bricking the device due to novice mistakes. Can you try to find out, if https://open.m-vave.com/fm1-workbench supports custom firmwares? I have also heard that there is an emulator VST3 plugin. Can you confirm that and find out, if that supports custiom firmware?

## Copilot

I’ll check the workbench and the VST3 against their documentation and source code, focusing on whether they can run your own firmware rather than just emulate the instrument’s interface.

The official project documents a rebuildable browser simulator, but that’s different from loading an arbitrary firmware image. I’m checking the plugin’s format and development workflow next.

**Yes: there is a VST3 emulator that can load your own firmware.** The strongest match is [Joakim Eriksson’s FM-1 emulator](https://github.com/joakimeriksson/fm1-emu): it emulates the FM-1’s processor and peripherals and loads compiled `.fwsc` firmware packages, including custom ones. It also has a [browser version](https://joakimeriksson.github.io/fm1-emu/) and a CLAP version.

### M-VAVE’s workbench

The [official workbench](https://open.m-vave.com/fm1-workbench) previews FM1 Open in the browser. More importantly for development, its [source repository](https://github.com/m-vave/fm1_open#build) says you can **modify its C sources and rebuild a local browser preview**:

```powershell
.\tools\Build-Web.ps1 -LocalPreview
```

That produces `build/preview/index.html`. This is a source-compiled WebAssembly simulation, **not an established way to upload arbitrary `.fwsc` files into the online workbench**. I couldn’t access the live site to check its interface directly; the documentation I found does not establish such an upload feature.

### The emulator plugin

The [fm1-emu plugin documentation](https://github.com/joakimeriksson/fm1-emu/tree/master/crates/fm1-plugin) explicitly says **Load firmware / save…** opens a `.fwsc` package. Thus you can build your own firmware, load it in your DAW, play it with MIDI and use the virtual panel—without hardware. The browser emulator also supports locally opened packages.

[Release 0.5.1](https://github.com/joakimeriksson/fm1-emu/releases/tag/v0.5.1) provides plugin downloads for **Apple Silicon macOS** and **Windows x86-64**. On Windows, install both files: the VST3 wrapper loads the CLAP plugin. Intel Mac builds are documented from source; the release does not list a Linux plugin binary.

### A practical first experiment

I’d start with [foma](https://github.com/konsumer/foma): it wraps a small C firmware template in a Docker-based toolchain. With Docker running:

```sh
git clone https://github.com/konsumer/foma.git
cd foma
./fm1 dev
```

Open `http://localhost:8000`, edit `src/main.c`, and hear changes in foma’s browser simulation. On Windows, its documented command is `fm1.cmd dev`. To try the **actual compiled device package** in the VST3, run `./fm1 build` and load the resulting `build/NAME-VERSION.fwsc` in fm1-emu.

One important distinction: foma’s fast browser preview recompiles your source to WebAssembly; fm1-emu runs the **device firmware instructions**. Neither proves a firmware is safe on real hardware: timing, peripherals and flash behavior can differ. The emulator’s documentation notes that its default mode is more generous than hardware; its standalone `FM1_STRICT=1` mode provides a more realistic, slower check.

Also, a correction to my earlier answer: Dinghy **does** document a working host simulator, and its own README gives the commands; I should not have recommended its CI build without checking.
