---
name: chatgpt-use
description: Ask ChatGPT (the user's own web subscription) a question from the command line, with local files uploaded as real attachments. Use when the user wants a second opinion, a review, an explanation, or research from GPT ("问一下 ChatGPT", "让 GPT 看看", "ask ChatGPT", "second opinion"), or wants their paid ChatGPT web subscription used instead of an API key. Runs one CLI round trip per question — you stay the agent loop; ChatGPT only answers.
allowed-tools: Bash(chatgpt-use:*)
---

# chatgpt-use — ChatGPT as a question-answering tool

One CLI call asks ChatGPT a question and prints its answer. **You remain the
agent**: read files, run commands, edit code, decide what happens next. ChatGPT
only answers what you ask it. Never delegate a whole task to it and never let it
drive a loop.

Requires a Chrome profile already logged in to chatgpt.com. No API key, no token.

## The only call you need

```sh
chatgpt-use ask "<question>" [--attach <file> ...] [--output-schema <file>] --no-stdin
```

- `--no-stdin` is important when you are **not** piping input: without it the
  command waits on stdin forever.
- The answer goes to **stdout**. Progress and warnings go to **stderr**.
- Exit code is non-zero when the turn failed; check it.

### Giving ChatGPT a file

`--attach` uploads the file through ChatGPT's own pipeline — PDF layout,
spreadsheet cells and image pixels survive, and non-UTF-8 binaries work. It is
the **only** way `ask` takes files.

```sh
chatgpt-use ask "Summarise the thermal limits in this datasheet" --attach spec.pdf --no-stdin
chatgpt-use ask "What changed between these boards?" --attach rev_a.png --attach rev_b.png --no-stdin
```

At most 10 files per message. Every file must finish uploading before the
message is sent, so a failure means **nothing was asked** — the call returns an
error and it is safe to retry.

Text you already have in the prompt does not need a file: put it in the question
itself (careful, it costs the prompt budget) or pipe it on stdin.

### Structured answers

Pass a JSON Schema and ChatGPT is asked to answer in that shape:

```sh
chatgpt-use ask "Review this diff" --attach diff.patch \
  --output-schema review.schema.json --no-stdin
```

Use this when you need to parse the answer — otherwise you are scraping prose.

## Typical uses

```sh
# Second opinion on a design decision
chatgpt-use ask "Is a 10ms control loop achievable on this hardware? $CONTEXT" --no-stdin

# Review a document or image you cannot read well yourself
chatgpt-use ask "What is wrong with this schematic?" --attach board.png --no-stdin

# Research a question you cannot answer from the repo
chatgpt-use ask "What are the tradeoffs of EKF vs Madgwick for IMU fusion?" --no-stdin
```

## Delegation modes

For a plan / review / debug / research verdict packet:

```sh
chatgpt-use ask "<task>" --mode plan|review|debug|research --attach <file> [--json] --no-stdin
```

These still answer once; the caller stays in control.

## What NOT to use

- **`chatgpt-use mcp`** — hands ChatGPT file and shell tools on this machine.
  Not needed for question answering, and expands what ChatGPT can touch.
- **`chatgpt-use work` / `--loop`** — runs ChatGPT as the agent loop. That is the
  opposite of the rule at the top of this file.
- **`serve`** — long-lived process; a CLI call per question is simpler and leaves
  nothing running.

## When a call fails

| stderr says | Meaning |
|---|---|
| `sign in` / no composer | The Chrome profile is logged out of chatgpt.com |
| rate-limited / `Too many requests` | Pause. One page load costs ~45 backend requests. |
| reject / unsupported type | ChatGPT refused that file type or size |
| timed out waiting for … to upload | The attachment never settled; nothing was sent |

Do not retry a rate-limit or throttle message in a loop — that is what causes it.

## Cost

Each question is one ChatGPT web turn on the user's subscription. It is not
free and not instant: expect tens of seconds. Batch related questions into one
prompt rather than asking five small ones.
