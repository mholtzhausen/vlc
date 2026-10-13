# Moth UI Implementation Plan

Implementing the "VLC Redesign" Figma design on top of VLC 4's Qt/QML interface, as the
interface for **Moth**, an open-source media player built from this fork of VLC.

This document is written for an engineer or coding agent who has **not** seen the
conversation that produced it. Read it top to bottom once before starting; after that,
use the phase sections as a work list.

---

## 0. Context and goals

**What Moth is.** A fork of VLC (`github.com/mholtzhausen/vlc`, to be renamed `moth`) that
keeps 100% of VLC's playback engine and feature set (codecs, formats, streaming,
recording, converting, discs, capture devices, network discovery, casting, filters) and
replaces the look of the desktop interface with a community Figma design.

**Non-negotiables (from the project owner):**

1. Performance must not regress versus upstream VLC 4.
2. Codec and format support must not regress. Do not touch `modules/codec`,
   `modules/demux`, `modules/access`, `src/` etc. unless a UI feature strictly needs a
   new API, and then keep it minimal.
3. Every VLC function stays reachable: streaming, recording, converting, effects/filters,
   all preferences. A feature may move or be restyled; it may not disappear.

**Why a fork of VLC rather than a new app on libVLC.** VLC 4's interface is already
Qt 6 + QML, already has a media library, side navigation, mini player, customisable
control bar and client-side window decorations. The Figma design maps closely onto that
structure, so this is mostly a re-theme plus targeted new views, not a rewrite.

**Scope of this plan:** the desktop Qt interface only (`modules/gui/qt`). macOS's native
`modules/gui/macosx` interface, skins2, ncurses, Android and iOS are out of scope.
On macOS, build and use the Qt interface.

---

## 1. Ground rules for whoever implements this

1. **Stay rebase-friendly with upstream.** Upstream VideoLAN changes the QML interface
   frequently. Add the upstream remote and rebase regularly:
   ```
   git remote add upstream https://code.videolan.org/videolan/vlc.git
   git fetch upstream master
   ```
   Prefer changing tokens, styles and QML over rewriting C++ models. Prefer adding new
   files over heavily editing shared ones. Keep each commit focused on one thing.
2. **Never renumber `ControlListModel::ControlType`** (`modules/gui/qt/player/control_list_model.hpp`).
   The numeric values are persisted in users' config. New controls go into the
   `RESERVED_0X1F` and `RESERVED_0X2B`–`RESERVED_0X3F` slots only.
3. **Never remove a VLC option or setting.** If a Figma settings page does not show an
   option, it must still be reachable through "Advanced Options" (see Phase 9).
4. **Keep translation support.** All new user-visible strings go through `qsTr()` in QML
   or `qtr()` / `N_()` in C++.
5. **Keep keyboard navigation and accessibility working.** VLC's QML uses a `Navigation`
   attached property system and `Accessible` roles everywhere. Every restyled or new
   control must keep focus handling, `Navigation.*Item` wiring and accessible names.
6. **Keep scaling working.** Sizes in VLC QML go through `VLCStyle` and
   `MainCtx.dp(px, scale)`. Never hard-code pixel sizes in components; add a token.
7. **Licence.** VLC is GPLv2+ (core LGPL). All Moth changes are GPLv2+. Bundled fonts
   (Inter, DM Mono) are SIL OFL 1.1 and must ship with their licence files.
   Credit the Figma design's author in `README.md` and the About dialog (Figma community
   files are usually CC BY 4.0; confirm on the file's community page).
8. **Trademarks.** "VLC" and the cone are VideoLAN trademarks. The design contains VLC
   strings and the cone. Phase 11 replaces them with Moth branding; until then, do not
   publish builds.
9. **Commit attribution.** Follow the repository's existing conventions and any
   instructions in your environment for commit trailers.

---

## 2. Source of truth: the Figma file

- **File:** "VLC Redesign (Community)"
- **URL:** `https://www.figma.com/design/Ewti3Do8MQbCKzyDbAZWC2/VLC-Redesign--Community-`
- **File key:** `Ewti3Do8MQbCKzyDbAZWC2`
- **Pages:** one page, `Page 1` (`0:1`). The left half is "Components", the right half is
  "Designs".
- All screens are drawn at **1920×1080** (main window) or **~1048×865** (dialogs).

### 2.1 How to read it

If you have the Figma MCP tools:

- `get_screenshot(fileKey, nodeId)` to see a frame. Use it before implementing any screen
  and after, to compare.
- `get_design_context(fileKey, nodeId)` for layout, spacing and colour details of a
  frame or component (load the `figma-design-to-code` skill first, as the tool requires).
  Its generated code is web/React; treat it as a **reference for measurements only** —
  the target is QML.
- `get_variable_defs(fileKey, nodeId)` for the colour tokens used in a frame.
- `get_metadata(fileKey, nodeId)` for structure. **Do not** call it on `0:1`; the page is
  enormous (600k+ characters). Call it on individual frames.
- `download_assets` / `get_design_context` asset URLs to export icons as SVG.

If you do not have them, open the URL in a browser and use Figma's Inspect panel.

### 2.2 Screen inventory (node IDs)

**Main window ("Designs" area, 1920×1080):**

| Node ID | Frame name | What it shows |
|---|---|---|
| `247:445` | Player | Video playing; title bar + full-bleed video + bottom control bar. **Primary player reference.** |
| `6261:15078` | Player | **Home** view: sidebar, Continue Watching, Listen Again, Frequently Visited Folders. |
| `970:2703` | Player | Library **grid** view (Videos › Shows › The Wand › S01), breadcrumbs, sort/view/search, floating mini-player bottom-right. |
| `970:3792` | Player | Library **list** view (same folder), mini-player docked bottom-left under the sidebar. |
| `187:328`, `970:3308`, `970:4028` | Player | Further player/library variants — screenshot and review each before Phase 5/6; not yet catalogued. |
| `4767:23726` | Player - Music | Music now-playing, sidebar collapsed. Blurred artwork backdrop, centred cover, waveform seek bar, ±10s. |
| `4778:25123` | Player - Music | Same with the left song-list sidebar open ("Songs › My Songs"). |
| `3328:11554` | Player - hide UI | Chrome hidden (video only). |
| `2425:13624` | Player - Light | Light theme reference. |
| `2107:5604`, `2160:6137`, `2160:6564`, `2160:7005`, `2160:7437`, `2160:7869`, `6587:19923`, `2160:8301`, `2160:8732` | Player - customize | "Customize Player" control-bar editor, nine states (drag in progress, items placed, etc.). |

**Settings window (1048×865):**

