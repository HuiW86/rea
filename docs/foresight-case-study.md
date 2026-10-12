# Case study: Google AI Edge Foresight on macOS

This investigation used REA to establish an application inventory, then combined
bounded native analysis, experimental Dart snapshot recovery, and synthetic
meeting playback to understand Foresight's download and meeting workflows.
It found a concrete shutdown ordering defect and reproduced transcript loss,
while leaving the upstream cause of missing long-utterance finals unresolved.

The accompanying [interactive walkthrough](./case-studies/foresight/index.html)
is in Chinese. Download `docs/public/case-studies/foresight/` and open `index.html`
for offline reading, including seven chronological reports and five locally
captured screenshots. Its controls simulate traced rules; they do not run
Foresight, record audio, or download models. External reference links need a
network connection. On GitHub, browse the bundle through the repository's
`docs/public/case-studies/foresight/` directory; raw HTML is not a hosted preview.

## Baseline and provenance

| Item                      | Recorded baseline                                                  |
| ------------------------- | ------------------------------------------------------------------ |
| Investigation date        | 2026-10-11                                                         |
| Target                    | Google AI Edge Foresight, macOS arm64                              |
| Application version/build | 1.0.0 / 1.0.0.28.20261005                                          |
| Main executable SHA-256   | `fa98b856bfbe9b40cdebfb9f975f7867ec5c0a3ca75b81214a096a60b6db8326` |
| REA                       | 6.4.0, Node 24.19.0                                                |
| Ghidra                    | 12.1.4; locally built arm64 decompiler                             |
| Supplemental Dart reader  | DAE 0.1.13 with an **experimental** SDK profile                    |
| Dart snapshot             | 3.14.0-294.0.dev; hash `e3b703b387b759dc21a3cbe8aa329f7a`          |

