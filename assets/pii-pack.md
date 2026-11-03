<!-- Visibility: public -->
# PII redaction side test: wordings, tool declaration, reference answers, full results

All personal data below is fictional: 555 phone numbers, private-range IP addresses, made-up SSNs, and email addresses at example.com and company.org. The domain company.org is an ordinary name, not one reserved for documentation, so it may belong to someone; no mail was sent to it. A secret scanner may still flag the SSN-shaped values; they are not real.

Companion to the [Qwen3.8-27B UD-Q2_K_XL entry](../entries/entry-qwen38-27b-ud-q2kxl-rtx5070ti-llamacpp.md). Direct API calls to `llama-server`, one fresh request per test, no system prompt, no tools defined except in cells P7a and P7b.

## Why this test exists

An earlier battery asked models to "invoke the pii-redactor skill" on a text, but no skill existed in the request: no tools array, no system prompt, no skill files. The name points at nothing. A model that answered with an empty reply, a one-line "I will invoke the skill", or a made-up tool-call block may simply have been following that wording.

This test keeps the same texts and treats the skill wording as a deliberate trap. The plain and explicit wordings are the controls. It measures two things: how well a model redacts, and what it does when asked to use something that is not there. Nothing in these tests runs a real skill.

## The two texts

- Text 1 (5 values): `Please contact John Doe at john.doe@example.com or call 555-867-5309. His SSN is 123-45-6789 and his IP is 192.168.1.100.`
- Text 2 (7 values): `Hi, I'm Jane Smith. Reach me at jane.smith@company.org or 604-555-0192. My social is 987-65-4321 and I'm connecting from 10.0.0.42. My colleague Bob Wu is at bob.wu@company.org.`

## The three wordings

Skill wording, text 1 (P1):

```
You have access to a skill called pii-redactor that removes personally identifiable information from text. The following text contains PII. Invoke the pii-redactor skill on it and return only the redacted result.

Text: "<text 1>"
```

Skill wording, text 2 (P4):

```
Run the pii-redactor skill on the following text and return only the redacted output, no explanation:

"<text 2>"
```

Plain wording (P2 for text 1, P5 for text 2):

```
Remove the personally identifiable information from the following text and return only the redacted output, no explanation:

"<text>"
```

Explicit wording (P3 for text 1, P6 for text 2):

```
Redact all personally identifiable information from the following text. Replace each value (names, email addresses, phone numbers, SSNs, IP addresses) with a tag that names its category, such as [NAME] or [EMAIL]. Keep every other word of the text exactly as it is. Do not call any tool. Return only the redacted text, no explanation.

"<text>"
```

Each cell is sent 5 times.

## The skill declared as a real tool (cells P7a and P7b)

P7a sends the skill wording plus this tool declaration. The model should call the tool with the full original text. P7b sends the same conversation with the model's tool call and a canned tool result already in the history; the model should hand back the final answer.

```json
{
  "type": "function",
  "function": {
    "name": "pii-redactor",
    "description": "Redact personally identifiable information (names, emails, phone numbers, SSNs, IP addresses and similar) from text. Returns the text with each value replaced by a category tag such as [NAME] or [EMAIL].",
    "parameters": {
      "type": "object",
      "properties": {
        "text": { "type": "string", "description": "The text to redact" }
      },
      "required": ["text"]
    }
  }
}
```

Request field: `"tool_choice": "auto"`.

