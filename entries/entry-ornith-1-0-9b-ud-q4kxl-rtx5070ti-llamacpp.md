<!-- Visibility: public -->
# Ornith-1.0-9B UD-Q4_K_XL on a 16 GB GPU: 105 tokens per second

Date: 16 to 17 Oct 2026

**In short:** A 9B model that decodes at about 105 tokens per second with every layer on the GPU at 64K context. The text battery is clean except the code task (the parse function never returns the loaded data), the buried-instruction long-context cell (L5) fails and prompt-only JSON (J4) is fenced. Model-card sampling fixed the code task but moved another cell; with thinking on, one short answer took 13,132 tokens.

## Hardware class
- CPU: Intel Core i7-7700 (7th generation desktop, 4 cores, 8 threads)
- GPU model and VRAM: NVIDIA GeForce RTX 5070 Ti, 16 GB
- RAM: 64 GB DDR4, running at about 2400 MT/s, dual channel
- OS family: Windows 10 (LTSC)
- GPU driver: NVIDIA 610.88 (driver package dated 21 Jul 2026)

## Setup
- Runner and version: llama.cpp `llama-server`, build b9682, Windows CUDA 12.4 binary ([llama.cpp](https://github.com/ggml-org/llama.cpp))
- Harness (direct API or agent): direct API (`/v1/chat/completions`), one fresh request per test, no system prompt, no agent, no skills loaded
- Model and quant: Ornith-1.0-9B (by DeepReinforce Team), Unsloth dynamic `UD-Q4_K_XL` GGUF (file `Ornith-1.0-9B-UD-Q4_K_XL.gguf`)
  - Quant repo: [unsloth/Ornith-1.0-9B-GGUF](https://huggingface.co/unsloth/Ornith-1.0-9B-GGUF)
  - Base model card: [unsloth/Ornith-1.0-9B](https://huggingface.co/unsloth/Ornith-1.0-9B)
  - Vision tests used a projector file from the same quant repo
  - Model license: MIT, from the license field of the [unsloth/Ornith-1.0-9B-GGUF](https://huggingface.co/unsloth/Ornith-1.0-9B-GGUF) and [unsloth/Ornith-1.0-9B](https://huggingface.co/unsloth/Ornith-1.0-9B) cards and the card text ("MIT licensed"); checked 2026-11-02. The license link on the Unsloth cards returned a 404 on 2026-11-02, so MIT rests on the card metadata and text only.
- Context size: 65536
- Thinking mode: off for the main tests; on for one labeled variant
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

- Runs per test cell: 1

## Test description
- Speed: a short prompt asking for a long answer (T6); generation speed is the server-reported decode rate.
- Text battery: 22 graded cells: ten general cells (T1 to T10: exact JSON, a 20K-token context stress test, tool calls, a two-turn code task, memory, two PII redaction prompts), five long-context needle cells (L1 to L5, about 22K tokens) and seven structured-output cells (J1 to J7).
- T9 and T10 ask the model to "invoke" or "run" a `pii-redactor` skill on a short fictional text. No such skill, tool or system prompt exists in the request: this is a deliberate trap, and a good answer redacts the text anyway. See [assets/pii-pack.md](../assets/pii-pack.md).
- Grading uses a written key. A JSON answer wrapped in a code fence counts as a fail where bare JSON was asked for (T1, T3, T4, J4).

## Measured results
- Generation: 105.2 tok/s (server-reported; 94.7 at the client)
- 20,471-token prompt answered cleanly (time not recorded in the summary data)
- Hardware monitor, text round: GPU load up to 98%, 264 W, 65 C, 9.8 GB dedicated GPU memory in use (board total), CPU up to 39%; vision round 11.0 GB, 210 W, no spill
- Time to first token and RAM use: not recorded

### Variants tried (one change at a time unless noted)

| Variant | What changed | Decode (tok/s) | Outcome |
|---|---|---|---|
| Model-card sampling | temp 0.6, top-p 0.95 (default 0.7 / 0.8) | 105.8 | No T1 to T10 partials; T7 passed; J3 partial, J4 fail, L5 fail |
| Thinking on, card sampling | `--reasoning on`, temp 0.6, top-p 0.95, output limit 16,384 | 103.6 | T1 and T3 fenced (fail), T7 partial; the long answer took 13,132 tokens (129 s) |

Wall-clock times are not comparable across thinking modes; decode speed is.

### Quality results (graded by a written key)

Text battery, long context and structured output (main run): 19 of 22 cells pass, 1 partial, 2 fail.

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
| V3 Counting shapes | pass |
| V4 Terminal screenshot | pass |
| V5 Invoice | pass |
| V6 No-image control | fail |
| V7 Two images | pass |

**Cells that were not a pass**

- T7 (partial): Static read: turn 2 adds YAML with extension detection, but parse_config calls _load_json or _load_yaml and never returns the result, so it returns None for every file (JSON breaks too).
- about 22K tokens: L5 (fail): One correct sentence about the log; it did not end with BANANA, so the buried instruction at entry 312 was not followed.
- J4 (fail): Same task with no response_format: values right but wrapped in a ```json fence, so the raw-parse and no-fence checks fail.
- V1 (partial): Both strings exact (Est. 1988; RIVERBEND / FARM). Scene correct on building, window, door, tree, sun, sign, ground, but it invented thin black lines framing the top and bottom edges and called the sign two planks (it is one board, two lines of text).
- V6 (fail): No scene invented, but it said it had reviewed the attached image and that it was blank, when no image was sent, instead of saying no image was attached. The key has only pass or fail for V6, and this answer does not say it sees no image, so it is a fail.

### PII

Only the two PII cells in the battery (T9 and T10) were run on this file, one run each: both clean.

## Verdict under these settings
Usable: yes, with limits

A fast 9B with clean tool calls and PII cells. Check generated code for returned values, and expect prompt-only JSON in a code fence.

## Caveats
- T7 is graded on a static read; the baseline function never returns the loaded data, while the card-sampling variant passes, so this is a one-run swing.
- The L5 runner hint (word found) is a false positive for the card-sampling run: the model described the buried instruction instead of following it.
- The thinking-on variant is the mode the model card intends; its decode speed is comparable but its wall-clock is not.
- One machine and one run per cell unless stated. In an earlier measurement on a different model (Qwen3.5-9B), about 2 of 23 grades changed between identical reruns, so a single grade difference is a data point, not an effect.
- Hardware, driver and runner build are the ones listed above; results on other cards or builds are not directly comparable.

## How to reproduce
1. Get a llama.cpp build from the repo linked above (this test used b9682) and the UD-Q4_K_XL file from the quant repo.
2. Start the server (replace the paths and pick any free port):

```
llama-server --model <path-to>/Ornith-1.0-9B-UD-Q4_K_XL.gguf --alias ornith-1.0-9b-ud-q4kxl --jinja --reasoning off --temp 0.7 --top-p 0.8 --top-k 20 --min-p 0.00 --flash-attn auto --fit on --fit-target 4096 --no-mmap --threads 8 --threads-batch 8 --ctx-size 65536 --parallel 1 --port <port> --host 127.0.0.1
```

3. Check the server before trusting any result: the model name echoed in each reply matches the alias, `finish_reason` is `stop`, and `cache_n` is 0 on the first request of a fresh server.
4. Send each test as a separate request with no system prompt. The test prompts for the main battery are not published yet; the PII prompts are in the [appendix](../assets/pii-pack.md).

`Status: Tested on llama.cpp b9682, 17 Oct 2026`
