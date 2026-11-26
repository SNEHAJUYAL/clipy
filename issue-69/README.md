# capture: list dshow devices that offer audio alongside video

Resolves clipy-recorder/clipy#69 (hiring task N11).

## What was wrong

On Windows, a device that offers both audio and video, like a capture card (`"Cam Link 4K"`) or a webcam with a built-in mic, never showed up as a microphone. Picking it by name in `startCapture` then failed with `no microphone named "..."`.

## Why it happened

Recent ffmpeg prints every media type a dshow device offers, joined with `", "`:

```
[dshow @ ...] "Cam Link 4K" (audio, video)
```

This comes from [`libavdevice/dshow.c` L579–L590](https://github.com/FFmpeg/FFmpeg/blob/288839e860c7e55e466170215fc478fa7e750e97/libavdevice/dshow.c#L579-L590): it prints the first type, then `", %s"` for each extra one, `unknown` for a type it can't name, and `(none)` when there are none. FFmpeg 6.1 has the same code.

`parseDshowAudioDevices` only accepted a single `(audio|video|none)`. A line with two types matched neither the tagged pattern nor the legacy one, so the device was silently dropped.

## What changed

- `packages/capture/src/sources.ts`: the tagged pattern now accepts a comma-separated list of types, and a device is kept if `audio` is one of them. That's two lines plus a one-line comment. Nothing else in the function changes: "Alternative name" lines are still skipped, and the older sectioned format goes through the same path as before.
- `packages/capture/test/sources.test.ts`: one new fixture (CRLF line endings, like real Windows output) and one test.

No new dependencies, and no changes outside the capture package.

## How I tested it

On Linux (Node 22.23.5, pnpm 10.18.0):

- **Bug reproduced on `main`:** for `"Integrated Camera" (video)`, `"Cam Link 4K" (audio, video)` and `"Microphone (Realtek(R) Audio)" (audio)`, the parser returned only the Realtek mic.
- **New test fails without the fix and passes with it.** It checks that `(audio, video)` and `(video, audio)` are listed, that `(video)` and `(unknown)` are not, and that alternative names stay out.
- `pnpm --filter @clipy/capture test`: 68 passed (67 before, plus the new one).
- `pnpm test` (whole workspace): moments 27 passed, capture 68 passed, transcribe 73 passed / 5 skipped, config 36 passed.
- `pnpm run typecheck` passes, and `git diff --check` is clean.
- I also tried some extra cases in a scratch test that isn't part of this PR, and all behaved as expected:
  - `(audio,video)` with no space
  - three types
  - `(none)` and `(data)`
  - a video-only device with "Audio" in its name
  - a legacy-format name with parentheses
  - trailing spaces with CRLF

## Limitations

I don't have a Windows machine with a multi-type device, so I tested against fixtures built from the ffmpeg source above rather than live `ffmpeg -list_devices` output.
