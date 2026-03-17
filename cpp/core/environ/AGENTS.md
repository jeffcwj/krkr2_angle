# cpp/core/environ — Platform Abstraction Layer

## OVERVIEW
Platform-specific implementations, Cocos2d-x bridge, UI forms, configuration managers, and SDL integration. Target: `core_environ_module`.

## STRUCTURE
```
environ/
├── cocos2d/        # Cocos2d-x application bridge (AppDelegate, MainScene, YUVSprite, CustomFileUtils)
├── ui/             # UI forms (Cocos-based): menus, preferences, dialogs, file selectors
│   ├── csd/        # Cocos Studio data helpers
│   └── extension/  # UI extension components
├── ConfigManager/  # Global, Individual, Locale config managers
├── win32/          # Windows-specific implementations
├── android/        # Android-specific implementations
├── linux/          # Linux-specific implementations
├── apple/          # macOS/iOS-specific implementations
├── sdl/            # SDL2 abstraction layer
├── Application.*   # Base application class
├── Platform.h      # Platform detection macros
├── DetectCPU.*     # CPU feature detection (SIMD)
├── XP3ArchiveRepack.* # XP3 archive repack utility
└── combase.h, cpu_types.h, typedefine.h, vkdefine.h  # Common type/platform headers
```

## WHERE TO LOOK

| Task | File(s) |
|------|---------|
| App startup/lifecycle | `cocos2d/AppDelegate.cpp` — applicationDidFinishLaunching is THE boot function |
| Main scene setup | `cocos2d/MainScene.cpp` |
| Add UI form | `ui/` — follow existing Form pattern (BaseForm subclass), register in AppDelegate |
| Platform-specific code | `win32/`, `android/`, `linux/`, `apple/`, `sdl/` |
| Configuration changes | `ConfigManager/` — Global (app-wide), Individual (per-game), Locale (i18n) |
| Custom file I/O | `cocos2d/CustomFileUtils.cpp` (.mm for macOS) |
| CPU detection/SIMD | `DetectCPU.cpp` |

## CONVENTIONS
- `TVPAppDelegate` (in cocos2d/) is the central application object created by all platform mains
- UI forms inherit from `BaseForm` (ui/BaseForm.h), use Cocos Studio .csb assets from `ui/cocos-studio/`
- Platform subdirs (win32/, android/, etc.) implement abstract interfaces defined in core modules
- `Platform.h` — prefer using these macros over raw `#ifdef _WIN32` etc.
- SDL used as cross-platform input/window abstraction (sdl/ subdir)
- Config managers use separate Global/Individual/Locale pattern — do not merge them

## ANTI-PATTERNS
- Do NOT put rendering code here — use `visual/` module
- Do NOT put business logic in UI forms — forms are presentation only
- Avoid `#ifdef PLATFORM` in cocos2d/ files — delegate to platform subdirs instead
