# Auto research loop

*Generalized from [Karpathy's autoresearch](https://github.com/karpathy/autoresearch). Same loop, any domain.*

---

## The Idea

An AI agent runs an infinite hill-climbing loop: **modify → run → measure → keep or revert → repeat**. No human in the loop. Wake up to a TSV of completed experiments.

This works for **any project where you have:**
1. A quantifiable metric (one or two scalars)
2. A fast feedback loop (seconds to minutes per run)
3. A sandboxed file to modify
4. An immutable eval harness

The core insight: most optimization problems share the same structure — you tweak something, measure the outcome, and decide whether to keep the change. An agent can do this 24/7 without fatigue, bias, or distraction.

## The Pattern

```
LOOP FOREVER:
  1. Read current state (code, config, prompt — whatever you're optimizing)
  2. Form a hypothesis ("what if I try X?")
  3. Edit the target file
  4. Git commit (audit trail)
  5. Run the experiment (fixed time/resource budget)
  6. Extract metrics from output
  7. If improved → keep (advance branch)
     If equal/worse → revert (git reset)
  8. Log to results.tsv
  9. NEVER STOP — human will interrupt when done
```

### Pattern Variants

**Exploration vs. Exploitation**: The default loop is pure hill-climbing (exploitation). To avoid local optima, inject randomness:

```
VARIANT — EXPLORE/EXPLOIT:
  Every Nth iteration (e.g., N=5):
    Skip the "revert if worse" step
    Keep the change regardless (explore a new region)
    Mark as "explore" in the TSV
  Otherwise:
    Run the standard keep/revert logic (exploit)
```

**Batch Mode**: When individual runs are noisy (e.g., stochastic benchmarks), run the same config K times and compare the mean:

```
VARIANT — BATCHED:
  After editing the target file:
    Run the experiment K times (e.g., K=3)
    Compute mean and standard deviation of metrics
    Only keep if mean improves AND improvement > 1 stddev
```

**Multi-Stage Pipeline**: When optimization has dependent stages (e.g., preprocess → train → evaluate):

```
VARIANT — PIPELINE:
  Stage 1: Optimize preprocessing config → metric: data_quality_score
  Stage 2: Freeze preprocessing, optimize training → metric: val_loss
  Stage 3: Freeze training, optimize inference → metric: latency_ms
```

## Setup Contract

Every autoexp run needs four things defined upfront:

| Component | What it is | Example |
|-----------|-----------|---------|
| **Target file** | The ONE file the agent can modify | `train.py`, `prompt.txt`, `config.yaml` |
| **Eval harness** | Immutable script that produces the metric | `evaluate.py`, `run_bench.sh` |
| **Metric(s)** | 1-2 scalar values, lower or higher = better | `val_bpb ↓`, `accuracy ↑`, `latency_ms ↓` |
| **Budget** | Time/cost cap per experiment | 5 min wall clock, max 100 runs, $X total |

### Dual Metric Mode

For two metrics (e.g., accuracy ↑ and latency ↓), define a dominance rule:

- **Primary/secondary**: Improve primary; secondary is a soft constraint (e.g., "accuracy must improve; latency shouldn't 2x")
- **Pareto**: Keep if better on either metric without regressing on the other
- **Weighted**: `score = w1 * metric1 + w2 * metric2` — collapse to single scalar
- **Threshold-gated**: Metric B must stay above a minimum threshold; only then is improvement in Metric A considered (e.g., "latency must be < 200ms; beyond that, maximize throughput")
- **Lexicographic**: Sort by Metric A first. Only break ties using Metric B (e.g., "minimize error rate first, then minimize model size among equally accurate models")

### Triple+ Metric Mode

When you genuinely need 3+ metrics (e.g., accuracy, latency, memory usage), collapse them:

```python
# Option 1: Weighted composite
score = 0.5 * accuracy + 0.3 * (1 / latency_ms) + 0.2 * (1 / memory_mb)

# Option 2: Constraint + optimize
# Constraints: latency < 100ms AND memory < 512MB
# Optimize: accuracy ↑
# If constraints violated → discard immediately
```

## Results Logging

Tab-separated. Simple. Diffable. No infrastructure.

```
commit	metric_1	metric_2	status	description
a1b2c3d	0.9979	44.0	keep	baseline
b2c3d4e	0.9932	44.2	keep	increased learning rate to 0.04
c3d4e5f	1.0050	44.0	discard	switched to GeLU activation
d4e5f6g	0.0000	0.0	crash	doubled model width (OOM)
e5f6g7h	0.9910	42.1	keep	added dropout 0.1 after attention layers
f6g7h8i	0.9915	43.8	discard	tried cosine annealing schedule
g7h8i9j	0.9880	40.5	keep	reduced embedding dim from 512 to 256
h8i9j0k	timeout	—	crash	batch size 2048 exceeded memory + time budget
```

Statuses: `keep` | `discard` | `crash` | `timeout` | `explore`

### Extended Logging

For richer analysis, optionally maintain a companion `experiments.jsonl`:

```json
{"commit": "a1b2c3d", "timestamp": "2026-03-08T02:14:00Z", "metrics": {"val_loss": 0.9979, "throughput": 44.0}, "status": "keep", "description": "baseline", "diff_lines": 0, "wall_time_s": 120, "tokens_used": 0}
{"commit": "b2c3d4e", "timestamp": "2026-03-08T02:19:30Z", "metrics": {"val_loss": 0.9932, "throughput": 44.2}, "status": "keep", "description": "increased learning rate to 0.04", "diff_lines": 1, "wall_time_s": 118, "tokens_used": 1420}
```

This enables post-hoc analysis: which types of changes yield the biggest improvements? How does wall time correlate with metric gain?

## Branch Isolation

Always experiment on a dedicated branch:

```bash
git checkout -b autoexp/<tag>  # e.g., autoexp/mar8-prompt-tuning
```

Main stays clean. The branch is your lab notebook. Every commit is a recoverable experiment.

### Branch Strategy for Parallel Experiments

When running multiple autoexp agents simultaneously on different aspects:

```bash
git checkout -b autoexp/mar8-learning-rate   # Agent 1: tuning LR
git checkout -b autoexp/mar8-architecture     # Agent 2: model structure
git checkout -b autoexp/mar8-regularization   # Agent 3: dropout/weight decay
```

After all agents finish, cherry-pick the best commits from each branch into a combined experiment branch and validate that the improvements compose.

## Where This Applies

| Domain | Target file | Metric | Eval harness |
|--------|------------|--------|-------------|
| **ML training** | `train.py` | val_loss ↓ | Fixed eval function |
| **Prompt engineering** | `prompt.txt` | eval_accuracy ↑ | LLM judge or test suite |
| **RAG pipelines** | `config.yaml` | retrieval_precision ↑ | Benchmark query set |
| **Compiler/perf tuning** | `flags.conf` | runtime_ms ↓ | Benchmark binary |
| **API optimization** | `handler.py` | p99_latency ↓ | Load test script |
| **System prompts** | `system.md` | task_score ↑ | Eval suite with rubric |
| **CSS/layout** | `styles.css` | lighthouse_score ↑ | Lighthouse CI |
| **SQL queries** | `query.sql` | exec_time_ms ↓ | EXPLAIN ANALYZE wrapper |
| **Infrastructure** | `terraform.tf` | cost_per_hour ↓ | `terraform plan` parser |
| **Regex patterns** | `patterns.yaml` | f1_score ↑ | Labeled match/no-match dataset |
| **Search ranking** | `ranking.py` | ndcg@10 ↑ | Relevance-judged query set |
| **Image processing** | `pipeline.py` | ssim_score ↑ / processing_time ↓ | Reference image comparison |
| **Audio processing** | `denoise_config.yaml` | snr_db ↑ | Test audio clips + measurement |
| **Data pipeline ETL** | `transform.py` | rows_per_second ↑ / error_rate ↓ | Fixed input dataset |
| **Caching strategy** | `cache_config.yaml` | hit_rate ↑ / memory_mb ↓ | Replay production access logs |
| **Feature engineering** | `features.py` | model_auc ↑ | Fixed train/test split |
| **Hyperparameter search** | `hparams.json` | val_metric ↑ | Training + eval script |
| **Email/notification templates** | `template.html` | render_time_ms ↓ / accessibility_score ↑ | Rendering engine + axe-core |
| **Compression settings** | `compress.conf` | compression_ratio ↑ / decode_time_ms ↓ | Benchmark file set |
| **Networking config** | `nginx.conf` | requests_per_sec ↑ / error_rate ↓ | `wrk` or `ab` load test |
| **Database indexing** | `indexes.sql` | avg_query_time_ms ↓ | Query benchmark suite |
| **Serialization format** | `schema.proto` / `schema.avsc` | serialize_time_μs ↓ / payload_bytes ↓ | Round-trip benchmark |
| **Log parsing rules** | `grok_patterns.conf` | parse_accuracy ↑ / parse_rate_eps ↑ | Labeled log samples |
| **CI/CD pipeline** | `.github/workflows/ci.yml` | pipeline_duration_s ↓ | Trigger + measure workflow time |
| **Docker image** | `Dockerfile` | image_size_mb ↓ / build_time_s ↓ | `docker build` + `docker images` |
| **Kubernetes resources** | `deployment.yaml` | pod_startup_s ↓ / resource_cost ↓ | `kubectl apply` + monitoring |
| **Game AI behavior** | `ai_config.json` | win_rate ↑ | Simulated matches vs. baseline |
| **Recommendation engine** | `rec_model.py` | precision@k ↑ / diversity ↑ | Offline eval on held-out set |
| **A/B test config** | `experiment.json` | conversion_rate ↑ | Simulated traffic replay |
| **Spelling/grammar rules** | `rules.yaml` | f1_score ↑ | Annotated error corpus |
| **Chatbot routing** | `intents.yaml` | classification_accuracy ↑ | Labeled utterance dataset |
| **Batch job scheduling** | `scheduler.conf` | total_makespan_s ↓ | Simulated job queue |
| **Memory allocator tuning** | `malloc.conf` | alloc_throughput ↑ / fragmentation ↓ | Allocation trace replay |

### Detailed Case Studies

#### Case 1: Prompt Engineering for Classification

```
Target: prompt.txt (system prompt for an LLM classifier)
Metric: accuracy ↑ on 200 labeled test examples
Eval: Send each test input to the LLM with the prompt, compare output to gold label
Budget: 50 experiments, $15 API cost cap

Typical agent moves:
  - Add few-shot examples
  - Reword instructions for clarity
  - Add chain-of-thought scaffolding
  - Constrain output format
  - Add edge case handling instructions
```

#### Case 2: SQL Query Optimization

```
Target: query.sql (a slow analytical query)
Metric: execution_time_ms ↓ (via EXPLAIN ANALYZE)
Eval: Run query 5 times on a staging database, take median
Budget: 30 experiments, 2 hours wall clock

Typical agent moves:
  - Rewrite subqueries as CTEs (or vice versa)
  - Change JOIN order
  - Add/remove index hints
  - Replace correlated subqueries with window functions
  - Materialize intermediate results
  - Adjust WHERE clause predicate ordering
```

#### Case 3: Docker Image Size Reduction

```
Target: Dockerfile
Metric: image_size_mb ↓ (secondary: build_time_s ↓)
Eval: docker build → docker images → extract size
Budget: 40 experiments, 3 hours

Typical agent moves:
  - Switch base image (ubuntu → alpine → distroless)
  - Merge RUN layers to reduce intermediate layers
  - Add .dockerignore entries
  - Multi-stage builds
  - Remove unnecessary packages
  - Order layers for better cache reuse
```

#### Case 4: Regex Pattern Matching

```
Target: patterns.yaml (list of regex rules for data extraction)
Metric: f1_score ↑ on labeled test corpus
Eval: Run each regex against test strings, compute precision/recall/F1
Budget: 60 experiments, 1 hour

Typical agent moves:
  - Broaden overly strict patterns
  - Add negative lookaheads to reduce false positives
  - Combine redundant patterns
  - Handle edge cases (unicode, whitespace variants)
  - Anchor patterns to reduce backtracking
```

#### Case 5: Caching Strategy Optimization

```
Target: cache_config.yaml (TTL, eviction policy, size limits)
Metric: cache_hit_rate ↑ (constraint: memory_mb < 512)
Eval: Replay 1 hour of production access logs against cache simulator
Budget: 80 experiments, 4 hours

Typical agent moves:
  - Adjust TTL per content type
  - Switch eviction policy (LRU → LFU → ARC)
  - Tune cache size partitions
  - Add prefetch rules for predictable access patterns
  - Adjust admission policy thresholds
```

#### Case 6: Feature Engineering for Tabular ML

```
Target: features.py (feature transformation pipeline)
Metric: model_auc ↑ on fixed validation set
Eval: Generate features → train lightweight model (e.g., XGBoost) → evaluate
Budget: 40 experiments, 6 hours

Typical agent moves:
  - Add interaction features (feature_A * feature_B)
  - Log-transform skewed distributions
  - Bin continuous variables
  - Add rolling window aggregates
  - Target encoding for high-cardinality categoricals
  - Remove noisy or redundant features
```

#### Case 7: Nginx Performance Tuning

```
Target: nginx.conf
Metric: requests_per_sec ↑ (constraint: error_rate < 0.1%)
Eval: Run wrk benchmark for 30 seconds, extract RPS and error count
Budget: 30 experiments, 2 hours

Typical agent moves:
  - Adjust worker_processes and worker_connections
  - Enable/tune gzip compression levels
  - Tune keepalive_timeout and keepalive_requests
  - Adjust proxy_buffer_size settings
  - Enable sendfile, tcp_nopush, tcp_nodelay
  - Tune upstream connection pooling
```

#### Case 8: Game AI Behavior Tuning

```
Target: ai_config.json (weights for AI decision-making)
Metric: win_rate ↑ against baseline AI (over 100 simulated matches)
Eval: Run match simulator, count wins/losses/draws
Budget: 100 experiments, 8 hours

Typical agent moves:
  - Adjust aggression/defense weight balance
  - Tune resource gathering priorities
  - Modify threat assessment thresholds
  - Change retreat health thresholds
  - Adjust exploration vs. known-path preferences
```

## What Makes This Work

- **Single file constraint** — prevents the agent from refactoring the universe
- **Fixed budget** — no runaway experiments
- **Git commit per try** — perfect audit trail, trivial revert
- **TSV logging** — zero-infra results tracking
- **"NEVER STOP"** — the agent is a tireless researcher, not an assistant waiting for permission
- **Deterministic eval** — same input always produces the same metric (or use batching for stochastic evals)
- **Isolation** — experiments happen on a branch, in a sandbox, away from production

## What This Doesn't Work For

- **Subjective quality** (UI aesthetics, writing style, music composition) — no scalar metric
- **Slow feedback** (deploy → wait for user traffic → measure) — loop stalls
- **Multi-file changes** (architectural refactors) — too much surface area
- **Safety-critical systems** (medical devices, flight control) — autonomous modification without human review is a bad idea
- **Irreversible side effects** (sending emails, charging credit cards, modifying production databases) — agent must not trigger real-world actions
- **Problems requiring creativity over optimization** (designing a new algorithm from scratch) — hill-climbing can't escape the search space it's given
- **Highly coupled systems** (changing one config requires coordinated changes in 5 others) — single-file constraint breaks down
- **Non-deterministic environments** (metrics swing ±20% between identical runs) — unless you use batched evaluation with statistical significance testing

### Workarounds for Edge Cases

| Problem | Workaround |
|---------|-----------|
| Slow eval (>10 min) | Use a proxy metric (e.g., subset eval, smaller dataset) |
| Subjective quality | Create a rubric-based LLM judge that outputs a score |
| Multi-file needed | Bundle related configs into a single YAML/JSON file |
| Noisy metrics | Run each experiment K times, use mean ± stddev |
| Large search space | Seed the agent with known good directions in the prompt |
| Agent gets stuck in local optimum | Inject periodic random restarts (explore mode) |
| Expensive eval (API costs) | Use cheaper model for exploration, expensive for validation |

## Cost Awareness

Karpathy's version costs GPU-hours. In an LLM-agent context, each iteration costs API tokens. Add a budget cap:

```
max_experiments: 50        # hard stop
max_cost_usd: 10.00        # estimated from token usage
max_wall_clock_hours: 8    # total run time
abort_on_consecutive_crashes: 3  # stop if 3 crashes in a row
```

The agent should log cumulative cost in the TSV or a separate `budget.log`.

### Cost Estimation by Domain

| Domain | Typical cost per experiment | Experiments per hour | 8-hour overnight cost |
|--------|---------------------------|---------------------|----------------------|
| Prompt engineering (API) | $0.05–$0.50 | 10–30 | $4–$120 |
| ML training (GPU) | $0.10–$5.00 | 3–12 | $2.40–$480 |
| SQL optimization | ~$0 (local DB) | 20–60 | ~$0 |
| Docker builds | ~$0 (local) | 5–15 | ~$0 |
| Load testing | ~$0 (local) | 10–20 | ~$0 |
| Infrastructure (cloud) | $0.01–$1.00 | 5–10 | $0.40–$80 |

## Agent Instructions Template

Use this as the base prompt when spawning an autoexp agent:

```markdown
You are an autonomous experimentation agent. Your task:

**Goal**: Optimize [METRIC] by modifying [TARGET_FILE].
**Eval command**: `[EVAL_COMMAND]`
**Metric extraction**: [HOW TO PARSE METRIC FROM EVAL OUTPUT]
**Direction**: [↑ higher is better / ↓ lower is better]
**Budget**: [MAX_EXPERIMENTS] experiments, [MAX_HOURS] hours, $[MAX_COST] cost
**Branch**: autoexp/[TAG]

Rules:
1. ONLY modify [TARGET_FILE]. Never touch the eval harness.
2. Git commit before every experiment run.
3. If metric improves or equals best → keep. Otherwise → revert.
4. Log every experiment to results.tsv (commit, metric, status, description).
5. If an experiment crashes, log it as "crash" and revert.
6. NEVER STOP until budget is exhausted or human interrupts.
7. Think carefully about each hypothesis. Prefer targeted changes over random edits.
8. After every 10 experiments, review the results.tsv to identify patterns.
```

## Advanced Techniques

### Warm Starting

Instead of starting from scratch, seed the agent with knowledge:

```
Previous best experiments (from last night's run):
- Learning rate 0.03 gave val_loss 0.9910 ✓
- Dropout 0.1 helped, 0.3 hurt
- GeLU worse than ReLU for this architecture
- Batch sizes > 1024 cause OOM

Start from commit [BEST_COMMIT] and explore from there.
```

### Checkpoint and Resume

For long-running experiments, design for interruption:

```bash
# Save state
cp results.tsv results.tsv.bak
git log --oneline autoexp/current > experiment_history.txt

# Resume later
git checkout autoexp/current
# Agent reads results.tsv to understand what's been tried
# Continues from where it left off
```

### Meta-Optimization

Run autoexp on autoexp — optimize the agent's own strategy:

```
Target: agent_strategy.yaml (parameters like explore_rate, batch_size, hypothesis_style)
Metric: best_metric_after_50_experiments ↑
Eval: Run a full 50-experiment autoexp loop, extract the best metric achieved
```

This is slow but powerful for tuning the exploration strategy itself.

## The Philosophy

> If each experiment takes ~5 minutes, you get ~12/hour. Over an 8-hour sleep, that's ~100 experiments. You wake up to a results table your agent built while you dreamed.

The human's job: define the metric, set the constraints, review the results.
The agent's job: relentlessly explore the space.

The power of this pattern isn't any single experiment — it's the compound effect of hundreds of small, methodical explorations. Most individual changes will be discards. But the few that stick accumulate into something no human would have the patience to find manually.

---

*Inspired by [@karpathy/autoresearch](https://github.com/karpathy/autoresearch). Generalized for any quantifiable optimization problem.*
