# Duplex Audio Home Assistant Voice Satellite With Wake Chime

Custom ESPHome configuration for the ESP32-S3-Box-3, forked from
[esphome/wake-word-voice-assistants](https://github.com/esphome/wake-word-voice-assistants)
(`esp32-s3-box-3/esp32-s3-box-3.yaml`).

**IMPORTANT** - In "Streaming" mode (wake word detection on HA with openWakeWord) we can't
pause the VAD. So the longer your chime is, the less time you have to start speaking afterwards.
With micro_wake_word (On-Device) you can make the chime as long as you want since we don't kick off the
assistant until after chime is done.

## Changes from upstream

- **Duplex I2S audio from esphome-intercom** swap `i2s_audio` platform for `i2s_audio_duplex` component from `esphome-intercom`. Allows simultaneous microphone capture and speaker output on a shared I2S bus without hack-ey starting and stopping of the mic or speaker pipelines. Esp for "streaming" mode where killing and restarting the voice_assistant causes all sorts of weird bugs.

- **Mixer / resampler speaker stack** Swap `i2s_audio` speaker with hardware speaker via an `audio_mixer` on two channels (`media_speaker_mix`, `tts_speaker_mix`), each fronted by a `resampler`. Media and TTS/announcement audio are mixed independently.
  - **NOTE** This is probably not necessary if you encode local files correctly
    as HA can transcode media during playback for streamed audio. If you need the memory, can probably remove the resamplers. I was just too lazy to keep testing once this worked.
- **Wake word chime** -- A confirmation chime plays on-device immediately after wake word detection in both modes (on-device mWW and streaming). The mic is muted during chime playback to prevent feedback, then unmuted before the voice pipeline proceeds.
- **PSRAM optimizations** -- Added `CONFIG_SPIRAM_FETCH_INSTRUCTIONS` and
  `CONFIG_SPIRAM_RODATA` sdkconfig options; mixer and resampler tasks use PSRAM
  stacks.
  - **NOTE** This is also probably avoidable if desired, I was just too lazy to do it properly.

## Usage Notes
- Either checkout `esphome-intercom` repo to `../esphome-intercom` or swap the  `external_component` config for the `github://n-IA-hane/esphome-intercom` version.
- `wake_chime_sound` is just a crappy placeholder - replace with something fun.
- `announcement: false` flag on chime playback is set to avoid triggering `on_announcement`.
- `chime_in_progress` global prevents blocks `media_player.on_idle` callback from restarting the wake word engine __I think you could prob remove this logic but again: lazy__.


---

**NOTE - AI Warning** The rest of this doc is AI generated (but still reviewed) because I continue to be lazy but still want some docs so that future-me can debug this in two years when it breaks after the great AI pricing rug-pull.

**NOTE** These diagrams don't show the "question/response" stages where voice_assistant loops back to listening when VA asks a question - but I did test it in both flows and it works as expected.

---

## Detailed flow

### Display phases

| ID | Name           | Display page         |
|----|----------------|----------------------|
| 1  | Idle           | `idle_page`          |
| 2  | Listening      | `listening_page`     |
| 3  | Thinking       | `thinking_page`      |
| 4  | Replying       | `replying_page`      |
| 10 | Not ready      | `no_ha_page`         |
| 11 | Error          | `error_page`         |
| 12 | Muted          | `muted_page`         |
| 20 | Timer finished | `timer_finished_page`|

### Mode A -- micro_wake_word ("On device") with chime

```mermaid
---
config:
  flowchart:
    defaultRenderer: "elk"
---
flowchart TD
    BOOT([Boot / HA client connected]) --> START_WW["start_wake_word:\nva.set_use_wake_word(false)\nmicro_wake_word.start()"]
    START_WW --> IDLE

    IDLE["IDLE (phase 1)\nmWW listening\nVA not running"]
    IDLE -->|mww.on_wake_word_detected| CHIME

    CHIME["play_chime_and_start_va\n1. chime_in_progress = true\n2. mute mic\n3. play chime (announcement: false)\n4. wait for playback to finish\n5. restore mic mute state\n6. chime_in_progress = false\n7. voice_assistant.start(wake_word)"]
    CHIME -->|va.on_listening| LISTENING

    LISTENING["LISTENING (phase 2)\nUser speaks..."]
    LISTENING -->|va.on_stt_vad_end| THINKING

    THINKING["THINKING (phase 3)\non_stt_end publishes text_request"]
    THINKING -->|va.on_tts_start| REPLYING

    REPLYING["REPLYING (phase 4)\nTTS plays via announcement pipeline"]
    REPLYING -->|va.on_end| WINDDOWN

    WINDDOWN["WIND-DOWN\n1. wait up to 0.5s for announcement to start\n2. wait until announcement + tts_speaker finish\n3. va.set_use_wake_word(false)\n4. micro_wake_word.start()\n5. phase = IDLE or MUTED"]
    WINDDOWN --> IDLE

    LISTENING -->|va.on_error| ERROR
    THINKING -->|va.on_error| ERROR
    REPLYING -->|va.on_error| ERROR
    ERROR["ERROR (phase 11)\n1s delay"] --> IDLE
```

**Key points:**
- VA is *not* running while idle. mWW owns the mic for wake word detection.
- `use_wake_word` stays `false` -- mWW detects the wake word externally and
  explicitly calls `voice_assistant.start`.
- The chime plays with `announcement: false` so it goes through the media
  pipeline. `chime_in_progress` prevents `on_idle` from prematurely restarting
  the wake word engine when the chime finishes.

### Mode B -- Streaming ("In Home Assistant") with chime

```mermaid
---
config:
  flowchart:
    defaultRenderer: "elk"
---
flowchart TD
    BOOT([Boot / HA client connected]) --> START_WW["start_wake_word:\nva.set_use_wake_word(true)\nvoice_assistant.start_continuous()"]
    START_WW --> IDLE

    IDLE["IDLE (phase 1)\nVA running continuously\nStreaming audio to HA\nHA detects wake word"]
    IDLE -->|"va.on_wake_word_detected\n(from HA server)"| CHIME
    IDLE -->|"va.on_wake_word_detected\n(from HA server)"| LISTENING

    CHIME["play_chime_streaming\n1. chime_in_progress = true\n2. mute mic\n   (HA receives silence ~1s)\n3. play chime (announcement: false)\n4. wait for playback to finish\n5. restore mic mute state\n6. chime_in_progress = false\n(NO voice_assistant.start — VA already running)"]

    LISTENING["LISTENING (phase 2)\nUser speaks..."]
    LISTENING -->|va.on_stt_vad_end| THINKING

    THINKING["THINKING (phase 3)\non_stt_end publishes text_request"]
    THINKING -->|va.on_tts_start| REPLYING

    REPLYING["REPLYING (phase 4)\nTTS plays via announcement pipeline"]
    REPLYING -->|va.on_end| WINDDOWN

    WINDDOWN["WIND-DOWN\n1. wait up to 0.5s for announcement to start\n2. wait until announcement + tts_speaker finish\n3. phase = IDLE or MUTED\n(streaming wake word auto-restarts — no explicit restart)"]
    WINDDOWN --> IDLE

    LISTENING -->|va.on_error| ERROR
    THINKING -->|va.on_error| ERROR
    REPLYING -->|va.on_error| ERROR
    ERROR["ERROR (phase 11)\n1s delay"] --> IDLE
```

**Key points:**
- VA runs continuously in idle. `use_wake_word` is `true` so HA performs
  server-side wake word detection on the audio stream.
- The chime script does *not* call `voice_assistant.start` because the VA
  session is already active. HA simply transitions from wake-word-detected into
  the STT phase on its own.
- The ~1s of silence HA receives during chime playback falls within the normal
  post-wake-word pause window.
- At `on_end`, streaming wake word detection auto-restarts -- no explicit
  `micro_wake_word.start` or `voice_assistant.start_continuous` is needed.

### Mode comparison

| Aspect                  | Mode A (mWW / On device)                  | Mode B (Streaming / In HA)                     |
|-------------------------|-------------------------------------------|-------------------------------------------------|
| Wake detection          | `micro_wake_word` on ESP32                | HA server-side via continuous audio stream       |
| VA state while idle     | Not running; mWW owns mic                 | Running continuously (`start_continuous`)        |
| Chime trigger           | `mww.on_wake_word_detected`               | `va.on_wake_word_detected`                       |
| Chime script            | `play_chime_and_start_va`                 | `play_chime_streaming`                           |
| After chime             | Calls `voice_assistant.start(wake_word)`  | Nothing -- VA already running                    |
| After `on_end`          | Restarts mWW explicitly                   | Streaming auto-restarts                          |
| `use_wake_word` flag    | Always `false`                            | Always `true`                                    |

### media_player callbacks

These fire for TTS announcements *and* non-VA media playback in both modes:

- **`on_announcement`** -- Stops the active wake word engine (mWW or VA
  continuous) if the mic is capturing, preventing speaker output from feeding
  back into the mic. For streaming mode, waits until VA stops before proceeding.
  If VA is not running (user-initiated media, not a voice pipeline), shows the
  muted display.
- **`on_idle`** -- When media finishes and VA is not running *and*
  `chime_in_progress` is false, restarts the wake word engine and returns to
  idle. The `chime_in_progress` guard is critical: without it, the chime
  finishing would trigger a premature wake word restart before VA starts (Mode A)
  or while the pipeline is mid-flight (Mode B).

### Speaker pipeline topology

```mermaid
---
config:
  flowchart:
    defaultRenderer: "elk"
---
flowchart BT
    MP["media_pipeline\n(media playback)"] --> MS
    AP["announcement_pipeline\n(TTS responses, timer alarm)"] --> TS
    MS["media_speaker\n(resampler)\nprob. not needed"] --> MSM["media_speaker_mix"]
    TS["tts_speaker\n(resampler)"] --> TSM["tts_speaker_mix"]
    MSM --> MIX["audio_mixer"]
    TSM --> MIX
    MIX --> HW["hw_speaker\n(i2s_audio_duplex)"]
```

The mixer combines both sources into the single hardware I2S output. Media and
announcements can play independently without blocking each other (though in
practice the chime sequence serializes them).
