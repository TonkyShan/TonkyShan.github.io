---
title: Figuring Out Claude's Message Threads with a Local Mock
---

On November 2, the Claude API launched a beta feature called message threads.

## What it solves

If you have written an agent, you know that every call to the model resends the system prompt, the tool definitions and the whole conversation so far. Each tool call is another turn, so after a few dozen of them every request is large, and the server has to validate and tokenize the whole conversation again each time.

Message threads let the server keep the conversation. The first request sends the full conversation as usual, plus `"thread": {"type": "create"}`. After that, each turn sends only the new messages, along with the id of the previous reply.

There are two limits. A thread expires 5 minutes after its last request. And pricing doesn't change: it saves request size and time to first token, not tokens.

## Simulating it locally

I had Claude Code write a local mock of the API from the official docs, implementing the thread rules: create, continue, a 5-minute expiry, each id can be continued only once, and no system prompt or tools on a continue request. The clock can be moved by hand, so "6 minutes later" doesn't mean actually waiting.

Then I pointed the official Go SDK at the mock. The SDK has a helper object for threads that rebuilds a thread when it expires, so you can see exactly which requests it sends when something goes wrong.

## How much it saves

The simulated agent reads code: a system prompt of about 8 KB, and a little over 2 KB per grep result.

| Tool calls | Total sent without a thread | Total sent with a thread | Ratio |
|---|---|---|---|
| 5 | 110 KB | 28 KB | 3.9x |
| 20 | 993 KB | 86 KB | 11.5x |
| 50 | 5.4 MB | 0.2 MB | 26.6x |

Without a thread, every turn resends an ever longer history, so the total grows with the square of the number of turns; with a thread it grows linearly. The amount the server has to process follows the same ratios.

These numbers are bytes sent and content processed, not latency. How much faster the first token really arrives can only be measured against the real API; the official wording is "lower and steadier".

## When something goes wrong

Next, six scenarios with deliberate faults:

| Scenario | What the SDK does |
|---|---|
| Waiting 6 minutes for an approval, the thread expires | The continue gets a 404; it rebuilds from the full local conversation and carries on continuing |
| The server answered but didn't store the turn | No sign of it at the time; the next continue gets a 404, then it rebuilds |
| The same id was already continued elsewhere | It gets an "already continued" error and rebuilds |
| The system prompt changed mid-thread | It fails locally without sending a single request |
| The model changed mid-thread | The request goes out, the server returns a 400, and the SDK passes it to the caller |
| The model refused | The refused turn is dropped, and the next turn starts a new thread |

The second scenario is worth a closer look. If, say, a tool call the model was writing gets cut off by max_tokens, the API returns the result as usual but doesn't store that turn in the thread. Nothing in the response says so; you only get the 404 when you try to continue on the next turn. A request that succeeded doesn't mean the state was saved.

## How the SDK is written

The Go SDK's thread code is only 654 lines, and the structure is simple.

It keeps a full copy of the conversation locally, and tracks which part of it has already been sent to the server; each turn sends only the difference. When something fails it only steps down, in one direction: first it tries to continue; if that fails, it rebuilds from the full local conversation; if the thread still can't be found after that, it stops using threads and sends plain requests. Because it never steps back up, a turn sends at most 3 requests and never flips between the two modes.

After a refusal, a stream that broke off midway, or a reply from a fallback model, the next turn starts a new thread straight away, because the client can't be sure what the server stored.

The official docs put it this way: "Your application stays the source of truth for the conversation." What the server keeps is only there to speed things up and can disappear at any time; the client has to be able to rebuild from its own data.

## When to use it

For tool-call loops that run without stopping, the more turns, the bigger the gain.

If the flow waits for a human approval or an external callback, the thread has to be rebuilt once 5 minutes pass, so it will expire almost every time, and using it just adds a round trip. It also doesn't support batches, token counting or context compaction, and it can't be used where zero data retention is required.
