<!-- Visibility: public -->
# Model and quant on a `<GPU class>`: the headline result in a few words

Copy this file, fill in every line, and replace the guidance wording with real values before publishing.

Date: `<day month year>` (or a range, such as `<14 to 29 Oct 2026>`)

**In short:** Two or three sentences. The speed, the grade totals and the one thing a reader most needs to know (a limit, a surprise, a better alternative).

## Hardware class
- CPU: model and generation, cores and threads
- GPU model and VRAM:
- RAM: size, type and speed
- OS family:
- GPU driver: version and package date

## Setup
- Runner and version: name, build number, and where the build came from. For a fork, say so and give the fork's own version output.
- Harness (direct API or agent): say which, and whether a system prompt, tools or skills were loaded
- Model and quant: model, quant name and file name
  - Quant repo: link to the page the file came from
  - Model license: the license named on the model card, with a link and the date it was checked
  - Attribution requested by the maker: only if the model repo asks for it
- Context size:
- Thinking mode: off, or the effort level
- Settings used:

| Setting | Value |
|---|---|
| GPU layers | |
| KV cache | |
| Flash attention | |
| Parallel slots | |
| Sampling | temperature, top-p, top-k, min-p, presence penalty; say which model-card row it follows and where it differs |

- Runs per test cell:

## Test description
What was sent, how it was graded, and what counts as pass, partial and fail. Link to the appendix if the prompts are published.

## Measured results
- Generation (tokens per second): say whether it is the server-reported rate or the rate seen by the client
- Prompt processing (tokens per second):
- Time to first output (seconds):
- GPU memory in use:
- RAM use:

Write "not recorded" for anything that was not measured. Do not estimate.

### Variants tried
One row per change. Change one thing at a time, or say that several changed together.

| Variant | What changed | Decode (tok/s) | Outcome |
|---|---|---|---|

### Quality results
One small table per group of cells, with the grade for each cell (pass, partial, fail).

| Cell | Result |
|---|---|

**Cells that were not a pass**

One short line per cell, in plain words: what the model did and why it was graded that way.

## Verdict under these settings
Usable: yes / yes, with limits / no

One short paragraph on why, and what to pick instead if there is a better option.

## Caveats
- Anything that limits how far the result carries (a setting that differs from the model card, a different build for one test, a metric that was not recorded).
- One machine and one run per cell unless stated. A single grade difference is a data point, not an effect.
- Hardware, driver and runner build are the ones listed above; results on other cards or builds are not directly comparable.

## How to reproduce
1. Get the runner build and the model file (links above).
2. Start the server. Use placeholders for paths and the port, such as `<port>`, `<runner-dir>` and `<models-dir>`.
3. Check the server before trusting any result: the model name in each reply matches the alias, `finish_reason` is `stop`, and nothing was served from a cache on the first request.
4. Send each test as a separate request.

`Status: Tested on <build>, <date>`
(or) `Status: Draft`
