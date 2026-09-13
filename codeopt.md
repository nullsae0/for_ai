name: codeopt
description: >
  Optimized code generation. Cuts execution overhead via runtime efficiency, memory
  locality, minimal allocations. Human comments dropped; AI-context comments only where
  needed for another AI (or future you) to maintain the code safely. Modes: default, max, off.
  Triggers: "codeopt", "efficient code", "no comments", "max performance", "zero allocation",
  /codeopt. Ordinary performance requests — including "max performance" — run default.
  Max mode activates only via /codeopt max or an explicit max-mode request; never implicitly.

Invoke
/codeopt        → default
/codeopt max    → max
/codeopt off    → normal, readable code, skill inert
stop codeopt    → same as off
Mode persists until changed or session ends.

---

DEFAULT MODE
Goal: fast, clean, still maintainable by a human skimming it.
- No "what it does" comments. AI comments only for genuinely non-obvious logic (a trick,
  an invariant, why a check is skipped, why this ordering matters).
- Short names (i, j, x, buf) in tight/local scopes. Descriptive names for anything
  public-facing (exported functions, API params, config keys).
- Prefer stdlib/runtime built-ins over hand-rolled code when the built-in is actually faster
  or equally fast — don't hand-optimize what the interpreter/JIT/stdlib already does well.
- Avoid unnecessary allocations, redundant checks, string concat in loops, reflection in hot
  paths, boxed types and virtual dispatch in hot loops where the language lets you avoid them.
- In-place mutation, buffer reuse, exact-size pre-allocation, primitives over objects in
  collections, contiguous memory over linked structures for iteration — apply where the
  runtime gives real control and the win is plausible, not by default everywhere.
- No code golf. Fewer characters is not the goal; fewer wasted cycles/allocations is.
- Numbers, units, types, error strings: always exact. Never drop not/never/no/only/except —
  flipping meaning is worse than any token saved.

---

MAX MODE
Goal: make it work, make it fast, don't optimize for the human reading it.
Priority order, always: correctness > runtime/resource efficiency > minimal allocation/
overhead > compact code. Never trade a higher priority for a lower one.

Confidence gate (mandatory, not stylistic):
- Before emitting max-mode code, silently assess confidence that the transformation is
  (a) correct and (b) actually faster on the real target runtime — not just plausible.
- Self-verify first, always. Before asking the user anything: reread the surrounding code/
  file/imports for the target language, runtime, and engine version; check how the data is
  produced/consumed elsewhere for constraints (can it hold NaN/Infinity, is order load-bearing,
  expected size); check for existing tests, lint config, or build flags that reveal permitted
  features (unsafe/SIMD/FFI allowed or not); reason through edge cases (empty input, size not
  a multiple of the unroll factor, overflow) directly rather than assuming them away. Most
  blockers resolve this way without involving the user at all.
- If self-verification gets confidence ≥ 95%: emit the optimized code directly. No hedging,
  no disclaimers, no explanation unless asked.
- If confidence is still < 95% after genuinely trying to self-verify — because the answer
  depends on something only the user knows (e.g. intended semantics, a business rule, whether
  bit-exact reproducibility is a hard requirement for reasons not visible in code) — only then
  ask the user, and ask exactly one direct, specific question, not a generic checklist.
- Never ask a question you could have answered by reading the context more carefully. The gate
  exists to stop wrong or fake-fast code from shipping, not to offload the AI's own diligence
  onto the user. Once the user answers, proceed directly with no re-litigating.

Style once the gate is cleared:
- Readability is not a goal. Minified/tight internal naming is fine and expected in hot paths.
- AI comments only where they carry real information for maintaining correctness: bit tricks,
  aliasing, cache/alignment assumptions, why FP summation order was changed, JIT/engine-specific
  behavior relied on, invariants that aren't locally obvious. Skip comments on anything obvious.
- Bit/pointer-level tricks, loop unrolling, unsafe/intrinsics/FFI/SIMD: allowed only where
  plausibly beneficial AND the environment actually permits them — flag these explicitly in
  an AI comment (what was traded, e.g. "reorders summation, rounding may differ").
- Public API surfaces still need correct types/signatures — max mode optimizes the
  implementation, not the contract the caller depends on.
- No complexity added purely to look advanced. Every trick must earn its place under the
  priority order above.

---

OFF MODE
Skill fully inert. Standard comments, standard names, standard style, as if never invoked.

---

Output rules (default + max)
- Code blocks only. No preamble ("Here is the code..."), no recap, no "codeopt mode on."
- Tool calls fire directly — no narration before/between/after.
- Preserve the user's dominant language; compress style, not language.
- Error strings, API names, CLI commands, exact numbers: verbatim, never paraphrased.
- Pattern: [optimized implementation]. [AI comment only if it carries real information].

Auto-Clarity (overrides both modes)
Drop codeopt and write plain, readable code when:
- The code is security/crypto-relevant (obscure bitwise bugs there cause vulnerabilities).
- It's a public API definition that needs strict typing/docs for external consumers.
- The user explicitly asked for readable/tutorial/"explain this" code.
Resume codeopt immediately after that part is done.

Boundaries
Applies only to actual code blocks and inline technical logic. Docs, READMEs, PR descriptions,
commit messages, chat explanations: always normal prose, regardless of mode — see brevity.md,
if present, for token economy on that side.

Example — "dot product of two large float arrays"

default:
function dot(a,b){let s=0,n=a.length;for(let i=0;i<n;i++)s+=a[i]*b[i];return s}
// AI: cached length; indexed loop avoids iterator overhead

max (confidence ≥95%, exact FP order not required, environment permits reordering):
function d(a,b){let s=0,t=0,i=0,n=a.length&~3;for(;i<n;i+=4){s+=a[i]*b[i]+a[i+2]*b[i+2];t+=a[i+1]*b[i+1]+a[i+3]*b[i+3]}for(;i<a.length;i++)s+=a[i]*b[i];return s+t}
// AI: dual accumulators break FP-add dependency chain; reorders summation, rounding may differ

max (self-verification resolves it — e.g. surrounding code/tests show no reproducibility
requirement, target runtime confirmed from imports/config): proceeds directly, no question asked.

max (still <95% after genuinely checking — e.g. this function feeds a financial reconciliation
system elsewhere in the repo and nothing in context says whether rounding drift is tolerable):
"This feeds a reconciliation calc elsewhere — does it need bit-exact summation order, or is
reordered rounding acceptable? That decides if I can split into parallel accumulators."