| Node ID | Category |
|---|---|
| `999:5047` | Global (collapsed advanced) |
| `3046:9282` | Global (Advanced Options expanded) |
| `3046:10057`, `3062:11379` | Interface |
| `3062:11911` | Privacy |
| `3062:12552`, `3072:13408` | Playback |
| `3073:14187`, `3073:15375` | Video |
| `3073:15879`, `3180:17398` | Audio |
| `3180:17989`, `6536:13908` | Subtitles |
| `6536:14482` | Controls |
| `6542:15251` | Library |
| `6543:15649` | Network |
| `6571:15958` | Performance |
| `6571:16429` | Recording |
| `6575:17137` | Extensions |

Settings navigation groups, in order: **App** (General, Interface, Privacy, Controls),
**Media** (Playback, Video, Audio, Subtitles, Library), **System** (Network, Performance,
Recording, Extensions). The window has a search field top-left, the page title in the
header, and a close button.

**Dialogs (1048×800 unless noted):**

| Node ID | Dialog |
|---|---|
| `1911:7545`, `1919:8707`, `1919:9278`, `3336:11929` | Add Subtitles to Playlist/Folder (bulk subtitle matching: origin "From Computer"/"From OpenSubtitles", target "To This Folder"/"To This File") |
| `2437:14832` | Keyboard Shortcuts (searchable table: Action / Hotkey / Global Hotkey, grouped Basic, Playback, …) |
| `6216:13614` | Open Network Stream (1048×631; multiple URLs, optional playlist name) |
| `6259:14954` | New Playlist (713×589; name, thumbnail, drop zone for files) |

**Onboarding (first run, 1090×759):** `1899:3055` (welcome + choose media folder),
`1902:3670`–`1902:6465` (theme and accent selection, nine variants), `1902:6770`
(basic keyboard shortcuts, "Finish").

**Menus and popups (Components area):**

| Node ID | Name | Contents |
|---|---|---|
| `6062:13354` | menu | App menu ("…" button): Open File(s) Ctrl+O, Open Folder Ctrl+F, Open Network Stream Ctrl+N, Open Disc Ctrl+D, Open Capture Device Ctrl+C, Convert Ctrl+R, Open Recent, Profile (Movies), Keyboard Shortcuts, Customize Player, Settings Ctrl+P, About Shift+F1 |
| `6648:20666` | Profiles | Same menu with Profile submenu: Movies, Shows, Music, Work, New |
| `6065:13616` | right-click menu | Player context menu: Play, Stop, Previous, Next, Record, Hide UI (Ctrl+\\), Playback ▸, Video ▸, Audio ▸, Subtitle ▸, More Settings, Playlist, Open Media, Quit |
| `2404:13340` | file context menu | Play, Mark as Watched, Favorite, Stream, Convert, Add to Queue, Add to Playlist, Show in File Explorer, Go to Source, Add Subtitle Tracks, Add Audio Tracks, Properties & Metadata |
| `2139:8935` | folder context menu | Add File, Add Folder, Edit Subtitle Tracks, Edit Audio Tracks, Resync, Info |
| `2459:9389` | playlist context menu | Play, Favorite, Shuffle, Export Playlist, Add to Queue, Add Subtitle Tracks, Add Audio Tracks, Properties & Metadata |
| `856:2215` | Sorting Options | Unwatched/Watched first, Newest/Oldest, Title, Artist, Album, Track number, URI, Disc number (asc/desc) |
| `858:3290` | View Options | Grid View, List View |
| `2107:5903` | context menu | Generic menu item template |

**Components (reuse their measurements):**

| Node ID | Component | Variants |
|---|---|---|
| `42:356` | Icon | ~200 icons: `name`, `direction`, `state` (default/active/selected), `value`, `EQ`, `two-tone` (yes/no) |
| `177:98` | Icon Wrapper | Default, hover (the 32×32 button around an icon) |
| `848:1712` | play | play, pause (the orange pill play button) |
| `244:184` | volume | 0%–100% in 10% steps, 150%, 200% |
| `491:1780` | Controls | Control bar variants: default, waveform, music, queue, bookmarks, chapters, subtitles, playback, sub timeline, audio, video, video settings, audio settings, subtitle settings (these are the bar with its **right-hand panels** open) |
| `970:3091` | Miniplayer | 5 variants (431 wide, 14 px radius, drop shadow) |
| `848:1996` | sidebar | Default, home, videos (×3), music (×3), network, playlists |
| `500:3063` | Button | Default, Variant2 (4 px radius) |
| `536:1014` | Toggle | on, off (68×32, 6 px radius) |
| `2372:8300` | Dropdown | default, open (6 px radius) |
| `804:1208`, `3176:16556` | Values | Segmented control, 2 or 3 options |
| `3021:9835` | setting category | selected no/yes (settings sidebar row) |
| `2404:9167` | media | watched, unwatched, hover-watched, hover-unwatched (grid tile) |
| `2459:9352` | playlist | unwatched, hover-unwatched |
| `2151:9673` | thumbnail | Seek-bar hover thumbnail |
| `1904:7238`, `1905:7312`, `2245:9805` | Current Time, Duration, audio value | Time labels |
| `1915:8397`, `1919:9121` | subtitle multiple / one | Subtitle track rows |
| `1959:5325`, `1963:5465` | target, origin | Add-subtitles dialog pickers |
| `2107:5935` | option | Menu option row |
| `2372:8834` | Setting Preset | Preset picker |
| `2372:8487` | EQ presets | default, open |
| `2372:8639` | Audio Visualizer | default, open |
| `2372:8733` | HDR Options | default, open |
| `6456:13892` | sub filename | default, open |
| `3178:16769` | EQ | Equalizer panel (2 variants) |
| `536:1855`, `543:990`, `614:1430` | Video - Essential / Crop / Colors | collapsed, expanded (400 wide side-panel sections) |
| `6114:13610`, `536:1513`, `536:1703`, `612:1223` | Audio - Equalizer / Compressor / Spatializer / Stereo Widener | collapsed, expanded |

---

## 3. Design tokens

The Figma file uses variables from a linked library (they are not local to the file, so
query them per frame with `get_variable_defs`). Values collected so far:

### 3.1 Colours — Dark (default theme)

| Token | Hex | Likely use |
|---|---|---|
| `Dark/BG/B2` | `#090909` | Deepest background (window behind panels) |
| `Dark/BG/B1 Transparent` (`Dark/BG/B1`) | `#0E0E0E` | Window / page background |
| `Dark/BG/A5` | `#151515` | Raised surface |
| `Dark/BG/A4` | `#191919` | Panels, sidebar, cards |
| `Dark/BG/A3 Transparent` | `#191919` | Translucent panels (pair with bg blur) |
| `Dark/BG/A1` | `#222222` | Inputs, hovered rows |
| `Dark/BG/Accent Dark` | `#1A1611` | Accent-tinted dark surface |
| `Dark/BG/Accent Bright` | `#371F06` | Seek-bar elapsed fill, accent-tinted surface |
| `Dark/FG/1` | `#EAEAEA` | Primary text |
| `Dark/FG/2` | `#C6C6C6` | Secondary text |
| `Dark/FG/A1` | `#969696` | Icons / tertiary text |
| `Dark/FG/A2` | `#8F8F8F` | Icons default |
| `Dark/FG/A3` | `#6B6B6B` | Disabled / watched (dimmed) |
| `Dark/FG/A4` | `#404040` | Very dim |
| `Dark/FG/B1` | `#161616` | Text on accent |
| `Dark/Divider/Dim` | `#222222` | Dividers |
| `Dark/Accent/Main` (`Main`) | `#D37918` | Accent (orange) |
| `Dark/Accent/DIm` | `#AB6314` | Accent pressed / dimmed |
| `Dark/Accent/Logo` | `#D37918` | Logo accent |

