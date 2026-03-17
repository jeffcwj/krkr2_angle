# cpp/plugins — Plugin Implementations

## OVERVIEW
STATIC library (`krkr2plugin`) containing all engine plugins. Linked directly into the main krkr2 binary — not dynamically loaded.

## STRUCTURE
```
plugins/
├── psdfile/        # PSD parser (psdparse sublib) — large, standalone
├── psbfile/        # PSB/E-mote format parser — types/, resources/ subdirs
├── motionplayer/   # E-mote motion animation player
├── layerex_draw/   # Extended layer drawing — windows/ (GDI+) and general/ variants
├── fstat/          # File statistics plugin
├── steam/          # Steam API integration (DISABLED in build)
├── DrawDeviceForSteam/ # Steam draw device (DISABLED)
├── json/           # JSON plugin (DISABLED — commented out)
├── simplebinder/   # Template-based TJS binding helper (header-only)
├── KAGParser/      # KAG parser plugin variant
└── (root .cpp files) # scriptsEx, csvParser, dirlist, varfile, saveStruct, etc.
```

## WHERE TO LOOK

| Task | File(s) |
|------|---------|
| Add new plugin | Create .cpp, add to `target_sources` in CMakeLists.txt |
| Add plugin with subdir | Create subdir + CMakeLists.txt, add_subdirectory + link in parent CMakeLists.txt |
| TJS binding for plugin | Use `simplebinder/simplebinder.hpp` or ncbind macros from `core/plugin/ncbind.hpp` |
| Plugin registration | Use `NCB_PRE_REGIST_CALLBACK` / `NCB_REGISTER_CLASS` in plugin main.cpp |

## CONVENTIONS
- Root-level plugins: single .cpp added to `target_sources(krkr2plugin ...)` directly
- Sub-directory plugins: separate CMake target (static lib), linked to krkr2plugin
- Plugin includes core headers via KRKR2CORE_PATH variable (set in root CMakeLists)
- `tp_stub.h` / `PluginStub.h` — legacy KiriKiri plugin stub headers, shared across plugins
- Some plugins have `manual.tjs` files for script-side registration

## ANTI-PATTERNS
- `steam/`, `DrawDeviceForSteam/`, `json/` are commented out — do NOT enable without adding their dependencies to vcpkg.json
- Do NOT add platform-specific code in root plugin .cpp files; use `layerex_draw/` pattern (windows/ vs general/ subdirs)
- `win32_dt.h` — Windows-specific types header used by some plugins; prefer cross-platform alternatives
