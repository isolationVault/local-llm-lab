<!-- Visibility: public -->
# Qwen3.8-27B UD-IQ4_XS on a 16 GB GPU: every cell passes, but only 2 to 4 tokens per second

Date: 21 to 22 Oct 2026

**In short:** This 4-bit file passed all 22 text cells, but at a 64K context about 30 of its 65 layers run on the CPU and generation is 2.3 tokens per second. The best of three settings tried (smaller fit margin and q8_0 cache) reached 3.9 tok/s, still CPU-bound. On this card the 2- and 3-bit files of the same model are the usable ones.

## Hardware class
- CPU: Intel Core i7-7700 (7th generation desktop, 4 cores, 8 threads)
- GPU model and VRAM: NVIDIA GeForce RTX 5070 Ti, 16 GB
- RAM: 64 GB DDR4, running at about 2400 MT/s, dual channel
- OS family: Windows 10 (LTSC)
- GPU driver: NVIDIA 610.88 (driver package dated 21 Jul 2026)

## Setup
- Runner and version: llama.cpp `llama-server`, build b9682, Windows CUDA 12.4 binary ([llama.cpp](https://github.com/ggml-org/llama.cpp))
- Harness (direct API or agent): direct API (`/v1/chat/completions`), one fresh request per test, no system prompt, no agent, no skills loaded
- Model and quant: Qwen3.8-27B, Unsloth dynamic `UD-IQ4_XS` GGUF (file `Qwen3.8-27B-UD-IQ4_XS.gguf`)
  - Quant repo: [unsloth/Qwen3.8-27B-GGUF](https://huggingface.co/unsloth/Qwen3.8-27B-GGUF)
  - Base model card: [Qwen/Qwen3.8-27B](https://huggingface.co/Qwen/Qwen3.8-27B)
  - Sampling and usage guide: [Unsloth Qwen3.8-27B guide](https://unsloth.ai/docs/models/qwen3.8)
  - Model license: Apache-2.0, from the license field of the [unsloth/Qwen3.8-27B-GGUF](https://huggingface.co/unsloth/Qwen3.8-27B-GGUF) and [Qwen/Qwen3.8-27B](https://huggingface.co/Qwen/Qwen3.8-27B) cards; checked 2026-11-02.
- Context size: 65536
- Thinking mode: off
- Settings used:

| Setting | Value |
|---|---|
| GPU layers | about 35 of 65 on the GPU, about 30 on the CPU (`--fit on --fit-target 4096`) |
| KV cache | f16 (default) |
| Flash attention | auto |
| Memory mapping | off (`--no-mmap`) |
| Threads | 8 (`--threads 8 --threads-batch 8`) |
| Parallel slots | 1 |
| Chat template | `--jinja` |
| Sampling, thinking off | temp 0.7, top-p 0.8, top-k 20, min-p 0, presence penalty 0 (not set; server default). The Instruct row of the [Unsloth Qwen3.8-27B page](https://unsloth.ai/docs/models/qwen3.8), except presence penalty, which that row lists as 1.5 |

- Runs per test cell: 1

## Test description
- Speed: a short prompt asking for a long answer (T6); generation speed is the server-reported decode rate.
- Text battery: 22 graded cells: ten general cells (T1 to T10: exact JSON, a 20K-token context stress test, tool calls, a two-turn code task, memory, two PII redaction prompts), five long-context needle cells (L1 to L5, about 22K tokens) and seven structured-output cells (J1 to J7).
- T9 and T10 ask the model to "invoke" or "run" a `pii-redactor` skill on a short fictional text. No such skill, tool or system prompt exists in the request: this is a deliberate trap, and a good answer redacts the text anyway. See [assets/pii-pack.md](../assets/pii-pack.md).
- Grading uses a written key. A JSON answer wrapped in a code fence counts as a fail where bare JSON was asked for (T1, T3, T4, J4).

## Measured results
- Generation: 2.33 tok/s (server-reported; 2.28 at the client)
- 20,471-token prompt answered cleanly (time not recorded in the summary data)
- GPU memory in use after load: 12,804 MiB at 32768 context and 12,810 MiB with presence penalty 1.5; 14,857 MiB in the q8_0-cache variant
- In the variant runs the CPU sat near 100% while generating and package temperature reached 97 to 98 C, with throttling flagged in 58 of 62 samples in the fit-target 2048 run
- Time to first token and RAM use: not recorded

### Variants tried (one change at a time unless noted)

| Variant | What changed | Decode (tok/s) | Outcome |
|---|---|---|---|
| Context 32768 | `--ctx-size 32768` (default run 65536) | 2.78 | +19%; 24 layers on the CPU instead of 30 |
| Presence penalty 1.5 | `--presence-penalty 1.5` (default 0) | 2.46 | Within run-to-run noise; the penalty does not explain the slow speed |
| Fit-target 2048 and q8_0 cache | `--fit-target 2048` and q8_0 K/V cache | 3.91 | +68%; about 15 layers still on the CPU |

The speed follows the number of layers left on the CPU. These variants are single speed measurements (the long answer only); the quality tables below are the default run.

### Quality results (graded by a written key)

Text battery, long context and structured output (main run): 22 of 22 cells pass, 0 partial, 0 fail.

**General cells (T1 to T10)**

| Cell | Result |
|---|---|
| T1 Sanity, exact JSON only | pass |
| T2 20K-token context stress | pass |
| T3 Tool call | pass |
| T4 Multi-step tool reasoning | pass |
| T5 No tool needed | pass |
| T6 Throughput (long answer) | pass |
| T7 Two-turn code task | pass |
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

### PII

Only the two PII cells in the battery (T9 and T10) were run on this file, with thinking off: both redacted every value and used tags. The PII side packs were not run on it.

## Verdict under these settings
Usable: no, not at 64K context on a 16 GB card (it is slower than reading speed)

Quality is fine, but 2 to 4 tok/s is too slow for chat or agent work. If you want a 4-bit 27B on a 16 GB card, expect to give up context or accept CPU speeds; the 2- and 3-bit quants of the same model run about 23 times faster here.

## Caveats
- Speed is set by how many layers fit on the GPU, so it depends on context size, cache type and what else uses VRAM (the desktop used about 2 GB). Another card or a lower context would change this result completely.
- The three variants were run on the long-answer speed test only, not the full battery.
- Presence penalty was not set in the main run (server default 0), while the Unsloth page lists 1.5 for the Instruct row. The 1.5 variant (variants table) was run on 22 Oct 2026, in the same session as the main run, on the long-answer speed test only: 2.46 against 2.33 tok/s, within run-to-run noise, and no quality cells were graded with it. The earlier September run of this file (2.2 tok/s) also had 1.5 mixed into its settings.
- One machine and one run per cell unless stated. In an earlier measurement on a different model (Qwen3.5-9B), about 2 of 23 grades changed between identical reruns, so a single grade difference is a data point, not an effect.
- Hardware, driver and runner build are the ones listed above; results on other cards or builds are not directly comparable.

## How to reproduce
1. Get a llama.cpp build from the repo linked above (this test used b9682) and the `UD-IQ4_XS` file from the quant repo.
2. Start the server (replace the paths and pick any free port):

```
llama-server --model <path-to>/Qwen3.8-27B-UD-IQ4_XS.gguf --alias qwen38-27b-iq4xs --jinja --reasoning off --temp 0.7 --top-p 0.8 --top-k 20 --min-p 0.00 --flash-attn auto --fit on --fit-target 4096 --no-mmap --threads 8 --threads-batch 8 --ctx-size 65536 --parallel 1 --port <port> --host 127.0.0.1
```

3. Check the server before trusting any result: the model name echoed in each reply matches the alias, `finish_reason` is `stop`, and `cache_n` is 0 on the first request of a fresh server.
4. Send each test as a separate request with no system prompt. The test prompts for the main battery are not published yet; the PII prompts are in the [appendix](../assets/pii-pack.md).

`Status: Tested on llama.cpp b9682, 22 Oct 2026`
