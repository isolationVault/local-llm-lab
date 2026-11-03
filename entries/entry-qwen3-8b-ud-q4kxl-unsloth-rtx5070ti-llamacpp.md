<!-- Visibility: public -->
# Qwen3-8B UD-Q4_K_XL on a 16 GB GPU: 124 tokens per second, same grades as the Q4_K_M control

Date: 21 Oct 2026

**In short:** Unsloth's dynamic 4-bit file of Qwen3-8B (5.14 GB against 5.03 GB for the bartowski Q4_K_M; decimal GB, as on the model cards) decodes at 124.4 tokens per second at its native 32K window and graded identically to the Q4_K_M control cell for cell: 17 pass, 3 partial, 2 fail across 22 cells, including the same skill-wording PII failure.

## Hardware class
- CPU: Intel Core i7-7700 (7th generation desktop, 4 cores, 8 threads)
- GPU model and VRAM: NVIDIA GeForce RTX 5070 Ti, 16 GB
- RAM: 64 GB DDR4, running at about 2400 MT/s, dual channel
- OS family: Windows 10 (LTSC)
- GPU driver: NVIDIA 610.88 (driver package dated 21 Jul 2026)

## Setup
- Runner and version: llama.cpp `llama-server`, build b9682, Windows CUDA 12.4 binary ([llama.cpp](https://github.com/ggml-org/llama.cpp))
- Harness (direct API or agent): direct API (`/v1/chat/completions`), one fresh request per test, no system prompt, no agent, no skills loaded
- Model and quant: Qwen3-8B, Unsloth dynamic `UD-Q4_K_XL` GGUF (file `Qwen3-8B-UD-Q4_K_XL.gguf`, 5.14 GB)
  - Quant repo: [unsloth/Qwen3-8B-GGUF](https://huggingface.co/unsloth/Qwen3-8B-GGUF)
  - Model license: Apache-2.0, from the license field of the [unsloth/Qwen3-8B-GGUF](https://huggingface.co/unsloth/Qwen3-8B-GGUF) card; checked 2026-11-02.
- Context size: 32768 (the model's native window)
- Thinking mode: off
- Settings used:

| Setting | Value |
|---|---|
| GPU layers | chosen by `--fit on --fit-target 4096` |
| KV cache | f16 (default) |
| Flash attention | auto |
| Memory mapping | off (`--no-mmap`) |
| Threads | 8 (`--threads 8 --threads-batch 8`) |
| Parallel slots | 1 |
| Chat template | `--jinja` |
| Sampling, thinking off | temp 0.7, top-p 0.8, top-k 20, min-p 0, presence penalty 0 |

- Runs per test cell: 1

## Test description
- Speed: a short prompt asking for a long answer (T6); generation speed is the server-reported decode rate.
- Text battery: 22 graded cells: ten general cells (T1 to T10: exact JSON, a 20K-token context stress test, tool calls, a two-turn code task, memory, two PII redaction prompts), five long-context needle cells (L1 to L5, about 22K tokens) and seven structured-output cells (J1 to J7).
- T9 and T10 ask the model to "invoke" or "run" a `pii-redactor` skill on a short fictional text. No such skill, tool or system prompt exists in the request: this is a deliberate trap, and a good answer redacts the text anyway. See [assets/pii-pack.md](../assets/pii-pack.md).
- Grading uses a written key. A JSON answer wrapped in a code fence counts as a fail where bare JSON was asked for (T1, T3, T4, J4).

## Measured results
- Generation: 124.4 tok/s (server-reported; 92.8 at the client)
- 19,771-token prompt answered cleanly
- Time to first token, RAM use and GPU memory: not recorded in the summary data

### Quality results (graded by a written key)

Text battery, long context and structured output (main run): 17 of 22 cells pass, 3 partial, 2 fail.

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
| T9 PII redaction, "invoke the skill" wording (trap) | fail |
| T10 PII redaction, "run the skill" wording (trap) | partial |

**Long context, about 20K tokens**

| Cell | Result |
|---|---|
| L1 Needle early | pass |
| L2 Needle middle | pass |
| L3 Needle late | partial |
| L4 Two needles, order | pass |
| L5 Buried instruction | fail |

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

**Cells that were not a pass**

- T7 (partial): Executed with PyYAML: .json, .yaml, .yml parse and malformed JSON and YAML raise errors. Partial by the grading key: top-level import yaml, so plain .json fails with ModuleNotFoundError when PyYAML is missing (same as the control).
- T9 (fail): Fail by the 2026-10-20 threshold: only the SSN and IP were masked (as XXX-XX-XXXX and XXX.XXX.XXX.XXX); John Doe, the email and the phone number stayed in plaintext, so 2 of 5 values redacted. Same result as the Q4_K_M control.
- T10 (partial): Email, phone, SSN, IP tagged ([EMAIL] and so on); Jane Smith and Bob Wu left in plaintext. Partial, same as the control.
- about 20K tokens: L3 (partial): Code word and entry 468 right, position said middle_third instead of last_third. Partial, same as the Q4_K_M control.
- about 20K tokens: L5 (fail): a one-sentence summary of the log that does not end with BANANA (same result as the control).

T9 (skill wording) masked only the SSN and IP address and left the name, email and phone; T10 left both names in plaintext. Same as the Q4_K_M control.

## Verdict under these settings
Usable: yes at its native 32K window

Equivalent to the bartowski Q4_K_M file on this battery. Pick on download size and tooling preference, not on these results.

## Caveats
- T7 (code task) partial from an unguarded top-level import; plain JSON config fails when PyYAML is missing.
- Same 32K window limit as the control, so treat it as a quality data point for short-context use.
- One machine and one run per cell unless stated. In an earlier measurement on a different model (Qwen3.5-9B), about 2 of 23 grades changed between identical reruns, so a single grade difference is a data point, not an effect.
- Hardware, driver and runner build are the ones listed above; results on other cards or builds are not directly comparable.

## How to reproduce
1. Get a llama.cpp build from the repo linked above (this test used b9682) and the UD-Q4_K_XL file from the quant repo.
2. Start the server (replace the paths and pick any free port):

```
llama-server --model <path-to>/Qwen3-8B-UD-Q4_K_XL.gguf --alias qwen3-8b-ud-q4kxl --jinja --reasoning off --temp 0.7 --top-p 0.8 --top-k 20 --min-p 0.00 --flash-attn auto --fit on --fit-target 4096 --no-mmap --threads 8 --threads-batch 8 --ctx-size 32768 --parallel 1 --port <port> --host 127.0.0.1
```

3. Check the server before trusting any result: the model name echoed in each reply matches the alias, `finish_reason` is `stop`, and `cache_n` is 0 on the first request of a fresh server.
4. Send each test as a separate request with no system prompt. The test prompts for the main battery are not published yet; the PII prompts are in the [appendix](../assets/pii-pack.md).

`Status: Tested on llama.cpp b9682, 21 Oct 2026`
