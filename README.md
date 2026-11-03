<!-- Visibility: public -->
# local-llm-lab

Results from running local language models on one consumer machine with llama.cpp. Each entry records the model file, the runner build, the exact settings and what was measured, so you can reproduce it or see why your numbers differ. These are single-machine results, not standardized benchmarks.

## Who it helps

People deciding whether a model, quant and runner will be usable on a machine like theirs, without running every test themselves.

## What these results are

Unless an entry says otherwise, tests use a direct API call with thinking off and no agent or skills. That is a controlled baseline, not how most people run these models day to day. Each entry states its harness and thinking mode.

Agent runs with skills are not covered yet. They can behave very differently from the baseline, so do not read a baseline grade as a promise for an agent setup.

## How to read an entry

Each entry is one Markdown file in `entries/`, written from `TEMPLATE.md`. An entry covers one hardware class, one runner and version, one harness, and one model and quant.

- The title gives the headline. The "In short" paragraph gives the result in plain words.
- Quality cells are graded by a written key as pass, partial or fail. Cells that were not a pass are explained in words.
- "Verdict under these settings" is a judgment for this machine and these settings only.
- Every entry ends with a status stamp: `Tested on <build>, <date>` or `Draft`.
- A sample of graded cells in every entry was regraded blind from the raw responses before publishing. This checks the grading; it is not a full regrade.
- Prompts and grading rules that are published live in `assets/`.

Before publishing, each entry is checked against `CHECKLIST.md`.

## Entries

| Entry | Context | Decode (tok/s) | Text, 22 cells (pass / partial / fail) | Vision, 7 cells (same order) | Usable |
|---|---|---|---|---|---|
| [Qwen3.8-27B UD-Q2_K_XL](entries/entry-qwen38-27b-ud-q2kxl-rtx5070ti-llamacpp.md) | 64K | 54.4 | 22 / 0 / 0 | 7 / 0 / 0 | yes |
| [Qwen3.8-27B UD-IQ4_XS](entries/entry-qwen38-27b-ud-iq4xs-rtx5070ti-llamacpp.md) | 64K | 2.3 | 22 / 0 / 0 | not run | no (too slow at 64K on 16 GB) |
| [Qwen3.8-27B GSQ-RCO IQ3_XXS](entries/entry-qwen38-27b-gsq-rco-iq3xxs-rtx5070ti-llamacpp.md) | 64K | 52.7 | 20 / 1 / 1 | 7 / 0 / 0 | yes for text; vision slow |
| [Qwen3.8-27B GSQ-RCO IQ2_S](entries/entry-qwen38-27b-gsq-rco-iq2s-rtx5070ti-llamacpp.md) | 64K | 55.1 | 20 / 2 / 0 | 5 / 1 / 1 | yes |
| [Ternary Bonsai 2 27B PTQ1_0](entries/entry-ternary-bonsai-2-27b-ptq1-0-rtx5070ti-llamacpp-fork.md) | 64K | 81.3 | 19 / 2 / 1 | 4 / 2 / 1 | yes, with limits |
| [Gemma 4 12B QAT UD-Q4_K_XL](entries/entry-gemma-4-12b-qat-ud-q4kxl-rtx5070ti-llamacpp.md) | 64K | 84.4 | 19 / 2 / 1 | 5 / 2 / 0 | yes |
| [Gemma 4 26B-A4B QAT UD-Q4_K_XL](entries/entry-gemma-4-26b-a4b-qat-ud-q4kxl-rtx5070ti-llamacpp.md) | 64K | 45.1 | 18 / 2 / 2 | 6 / 1 / 0 | yes, but the 12B is as good and twice as fast |
| [Qwen3.5-9B Q4_K_M](entries/entry-qwen35-9b-q4km-unsloth-rtx5070ti-llamacpp.md) | 64K | 110.2 | 16 / 4 / 2 | 5 / 2 / 0 | yes, with limits |
| [Qwen3-8B Q4_K_M](entries/entry-qwen3-8b-q4km-bartowski-rtx5070ti-llamacpp.md) | 32K | 126.9 | 17 / 3 / 2 | not run | yes at 32K |
| [Qwen3-8B UD-Q4_K_XL](entries/entry-qwen3-8b-ud-q4kxl-unsloth-rtx5070ti-llamacpp.md) | 32K | 124.4 | 17 / 3 / 2 | not run | yes at 32K |
| [Qwen3-VL-8B-Instruct UD-Q4_K_XL](entries/entry-qwen3-vl-8b-instruct-ud-q4kxl-rtx5070ti-llamacpp.md) | 64K | 11.8 (123.1 at 32K) | 17 / 4 / 1 | 6 / 0 / 1 | yes at 32K; slow at 64K |
| [Ornith-1.0-9B UD-Q4_K_XL](entries/entry-ornith-1-0-9b-ud-q4kxl-rtx5070ti-llamacpp.md) | 64K | 105.2 | 19 / 1 / 2 | 5 / 1 / 1 | yes, with limits |
| [Ornith-1.0-9B Q8_0](entries/entry-ornith-1-0-9b-q8-0-rtx5070ti-llamacpp.md) | 64K | 77.6 | 16 / 4 / 2 | 6 / 1 / 0 | yes, but UD-Q4_K_XL is faster and better on text |

Context and GPU offload differ between rows, so speeds are not like-for-like. One-cell differences are noise. Bonsai vision ran on mainline llama.cpp b9682, not the fork build.

Appendix: [PII redaction side test: wordings, reference answers and full results](assets/pii-pack.md)

## Sanitized entries only

Every entry is sanitized and contains no personal identifiers. Hardware is described by its parts (CPU, GPU, RAM, driver, OS family), never by machine name.

## Licenses and credit

- The text and tables in this repo are under the MIT license (see `LICENSE`).
- Every model keeps its own license. Each entry names the license shown on the model card, links it, and gives the date it was checked. Check it again before you reuse a model.
- This repo is not affiliated with the model authors, quantizers or runner projects named in the entries. Where a model's maker asks for attribution, the entry includes it.

## No warranty

One machine, one date, one run per cell unless an entry says more. Numbers on your hardware, driver or build will differ. Treat every result as a data point, not a guarantee.
