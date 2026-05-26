# MultiplayerSample audio migration brief

**Goal:** Migrate MultiplayerSample's audio code from the legacy O3DE `AudioSystem` (AzAudio / ATL) APIs to the modern `MiniAudio` APIs, so that audio works on Linux and the project's audio stack matches the engine's current direction.

This document is a self-contained hand-off for a contributor picking up this task. It assumes no prior context on the project beyond what's written here.

---

## TL;DR

- MultiplayerSample currently calls `AudioTriggerComponent` / `AudioProxyComponent` / `AudioListenerComponent` (legacy AzAudio) APIs from its C++ source, ScriptCanvas graphs, and prefabs.
- On Linux, no AzAudio backend implementation is enabled in MPS's gem set. Every `AudioObject` allocation fails with "Failed to get a new instance of an AudioObject from the implementation." Log floods, main loop slows down, eventual cascade-crash under sustained play.
- The engine's `MiniAudio` Gem (`o3de/o3de`'s `Gems/MiniAudio/`) is already enabled in MPS's `project.json` but never called from MPS's own code.
- The migration is **architecturally non-trivial** because AzAudio uses one-component-plays-many-string-triggers while MiniAudio uses one-component-per-sound-asset. Each entity needs structural rework, not a search-and-replace.
- Estimated effort: 1 to 4 days focused work for someone familiar with O3DE audio. 1 to 2 weeks for a contributor learning O3DE audio as they go.
- Use the maintainer's fork at https://github.com/nickschuetz/o3de-multiplayersample for the work; PR back to upstream `o3de/o3de-multiplayersample`.

---

## Context that motivated this work

A community contributor running MultiplayerSample on Fedora 44 / Linux found:

1. The launcher loads `startmenu` and gameplay works (player movement, network sync, NewStarbase rendering, multiplayer round timing).
2. Sustained play produces hundreds of `[Error] (System) - Failed to get a new instance of an AudioObject from the implementation` log entries. Eventually both client and server abort (the server times out on packet processing because audio errors slow the main loop).
3. Reproduced as `make play-mps-host` + `make play-mps-client` in https://github.com/nickschuetz/o3de-rpm.

The root cause is confirmed on the O3DE Discord (`gems-and-features` channel, 2026-05-22): MPS needs to migrate to MiniAudio. MiniAudio does not use the `AudioSystem`, `AudioObject`, or ATL pathway at all; it is independent of the legacy stack. That confirms the bug isn't a tunable pool-size problem (the engine's own error message suggests bumping `s_AudioObjectPoolSize`, but the pool isn't the bottleneck; the backend implementation is missing entirely).

The fork's maintainer pushed [PR #502](https://github.com/o3de/o3de-multiplayersample/pull/502) (cmake fix unrelated to audio) at the same time. Audio is the next pending issue.

---

## Working directory + fork setup

The canonical local checkout:

```
$HOME/PROJECTS/o3de-multiplayersample/
```

with remotes:

```
origin    https://github.com/nickschuetz/o3de-multiplayersample.git
upstream  https://github.com/o3de/o3de-multiplayersample.git
```

For new work:

```bash
cd $HOME/PROJECTS/o3de-multiplayersample
git fetch upstream development
git checkout -b miniaudio-migration upstream/development
```

PR target when ready: `o3de/o3de-multiplayersample:development` from `nickschuetz/o3de-multiplayersample:miniaudio-migration`.

Engine install for testing: `/opt/O3DE/26.05.0` (the Fedora `o3de2605` RPM). The maintainer's Linux box has this set up already; for other environments, see https://github.com/nickschuetz/o3de-rpm for how to install.

The companion assets repo is at `$HOME/PROJECTS/o3de-multiplayersample-assets/` (origin: `https://github.com/o3de/o3de-multiplayersample-assets.git`). Audio assets live there. Probably needs touching as part of this work.

---

## The 16 files to migrate

### C++ source (2 components)

```
Gem/Code/Source/Components/BackgroundMusicComponent.cpp    ~110 lines
Gem/Code/Source/Components/BackgroundMusicComponent.h
Gem/Code/Source/Components/Multiplayer/GameplayEffectsComponent.cpp    ~206 lines
Gem/Code/Source/Components/Multiplayer/GameplayEffectsComponent.h
```

