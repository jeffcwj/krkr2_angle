# tests — Unit Tests

## OVERVIEW
Catch2 v3 test suite. Each test module is a separate executable with custom `main()` (spdlog init + `Catch::Session().run()`). Test fixtures via `TEST_FILES_PATH` configured at CMake time.

## STRUCTURE
```
tests/
├── CMakeLists.txt         # Top-level: find Catch2, configure TEST_FILES_PATH
├── test_config.h.in       # Generates test_config.h with TEST_FILES_PATH
├── test_files/
│   ├── tjs2/              # .tjs script fixtures
│   └── emote/             # .pimg, .psb binary fixtures
└── unit-tests/
    ├── core/
    │   ├── movie/         # TestMovieModule: ffmpeg init test
    │   ├── tjs2/          # TestTjs2: TJS engine + string tests
    │   └── visual/        # Visual module tests
    └── plugins/           # Plugin tests (psbfile)
```

## HOW TO ADD A TEST

1. Create `tests/unit-tests/<module>/your_test.cpp`:
   ```cpp
   #include <catch2/catch_test_macros.hpp>
   #include "test_config.h"  // for TEST_FILES_PATH
   TEST_CASE("descriptive name") { /* ... */ }
   ```
2. Add `your_test.cpp` to `SOURCES` in that module's `CMakeLists.txt`
3. The CMake foreach auto-creates executable, links Catch2 + module lib, discovers tests

## CONVENTIONS
- **Custom main() per target:** Each test dir has `main.cpp` that inits spdlog loggers then runs `Catch::Session()`
- **CMake pattern:** `string(REPLACE ".cpp" "" BASENAMES_SOURCES "${SOURCES}")` → one executable per source file
- **Linking:** Tests link `Catch2::Catch2` (PRIVATE) + relevant module lib (PUBLIC, e.g., `krkr2core`, `core_movie_module`)
- **Test discovery:** `catch_discover_tests()` for CTest integration
- **Fixtures:** Use `TEST_FILES_PATH` macro from `test_config.h` to access `test_files/` directory
- **Config dir:** `target_include_directories(${name} PRIVATE "${TEST_CONFIG_DIR}")` to find generated `test_config.h`

## WHERE TO LOOK

| Task | File(s) |
|------|---------|
| Add test module | Create `unit-tests/<module>/` with CMakeLists.txt + main.cpp following existing pattern |
| Add fixture files | Place in `test_files/<module>/`, access via `TEST_FILES_PATH "/<module>/filename"` |
| Run tests | `ctest --test-dir out/<platform>/debug --output-on-failure` |

## ANTI-PATTERNS
- Do NOT use `Catch2::Catch2WithMain` — project uses custom main() with spdlog init
- Do NOT hardcode fixture paths — always use `TEST_FILES_PATH` from test_config.h
