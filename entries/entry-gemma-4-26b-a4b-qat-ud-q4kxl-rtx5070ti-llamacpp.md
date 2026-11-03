<!-- Visibility: public -->
# Gemma 4 26B-A4B QAT UD-Q4_K_XL on a 16 GB GPU: 45 tokens per second with part of the experts in system RAM

Date: 1 Nov 2026

**In short:** A 26B mixture-of-experts model (about 4B parameters active per token) whose 14.2 GB file does not fully fit a 16 GB card at 64K context: 13 of its 31 layers keep part of their expert weights in system RAM, and decode is 45 tokens per second, about half of the 12B sibling's 84. Quality is level with the 12B: 18 of 22 text cells pass, the 59K needle pack passes 5 of 5, vision passes 6 of 7, and the hard PII pack grades 25 pass and 5 partial (12B: 22 and 8). The MTP speed head adds 33% on copy-style output and costs about 25% on free text.

## Hardware class
- CPU: Intel Core i7-7700 (7th generation desktop, 4 cores, 8 threads)
- GPU model and VRAM: NVIDIA GeForce RTX 5070 Ti, 16 GB
- RAM: 64 GB DDR4, running at about 2400 MT/s, dual channel
- OS family: Windows 10 (LTSC)
- GPU driver: NVIDIA 610.88 (driver package dated 21 Jul 2026)

