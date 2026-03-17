# cpp/core/sound — Audio Subsystem

## OVERVIEW
CMake target `core_sound_module` (STATIC). Wave decoding, playback, DSP filtering, loop management. Uses OpenAL for output, Vorbis/Opus/FFmpeg for decoding. Android uses Oboe additionally.

## ARCHITECTURE

```
WaveIntf.h          ← tTVPWaveDecoder (abstract decoder interface)
├── FFWaveDecoder   ← Decodes via FFmpeg (any format FFmpeg supports)
├── VorbisWaveDecoder ← Ogg Vorbis decoder (libvorbisfile)
└── OpusWaveDecoder   ← Opus decoder (libopusfile)

SoundBufferBaseIntf.h ← tTJSNI_BaseSoundBuffer (volume, fade, status)
└── tTJSNI_SoundBuffer  ← Platform impl (win32/SoundBufferBaseImpl)

WaveLoopManager     ← Loop point management with labels/links
WaveSegmentQueue    ← Segment-based queued playback
PhaseVocoderDSP/Filter ← Time-stretch without pitch change
MathAlgorithms      ← FFT, windowing functions for DSP
```

## KEY FILES

| File | Role |
|------|------|
| `WaveIntf.h/cpp` | Decoder interface, creator registry, PCM format conversion |
| `SoundBufferBaseIntf.h/cpp` | Base sound buffer: status, fade, volume |
| `FFWaveDecoder.cpp` | FFmpeg-based decoder (generic format support) |
| `VorbisWaveDecoder.cpp` | Ogg Vorbis + Opus decoders |
| `WaveLoopManager.cpp` | Loop points, cross-fade labels, link conditions |
| `PhaseVocoderDSP.cpp` | Phase vocoder for time-stretching |
| `CDDAIntf.cpp` / `MIDIIntf.cpp` | CD audio and MIDI stubs |
| `win32/WaveImpl.cpp` | tTJSNI_WaveSoundBuffer platform implementation |
| `win32/WaveMixer.cpp` | OpenAL-based mixer |

## WHERE TO LOOK

| Task | File(s) |
|------|---------|
| Add audio format | Create `XxxDecoder` implementing `tTVPWaveDecoder`, register via `TVPRegisterWaveDecoderCreator` |
| Fix playback issues | `win32/WaveImpl.cpp` — buffer management, OpenAL calls |
| Loop behavior | `WaveLoopManager.cpp` — labels, links, conditions |
| DSP/time-stretch | `PhaseVocoderDSP.cpp`, `PhaseVocoderFilter.cpp` |
| PCM conversion | `WaveIntf.cpp` — TVPConvertPCMTo16bits, TVPConvertPCMToFloat |
| Android audio | CMakeLists.txt — Oboe linked on ANDROID |

## CONVENTIONS
- Decoder registration: implement `tTVPWaveDecoderCreator::Create()`, call `TVPRegisterWaveDecoderCreator()` at init
- PCM format described by `tTVPWaveFormat` struct (samples/sec, channels, bits, float flag)
- Sound status enum: `ssUnload`, `ssStop`, `ssPlay`, `ssPause`, `ssReady`
- Fading handled at base class level: `Fade(to, time, blanktime)` with periodic TimerBeatHandler
- TJS2 binding: `tTJSNC_WaveSoundBuffer` native class exposed to script

## DEPENDENCIES
Vorbis, Opus, OpusFile, OpenAL (all platforms). Oboe (Android only). Links: tjs2 (PUBLIC), core_base_module, core_movie_module, core_environ_module, core_utils_module (PRIVATE).

## ANTI-PATTERNS
- `WaveFormatConverter_SSE.cpp` and `xmmlib.cpp` are commented out in CMakeLists — SSE path disabled
- `win32/` subdir name is misleading — used on all platforms, not Windows-only