The original application remained Google-signed. Analysis did not replace its
executable or models. No original Foresight source code was obtained. Public
product context: [Google's Foresight page](https://developers.google.com/edge/foresight).

The [manifest](./case-studies/foresight/manifest.json) distinguishes distributed
files and embedded screenshots from hashes identifying evidence retained in the
local archive. An absent raw artifact is not made publicly available by its hash.
This public case excludes application binaries, model weights, full extracted
prompts, provider project databases, personal meeting databases, application API
keys, and private project build logs. Report references to original evidence
folders are archive identifiers, not promises that those files are distributed.

Two model setup screenshots in the original 39-image collection came from
[this external article](https://talat.app/blog/google-ai-edge-foresight), not a
locally verified first-run download. They are linked rather than embedded here.
The five embedded captures show local setup and synthetic meeting experiments.

## The investigation, including failures

| Stage               | What ran                                                                                                                           | What the result establishes                                                                                    |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| Inventory           | REA `inspect_artifact`, `inspect_macho`, `inspect_plist`, `inspect_signature`, `apple_application_graph`, `trace_dylib_resolution` | Application inventory and entry points; `complete-within-inventory` does not mean full business reconstruction |
| Deep import         | REA/Ghidra import, bounded by 1,200,000 ms                                                                                         | Timed out after 20 minutes; no successful full analysis session                                                |
| Recovery            | Bounded Ghidra `-noanalysis`, symbols, LLVM disassembly                                                                            | Selected function bodies and call sites; supplemental work, not default REA import success                     |
| Dart recovery       | Experimental DAE profile, official SDK layouts, independent symbol checks                                                          | Concrete fields and function addresses checked against instructions; not original source recovery              |
| Downloads           | Trace URL construction, retry objects, branches; small HTTP Range probes                                                           | Protocol and retry/length rules; old candidate IDs returned 404, no verified current full download URL         |
| Workflows           | Trace transcription, query dispatch, retrieval, enhancement, meeting end                                                           | Static implementation evidence, distinct from runtime acceptance                                               |
| Playback            | Fixed English/Chinese synthetic audio in the real application                                                                      | One English Q&A and note enhancement observed; long transcript coverage and Chinese playback failed            |
| Loss follow-up      | Read only synthetic-marked stored records; empty-session continuous and paused playback                                            | Loss is not just hidden UI text and does not require note editing; upstream cause remains open                 |
| Chinese alternative | Direct paced Chinese replay into an existing Apple speech integration                                                              | One synthetic replay passed; no far-field, diarization, or cross-platform acceptance                           |

Chronological reports retain what was unknown at each stage. Later reports
supersede early gaps; do not combine an early unknown with a later confirmation
as if they were contradictory observations.

## Downloads: address, retry, resume, and validation

The traced address template is:

```text
https://odml.pa.googleapis.com/v1/medias/foresight/{blobId}:download?alt=media&key=<redacted>
Range: bytes={currentOffset}-{endInclusive}
```

The blob identifier comes from model metadata. Local filenames are candidates,
not proof of a current server-side identifier. Five old filenames and one
extension-free candidate returned 404 in small Range probes. This does not
establish server-wide model removal or provide a usable current direct link.

There are five total attempts, including the initial request. Nominal waits are
1, 2, 4, and 8 seconds with ±25% jitter, capped at 10 seconds. The HTTP retry list
is 429, 500, 502, 503, and 504; 501 is excluded. Network client, socket, and timeout
exception families have separate retry branches.

Parallelism is `clamp(ceil(expectedBytes / 838860800), 1, 8)`. The in-flight
current offset advances as data arrives, allowing retries within a run to resume.
Persistence of progress across application restarts was not established. A 206
response is accepted; a 200 response is allowed for the original whole-file
chunk. That does not establish strict checks of every `Content-Range` field.

The traced paths check status, nonempty data, file existence, and expected size.
No model content hash/signature comparison was found in those paths. Preallocated
length alone cannot establish that every byte was written. Short-stream or corrupt
payload injection was not performed, so this remains a static validation concern,
not a reproduced corruption exploit.

## Three separate pipelines

- **Transcription:** microphone/system PCM → source buffering → worker isolate
  and serialized/merged requests → interim/final text → source-aware buffers,
  cleaning, UI publication, and persistence. Capture uses 48 kHz, processor
  configuration uses 24 kHz, and a native resampler exists. Different numbers
  alone do not demonstrate a sampling bug.
- **Question detection:** stable text window → dedicated local model → parsing →
  hash and approximate duplicate filtering → new/obsolete query IDs. The traced
  detector receives text even though its manager also prepares audio. Interim
  text drops its last five whitespace words and keeps a latest-200-word window;
  hash suppression uses five seconds, approximate state uses ten seconds, and
  traced similarity thresholds are 0.7/0.8/0.9. One displayed duplicate card does
  not prove both repeated questions reached the detector.
- **Note enhancement:** each shorthand item retrieves related transcript chunks
  by embedding similarity → shared excerpts are grouped within an estimated
  budget → local streamed generation → item-indexed updates and original-text
  fallback for missing output. The estimate `350 × excerpts + 50 × items + 300`
  is bounded by 2000; it is not a measured tokenizer count. UI triggers include
  Enter, a 60-second edit/transcript timer, and pending-run coalescing; saving
  has a separate one-second delay.

Prompt instructions to preserve names, bind excerpts, and avoid invented facts
express design intent, not guaranteed factual verification. The observed
`venue TBD` item remained unchanged in one experiment, not a comprehensive
hallucination benchmark.

Silence stopping defaults to 15 minutes. Scheduled ending, stale notifications,
and wake-up checks have separate branches. A strict greater-than-15-minute
scheduled overrun check does not prove an independent exact-minute stop timer.

## Transcript loss: confirmed mechanism and open cause

The English input lasted 26.293 seconds. Stored synthetic-marked records retained
only a prefix. A new empty session reproduced an interim prefix disappearing at
stop without note editing. The same samples split at 9/18 seconds with five
seconds of silence after each segment lasted 41.293 seconds and still failed.
Microphone/system routes were not isolated and captured samples were not counted.

The independently traced shutdown order is:

```text
startDetecting:
  result stream subscription stored at field offset 0x2f
  audio input subscription stored at field offset 0x37
stopDetecting:
  0x1006f7744–0x1006f7788 cancel result subscription (0x2f)
  0x1006f778c–0x1006f77d0 cancel audio subscription (0x37)
  0x1006f77d8             stop capture
  0x1006f7808             endTranscription / flush
```

A final emitted after flush cannot reach the already canceled result subscription.
This establishes a shutdown defect, **not the sole cause of all observed loss**.
The manager's existing interim flush did not preserve these long drafts; its
field-level failure reason remains unresolved. The audio-request timeout is
30 seconds; its `TimeoutException` branch returns without the restart used by
other exceptions. There was no runtime log proving that timeout occurred here.

A safer implementation stops new capture, submits the tail, drains accepted
samples, keeps consuming results, obtains persistence acknowledgment, then
cancels subscriptions. On timeout it retains audio and drafts and marks the tail
unconfirmed. Finalized intervals must not be overwritten by an empty result.

## Chinese adaptation and reconstruction readiness

An existing private meeting project supplied a Chinese Apple speech integration
on macOS/iOS 26+. Direct real-time-paced replay of a 16 kHz mono Float32 synthetic
sample produced first text at 1.307 seconds, 103 partial results, and one final
result; input ended at 28.095 seconds. One integration test passed with zero
failures, retaining names, currency, questions, and the final progress-check
phrase. Homophone errors remained.

That project's checkout was `a10e150a6da09c0f14a0fda5e91605802daa9308`, **with
uncommitted changes**. This fork contains an observation summary, not that private
implementation or a reproducible clean-checkout benchmark. A successful direct
replay also does not isolate the cause of Foresight's capture failures.

Candidates were checked against official Apple SpeechAnalyzer, sherpa-onnx,
SenseVoice, and whisper.cpp documentation. The default for that existing project
was reuse of its installed Apple Chinese path. Cross-platform streaming remains
an additional evaluation: the documented online Paraformer candidate lacks
timestamps, and SenseVoice requires separate diarization components. No
same-sample benchmark established a winner among all candidates.

For Chinese, replace whitespace word windows with a stable-prefix and
language-aware segmentation strategy; recalibrate retrieval and duplicate
thresholds using Chinese cases. Protect names, currencies, negation, and promise
strength. Keep live captions provisional and retain audio for authoritative
post-stop transcription.

There is enough evidence to begin a UI/workflow MVP. Equivalent behavior,
long-session integrity, actual Chinese meetings, multiple speakers, overload,
route isolation, corrupted downloads, and recovery still require acceptance
work. The case is an investigation record, not a finished clone or a provider
conformance claim.

## Reading and checking the public bundle

The walkthrough contains ten selectable stages, a glossary, retry simulations,
three pipeline tabs, a bad/good shutdown stepper, and screenshot enlargement.
Seven linked reports retain detailed addresses and historical limits. The
experimental profile and 19-function symbol cross-check are downloadable.
The profile derives from DAE; its original MIT license is preserved in
`dae-profile-LICENSE.txt` inside the bundle.

The original walkthrough passed 21 checks with DOM test doubles. These establish
script/control behavior only; they do not verify browser rendering or layout.
For this public export, verify that each relative link resolves, its JSON and
inline script parse, and all distributed hashes match `manifest.json`. The
export does not repeat the original application experiments.

To investigate a matching installation, start from [Apple application
analysis](./apple-application-analysis.md) and [native
investigation](./native-investigation.md), collect a new inventory and executable
hash, and enumerate the connected MCP tools before invoking their schemas.
Do not blindly reuse addresses or the experimental Dart profile on another
build. Native analysis providers are bring-your-own; the case does not install
or bundle them.
