<!-- Visibility: public -->
# Qwen3.8-27B GSQ-RCO IQ2_S on a 16 GB GPU: 55 tokens per second, fully on the GPU

Date: 28 Oct 2026

**In short:** This 2-bit-class file (9.3 GB, from ISTA-DASLab) runs entirely on the GPU at 64K context and decodes at 55.1 tokens per second, level with UD-Q2_K_XL. 20 of 22 text cells pass (partials on the code task and the skill-wording PII prompt) and the 59K needle pack passes 5 of 5. The vision pack has one weak cell, the no-image control.

## Hardware class
- CPU: Intel Core i7-7700 (7th generation desktop, 4 cores, 8 threads)
- GPU model and VRAM: NVIDIA GeForce RTX 5070 Ti, 16 GB
- RAM: 64 GB DDR4, running at about 2400 MT/s, dual channel
- OS family: Windows 10 (LTSC)
- GPU driver: NVIDIA 610.88 (driver package dated 21 Jul 2026)

## Setup
- Runner and version: llama.cpp `llama-server`, build b9682, Windows CUDA 12.4 binary ([llama.cpp](https://github.com/ggml-org/llama.cpp))
- Harness (direct API or agent): direct API (`/v1/chat/completions`), one fresh request per test, no system prompt, no agent, no skills loaded
- Model and quant: Qwen3.8-27B, ISTA-DASLab `GSQ-RCO` `IQ2_S` GGUF (file `Qwen3.8-27B-GSQ-RCO-IQ2_S.gguf`, 9.26 GB)
  - Quant repo: [ISTA-DASLab/Qwen3.8-27B-GSQ-RCO-GGUF](https://huggingface.co/ISTA-DASLab/Qwen3.8-27B-GSQ-RCO-GGUF)
  - Base model card: [Qwen/Qwen3.8-27B](https://huggingface.co/Qwen/Qwen3.8-27B)
  - Vision tests also used the `mmproj-Qwen3.8-27B-BF16` projector from the same quant repo
  - Model license: Apache-2.0, from the license field of the [ISTA-DASLab/Qwen3.8-27B-GSQ-RCO-GGUF](https://huggingface.co/ISTA-DASLab/Qwen3.8-27B-GSQ-RCO-GGUF) and [Qwen/Qwen3.8-27B](https://huggingface.co/Qwen/Qwen3.8-27B) cards; checked 2026-11-02.
- Context size: 65536
- Thinking mode: off for the main tests; on (effort xhigh and medium) for the PII tests only
- Settings used:

| Setting | Value |
|---|---|
| GPU layers | all (`--fit on --fit-target 2048`) |
| KV cache | q8_0 for K and V |
| Flash attention | auto |
| Memory mapping | off (`--no-mmap`) |
| Threads | 8 (`--threads 8 --threads-batch 8`) |
| Parallel slots | 1 |
| Chat template | `--jinja` |
| Sampling, thinking off | temp 0.7, top-p 0.8, top-k 20, min-p 0, presence penalty 0 (not set; server default). The Instruct row of the [Unsloth Qwen3.8-27B page](https://unsloth.ai/docs/models/qwen3.8), except presence penalty, which that row lists as 1.5 |
| Sampling, thinking on (PII tests only) | temp 1.0, top-p 0.95, top-k 20, min-p 0, presence penalty 0 (the Thinking row of the Unsloth page) |

- Runs per test cell: 1 for the main tests; 5 repeats per cell for the PII tests

## Test description
- Speed: a short prompt asking for a long answer (T6); generation speed is the server-reported decode rate.
- Text battery: 22 graded cells: ten general cells (T1 to T10: exact JSON, a 20K-token context stress test, tool calls, a two-turn code task, memory, two PII redaction prompts), five long-context needle cells (L1 to L5, about 22K tokens) and seven structured-output cells (J1 to J7).
- T9 and T10 ask the model to "invoke" or "run" a `pii-redactor` skill on a short fictional text. No such skill, tool or system prompt exists in the request: this is a deliberate trap, and a good answer redacts the text anyway. See [assets/pii-pack.md](../assets/pii-pack.md).
- Grading uses a written key. A JSON answer wrapped in a code fence counts as a fail where bare JSON was asked for (T1, T3, T4, J4).

## Measured results
- Generation: 55.1 tok/s (short prompt, 431 tokens, server-reported; 39.5 measured at the client)
- 20,471-token prompt answered in 18.0 s (UD-Q2_K_XL 13.9 s on the same recipe)
- GPU memory in use after load: 13,736 MiB for text (the desktop's own use was 2,291 MiB before loading)
- Time to first token, RAM use and thermals: not recorded for this run

### Variants tried (one change at a time unless noted)

| Variant | What changed | Decode (tok/s) | Outcome |
|---|---|---|---|
| Vision projector loaded | `--mmproj` added to the text recipe | not recorded here | see the vision table; the no-image control failed in the first run and the repeat |

### Quality results (graded by a written key)

Text battery, long context and structured output (main run): 20 of 22 cells pass, 2 partial, 0 fail.

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
| T9 PII redaction, "invoke the skill" wording (trap) | partial |
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
| J2 Enum trap | pass |
| J3 Nested array and total | pass |
| J4 Prompt-only JSON (no response format) | pass |
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
| V6 No-image control | fail |
| V7 Two images | pass |

**Cells that were not a pass**

- T7 (partial): Executed (PyYAML 6.0.3): turn 2 parses .json, .yaml, .yml and rejects .txt with ValueError. Bugs: top-level import yaml (JSON breaks when PyYAML is missing) and the malformed-JSON re-raise rebuilds JSONDecodeError with the wrong arguments (TypeError).
- T9 (partial): Output is one bare [REDACTED]; sentence dropped, no value left in plaintext. Partial by the grading key (collapsed to bare tags). PII side pack 2026-10-28: the same wording went 5 of 5 pass, so this partial was noise.
- V3 (partial): red 5, blue circles 3, blue squares 3 (true 4), green triangles 2, yellow stars 1: one count off by one, partial.
- V6 (fail): no image was sent and it described a white smartphone held in a hand with invented screen text (Q2_K_XL said no image was attached).

### PII side test (fictional data)

| Mode | Redaction requests (pass / partial / fail) | Tool-control requests correct |
|---|---|---|
| Thinking off | 30 requests: 30 / 0 / 0 | 20 of 20 |
| Thinking on, effort xhigh | 30 requests: 30 / 0 / 0 | 20 of 20 |
| Thinking on, effort medium | 30 requests: 30 / 0 / 0 | 20 of 20 |

### Harder PII pack (two longer fictional texts, 30 requests per run)

| Mode | Pass / partial / fail / capped | Median decode (tok/s) | Minutes for 30 requests | Median tokens per answer |
|---|---|---|---|---|
| Thinking off | 18 / 12 / 0 / 0 | 55.5 | 2.5 | 270 |
| Thinking on, effort xhigh | 20 / 1 / 0 / 9 | 54.3 | 53.1 | 6041 |
| Thinking on, effort medium | 27 / 3 / 0 / 0 | 54.3 | 15.2 | 1555 |

Capped means the answer hit the 8,192-token limit set for the run and produced no answer; it is reported separately and not counted as a redaction failure. Grading rules are in [assets/pii-pack.md](../assets/pii-pack.md).

Full wordings, reference answers and the same tables for the other two quants of this model are in [assets/pii-pack.md](../assets/pii-pack.md).

## Verdict under these settings
Usable: yes

Same speed class as UD-Q2_K_XL (about 55 tok/s) in a smaller file, with all layers on the GPU. Text quality is in line with the larger quants; the no-image vision control is the one cell to watch.

## Caveats
- T9 collapsed to a single bare tag (partial) and T7 had a code-quality partial; both are single-run grades.
- The vision no-image control (V6) failed in both vision runs on this quant while it passed on IQ3_XXS, so treat vision on this file as less reliable.
- This is a 2-bit-class quant; contexts beyond about 59K tokens were not tested.
- Presence penalty was not set: the runs left it at the server default of 0, while the Unsloth page lists 1.5 for the Instruct row (it lists 0 for the Thinking row). The thinking-off results here therefore differ from that row in this one setting.
- One machine and one run per cell unless stated. In an earlier measurement on a different model (Qwen3.5-9B), about 2 of 23 grades changed between identical reruns, so a single grade difference is a data point, not an effect.
- Hardware, driver and runner build are the ones listed above; results on other cards or builds are not directly comparable.

## How to reproduce
1. Get a llama.cpp build from the repo linked above (this test used b9682) and the IQ2_S file from the quant repo.
2. Start the server (replace the paths and pick any free port):

```
llama-server --model <path-to>/Qwen3.8-27B-GSQ-RCO-IQ2_S.gguf --alias qwen38-27b-gsq-iq2s --jinja --reasoning off --temp 0.7 --top-p 0.8 --top-k 20 --min-p 0.00 --flash-attn auto --cache-type-k q8_0 --cache-type-v q8_0 --fit on --fit-target 2048 --no-mmap --threads 8 --threads-batch 8 --ctx-size 65536 --parallel 1 --port <port> --host 127.0.0.1
```

3. Check the server before trusting any result: the model name echoed in each reply matches the alias, `finish_reason` is `stop`, and `cache_n` is 0 on the first request of a fresh server.
4. Send each test as a separate request with no system prompt. The test prompts for the main battery are not published yet; the PII prompts are in the [appendix](../assets/pii-pack.md).

`Status: Tested on llama.cpp b9682, 28 Oct 2026`