### 3.2 Colours — Light

| Token | Hex |
|---|---|
| `Light/BG/B1 Transparent` | `#E7E7E7` |
| `Light/BG/B2` | `#DEDEDE` |
| `Light/BG/A3 Transparent` | `#D5D5D5` |
| `Light/BG/A5` | `#888888` |
| `Light/BG/Accent Bright` | `#D8BA9A` |
| `Light/FG/A2` | `#2E2E2E` |
| `Light/FG/A3` | `#6B6B6B` |
| `Light/Divider/Dim` | `#ACACAC` |
| `Light/Divider/Bright` | `#9B9B9B` |
| `Light/Accent/Main` | `#E8851A` |
| `Light/Accent/Logo` | `#C2711A` |

The light set above is incomplete (only tokens used on the Light player frame). Before
Phase 2 is finished, run `get_variable_defs` on more light-theme nodes and fill in the
missing Light equivalents of every Dark token (FG/1, FG/2, BG/A1, BG/A4, etc.). If a light
token does not exist in the design, derive it and mark it `// derived` in code.

### 3.3 Accent swatches (Settings › Global › Accent color, and onboarding)

In order: `#D37918` (default orange), `#E15032`, `#C49E23`, `#19A234`, `#18AAD3`,
`#185DD3`, `#8E42FF`, `#F052B3`, `#AFAFAF`.

VLC already has an accent option: `qt-accent-color` (string: `default`, `system`, or a
colour). The swatch picker should write that option.

### 3.4 Typography

The design uses **Inter** almost exclusively, plus **DM Mono** for keyboard-key labels.
Observed usage counts:

| Font | Size(s) px | Use |
|---|---|---|
| Inter Regular | 14 (by far the most common), 12, 16, 15, 20, 24 | Body, list items, labels |
| Inter Semi Bold | 14, 16 | Section headings, settings section titles |
| Inter Medium | 10, 14, 16, 24 | Small labels, buttons, now-playing title |
| Inter Light | 16, 24, 32 | Large headings (onboarding) |
| DM Mono Regular | 14 | Hotkey key caps |

Tasks: bundle Inter (variable or Regular/Medium/SemiBold/Light) and DM Mono as Qt
resources with their OFL licences, register them with `QFontDatabase` at start-up, and
set the QML default font family. Map VLC's existing font-size tokens in `VLCStyle.qml`
(`fontSize_small` 10, `fontSize_normal` 12, `fontSize_large` 14, `fontSize_xlarge` 16,
`fontSize_xxlarge` 20, `fontSize_xxxlarge` 24 at scale 1.0) so that the most common body
text renders at 14 like the design. Prefer changing which token a component uses over
changing the token values globally, if changing values breaks layouts.

### 3.5 Shape, effects, spacing

- Corner radius: menus and dialogs/windows **14 px** (settings window, network dialog,
  mini-player); context menus **10 px**; dropdowns and toggles **6 px**; buttons **4 px**.
  The main content area in the player (`247:445`) and library frames also has a rounded
  inner panel — measure it with `get_design_context`.
- Effects: `bg blur` = background blur radius 50. `bg blur + shadow` = background blur 50
  plus drop shadow `#00000075`, offset (0, 8), radius 15. The mini-player uses a drop
  shadow of radius ~12.
  VLC already has blur infrastructure (`widgets/qml/DualKawaseBlur.qml`,
  `FrostedGlassEffect.qml`, `AcrylicBackground.qml`, option `qt-backdrop-blur`) — reuse it.
- Spacing: the title bar is 40 px tall, the control bar is 54 px tall, icon buttons are
  24–28 px icons inside 32–36 px hit areas, settings rows are 42–46 px tall, settings
  value controls are 320 px wide and right-aligned at a 784 px content width. Use
  `get_design_context` on each frame for exact paddings.

---

## 4. Map of the code you will touch

All paths are relative to `modules/gui/qt/`.

