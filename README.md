# for_ai

Two lightweight instruction files that make AI coding/chat assistants (Claude, and
likely others that support similar skill/instruction files) cut wasted output —
without cutting the substance.

Drop the folder in, the assistant reads it, output gets leaner. That's it.

## What's in here

| File | Applies to | Modes |
|---|---|---|
| [`codeopt.md`](./codeopt.md) | Code blocks only | `default`, `max`, `off` |
| [`brevity.md`](./brevity.md) | Prose/chat replies only | `on` (default), `off` |

They're independent — use either, both, or neither. Neither touches the other's
domain: codeopt never rewrites your explanations, brevity never rewrites your code.

### codeopt

Writes code the way a compiler backend would optimize it: standard-library and
built-in primitives first, no filler comments, short names in tight scopes, no
unnecessary abstraction. `max` mode adds low-level tricks (bit ops, unrolling,
unsafe/SIMD where the runtime allows it) — but only after the model has actually
checked the surrounding code/context for correctness and verified the optimization
is real, not just plausible. If it can't verify confidently from context, it asks
one specific question instead of guessing.

Auto-drops to normal, readable code for security/crypto, public APIs, and explicit
"explain this" requests — compression never overrides correctness or clarity where
either actually matters.

### brevity

Cuts hedging, pleasantries, restated context, and padding from explanations and
chat replies. On by default. Never cuts the actual reasoning, caveats, warnings, or
anything that changes what you should do or believe — it targets words that carried
no information, not information itself.

*(Naming note: "brevity" is not a reference to "Verity" or any related trend —
just the plain English word for what it does.)*

## Does this actually save tokens?

Tested with a batch of 8 mixed prompts (explanations + code), same wording, one
continuous session, skills-on vs. skills-off. Token counts verified with
[OpenAI's tokenizer](https://platform.openai.com/tokenizer) (tiktoken, cl100k_base):

- **Output tokens:** 953 vs. 2,768 — a **65.6% reduction** in response length.
- **Fixed cost:** loading both files costs ~2,400 input tokens, paid once per
  session (not per message).

The catch: that fixed cost has to be earned back. In this test, breakeven landed
around message 10–11 in a single session — after that point, every further
question in the same session is close to pure savings. In short sessions with only
a couple of messages, the fixed cost can outweigh what compression saves. This
scales with how verbose your unassisted responses would've been to begin with —
terser topics or an assistant that's already concise will see smaller gains.

Net: **built for long sessions with genuinely verbose-prone content** (explanations,
code, multi-part questions) — not for one-off quick questions.

### What about accuracy?

Spot-checked the same 8-prompt batch for correctness, not just length — same
working code, same core technical points, nothing dropped that was actually asked
for. Not a rigorous eval, and only tested on well-known textbook-style problems —
harder or ambiguous prompts, where a compressed answer is more likely to drop a
real caveat, haven't been tested yet. Treat this as a promising early sign, not a
guarantee.

## Install

Drop `codeopt.md` and `brevity.md` into a folder the assistant reads at session
start (for Claude: a skills directory, or a folder path referenced in your project
setup). No build step, no dependencies.

## Usage

```
/codeopt              # code: default mode
/codeopt max           # code: max mode (confidence-gated)
/codeopt off            # code: plain, readable

/brevity off            # prose: normal, unabbreviated
/brevity on              # prose: back on (default)
```

## Credit

Drafted collaboratively with Claude (Anthropic). Design decisions, testing, and
scope calls are the author's; drafting and iteration were done with AI assistance.

## License

MIT — do whatever you want with it.
