# cpp/core/base — I/O, Archives, Events, KAG Parser

## OVERVIEW
CMake target `core_base_module` (STATIC). Foundation layer: archive formats, stream I/O, event dispatching, KAG scenario parser, script management, system initialization, message/localization.

## KEY FILES

| File | Role |
|------|------|
| `XP3Archive.cpp/h` | KiriKiri native XP3 format: segmented, compressed (zlib), with extraction filters |
| `ZIPArchive.cpp` | ZIP archive via LibArchive |
| `TARArchive.cpp` | TAR archive via LibArchive |
| `7zArchive.cpp` | 7z archive via LibArchive |
| `BinaryStream.cpp/h` | tTJSBinaryStream — seekable binary I/O base |
| `TextStream.cpp/h` | Unicode text stream (auto-detect encoding via uchardet) |
| `UtilStreams.cpp/h` | Utility stream wrappers |
| `StorageIntf.cpp/h` | Universal storage system: media registry, path resolution, auto-search, caching |
| `EventIntf.cpp/h` | Event queue: script events, input events, continuous/compact callbacks, AsyncTrigger |
| `KAGParser.cpp/h` | KAG (KiriKiri Adventure Game) scenario parser: tags, macros, labels, call stack |
| `ScriptMgnIntf.cpp/h` | Script manager interface |
| `SystemIntf.cpp/h` | System interface (exit, restart) |
| `SysInitIntf.cpp/h` | System initialization / command-line parsing |
| `MsgIntf.cpp/h` | Localized message strings |
| `CharacterSet.cpp/h` | Character encoding conversion |
| `impl/` | Platform-specific implementations (EventImpl, StorageImpl, etc.) |

## WHERE TO LOOK

| Task | File(s) |
|------|---------|
| Add archive format | Create `XxxArchive.cpp` extending `tTVPArchive`; register in `TVPOpenArchive()` |
| XP3 extraction filter | `XP3Archive.h` — `tTVPXP3ArchiveExtractionFilter` callback typedef |
| Storage media plugin | `StorageIntf.h` — implement `iTVPStorageMedia`, register via `TVPRegisterStorageMedia` |
| Event posting | `EventIntf.h` — `TVPPostEvent()`, `TVPPostInputEvent()` |
| KAG tag processing | `KAGParser.cpp` — `GetNextTag()`, macro expansion, label navigation |
| Text encoding | `CharacterSet.cpp` — uses uchardet for auto-detection |
| Path/file utilities | `StorageIntf.h` — TVPExtractStorageName, TVPSearchPlacedPath, TVPNormalizeStorageName |

## CONVENTIONS
- Archive base class `tTVPArchive`: ref-counted, hash-indexed names, `NormalizeInArchiveStorageName` required
- XP3 segment cache limit: `TVPSegmentCacheLimit` (bytes) — tunable
- Event priority flags: `TVP_EPT_POST`, `TVP_EPT_REMOVE_POST`, `TVP_EPT_IMMEDIATE`, `TVP_EPT_EXCLUSIVE`, `TVP_EPT_IDLE`
- Continuous events: implement `tTVPContinuousEventCallbackIntf::OnContinuousCallback(tick)`
- Compact events: `TVP_COMPACT_LEVEL_IDLE`(5) through `TVP_COMPACT_LEVEL_MAX`(100)
- KAGParser: label cache lazily built via `EnsureLabelCache()`; scenario files cached by `tTVPScenarioCacheItem`
- Archive delimiter: `>` character (`TVPArchiveDelimiter`)

## DEPENDENCIES
uchardet, unrar, LibArchive, SDL2, zstd, cocos2dx. Links: tjs2 (PUBLIC), core_visual_module, core_plugin_module, core_environ_module, core_extension_module, core_sound_module, core_utils_module (PRIVATE).

## ANTI-PATTERNS
- `impl/` contains platform bridging — do NOT add new platform-specific code outside `impl/`
- `StorageIntf.h` mixes interface declarations with inline implementations (TArchiveStream) — maintain but don't extend this pattern
