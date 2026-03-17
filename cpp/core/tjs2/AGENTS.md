# cpp/core/tjs2 — TJS2 Scripting Engine

## OVERVIEW
Full TJS2 (T Visual Presenter JavaScript-like) scripting language implementation: lexer, parser (Bison-generated), bytecode compiler, virtual machine, object system.

## STRUCTURE
```
tjs2/
├── bison/              # Grammar files: tjs.y, tjsdate.y, tjspp.y
├── script/             # Python helper (create_world_map.py), word tables
├── tjs.{h,cpp}         # Top-level engine class (tTJS)
├── tjsLex.{h,cpp}      # Lexical analyzer
├── tjsInterCodeGen.*   # Intermediate code generator (compiler)
├── tjsInterCodeExec.*  # Bytecode VM executor
├── tjsByteCodeLoader.* # Bytecode serialization/deserialization
├── tjsObject.*         # Core object system
├── tjsVariant.*        # Variant type (dynamic typing)
├── tjsString.*         # String implementation
├── tjsArray.*          # Array object
├── tjsDictionary.*     # Dictionary object
├── tjsRegExp.*         # Regular expression support (oniguruma)
├── tjsDate.*           # Date object + parser
├── tjsMath.*           # Math builtins
├── tjsNative.*         # Native function binding
├── tjsInterface.*      # Interface definitions
├── tjsNamespace.*      # Namespace management
├── tjsScriptBlock.*    # Script block (compilation unit)
├── tjsScriptCache.*    # Compiled script caching
├── tjsException.*      # Exception handling
└── tjsConfig.*         # TJS configuration
```

## WHERE TO LOOK

| Task | File(s) |
|------|---------|
| Add TJS builtin function | `tjsMath.cpp` (example), use native binding pattern |
| Modify parser grammar | `bison/tjs.y` — requires bison 3.8.2+ to regenerate |
| Change lexer rules | `tjsLex.cpp` |
| Debug bytecode execution | `tjsInterCodeExec.cpp` — main VM dispatch loop |
| Add new object type | Follow `tjsArray.*` or `tjsDictionary.*` pattern |
| Understand variant system | `tjsVariant.h` — central dynamic type |

## CONVENTIONS
- All files prefixed with `tjs` — namespace-by-convention
- Bison grammars in `bison/` must be regenerated with bison 3.8.2+ (winflexbison on Windows)
- `tjsCommHead.h` — common precompiled header included by most files
- `tjsConfig.h` — compile-time configuration flags
- `tjsTypes.h` — fundamental type definitions
- `tjsHashSearch.h` — custom hash map implementation (not std::unordered_map)
- Uses Mersenne Twister RNG (`tjsMT19937ar-cok.*`)
- `tjsGlobalStringMap` — interned string pool for performance

## ANTI-PATTERNS
- Do NOT use std::string for TJS string operations — use `tTJSString` / `tTJSVariantString`
- Do NOT modify generated parser output directly — edit `.y` files and regenerate
- `tjsOctPack.*` — octave packing, legacy code; avoid extending
