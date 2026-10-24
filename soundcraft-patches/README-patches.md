# SoundCraft patch set: VST3 MIDI output, track routing and a Linux editor host

These four unified diffs apply in order to the official upstream commit
`c1bf044a85b55e9aa083624c39fabddbb08dadeb` (`Release 0.9.0`, 2026-10-17). The checkout also has
`upstream` pointing to `https://github.com/storytold/soundcraft`; the base commit is contained in
`upstream/main`. No commit is created in the source checkout.

```sh
git apply --check soundcraft-patches/0001-fix-instrument-plugin-insert-menu.patch
git apply soundcraft-patches/0001-fix-instrument-plugin-insert-menu.patch
git apply --check soundcraft-patches/0002-add-vst3-midi-output-events.patch
git apply soundcraft-patches/0002-add-vst3-midi-output-events.patch
git apply --check soundcraft-patches/0003-add-track-to-track-midi-routing.patch
git apply soundcraft-patches/0003-add-track-to-track-midi-routing.patch
git apply --check soundcraft-patches/0004-add-linux-x11-vst3-editor-host.patch
git apply soundcraft-patches/0004-add-linux-x11-vst3-editor-host.patch
```

Reverse the same patches in reverse order to undo. The sequence was applied in a fresh detached
worktree of the base commit and compared with the tracked source tree: the patched tree is
`6cf980661a3b8e8fc5d7922a2ff2b04d4d68a6d6`, the working tree `09fa5d0f7dab916f99de4b5cc3973d33a6699c4a`;
they differ only in `Cargo.toml`, which carries a `resolver = "2"` change that was already in the
working tree before this work (a workaround for an old Cargo) and is therefore not part of the
patches.

## Patch purposes and dependencies

1. **`0001-fix-instrument-plugin-insert-menu.patch`** — separates audio inserts from the instrument
   picker and MIDI-processor classification, fixes VST3 sub-category recognition, gives MIDI tracks
   an instrument slot, and keeps unsupported CLAP/AU MIDI processors out of audio inserts. It also
   adds the Mix-window MIDI path control; patch 0003 supplies the command and processing it invokes.
2. **`0002-add-vst3-midi-output-events.patch`** — activates VST3 event-output buses, receives and
   converts bounded events (notes, velocity, channel, controllers, pressure, pitch bend, program
   change, and block sample offsets), and permits event-only plugins without audio buses. It depends
   on patch 0001's DSP event API and descriptor fields.
3. **`0003-add-track-to-track-midi-routing.patch`** — adds serde-defaulted session routes, command/UI
   dispatch, deterministic same-block dependency ordering, bounded event transfer, cycle rejection,
   and note release when routes or source state change. It depends on patches 0001 and 0002.
4. **`0004-add-linux-x11-vst3-editor-host.patch`** — hosts VST3 editor windows on Linux and the
   BSDs: an X11 window (VST3's `X11EmbedWindowID` platform type) the plugin embeds itself into,
   loaded through `x11-dl` so no X11 development files are needed to build. The window is created
   override-redirect at the top-left screen corner, because Wine translates pointer events for an
   embedded window against the Win32 screen origin instead of against the wrapper window: only
   when window and screen coordinates coincide do clicks land on the control under the pointer.
   Because such a window never becomes the window manager's active window, the host also sets the
   input focus to the plugin's own child window and raises the editor while the pointer rests over
   it, so typing works and the window stays reachable when SoundCraft is raised over it. It also
   makes a failing editor visible where the button was pressed instead of only in the Edit window's
   status line, and disables the button with the reason where the platform cannot host editors. It
   depends on patches 0001 and 0002 (descriptor fields, DSP API).

## Issue #372 assessment

Issue #372 is from 0.6.1 and expects CLAP/VST3 instruments in `Insert A`. In the checked-out 0.9.0
source, that slot is intentionally an **audio-effect** insert; instruments belong in the dedicated
`INSTRUMENT` slot. The Windows scan roots already include `%COMMONPROGRAMFILES%\CLAP` and
`%COMMONPROGRAMFILES%\VST3`, so the source does not show the reported default-path omission. An
empty effect list could nevertheless have been worded misleadingly as "No CLAP plugins found"; the
menus now say when no matching audio effects or instruments are present. Vital and TruePianos are
not installed here; Instacomposer 3 (the Windows VST3, hosted through yabridge) is, and it is used
for the editor verification below, but this checkout makes no claim about the native scan of those
binaries' metadata. A synthetic VST3 fixture covers event-only hosting and event conversion.

## Verification

Executed on Linux x86-64:

- `cargo fmt --all -- --check` — passed.
- `cargo test -p soundcraft-vst3-host` — passed, including the fixture E2E (event mapping, editor
  probe and open/close on X11, state round trips).
- `cargo test -p soundcraft-ui-egui -p soundcraft-model -p soundcraft-mix -p soundcraft-engine -p soundcraft-clap-host -p soundcraft-au-host -p soundcraft-dsp` — passed (MIDI routing, cycle rejection, session persistence, picker filters).
- `cargo check --workspace` — passed.
- `cargo clippy --workspace --all-targets -- -D warnings` — passed.
- End-to-end in the built 0.9.0 app on X11: Instacomposer 3 (a Windows VST3 through yabridge and
  Wine) opened its editor, rendered, accepted mouse clicks (menu, chord grid, text field) and
  keyboard input, and closed from the plugin window in SoundCraft.
- The patches were applied in order with `git apply --check` in a clean detached worktree and
  reproduced the tracked tree.

The container lacked ALSA development metadata, so checks used a local `alsa.pc` plus a link to the
installed `libasound.so.2`; playback tests emitted expected "JACK server is not running"/no-device
warnings but passed. The WASM step of `cargo xtask ci` could not run here (the toolchain has no
`wasm32-unknown-unknown` target).

UI/API limitations:

- CLAP and Audio Unit MIDI **output** is not collected; CLAP note-effects and AU music-effects are
  not presented as working MIDI generators. Their MIDI input/instrument support remains available.
- The route control is in the Mix window, not the Edit window's I/O column.
- Routed events drive destination instruments in real time; they are not recorded into destination
  MIDI clips. External MIDI devices are outside this patch set.
- VST3 MIDI processors are recognized from declared sub-categories for menu purposes. A plugin with
  no useful category metadata cannot be identified by the metadata-only scan until instantiated.
- A Windows VST3 running under Wine (yabridge) gets a borderless editor pinned to the top-left screen
  corner. Wine reports the plugin's mouse position against the Win32 screen origin rather than the
  wrapper window, so the host window has to sit at (0,0) for clicks to land on the right control.
  Such a window is not managed by the window manager: hovering it raises it above SoundCraft, and it
  is closed from the plugin window inside SoundCraft. Verified end-to-end with Instacomposer 3
  (yabridge 5.1.1, Wine 11 built with the D2D1/DComp patches; no DXVK involved): the GUI renders,
  menus open, chords and the license field accept mouse and keyboard input, and closing from
  SoundCraft tears the window down.
