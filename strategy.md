# smartsweep strategy

This file is the human layer of the smartsweep agent loop — the equivalent of `program.md` in karpathy/autoresearch, adapted for autonomous inference benchmarking instead of model training.

The agent reads this file to understand the research objective, constraints, and loop behavior. Edit this file to change what the agent optimizes for.

---

## Objective

Find the fastest inference configuration for a given GGUF model on this hardware.

The primary metric is **tok/s** (tokens per second, prompt + generation combined from llama-bench).
The optimization target is:

> Maximize `tok/s` subject to `ctx >= MIN_CTX` and no OOM.

Set `MIN_CTX` below based on your use case.

---

## Configuration

```
MODEL:      path to the .gguf file to benchmark (required)
OUTPUT_DIR: ./results (default)
MIN_CTX:    4096      # minimum context size that matters to you
GOAL:       tok/s     # primary axis to maximize (tok/s, pp/s, or tg/s)
```

---

## Setup

To start a new experiment session:

1. **Agree on a run tag**: propose a tag based on today's date (e.g. `apr5`). The branch `smartsweep/<tag>` must not already exist.
2. **Create the branch**: `git checkout -b smartsweep/<tag>` from main.
3. **Read in-scope files**:
   - `strategy.md` — this file. The research objective and loop rules.
   - `llamaseye.sh` — the sweep engine. Read the phase descriptions and key functions. Do not modify the core sweep logic.
   - `example.env` — all available `SWEEP_*` environment variables and their defaults.
4. **Verify the model exists** at the path in `MODEL` above.
5. **Initialize results.tsv**: copy the header row from the template below.
6. **Run the baseline**: phases 0–6 with default settings to establish the working sets and context ceiling.
7. **Confirm setup**, then begin the experiment loop.

---

## Experiment loop

The agent loop runs Phase 7 (combo matrix) repeatedly, each time proposing a targeted variation based on prior results.

**What you CAN modify:**
- `SWEEP_*` environment variables — sweep ranges, batch sizes, thread counts, KV types, etc.
- Which phases to re-run via `--only-phases`
- The goal expression passed to `--goal`

**What you CANNOT modify:**
- The core sweep logic in `llamaseye.sh`
- Output file formats (`sweep.jsonl`, `sweep.md`) — these are the source of truth
- Hardware detection or OOM handling

**The loop:**

LOOP FOREVER:

1. Read `results/*/sweep.jsonl` — extract the current best tok/s and its full config
2. Propose a targeted variation: adjust one or two parameters around the current best
3. Run the variation: `bash llamaseye.sh --model <MODEL> --only-phases 7 --goal "tok/s" [other flags]`
4. Extract the best result: `jq -r 'select(.phase==7) | [.tok_s, .ngl, .ctx, .fa, .ctk, .batch, .ubatch, .threads] | @tsv' results/*/sweep.jsonl | sort -rn | head -5`
5. Record results in `results.tsv`
6. If tok/s improved → keep the config, advance the branch with a git commit
7. If tok/s equal or worse → note the result, do not commit, try a different variation next round

**Simplicity criterion**: all else being equal, simpler is better. A 1% tok/s gain that requires a complex multi-flag incantation is worse than a clean config that gets 0.5% less. Prefer fewer flags, fewer overrides.

**NEVER STOP**: Once the loop has begun, do not pause to ask if you should continue. Run until manually interrupted.

---

## Output format

After each Phase 7 run, extract the best result with:

```bash
jq -r 'select(.phase==7) | [.tok_s, .ngl, .ctx, .fa, .ctk, .batch, .ubatch, .threads, .nkvo] | @tsv' \
  results/*/sweep.jsonl | sort -rn | head -1
```

Log to `results.tsv` (tab-separated):

```
commit	tok_s	ctx	ngl	fa	ctk	batch	ubatch	threads	nkvo	status	description
```

- `status`: `keep`, `discard`, or `oom`
- `description`: short note on what this experiment tried

Example:

```
commit	tok_s	ctx	ngl	fa	ctk	batch	ubatch	threads	nkvo	status	description
a1b2c3d	142.3	4096	35	1	q8_0	512	512	8	0	keep	baseline
b2c3d4e	149.1	4096	35	1	q4_0	512	512	8	0	keep	ctk q4_0 saves VRAM, faster
c3d4e5f	138.0	4096	35	1	q8_0	1024	512	8	0	discard	larger batch slower on this GPU
d4e5f6g	0.0	8192	35	1	q8_0	512	512	8	0	oom	doubled ctx OOM
```

---

## Notes for the agent

- `sweep.jsonl` is the source of truth — always derive your analysis from it, not from `sweep.md`
- OOM at a given ctx does not mean that ctx is impossible — try flipping `nkvo` or using a more compressed `ctk` type
- Phase 6 already handles OOM fallbacks automatically; use its output (`WS_CTX`) to set your ctx ceiling for Phase 7 variations
- The `--fine-ctx` flag enables midpoint bisection in Phase 6 for a more precise context ceiling
- If tok/s has plateaued for 5+ rounds, try a more radical change: different FA setting, very different batch size, or re-run Phase 5 with a wider range
