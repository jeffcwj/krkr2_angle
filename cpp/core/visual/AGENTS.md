# cpp/core/visual — Rendering & Image Subsystem

## OVERVIEW
Image loading, layer management, bitmap operations, transitions, and OpenGL rendering for the KiriKiri2 engine. Target: `core_visual_module`.

## STRUCTURE
```
visual/
├── ogl/                    # OpenGL rendering backend
│   ├── RenderManager_ogl.cpp  # OGL render manager
│   ├── pvrtc.*, pvr.h      # PVRTC texture compression
│   ├── etcpak.*             # ETC texture packing
│   ├── astcrt.*             # ASTC texture compression
│   └── imagepacker.*        # Texture atlas packing
├── Load*.cpp               # Image format loaders (one per format)
│   ├── LoadPNG, LoadJPEG, LoadTLG, LoadWEBP
│   ├── LoadBPG, LoadJXR, LoadPVRv3
├── SaveTLG*.cpp            # TLG5/TLG6 savers
├── Layer*.{h,cpp}          # Layer system: LayerIntf, LayerBitmapIntf, LayerManager, LayerTreeOwner
├── Bitmap*.{h,cpp}         # Bitmap management
├── Window*.{h,cpp}         # Window interface
├── Trans*.{h,cpp}          # Transition effects
├── RenderManager.*         # Abstract render manager
├── VideoOvlIntf.*          # Video overlay interface
├── CharacterData.*         # Font/character rendering
├── PrerenderedFont.*       # Pre-rendered bitmap fonts
├── tvpgl.*                 # Low-level graphics library (pixel ops)
├── tvpgl_asm_init.h        # ASM-optimized graphics init
├── argb.*                  # ARGB pixel operations
└── tvpps.inc               # Pixel shader include
```

## WHERE TO LOOK

| Task | File(s) |
|------|---------|
| Add image format | Create `Load{Format}.cpp`, follow `LoadPNG.cpp` pattern, register in CMakeLists |
| Modify layer rendering | `LayerIntf.cpp`, `LayerBitmapIntf.cpp` |
| Change OGL pipeline | `ogl/RenderManager_ogl.cpp` |
| Add texture compression | `ogl/` — follow etcpak/pvrtc/astcrt pattern |
| Transition effects | `TransIntf.cpp`, `transhandler.h` |
| Pixel operations | `tvpgl.cpp` (C), `argb.cpp` |

## CONVENTIONS
- One loader file per image format: `Load{Format}.cpp`
- TLG is KiriKiri's native image format (TLG5/TLG6) — both load and save supported
- `tvpgl` — low-level C-style pixel operations, some with SSE/ASM variants (`tvpgl_asm_init.h`)
- Layer system uses `Intf` suffix for interface classes, `Impl` for implementations
- OGL subdirectory handles GPU texture formats (PVRTC, ETC, ASTC) for mobile
- `RenderManager` is abstract; `RenderManager_ogl` and `RenderManager_software` are backends
- `voMode.h` — video overlay mode definitions
- `tvpfontstruc.h` — font structure definitions
- `tvpinputdefs.h`, `tvphal.h` — input/HAL definitions shared across visual subsystem

## ANTI-PATTERNS
- Do NOT add format-specific logic outside Load*.cpp files
- `tvpgl` functions are performance-critical — profile before modifying
