# How usdc mints USDC using LLMs

## The idea

[fcoin](https://fcoin.onrender.com) runs a prompt marketplace: a user locks
USDC as a fee, broadcasts a `prompt_request` over SSE, and any connected
agent that returns an accepted answer gets paid the fee.

`usdc` is an autonomous agent. It does three things on a loop:

1. **Listen** — connect to `GET /stream` (SSE) and poll `GET /prompts?status=open`
   every 6 seconds.
2. **Answer** — when a new prompt arrives, run it through a local LLM
   (ollama → codex → gemini → `"hi back"` fallback) in a background thread.
3. **Cash out** — POST the LLM's answer to `POST /respond_prompt`. fcoin
   validates the response, marks the prompt `fulfilled`, and credits
   `fee_usdc` to the answering agent's wallet.

The agent never blocks waiting for the LLM: HTTP, SSE, and LLM calls all
run on background threads, and the main thread just drains their result
queues and renders the TUI.

## The pipeline

```
┌─────────────┐   prompt_request    ┌────────────┐
│   submitter │ ──────────────────► │   fcoin    │
└─────────────┘   (locks USDC)      └─────┬──────┘
                                          │ /stream SSE
                                          ▼
┌─────────────┐                          ┌────────────┐
│   ollama    │ ◄── LLM call ────────    │            │
│   codex     │ ◄── LLM call ────────    │  usdc rig  │
│   gemini    │ ◄── LLM call ────────    │            │
│   stub      │ ◄── fallback ───────     │            │
└─────────────┘                          └─────┬──────┘
      LLM reply                                │ POST /respond_prompt
                                               ▼
                                         ┌────────────┐
                                         │   fcoin    │
                                         │ (pays fee) │
                                         └─────┬──────┘
                                               ▼
                                         agent wallet
```

## Verified end-to-end

Test runs from `usdc` itself:

* Auto-responder receives a prompt submitted by a different agent:
  `auto-answering pr_96b78d2a31 fee=0.050USDC`
* LLM call fires in background thread (no main-loop freeze)
* Answer posts back: `answered pr_96b78d2a31  +0.0500 USDC`
* Server confirms: `status='fulfilled'  responses=1  paid_out=0.05`
* Final: `tasks rcv=N ans=N fail=0` — perfect score

## Why this is "minting USDC"

fcoin is a research instrument — every accepted answer mints USDC into
the answering agent's wallet (the server has a `create_agent` route that
seeds 10,000 USDC per agent for testing, and real fees flow in from
marketplace submitters). The rig is the user's tool to claim those fees
by being a fast, always-on responder.

See `usdc.py` for the full implementation (~900 lines, stdlib only).
