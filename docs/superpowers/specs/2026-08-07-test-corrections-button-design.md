# Test corrections button — Mood speech corrections tab

## Problem

`admin → Moods → Speech` lets an operator maintain find-and-replace speech
corrections (`settings.tts.corrections`, applied by
`audio/speech-text.ts:normalizeForSpeech`) — e.g. "GHz" → "gigahertz",
"Hozier" → "Ho-zeer". There's no way to hear a correction actually applied
before saving. The operator has to save, wait for a real on-air line to use
the word, and listen live.

## Goal

A "Test corrections" control in the Speech tab: operator types a test
sentence, hits play, hears it synthesized through the station's **default**
TTS engine/voice with the corrections **currently in the list** applied —
including unsaved edits, so testing a new rule doesn't require Save first.

## Non-goals

- No engine/voice picker in this tab. Uses `settings.tts.defaultEngine` and
  that engine's configured voice, unchanged.
- No auto-generated test sentence. Free-text input only.
- Does not persist anything. Purely a preview, like the existing "Play
  sample" buttons elsewhere in Settings → TTS.

## Design

### Server (`controller/`)

**`audio/tts.ts` — `synthesizeSample()`**

Add one new optional param:

```ts
corrections?: { from: string; to: string }[]
```

When present, sanitize and use it in place of `settings.get().tts?.corrections`
for that single synth call:
- cap to 100 entries (mirrors `MoodsPanel`'s row cap)
- `from` trimmed to ≤80 chars, `to` to ≤160 chars (mirrors the tab's
  `Input maxLength`s)
- non-string / malformed entries dropped, not rejected — this is a scratch
  preview, not a persisted write, so lenient-drop beats a 400

When the param is omitted (every existing call site), behavior is byte-for-
byte unchanged — falls through to the saved `settings.tts.corrections`.

**`routes/settings/tts.ts` — `POST /settings/tts/preview`**

Parse `body.corrections` the same defensive way the route already parses
`voiceSettings`/`fishSettings` (array check, else `undefined`), pass through
to `synthesizeSample`.

### Web (`web/`)

**`components/admin/tts/previewApi.ts`**

Add to `PreviewParams`:
```ts
text?: string;
corrections?: { from: string; to: string }[];
```
Both already flow through the JSON body as-is (no new wiring needed beyond
the type + the existing `JSON.stringify(params)`).

**`components/admin/tts/VoicePreviewButton.tsx`**

Add `text?: string` and `corrections?: { from: string; to: string }[]` props,
forwarded into `fetchPreviewSample`'s params. Deliberately **excluded** from
the stale-sample reset effect's dependency array — same treatment as the
existing `voiceSettings` prop (unstable inline object/array at call sites;
only engine/voice/speed/etc. changes should discard a sample).

**`components/admin/MoodsPanel.tsx`**

- `load()` additionally reads `values.tts.defaultEngine` plus the matching
  per-engine voice field (`kokoro.voice` / `chatterbox.referenceVoice` /
  `pocketTts.voice` / `cloud.voice`), `cloud.provider`, `cloud.model`, and
  `speed[engine]` — same lookups `TtsSection.tsx`'s "Play sample" row
  already does (`web/components/admin/settings/TtsSection.tsx:1381-1398`).
  Stored in a small `testVoice` state object; read-only in this tab (no
  editing — operator changes the real voice in Settings → TTS).
- New block under the corrections list, above/near the existing "Add
  correction" / "Save corrections" buttons:
  - A labeled text input, `maxLength={200}` (matches server's
    `PREVIEW_TEXT_MAX`), placeholder something like "Type a line using a
    word you corrected…".
  - `<VoicePreviewButton>` wired with the loaded `testVoice` fields, plus
    `text={testText}` and `corrections={effectiveCorr}` (the tab's existing
    computed working list — already trims/filters live edits, see
    `MoodsPanel.tsx:164-168`), `disabled={!testText.trim()}`.
- Small field-hint: "Uses the corrections above, including unsaved changes,
  spoken by the station's default voice."

## Data flow

```
testText (typed) ──┐
effectiveCorr ──────┼──> VoicePreviewButton ──> POST /settings/tts/preview
testVoice (engine/  ┘         { engine, voice, cloudProvider, cloudModel,
 voice/provider/                speed, text, corrections }
 model/speed, from
 saved settings)                        │
                                         v
                          synthesizeSample() applies
                          normalizeForSpeech(text, corrections override)
                                         │
                                         v
                              WAV/MP3 blob ──> <audio> plays
```

## Error handling

Unchanged from the existing preview endpoint: synth failure → 422 with a
message, surfaced by `VoicePreviewButton`'s existing error state. No new
failure modes introduced — corrections sanitization drops bad rows rather
than erroring.

## Testing

- `synthesizeSample`'s corrections-override branch is a small, pure-ish
  addition; cover it in `controller/scripts/` alongside the existing
  `speech-text.test.ts` / TTS fallback tests if a natural seam exists,
  otherwise exercise via the existing preview-route test surface if one
  exists. (No dedicated new test file is required by this spec — check at
  implementation time whether `scripts/*.test.ts` already covers
  `/settings/tts/preview` and extend it rather than adding a new file.)
- Manual: add a correction row (don't save), type a sentence containing the
  "from" word, hit test, confirm the spoken output uses the "to" form.
