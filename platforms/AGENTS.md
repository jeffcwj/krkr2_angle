# platforms — Platform Entry Points

## OVERVIEW
Per-platform entry points and bootstrap. Each platform constructs `TVPAppDelegate` and calls `run()`. Actual engine logic lives in `cpp/core/environ/` — this directory is thin glue only.

## STRUCTURE
```
platforms/
├── android/        # Gradle project (app + libcocos2dx + libsdl)
│   ├── cpp/krkr2_android.cpp   # JNI bridge: cocos_android_app_init + native* functions
│   ├── app/                    # Android app module (Java/Kotlin)
│   ├── libcocos2dx/            # Cocos2d-x Android Java bridge
│   └── libsdl/                 # SDL2 Android Java bridge
├── windows/
│   ├── main.cpp    # _tWinMain → TVPAppDelegate::run()
│   ├── game.rc     # Windows resource script (icons, version)
│   └── res/        # Icon assets
├── linux/
│   └── main.cpp    # main() with gtk_init → TVPAppDelegate::run()
└── apple/macos/
    ├── main.cpp    # main() → TVPAppDelegate::run()
    ├── Info.plist  # macOS bundle config
    └── Icon.icns   # App icon
```

## WHERE TO LOOK

| Task | File(s) |
|------|---------|
| Add platform entry logic | `<platform>/main.cpp` — add before `pAppDelegate->run()` |
| Android JNI bindings | `android/cpp/krkr2_android.cpp` — all Java_org_tvp_kirikiri2_* functions |
| Android touch/input | `krkr2_android.cpp` — nativeTouches*, nativeKeyAction, nativeHoverMoved |
| Android Gradle config | `android/build.gradle`, `android/app/build.gradle` |
| Windows resources | `windows/game.rc`, `windows/res/` |
| macOS bundle settings | `apple/macos/Info.plist` |

## CONVENTIONS
- **Boot sequence:** All platforms: init spdlog loggers ("core", "tjs2", "plugin") → construct TVPAppDelegate → call run()
- **Arg handling:** Linux passes argv[1] to `TVPMainFileSelectorForm::filePath`; Windows uses CommandLineToArgvW + boost::locale UTF conversion
- **Android specifics:**
  - Entry via `cocos_android_app_init(JNIEnv*)` (not main())
  - google_breakpad for crash dumps (initDump JNI call)
  - Touch events proxied via `Android_PushEvents()` into cocos2d thread
  - SDL2 loaded dynamically: `dlopen("libSDL2.so", RTLD_LAZY)`
  - Input key mapping: KEYCODE_BACK→KEY_ESCAPE, etc.
- **No engine logic here** — platform dirs are bootstrap only; all behavior goes in `cpp/core/environ/`

## ANTI-PATTERNS
- Do NOT add engine features in platform mains — use `cpp/core/environ/` platform subdirs
- Android `krkr2_android.cpp` has VLA (`jint id[size]`) — non-standard C++, do not spread
