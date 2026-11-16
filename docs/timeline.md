# Timeline and EditPlan

Montaj describes an edit at two levels:

- **EditPlan**: what the director writes. Shots are measured in **beats**, never seconds.
- **Timeline**: what the renderer executes. Everything is in **seconds** at a fixed canvas and frame rate.

The conform step turns an EditPlan plus the music's beat grid into a Timeline. Every video job returns its Timeline in `output.timeline`; you can edit it and send it back (`POST /v1/jobs/{id}/revisions` with `timeline`) or render your own (`POST /v1/renders`).

The models live in `packages/core/src/montaj_core/timeline/` (`schema.py`, `plan.py`, `validate.py`, `conform.py`). A JSON Schema of the Timeline ships with the SDK (`montaj.timeline_schema()`, file `packages/sdk/src/montaj/schemas/timeline.schema.json`) and is returned by the MCP tool `get_timeline_schema`.

## Timeline

All models are strict: unknown fields are rejected.

### Timeline (top level)

| Field | Type | Default | Rules |
|---|---|---|---|
| `version` | `"1"` | `"1"` | the only version |
| `seed` | integer | 0 | carried through for reproducibility |
| `output` | OutputSpec | required | |
| `background` | `#rrggbb` | `#000000` | shows behind `contain` fits and under transitions |
| `clips` | list of Clip | required | at least one; ids unique |
| `audio` | list of AudioClip | `[]` | |
| `texts` | list of TextClip | `[]` | |
| `licence` | LicenceRecord or null | null | the music's licence record |

### OutputSpec

