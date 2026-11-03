<!-- Visibility: public -->
# Gemma 4 12B QAT UD-Q4_K_XL on a 16 GB GPU: 84 tokens per second with a 64K context

Date: 22 to 31 Oct 2026

**In short:** With every layer on the GPU and the default f16 cache, this 12B model decodes at 84 tokens per second at 64K context and passes 19 of 22 text cells (partials on the code task and the enum cell, and a fenced JSON cell that fails), the 59K needle pack 5 of 5, and 5 to 6 of 7 vision cells. Thinking off is the mode to use: for PII redaction, thinking on gave no gain, added two new failure modes and took about 15 times longer.

## Hardware class
- CPU: Intel Core i7-7700 (7th generation desktop, 4 cores, 8 threads)
- GPU model and VRAM: NVIDIA GeForce RTX 5070 Ti, 16 GB
- RAM: 64 GB DDR4, running at about 2400 MT/s, dual channel
- OS family: Windows 10 (LTSC)
- GPU driver: NVIDIA 610.88 (driver package dated 21 Jul 2026)

## Setup
- Runner and version: llama.cpp `llama-server`, build b9682, Windows CUDA 12.4 binary ([llama.cpp](https://github.com/ggml-org/llama.cpp))
- Harness (direct API or agent): direct API (`/v1/chat/completions`), one fresh request per test, no system prompt, no agent, no skills loaded
- Model and quant: Gemma 4 12B instruction-tuned, quantization-aware-trained (QAT), Unsloth dynamic `UD-Q4_K_XL` GGUF (file `gemma-4-12B-it-qat-UD-Q4_K_XL.gguf`, 6.7 GB)
  - Quant repo: [unsloth/gemma-4-12B-it-qat-GGUF](https://huggingface.co/unsloth/gemma-4-12B-it-qat-GGUF)
  - Vision tests used the F16 projector (175 MB) from the same quant repo
  - Model license: Apache-2.0, from the license field of the [unsloth/gemma-4-12B-it-qat-GGUF](https://huggingface.co/unsloth/gemma-4-12B-it-qat-GGUF) card; checked 2026-11-02. The card's license link pointed to a Google page that showed plain Apache 2.0 text on 2026-11-02.
- Context size: 65536
- Thinking mode: off for the main tests; on (card sampling) for the PII tests
- Settings used:

| Setting | Value |
|---|---|
| GPU layers | all (`--fit on --fit-target 4096`; the full-pass rerun used `--fit off -ngl 99`, same result) |
| KV cache | f16 (default) |
| Flash attention | auto |
| Memory mapping | off (`--no-mmap`) |
| Threads | 8 (`--threads 8 --threads-batch 8`) |
| Parallel slots | 1 |
| Chat template | `--jinja` |
| Sampling, thinking off | temp 0.7, top-p 0.8, top-k 20, min-p 0, presence penalty 0 |
| Sampling, thinking on | temp 1.0, top-p 0.95, top-k 64 (model card values) |

- Runs per test cell: 1 for the main tests (the vision pack twice); 5 repeats per cell for the PII tests

## Test description
- Speed: a short prompt asking for a long answer (T6); generation speed is the server-reported decode rate.
- Text battery: 22 graded cells: ten general cells (T1 to T10: exact JSON, a 20K-token context stress test, tool calls, a two-turn code task, memory, two PII redaction prompts), five long-context needle cells (L1 to L5, about 22K tokens) and seven structured-output cells (J1 to J7).
- T9 and T10 ask the model to "invoke" or "run" a `pii-redactor` skill on a short fictional text. No such skill, tool or system prompt exists in the request: this is a deliberate trap, and a good answer redacts the text anyway. See [assets/pii-pack.md](../assets/pii-pack.md).
- Grading uses a written key. A JSON answer wrapped in a code fence counts as a fail where bare JSON was asked for (T1, T3, T4, J4).

## Measured results
- Generation: 84.4 tok/s (server-reported; 53.2 measured at the client); 76.7 with a q8_0 cache; 79.7 to 85.1 on vision requests
- 20,474-token prompt answered in 5.8 s (8.1 s in the q8_0-cache run); the 22K needle cells took about 6.7 s each
- Long context: the 59K-token pack passed 5 of 5, about 21.5 s per request, decode 72.7 to 75.7 tok/s
- GPU memory in use after load: 10,653 MiB (default cache); 10,640 MiB with the vision projector; 9,639 MiB in the q8_0-cache variant
- Thermals (full pass, hardware monitor): CPU package 60.9 C average and 81 C max; GPU power 150 W average; throttling flagged in 0 of 54 samples
- Time to first token and RAM use: not recorded

### Variants tried (one change at a time unless noted)

| Variant | What changed | Decode (tok/s) | Outcome |
|---|---|---|---|
| q8_0 cache, fit-target 2048 | K/V cache f16 to q8_0 and fit-target 4096 to 2048 | 76.7 | 21 of 22 cells pass as first graded (its code-task grade was not revisited and has the same import flaw as the default run); fully on the GPU at 64K |
| Thinking off, card sampling | sampling only (temp 1.0, top-p 0.95, top-k 64) | 83.9 (hard PII pack) | Same verdicts as thinking off at 0.7, so sampling is not what separates the runs |
| Thinking on | `--reasoning on` with card sampling | 81.9 (hard PII pack) | No net gain; two new failure modes; about 15 times slower |
| Thinking on, 24,576-token limit | answer limit raised from 8,192 | 82.1 (hard PII pack) | Removes the two cut-offs, not the invalid tool-call answer |

### Quality results (graded by a written key)

Text battery, long context and structured output (main run): 19 of 22 cells pass, 2 partial, 1 fail.

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
| L3 Needle late | pass |
| L4 Two needles, order | pass |
| L5 Buried instruction | pass |

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
| J2 Enum trap | partial |
| J3 Nested array and total | pass |
| J4 Prompt-only JSON (no response format) | fail |
| J5 Null not invented | pass |
| J6 Native tool call | pass |
| J7 Number words | pass |

**Vision (V1 to V7)**

| Cell | Result |
|---|---|
| V1 Scene and text | partial |
| V2 Chart to table | pass |
| V3 Counting shapes | partial |
| V4 Terminal screenshot | pass |
| V5 Invoice | pass |
| V6 No-image control | pass |
| V7 Two images | pass |

**Cells that were not a pass**

- T7 (partial): Executed: .json, .yaml, .yml parse; unsupported extension, non-dict root, malformed JSON and missing file handled. Top-level import yaml (whole function fails without PyYAML).
- J2 (partial): low (valid JSON, one value wrong: the urgent production outage should be high, enum trap; the q8_0 KV run said high; J2 is a known noise cell). Partial by the key (valid JSON, one value wrong).
- J4 (fail): No response_format: values right but wrapped in a ```json fence. Fail by the fence rule (known flip cell).
- V1 (partial): Strings exact (Est. 1988; RIVERBEND FARM) and the scene is right, but it invents a thin black vertical bar down the left edge that is not in the image (checked against the image). Partial.
- V3 (partial): red 5, blue circles 4 (true 3), blue squares 4, green triangles 2, yellow stars 1: one count off by one (fenced JSON). Partial.

### PII side test (fictional data)

| Mode | Redaction requests (pass / partial / fail) | Tool-control requests correct |
|---|---|---|
| Thinking off | 30 requests: 30 / 0 / 0 | 20 of 20 |
| Thinking on (card sampling) | 30 requests: 30 / 0 / 0 | 20 of 20 |

### Harder PII pack (two longer fictional texts, 30 requests per run)

| Mode | Pass / partial / fail / capped | Median decode (tok/s) | Minutes for 30 requests | Median tokens per answer |
|---|---|---|---|---|
| Thinking off | 22 / 8 / 0 / 0 | 84.1 | 1.7 | 265 |
| Thinking off, card sampling | 22 / 8 / 0 / 0 | 83.9 | 1.6 | 263 |
| Thinking on, 8,192-token limit | 21 / 5 / 2 / 2 | 81.9 | 24.9 | 3104 |
| Thinking on, 24,576-token limit | 24 / 4 / 2 / 0 | 82.1 | 24.5 | 3331 |

Capped means the answer hit the 8,192-token limit set for the run and produced no answer; it is reported separately and not counted as a redaction failure. Grading rules are in [assets/pii-pack.md](../assets/pii-pack.md).

With thinking off, the strict explicit wording on the first long text left a private-range router address in plaintext in 5 of 5 requests; thinking on redacted it in the 4 of 4 requests that finished. A follow-up wording that names private addresses fixed it in 10 of 10 thinking-off requests. Over-redaction of order numbers and dates was common with thinking on.

## Verdict under these settings
Usable: yes

Fast, fully GPU-resident and reliable on the text, long-context and structured-output cells. Use thinking off; for PII redaction use wording that names private addresses.

## Caveats
- The default-cache run's code task (T7) was regraded from pass to partial after a second look (an unguarded top-level import makes the whole function fail without PyYAML); the q8_0-cache run has the same flaw but was not regraded. The two runs also differ in J2, a known noisy cell (partial in the default run, pass in the q8_0 run); J4 was fenced in both. Treat the two recipes as the same quality.
- Vision counting (V3, blue circles) was off by one in both vision runs; V1 flipped between runs.
- Thinking and sampling changed together against the thinking-off runs, so 'thinking on versus off' here means the model card setup versus our recipe.
- The load log of this llama.cpp build does not print offload lines at default verbosity, so full GPU residency is inferred from speed and memory use.
- One machine and one run per cell unless stated. In an earlier measurement on a different model (Qwen3.5-9B), about 2 of 23 grades changed between identical reruns, so a single grade difference is a data point, not an effect.
- Hardware, driver and runner build are the ones listed above; results on other cards or builds are not directly comparable.

## How to reproduce
1. Get a llama.cpp build from the repo linked above (this test used b9682) and the `UD-Q4_K_XL` file from the quant repo.
2. Start the server (replace the paths and pick any free port):

```
llama-server --model <path-to>/gemma-4-12B-it-qat-UD-Q4_K_XL.gguf --alias gemma-4-12b-std --jinja --reasoning off --temp 0.7 --top-p 0.8 --top-k 20 --min-p 0.00 --flash-attn auto --fit on --fit-target 4096 --no-mmap --threads 8 --threads-batch 8 --ctx-size 65536 --parallel 1 --port <port> --host 127.0.0.1
```

3. Check the server before trusting any result: the model name echoed in each reply matches the alias, `finish_reason` is `stop`, and `cache_n` is 0 on the first request of a fresh server.
4. Send each test as a separate request with no system prompt. The test prompts for the main battery are not published yet; the PII prompts are in the [appendix](../assets/pii-pack.md).

`Status: Tested on llama.cpp b9682, 31 Oct 2026`