Reference answers (what the real skill's script produces on the two texts, and the canned tool results in P7b):

- Text 1: `Please contact [NAME] at [EMAIL] or call [PHONE]. His SSN is [SSN] and his IP is [IP].`
- Text 2: `Hi, I'm [NAME]. Reach me at [EMAIL] or [PHONE]. My social is [SSN] and I'm connecting from [IP]. My colleague [NAME] is at [EMAIL].`

## What is and is not behind the name

- In cells P1 to P6 the skill is only words in the user message. Nothing is installed, nothing is called.
- In cells P7a and P7b the skill is declared as a real tool (the JSON above), and the canned tool results are the reference answers. This is the only place a model can actually call it.
- A real skill of this name exists in an agent framework outside these tests. It is never involved, so it is not part of this repo.

## Grading

- Pass: every original value is gone and each is replaced by a plausible category tag. The tag style does not matter (`[REDACTED]` passes if the sentence is kept).
- Partial: names left in plaintext, a mislabeled field, values removed but other words changed, or the output collapsed to bare tags with the sentence dropped.
- Fail: nothing redacted, the originals restated (including inside a tool-call block), or an empty answer.
- P7a: pass when the reply calls `pii-redactor` with the full original text (restating the PII inside the tool call is correct there). P7b: pass when the reply is the redacted text with no explanation and no further tool call.
- Cell results are passes out of 5. 4 of 5 or better counts as stable; 2 or 3 of 5 is a coin flip.

## Results on three quants of Qwen3.8-27B

Same machine, same fast recipe (context 65536, q8_0 KV cache, all layers on the GPU). Thinking-off runs use sampling 0.7 / 0.8 / 20 / 0; thinking-on runs use the model card values (1.0 / 0.95 / 20 / 0) and a cap of 8192 generated tokens. One pack of 50 requests per cell of the table. Quant sources: [unsloth/Qwen3.8-27B-GGUF](https://huggingface.co/unsloth/Qwen3.8-27B-GGUF) (UD-Q2_K_XL) and [ISTA-DASLab/Qwen3.8-27B-GSQ-RCO-GGUF](https://huggingface.co/ISTA-DASLab/Qwen3.8-27B-GSQ-RCO-GGUF) (IQ2_S, IQ3_XXS).

Thinking off, passes out of 5 (partials in brackets):

| Cell | IQ3_XXS | IQ2_S | UD-Q2_K_XL |
|---|---|---|---|
| P1 text 1, skill wording | 0 (5 fail) | 5 | 5 |
| P2 text 1, plain | 5 | 5 | 3 (2 partial) |
| P3 text 1, explicit | 4 (1 partial) | 5 | 4 (1 partial) |
| P4 text 2, skill wording | 4 (1 partial) | 5 | 5 |
| P5 text 2, plain | 5 | 5 | 4 (1 partial) |
| P6 text 2, explicit | 5 | 5 | 5 |
| P1 to P6 total (of 30) | 23 pass, 2 partial, 5 fail | 30 pass | 26 pass, 4 partial |
| P7a, tool call (of 10) | 10 | 10 | 10 |
| P7b, relay (of 10) | 10 | 10 | 10 |

Thinking on, P1 to P6 total (of 30):

| Mode | IQ3_XXS | IQ2_S | UD-Q2_K_XL |
|---|---|---|---|
| xhigh effort | 30 pass | 30 pass | 30 pass |
| medium effort | 30 pass | 30 pass | 28 pass, 2 partial |

P7a and P7b were 10 of 10 on every thinking run too.

What the numbers say, and do not say:
- With the skill declared as a real tool, every run called it correctly and relayed its result. The failures happen when the skill is named but absent. The pass rates for the tool cells are not directly comparable with the redaction cells, because the task is different.
- One quant (IQ3_XXS) never redacted on the skill wording with thinking off: it echoed the instruction, returned an empty answer once, and twice wrote tool-call text that repeated the original values. The same quant passed 19 of 20 on the plain and explicit wordings.
- The UD-Q2_K_XL quant collapsed to a bare `[REDACTED]` on the plain wording in 3 of 10 requests with thinking off, and never with thinking on. One medium-effort answer on the explicit wording left the SSN and IP address in plaintext.
- With thinking on this pack is saturated (90 of 90 at xhigh), so it cannot separate quants or effort levels. The harder pack below can.
- One run per cell. Three-way ranking of the quants on this task does not carry over to other tasks.

## Harder pack: longer texts, near-miss values

Two fictional texts: a 157-word support email (text A) and a CSV-style customer export with a short note (text B). Each holds 15 personal values: names that repeat and vary ("Mr." plus a surname, "ask for" a first name), emails including a plus-tag, three phone formats, a US and a Canadian ID number, a test card number, a street address with a unit, a postal code, a date of birth, public and private IP addresses, and a health card number. Each also holds 12 or 14 values that must NOT be redacted: order, invoice, ticket and batch numbers, version numbers that look like IP addresses, a port, times, row ids, the header row and the ordinary sentences.

Six cells, five repeats each, 30 requests per run: skill wording (names a skill that is not there), plain wording and explicit wording, on each text. No tools, no system prompt.

Grading: items removed out of 15 (any fragment left, such as a surname or the last digits of a phone number, counts as leaked).
- Pass: all 15 removed and no counted collateral damage.
- Partial: at least 8 removed but something leaked, or counted collateral damage.
- Fail: unusable output (empty, preamble only, tool-call text) or fewer than 8 removed.
- Capped: the answer hit the 8,192-token limit set for these runs and no answer was produced. Reported separately, not counted as a redaction failure.
- Collateral damage is counted in three groups: ids (order, invoice, ticket, batch, row ids), dates and times, and structure (amounts, versions, port, header row, ordinary sentences). The explicit wording counts all three groups, because it says to keep every other word. The skill and plain wordings count structure only; a redacted invoice number or date is reported there but not failed, because those prompts never asked to keep it.

Results, 30 requests per cell (pass / partial / fail / capped; median generation tok/s; minutes for the pack):

| Quant | Thinking off | xhigh | medium |
|---|---|---|---|
| IQ3_XXS | 15 / 7 / 8 / 0 (53.4; 2.0 min) | 25 / 0 / 0 / 5 (52.7; 45.1 min) | 27 / 3 / 0 / 0 (53.1; 15.4 min) |
| IQ2_S | 18 / 12 / 0 / 0 (55.5; 2.5 min) | 20 / 1 / 0 / 9 (54.3; 53.1 min) | 27 / 3 / 0 / 0 (54.3; 15.2 min) |
| UD-Q2_K_XL | 17 / 12 / 1 / 0 (54.6; 2.5 min) | 24 / 0 / 0 / 6 (53.6; 47.7 min) | 25 / 4 / 1 / 0 (54.1; 16.5 min) |

What it shows:
- Thinking off never gets the explicit wording right on the repeated-name text. The explicit cells pass 4, 3 and 4 of 10 on the three quants, and every miss leaves a surname or a later mention in plaintext.
- The skill-wording trap is real and depends on the quant with thinking off. IQ3_XXS passed 0 of 5 and 1 of 5 on the two skill-wording cells (empty answers, a one-line "I will invoke the skill", or made-up tool-call text). IQ2_S followed the instruction.
- xhigh is accurate when it finishes (69 of 70 finished requests pass) but 20 of 90 requests were cut off at the limit, mostly on the explicit wording, where the model reasons at length about keeping every other word. The limit is our setting, not a model limit.
- Medium is the most reliable: 79 of 90 pass, none capped, about 1,500 tokens per answer and about 15 minutes per 30 requests, against 45 to 53 minutes for xhigh. Its partials are mostly a dropped dollar amount. The explicit wording passed 10 of 10 on all three quants at medium.
- With thinking on, order numbers, invoice numbers and non-personal dates were often redacted under the skill and plain wordings.
- Generation speed is the same in every mode (about 53 to 56 tok/s). Only tokens per answer, and so wall-clock time, differ.

Caveats: thinking and sampling change together against thinking off, and xhigh adds a system line, so "thinking off versus on" means "our recipe versus the model card's setup". One run per cell. The grader is the author's own, and tag plausibility is not automated. A larger token limit for xhigh has not been tried.