| Area | Files |
|---|---|
| Theme / tokens | `style/VLCStyle.qml` (sizes, fonts, durations), `style/systempalette.cpp` (all colours per colour set and state; `Day`/`Night` schemes; reads `qt-accent-color`), `style/colorcontext.{hpp,cpp}` (ColorSet/Section/Name/State roles), `style/AcrylicController.qml` |
| Icon font | `pixmaps/icons/*.svg` (≈100 source SVGs), `pixmaps/VLCIcons.json` (glyph list), `pixmaps/makeIconFont.py` (fontforge script that builds `pixmaps/VLCIcons.ttf` and generates the `VLCIcons` QML/C++ index), `style/vlcicons.{hpp,cpp}`, `assets.qrc` |
| Window shell | `maininterface/qml/MainInterface.qml`, `MainDisplay.qml` (≈1160 lines; hosts views, mini player, playlist pane), `MainViewLoader.qml`, `LocalTopbar.qml`, `MenuTopbar.qml`, `SideNavigationPane.qml`, `SideNavigationDelegate.qml`, `NoMedialibHome.qml`; C++ `mainctx.*`, `navigationmodel.*`, `compositor_*` (video/UI compositing per platform — avoid changing) |
| Window decorations | `widgets/qml/CSDWindowButton*.qml`, `CSDThemeButton*.qml`, `CSDTitlebarTapNDrapHandler.qml`, `CSDMouseStealer.qml`; option `qt-titlebar` |
| Player | `player/qml/Player.qml` (≈1400 lines; video/audio view, blurred cover backdrop via `DualKawaseBlur`), `TopBar.qml`, `ControlBar.qml`, `ControlLayout.qml`, `PlayerControlLayout.qml`, `SliderBar.qml` (seek bar), `MiniPlayer.qml`, `PIPPlayer.qml`, `MinimalView.qml`, `PlayerToolbarVisibilityFSM.qml` (auto-hide), `TracksMenu.qml` / `TracksPage*.qml` (audio/subtitle/speed panels), `PlaybackSpeed.qml`, `ResumeDialog.qml` |
| Control widgets | `player/qml/controlbarcontrols/*.qml` (one file per control), C++ `player/control_list_model.hpp` (ControlType enum), `player/player_controlbar_model.*`, `player/control_list_filter.*` |
| Control-bar editor + profiles | `dialogs/toolbar/qml/ToolbarEditor*.qml`, `EditorDND*.qml`; C++ `dialogs/toolbar/controlbar_profile*.cpp` (built-in profiles: Default, Minimalist, One-liner, Simplest, Classic) |
| Media library | `medialibrary/qml/*.qml` (HomePage, VideoAll/Grid/List, Music*, Playlist*, MLContextMenu, VideoInfoExpandPanel), C++ models in `medialibrary/*.cpp` |
| Network / folder browsing | `network/qml/*.qml` (BrowseDisplay, NetworkAddressbar, ServicesSources …) |
| Shared widgets | `widgets/qml/*.qml` (IconToolButton, ButtonExt, SliderExt, ComboBoxExt, TextFieldExt, SearchBox, SortControl, GridItem, ExpandGridView, TableViewExt, ProgressBarExt, VideoProgressBar, PlayCover, MediaCover, ToolTipExt, ViewHeader, BannerCone, VoronoiSnow …) |
| Menus | `menus/qml/Menubar.qml`, `GlobalShortcuts.qml`; C++ `menus/*` (QMenu-based context menus) |
| QML dialogs | `dialogs/dialogs/qml/*` (modal dialog plumbing) |
| **Qt Widgets** dialogs (not QML) | `dialogs/preferences/*` (simple prefs `.ui` pages: interface, audio, video, subtitles, input, medialibrary; `complete_preferences.*` = advanced tree; `expert_*` = expert table), `dialogs/extended/*` (equalizer, video effects), `dialogs/open/*` (open file/disc/network/capture), `dialogs/sout/*` (stream wizard, convert, profiles), `dialogs/firstrun/*` (first-run wizard), `dialogs/help/*` (About, update), `dialogs/mediainfo/*`, `dialogs/bookmarks/*`, `dialogs/messages/*`, `dialogs/plugins/*`, `dialogs/vlm/*`, `dialogs/epg/*` |
| Interface options | `qt.cpp` (`add_bool("qt-...")` etc.: `qt-titlebar`, `qt-menubar`, `qt-pin-controls`, `qt-minimal-view`, `qt-system-tray`, `qt-notification`, `qt-start-minimized`, `qt-video-autoresize`, `qt-name-in-title`, `qt-fs-controller`, `qt-interface-scale`, `qt-error-dialogs`, `qt-updates-notif`, `qt-privacy-ask`, `qt-fullscreen-screennumber`, `qt-bgcone`, `qt-max-volume`, `qt-fs-sensitivity`, `qt-auto-raise`, `qt-close-to-system-tray`, `qt-backdrop-blur`, `qt-pip-mode`, `qt-accent-color`, …) |
| Tests | `tests/` (Qt C++ model tests, `tst_FSM.qml`), `quicktest.pro`, `qtest.pro`, meson test targets |
| Build | top-level `meson.build`, `meson_options.txt` (option `qt`), `modules/gui/qt/meson.build`; Windows notes in `doc/BUILD-win32.md` |

Useful facts:

- The design's Home "Continue Watching" row already exists in `HomePage.qml`
  (`continueWatchingRow`); the current second row is Favorites.
- The player already blurs the cover art behind audio playback (`Player.qml`,
  `DualKawaseBlur`), which is the basis for the music view.
- The control-bar editor already supports drag and drop between a bar and an item pool,
  per-profile layouts, and left/centre/right zones.
- Colours are not hard-coded in QML: components ask a `ColorContext` for a role
  (e.g. `theme.bg.primary`, `theme.accent`). Re-theming is done in `systempalette.cpp`.

---

## 5. Gap analysis (design vs. current VLC 4 UI)

| Design element | Exists today | Work type |
|---|---|---|
| Dark/light palettes, accent swatches | Yes (Day/Night schemes, `qt-accent-color`) | Retheme |
| Inter + DM Mono | No (system font) | Add fonts |
| Icon set (~200 incl. two-tone) | ~100 icons in an icon font | Replace/extend |
| Custom title bar (logo, "…", sidebar toggle, breadcrumb, centred title, window buttons) | CSD title bar exists with different layout | Restyle + re-layout |
| Side navigation (Home, Videos, Music, Network, Playlists, Recent, Add item) | Yes, but with different items ("Recent" and "Add item" do not exist as nav entries) | Restyle + 2 entries |
| Library top bar (back/forward, breadcrumbs, item count, sort, view toggle, search, "+", "…") | Partly (LocalTopbar, NetworkAddressbar, SortControl, SearchBox) | Restyle + re-layout |
| Grid tiles with watched progress bar + dimmed watched titles | Yes (progress bar), styling differs | Restyle |
| Floating mini-player card (artwork, progress, title, folder/artist links, queue button) | Bottom-bar mini player | **Redesign** |
| Home rows: Continue Watching, Listen Again, Frequently Visited Folders | Continue Watching + Favorites | Restyle + 2 new rows (needs data) |
| Player control bar (prev/play/next, times, seek, volume with coloured level, audio, subtitles, settings, queue) | Yes (profiles) | New default profile + restyle |
| Seek-bar hover thumbnail | Partial (VLC 4 has preview on hover in some builds; check `SliderBar.qml`) | Verify / restyle |
| Right-hand panels from the control bar (queue, bookmarks, chapters, subtitles, playback, audio, video, video/audio/subtitle settings, sub timeline) | Tracks menu, playlist pane; effects are Qt Widgets dialogs | **New QML panels** |
| Music now-playing (centred cover, title, waveform seek bar, ±10s) | Audio view with blurred cover, normal slider | Restyle + **waveform** |
| Hide-UI mode | Auto-hide exists; explicit "Hide UI" (Ctrl+\\) toggle may not | Small addition |
| Customize Player dialog | Yes (ToolbarEditor) | Restyle + **new controls** |
| Settings window (App/Media/System, search) | Qt Widgets simple + advanced prefs | **New QML window** |
| Keyboard Shortcuts dialog | Hotkeys live inside prefs (widgets) | **New QML dialog** |
| Open Network Stream, New Playlist, Add Subtitles | Widgets open dialog; playlist creation; no bulk-subtitle tool | **New QML dialogs** (Add Subtitles is new functionality) |
| Onboarding (folder, theme/accent, shortcuts) | First-run wizard (widgets, privacy/ML) | **New QML flow** |
| Menus (app menu, context menus, sort/view popovers) | QMenu-based and QML menus | Restyle (+ new entries) |
| Profiles (Movies/Shows/Music/Work) in app menu | No | **New feature** (see Phase 10) |

### 5.1 Control-bar items: design vs. ControlType

