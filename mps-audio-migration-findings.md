# MultiplayerSample audio migration: findings log

Companion document to `mps-audio-migration-brief.md`. The brief is the original hand-off (what was believed at the start). This document captures what was discovered when those beliefs were verified against the actual code in `$HOME/PROJECTS/o3de/Gems/MiniAudio/`, `$HOME/PROJECTS/o3de-multiplayersample/`, and `$HOME/PROJECTS/o3de-multiplayersample-assets/`.

Update this as new findings emerge. The brief stays as the original baseline.

---

## Summary of revisions to the brief

| Brief's claim | Verified status | Where it lives |
|---|---|---|
| MiniAudio public API: `Play/Stop/SetSoundAsset/SetSoundAssetRef/cones/volume` | Accurate. Methods exist as listed. | [API surface verification](#1-miniaudio-api-surface-verified) |
| `MiniAudioListenerComponent` is a 1:1 swap for `AudioListenerComponent` | Accurate. | [API surface verification](#1-miniaudio-api-surface-verified) |
| `AudioProxyComponent` has no MiniAudio equivalent and should be removed | Accurate. | [API surface verification](#1-miniaudio-api-surface-verified) |
| `Audio::AudioTriggerNotificationBus` has a MiniAudio equivalent to find | **Wrong.** The gem exposes no notification bus at all. | [Notification gap](#2-notification-gap-no-on-finish-in-the-miniaudio-gem) |
| Estimated effort: 1 to 4 days focused work | **Understated.** Code is roughly that, but the asset side is missing entirely. | [Asset gap](#5-asset-gap-mps-has-never-shipped-game-audio) |
| "Audit the .atl_audio_controls files; find the underlying sound file in the assets repo" | **Not workable.** The assets repo has zero raw audio files, on any branch, in any commit. | [Asset gap](#5-asset-gap-mps-has-never-shipped-game-audio) |

---

## 1. MiniAudio API surface (verified)

Read on 2026-05-22 against `$HOME/PROJECTS/o3de/Gems/MiniAudio/Code/Include/MiniAudio/`.

### Playback bus (`MiniAudioPlaybackBus.h`)
Methods present:
- `Play()`, `Stop()`, `Pause()`
- `SetLooping(bool)` / `IsLooping()`
- `SetSoundAsset(AZ::Data::Asset<SoundAsset>)` / `GetSoundAsset()`
- `SetSoundAssetRef(const SoundAssetRef&)` / `GetSoundAssetRef()` (scripting-friendly wrapper)
- `SetVolumePercentage` / `SetVolumeDecibels` (+ getters)
- Cone attenuation: inner/outer angle (radians + degrees), outer volume (percentage + dB), fixed direction, directional attenuation factor, direction vector

### Listener bus (`MiniAudioListenerBus.h`)
- `SetFollowEntity(EntityId)` (binds listener to an entity's transform)
- `SetPosition(Vector3)` (override position)
- `GetChannelCount()`
- Global volume (percentage + dB)
- Cone attenuation (same shape as playback bus)

Listener migration is the easiest piece of the whole job.

### System bus (`MiniAudioBus.h`)
- `GetSoundEngine() -> ma_engine*` (escape hatch to the raw library)
- `SetGlobalVolume(float)` / `GetGlobalVolume()`
- `SetGlobalVolumeInDecibels(float)`
- `GetChannelCount()`

### Assets
- `SoundAsset` (C++ type, `AZ::Data::Asset<SoundAsset>`) — file extension `.miniaudio`, asset group `Sound`
- `SoundAssetRef` (ScriptCanvas/Lua wrapper around `SoundAsset`)

Both exist. Use `SoundAsset` from C++; use `SoundAssetRef` for any property that gets serialized into a prefab or exposed to ScriptCanvas.

---

## 2. Notification gap: no on-finish in the MiniAudio gem

The brief assumed MiniAudio had an equivalent of `Audio::AudioTriggerNotificationBus::ReportTriggerFinished`. It does not.

Searched `$HOME/PROJECTS/o3de/Gems/MiniAudio/Code/{Include,Source}/` for: `Notif`, `Callback`, `Finish`, `OnEnd`, `OnPlayback`, `OnSound`, `IsPlaying`, `at_end`. **Zero hits.** The `MiniAudioPlaybackComponentController` (`Code/Source/Clients/MiniAudioPlaybackComponentController.h`) exposes only the Play/Stop/setters/getters from the public bus. There is no way to ask "is this sound done?" and no event fired when one finishes.

### But the underlying miniaudio library supports it

`miniaudio.h` 0.11.22 (the bundled version per `Gems/MiniAudio/3rdParty/Findminiaudio.cmake`, verified against the engine install at `/opt/O3DE/26.05.0/include/miniaudio/miniaudio.h`) provides:

- `ma_bool32 ma_sound_is_playing(const ma_sound*)`
- `ma_bool32 ma_sound_at_end(const ma_sound*)`
- `ma_result ma_sound_set_end_callback(ma_sound*, ma_sound_end_proc callback, void* pUserData)`
- `ma_result ma_sound_get_cursor_in_pcm_frames(ma_sound*, ma_uint64*)`
- `ma_result ma_sound_get_length_in_pcm_frames(ma_sound*, ma_uint64*)`

Header comment on `endCallback`: "Fired when the sound reaches the end. **Will be fired from the audio thread.** Do not restart, uninitialize or otherwise change the state of the sound from here. Instead fire an event or set a variable to indicate to a different thread."

So the underlying capability is there; the gem simply doesn't surface it. Any upstream notification-bus PR would need to marshal from the audio thread to the game thread (queued ebus, `AZ::TickBus::QueueFunction`, or similar).

---

## 3. Both C++ components depend on the missing notification

### `BackgroundMusicComponent`
Reference: `Gem/Code/Source/Components/BackgroundMusicComponent.{h,cpp}`.

The component is a **playlist shuffler driven by track-finish events**:
- Inherits `Audio::AudioTriggerNotificationBus::Handler`
- `Activate()` calls `ReportTriggerFinished(INVALID_AUDIO_CONTROL_ID)` to bootstrap playback of track 0
- `ReportTriggerFinished()` (cpp:85) increments `m_trackIndex`, picks the next track, calls `ExecuteTrigger` on it via `AudioProxyComponentRequestBus`

Without a finish signal, the playlist cannot advance. This component cannot be ported as-is.

### `GameplayEffectsComponent`
Reference: `Gem/Code/Source/Components/Multiplayer/GameplayEffectsComponent.{h,cpp}` + `Gem/Code/Source/AutoGen/GameplayEffectsComponent.AutoComponent.xml`.

The component spawns one-shot audio prefabs and uses the finish callback as the **prefab garbage collector**:

1. RPC arrives (`HandleRPC_OnEffect` / `HandleRPC_OnPositionalEffect`) carrying a `SoundEffect` enum value
2. `SpawnEffect()` (cpp:131):
   - Looks up the string trigger name for the enum value
   - Spawns a default prefab via `NetworkPrefabSpawnerComponent::SpawnDefaultPrefab` (the default is `Sound_Effect.prefab`)
   - In the on-activate callback: subscribes to `AudioTriggerNotificationBus` on the spawned entity, stores its spawn ticket in `m_spawnedEffects`, calls `ExecuteTrigger(triggerName)`
3. `ReportTriggerFinished()` (cpp:105): erases the ticket from `m_spawnedEffects`, which despawns the prefab

Without a finish signal, every sound effect leaks a prefab entity plus its spawn ticket. At MPS's RPC volume this is a fast leak.

---

## 4. Architectural specifics for the prefab side

### Same prefab, many sounds

The spawned `Sound_Effect.prefab` template (`Prefabs/Sound_Effect.prefab`) has:
- `TransformComponent`
- `EditorAudioTriggerComponent`
- `AudioProxyComponent`
- A `SoundEffectComponent` (MPS-defined, presumably a small glue component)

One prefab handles all 30 distinct sound effects because the trigger name is passed in at spawn time and resolved through Wwise/ATL externally.

In MiniAudio's one-component-per-sound model, the cleanest port keeps the same shape:
- Replace `EditorAudioTriggerComponent` + `AudioProxyComponent` with a single `MiniAudioPlaybackComponent`
- At spawn time, call `SetSoundAsset(asset)` based on the `SoundEffect` enum, then `Play()`
- Despawn when the finish notification fires

### AutoComponent.xml regeneration required

`Gem/Code/Source/AutoGen/GameplayEffectsComponent.AutoComponent.xml` currently declares 28 properties like:

```xml
<ArchetypeProperty Type="AZStd::string" Name="PlayerFootSteps" ExposeToEditor="true" Description="Audio trigger name" />
```

Each one is a Wwise trigger name. These getter-generated strings are consumed at cpp:41-73 and stored in `m_soundTriggerNames`.

To migrate: change `Type="AZStd::string"` to `Type="AZ::Data::Asset<SoundAsset>"` (for C++ use) or whatever shape works for the AutoComponent codegen with the MiniAudio asset type. The cpp side stops storing strings and instead stores asset references, which get handed to `MiniAudioPlaybackComponent::SetSoundAsset` at spawn time.

Unknown: whether the AutoComponent codegen supports `AZ::Data::Asset<T>` properties. Worth confirming by looking at other AutoComponents in MPS or the engine that use asset references (e.g., spawnable references).

---

## 5. Asset gap: MPS has never shipped game audio

This is the largest scope finding and was not addressed in the brief.

### What was checked

`$HOME/PROJECTS/o3de-multiplayersample-assets/`:
- Current `development` branch: zero `.wav`, `.ogg`, `.mp3`, `.flac`, `.atl_audio_controls`, `.miniaudio` files.
- Git history across all branches (`development`, `upstream/popcornfx-wwise`, `upstream/hexanyaudio`, `upstream/UpdatePopcornFXRepo`): same. Searched both committed files and LFS-tracked patterns (`.gitattributes` declares LFS filters for `.bnk`, `.wem`, but no `.wav`/`.ogg`/`.mp3` patterns).

### What the upstream branches contain

- **`upstream/popcornfx-wwise`** (the pre-strip branch the WWISE-strip commit `db0849f` refers to, "Note that the popcornFX and Wwise implementations are still present in the popcornfx-wwise branch"): zero raw audio. Just art assets + PopcornFX particle libraries.
- **`upstream/hexanyaudio`** (presumably the work from upstream [PR #175](https://github.com/o3de/o3de-multiplayersample-assets/pull/175) "switch-to-openparticlesystem-miniaudio"): the only branch with any audio paths at all. Contents under `Project/Sounds/`:
  - Wwise sound banks (`.bnk`): templated, mostly empty/placeholder (names like `SB_SX_CHAR_CharacterName.bnk`, `SB_MX_LVL_Level01.bnk`).
  - Source `.wav` files under `Project/Sounds/wwise_project/Originals/SFX/Placeholder/`: ten generic templates (`temp_mono_oneshot_short.wav`, `temp_mono_ambience.wav`, etc.) plus ten `sx_robotvoice_number_NN.wav` placeholders. These are stand-ins, not real game audio.

In short: there is no source-of-truth for real MPS sound effects anywhere in git history. The Wwise project structure was scaffolded but never populated with final sounds.

### What the ATL configuration declares

`libs/gameaudio/wwise/multiplayersample_controls.xml` (read-only side, lives in the main MPS repo, not the assets repo): 60 `ATLTrigger` entries (30 unique sounds, each with a `Play_`/`play_` and `Stop_`/`stop_` variant). Naming follows Wwise conventions:

- `play_sx_player_footstep`, `play_sx_player_pain`, `play_sx_player_knockdown`, `play_sx_player_armor_breaking`, `play_sx_player_armor_mend`, `play_sx_player_exertion`
- `play_sx_wpn_laserpistol_fire`, `play_sx_wpn_laserpistol_impact`, `play_sx_wpn_bubblegun_*` (fire, projectile, explosion, cooldown, ready)
- `play_sx_int_gem_pickup`, `play_sx_int_jumppad_launch`, `play_sx_int_teleporter_*`, `play_sx_int_energyballtrap_*`, `play_sx_int_defenseturret_*`, `play_sx_int_speedpowerup_activate`
- `play_sx_ui_game_countdown`, `play_sx_ui_game_end`, `play_sx_ui_game_fanfare_victory`, `play_sx_ui_game_fanfare_defeat`, `play_sx_ui_game_round_start`, `play_sx_ui_game_round_end`
- Music: `rocket`, `future_gladiator`, `beauty_flow` (those track names match Kevin MacLeod CC-BY tracks at incompetech.com, which would be permissively-licensed if sourced from there)

The XML also declares `WwiseRtpc` (parameters), `WwiseState` (game states), `WwiseSwitch` (switches), `WwiseAuxBus` (mixing/routing). These Wwise-specific features have no MiniAudio equivalent. Most are probably never actually exercised by MPS gameplay code (the XML was templated from a Wwise project); needs a quick audit to confirm.

### Implication

The migration also requires sourcing or creating roughly 30 sound effects + 3 music tracks as `.wav` or `.ogg` files, under licenses compatible with O3DE's Apache-2.0/MIT release. This is a separate workstream from the code migration. It is not a 1-to-4-day effort.

Options:
1. Source from CC0/CC-BY libraries (freesound.org, OpenGameArt, Kenney's audio packs, Sonniss GDC bundle).
2. Generate via AI tools (license terms vary; needs verification per tool).
3. Commission from the community / a contributor with audio production capacity.
4. Ship the code migration with `nullptr`/empty `SoundAssetRef`s and accept silent gameplay until a follow-up asset PR lands.

Option 4 is the cleanest sequencing — code migration becomes independently mergeable.

---

## 6. ScriptCanvas migration shape

The brief lists 5 ScriptCanvas files. Confirmed:
```
scriptcanvas/JumpPad.scriptcanvas
scriptcanvas/PredictiveJumpPad.scriptcanvas
scriptcanvas/AudioTester.scriptcanvas
scriptcanvas/ShieldGeneratorRoundEffects.scriptcanvas
scriptcanvas/EnableAudioListener.scriptcanvas
```

`AudioTester.scriptcanvas` only references `AudioTriggerComponentRequestBus` with `Play`/`Stop` methods. No `ExecuteTrigger`/`KillTrigger` complexity. Brief is correct that this is the simplest starting case.

These files are JSON-serialized node graphs. Editing them by hand is possible but error-prone for non-trivial graphs (node IDs, slot references, connection lists). Best done in the Editor's ScriptCanvas tool.

---

## 7. Recommended PR split

Given the findings, a clean three-PR split across two repos:

| # | Repo | Scope | Depends on |
|---|---|---|---|
| 1 | `o3de/o3de` | Add `MiniAudioPlaybackNotificationBus::OnPlaybackFinished`. Wraps `ma_sound_set_end_callback`, marshals to TickBus. ~30-line patch plus reflection + tests. | nothing |
| 2 | `o3de/o3de-multiplayersample` | The 16-file migration in the brief. Components use the new notification bus from (1). Prefabs reshaped. ScriptCanvas re-authored. AutoComponent.xml regenerated with `SoundAsset` properties. | (1) merged (or vendored temporarily for development) |
| 3 | `o3de/o3de-multiplayersample-assets` | New CC0/CC-BY audio assets. Roughly 30 SFX + 3 music tracks as `.wav` or `.ogg`. | nothing technical |

(2) and (3) are independent. (2) can ship with empty `SoundAssetRef`s and play silently; (3) wires up the assets in a follow-up.

---

## 8. Open questions

- Does the AutoComponent codegen support `AZ::Data::Asset<T>` properties? Check by greping the engine for other `<ArchetypeProperty Type="AZ::Data::Asset` examples.
- Does `MiniAudioPlaybackComponent` work cleanly inside a spawnable that gets despawned mid-playback? The on-finish callback fires from the audio thread, but if the component is destroyed first the callback target is gone. Need to verify the gem's lifecycle handles this safely, or design (1) to handle it.
- Does MPS's audio actually use Wwise RTPCs/Switches/States/AuxBus in code, or are they just XML leftovers? Grep MPS C++ + ScriptCanvas for any Audio::IAudioSystem calls that pass parameter/state names.
- Does `BackgroundMusicComponent` actually have a playlist populated anywhere, or is the playlist authored in a prefab that we haven't read yet?
- Are the 3 music track names (`rocket`, `future_gladiator`, `beauty_flow`) actually Kevin MacLeod tracks? If so, those are CC-BY 3.0/4.0 from incompetech.com and can ship cleanly with attribution.

---

## 9. Reference: source-of-truth file paths

For the next contributor walking in:

**Engine (read-only research):**
- `$HOME/PROJECTS/o3de/Gems/MiniAudio/Code/Include/MiniAudio/MiniAudioPlaybackBus.h`
- `$HOME/PROJECTS/o3de/Gems/MiniAudio/Code/Include/MiniAudio/MiniAudioListenerBus.h`
- `$HOME/PROJECTS/o3de/Gems/MiniAudio/Code/Include/MiniAudio/MiniAudioBus.h`
- `$HOME/PROJECTS/o3de/Gems/MiniAudio/Code/Include/MiniAudio/SoundAsset.h`
- `$HOME/PROJECTS/o3de/Gems/MiniAudio/Code/Include/MiniAudio/SoundAssetRef.h`
- `$HOME/PROJECTS/o3de/Gems/MiniAudio/Code/Source/Clients/MiniAudioPlaybackComponentController.{h,cpp}`
- `$HOME/PROJECTS/o3de/Gems/MiniAudio/3rdParty/Findminiaudio.cmake` (pins miniaudio to 0.11.22, hash `350784a9`)
- `/opt/O3DE/26.05.0/include/miniaudio/miniaudio.h` (the bundled library, for confirming `ma_sound_*` APIs)

**MPS code (write):**
- `$HOME/PROJECTS/o3de-multiplayersample/Gem/Code/Source/Components/BackgroundMusicComponent.{h,cpp}`
- `$HOME/PROJECTS/o3de-multiplayersample/Gem/Code/Source/Components/Multiplayer/GameplayEffectsComponent.{h,cpp}`
- `$HOME/PROJECTS/o3de-multiplayersample/Gem/Code/Source/AutoGen/GameplayEffectsComponent.AutoComponent.xml`
- `$HOME/PROJECTS/o3de-multiplayersample/Gem/Code/Source/Components/PerfTest/NetworkPrefabSpawnerComponent.{h,cpp}` (the `SpawnDefaultPrefab` caller)

**MPS prefabs (write):**
- `$HOME/PROJECTS/o3de-multiplayersample/Prefabs/{AudioTester,Sound_Effect,Energy_Ball,GamePlay_Effects,Player,BubbleBall}.prefab`
- `$HOME/PROJECTS/o3de-multiplayersample/Levels/NewStarbase/NewStarbase.prefab`

**MPS ScriptCanvas (re-author in Editor):**
- `$HOME/PROJECTS/o3de-multiplayersample/scriptcanvas/{JumpPad,PredictiveJumpPad,AudioTester,ShieldGeneratorRoundEffects,EnableAudioListener}.scriptcanvas`

**MPS audio configuration (delete or transform):**
- `$HOME/PROJECTS/o3de-multiplayersample/libs/gameaudio/wwise/multiplayersample_controls.xml` (60 ATL triggers)
- `$HOME/PROJECTS/o3de-multiplayersample/libs/gameaudio/wwise/default_controls.xml`

**Assets repo (needs to gain audio):**
- `$HOME/PROJECTS/o3de-multiplayersample-assets/Gems/` (no `audio_mps` or similar gem currently exists; would need to be created)

---

End of findings log. Append future research below this line.
