# Styles and music

## Style presets

A style sets pacing, transitions, camera motion, colour, framing, the title, the preferred aspect and how the music sits under clip sound. Seven presets exist (`packages/core/src/montaj_core/director/styles.py`). Both directors use them: the heuristic director follows them exactly, and Claude receives the table as guidance.

### Choosing a style

In order:

1. `style` in the request.
2. Keywords in the prompt (`infer_style`).
3. For the heuristic director only: the music's tempo (118 BPM or faster is `upbeat`, 88 or slower is `calm`, otherwise `cinematic`).
4. The default, `upbeat`.

Prompt keywords (a trailing `*` matches any word starting with it):

| Style | Keywords |
|---|---|
| cinematic | cinematic, cinema*, film, movie, epic, dramatic, drama, trailer, moody, story, storytelling, documentary |
| upbeat | upbeat, hype, energetic, energy, fun, fast, happy, summer, vibe*, sport*, workout, gym, dynamic, punchy |
| calm | calm, relax*, peaceful, serene, gentle, slow, chill, sunset, sunrise, meditat*, cozy, cosy, soft, quiet, tranquil, dreamy, nature |
| travel | travel*, trip, vacation, holiday*, beach, journey, adventure, explor*, road trip, roadtrip, city, mountain*, hike, hiking, tour, abroad, island, wanderlust |
| celebration | celebrat*, birthday, bday, party, wedding, anniversary, graduation, new year, festival, congrat*, dance, christmas, holi, diwali, reunion |
| product | product*, launch, unbox*, showcase, brand*, advert*, commercial, ecommerce, e-commerce, shop, store, promo*, sale, catalog*, catalogue |
| minimal | minimal*, simple, clean, elegant, portfolio, sleek, understated, monochrome, gallery |

Each keyword hit scores a point, and naming the style itself adds two more (so it counts three times). The highest score wins. On a tie `celebration` wins, otherwise the order of the table above decides. No hits means no inferred style.

Example: "upbeat summer trip" scores upbeat 4 (`upbeat` counts three, `summer` one) and travel 1 (`trip`), so the style is `upbeat`.

### The presets

Shot length depends on the music's energy at that point: low below 0.45, high from 0.72, mid in between. The target seconds are converted to a whole number of beats, snapped to 1, 2, 4, 8 or 16 beats and never below 0.3 s.

| Style | Shot seconds (low / mid / high energy) | Transitions | Motion (intensity) | Colour | Fit | Title | Aspect |
|---|---|---|---|---|---|---|---|
| cinematic | 4.0 / 2.7 / 1.6 | dissolve, fade; 1 in 4 shots | zoom_in, pan_right, zoom_out, pan_left (0.35) | film | cover | center | 16:9 |
| upbeat | 2.0 / 1.2 / 0.6 | slide_left, zoom, slide_right; 1 in 6 | zoom_in, pan_left, zoom_out, pan_right (0.6) | vivid | cover | center | 9:16 |
| calm | 5.0 / 3.6 / 2.6 | dissolve; 1 in 2 | zoom_in, pan_right, zoom_out, pan_left (0.25) | warm | cover | bottom | 16:9 |
| travel | 3.0 / 2.0 / 1.0 | slide_left, dissolve, wipe_left; 1 in 5 | pan_right, zoom_in, pan_left, zoom_out, pan_up (0.5) | warm | cover | center | 16:9 |
| celebration | 2.0 / 1.0 / 0.5 | zoom, fade; 1 in 6 | zoom_in, zoom_out, pan_left, pan_right (0.7) | vivid | cover | center | 9:16 |
| product | 2.6 / 1.6 / 0.9 | slide_left; 1 in 8 | zoom_in, pan_right, zoom_out, pan_left (0.3) | none | blur_fill | bottom | 1:1 |
| minimal | 3.6 / 2.6 / 1.8 | hard cuts only | zoom_in, zoom_out (0.2) | none | cover | no title | 4:5 |

| Style | Transition into a calmer section | Transition length (beats, max seconds) | Clip volume under music | Music fade-out (s) | Preferred tempo (BPM) |
|---|---|---|---|---|---|
| cinematic | dissolve | 2, 1.2 | 0.25 | 2.5 | 90 |
| upbeat | slide_left | 0.5, 0.4 | 0 | 1.0 | 124 |
| calm | dissolve | 2, 1.5 | 0.2 | 3.0 | 76 |
| travel | dissolve | 1, 0.8 | 0.3 | 2.0 | 110 |
| celebration | fade | 0.5, 0.4 | 0.15 | 1.5 | 128 |
| product | slide_left | 0.5, 0.4 | 0 | 1.0 | 116 |
| minimal | cut | 1, 0.5 | 0 | 1.5 | 100 |

All other cuts are hard cuts on the beat. Motions are used in rotation.

### Aspect

For montages, in order: `aspect` in the request; an aspect named in the prompt; the style's aspect. If the style asks for 16:9 but at least two thirds of the material is portrait, the heuristic director switches to 9:16.

Words that name an aspect:

| Aspect | Words |
|---|---|
| 9:16 | 9:16, vertical, reel, reels, tiktok, shorts, story, stories |
| 1:1 | 1:1, square |
| 4:5 | 4:5 |
| 16:9 | 16:9, landscape, widescreen, horizontal, youtube |

For video edits without an `aspect`, the canvas follows the prompt, else the canvas closest to the first video's shape.