The Customize Player pool in the design lists these items. Map them as follows; "NEW"
items need a new ControlType in a reserved slot plus a QML control file in
`player/qml/controlbarcontrols/` and registration in `PlayerControlbarControls.qml` /
`control_list_model.cpp` (follow how an existing control like `SnapshotButton` is wired
end to end).

| Design item | ControlType |
|---|---|
| Spacer | `WIDGET_SPACER` / `WIDGET_SPACER_EXTEND` |
| Divider | NEW `DIVIDER` (visual separator) |
| Playback Speed | `PLAYBACK_SPEED_BUTTON` |
| Forward 10s / Backward 10s | `SKIP_FW_BUTTON` / `SKIP_BACK_BUTTON` (jump size from the `short-jump-size` option) |
| Forward 15s / Backward 15s | NEW `SKIP_FW_15_BUTTON` / `SKIP_BACK_15_BUTTON`, or make the jump size a per-control setting — decide once and document it |
| Stop | `STOP_BUTTON` |
| Repeat | `LOOP_BUTTON` |
| Shuffle | `RANDOM_BUTTON` |
| Record | `RECORD_BUTTON` |
| Snapshot | `SNAPSHOT_BUTTON` |
| Picture in Picture | NEW `PIP_BUTTON` (VLC 4 has `PIPPlayer.qml` and `qt-pip-mode`; check what triggers it) |
| HDR | NEW `HDR_BUTTON` (opens the HDR Options popup `2372:8733`; tone-mapping options from the video output) |
| Zoom | NEW `ZOOM_BUTTON` (video zoom/crop presets) |
| Right Sidebar | NEW `RIGHT_SIDEBAR_BUTTON` (toggles the right panel area) |
| Quit VLC | `QUIT_BUTTON` (label becomes "Quit Moth" after rebrand) |
| VLC Settings | `EXTENDED_BUTTON` currently opens effects; add NEW `SETTINGS_BUTTON` that opens the new Settings window |
| Go Home | NEW `HOME_BUTTON` (leave player, show Home) |
| Thumbnail | `ARTWORK_INFO` (verify it matches; otherwise NEW) |
| File Name / Artist Name / Folder Name / Playlist Name | NEW text widgets (`FILE_NAME_WIDGET`, `ARTIST_NAME_WIDGET`, `FOLDER_NAME_WIDGET`, `PLAYLIST_NAME_WIDGET`) |
| Properties | `INFO_BUTTON` |
| Replay | NEW `REPLAY_BUTTON` (seek to 0 and play) |
| Chapters | NEW `CHAPTERS_BUTTON` (opens chapters panel); keep `CHAPTER_PREVIOUS/NEXT_BUTTON` |
| Favorite | NEW `FAVORITE_BUTTON` (media library favourite flag; HomePage already uses favourites) |
| Bookmark | `BOOKMARK_BUTTON` (check whether it adds or lists; split if needed) |
| Bookmarks | NEW `BOOKMARKS_BUTTON` (opens bookmarks panel) |
| Playlists | `PLAYLIST_BUTTON` |

Available reserved slots: `RESERVED_0X1F` (buttons range) and `RESERVED_0X2B`–
`RESERVED_0X3F` (21 slots). That is enough for the ~19 new IDs above, with a couple
spare; if more are needed, do not extend past `0x3F` without reading how `WIDGET_SPACER`
(`0x40`) is used. Existing controls not shown in the design pool (A-B loop, frame step,
teletext, aspect ratio, DVD menu, renderer/cast, program, navigation box, reverse,
language) **must remain available** in the editor pool.

---

## 6. Phases

Each phase lists tasks and a **Done when** checklist. Phases 2–4 unblock everything else;
after that, phases can be parallelised across agents with care (they touch different
files), except Phase 9 depends on Phase 2's form controls.

### Phase 1 — Build, baseline and tooling

Tasks:
1. Build VLC from this fork with meson and Qt 6 on the developer's platform (Linux first
   is easiest; see `doc/BUILD-win32.md` for Windows, contribs via `contrib/`). Enable the
   Qt interface and the media library.
2. Add the `upstream` remote (see §1) and note the base commit in this document's
   changelog (§9).
3. Create a test media folder that mirrors the design's sample data (a "Shows/The Wand/S01"
   folder with ~15 short videos, a few songs with and without cover art, a playlist,
   subtitle files) so screenshots are comparable.
4. Capture baseline screenshots of every screen in §2.2 as they look in stock VLC 4.
   Store them outside the repo or under `docs/moth-ui/baseline/`.
5. Make a small script (`extras/moth/screens.sh` or similar) that launches the app on the
   test folder with a clean config dir (`--config`/`XDG_CONFIG_HOME`), so visual checks
   are reproducible.

Done when: the app builds and runs, plays a video and a song, and the baseline screenshots
exist.

### Phase 2 — Design tokens (colours, fonts, shapes)

Tasks:
1. In `style/systempalette.cpp`, change the Night (dark) and Day (light) schemes to the
   Figma values in §3.1–3.2. Map each ColorSet/Section/Name/State to a token. Keep a
   comment table at the top of the function listing token → role so future edits are easy.
   The accent default (`qt-accent-color=default`) becomes `#D37918` dark / `#E8851A` light.
2. Make dark the default colour scheme on first run if it is not already (the design is
   dark-first).
3. Bundle Inter and DM Mono (OFL) under `pixmaps/fonts/` or similar, add them to
   `assets.qrc`, register them at start-up, set them as the QML default font.
4. In `style/VLCStyle.qml`, add/adjust tokens: corner radii (4, 6, 10, 14), title bar
   height 40, control bar height 54, blur radius, shadow parameters, settings row height.
5. Grep QML for hard-coded colours, radii and sizes that bypass tokens and route them
   through tokens.

Done when: every existing screen picks up the new colours and font without layout
breakage; dark and light both look coherent; accent colour option still works with all
nine swatch values.

### Phase 3 — Icons

Tasks:
1. Export every symbol in the Icon component set `42:356` as SVG (24×24). Keep Figma's
   naming: `name`, `state`, `direction`, `value`, `two-tone`.
2. Build a mapping table from existing VLC glyph keys (`pixmaps/VLCIcons.json`) to Figma
   icon names. Every existing key must map to something; record unmapped keys and keep the
   old SVG for them.
3. Single-colour icons: replace the SVGs in `pixmaps/icons/`, add new keys to
   `VLCIcons.json`, and rebuild `VLCIcons.ttf` with `makeIconFont.py` (requires
   fontforge). The script's documented constraints apply: no overlapping paths, viewBox
   equal to the SVG size. Run Figma's "Outline stroke" and "Flatten"/union on the exported
   SVGs, or post-process with a path tool, so they convert cleanly.
4. **States** (`default` / `active` / `selected`): these are usually the same glyph in a
   different colour or a filled variant. Use colour roles for colour-only states; add a
   separate glyph only when the shape differs (e.g. filled home icon when selected).
