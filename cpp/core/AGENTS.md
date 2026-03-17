# cpp/core — Engine Core

## OVERVIEW
INTERFACE library (`krkr2core`) linking 9 subsystem modules. Heart of the KiriKiri2 emulator engine.

## STRUCTURE
```
core/
├── base/       # I/O, archives, events, KAG parser     → core_base_module
├── common/     # Shared header (Defer.h only)
├── environ/    # Platform abstraction + UI forms        → core_environ_module
├── extension/  # Engine extension API                   → core_extension_module
├── movie/      # Video via ffmpeg                       → core_movie_module
├── plugin/     # Plugin loading, ncbind                 → core_plugin_module
├── sound/      # Audio decoders, DSP                    → core_sound_module
├── tjs2/       # TJS2 scripting VM                      → tjs2
├── utils/      # Timers, threads, RNG                   → core_utils_module
└── visual/     # Rendering, image loaders, layers       → core_visual_module
```

## WHERE TO LOOK

| Task | File(s) |
|------|---------|
| Add dependency to core | `CMakeLists.txt` — add find_package + target_link_libraries to INTERFACE |
| Change compile definitions | `CMakeLists.txt` — target_compile_definitions block |
| Platform-specific linking | `CMakeLists.txt` — ANDROID/APPLE conditionals at bottom |
| Understand module deps | Each subdir has own CMakeLists.txt with its target name |

## CONVENTIONS
- All modules built as separate CMake targets, linked via INTERFACE lib
- Cocos2d-x linked at this level: `find_package(cocos2dx CONFIG REQUIRED)`
- Android links: log, android, EGL, GLESv2, GLESv1_CM, OpenSLES
- Include path for common/ set via `include_directories` (not target-scoped)

## ANTI-PATTERNS
- Do NOT add platform-specific code here — use `environ/` subdirs
- `common/` has a single header; avoid growing it into a dump-all utils dir
