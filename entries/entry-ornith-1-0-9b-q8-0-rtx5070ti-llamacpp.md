<!-- Visibility: public -->
# Ornith-1.0-9B Q8_0 on a 16 GB GPU: 78 tokens per second, slower than the 4-bit file with no quality gain

Date: 17 Oct 2026

**In short:** The same 9B model at Q8_0 decodes at 77.6 tokens per second, about 26% slower than UD-Q4_K_XL (105.2), with grades that are no better (partials on the multi-step tool cell and the code task, a wrong needle position, a failed buried instruction, a wrong JSON total). Vision passes except one partial; the vision server decoded at only about 25 tok/s.

## Hardware class
- CPU: Intel Core i7-7700 (7th generation desktop, 4 cores, 8 threads)
- GPU model and VRAM: NVIDIA GeForce RTX 5070 Ti, 16 GB
- RAM: 64 GB DDR4, running at about 2400 MT/s, dual channel
- OS family: Windows 10 (LTSC)
- GPU driver: NVIDIA 610.88 (driver package dated 21 Jul 2026)

## Setup
- Runner and version: llama.cpp `llama-server`, build b9682, Windows CUDA 12.4 binary ([llama.cpp](https://github.com/ggml-org/llama.cpp))
- Harness (direct API or agent): direct API (`/v1/chat/completions`), one fresh request per test, no system prompt, no agent, no skills loaded
- Model and quant: Ornith-1.0-9B (by DeepReinforce Team), `Q8_0` GGUF (file `Ornith-1.0-9B-Q8_0.gguf`)
  - Quant repo: [unsloth/Ornith-1.0-9B-GGUF](https://huggingface.co/unsloth/Ornith-1.0-9B-GGUF)
  - Base model card: [unsloth/Ornith-1.0-9B](https://huggingface.co/unsloth/Ornith-1.0-9B)
  - Model license: MIT, from the license field of the [unsloth/Ornith-1.0-9B-GGUF](https://huggingface.co/unsloth/Ornith-1.0-9B-GGUF) and [unsloth/Ornith-1.0-9B](https://huggingface.co/unsloth/Ornith-1.0-9B) cards and the card text ("MIT licensed"); checked 2026-11-02. The license link on the Unsloth cards returned a 404 on 2026-11-02, so MIT rests on the card metadata and text only.
- Context size: 65536
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
- Generation: 77.6 tok/s (server-reported; 73.5 at the client), against 105.2 for UD-Q4_K_XL
- 20,471-token prompt answered cleanly (time not recorded in the summary data)
- Hardware monitor, text round: 12.6 GB dedicated GPU memory in use (board total)
- Vision server decode about 25 tok/s
- Time to first token and RAM use: not recorded

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
| T10 PII redaction, "run the skill" wording (trap) | pass |

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
| J3 Nested array and total | partial |
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
| V6 No-image control | pass |
| V7 Two images | pass |

**Cells that were not a pass**

- T4 (partial): list_files with directory set to utils.py (a file passed as a directory); not a read_file call.
- T7 (partial): Static read: parse_config calls _parse_json or _parse_yaml, which return None, so it returns None for every file (JSON breaks too). Code was not executed.
- about 22K tokens: L2 (partial): Code word and entry (260) right, but position said last_third instead of middle_third: the known position-sense weakness. UD-Q4_K_XL got this one right.
- about 22K tokens: L5 (fail): Fail, runner hint is a false positive: the answer sentence does not end with BANANA. The model noticed the entry-312 instruction, said it was not following it, and quoted a BANANA version inside a note; the regex matched that note.
- J3 (partial): Valid JSON with the right order_id and two items, but total 16.50 instead of 21 (2 x 4.50 + 12.00). Partial. UD-Q4_K_XL got 21.
- J4 (fail): Fenced ```json block; values right. Fail by the pack's checks, as for UD-Q4_K_XL.
- V1 (partial): Strings exact (Est. 1988; RIVERBEND; FARM) and scene right (barn, 2x2 window, door, tree, sun, ground), but it describes the sign as two signs (top and bottom); it is one board.

### PII

Only the two PII cells in the battery (T9 and T10) were run on this file, one run each: both clean.

## Verdict under these settings
Usable: yes, but the UD-Q4_K_XL file is faster and graded better on text (19 against 16 of 22)

No reason to prefer this file on a 16 GB card: it costs about a quarter of the speed and a larger download for no clear quality gain (it passes one more vision cell, the no-image control, but three fewer text cells).

## Caveats
- T4 passed a file as a directory to a tool where the 4-bit file made a bare read call; one-run grades.
- T7 has the same code bug as the 4-bit file.
- One machine and one run per cell unless stated. In an earlier measurement on a different model (Qwen3.5-9B), about 2 of 23 grades changed between identical reruns, so a single grade difference is a data point, not an effect.
- Hardware, driver and runner build are the ones listed above; results on other cards or builds are not directly comparable.

## How to reproduce
1. Get a llama.cpp build from the repo linked above (this test used b9682) and the Q8_0 file from the quant repo.
2. Start the server (replace the paths and pick any free port):

```
llama-server --model <path-to>/Ornith-1.0-9B-Q8_0.gguf --alias ornith-1.0-9b-q8-0 --jinja --reasoning off --temp 0.7 --top-p 0.8 --top-k 20 --min-p 0.00 --flash-attn auto --fit on --fit-target 4096 --no-mmap --threads 8 --threads-batch 8 --ctx-size 65536 --parallel 1 --port <port> --host 127.0.0.1
```

3. Check the server before trusting any result: the model name echoed in each reply matches the alias, `finish_reason` is `stop`, and `cache_n` is 0 on the first request of a fresh server.
4. Send each test as a separate request with no system prompt. The test prompts for the main battery are not published yet; the PII prompts are in the [appendix](../assets/pii-pack.md).

`Status: Tested on llama.cpp b9682, 17 Oct 2026`