### Length

Without `target_duration`, a montage is as long as the material sustains: between 10 and 30 seconds without music, up to 120 seconds with music. With `target_duration` (3 to 600 s) the plan is fitted to that length on the beat grid. The heuristic title is at most five words and is held for about 2.5 seconds.

## Music selection

For each montage or video edit, in order:

1. **`music_asset_id` in the request.** If the asset is registered as a library track, its library licence is used. Otherwise the licence record is `source: "user"` with the text "Supplied by the user; the uploader is responsible for the rights to this track".
2. **Montages only: a track from the organisation's library** (`/v1/music`), scored for the style. See below.
3. **Montages only: a procedural bed** generated for the style.

Video edits never pick or generate music. Without `music_asset_id` they have no music track and plan on a 0.5 s virtual beat grid.

A track shorter than 2 seconds, or with fewer usable beats than one bar, fails the job with `music_too_short`.

### Beat analysis

Beats, downbeats, tempo and coarse sections with an energy value come from librosa (`analysis/beats.py`). Beat times are refined towards the nearest attack, half and double tempo errors are folded, and the downbeat phase is chosen from accent strength. Results are cached per file content in `MONTAJ_CACHE_DIR/beats/`.

librosa is the only beat-tracking backend; there is no optional alternative.

### Library scoring

Each library track gets a score for the style:

```text
fit = 0.35 * tag + 0.30 * tempo + 0.20 * energy + 0.15 * duration
```

| Part | Score |
|---|---|
| tag | 1 if the style name is one of the track's tags; 0.6 if any of the style's tags is; else 0 |
| tempo | `max(0, 1 - 3 * relative distance)` to the style tempo, also trying half and double time; halved when outside the style's tempo range |
| energy | `1 - |track energy - style energy|` |
| duration | `(duration / target) ^ 2` when the track is shorter than the needed length, else 1 |

The highest score wins; ties break on track id. A track with no `tempo` is measured from its audio; a track with no `energy` counts as 0.5.

The worker scores the library from its stored metadata first and fetches only the candidate files: for every style the montage could end up with (the requested one, else each style), the track that would win, plus every track stored without a `tempo` (those cannot be scored without their audio). The pick is the same as scoring every track, so setting `tempo` keeps large libraries cheap.

Style music profiles:

| Style | Tempo (range) | Energy | Tags |
|---|---|---|---|
| cinematic | 90 (70 to 110) | 0.55 | cinematic, epic, orchestral |
| upbeat | 124 (110 to 140) | 0.85 | upbeat, pop, happy, energetic |
| calm | 76 (60 to 95) | 0.3 | calm, ambient, chill, acoustic |
| travel | 110 (95 to 128) | 0.65 | travel, adventure, indie |
| celebration | 128 (115 to 145) | 0.9 | celebration, party, dance |
| product | 116 (100 to 130) | 0.6 | product, corporate, tech |
| minimal | 100 (80 to 120) | 0.4 | minimal, electronic, clean |

Tag your tracks with these words and set `tempo` and `energy` to get predictable picks.

### Adding library tracks

```bash
uv run montaj music add sunrise.mp3 --title "Sunrise" --licence "CC-BY-4.0" \
  --attribution "Artist Name" --tags upbeat,travel --tempo 118 --energy 0.7
```

This uploads the file and calls `POST /v1/music`. `--tempo` (20 to 300 beats per minute) and `--energy` (0 calm to 1 intense) are optional; a track stored without a tempo is measured from its audio when it is scored. Tracks are per organisation and are hidden when their audio asset is deleted. There is no endpoint to delete a track.

`montaj_core.music.library.load_library` can also read a folder with a `library.json` manifest (`{"version": 1, "tracks": [{"file", "id", "title", "tempo", "energy", "tags", "licence": {"licence", "attribution"}}]}`). That loader is a core-library helper only: the API and the worker do not call it, and the live library is the per-organisation `/v1/music`.

### Procedural bed

When a montage has no music and the library has nothing, Montaj synthesises a bed (`music/procedural.py`):

- Four voices: kick, hi-hat, bass and pad, over a four-chord loop in 4/4 at the style's tempo.
- A new section every 8 bars. The last 3 seconds (or 15 % of the length, whichever is shorter) fade out.
- 48 kHz stereo 16-bit WAV, at most 3600 seconds.
- Deterministic: the same style, length and seed give the same bytes.
- Its length is the target length (or what the material sustains) plus 3 seconds.

The bed is stored as an audio asset named `music.wav` and returned in `output.music` with `generated: true`. A prompt revision reuses it.

It is a placeholder with a steady pulse, not a musical arrangement. Use real tracks for anything you publish.

## The licence record

Every video job output has `output.licence` (and the Timeline has `licence`), or null when there is no music:

| `source` | `track_id` | `licence` text |
|---|---|---|
| `user` | the music asset id | "Supplied by the user; the uploader is responsible for the rights to this track" |
| `library` | the `mus_...` track id | the licence you registered, with its `title` and `attribution` |
| `procedural` | `procedural-<style>-<seed>` | "Generated by Montaj's procedural synthesiser; no third-party rights" |

Montaj does not check licences. The record says where the music came from; you are responsible for the rights to what you upload or register.

For explicit renders the licence is whatever the Timeline carries. A timeline revision keeps the parent's licence when the patched Timeline has none and uses only the parent's audio assets.