## Setup
- Runner and version: llama.cpp `llama-server`, build b9682, Windows CUDA 12.4 binary ([llama.cpp](https://github.com/ggml-org/llama.cpp))
- Harness (direct API or agent): direct API (`/v1/chat/completions`), one fresh request per test, no system prompt, no agent, no skills loaded
- Model and quant: Gemma 4 26B-A4B instruction-tuned, quantization-aware-trained (QAT), Unsloth dynamic `UD-Q4_K_XL` GGUF (file `gemma-4-26B-A4B-it-qat-UD-Q4_K_XL.gguf`, 14.2 GB)
  - Quant repo: [unsloth/gemma-4-26B-A4B-it-qat-GGUF](https://huggingface.co/unsloth/gemma-4-26B-A4B-it-qat-GGUF)
  - Vision tests used the F16 projector `mmproj-F16-gemma-4-26B-A4B-it.gguf`
  - The MTP variant used the drafter file `mtp-gemma-4-26B-A4B-it.gguf` (252 MB)
  - Model license: Apache-2.0, from the license field of the [unsloth/gemma-4-26B-A4B-it-qat-GGUF](https://huggingface.co/unsloth/gemma-4-26B-A4B-it-qat-GGUF) card; checked 2026-11-02. The card's license link pointed to a Google page that showed plain Apache 2.0 text on 2026-11-02.
- Context size: 65536
- Thinking mode: off for the main tests; on (card sampling) for the PII tests
- Settings used:

| Setting | Value |
|---|---|
| GPU layers | all 31 on the card, 13 of them with part of their expert weights in system RAM (`--fit on --fit-target 4096`); 16 layers with the vision projector |
| KV cache | f16 (default) |
| Flash attention | auto |
| Memory mapping | off (`--no-mmap`) |
| Threads | 8 (`--threads 8 --threads-batch 8`) |
| Parallel slots | 1 |
| Chat template | `--jinja` |
| Sampling, thinking off | temp 0.7, top-p 0.8, top-k 20, min-p 0, presence penalty 0 |
| Sampling, thinking on | temp 1.0, top-p 0.95, top-k 64 (model card values) |

- Runs per test cell: 1 for the main tests; 5 repeats per cell for the PII tests

## Test description
- Speed: a short prompt asking for a long answer (T6); generation speed is the server-reported decode rate.
- Text battery: 22 graded cells: ten general cells (T1 to T10: exact JSON, a 20K-token context stress test, tool calls, a two-turn code task, memory, two PII redaction prompts), five long-context needle cells (L1 to L5, about 22K tokens) and seven structured-output cells (J1 to J7).
- T9 and T10 ask the model to "invoke" or "run" a `pii-redactor` skill on a short fictional text. No such skill, tool or system prompt exists in the request: this is a deliberate trap, and a good answer redacts the text anyway. See [assets/pii-pack.md](../assets/pii-pack.md).
- Grading uses a written key. A JSON answer wrapped in a code fence counts as a fail where bare JSON was asked for (T1, T3, T4, J4).

## Measured results
- Generation: 45.1 tok/s (server-reported; 40.3 measured at the client), against 84.4 for the 12B; 46.6 to 48.8 at about 22K tokens and 36.5 to 46.0 at about 58.6K tokens
- 20,474-token prompt answered in 21.8 s (12B: 5.8 s)
- Placement: 8,942 MiB of weights on the card and 5,027 MiB of expert weights in a system RAM buffer; the vision projector adds 1,285 MiB to the fit and pushes the count of part-offloaded layers from 13 to 16
- GPU memory in use after load (board readings): 12,389 MiB with the vision projector; 12,977 MiB with the MTP drafter, which is 380 MiB more than without it
- Vision requests decoded at 28 to 41 tok/s (median 38.2); the 12B decodes 82 to 85
- Thermals: the CPU ran at 93 to 100 C on sustained decode and was flagged as throttling in 64 of 96 samples over the hard PII pack, with no visible loss in decode speed
- Time to first token and system RAM use beyond the expert buffer: not recorded

### Variants tried (one change at a time unless noted)

| Variant | What changed | Decode (tok/s) | Outcome |
|---|---|---|---|
| MTP speed head | `--model-draft` with the MTP drafter, `--spec-type draft-mtp`, draft length 4 | 61.7 on the hard PII pack; 32.6 to 35.2 on the long answer | +33% on copy-style output (86% of drafted tokens accepted), about 25% slower on free text (45 to 48% accepted); grades unchanged within noise (26 pass, 4 partial) |
| Thinking off, card sampling | sampling only (temp 1.0, top-p 0.95, top-k 64) | 42.0 (hard PII pack) | 23 pass, 7 partial; the same verdicts as the 0.7 run within noise; the CPU was throttling in that window |
| Thinking on | `--reasoning on` with card sampling, 8,192-token answer limit | 47.4 (hard PII pack) | 21 pass, 6 partial, 3 capped; about 18 times the wall-clock (55.2 minutes against 3) |

The draft length was 4 only; the default of 3 was not compared. MTP with thinking on was not tried.

### Quality results (graded by a written key)

Text battery, long context and structured output (main run): 18 of 22 cells pass, 2 partial, 2 fail.

**General cells (T1 to T10)**

| Cell | Result |
|---|---|
| T1 Sanity, exact JSON only | pass |
| T2 20K-token context stress | pass |
| T3 Tool call | pass |
| T4 Multi-step tool reasoning | pass |
| T5 No tool needed | pass |
| T6 Throughput (long answer) | pass |
| T7 Two-turn code task | partial |
| T8 Memory within a conversation | pass |
| T9 PII redaction, "invoke the skill" wording (trap) | pass |
| T10 PII redaction, "run the skill" wording (trap) | pass |

**Long context, about 22K tokens**

| Cell | Result |
|---|---|
| L1 Needle early | pass |
| L2 Needle middle | pass |
| L3 Needle late | partial |
| L4 Two needles, order | pass |
| L5 Buried instruction | fail |

**Long context, about 59K tokens**

| Cell | Result |
|---|---|
| L1 Needle early | pass |
| L2 Needle middle | pass |
| L3 Needle late | pass |
| L4 Two needles, order | pass |
| L5 Buried instruction | pass |

**Structured output (J1 to J7)**

| Cell | Result |
|---|---|
| J1 Schema extraction | pass |
| J2 Enum trap | pass |
| J3 Nested array and total | pass |
| J4 Prompt-only JSON (no response format) | fail |
| J5 Null not invented | pass |
| J6 Native tool call | pass |
| J7 Number words | pass |

**Vision (V1 to V7)**

| Cell | Result |
|---|---|
| V1 Scene and text | pass |
| V2 Chart to table | pass |
| V3 Counting shapes | partial |
| V4 Terminal screenshot | pass |
| V5 Invoice | pass |
| V6 No-image control | pass |
| V7 Two images | pass |

**Cells that were not a pass**

- T7 (partial): Executed: .json, .yaml, .yml and .YAML parse; malformed JSON and a missing file handled. Top-level import yaml: the whole function fails without PyYAML, JSON included.
- about 22K tokens: L3 (partial): Code word and entry 468 right, but said middle_third instead of last_third. Partial. The same model finds all five at 60K, so a one-run miss at 22K is probably noise.
- about 22K tokens: L5 (fail): it summarised the document and did not end with BANANA. Same note as L3 (60K passes). About 20 s per request, decode 46.6 to 48.8 tok/s (12B about 6.7 s).
- J4 (fail): No response_format: values right but wrapped in a ```json fence. Fail by the fence rule (known flip cell; the 12B failed it too).
- V3 (partial): red circles 4 (true 5) and blue squares 3 (true 4); the other three counts right. Two off by one: partial.

### PII side test (fictional data)

| Mode | Redaction requests (pass / partial / fail) | Tool-control requests correct |
|---|---|---|
| Thinking off | 30 requests: 30 / 0 / 0 | 20 of 20 |
| Thinking on (card sampling) | 30 requests: 30 / 0 / 0 | 20 of 20 |

### Harder PII pack (two longer fictional texts, 30 requests per run)

| Mode | Pass / partial / fail / capped | Median decode (tok/s) | Minutes for 30 requests | Median tokens per answer |
|---|---|---|---|---|
| Thinking off | 25 / 5 / 0 / 0 | 46.4 | 3 | 263 |
| Thinking off, card sampling | 23 / 7 / 0 / 0 | 42 | 3.2 | 264 |
| Thinking on, 8,192-token limit | 21 / 6 / 0 / 3 | 47.4 | 55.2 | 4823 |
| Thinking off, MTP speed head | 26 / 4 / 0 / 0 | 61.7 | 2.3 | 267 |

Capped means the answer hit the 8,192-token limit set for the run and produced no answer; it is reported separately and not counted as a redaction failure. Grading rules are in [assets/pii-pack.md](../assets/pii-pack.md).

The strict explicit wording on the first long text leaves a private-range router address in plaintext in 5 of 5 thinking-off requests, as it does on the 12B in every thinking-off run. Thinking removes it (4 of 5 pass, the fifth hit the token limit) but adds over-redaction of prices, company names and timestamps and 3 token-limit cut-offs. There were no invalid tool-call answers (the 12B had 2 with thinking on). A 24,576-token limit was not run.

## Verdict under these settings
Usable: yes, but the 12B grades about the same (19 against 18 of 22) at about twice the speed

Quality is level with the 12B (battery 18 of 22 against 19 of 22, hard PII 25/5/0 against 22/8/0), at about half the speed because part of the experts sit in system RAM. On this card the 12B is the better default; choose the 26B-A4B if you specifically want the larger model. Use the MTP head only for copy-style work such as redaction.

## Caveats
- Part of the experts live in system RAM, so speed depends on the RAM (here about 2400 MT/s, dual channel) and the CPU as well as the GPU. The CPU ran hot and throttled with no visible loss in decode speed; a different machine could differ in either direction.
- L3 and L5 missed at about 22K tokens but the 59K pack passed 5 of 5, so those two misses are probably noise. J4 was fenced JSON. T7 is a partial for the same unguarded top-level import the 12B has.
- The F16 projector was used; the BF16 projector and a higher image token budget were not tested.
- Two checks were not saved (the server properties dump for the battery run and a baseline memory reading for the vision run); neither changes a grade.
- One machine and one run per cell unless stated. In an earlier measurement on a different model (Qwen3.5-9B), about 2 of 23 grades changed between identical reruns, so a single grade difference is a data point, not an effect.
- Hardware, driver and runner build are the ones listed above; results on other cards or builds are not directly comparable.

## How to reproduce
1. Get a llama.cpp build from the repo linked above (this test used b9682) and the `UD-Q4_K_XL` file from the quant repo.
2. Start the server (replace the paths and pick any free port):

```
llama-server --model <path-to>/gemma-4-26B-A4B-it-qat-UD-Q4_K_XL.gguf --alias gemma-4-26b-a4b-nothink --jinja --reasoning off --temp 0.7 --top-p 0.8 --top-k 20 --min-p 0.00 --flash-attn auto --fit on --fit-target 4096 --no-mmap --threads 8 --threads-batch 8 --ctx-size 65536 --parallel 1 --port <port> --host 127.0.0.1

# MTP variant: add these flags
--model-draft <path-to>/mtp-gemma-4-26B-A4B-it.gguf --spec-type draft-mtp --spec-draft-n-max 4
```

3. Check the server before trusting any result: the model name echoed in each reply matches the alias, `finish_reason` is `stop`, and `cache_n` is 0 on the first request of a fresh server.
4. Send each test as a separate request with no system prompt. The test prompts for the main battery are not published yet; the PII prompts are in the [appendix](../assets/pii-pack.md).

`Status: Tested on llama.cpp b9682, 1 Nov 2026`