| Field | Type | Default | Rules |
|---|---|---|---|
| `width`, `height` | integer | required | 16 to 7680, even |
| `fps` | integer | 30 | 1 to 120 |
| `video_codec` | `h264`, `h265` | `h264` | |
| `quality` | `draft`, `standard`, `high` | `standard` | encoder settings, see [Encoding](#encoding) |

### Clip (one entry of the video track)

| Field | Type | Default | Rules |
|---|---|---|---|
| `id` | string | required | 1 to 128 characters, unique |
| `asset_id` | string | required | 1 to 128 characters |
| `kind` | `image`, `video` | required | must match the asset |
| `duration` | number | required | > 0 seconds on the timeline |
| `source_in` | number | 0 | >= 0, seconds into a video source |
| `speed` | number | 1 | 0.1 to 10; a video uses `duration * speed` seconds of source |
| `fit` | `cover`, `contain`, `blur_fill` | `cover` | `cover` crops to fill, `contain` letterboxes on `background`, `blur_fill` puts a contained copy over a blurred cover copy |
| `focus_x`, `focus_y` | number | 0.5 | 0 to 1; where the crop and the zoom centre |
| `motion` | Motion | `{"type": "none", "intensity": 0.5}` | |
| `transition_in` | Transition or null | null | transition from the previous clip |
| `color` | ColorAdjust | neutral | |
| `volume` | number | 0 | 0 to 4; the clip's own sound (0 mutes it) |

### Motion

| Field | Values | Default |
|---|---|---|
| `type` | `none`, `zoom_in`, `zoom_out`, `pan_left`, `pan_right`, `pan_up`, `pan_down` | `none` |
| `intensity` | 0 to 1 | 0.5 |

Zooms cover 6 % to 25 % of the frame depending on intensity; pans travel 5 % to 20 %. `pan_left` moves the camera left, so the picture slides right. Motion is rendered by scaling a supersampled frame and cropping it each frame (not with `zoompan`), so slow moves do not jitter.

### Transition

| Field | Values | Rules |
|---|---|---|
| `type` | `cut`, `fade`, `dissolve`, `slide_left`, `slide_right`, `wipe_left`, `wipe_right`, `zoom` | |
| `duration` | 0 to 10 seconds | a `cut` must have duration 0; others may not be longer than either neighbouring clip |

The first clip may not have an incoming transition with a duration above 0. FFmpeg `xfade` names used: `fade` is `fadeblack` (through black), `dissolve` is `fade` (cross-dissolve), `slide_left` is `slideleft`, `slide_right` is `slideright`, `wipe_left` is `wipeleft`, `wipe_right` is `wiperight`, `zoom` is `zoomin`.

### ColorAdjust

| Field | Range | Default |
|---|---|---|
| `preset` | `none`, `warm`, `cool`, `vivid`, `film`, `mono` | `none` |
| `brightness` | -1 to 1 | 0 |
| `contrast` | 0 to 3 | 1 |
| `saturation` | 0 to 3 | 1 |

Preset filters: `warm` and `cool` use `colorbalance`; `vivid` is `eq` contrast 1.06 and saturation 1.28; `film` is a `curves` S-curve with `eq` saturation 0.85 and a `colorbalance` tint; `mono` is `hue=s=0`.

### AudioClip

| Field | Type | Default | Rules |
|---|---|---|---|
| `asset_id` | string | required | an audio asset or a video with sound |
| `start` | number | required | >= 0, seconds on the timeline |
| `source_in` | number | 0 | >= 0, seconds into the source |
| `duration` | number | required | > 0 |
| `volume` | number | 1 | 0 to 4 |
| `fade_in`, `fade_out` | number | 0 | >= 0 seconds |

### TextClip

| Field | Type | Default | Rules |
|---|---|---|---|
| `text` | string | required | 1 to 500 characters |
| `start` | number | required | >= 0 |
| `duration` | number | required | > 0 |
| `position` | `top`, `center`, `bottom` | `bottom` | |
| `role` | `title`, `caption` | `caption` | title: bold, 8 % of the frame height; caption: 5.5 % |
| `size_scale` | number | 1 | above 0, at most 4; multiplies the size |

Text is written as an ASS subtitle file and burned in with libass (font Arial, with the system's fallback when Arial is missing; the Docker image ships DejaVu fonts). Margins are 6 % of the frame.

### LicenceRecord

| Field | Type |
|---|---|
| `source` | `user`, `library`, `procedural` |
| `track_id` | string or null |
| `title` | string or null |
| `licence` | string (required) |
| `attribution` | string or null |

See [styles-and-music.md](styles-and-music.md#the-licence-record).

### Clip timing

Clip start times are not stored; they are derived:

```text
start[0] = 0
start[i] = start[i-1] + duration[i-1] - transition_in[i].duration
total    = start[last] + duration[last]
```

A transition overlaps the end of one clip with the start of the next, so it shortens the total. Durations are not required to be whole frames: the renderer rounds each clip to the nearest frame (at least one).

### API limits for explicit renders

`POST /v1/renders`, `POST /v1/renders/estimate` and timeline revisions accept at most 1000 clips and a total of at most 600 seconds. Clip assets must exist and match `kind`; audio assets must be audio or a video with sound.

## A complete example

A 9:16 render with an image that zooms in, a video that slides in, an image cut in with a blurred fill, a music track with a fade-out, a title and a library licence. This file was validated against `montaj_core.timeline.schema.Timeline` and the API's `RenderCreateRequest`; its clip starts are 0.0, 1.8 and 4.0 seconds and the total is 6.1 seconds.

```json
{
  "version": "1",
  "seed": 42,
  "output": {"width": 1080, "height": 1920, "fps": 30, "video_codec": "h264", "quality": "standard"},
  "background": "#101010",
  "clips": [
    {
      "id": "clip_000",
      "asset_id": "ast_1a2b3c4d5e6f7a8b9c0d1e2f",
      "kind": "image",
      "duration": 2.0,
      "source_in": 0.0,
      "speed": 1.0,
      "fit": "cover",
      "focus_x": 0.5,
      "focus_y": 0.4,
      "motion": {"type": "zoom_in", "intensity": 0.6},
      "transition_in": null,
      "color": {"preset": "vivid", "brightness": 0.0, "contrast": 1.0, "saturation": 1.0},
      "volume": 0.0
    },
    {
      "id": "clip_001",
      "asset_id": "ast_9f8e7d6c5b4a39281706f5e4",
      "kind": "video",
      "duration": 2.2,
      "source_in": 1.5,
      "speed": 1.0,
      "fit": "cover",
      "focus_x": 0.5,
      "focus_y": 0.5,
      "motion": {"type": "none", "intensity": 0.5},
      "transition_in": {"type": "slide_left", "duration": 0.2},
      "color": {"preset": "vivid", "brightness": 0.0, "contrast": 1.0, "saturation": 1.0},
      "volume": 0.3
    },
    {
      "id": "clip_002",
      "asset_id": "ast_0f1e2d3c4b5a69788796a5b4",
      "kind": "image",
      "duration": 2.1,
      "source_in": 0.0,
      "speed": 1.0,
      "fit": "blur_fill",
      "focus_x": 0.5,
      "focus_y": 0.5,
      "motion": {"type": "pan_right", "intensity": 0.5},
      "transition_in": {"type": "cut", "duration": 0.0},
      "color": {"preset": "vivid", "brightness": 0.05, "contrast": 1.1, "saturation": 1.0},
      "volume": 0.0
    }
  ],
  "audio": [
    {
      "asset_id": "ast_5e6f7a8b9c0d1e2f3a4b5c6d",
      "start": 0.0,
      "source_in": 0.5,
      "duration": 6.1,
      "volume": 1.0,
      "fade_in": 0.0,
      "fade_out": 1.0
    }
  ],
  "texts": [
    {
      "text": "Summer 2026",
      "start": 0.0,
      "duration": 2.0,
      "position": "center",
      "role": "title",
      "size_scale": 1.0
    }
  ],
  "licence": {
    "source": "library",
    "track_id": "mus_4d5e6f7a8b9c0d1e2f3a4b5c",
    "title": "Sunrise",
    "licence": "CC-BY-4.0",
    "attribution": "Artist Name"
  }
}
```

Its estimate (`POST /v1/renders/estimate` or `montaj render timeline.json --estimate`):

```json
{"object": "render_estimate", "duration": 6.1, "frames": 183, "clips": 3, "segments": 3, "transitions": 1,
 "transition_types": ["slide_left"], "cuts": 1, "texts": 1, "audio_tracks": 1,
 "output": {"width": 1080, "height": 1920, "fps": 30, "video_codec": "h264", "quality": "standard"}}
```

Replace the asset ids with your own before rendering:

```bash
uv run montaj render timeline.json --out render.mp4
```

The SDK and the curl body wrap it: `{"timeline": { ... }}`.

## Editing a returned Timeline

1. Read `output.timeline` from a succeeded job (`GET /v1/jobs/{id}`, the MCP `get_job` tool with `include_timeline=true`, or `montaj jobs get`).
2. Change what you want: swap an `asset_id`, change a `duration`, a `transition_in`, a `color` preset, a text.
3. Send the whole Timeline back: `POST /v1/jobs/{id}/revisions` with `{"timeline": ...}`, or `montaj revise job_... --timeline patched.json`.

Each clip is rendered to a segment once and cached by a key built from everything that affects its pixels: kind, duration, source_in, speed, fit, focus, motion, colour, the asset's content hash, the background (when it can show), the canvas, the frame rate, the quality and the renderer version. The clip id, `volume`, `transition_in` and the output codec are not part of the key. So changing a transition, the audio, a text or one clip's colour re-renders only the clips whose pixels changed; the job reports `output.segments.reused`.

## EditPlan (the director's output)

```json
{
  "title": "Summer Trip",
  "rationale": "Open on the widest shot, build pace into the chorus.",
  "style": "upbeat",
  "aspect": "9:16",
  "music": {"asset_id": "ast_MUSIC", "start_beat": 0},
  "shots": [
    {"asset_id": "ast_A", "beats": 4, "source_in": 0.0, "motion": "zoom_in", "transition": "cut", "focus_x": 0.5, "focus_y": 0.4},
    {"asset_id": "ast_CLIP", "beats": 2, "source_in": 1.2, "motion": "none", "transition": "slide_left", "focus_x": 0.5, "focus_y": 0.5}
  ],
  "texts": [{"text": "Summer Trip", "at_shot": 0, "shots_long": 1, "position": "center", "role": "title"}]
}
```

- `shots[i].beats`: how many beats the shot lasts. `transition` is the transition **into** the shot (ignored on the first).
- `source_in`: seconds into a video (ignored for images).
- `music.start_beat`: the beat of the track where the edit starts.
- `texts[i]`: shown from shot `at_shot` for `shots_long` shots.

The JSON Schema sent to Claude has no numeric bounds (the structured-output format does not accept them); every object forbids extra properties and requires every property. Bounds are checked afterwards by the validator.

### Validation

`validate_plan` checks the plan against the analysed assets and the beat grid. Error codes:

| Code | Problem |
|---|---|
| `empty_plan` | no shots |
| `unknown_asset` | a shot names an asset that is not a candidate |
| `unsupported_asset_kind` | a shot uses an audio asset |
| `invalid_beats` | beats below 1 |
| `shot_too_short` | a shot shorter than 0.3 s at this tempo |
| `source_out_of_range` | `source_in` outside the video |
| `plan_exceeds_track` | the shots run past the end of the music |
| `repeated_asset` | the same asset in consecutive shots |
| `invalid_focus` | focus outside 0..1 |
| `invalid_start_beat` | `start_beat` outside the grid |
| `unknown_music_asset`, `invalid_music_asset` | the plan's music asset is unknown or not audio |
| `invalid_text_range` | a text points outside the shots |
| `empty_text` | a blank text |

The Claude director gets the errors back and may repair the plan up to two times. If the plan is still invalid, the heuristic plan is used (`director.source: "heuristic_fallback"`). If that is also invalid, the job fails with `director_failed`.

## Conform: beats to frames

`conform(plan, grid, assets, output, options)` produces the Timeline.

- **Boundaries are quantised one by one from the start beat.** Shot boundary k lands at `round((beat_time(s + c_k) - beat_time(s)) * fps)` frames, where `s` is the start beat and `c_k` the cumulative beats. Rounding never accumulates, so every cut is within half a frame of its beat.
- **Transitions are centred on the beat.** A non-cut transition gets an even number of frames (`transition_beats`, capped at `max_transition_seconds` and by both neighbouring shots), half before the beat and half after.
- **The ending lands on a downbeat or section boundary.** The last shot is extended by up to `max_extend_beats` (4) beats to reach one; otherwise it is trimmed back to the previous one.
- **Video shots longer than their source are slowed** (`speed` below 1) rather than frozen.
- **Music** becomes one AudioClip starting at 0 with `source_in` at the start beat's time and a fade-out.
- Clip ids are `clip_000`, `clip_001` and so on.

`ConformOptions` defaults (the style presets override several, see [styles-and-music.md](styles-and-music.md)):

| Option | Default |
|---|---|
| `transition_beats` | 1 |
| `max_transition_seconds` | 1.0 |
| `motion_intensity` | 0.5 |
| `fit` | `cover` |
| `music_volume` | 1.0 |
| `fade_out_seconds` | 1.5 |
| `clip_volume_with_music` | 0.0 |
| `clip_volume_without_music` | 1.0 |
| `end_on_downbeat` | true |
| `max_extend_beats` | 4 |

Without music (video edits without `music_asset_id`) the beat grid is a virtual grid with a beat every 0.5 s.

### Canvas sizes

Montages and video edits pick the canvas from the aspect:

| Aspect | standard and high | draft |
|---|---|---|
| 16:9 | 1920x1080 | 960x540 |
| 9:16 | 1080x1920 | 540x960 |
| 1:1 | 1080x1080 | 540x540 |
| 4:5 | 1080x1350 | 540x676 |

Draft halves each side and rounds to an even number. Explicit renders use the Timeline's `output` as given, at any quality.

## Encoding

| Quality | h264 CRF | h265 CRF | x264/x265 preset |
|---|---|---|---|
| `draft` | 28 | 30 | veryfast |
| `standard` | 20 | 24 | medium |
| `high` | 17 | 21 | slow |

- Output: MP4 with `+faststart`, yuv420p, AAC audio at 192 kb/s, 48 kHz.
- Cached segments are intermediate files encoded at CRF 18 (draft), 12 (standard) or 10 (high) with preset veryfast, so the final encode starts from near-lossless material.
- Transitions are joined with `xfade` in chunks of 8 clips.
- HDR sources (PQ or HLG) are tone-mapped to BT.709 with `zscale` and `tonemap=hable`.
- Audio: the music is ducked under clip sound with `sidechaincompress`, then the mix is normalised in two `loudnorm` passes to -14 LUFS integrated, -1.5 dBTP true peak, LRA 11. Clip sound gets 5 ms fades at cuts to avoid clicks.
- The poster is a JPEG frame from the middle of the first clip.
