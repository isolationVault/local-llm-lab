<!-- Visibility: public -->
# Qwen3.8-27B UD-Q2_K_XL on a 16 GB GPU: fully on the GPU at 64K context

Date: 28 Oct 2026 (runs between 22 and 28 Oct 2026)

**In short:** with every layer on the GPU (q8_0 cache, fit-target 2048), this 2-bit 27B model runs at 54 tokens per second on a 16 GB card and passed the whole test battery. The same file with default-style settings ran at 3.6 tokens per second, so the recipe mattered more than the model.

## Hardware class
- CPU: Intel Core i7-7700 (7th generation desktop, 4 cores, 8 threads)
- GPU model and VRAM: NVIDIA GeForce RTX 5070 Ti, 16 GB
- RAM: 64 GB DDR4, running at about 2400 MT/s, dual channel
- OS family: Windows 10 (LTSC)
- GPU driver: NVIDIA 610.88 (driver package dated 21 Jul 2026)

## Setup
- Runner and version: llama.cpp `llama-server`, build b9682, Windows CUDA 12.4 binary ([llama.cpp](https://github.com/ggml-org/llama.cpp))
- Harness (direct API or agent): direct API (`/v1/chat/completions`), one fresh request per test, no system prompt, no agent, no skills loaded
- Model and quant: Qwen3.8-27B, Unsloth dynamic `UD-Q2_K_XL` GGUF
  - Quant repo: [unsloth/Qwen3.8-27B-GGUF](https://huggingface.co/unsloth/Qwen3.8-27B-GGUF)
  - Base model card: [Qwen/Qwen3.8-27B](https://huggingface.co/Qwen/Qwen3.8-27B)
  - Sampling and usage guide: [Unsloth Qwen3.8-27B guide](https://unsloth.ai/docs/models/qwen3.8)
  - Vision tests also used the `mmproj-F16.gguf` projector from the same quant repo
  - Model license: Apache-2.0, from the license field of the [unsloth/Qwen3.8-27B-GGUF](https://huggingface.co/unsloth/Qwen3.8-27B-GGUF) and [Qwen/Qwen3.8-27B](https://huggingface.co/Qwen/Qwen3.8-27B) cards; checked 2026-11-02.
- Context size: 65536
- Thinking mode: off for the main tests; on (effort xhigh and medium) for the PII side test only
- Settings used (the "fast recipe"):

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
| Sampling, thinking on | temp 1.0, top-p 0.95, top-k 20, min-p 0, presence penalty 0 (the Thinking row of the Unsloth page) |

- Runs per test cell: 1 for the main tests; 5 repeats per cell for the PII side test

## Test description
- Speed: a short prompt (36 tokens) asking for a long answer (454 tokens generated).
- Text battery: 22 graded cells covering tool calls, a 20K-token context stress test, long-context needle questions, structured JSON output, and a two-turn code task.
- Long context: five needle questions in a prompt of about 59,300 tokens (about 6,200 below the limit).
- Vision: seven image tests (scene and text, chart to table, counting, terminal screenshot, invoice, a no-image control, two images).
- PII side test: fictional texts with a name, email, phone, SSN and IP address, asked three ways, 50 requests per run. One of the three wordings says "invoke the pii-redactor skill" but no such skill exists in the request. That is a deliberate trap; the other two wordings are the controls. See [assets/pii-pack.md](../assets/pii-pack.md).

## Measured results
- Prompt processing (tokens per second): about 1,480 (20,471-token prompt in 13.9 s); about 1,190 at 59K tokens
- Generation (tokens per second): 54.4 (short prompt, 454 tokens); 37 to 56 across the whole battery; 39 to 40 with a 59K-token prompt
- Time to first token (not recorded)
- VRAM use: 13,908 MiB loaded for text; 15,042 MiB with the vision projector; shared GPU memory 0.19 GB in Task Manager (no spill)
- RAM use: not recorded for the fast recipe. With long prompts the server's prompt cache grows (about 2.1 GB per 59K prompt plus about 2.4 GB of checkpoints, capped at 8 GB by default) and then levels off.
- Thermals: GPU about 62 to 63 C, up to about 290 W under load; CPU package 84 C max; thermal throttling flagged in 0 of 423 samples (about 14 minutes) of the main run, against 532 of 627 on the slow recipe

### Same file, different recipe

| Recipe | Layers on CPU | Generation (tok/s) | Whole battery |
|---|---|---|---|
| Default-style: fit-target 4096, f16 KV cache | about 18 | 3.61 | 20.6 min |
| Fast recipe: fit-target 2048, q8_0 KV cache | 0 | 54.4 | 3.1 min |

Both changes were made together, so their effects are not separated. Nothing about the model changed.

### Quality results (one run per cell)

| Pack | Result |
|---|---|
| Text, long-context and structured-output cells | 22 of 22 pass |
| Long context at about 59K tokens | 5 of 5 pass |
| Vision | 7 of 7 pass |

### PII side test (fictional data, 30 redaction requests per run, plus 20 tool requests)

| Mode | Redaction requests passing (of 30) | Notes |
|---|---|---|
| Thinking off | 26 (4 partial) | 3 of 10 "plain wording" answers collapsed to a bare `[REDACTED]` and dropped the sentence |
| Thinking on, xhigh | 30 | Short pack, so this says little about rare failures |
| Thinking on, medium | 28 (2 partial) | One answer left the SSN and IP address in the clear |
| Control: the skill declared as a real tool (all modes) | 20 of 20 correct | Calls the tool with the exact text, then relays the result |

The same test on two other quants of this model gave different results with thinking off (one quant failed the "invoke the skill" wording 5 of 5 times).

A harder PII pack (two longer fictional texts, 15 personal values each, plus values that must not be redacted) was also run, 30 requests per mode. On this quant: thinking off 17 pass, 12 partial, 1 fail; xhigh 24 pass and 6 requests cut off at our 8,192-token limit; medium 25 pass, 4 partial, 1 fail. Medium was the most reliable setting on all three quants. Details for all of it are in [assets/pii-pack.md](../assets/pii-pack.md).

## Verdict under these settings
Usable: yes

About 54 tok/s at short context and about 40 tok/s near 59K is comfortable for chat and for agent work on a 16 GB card, and every layer stays on the GPU. The same file ran at 3.61 tok/s on the first recipe tried, so the recipe matters more than the model here.

## Caveats
- Presence penalty was not set: the runs left it at the server default of 0, while the Unsloth page lists 1.5 for the Instruct row (it lists 0 for the Thinking row). The thinking-off results here therefore differ from that row in this one setting.
- One machine, one run per cell for the main tests. In an earlier measurement on a different model (Qwen3.5-9B), about 2 of 23 grades changed between identical reruns, so a single grade difference is a data point, not an effect.
- This is a 2-bit quant. The q8_0 cache showed no visible quality cost on these packs, but contexts beyond about 59K tokens were not tested.
- Answers were correct but not always tidy: one long-context answer opened with about 4,500 characters of visible reasoning even with thinking off, and the code task's second turn swallowed errors instead of raising them.
- The thinking-on PII runs changed three things at once against thinking off: thinking, the sampling values, and (for xhigh) a system line the chat template adds.
- PII results use short fictional texts. A pass here is not a privacy guarantee.
- On the harder PII pack, thinking-on answers that hit our 8,192-token limit are reported as capped, not as failures. A larger limit might change the xhigh numbers.

## How to reproduce
1. Get a llama.cpp build from the repo linked above (this test used b9682) and the `UD-Q2_K_XL` file from the quant repo.
2. Start the server (replace the path and pick any free port):

```
llama-server --model <path-to>/Qwen3.8-27B-UD-Q2_K_XL.gguf --alias qwen38-27b-q2kxl-ft2048-q8kv --jinja --reasoning off --temp 0.7 --top-p 0.8 --top-k 20 --min-p 0.00 --flash-attn auto --cache-type-k q8_0 --cache-type-v q8_0 --fit on --fit-target 2048 --no-mmap --threads 8 --threads-batch 8 --ctx-size 65536 --parallel 1 --port <port> --host 127.0.0.1
```

3. Check the server before trusting any result: the model name echoed in each reply matches the alias, `finish_reason` is `stop`, and `cache_n` is 0 on the first request of a fresh server.
4. Send each test as a separate request with no system prompt. The test prompts for the main battery are not published yet; the PII prompts are in the [appendix](../assets/pii-pack.md).

`Status: Tested on llama.cpp b9682, 28 Oct 2026`
