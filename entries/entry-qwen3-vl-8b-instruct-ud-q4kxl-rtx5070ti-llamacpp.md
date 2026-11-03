<!-- Visibility: public -->
# Qwen3-VL-8B-Instruct UD-Q4_K_XL on a 16 GB GPU: 12 tokens per second at 64K context, 123 at 32K

Date: 21 Oct 2026

**In short:** With an f16 cache and a 65,536 context, layers spill to the CPU and this vision model decodes at only 11.8 tokens per second; the same file with a 32,768 context decodes at about 120 to 123 tok/s, and a q8_0 cache at 65,536 reaches 36.9. Text grades match the plain Qwen3-8B control apart from the skill-wording PII prompt, which passed here. The shape-counting vision cell (V3) fails in every variant.

## Hardware class
- CPU: Intel Core i7-7700 (7th generation desktop, 4 cores, 8 threads)
- GPU model and VRAM: NVIDIA GeForce RTX 5070 Ti, 16 GB
- RAM: 64 GB DDR4, running at about 2400 MT/s, dual channel
- OS family: Windows 10 (LTSC)
- GPU driver: NVIDIA 610.88 (driver package dated 21 Jul 2026)

## Setup
- Runner and version: llama.cpp `llama-server`, build b9682, Windows CUDA 12.4 binary ([llama.cpp](https://github.com/ggml-org/llama.cpp))
- Harness (direct API or agent): direct API (`/v1/chat/completions`), one fresh request per test, no system prompt, no agent, no skills loaded
- Model and quant: Qwen3-VL-8B-Instruct, Unsloth dynamic `UD-Q4_K_XL` GGUF with the F16 projector (files `Qwen3-VL-8B-Instruct-UD-Q4_K_XL.gguf` and `Qwen3-VL-8B-Instruct-GGUF-mmproj-F16.gguf`)
  - Quant repo: [unsloth/Qwen3-VL-8B-Instruct-GGUF](https://huggingface.co/unsloth/Qwen3-VL-8B-Instruct-GGUF)
  - Usage guide: [Unsloth Qwen3-VL guide](https://unsloth.ai/docs/models/tutorials/qwen3-how-to-run-and-fine-tune/qwen3-vl-how-to-run-and-fine-tune)
  - Model license: Apache-2.0, from the license field of the [unsloth/Qwen3-VL-8B-Instruct-GGUF](https://huggingface.co/unsloth/Qwen3-VL-8B-Instruct-GGUF) card; checked 2026-11-02.
- Context size: 65536 for the main run; 32768 in a labeled variant
- Thinking mode: off
- Settings used:

| Setting | Value |
|---|---|
| GPU layers | chosen by `--fit on --fit-target 4096`; layers spill to the CPU at 65536 |
| KV cache | f16 (default); q8_0 in a labeled variant |
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
- Generation: 11.8 tok/s (server-reported; 11.2 at the client) at context 65536 with an f16 cache
- 19,767-token prompt answered cleanly; long-context decode at about 22K tokens was 1.9 to 2.0 tok/s (8.8 to 9.4 with a q8_0 cache)
- Vision requests decoded at about 11 tok/s (34 to 39 with a q8_0 cache)
- Hardware monitor (32768 variant): GPU load up to 98%, 233 W, 12.6 GB dedicated GPU memory in use, shared GPU memory 0.16 GB (1.7 GB in the 65536 run)
- Time to first token and RAM use: not recorded

### Variants tried (one change at a time unless noted)

| Variant | What changed | Decode (tok/s) | Outcome |
|---|---|---|---|
| Context 32768 | `--ctx-size 32768` (main run 65536) | 123.1 (first run; a rerun gave 120.6) | Long-answer test only; same speed as the plain Qwen3-8B, so nothing spills |
| q8_0 cache | K/V cache f16 to q8_0 at 65536 | 36.9 | 3.1 times faster, same grades; still well below the plain 8B, so some work still spills |
| More image tokens | `--image-min-tokens 1024` (advised by the startup log) | 8.8 to 11.2 | Image prompts grew from 366 to 563 tokens to about 1,100; grades unchanged, V3 still fails |

The speed follows whether the context allocation fits in VRAM, not the model or the projector.

### Quality results (graded by a written key)

Text battery, long context and structured output (main run): 17 of 22 cells pass, 4 partial, 1 fail.

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
| T10 PII redaction, "run the skill" wording (trap) | partial |

**Long context, about 22K tokens, f16 cache (main run)**

| Cell | Result |
|---|---|
| L1 Needle early | pass |
| L2 Needle middle | pass |
| L3 Needle late | partial |
| L4 Two needles, order | pass |
| L5 Buried instruction | fail |

**Long context, about 22K tokens, q8_0 cache**

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
| J3 Nested array and total | partial |
| J4 Prompt-only JSON (no response format) | pass |
| J5 Null not invented | pass |
| J6 Native tool call | pass |
| J7 Number words | pass |

**Vision (V1 to V7)**

| Cell | Result |
|---|---|
| V1 Scene and text | pass |
| V2 Chart to table | pass |
| V3 Counting shapes | fail |
| V4 Terminal screenshot | pass |
| V5 Invoice | pass |
| V6 No-image control | pass |
| V7 Two images | pass |

**Cells that were not a pass**

- T7 (partial): Executed with PyYAML: .json and .yaml parse, malformed JSON and YAML, empty file, unsupported extension and missing file raise the right errors.
- T10 (partial): Email, phone, SSN, IP tagged ([EMAIL_ADDRESS] and so on); Jane Smith and Bob Wu left in plaintext. Partial, same as the control and the 9B baseline runs.
- about 22K tokens, f16 cache (main run): L3 (partial): Code word and entry 468 right, position said middle_third instead of last_third. Partial, same wrong answer as the Qwen3-8B control.
- about 22K tokens, f16 cache (main run): L5 (fail): a one-sentence summary of the log that does not end with BANANA.
- about 22K tokens, q8_0 cache: L3 (partial): Code word and entry 468 right, position middle_third instead of last_third. Partial, same as the base run.
- about 22K tokens, q8_0 cache: L5 (fail): a one-sentence summary that does not end with BANANA (same result as the base run).
- J3 (partial): order_id 88, two items, but total 27.00 instead of 21 (2 x 4.50 + 12). Partial. J3 is a known flip cell: the Qwen3-8B control got 21.00.
- V3 (fail): red_circles 6 (is 5), blue_circles 4 (is 3), blue_squares 5 (is 4), green_triangles 2 and yellow_stars 1 right: three counts off by one, so fail by the grading key (partial allows one or two). JSON only.

T9 (skill wording) redacted all five values; T10 left both names in plaintext.

## Verdict under these settings
Usable: yes at 32K context; slow at 64K on this card unless the cache is quantized

A capable vision model on a 16 GB card, but the context and cache settings decide whether it runs at 12 or 123 tok/s. Use a 32K context, or q8_0 cache at 64K, and do not trust the counting cell (V3).

## Caveats
- V3 (count shapes by color) failed in every run with the same wrong counts, so more image tokens did not help.
- Text grades: 17 pass, 4 partial (T7, T10, L3, J3), 1 fail (L5) across the 22 graded cells, one run.
- One machine and one run per cell unless stated. In an earlier measurement on a different model (Qwen3.5-9B), about 2 of 23 grades changed between identical reruns, so a single grade difference is a data point, not an effect.
- Hardware, driver and runner build are the ones listed above; results on other cards or builds are not directly comparable.

## How to reproduce
1. Get a llama.cpp build from the repo linked above (this test used b9682), the `UD-Q4_K_XL` file and the F16 projector from the quant repo.
2. Start the server (replace the paths and pick any free port):

```
llama-server --model <path-to>/Qwen3-VL-8B-Instruct-UD-Q4_K_XL.gguf --mmproj <path-to>/Qwen3-VL-8B-Instruct-GGUF-mmproj-F16.gguf --alias qwen3-vl-8b-instruct --jinja --reasoning off --temp 0.7 --top-p 0.8 --top-k 20 --min-p 0.00 --flash-attn auto --fit on --fit-target 4096 --no-mmap --threads 8 --threads-batch 8 --ctx-size 65536 --parallel 1 --port <port> --host 127.0.0.1
```

3. Check the server before trusting any result: the model name echoed in each reply matches the alias, `finish_reason` is `stop`, and `cache_n` is 0 on the first request of a fresh server.
4. Send each test as a separate request with no system prompt. The test prompts for the main battery are not published yet; the PII prompts are in the [appendix](../assets/pii-pack.md).

`Status: Tested on llama.cpp b9682, 21 Oct 2026`
