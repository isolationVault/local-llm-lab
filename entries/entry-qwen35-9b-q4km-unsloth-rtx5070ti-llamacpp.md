<!-- Visibility: public -->
# Qwen3.5-9B Q4_K_M on a 16 GB GPU: 110 tokens per second, and how much grades move between identical runs

Date: 17 to 18 Oct 2026

**In short:** A 9B model that decodes at about 110 tokens per second at 64K context with room to spare. Four runs of the same file show how much grades move on their own: a few cells flipped between pass and partial in every pair of runs, so single grade differences between models are not evidence. The second PII prompt (T10) left names in plaintext in three of the four runs.

## Hardware class
- CPU: Intel Core i7-7700 (7th generation desktop, 4 cores, 8 threads)
- GPU model and VRAM: NVIDIA GeForce RTX 5070 Ti, 16 GB
- RAM: 64 GB DDR4, running at about 2400 MT/s, dual channel
- OS family: Windows 10 (LTSC)
- GPU driver: NVIDIA 610.88 (driver package dated 21 Jul 2026)

## Setup
- Runner and version: llama.cpp `llama-server`, build b9682, Windows CUDA 12.4 binary ([llama.cpp](https://github.com/ggml-org/llama.cpp))
- Harness (direct API or agent): direct API (`/v1/chat/completions`), one fresh request per test, no system prompt, no agent, no skills loaded
- Model and quant: Qwen3.5-9B, Unsloth `Q4_K_M` GGUF (file `Qwen3.5-9B-Q4_K_M.gguf`)
  - Quant repo: [unsloth/Qwen3.5-9B-GGUF](https://huggingface.co/unsloth/Qwen3.5-9B-GGUF)
  - Model license: Apache-2.0, from the license field of the [unsloth/Qwen3.5-9B-GGUF](https://huggingface.co/unsloth/Qwen3.5-9B-GGUF) card; checked 2026-11-02.
- Context size: 65536
- Thinking mode: off
- Settings used:

| Setting | Value |
|---|---|
| GPU layers | chosen by `--fit on --fit-target 4096`; no spill seen |
| KV cache | f16 (default) |
| Flash attention | auto |
| Memory mapping | off (`--no-mmap`) |
| Threads | 8 (`--threads 8 --threads-batch 8`) |
| Parallel slots | 1 |
| Chat template | `--jinja` |
| Sampling, thinking off | temp 0.7, top-p 0.8, top-k 20, min-p 0, presence penalty 0 |

- Runs per test cell: 4 full runs of the same file (two default, two with presence penalty 1.5); the tables show the first default run

## Test description
- Speed: a short prompt asking for a long answer (T6); generation speed is the server-reported decode rate.
- Text battery: 22 graded cells: ten general cells (T1 to T10: exact JSON, a 20K-token context stress test, tool calls, a two-turn code task, memory, two PII redaction prompts), five long-context needle cells (L1 to L5, about 22K tokens) and seven structured-output cells (J1 to J7).
- T9 and T10 ask the model to "invoke" or "run" a `pii-redactor` skill on a short fictional text. No such skill, tool or system prompt exists in the request: this is a deliberate trap, and a good answer redacts the text anyway. See [assets/pii-pack.md](../assets/pii-pack.md).
- Grading uses a written key. A JSON answer wrapped in a code fence counts as a fail where bare JSON was asked for (T1, T3, T4, J4).

## Measured results
- Generation: 109.3 and 110.2 tok/s in the two default runs (server-reported; about 94 at the client); 97.9 and 98.2 with presence penalty 1.5
- 20,471-token prompt answered cleanly (time not recorded in the summary data)
- Hardware monitor over the text round: GPU load up to 98%, 273 W, 63 C, 12.2 GB dedicated GPU memory in use (board total), CPU up to 62%
- Time to first token and RAM use: not recorded

### Variants tried (one change at a time unless noted)

| Variant | What changed | Decode (tok/s) | Outcome |
|---|---|---|---|
| Default, run 1 | first run, series sampling | 109.3 | T4, T7, T10 partial |
| Default, run 2 | identical settings, fresh server | 110.2 | T7, T10 partial; T4 passed and J3 flipped to partial; 11 of 23 outputs byte-identical to run 1 |
| Presence penalty 1.5, run 1 | `--presence-penalty 1.5` | 98.2 | T7 partial; T4 and T10 pass |
| Presence penalty 1.5, run 2 | identical to the line above, fresh server | 97.9 | T10 partial |

Across the four runs, every cell that changed grade changed in at least one pair of identical-setting runs, so no grade difference can be credited to the penalty setting. Decode was about 10% lower with the penalty in both runs.

### Quality results (graded by a written key)

Text battery, long context and structured output (main run): 16 of 22 cells pass, 4 partial, 2 fail.

**General cells (T1 to T10)**

| Cell | Result |
|---|---|
| T1 Sanity, exact JSON only | pass |
| T2 20K-token context stress | pass |
| T3 Tool call | pass |
| T4 Multi-step tool reasoning | partial |
| T5 No tool needed | pass |
| T6 Throughput (long answer) | pass |
| T7 Two-turn code task | partial |
| T8 Memory within a conversation | pass |
| T9 PII redaction, "invoke the skill" wording (trap) | pass |
| T10 PII redaction, "run the skill" wording (trap) | partial |

**Long context, about 22K tokens**

| Cell | Result |
|---|---|
| L1 Needle early | pass |
| L2 Needle middle | partial |
| L3 Needle late | pass |
| L4 Two needles, order | pass |
| L5 Buried instruction | fail |

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
| V1 Scene and text | partial |
| V2 Chart to table | pass |
| V3 Counting shapes | partial |
| V4 Terminal screenshot | pass |
| V5 Invoice | pass |
| V6 No-image control | pass |
| V7 Two images | pass |

**Cells that were not a pass**

- T4 (partial): list_files with directory set to src/utils.py (a file passed as a directory); not a read_file call.
- T7 (partial): Static read: top-level import yaml, so plain .json configs would raise ImportError if PyYAML is missing. Code was not executed.
- T10 (partial): Email, phone, SSN, IP tagged, but both names (Jane Smith, Bob Wu) left in plaintext: partial.
- about 22K tokens: L2 (partial): Code word and entry (260) right, position last_third instead of middle_third (the known position-sense weakness).
- about 22K tokens: L5 (fail): it gave a one-sentence summary of the log and did not end with BANANA; the entry-312 instruction was ignored.
- J4 (fail): No response_format: values right but wrapped in a ```json fence. Fail by the pack's checks, as for the Ornith rows.
- V1 (partial): Strings exact (Est. 1988; RIVERBEND; FARM), scene mostly right, but it invented a thick black line along the top edge and described the sign as held by two posts. Partial by the grading key, consistent with the Ornith and Bonsai rows.
- V3 (partial): blue_circles 4 instead of 3; the other four counts exact; JSON in a code fence. One count off by one: partial.

### PII

Only the two PII cells in the battery (T9 and T10) were run on this file: T9 was clean in all four runs; T10 left two names in plaintext in three of the four.

## Verdict under these settings
Usable: yes, with limits

A reliable fast baseline, and the reference for how noisy single-run grades are. Watch the PII prompt that uses typed names, the code task's import handling and prompt-only JSON, which fenced its answer in some runs.

## Caveats
- T7 (code task) is graded on a static read plus a run with and without PyYAML installed; an unguarded top-level import makes it a partial.
- L5 (buried instruction) failed and J4 (prompt-only JSON) was fenced in several runs; L2 was wrong in one run.
- One machine and one run per cell unless stated. In an earlier measurement on a different model (Qwen3.5-9B), about 2 of 23 grades changed between identical reruns, so a single grade difference is a data point, not an effect.
- Hardware, driver and runner build are the ones listed above; results on other cards or builds are not directly comparable.

## How to reproduce
1. Get a llama.cpp build from the repo linked above (this test used b9682) and the Q4_K_M file from the quant repo.
2. Start the server (replace the paths and pick any free port):

```
llama-server --model <path-to>/Qwen3.5-9B-Q4_K_M.gguf --alias qwen35-9b-q4km-unsloth --jinja --reasoning off --temp 0.7 --top-p 0.8 --top-k 20 --min-p 0.00 --flash-attn auto --fit on --fit-target 4096 --no-mmap --threads 8 --threads-batch 8 --ctx-size 65536 --parallel 1 --port <port> --host 127.0.0.1
```

3. Check the server before trusting any result: the model name echoed in each reply matches the alias, `finish_reason` is `stop`, and `cache_n` is 0 on the first request of a fresh server.
4. Send each test as a separate request with no system prompt. The test prompts for the main battery are not published yet; the PII prompts are in the [appendix](../assets/pii-pack.md).

`Status: Tested on llama.cpp b9682, 18 Oct 2026`