Both currently:

```cpp
#include <LmbrCentral/Audio/AudioProxyComponentBus.h>
#include <LmbrCentral/Audio/AudioTriggerComponentBus.h>

m_audioSystem = AZ::Interface<Audio::IAudioSystem>::Get();
Audio::AudioTriggerNotificationBus::Handler::BusConnect(Audio::TriggerNotificationIdType{ GetEntityId() });
// ...
LmbrCentral::AudioProxyComponentRequestBus::Event(/* ... */);
```

These need to be rewritten against the MiniAudio API surface (see component-mapping table below).

### ScriptCanvas graphs (5 files)

```
scriptcanvas/JumpPad.scriptcanvas
scriptcanvas/PredictiveJumpPad.scriptcanvas
scriptcanvas/AudioTester.scriptcanvas
scriptcanvas/ShieldGeneratorRoundEffects.scriptcanvas
scriptcanvas/EnableAudioListener.scriptcanvas
```

Each contains nodes referencing `AudioTriggerComponent` / `AudioProxyComponent` / `AudioListenerComponent` buses. Need to be re-authored in the ScriptCanvas editor to use the corresponding MiniAudio nodes (`MiniAudioPlaybackComponent` / `MiniAudioListenerComponent`).

`AudioTester.scriptcanvas` is probably the simplest case to start with: the prefab pairs to it (`Prefabs/AudioTester.prefab`) and the script just tests audio playback. Good first migration target to learn the pattern.

### Prefabs (7 files)

```
Prefabs/AudioTester.prefab
Prefabs/Sound_Effect.prefab
Prefabs/Energy_Ball.prefab
Prefabs/GamePlay_Effects.prefab
Prefabs/Player.prefab
Prefabs/BubbleBall.prefab
Levels/NewStarbase/NewStarbase.prefab
```

Each contains entities with one or more of:
- `AudioProxyComponent`  → remove (no MiniAudio equivalent)
- `AudioTriggerComponent` → replace with `MiniAudioPlaybackComponent`
- `AudioListenerComponent` → replace with `MiniAudioListenerComponent`

Editing prefabs is easiest via the Editor's prefab outliner; direct JSON editing of the `.prefab` files works but is error-prone for complex prefabs.

---

## Component-mapping table

| AzAudio (legacy, ATL/Wwise-shaped) | MiniAudio replacement | Migration nature |
|---|---|---|
| `AudioListenerComponent` + `AudioListenerComponentRequest` | `MiniAudioListenerComponent` + `MiniAudioListenerRequestBus` | Direct 1:1 swap. Easiest. |
| `AudioProxyComponent` + `AudioProxyComponentBus` | (none) | Remove entirely; MiniAudio doesn't use a proxy. |
| `AudioTriggerComponent` + `AudioTriggerComponentBus` | `MiniAudioPlaybackComponent` + `MiniAudioPlaybackRequestBus` | Semantic mismatch -- see "Architectural mismatch" below. |
| `Audio::AudioTriggerNotificationBus` (trigger-finished callback) | MiniAudio uses a different notification mechanism | Needs port. Look in `Gems/MiniAudio/Code/Include/MiniAudio/` for the equivalent event bus. |
| `Audio::TAudioControlID` (string-named trigger ID) | `AZ::Data::Asset<SoundAsset>` / `SoundAssetRef` (asset-reference model) | Structural change. |

### Key AzAudio API methods being used

From `LmbrCentral/Audio/AudioTriggerComponentBus.h`:

```cpp
virtual void Play() = 0;
virtual void Stop() = 0;
virtual void ExecuteTrigger(const char* triggerName) = 0;
virtual void KillTrigger(const char* triggerName) = 0;
virtual void KillAllTriggers() = 0;
```

### Corresponding MiniAudio API (from `Gems/MiniAudio/Code/Include/MiniAudio/MiniAudioPlaybackBus.h` in the engine source)

```cpp
virtual void Play() = 0;
virtual void Stop() = 0;
virtual void Pause() = 0;
virtual void SetLooping(bool loop) = 0;
virtual bool IsLooping() const = 0;
virtual void SetSoundAsset(AZ::Data::Asset<SoundAsset> soundAsset) = 0;
virtual AZ::Data::Asset<SoundAsset> GetSoundAsset() const = 0;
virtual void SetVolumePercentage(float volume) = 0;
virtual float GetVolumePercentage() const = 0;
// ...plus positional/directional controls (cone attenuation, direction vectors)
```

