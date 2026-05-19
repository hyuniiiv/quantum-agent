---
name: cosmos:spawn
description: QuantumAgent — Spawn N parallel cosmos agents in isolated git worktrees, each tackling the same goal with a different strategy. Agents share live discoveries through Quantum Memory (.quantum/*.jsonl) — entanglement without external infrastructure.
---

# cosmos:spawn

Spawn N parallel cosmos agents to explore a goal using different strategies.
Each cosmos runs in its own git worktree. Agents write insights to
`.quantum/<name>/insights.jsonl` in real time and read each other's insights
between implementation steps — achieving live entanglement without external
infrastructure.

## Trigger

`/cosmos spawn --goal "<goal>" --strategies "<strategy1,strategy2,...>"`

## Execution Steps

### Step 1 — Parse arguments

From the user's trigger message, extract:
- `--goal` value → the shared objective
- `--strategies` value → comma-separated; split into a list

Assign cosmos names in order: `alpha`, `beta`, `gamma`, `delta`, `epsilon` (max 5).

Example: `--strategies "jwt,session,oauth2"` → alpha=jwt, beta=session, gamma=oauth2

**Pauli Exclusion check:** If any two strategies are identical (case-insensitive), stop immediately:

```
❌ Pauli Exclusion violated: "<strategy>" appears more than once.
   No two cosmos can occupy the same state. Each strategy must be distinct.
```

**Model diversity (v3.3):** Parse `--models <m1,m2,...>` if present. The list must have the same length as `--strategies`. Each cosmos uses its corresponding model — this enables *asymmetric token economics* (cheap model for exploration, expensive for analysis) and *blind-spot mitigation* (different models reduce shared bias).

Allowed model names depend on your Claude Code installation; common values:
- `claude-haiku-4-5` — fastest, cheapest, good for fan-out exploration
- `claude-sonnet-4-6` — balanced (default if `--models` omitted)
- `claude-opus-4-7` — most capable, best for final synthesis

Example: `--strategies "jwt,session,oauth" --models "haiku,sonnet,opus"` assigns alpha=haiku, beta=sonnet, gamma=opus.

If `--models` is omitted, all cosmos use the inherited default model. Length mismatch is rejected:
```
❌ --models length (N) must match --strategies length (M).
```

**Entanglement mode:** Parse `--entanglement <mode>` if present. Capture as `<entanglement_mode>` for use in Steps 5 and 6. Allowed values:

| Mode | Behavior |
|------|---------|
| `none` | Cosmos do NOT read other cosmos insights. Pure independent exploration. |
| `passive` *(default)* | Cosmos read other insights between major implementation steps. Current behavior. |
| `active` | Cosmos read AND must record `read_from: cosmos:<source>` when adopting another cosmos's pattern. |
| `strict` | Heartbeat protocol — each cosmos publishes `heartbeat` per major step, and MUST write `heartbeat-ack` for every unacknowledged heartbeat from other cosmos before its next step. Verifiable live entanglement. |

If `--entanglement` is omitted, default to `passive`. Reject unknown values:
```
❌ Unknown entanglement mode: "<mode>".
   Allowed: none | passive (default) | active | strict
```

### Step 2 — Detect repo root

Run:
```bash
git rev-parse --show-toplevel
```

Store the output as `<repo_root>`. All absolute paths below use this.

### Step 2.1 — Start live dashboard

Start the QuantumAgent live dashboard in the background before launching cosmos agents, so insights are visible in real time:

```bash
if [ -f "<repo_root>/tools/dashboard.js" ]; then
  if [[ "$OS" == "Windows_NT" ]]; then
    cmd //c start //b node "<repo_root>/tools/dashboard.js"
  else
    nohup node <repo_root>/tools/dashboard.js </dev/null >/dev/null 2>&1 &
    disown $!
  fi
fi
```

The server listens on port 3141 and auto-opens http://localhost:3141 in the default browser. If already running (`EADDRINUSE`), it opens the browser and exits silently — no duplicate process.

---

### Step 2.5 — Load macro-scale context (optional)

**Project Spin** — read if exists:
```bash
[ -f <repo_root>/.quantum/project/spin.json ] && cat <repo_root>/.quantum/project/spin.json
```

If present, parse as JSON and capture: `name`, `type`, `description`, `immutable_constraints`. These define the project's quantum identity — every cosmos inherits them as invariant constraints. A strategy that violates a project spin constraint is not exploring the goal; it is exploring a different problem.

**Active Singularities** — read if exists:
```bash
[ -f <repo_root>/.quantum/singularities/events.jsonl ] && cat <repo_root>/.quantum/singularities/events.jsonl
```

Parse each line as JSON: `{name, ts, trigger, invalidates, description}`. Sort by `ts` ascending. These are project-level events that reshape the solution space — patterns from before a singularity's `ts` may be stale. Cosmos must operate in the post-singularity era.

If neither file exists: continue silently. Macro context is optional. A project without declared spin or singularities runs as a "free" multiverse with no inherited constraints — useful for greenfield exploration.

### Step 3 — Create quantum memory directories

For each cosmos `<name>`:

```bash
mkdir -p <repo_root>/.quantum/<name>
touch <repo_root>/.quantum/<name>/insights.jsonl
```

### Step 4 — Create git worktrees

For each cosmos `<name>`:

```bash
git worktree add <repo_root>/cosmos/<name> -B cosmos/<name>
```

If the branch already exists, append `--force`.

Verify with:
```bash
git worktree list
```

### Step 5 — Write CLAUDE.md into each worktree

For each cosmos `<name>` with strategy `<strategy>`, write the following to
`<repo_root>/cosmos/<name>/CLAUDE.md`:

~~~markdown
# cosmos:<name>

## Goal
<goal>

## Strategy
<strategy>

## Project Spin (if spin.json defined)
This project's immutable quantum identity — your strategy must operate within these:
- **Name:** <name>
- **Type:** <type>
- **Description:** <description>
- **Immutable constraints:**
  - <constraint 1>
  - <constraint 2>

You cannot violate these. They are not optional tradeoffs. If your strategy requires violating a project spin constraint, you are exploring the wrong problem.

## Active Singularities (if events.jsonl had any)
The following project-level events have reshaped current context:
- **<name>** (<ts>) — <description>
  - Trigger: <trigger>
  - Invalidates: <invalidates>

Insights or patterns from before these singularities may be stale. Treat with caution. The "current era" is defined by the most recent singularity.

## Entanglement Mode: <entanglement_mode>

**Mode-specific behavior:**
- `none` — You are in independent mode. Do NOT read other cosmos insight files. Pure parallel exploration without cross-influence. Skip the "Entanglement" subsection below.
- `passive` *(default)* — Read other cosmos insights between major implementation steps. Adopt patterns through your own strategic lens. This is the standard quantum behavior.
- `active` — Read other cosmos insights AND record citation in any derived insight. Whenever you adopt or react to another cosmos's finding, your insight MUST include `read_from: cosmos:<source>` and reference the originating ts. Used when traceability matters (security, compliance, debugging).
- `strict` — Heartbeat protocol enforced. At every major step boundary you MUST:
  1. Write a `heartbeat` insight: `{"type":"heartbeat","step":<N>,"content":"<name> at step <N>","ts":"..."}`
  2. Read all other cosmos heartbeats published since your last ACK
  3. For every unacknowledged heartbeat from another cosmos, write `{"type":"heartbeat-ack","content":"acknowledged <other>:step <M>","refs":["<other>@<ts>"],"ts":"..."}`
  4. ONLY then proceed with the next implementation step

  Strict mode produces a verifiable entanglement graph — `/cosmos observe` flags any unacknowledged heartbeat as a broken entanglement channel. Used when race conditions, distributed system semantics, or compliance traceability require *proof* of live agent-to-agent communication.

## Quantum Memory Rules

### Writing insights
After every significant decision or discovery (new file created, design decision
made, key pattern found, bug discovered), append one line to your insights file.

Your insights file (absolute path):
  <repo_root>/.quantum/<name>/insights.jsonl

Format — one JSON object per line, with a `type` field for cross-agent interop:
```
{"type": "<type>", "content": "<insight text>", "ts": "<ISO 8601 e.g. 2026-05-12T12:00:00Z>"}
```

**Allowed `type` values** (pick the one that fits; default to `discovery`):

| type | When to use |
|---|---|
| `discovery` | General finding, new file, pattern noticed, fact uncovered |
| `decision` | Chose an approach, library, data structure, or API shape |
| `blocker` | Stuck — describe what's blocking and what you tried |
| `tunnel` | Found a solution bypassing a constraint you assumed was hard (was `[TUNNEL]`) |
| `jump` | Reading another cosmos caused a discontinuous architectural shift (was `[JUMP]`) |
| `resonance` | Noticed agreement with another cosmos on a pattern/decision |
| `complete` | Implementation done, ready to be considered for crystallize |

Use Bash append ONLY (never use the Write tool on this file — it overwrites).
Append is the natural atomic operation on POSIX for sub-PIPE_BUF writes:
```bash
echo '{"type":"discovery","content":"...","ts":"2026-05-12T12:00:00Z"}' >> <repo_root>/.quantum/<name>/insights.jsonl
```

> **Concurrency note**: if you spawn sub-agents that may write to the SAME
> insights file concurrently, wrap the append with `flock` (Linux/macOS) or
> write to a temp file then `mv` atomically. A single agent appending
> sequentially does not need this — POSIX guarantees atomic small appends.

### Entanglement (REQUIRED)
After EACH major implementation step, read all cosmos insight files:

  <repo_root>/.quantum/alpha/insights.jsonl   (skip if missing)
  <repo_root>/.quantum/beta/insights.jsonl    (skip if missing)
  <repo_root>/.quantum/gamma/insights.jsonl   (skip if missing)
  <repo_root>/.quantum/delta/insights.jsonl   (skip if missing)
  <repo_root>/.quantum/epsilon/insights.jsonl (skip if missing)

If another cosmos is converging on the same pattern → that is a Resonance signal.
Note it in your next insight. You may adopt it.

If another cosmos found a bug or edge case → read it carefully. Apply the fix if
it applies to your implementation too.

**Final entanglement read (REQUIRED):** Before marking your implementation complete,
perform one last read of all cosmos insight files. If new bug fixes or security
findings have appeared since your previous read, apply them and write a final
insight before stopping.

### Spin Preservation (pre-action principle)
Your strategy is your spin — an immutable intrinsic property. It defines which
direction you explore. You may adopt patterns from other cosmos via entanglement,
but you must reconstruct them through your own strategic lens. You cannot change
your core approach mid-run.

Decoherence is what happens when you violate Spin Preservation: you wholesale copy
another cosmos's implementation and lose strategic independence. `/cosmos observe`
will flag this. Influence without convergence — that is entanglement.

### Quantum Tags (REQUIRED)
Set `type` accordingly when the condition applies:

**`type: "tunnel"`** — you found a solution that bypasses a constraint you assumed was hard.
Example: `{"type":"tunnel","content":"Redis sorted sets eliminate the need for a separate rate-limit table entirely","ts":"..."}`

**`type: "jump"`** — reading another cosmos's insight caused a discontinuous architectural shift in your approach (not gradual adaptation — a sudden leap to a qualitatively different solution).
Example: `{"type":"jump","content":"Switched from polling to event-sourcing after reading alpha's insight on audit trail requirements","ts":"..."}`

> Legacy compatibility: older insights without a `type` field, or with
> `[TUNNEL]`/`[JUMP]` text prefixes in `content`, must still be readable.
> Treat missing `type` as `discovery`. Text prefixes can be recognized as
> the corresponding type during observe/crystallize.

Use these tags sparingly — only when the condition genuinely applies.
~~~

### Step 6 — Dispatch parallel agents

In a SINGLE response, make N Agent tool calls — one per cosmos. All run in
true parallel. Do NOT await one before starting the next.

**If `--models` was supplied (v3.3+):** Pass `model: <name>` to each Agent
tool call, using the model assigned to that cosmos in Step 1. Different cosmos
can use different models — this is the recommended way to mitigate the
"shared blind spot" problem where all cosmos inherit the same model's biases.

Construct each dispatch prompt by composing four blocks based on `<entanglement_mode>`:

**Block A — Preamble (always):**

```
You are cosmos:<name> in a QuantumAgent multiverse exploration.

Working directory: <repo_root>/cosmos/<name>
(This is a git worktree of the main repo.)

Goal: <goal>
Your strategy: <strategy>

<project_spin_summary>   (only if spin.json existed — 3-5 lines)
<active_singularities>   (only if events.jsonl had any — list active events)

━━━ QUANTUM MEMORY — follow these rules throughout ━━━

1. WRITE insights after every major step. Append to:
     <repo_root>/.quantum/<name>/insights.jsonl

   One JSON line per insight:
   {"type":"<type>","content":"<insight>","ts":"<ISO timestamp>"}

   Bash append only:
   echo '{"type":"discovery","content":"...","ts":"2026-05-12T12:00:00Z"}' >> <repo_root>/.quantum/<name>/insights.jsonl
```

**Block B — Rule 2, varies by `<entanglement_mode>`:**

- **`none`** — emit:
  ```
  2. INDEPENDENT MODE: do NOT read other cosmos insight files.
     This run uses pure parallel exploration — no cross-influence allowed.
     Resonance/Uncertainty will be computed post-hoc by /cosmos observe.
  ```

- **`passive`** *(default)* — emit:
  ```
  2. READ all cosmos insights after each major step:
       <repo_root>/.quantum/alpha/insights.jsonl   (skip if missing)
       <repo_root>/.quantum/beta/insights.jsonl    (skip if missing)
       <repo_root>/.quantum/gamma/insights.jsonl   (skip if missing)
       <repo_root>/.quantum/delta/insights.jsonl   (skip if missing)
       <repo_root>/.quantum/epsilon/insights.jsonl (skip if missing)

     Resonance: if multiple cosmos independently reach the same conclusion — trust it.
     Spin/Decoherence: your strategy is your spin — immutable. You may adopt patterns from
     other cosmos (entanglement), but wholesale copying loses your independence. Influence
     without convergence.
  ```

- **`active`** — emit the `passive` rule 2 plus:
  ```
     ACTIVE MODE ADDITION: whenever you adopt or react to another cosmos's insight,
     your insight MUST include a `read_from` field citing the source:
       {"type":"...","content":"...","read_from":"cosmos:beta@2026-05-12T10:31:00Z","ts":"..."}
     This produces a traceable entanglement graph — required for security, compliance,
     and debugging use cases.
  ```

- **`strict`** — emit the `passive` rule 2 plus the heartbeat protocol:
  ```
     STRICT MODE — heartbeat protocol enforced:

     At EVERY major step boundary you MUST execute this sequence:

     (a) WRITE your heartbeat:
         echo '{"type":"heartbeat","step":<N>,"content":"<your-name> at step <N>","ts":"<ts>"}' \
           >> <repo_root>/.quantum/<your-name>/insights.jsonl

     (b) READ all other cosmos insight files; find their `heartbeat` entries with ts
         later than your last `heartbeat-ack` for that cosmos.

     (c) For EACH unacknowledged heartbeat from another cosmos, WRITE an ACK:
         echo '{"type":"heartbeat-ack","content":"acknowledged <other>:step <M>","refs":["<other>@<their-ts>"],"ts":"<now>"}' \
           >> <repo_root>/.quantum/<your-name>/insights.jsonl

     (d) ONLY THEN proceed to the next implementation step.

     Failure mode: if /cosmos observe finds heartbeats without ACKs, it flags broken
     entanglement. Strict mode is for race-condition debugging, distributed-system
     design, and compliance audits that require *verifiable* live agent communication.
  ```

**Block C — Rule 3 (always):**

```
3. TAG quantum breakthrough insights via the `type` field. Use sparingly —
   a false positive dilutes the signal.

   "tunnel" — you bypassed a constraint you assumed was hard.
       ✅ "Redis sorted sets eliminate the need for a separate rate-limit table"
       ✅ "Token jti IS the revocation key — no separate table needed"
       ❌ "Found an unrelated bug while doing X" (this is serendipity, use discovery)
       ❌ "Optimized the inner loop" (this is optimization, use discovery)
       ❌ "Used a clever data structure" (this is design, use decision)

   "jump"   — another cosmos's insight caused a *discontinuous* architectural shift.
              You MUST cite which cosmos and which ts (`read_from`) — otherwise
              it's gradual adaptation, not a jump.
       ✅ "[read alpha@T+0:14:32] Switched from polling to event-sourcing entirely
            after seeing alpha's audit-trail requirement"
       ✅ "[read gamma@T+0:18:05] Rebuilt offline queue around HMAC signing"
       ❌ "Adopted alpha's helper function" (this is borrowing, use discovery)
       ❌ "Refactored after reading another insight" (gradual, use decision)
       ❌ "Changed approach mid-implementation" (no citation = not a jump)
```

**Block D — Final closing, varies by mode:**

- For **`none`** mode, emit:
  ```
  ━━━ Now implement the goal using your strategy. Work autonomously until complete.
      No entanglement reads required (independent mode). ━━━
  ```

- For **`passive`** or **`active`** mode, emit:
  ```
  ━━━ Now implement the goal using your strategy. Work autonomously until complete.

     BEFORE MARKING YOURSELF DONE — mandatory final entanglement read:
     Read all cosmos insight files one last time (same paths as rule 2 above).
     If any other cosmos recorded a bug fix, edge case, or security finding after
     your last read, apply it to your implementation and write a final insight.
     Only then stop.
  ━━━
  ```

- For **`strict`** mode, emit:
  ```
  ━━━ Now implement the goal using your strategy. Work autonomously until complete.

     STRICT MODE COMPLETION SEQUENCE:
     1. Final entanglement read — same as passive/active modes
     2. Write a final heartbeat with step="final"
     3. Read all other cosmos for any heartbeats published after your last ACK
        and write ACKs for each
     4. Only then stop

     Any unacknowledged heartbeat will be flagged by /cosmos observe as a
     broken entanglement channel for this run.
  ━━━
  ```

### Step 7 — Report launch status

Immediately before dispatching agents, output:

```
🌌 Spawning <N> cosmos...

  [alpha] cosmos/alpha  strategy: <strategy1>  model: <model1>
  [beta]  cosmos/beta   strategy: <strategy2>  model: <model2>
  [gamma] cosmos/gamma  strategy: <strategy3>  model: <model3>

⚛️  Quantum Memory: .quantum/
🔗 Entanglement mode: <entanglement_mode>
🌍 Project spin: <name>  (if loaded)
☄️  Active singularities: <count>  (if any)

Agents running. Superposition snapshot when complete.
```

The `model:` column appears for each cosmos. If `--models` was omitted, show `model: (inherited)` for all. If spin.json or events.jsonl were missing, omit those lines silently.

### Step 8 — Auto-observe on completion

After all agents return, automatically run `/cosmos observe`.