5. **Two-tone** icons: a font glyph is one colour. Implement two-tone as two glyphs
   (`name` and `name_secondary`) layered in an `IconLabel`, the secondary drawn with a
   dimmer colour role. Use two-tone only where the design does (menus and settings nav use
   them).
6. Remove or replace any glyph that is the VLC cone (`vlc` icon) — see Phase 11.

Done when: no screen shows an old-style icon; the font regenerates from the repo with one
command; the mapping table is committed (e.g. `pixmaps/moth-icon-map.md`).

### Phase 4 — Window shell: title bar, side navigation, library top bar

References: `247:445`, `6261:15078`, `970:2703`, `970:3792`, sidebar component `848:1996`.

Tasks:
1. Title bar (40 px): left — app logo, "…" app menu, sidebar toggle, divider, current
   location (folder icon + name); centre — current media title or app name; right —
   minimise, maximise, fullscreen, close (Icon Wrapper hover state from `177:98`).
   Rework `LocalTopbar.qml` / the CSD title bar to this layout. Keep native decorations
   working when `qt-titlebar` is enabled.
2. "…" app menu with the entries of `6062:13354`. Each entry must open the existing
   functionality (open dialogs, convert, stream, recent media, preferences, about).
   Shortcuts must match VLC's real hotkeys; if the design's shortcut differs from VLC's
   (e.g. Ctrl+F for Open Folder, Ctrl+R for Convert, Ctrl+C for capture), keep VLC's
   actual binding and show it, and note the discrepancy in §8.
3. Side navigation: items Home, Videos, Music, Network, Playlists, Recent, Add item, with
   the selected state (orange icon, lighter row) from `848:1996`. Collapsible (full width
   with labels ↔ icon rail, see `4778:25123` where the rail is shown). "Recent" maps to
   recently played media; "Add item" opens a menu: add folder to library, open file, open
   network stream.
4. Library top bar: back/forward, breadcrumbs (`Videos › Shows › The Wand › S01`),
   item count, sort button (popover `856:2215`), view toggle (`858:3290`), search field,
   "+" (add), "…" (more). Reuse `SortControl`, `SearchBox`, navigation history from
   `navigationmodel`.
5. Content area: rounded panel inset inside the window background (see `247:445`).

Done when: navigating between Home, Videos, Music, Network, Playlists works with the new
chrome on dark and light; keyboard navigation (Tab/arrows) still reaches every element;
window drag, double-click maximise and resize handles still work on Windows, Linux X11
and Wayland.

### Phase 5 — Player

References: `247:445`, `3328:11554`, `2425:13624`, Controls component `491:1780`,
play `848:1712`, volume `244:184`, thumbnail `2151:9673`, right-click menu `6065:13616`.

Tasks:
1. New built-in control-bar profile "Moth" (in `dialogs/toolbar/controlbar_profile_model.cpp`)
   matching `247:445`: left — previous, play (orange pill), next; centre — current time,
   seek bar, duration; right — volume, audio-visualiser/levels, subtitles, settings, queue.
   Make it the default profile for new configs. Keep the other built-in profiles.
2. Restyle `ControlBar.qml` / `SliderBar.qml`: seek bar track `Dark/BG/A1`-ish, elapsed
   fill `Dark/BG/Accent Bright`, orange handle; hover thumbnail preview card
   (`2151:9673`).