`Play()` and `Stop()` line up directly. `ExecuteTrigger(name)` and `KillTrigger(name)` do not -- MiniAudio plays whatever sound is bound to the component, not a named trigger from an external catalog.

---

## Architectural mismatch (the biggest gotcha)

**AzAudio model:** one `AudioTriggerComponent` per entity plays many triggers identified by string names. Each `ExecuteTrigger("Play_Gem_Pickup")` and `KillTrigger("Play_Background_Music")` resolves through ATL → backend (Wwise) configuration external to the project. Sound mapping lives in `.atl_audio_controls` XML files; actual sound files are referenced indirectly.

**MiniAudio model:** one `MiniAudioPlaybackComponent` per sound asset. The sound is bound to the component via `SoundAssetRef` (a direct `.wav` / `.ogg` / `.mp3` reference). To play multiple distinct sounds from one entity, you either:

1. Add multiple `MiniAudioPlaybackComponent` instances (one per sound). Preferred.
2. Swap the `SoundAssetRef` at runtime via `SetSoundAssetRef()` then `Play()`. Single component, but contention if you need overlapping sounds.

This means an entity that today has one `AudioTriggerComponent` playing 5 trigger names becomes an entity with 5 `MiniAudioPlaybackComponent` instances. Each entity in the affected prefabs needs to be reshaped.

---

## Asset side considerations

MPS likely ships `.atl_audio_controls` XML files (look under `o3de-multiplayersample-assets/Gems/<gem_name>/Assets/...`) that map trigger names to backend IDs. These don't translate to MiniAudio's model.

What probably needs to happen:

1. Audit the `.atl_audio_controls` files to enumerate every trigger name actually used.
2. For each trigger, find the underlying sound file in the assets repo (probably `.wav` or `.ogg`).
3. Bind each sound file directly to the appropriate `MiniAudioPlaybackComponent` in the prefabs.

If MPS's audio was authored against Wwise events with mixing / DSP effects / RTPC modulation that don't have MiniAudio equivalents, those features will degrade. Worth a quick survey of the `.atl_audio_controls` complexity before committing to scope.

---

## What to research before starting

**Read the engine source for MiniAudio:**

```
$HOME/PROJECTS/o3de/Gems/MiniAudio/
```

Specifically:
- `Code/Include/MiniAudio/MiniAudioPlaybackBus.h` -- the API surface
- `Code/Include/MiniAudio/MiniAudioListenerBus.h` -- listener API
- `Code/Include/MiniAudio/MiniAudioBus.h` -- system-level bus
- `Code/Source/MiniAudioSystemComponent.cpp` -- how the system component initializes
- `README.md` -- gem overview

**Find existing MiniAudio usage in other O3DE samples / projects:**

```bash
# Inside the engine source
grep -rln "MiniAudioPlaybackComponent\|MiniAudioListenerComponent\|MiniAudioPlaybackRequest" \
  ~/PROJECTS/o3de/AutomatedTesting ~/PROJECTS/o3de/Templates 2>/dev/null
```

`AutomatedTesting` and the project templates may have working MiniAudio examples to copy from.

**The o3de docs on MiniAudio:**

https://www.o3de.org/docs/user-guide/gems/reference/audio/miniaudio/

**Discord thread context** (Open 3D Foundation Discord, `gems-and-features` channel, "Demo Games Glow Up - especially Newspaper and MPS Demo game" thread, dated 2026-05-22): contains engine-maintainer guidance on the MiniAudio direction plus community discussion of MPS history and prior community-private fixes.

---

## What to research as you go

- **The `Audio::AudioTriggerNotificationBus` callback in `BackgroundMusicComponent`**: the engine's MiniAudio gem probably has an equivalent "sound finished playing" notification. Find it (look for `OnSoundFinished` / `OnPlaybackFinished` or similar in the MiniAudio bus headers) and port `BackgroundMusicComponent`'s notification handler to it.

