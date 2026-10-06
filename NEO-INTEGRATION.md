# Steam Audio as a dependency of NEOTOKYO;Rebuild (neo)

This checkout is an untouched mirror of `ValveSoftware/steam-audio` (v4.8.1). It is kept as a
sibling of the `neo` repo so the game can take upstream Steam Audio updates by fast-forwarding
this branch, and so `phonon` can be rebuilt from source when a prebuilt release is not enough.
Nothing in this repo is neo-specific except this file.

## How neo consumes Steam Audio

neo never links against `phonon`. The whole contact surface is:

| What | Where in `neo` | Source of truth here |
| --- | --- | --- |
| Public C headers (`phonon.h`, `phonon_interfaces.h`, `phonon_version.h`) | `src/thirdparty/steamaudio/include/` (copied verbatim) | `core/src/core/phonon.h`, `core/src/core/phonon_interfaces.h`, generated `phonon_version.h` in the release zip |
| Runtime library (`libphonon.so` / `phonon.dll`) | downloaded by CMake (`src/cmake/steamaudio.cmake`) from the GitHub release zip into `game/neo/bin/<plat>/`, loaded at runtime with `dlopen` / `LoadLibrary` | release artefacts (`steamaudio_<ver>.zip`), or a local build of `core/` |
| Backend code | `src/game/client/neo/audio/neo_spatializer_steamaudio.cpp`, the only neo file that includes `phonon.h` | — |

Everything else in neo talks to the backend-neutral `NeoSpatial::ISpatializer` interface
(`src/game/client/neo/audio/neo_spatializer.h`), so a Steam Audio upgrade is:

1. Fast-forward this repo to the upstream tag.
2. In neo, bump the release URL and SHA256 in `src/cmake/steamaudio.cmake` and recopy the three
   headers into `src/thirdparty/steamaudio/include/`.
3. Rebuild. If a function neo loads at runtime was renamed, `CreateSteamAudioSpatializer` reports
   the missing symbol in its error string and the game falls back to the panner backend.

## Building phonon from this source instead of downloading it

Useful for debugging inside Steam Audio, or for a platform the release zip does not cover.
Follow `core/doc/build-instructions.rst`; the short version for Linux x64:

```sh
cd core/build
python3 get_dependencies.py --platform linux          # downloads/builds pffft, mysofa, flatbuffers, ...
cd ..
cmake -S . -B build/linux-x64 -DCMAKE_BUILD_TYPE=Release \
      -DSTEAMAUDIO_BUILD_TESTS=OFF -DSTEAMAUDIO_BUILD_BENCHMARKS=OFF -DSTEAMAUDIO_BUILD_SAMPLES=OFF
cmake --build build/linux-x64 --target install        # produces an SDK-shaped tree under bin/
```

Then point neo at that tree (it must contain `include/phonon.h` and `lib/<plat>/libphonon.so` or
`lib/<plat>/phonon.dll`, the same layout as the release zip):

```sh
cd neo/src
cmake --preset linux-debug -DNEO_STEAMAUDIO_SDK_PATH=/path/to/steam-audio/core/bin/steamaudio
```

With that cache variable set, neo's CMake skips the download entirely.

## Scope of the current integration (proof of concept)

- Binaural rendering only: `iplBinauralEffectApply` with the default HRTF, bilinear interpolation.
- Not yet used: scene geometry / occlusion (`IPLScene`, `IPLSimulator`), reflections, pathing,
  custom SOFA HRTFs. The neo-side interface already reserves `SetSceneGeometry()` for the
  occlusion phase, which would feed the map's collision mesh into an `IPLStaticMesh`.
