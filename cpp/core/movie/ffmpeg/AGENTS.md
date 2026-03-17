# cpp/core/movie/ffmpeg — FFmpeg Media Player

## OVERVIEW
Full media player stack (85 files) ported from XBMC/Kodi. Demuxes, decodes, syncs, and renders video/audio via FFmpeg. Namespace: `NS_KRMOVIE_BEGIN`/`NS_KRMOVIE_END`.

## ARCHITECTURE

```
KRMoviePlayer.h   ← TVPMoviePlayer (iTVPVideoOverlay + CBaseRenderer)
├── VideoPlayer.h  ← BasePlayer (main loop thread: demux→decode→render)
│   ├── DemuxFFmpeg     ← CDVDDemuxFFmpeg: reads AVPackets via libavformat
│   ├── VideoPlayerVideo ← IDVDStreamPlayerVideo: decodes via VideoCodecFFmpeg
│   ├── VideoPlayerAudio ← IDVDStreamPlayerAudio: decodes via AudioCodecFFmpeg
│   ├── Clock            ← CDVDClock: A/V sync master clock
│   └── CRenderManager   ← frame queue + flip logic (NUM_BUFFERS=6)
├── KRMovieLayer.h ← VideoPresentLayer: renders to tTVPBaseTexture (layer mode)
└── VideoRenderer.h ← CRenderManager: buffer queue, present steps, clock sync
```

## KEY CLASSES

| Class | Role |
|-------|------|
| `TVPMoviePlayer` | Base: implements iTVPVideoOverlay, receives decoded frames |
| `VideoPresentOverlay` | Renders frames via cocos2d::Node + YUV sprite |
| `MoviePlayerOverlay` | Overlay mode: attaches to tTJSNI_Window |
| `VideoPresentLayer` | Layer mode: writes to tTVPBaseTexture double-buffer |
| `MoviePlayerLayer` | Layer mode entry: BuildGraph + play events |
| `BasePlayer` | Core loop: CThread subclass, owns demuxer + stream players |
| `CDVDDemuxFFmpeg` | FFmpeg demuxer wrapping AVFormatContext |
| `CRenderManager` | Frame queue (free→queued→discard), flip/present pipeline |
| `CDVDClock` | Sync clock with speed adjustment |

## WHERE TO LOOK

| Task | File(s) |
|------|---------|
| Add codec support | `FactoryCodec.cpp` — codec factory registration |
| Fix A/V sync | `Clock.cpp`, `VideoPlayerAudio.cpp` — clock master + audio sync |
| Debug frame drops | `VideoRenderer.cpp` — m_lateframes, m_QueueSkip counters |
| Modify demuxer behavior | `DemuxFFmpeg.cpp` — Open(), Read(), SeekTime() |
| Change render format | `RenderFormats.h` — ERenderFormat enum |
| Audio engine integration | `AudioDevice.cpp`, `AE*.cpp` — audio engine (ported from Kodi) |
| Add video filter/effect | `BaseRenderer.cpp` — Configure(), AddVideoPicture() |
| Thread/message issues | `Thread.cpp`, `MessageQueue.cpp`, `Message.cpp` |

## CONVENTIONS
- All classes in `KRMovie` namespace (via NS_KRMOVIE_BEGIN/END macros)
- XBMC/Kodi heritage: DVD_NOPTS_VALUE sentinel, CDVDMsg message system, CThread base
- Frame buffer ring: MAX_BUFFER_COUNT=4 (KRMoviePlayer), NUM_BUFFERS=6 (renderer)
- OMXClock is a dummy stub (no OMX acceleration) — kept for interface compat
- Streams use IStream COM interface, not std::istream
- Events dispatched via `std::function<void(KRMovieEvent, void*)>` callback

## ANTI-PATTERNS
- Many stub methods in TVPMoviePlayer (SetStopFrame, overlay functions) — not implemented
- Do NOT remove OMXClock dummy — code references it throughout BasePlayer
- AE* (AudioEngine) files are ported from Kodi — heavily coupled, modify with care