- **`SoundAsset` vs `SoundAssetRef`**: the MiniAudio gem has both types. The `Ref` version is for ScriptCanvas-friendly binding; the asset version is for C++ direct references. Pick the right one per use site.

- **Positional / directional audio**: MiniAudio supports cone-attenuation, fixed-direction, listener-relative attenuation. If MPS's `.atl_audio_controls` use any of these features, port the configuration.

- **Network replication of audio events**: MPS is a multiplayer game. Check whether the audio trigger calls in `GameplayEffectsComponent` are network-replicated. If so, the MiniAudio equivalents need to fit into the same replication path. Look at how `GameplayEffectsComponent` invokes the audio trigger and whether the call site is client-side, server-side, or replicated.

- **ScriptCanvas node availability**: confirm that the MiniAudio Gem exposes ScriptCanvas nodes for `Play` / `Stop` / `SetSoundAssetRef`. If it doesn't expose what you need, that's a separate engine-side gap to surface (or to add as part of this PR).

---

## Validation approach

After each migration step, rebuild and run the existing Tier 9 / Tier 11 tests from https://github.com/nickschuetz/o3de-rpm:

```bash
cd $HOME/PROJECTS/o3de-rpm
make test-multiplayer-sample   # full build + bake + launcher smoke; ~3-10 min warm
make test-tier11-multiplayer   # post-load liveness; should still pass
```

For audio-specific validation:

```bash
make play-mps-host
make play-mps-client
# In the client window, play for ~5 minutes and listen for sounds.
# Open a terminal and tail $HOME/PROJECTS/o3de-multiplayersample/user/log/Game.log
# Confirm no "Failed to get a new instance of an AudioObject" entries.
```

If the existing tests still pass AND the audio errors are gone AND the game has actual audible sound, the migration is succeeding.

---

## Submission approach

PR target: `o3de/o3de-multiplayersample:development` from `nickschuetz/o3de-multiplayersample:miniaudio-migration`.

PR body should:
- Reference the Discord thread that motivated this work (cited above)
- List the 16 files affected
- Explain the architectural shift (string-triggers → asset-references)
- Confirm validation against Tier 9 / Tier 11 tests
- Include before/after observations on the audio-error log floods

Note: every commit on this branch needs DCO sign-off (`git commit -s`). The repo enforces it. See https://developercertificate.org for what DCO means.

---

## Hard guardrails

1. **Don't break the existing playable state.** MPS currently boots on Linux, loads `startmenu`, connects client to server, runs NewStarbase gameplay. The migration must preserve that.
2. **Don't touch unrelated code.** [PR #502](https://github.com/o3de/o3de-multiplayersample/pull/502)'s recent cmake revert is the only other in-flight change. Stay scoped to audio.
3. **DCO sign-off on every commit** (`git commit -s` for each).
4. **ASCII grammar conventions**: no em-dashes (`—`), no smart-unicode punctuation, no `--` as a stylistic em-dash substitute. Use commas, periods, parens, semicolons, colons. `--` is fine only for literal CLI flags.
5. **Keep the maintainer's fork's branches clean.** If experiments don't pan out, force-push only the experimental branch, never `development`.

---

## Cross-references

- The packaging-side test infrastructure that exposes this issue: https://github.com/nickschuetz/o3de-rpm (`tests/multiplayersample-build-test.sh` for Tier 9, `tests/post-load-liveness-test.sh` for Tier 11, `Makefile` `play-mps-*` targets for interactive validation).
- The packaging-side follow-up record: `FOLLOW_UPS.md` in the same repo, "MPS audio migration research" entry.
- Recent related upstream work: [o3de/o3de-multiplayersample-assets#177](https://github.com/o3de/o3de-multiplayersample-assets/pull/177) (asset case-sensitivity fix, merged); [o3de/o3de-multiplayersample#499](https://github.com/o3de/o3de-multiplayersample/pull/499) (PopcornFX/WWISE removal, merged); [o3de/o3de-multiplayersample#502](https://github.com/o3de/o3de-multiplayersample/pull/502) (cmake gem-double-load fix, merged).
- The MiniAudio Gem itself: `o3de/o3de`'s `Gems/MiniAudio/` directory; documentation at https://www.o3de.org/docs/user-guide/gems/reference/audio/miniaudio/.

---

End of brief.
