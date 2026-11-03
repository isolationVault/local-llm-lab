<!-- Visibility: public -->
# Ternary Bonsai 2 27B PTQ1_0 on a 16 GB GPU: about 80 tokens per second, with and without the MTP head

Date: 14 to 29 Oct 2026

**In short:** A ternary (about 1.75 bits per weight) 27B model that runs entirely on the GPU and decodes at 81 tokens per second at 64K context with the MTP speed head, 78 without it. Text and structured-output cells are tidy; the buried-instruction cell (L5) fails at both 22K and 60K tokens, and PII wording is weaker with thinking off than with thinking medium.

## Hardware class
- CPU: Intel Core i7-7700 (7th generation desktop, 4 cores, 8 threads)
- GPU model and VRAM: NVIDIA GeForce RTX 5070 Ti, 16 GB
- RAM: 64 GB DDR4, running at about 2400 MT/s, dual channel
- OS family: Windows 10 (LTSC)
- GPU driver: NVIDIA 610.88 (driver package dated 21 Jul 2026)

## Setup
- Runner and version: llama.cpp `llama-server` from the sudoingX fork (`llama-server --version` reports 0.2.0-dev, build 10759, commit de947abc3) for the text, long-context, structured-output and PII runs; mainline build b9682 for the vision run. Windows CUDA binaries
- Harness (direct API or agent): direct API (`/v1/chat/completions`), one fresh request per test, no system prompt, no agent, no skills loaded
- Model and quant: Ternary Bonsai 2 27B (Prism ML), `PTQ1_0` GGUF with and without the Qwen 3.8 MTP speed head
  - Model repo: [prism-ml/Ternary-Bonsai-2-27B-gguf](https://huggingface.co/prism-ml/Ternary-Bonsai-2-27B-gguf)
  - MTP build: [sudoingx/Ternary-Bonsai-2-27B-PTQ1_0-MTP-GGUF](https://huggingface.co/sudoingx/Ternary-Bonsai-2-27B-PTQ1_0-MTP-GGUF)
  - Installer and serve scripts: [sudoingX/local-ai-on-your-gaming-gpu](https://github.com/sudoingX/local-ai-on-your-gaming-gpu)
  - Attribution requested by the maker (NOTICE.txt in the model repo): "Created using Bonsai by Prism ML."
  - Model license: Apache-2.0, from the license field of the [prism-ml/Ternary-Bonsai-2-27B-gguf](https://huggingface.co/prism-ml/Ternary-Bonsai-2-27B-gguf) and [sudoingx/Ternary-Bonsai-2-27B-PTQ1_0-MTP-GGUF](https://huggingface.co/sudoingx/Ternary-Bonsai-2-27B-PTQ1_0-MTP-GGUF) cards; checked 2026-11-02.
  - Runner licenses, checked 2026-11-02: the bonsai2 branch of the [sudoingx/llama.cpp](https://github.com/sudoingx/llama.cpp) fork has a LICENSE reading "MIT License, Copyright (c) 2023-2026 The ggml authors", and ggml-org/llama.cpp is MIT. The installer repo [sudoingX/local-ai-on-your-gaming-gpu](https://github.com/sudoingX/local-ai-on-your-gaming-gpu) has no LICENSE file; its README says Apache 2.0 (a README statement, not a detected license).
- Context size: 65536 (the first two passes used 131072)
- Thinking mode: off for the main tests; medium effort for some runs (listed below)
- Settings used:

| Setting | Value |
|---|---|
| GPU layers | all (board memory about 11 to 12 GB) |
| KV cache | q4_0 for K and V |
| MTP speed head | on for the main run; a second run used the file without it |
| Parallel slots | 1 |
| Sampling, thinking off | temp 0.7, top-p 0.8, top-k 20, min-p 0, presence penalty not set (server value 0); the Instruct row of the [Unsloth Qwen3.8-27B page](https://unsloth.ai/docs/models/qwen3.8) except presence penalty, which that row lists as 1.5. Also used with thinking medium in pass D |
| Sampling, as shipped by the installer | temp 1.0, top-p 0.95, top-k 20, presence penalty not set (server value 0); used for passes A and B and the thinking-medium PII runs. Follows the Thinking row of the [Unsloth Qwen3.8-27B page](https://unsloth.ai/docs/models/qwen3.8) except min-p: passes A and B left it at the server default 0.05 (reported by the server in pass B; pass A used the same flags), and the thinking-medium PII runs set it to 0 |

- Runs per test cell: 1 for the main tests; 5 repeats per cell for the PII tests

## Test description
- Speed: a short prompt asking for a long answer (T6); generation speed is the server-reported decode rate.
- Text battery: 22 graded cells: ten general cells (T1 to T10: exact JSON, a 20K-token context stress test, tool calls, a two-turn code task, memory, two PII redaction prompts), five long-context needle cells (L1 to L5, about 22K tokens) and seven structured-output cells (J1 to J7).
- T9 and T10 ask the model to "invoke" or "run" a `pii-redactor` skill on a short fictional text. No such skill, tool or system prompt exists in the request: this is a deliberate trap, and a good answer redacts the text anyway. See [assets/pii-pack.md](../assets/pii-pack.md).
- Grading uses a written key. A JSON answer wrapped in a code fence counts as a fail where bare JSON was asked for (T1, T3, T4, J4).

## Measured results
- Generation: 81.29 tok/s (server-reported, MTP head on; 73.6 at the client); 78.01 without the head
- 20,471-token prompt answered in 13.0 s (prefill about 1,605 tok/s); 12.0 s without the head
- Long context: the 60K-token pack took 48.6 to 49.2 s per request, with decode between 46.5 and 66.7 tok/s
- GPU memory in use after load: about 11.0 to 11.9 GB with the head and about 10.3 to 10.5 GB without it (board readings, at 65536 context)
- The MTP head is worth about 4% on a long answer (81.29 versus 78.01) and about 24% on short answers (median 96.7 versus 77.9 tok/s on the hard PII pack)
- Time to first token and thermals: not recorded for the main run

### Variants tried (one change at a time unless noted)

| Variant | What changed | Decode (tok/s) | Outcome |
|---|---|---|---|
| Pass A | thinking medium, shipped sampling, ctx 131072 | 88.77 | 13 of 13 requests clean; fastest 27B-class result on this card; cannot share the GPU with a second model |
| Pass B | thinking off, shipped sampling, ctx 131072 | 81.01 | 13 of 13 clean; wall-clock 5 to 6 times faster than thinking medium |
| Pass C (main run) | thinking off, series sampling, ctx 65536 | 81.29 | T9 collapsed to bare tags (partial) |
| Pass D | thinking medium, series sampling, ctx 131072 | 92.63 | Format clean; T9 restated the original values; the long answer took 38 s |
| Pass E | no MTP file, thinking off, series sampling, ctx 65536 | 78.01 | Same quality within noise; T7 and T9 partial |

The main tables below are pass C. The vision run used mainline b9682 (not the fork build) and passed 4, with 2 partial and 1 strict fail.

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
| T7 Two-turn code task | pass |
| T8 Memory within a conversation | pass |
| T9 PII redaction, "invoke the skill" wording (trap) | partial |
| T10 PII redaction, "run the skill" wording (trap) | pass |

**Long context, about 22K tokens**

| Cell | Result |
|---|---|
| L1 Needle early | pass |
| L2 Needle middle | pass |
| L3 Needle late | pass |
| L4 Two needles, order | partial |
| L5 Buried instruction | fail |

**Long context, about 60K tokens**

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
| J4 Prompt-only JSON (no response format) | pass |
| J5 Null not invented | pass |
| J6 Native tool call | pass |
| J7 Number words | pass |

**Vision (V1 to V7)**

| Cell | Result |
|---|---|
| V1 Scene and text | partial |
| V2 Chart to table | pass |
| V3 Counting shapes | partial |
| V4 Terminal screenshot | fail |
| V5 Invoice | pass |
| V6 No-image control | pass |
| V7 Two images | pass |

**Cells that were not a pass**

- T9 (partial): Five bare [REDACTED] tags, sentence dropped, no leak (partial: bare-tag collapse, confirmed 2026-10-29)
- about 22K tokens: L4 (partial): later_code FALCON-5580 right, but the entry numbers came back as 0130 and 0390 (leading zeros, not valid JSON numbers; the entries are 130 and 390).
- about 22K tokens: L5 (fail): one summary sentence, but it does not end with BANANA. Decode 76.0 to 80.6 tok/s over the pack; ctx 65536, q4_0 KV, series sampling, MTP head on.
- about 60K tokens: L5 (fail): one summary sentence, but it does not end with BANANA (the buried instruction was not followed). Each 60K request took 48.6 to 49.2 s, decode 46.5 to 66.7 tok/s; one run, MTP head on.
- V1 (partial): Both strings exact, scene right, but it invented 'thin black borders on the top and bottom' (same invented-edge-lines slip as Ornith UD-Q4_K_XL). Partial by the grading key.
- V3 (partial): red_circles 6 instead of 5; the other four counts exact; JSON in a code fence. One count off by one: partial.
- V4 (fail): error_type KeyError, missing_key window, failing_line 41 all right (innermost frame), but failing_file is 'srv/app/transform.py' with the leading slash dropped. The key's partial case is only the outer-frame trap, so a strict read of the key is a fail, though it is a near miss.

### PII side test (fictional data)

| Mode | Redaction requests (pass / partial / fail) | Tool-control requests correct |
|---|---|---|
| Thinking off | 30 requests: 26 / 4 / 0 | 20 of 20 |
| Thinking on, effort medium | 30 requests: 27 / 0 / 3 | 20 of 20 |

### Harder PII pack (two longer fictional texts, 30 requests per run)

| Mode | Pass / partial / fail / capped | Median decode (tok/s) | Minutes for 30 requests | Median tokens per answer |
|---|---|---|---|---|
| Thinking off, MTP head | 20 / 6 / 4 / 0 | 96.7 | 1.6 | 275 |
| Thinking off, no MTP head | 18 / 11 / 1 / 0 | 77.9 | 1.9 | 283 |
| Thinking on, effort medium, MTP head | 28 / 2 / 0 / 0 | 93.3 | 9.5 | 1528 |

Capped means the answer hit the 8,192-token limit set for the run and produced no answer; it is reported separately and not counted as a redaction failure. Grading rules are in [assets/pii-pack.md](../assets/pii-pack.md).

## Verdict under these settings
Usable: yes, with limits

The fastest 27B-class file tried on this card (about 80 to 90 tok/s) with a small memory footprint. Use it where speed matters and long-context instruction-following is not critical; for PII redaction use thinking medium (28 of 30 on the hard pack) and accept about 9.5 minutes per 30 requests against about 1.6.

## Caveats
- The weights are from Prism ML; the runner is a third-party fork of llama.cpp.
- Presence penalty was not set: the runs left it at the server default of 0, while the Unsloth page lists 1.5 for the Instruct row (it lists 0 for the Thinking row). The shipped-sampling passes also left min-p at the server default (see the settings table).
- L5 (end the summary with a given word) failed at 22K and at 60K tokens; the position cells L1 to L3 and the 60K L4 passed.
- Vision ran on a different build than the other tests.
- The five passes (A to E) differ in context, thinking and sampling at once, so compare them only as separate runs.
- One machine and one run per cell unless stated. In an earlier measurement on a different model (Qwen3.5-9B), about 2 of 23 grades changed between identical reruns, so a single grade difference is a data point, not an effect.
- Hardware, driver and runner build are the ones listed above; results on other cards or builds are not directly comparable.

## How to reproduce
1. Get the build and the files. The runs used the Windows CUDA package that the fork's installer and serve scripts ship ([sudoingX/local-ai-on-your-gaming-gpu](https://github.com/sudoingX/local-ai-on-your-gaming-gpu)); its `llama-server --version` reports 0.2.0-dev (build 10759, commit de947abc3). Download `Ternary-Bonsai-2-27B-PTQ1_0-mtp.gguf` from the [MTP repo](https://huggingface.co/sudoingx/Ternary-Bonsai-2-27B-PTQ1_0-MTP-GGUF) and the vision projector `Ternary-Bonsai-2-27B-mmproj-Q8_0.gguf` from the [Prism ML repo](https://huggingface.co/prism-ml/Ternary-Bonsai-2-27B-gguf). The run without the MTP head used `Ternary-Bonsai-2-27B-PTQ1_0.gguf` from the Prism ML repo.
2. Start the server, either with the fork's serve script or with the command used for the main runs (Windows Command Prompt; paths shortened):

   ```
   set GGML_CUDA_BATCH_INVARIANT=1
   <runner-dir>\bin\llama-server.exe -m <models-dir>\Ternary-Bonsai-2-27B-PTQ1_0-mtp.gguf --mmproj <models-dir>\Ternary-Bonsai-2-27B-mmproj-Q8_0.gguf -c 65536 -ngl 99 -fa on -np 1 -ctk q4_0 -ctv q4_0 --jinja --alias bonsai-2-27b-nothink --host 127.0.0.1 --port <port> --reasoning off --temp 0.7 --top-p 0.8 --top-k 20 --min-p 0.00 --spec-type draft-mtp --spec-draft-n-max 1
   ```

   For the run without the MTP head, point `-m` at the non-MTP file and drop `--spec-type` and `--spec-draft-n-max`. For thinking medium, replace `--reasoning off` with `--reasoning-effort medium`. The settings table above lists the context and sampling values for each pass.
3. Check the server before trusting any result: the model name echoed in each reply matches the alias, `finish_reason` is `stop`, and `cache_n` is 0 on the first request of a fresh server.
4. Send each test as a separate request with no system prompt. The test prompts for the main battery are not published yet; the PII prompts are in the [appendix](../assets/pii-pack.md).

`Status: Tested on llama.cpp fork build 10759 with mainline b9682 for the vision run, 29 Oct 2026`