3. Volume widget: icon + horizontal level with the green-to-yellow fill shown in the
   design and steps up to 200% (`244:184`; VLC's max comes from `qt-max-volume`).
4. Play button: orange pill (`848:1712`), play/pause states.
5. Auto-hide: keep `PlayerToolbarVisibilityFSM` behaviour; add an explicit "Hide UI"
   toggle (Ctrl+\\, from the right-click menu) that hides all chrome as in `3328:11554`.
6. Right-click player menu `6065:13616` with Playback/Video/Audio/Subtitle submenus;
   reuse VLC's existing menu builders in `menus/` so every existing action is reachable.
7. Light theme check against `2425:13624`.

Done when: playing a video looks like `247:445`; fullscreen, auto-hide, Hide UI,
keyboard shortcuts and mouse wheel volume all work; every previous control-bar profile
still loads.

### Phase 6 — Right-hand player panels

References: the variants of Controls `491:1780` (queue, bookmarks, chapters, subtitles,
playback, sub timeline, audio, video, video settings, audio settings, subtitle settings),
plus panel sections `536:1855`, `543:990`, `614:1430`, `6114:13610`, `536:1513`,
`536:1703`, `612:1223`, `3178:16769`, `2372:8487`, `2372:8639`, `2372:8733`,
`6456:13892`, `1915:8397`, `1919:9121`, `2372:8834`.

These replace popups and Qt Widgets dialogs with a slide-in panel on the right of the
player (≈400 px wide, blurred translucent background).

Tasks:
1. A generic right-panel container in the player (open/close animation, one panel at a
   time, keyboard focus moves into it, Esc closes).
2. **Queue** panel: reuse `playlist/qml/PlaylistPane.qml` content.
3. **Subtitles** panel: track list (`1915:8397` multiple / `1919:9121` one), add subtitle
   file, delay, and **sub timeline** variant (sync adjustment). Reuse `TracksPageSubtitle.qml`
   logic and the delay estimator (`player/delay_estimator.*`).
4. **Audio** panel: track list, delay, device; **Playback** panel: speed (reuse
   `TracksPageSpeed.qml`), A-B loop.
5. **Video settings** panel: sections Essential, Crop, Colors (collapsible), HDR options.
   Back them with the same VLC options/filters the widgets dialog `dialogs/extended/`
   uses (adjust, crop, deinterlace, sharpen, …). Read `extended_panels.cpp` to see which
   variables each widget drives, and drive the same variables from QML (expose a small
   C++ model if needed).
6. **Audio settings** panel: Equalizer with presets (`2372:8487`; the presets must be
   VLC's built-in equalizer presets — the icon set has one icon per preset), Compressor,
   Spatializer, Stereo Widener, and the Audio Visualizer selector (`2372:8639`). Same
   backing as `dialogs/extended/` (equalizer.ui etc.).
7. **Chapters** and **Bookmarks** panels: list, jump, add/remove bookmark (reuse
   `dialogs/bookmarks/` logic or `medialibrary/qml/Bookmarks.qml`).
8. Keep `dialogs/extended` reachable from "More Settings" until the panels cover every
   option, then keep it anyway as the "advanced" route.

Done when: every effect, filter and track option available in VLC's Tools › Effects and
Filters dialog is reachable from a panel or from the advanced route, and changes apply
live during playback.

### Phase 7 — Media library and Home

References: `6261:15078`, `970:2703`, `970:3792`, media tile `2404:9167`, playlist tile
`2459:9352`, context menus `2404:13340`, `2139:8935`, `2459:9389`, Miniplayer `970:3091`.

Tasks:
1. Grid tiles: 16:9 thumbnails, rounded, title below; watched items shown dimmed with a
   full orange progress bar; in-progress items show a partial orange bar; hover states per
   `2404:9167`.
2. List view: two-up/four-column compact list with small thumbnail, as in `970:3792`;
   currently playing item in orange.
3. Home: rows **Continue Watching** (existing), **Listen Again** (recently played audio —
   query the media library history for audio), **Frequently Visited Folders** (new: needs
   a count or recency of folder visits; if the media library has no API for this, store
   visit counts in the Qt settings and order folders by them). Each row has a "›" to see
   all. Keep Favorites available (as a row or a nav filter).
4. Mini-player as a **floating card** (431 px wide, 14 px radius, shadow): video/art
   preview, progress, title, folder link, artist/show link, queue button
   (`970:2703` bottom-right; `970:3792` docked bottom-left). Five variants in `970:3091`
   — implement each state they show. Clicking the preview returns to the player.
5. Context menus per `2404:13340` / `2139:8935` / `2459:9389`. "Stream" and "Convert" must
   open VLC's existing stream/convert flows pre-filled with the item. "Mark as Watched"
   sets progress to complete in the media library. "Show in File Explorer" opens the
   containing folder via the OS.
6. Sort options per `856:2215`, mapped to the media library's sort criteria; include only
   the criteria valid for the current view.
7. Empty states (library empty, no search results, folder empty) are **not** in the
   design: create them in the same visual language (icon + one line + action button).

Done when: Home, Videos, Music, Network and Playlists views match the design at
1920×1080 and remain usable at 1024×640; thumbnails load lazily without scroll jank on a
library of ≥2,000 items (performance non-negotiable).

### Phase 8 — Music mode and waveform seek bar

References: `4767:23726`, `4778:25123`, Controls variants "waveform" and "music".

Tasks:
1. Audio now-playing layout: full-window blurred artwork backdrop (exists), centred
   rounded cover (~740 px at 1080p), "title - artist" below, collapsible left song list
   (current album/folder/playlist with breadcrumb, sort, list mode, search).
2. Music control bar: previous, back 10s, play, forward 10s, next; times; **waveform seek
   bar**; volume; visualiser button; queue.
3. **Waveform** (new C++):
   - Compute a compact peak array (e.g. 1,000–2,000 min/max pairs per track) on a
     background thread by decoding the audio with a separate libVLC/VLC input (use the
     `amem`-style audio callback or a `vlc_player` with a dummy audio output at max
     speed), **never** on the playback pipeline.
   - Cache results on disk keyed by media MRL + size + mtime (e.g. in the user cache dir).
   - Expose to QML as a model/property on the player controller; draw with a QML `Shape`
     or a custom `QQuickItem` (bars, elapsed part orange, rest grey).
   - Fallback to the normal seek bar while peaks are computing or for streams/live media.
   - Make it optional in settings (performance).
4. Ensure videos with audio-only tracks and internet radio streams fall back gracefully.

Done when: playing a song looks like `4767:23726`; the waveform appears within a few
seconds for a local 5-minute track on a mid-range machine, is cached, and CPU use during
playback is unchanged once cached.

### Phase 9 — Settings window (QML)

References: all frames in the Settings table in §2.2; components `3021:9835`
(category row), `536:1014` (toggle), `2372:8300` (dropdown), `804:1208` / `3176:16556`
(segmented values), `500:3063` (button), slider rows ("Hide after 0s–10s", "Popup
duration").

This is the largest single task. VLC's simple preferences are Qt Widgets `.ui` pages
backed by VLC config variables; the advanced tree is generated from module configs.

Tasks:
1. Build a reusable QML settings toolkit: window shell (search, sidebar groups, header,
   close), section header, row with label + info tooltip ("ⓘ" icon on many rows) + value
   control (toggle, dropdown, segmented, slider with ticks, text field, file picker with
   Browse, button). All values bound to VLC config variables through a C++ model
   (`config_GetInt/Float/Psz`, `config_PutInt/...`, `config_SaveConfigFile`); look at how
   `dialogs/preferences/preferences_widgets.cpp` reads and writes each type, and reuse its
   logic rather than duplicating option parsing.
2. Settings data should be declarative: one QML (or JSON) description per page listing
   rows as `{ label, help, option name, control type, choices }`, so pages are cheap to
   build and review.
3. Implement each page per its Figma frame. For every row, find the VLC option it maps
   to (search `qt.cpp`, `src/libvlc-module.c`, and the `sprefs_*.ui` files for the label).
   Known mappings: Theme → colour scheme setting; Accent color → `qt-accent-color`;
   Show confirmation and error dialogs → `qt-error-dialogs`; Check for updates →
   `qt-updates-notif`; Play video inside main window / Show window decorations /
   Show file name in title bar / Resize window to fit video → embedded video,
   `qt-titlebar`, `qt-name-in-title`, `qt-video-autoresize`; Open fullscreen on →
   `qt-fullscreen-screennumber`; Start in compact mode → `qt-minimal-view`;
   Show controls in fullscreen → `qt-fs-controller`; System tray →
   `qt-system-tray`, `qt-start-minimized`, `qt-notification`; Use a custom skin →
   skins2 (consider hiding on Moth); Log file / Log details / Save a log / system log
   options → the logger modules' options. Verify each one; do not guess.
4. Any row in the design with **no** matching VLC option: either implement the behaviour
   (if small and UI-only) or leave it out and list it in §8. Do not ship dead toggles.
5. **Advanced Options**: each page's "Advanced Options" expander shows additional rows
   from the design. Additionally add a global "All settings (advanced)" entry that opens
   VLC's existing complete preferences tree (`complete_preferences.*`) so nothing is lost.
6. Search: filter rows by label, help text and option name across all pages.
7. Apply model: design shows instant toggles (no Save button) — write config on change
   and save to disk debounced; options that need a restart show a small "restart
   required" note.
8. Open Settings from the app menu (Ctrl+P), the control-bar Settings control and the
   tray menu; retire the old simple-preferences dialog only when every simple-preference
   option is reachable in the new window.

Done when: every option from VLC's simple preferences (all six `sprefs_*.ui` pages) is
reachable in the new window or through the advanced tree; changing a setting persists
across restarts; search finds options by label.

### Phase 10 — Dialogs, onboarding and other new features

Tasks:
1. **Customize Player** (`2107:5604` … `2160:8732`): restyle `ToolbarEditor*.qml` to the
   modal shown (title, hint text, item grid with icon + label, the live bar below with
   dashed drop zones, Cancel / Save). Add the NEW control types from §5.1. Keep profile
   management (the design's frames do not show it; keep VLC's profile selector in the
   dialog header).
2. **Keyboard Shortcuts** (`2437:14832`): searchable table bound to VLC's hotkey
   variables (`key-*` and `global-key-*`), edit by click-to-record, conflict detection.
   Reuse logic from the hotkeys part of `simple_preferences.cpp`.
3. **Open Network Stream** (`6216:13614`): multiple URLs (one per line), optional
   "playlist name" that saves them as a media-library playlist. Keep the full
   widgets-based Open dialog (file/disc/network/capture with advanced options such as
   caching and `:sout` options) reachable via "Advanced…".
4. **New Playlist** (`6259:14954`): name, thumbnail, drop zone; creates a media-library
   playlist.
5. **Add Subtitles to Playlist/Folder** (`1911:7545` and variants): new feature. Pick
   origin (computer folder, or OpenSubtitles via VLC's existing `vlsub` Lua extension if
   present), target (this folder / this file), auto-match subtitle files to videos by
   name (e.g. `S01E04`), show the pairing list, Save writes the association. Decide the
   storage: copy/rename the subtitle next to the video (simplest, works everywhere) or
   store it as a media-library attachment; document the choice.
6. **Onboarding** (`1899:3055`, `1902:*`): welcome → choose media folder (adds to media
   library) → theme + accent → keyboard shortcuts → finish. Must also keep the privacy /
   network-access question VLC's first-run wizard asks (`qt-privacy-ask`,
   metadata fetching), as an extra step styled the same way.
7. **Profiles** (`6648:20666`: Movies, Shows, Music, Work, New): new concept. Suggested
   meaning: a profile is a named bundle of settings (control-bar profile, default view,
   library folders, audio/video settings) the user can switch between. Confirm with the
   project owner before building; it is the least defined feature in the design.
8. Restyle remaining QML popups/menus to the menu style (`2107:5903`, 10 px radius, blur
   + shadow). QMenu-based menus: style them with a QStyle/stylesheet in the same colours
   so native menus match.
9. Restyle (do not rewrite) the remaining Qt Widgets dialogs (Stream wizard, Convert,
   Media Information, Messages, Plugins, VLM, EPG, Help/About) with a Qt Widgets
   stylesheet or palette using the same tokens so they are not visually jarring.

Done when: every dialog in §2.2 exists; the first run of a clean config shows onboarding;
all old dialogs are still reachable and readable in both themes.

### Phase 11 — Rebrand to Moth and package

Tasks:
1. Replace user-visible "VLC" / "VLC media player" strings in the Qt interface
   (`qt.cpp`, `dialogs/help/help.cpp`, `dialogs/systray/systray.cpp`,
   `dialogs/firstrun/*`, `dialogs/preferences/preferences.cpp`,
   `maininterface/qml/UpdatePane.qml`, menu labels such as "Quit VLC", "VLC Settings",
   "About VLC", onboarding copy). Keep "based on VLC" credit text in About.
2. Replace the cone: `widgets/qml/BannerCone.qml` (used in `player/qml/TopBar.qml` and
   `maininterface/qml/LocalTopbar.qml`), the `VoronoiSnow` easter egg in `MainDisplay.qml`,
   the `vlc` icon glyph, window/app icons under `share/` and platform packaging icons.
   New logo concept: a moth whose wings are two rounded screen shapes with an eye-spot on
   each wing; must read at 16 px. Logo design itself is a separate task.
3. Rename the executable, desktop file, app ID, config/cache directory names and
   installer names to Moth so Moth does not overwrite or read a user's VLC config.
   **This touches packaging and `src/` paths; do it last and carefully.**
4. Disable or repoint VLC's update checker (`qt-updates-notif`, `dialogs/help/` update
   code) so Moth never offers to install VLC builds.
5. README, credits (VideoLAN, Figma design author, font licences), LICENSE unchanged
   (GPLv2+).
6. Release builds for the target platforms (Windows installer, Linux AppImage/Flatpak,
   macOS app bundle with the Qt interface).

Done when: nothing in the UI says VLC except the credit; Moth and VLC can be installed
side by side without sharing config.

---

## 7. Verification for every phase

1. **Visual:** screenshot the implemented screen at 1920×1080 at interface scale 1.0 and
   compare side by side with the Figma frame (`get_screenshot`). Check colours with a
   picker against §3. Also check 1280×720 and a small window (~1024×640), and interface
   scale 1.5.
2. **Both themes:** dark and light.
3. **Keyboard only:** every new control reachable and operable without a mouse; focus
   visible.
4. **Screen reader:** new controls have accessible names (spot-check with Orca or
   Narrator).
5. **Performance:** play a 4K HEVC/AV1 file and a 1080p file; CPU/GPU use with the UI
   visible and hidden must not be meaningfully higher than baseline stock VLC on the same
   commit. Library scrolling must stay smooth with ≥2,000 items. Blur effects must respect
   `qt-backdrop-blur` (off = no blur cost).
6. **Feature parity smoke test:** open file, folder, disc (if available), network stream,
   capture device; stream to HTTP; convert a file; record; snapshot; subtitles
   load/sync; audio/subtitle track switching; equaliser; video effects; playlists;
   media library scan; network shares browsing; casting/renderer discovery.
7. **Tests:** existing Qt tests (`modules/gui/qt/tests`, meson `test` targets) still pass;
   add unit tests for new C++ models (waveform cache, settings model, folder-visit stats).
8. **Rebase check:** after each phase, rebase on `upstream/master` and fix conflicts while
   the change is fresh.

---

## 8. Open questions and known discrepancies (keep this list updated)

1. Light-theme tokens are incomplete (§3.2). Collect them from more light frames; derive
   the rest.
2. Frames `187:328`, `970:3308`, `970:4028` are not yet catalogued.
3. Design shortcuts that differ from VLC defaults (e.g. Ctrl+F "Open Folder", Ctrl+C
   "Open Capture Device", Ctrl+R "Convert", Ctrl+P "Settings", Ctrl+\\ "Hide UI", Shift+F1
   "About") — decide per shortcut whether Moth changes the default binding or shows VLC's
   existing one. Default: keep VLC's bindings, add new ones only where VLC has none.
4. "Profiles" (Movies/Shows/Music/Work) semantics — confirm with the project owner.
5. Forward/Backward 15s as separate controls vs. a configurable jump — decide.
6. Settings rows with no VLC equivalent — list them here as they are found.
7. Small-window and tablet-size layouts are not designed — follow the design language.
8. Empty, loading and error states are not designed — follow the design language.
9. Two-tone icons via stacked glyphs (Phase 3) — confirm it renders crisply at 1.0 and
   1.5 scale; fall back to SVG images if not.
10. The Figma file's licence and author attribution — confirm on the community page.

---

## 9. Changelog of this plan

- 2026-10-13 — First version. Repo HEAD at time of writing:
  `28184b3711af3585993cefb22f64740c3696163c` ("qml: fix and animate PlayCover hover
  color"), VLC 4.0.0-dev, Qt 6.
