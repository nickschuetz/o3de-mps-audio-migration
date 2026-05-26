# MultiplayerSample audio migration

Coordination repo for migrating MultiplayerSample's audio code from O3DE's legacy AzAudio (ATL/Wwise-shaped) APIs to the modern MiniAudio gem.

> **Status:** Research complete, no code changes yet. Three PRs planned across two repos (see [Planned work](#planned-work)). Looking for input from [O3DE SIG-Network](https://github.com/o3de/sig-network) on the architectural points called out below.

---

## TL;DR for SIG-Network

The MultiplayerSample audio bug on Linux is **a networking reliability failure dressed up as an audio bug.** No AzAudio backend implementation is enabled in MPS's gem set on Linux, so every `AudioObject` allocation fails. The error log floods, the main loop slows down, and the server eventually times out on packet processing. Both client and server abort under sustained play.

The fix is to migrate MPS to the MiniAudio gem, which already ships enabled in MPS's `project.json` but is never called from MPS code. This effort is the coordination home for that migration.

Two points where SIG-Network's input would help:

1. **`GameplayEffectsComponent` migration shape.** It currently fires per-effect RPCs that each spawn a one-shot audio prefab on every client. MiniAudio's one-component-per-sound asset model forces a structural reshape, including changes to the autogen properties. We have a proposal below; a sanity check on the RPC-plus-spawn pattern would be valuable.

2. **MiniAudio gem upstream gap.** The gem is missing an `OnPlaybackFinished` notification, which `GameplayEffectsComponent` relies on as a **prefab garbage collector**. Without it, every sound effect leaks a spawned prefab plus its `EntitySpawnTicket`. Worth your awareness since spawn-and-despawn lifecycle is your domain. We propose to land the notification bus as a small upstream PR against `o3de/o3de`.

---

## How to engage

Per SIG-Network's published process:

- **Monthly meeting:** third Thursday of each month (general meeting).
- **Weekly triage:** Thursdays.
- **Get a topic on the agenda:** open an issue on [github.com/o3de/sig-network](https://github.com/o3de/sig-network) labeled `mtg-agenda` for an upcoming meeting and link this README.
- **Lightweight pre-announce:** `#sig-network` channel in the Open 3D Foundation Discord (channel topic: *Communication Standards, Cloud networking, Multiplayer Client / Server / Peer networking*).
- **Formal proposals:** the [RFC process](https://github.com/o3de/sig-network/blob/main/rfcs/README.md).

For this migration: the agenda-issue route is probably the right fit, since the two open questions ([GameplayEffectsComponent architecture review](#gameplayeffectscomponent-the-networking-relevant-component) and the [upstream notification-bus PR shape](#blocker-1-the-notification-gap)) benefit from synchronous discussion.

---

## The bug, in detail

A community contributor running MultiplayerSample on Fedora 44 / Linux found:

1. The launcher loads `startmenu` and gameplay works (player movement, network sync, NewStarbase rendering, multiplayer round timing).
2. Sustained play produces hundreds of `[Error] (System) - Failed to get a new instance of an AudioObject from the implementation` log entries.
3. Eventually both client and server abort. The server times out on packet processing because audio errors slow the main loop.

Reproduced as `make play-mps-host` + `make play-mps-client` in https://github.com/nickschuetz/o3de-rpm.

The root cause is confirmed: MPS needs to migrate to MiniAudio. MiniAudio does not go through the `AudioSystem` / `AudioObject` / ATL pathway at all; it is independent of the legacy stack. That rules out tunable pool-size fixes; the backend isn't full, it's absent.

---

## Architectural shift

The migration is not search-and-replace. Each affected entity needs structural rework.

| AzAudio (legacy) | MiniAudio replacement | Nature |
|---|---|---|
| `AudioListenerComponent` | `MiniAudioListenerComponent` | Direct 1:1 swap |
| `AudioProxyComponent` | (none) | Remove entirely |
| `AudioTriggerComponent` (one component plays many string-named triggers) | `MiniAudioPlaybackComponent` (one component per sound asset) | **Semantic mismatch** |
| `Audio::TAudioControlID` (string-named trigger from external ATL/Wwise catalog) | `AZ::Data::Asset<SoundAsset>` / `SoundAssetRef` (direct `.wav`/`.ogg` reference) | Structural change |
| `Audio::AudioTriggerNotificationBus::ReportTriggerFinished` | (none — see [The notification gap](#blocker-1-the-notification-gap)) | Needs upstream work |

An entity that today has one `AudioTriggerComponent` playing five trigger names becomes an entity with five `MiniAudioPlaybackComponent` instances, or one component whose `SoundAssetRef` gets swapped at runtime. We're leaning toward the latter for `Sound_Effect.prefab` (see next section), the former for entities with overlapping sounds.

---

## GameplayEffectsComponent: the networking-relevant component

This is the audio path that intersects with networking, and where we'd most value team input.

**File**: `Gem/Code/Source/Components/Multiplayer/GameplayEffectsComponent.{h,cpp}` + `Gem/Code/Source/AutoGen/GameplayEffectsComponent.AutoComponent.xml`.

### Current flow

1. Authority side raises a `SoundEffect` enum value, calls `RPC_OnEffect(effect)` or `RPC_OnPositionalEffect(effect, position)` (declared in the AutoComponent XML; reliable RPCs, `InvokeFrom="Authority" HandleOn="Client"`).
2. On each client, `SpawnEffect()` (cpp:131):
   - Looks up the string trigger name corresponding to the enum value (28 string properties stored on the component, populated at activate time from `<ArchetypeProperty Type="AZStd::string">` declarations in the autogen XML).
   - Spawns a default prefab via `NetworkPrefabSpawnerComponent::SpawnDefaultPrefab` (resolves to `Prefabs/Sound_Effect.prefab`).
   - In the on-activate callback: subscribes to `AudioTriggerNotificationBus` on the spawned entity, stores the spawn ticket in `m_spawnedEffects`, calls `ExecuteTrigger(triggerName)`.
3. `ReportTriggerFinished()` (cpp:105) erases the ticket from `m_spawnedEffects`. That despawns the prefab.

### Proposed flow under MiniAudio

1. Authority side raises the same `SoundEffect` enum value, same RPCs. **No change to the network surface.**
2. On each client, `SpawnEffect()`:
   - Looks up the **`AZ::Data::Asset<SoundAsset>`** corresponding to the enum value (the 28 `<ArchetypeProperty Type="AZStd::string">` declarations become `Type="AZ::Data::Asset<SoundAsset>"`).
   - Spawns the same `Sound_Effect.prefab`, now containing a `MiniAudioPlaybackComponent` instead of `AudioProxyComponent + AudioTriggerComponent`.
   - On activate: calls `MiniAudioPlaybackRequestBus::Event(entityId, &MiniAudioPlaybackRequestBus::Events::SetSoundAsset, asset)`, then `Play()`, then subscribes to the on-finish notification (which doesn't exist yet, see below).
3. On finish: erases the ticket, prefab despawns.

### Questions for SIG-Network

- Is the per-effect-RPC, per-client-spawn pattern still the recommended shape in 2026, or has the team moved toward something like replicating sound events as component state changes?
- Does the AutoComponent codegen support `AZ::Data::Asset<T>` as an `ArchetypeProperty` `Type`? We haven't verified.
- The spawned prefab gets despawned mid-callback (audio thread fires "finished" -> we despawn the entity owning the component that owns the `ma_sound`). Is there a recommended lifecycle pattern for "self-despawning networked entities driven by an async signal"?

---

## Blocker 1: the notification gap

The MiniAudio gem in `o3de/o3de` exposes **no on-finish notification, no `IsPlaying` query, no callback surface at all** (verified by searching `Gems/MiniAudio/Code/{Include,Source}/` for `Notif`, `Callback`, `Finish`, `OnEnd`, `IsPlaying`). The `MiniAudioPlaybackComponentController` has only `Play`/`Stop`/`Pause`/setters/getters.

The underlying `miniaudio` library (version 0.11.22, pinned in `Gems/MiniAudio/3rdParty/Findminiaudio.cmake`) supports this fully:

- `ma_bool32 ma_sound_is_playing(const ma_sound*)`
- `ma_bool32 ma_sound_at_end(const ma_sound*)`
- `ma_result ma_sound_set_end_callback(ma_sound*, ma_sound_end_proc, void* pUserData)`

The end callback **fires from the audio thread** (per the upstream header comment: "Do not restart, uninitialize or otherwise change the state of the sound from here. Instead fire an event or set a variable to indicate to a different thread"). Any notification-bus PR needs to marshal from audio thread to game thread via a queued ebus or `AZ::TickBus::QueueFunction`.

Two components depend on this:
- **`BackgroundMusicComponent`** uses `ReportTriggerFinished` to advance to the next track in its playlist (BackgroundMusicComponent.cpp:85). Without a finish signal, playlist cannot advance.
- **`GameplayEffectsComponent`** uses it as the prefab garbage collector (GameplayEffectsComponent.cpp:105). Without a finish signal, every sound effect leaks an entity and a spawn ticket.

Proposed upstream PR: add `MiniAudio::MiniAudioPlaybackNotificationBus::OnPlaybackFinished` that wraps `ma_sound_set_end_callback` and marshals to TickBus. Approximately 30-line patch plus reflection and a small test.

---

## Blocker 2: missing audio assets

The `o3de-multiplayersample-assets` repo contains zero raw audio files (`.wav`, `.ogg`, `.mp3`, `.atl_audio_controls`, `.miniaudio`) on the `development` branch. Verified by searching:

- Current `development` branch: nothing.
- All git history across all branches (`development`, `popcornfx-wwise`, `hexanyaudio`, `UpdatePopcornFXRepo`): nothing committed, nothing tracked by LFS.
- `upstream/popcornfx-wwise` branch (the pre-strip branch referenced by the WWISE-strip commit `db0849f`): zero raw audio.
- `upstream/hexanyaudio` branch (apparent in-progress migration attempt from [PR #175](https://github.com/o3de/o3de-multiplayersample-assets/pull/175)): only Wwise placeholders under `Project/Sounds/wwise_project/Originals/SFX/Placeholder/` (`temp_mono_oneshot_short.wav` and similar) plus empty/templated `.bnk` files. Not real game audio.

MPS has never shipped final game audio. The `libs/gameaudio/wwise/multiplayersample_controls.xml` file in the MPS repo declares 60 ATL triggers (30 unique sounds, each with `Play_`/`Stop_` variants) plus Wwise RTPCs/Switches/States/AuxBus, but the underlying Wwise project was never populated.

So the migration also requires sourcing or creating roughly 30 SFX plus 3 music tracks as `.wav`/`.ogg`, under licenses compatible with O3DE's Apache-2.0/MIT release. Candidates: CC0/CC-BY libraries (freesound.org, OpenGameArt, Kenney audio packs, Sonniss GDC bundle), or community contributions. The three music track names in the ATL (`rocket`, `future_gladiator`, `beauty_flow`) match Kevin MacLeod CC-BY 4.0 tracks from incompetech.com, which would ship cleanly with attribution.

This is a separate workstream from the code migration and not on the critical path: the code PR can ship with empty `SoundAssetRef`s (silent gameplay) and the asset PR can follow.

---

## Planned work

| # | Repo | Scope | Depends on |
|---|---|---|---|
| 1 | `o3de/o3de` | Add `MiniAudioPlaybackNotificationBus::OnPlaybackFinished`. Wraps `ma_sound_set_end_callback`, marshals to TickBus. Self-contained, useful to any MiniAudio consumer. | nothing |
| 2 | `o3de/o3de-multiplayersample` | The 16-file migration: 2 C++ components, 5 ScriptCanvas graphs, 7 prefabs, AutoComponent XML regeneration, ATL config removal. | (1) merged or vendored |
| 3 | `o3de/o3de-multiplayersample-assets` | New CC0/CC-BY audio assets. Roughly 30 SFX plus 3 music tracks as `.wav` or `.ogg`. | nothing technical |

PR (2) can ship silent (empty `SoundAssetRef`s) and PR (3) can follow independently. That keeps (2) reviewable and unblocks the Linux launcher regardless of asset readiness.

All commits to MPS forks need DCO sign-off (`git commit -s`); repo enforces it.

---

## Open questions

Live questions we're tracking. SIG-Network perspectives welcome on any of them.

- Does the AutoComponent codegen support `AZ::Data::Asset<T>` as an `ArchetypeProperty` `Type`? Need to grep the engine for prior art.
- Does `MiniAudioPlaybackComponent` handle being destroyed mid-playback safely? The audio-thread end-callback could fire after entity destruction, into a stale `this` pointer. Needs to be handled in the upstream notification-bus PR.
- Does MPS actually use Wwise RTPCs / Switches / States / AuxBus in code, or are those just XML leftovers from a Wwise project template? Audit needed.
- Does `BackgroundMusicComponent` have a playlist populated anywhere, or is the playlist authored in a prefab we haven't read yet?
- Music track names (`rocket`, `future_gladiator`, `beauty_flow`) appear to be Kevin MacLeod CC-BY tracks. Confirm provenance and attribution path before sourcing.

---

## Source documents

These are the working documents and stay maintained alongside this README. The README is a digest; the source docs are where new research lands first.

- [`mps-audio-migration-brief.md`](mps-audio-migration-brief.md) — original hand-off, written before verification against engine source. Treat as historical baseline.
- [`mps-audio-migration-findings.md`](mps-audio-migration-findings.md) — ongoing research log. Verifies the brief, records what's been checked vs. what hasn't, captures discoveries as they happen.

---

## Repo layout

```
.
├── README.md                          # This file. Digest for the networking team.
├── mps-audio-migration-brief.md       # Original hand-off (read-only baseline).
└── mps-audio-migration-findings.md    # Ongoing research log (append as discoveries happen).
```

---

## Related links

**SIG-Network:**
- SIG-Network repo: https://github.com/o3de/sig-network
- Discord: `#sig-network` in the Open 3D Foundation Discord
- RFC process: https://github.com/o3de/sig-network/blob/main/rfcs/README.md

**This migration:**
- MPS code fork: https://github.com/nickschuetz/o3de-multiplayersample (PR target: `o3de/o3de-multiplayersample:development`)
- MPS assets fork: https://github.com/nickschuetz/o3de-multiplayersample-assets (PR target: `o3de/o3de-multiplayersample-assets:development`)
- Packaging-side test infrastructure that exposes the bug: https://github.com/nickschuetz/o3de-rpm

**O3DE audio:**
- O3DE engine source (MiniAudio gem): https://github.com/o3de/o3de/tree/development/Gems/MiniAudio
- MiniAudio Gem docs: https://www.o3de.org/docs/user-guide/gems/reference/audio/miniaudio/
- `miniaudio` upstream library (0.11.22): https://github.com/mackron/miniaudio
