# VeriBTS — Artifact and Data Availability Guide

Companion artifact for the FSE 2027 submission
**"VeriBTS: A General LLM-Enabled CEGIS Framework for Behavior Tree Generation"**.

---

We package all experiments as a Docker-based artifact, enabling users to reproduce the results by running the provided commands and inspecting the generated outputs.



## 1. What the artifact contains

One Docker image bundles **all six instances** of VeriBTS, both ablation
baselines, the BtBot baseline, all verifiers, both task suites (the adapted
70-task benchmark and the 10-task stress set), and all statistics scripts:

| Paper instance (§4) | Executable in image |
|---|---|
| **Examples** (§4.3.1) | `/app/BtBotRun/BtBotRun` |
| **Goal** (§4.3.2) | `/app/pdpl_baselines-linux` |
| **MITL–TA** (§4.3.3) | `/app/llmbt-uppaal` |
| **LTL–BB** (§4.3.3, BehaVerify) | `/app/BTRun-linux/BTRun` |
| **LTL–CSP** (§4.3.3, CSP) | `/app/ltl4bt_baselines-linux` |
| **LTL–Direct** (§4.3.3) | `/app/llmbt-runner` |

Method naming used by the runners (identical across all six instances):

| Paper term | Runner flag / experiment name |
|---|---|
| VeriBTS | `ce` · `full` · `LLM4BtBot` |
| NoFeedback (ablation) | `noce` · `LLM4BtBot_noFeedback` |
| OnlyFailedBT (ablation) | `withBT` / `withbt` · `LLM4BtBot_noFeedback_withBT` |
| BtBot (baseline, Examples instance only) | phase `sketch` |

LLM generators : `gpt-4o`, `gpt-5.5`, `claude-opus-4-6`.

Experimental protocol implemented by the runners (§4.4): 3 models × 3
methods × **5 repetitions**, 70 benchmark tasks + 10 stress tasks, at most
**10 LLM calls per task**; transport-level errors are retried and never
count toward the limit.

## 2. Requirements

* Linux x86-64 host with **Docker**.
* ~5 GB free disk.
* **UPPAAL** license available: d5f8c4f4-29b1-469d-a202-ec7f02d24e33
* No GPU, no Python, no verifier installation — everything is inside the image.

## 3. Obtaining the image

```bash
docker pull veribts/btltl-baselines:1.3       
```

Image layout (`/app`):

```
/app/ltl4bt_baselines-linux      LTL–CSP instance          → results in /app/results
/app/pdpl_baselines-linux        Goal instance             → results in /app/results
/app/BTRun-linux/BTRun           LTL–BB instance           → results_hard/ results_complex/
/app/BtBotRun/BtBotRun           Examples instance + BtBot → BtBotResults/
/app/llmbt-runner                LTL–Direct instance       → pass --out-root /app/results
/app/llmbt-uppaal                MITL–TA instance          → pass --out /app/results
/app/stats/                      statistics scripts (§6)
/app/results/                    default results directory (mount a host dir here)
```

## 4. Smoke tests (zero LLM cost)

Recommended first step after `docker pull`:

```bash
docker run --rm veribts/btltl-baselines:1.3 bash -c '
/app/BtBotRun/BtBotRun check                      # Examples  : bundled checker self-test
/app/BTRun-linux/BTRun --selftest                 # LTL–BB    : NuSMV + BehaVerify toolchain
/app/llmbt-runner selftest                        # LTL–Direct: verifier + task cards
/app/llmbt-uppaal selftest                        # MITL–TA   : UPPAAL engine + bt2ta pipeline'
```

Each self-test prints a per-component report and must end without errors
(`SELFTEST PASSED` for `llmbt-runner`, `[selftest] passed (offline +
engine)` for `llmbt-uppaal`). The Goal (PDPL) and LTL–CSP runners have no
separate self-test command; §5 shows one-task pilots for them. Note: the
gateway section of `llmbt-runner selftest` (part 2/3) depends on the
third-party `4sapi.org` service being up; an `HTTP 502` there indicates an
upstream outage — the verifier (1/3) and task-card (3/3) sections still
validate the artifact.

Optional connectivity test (a few cents of LLM traffic, one minimal call per
API key): `docker run --rm veribts/btltl-baselines:1.3 /app/BtBotRun/BtBotRun ping`.

## 5. Re-running the experiments

All commands below write results into host directories via bind mounts.
Every runner supports **resume**: re-running the same command after an
interruption continues at the first unfinished repetition/task.

Cost notice: a full configuration (3 models × 3 methods × 5 repetitions) is
70×3×3×5 = 3 150 task-runs on the benchmark and 450 on the stress set *per
instance*. All LLM traffic is billed to prepaid gateway keys embedded in the
image. For a first check, run the **pilot** variant of each command
(`--limit` / `--rounds 1` / `--repeat 1`), then launch the full run.

Work through the instances one by one (all examples assume `mkdir -p results
btrun btbot` in the current directory):

**(a) Goal — PDPL runner**

```bash
# pilot (~1 task, 1 repetition): verifier + gateway end-to-end
docker run --rm -v $PWD/results:/app/results veribts/btltl-baselines:1.3 \
    /app/pdpl_baselines-linux --bench bt70 --baselines ce --models gpt-4o --runs 1 --limit 1

# full benchmark (Table 3 "Goal") + stress set (Table 4 "Goal")
docker run --rm -v $PWD/results:/app/results veribts/btltl-baselines:1.3 \
    /app/pdpl_baselines-linux --bench bt70
docker run --rm -v $PWD/results:/app/results veribts/btltl-baselines:1.3 \
    /app/pdpl_baselines-linux --bench bt10x
```

**(b) LTL–CSP — PAT-backed runner**

```bash
# pilot: 1 task, 1 repetition
docker run --rm -v $PWD/results:/app/results veribts/btltl-baselines:1.3 \
    /app/ltl4bt_baselines-linux --bench bt70 --models gpt-4o --variants ce --rounds 1 --limit 1

# full runs -- pass --models explicitly (the runner's DEFAULT model set is
# gpt-5.5/claude-opus-4-6/claude-sonnet-4-6, NOT the paper's trio)
docker run --rm -v $PWD/results:/app/results veribts/btltl-baselines:1.3 \
    /app/ltl4bt_baselines-linux --bench bt70 --models gpt-4o gpt-5.5 claude-opus-4-6   # Table 3 "LTL–CSP"
docker run --rm -v $PWD/results:/app/results veribts/btltl-baselines:1.3 \
    /app/ltl4bt_baselines-linux --bench bt10 --models gpt-4o gpt-5.5 claude-opus-4-6   # Table 4 "LTL–CSP"
```

**(c) LTL–BB — BTRun (BehaVerify/NuSMV)**

```bash
# pilot: 2 tasks, 1 repetition
docker run --rm -v $PWD/btrun:/data veribts/btltl-baselines:1.3 \
    /app/BTRun-linux/BTRun --suite hard --limit 2 --repeat 1 --results_root /data

# full: hard = Table 3 "LTL–BB", complex = Table 4 "LTL–BB"
docker run --rm -v $PWD/btrun:/data veribts/btltl-baselines:1.3 \
    /app/BTRun-linux/BTRun --suite both --results_root /data
```

**(d) Examples + BtBot baseline — BtBotRun**

```bash
# pilot: one baseline, one model, one round
docker run --rm -w /app/BtBotRun -v $PWD/btbot:/app/BtBotRun/BtBotResults \
    veribts/btltl-baselines:1.3 /app/BtBotRun/BtBotRun main --exps LLM4BtBot --llms gpt-4o --rounds 1

# full: main = Table 2 (VeriBTS/NoFeedback/OnlyFailedBT),
#       sketch = Table 2 "BtBot" column, complex = Table 4 "Examples"
docker run --rm -w /app/BtBotRun -v $PWD/btbot:/app/BtBotRun/BtBotResults \
    veribts/btltl-baselines:1.3 /app/BtBotRun/BtBotRun all --rounds 5
```

**(e) LTL–Direct — llmbt-runner**

```bash
# pilot: single task, single run, single method
docker run --rm -v $PWD/results:/app/results veribts/btltl-baselines:1.3 \
    /app/llmbt-runner run --project hard --baselines ce --models gpt-4o --runs 1 --tasks ABC3 --out-root /app/results

# full (add --jobs 6 to parallelize; --dry-run prints the plan, no calls)
docker run --rm -v $PWD/results:/app/results veribts/btltl-baselines:1.3 \
    /app/llmbt-runner run --project hard --out-root /app/results        # Table 3 "LTL–Direct"
docker run --rm -v $PWD/results:/app/results veribts/btltl-baselines:1.3 \
    /app/llmbt-runner run --project llmtask --out-root /app/results     # Table 4 "LTL–Direct"
docker run --rm -v $PWD/results:/app/results veribts/btltl-baselines:1.3 \
    /app/llmbt-runner run --project hard --out-root /app/results --dry-run
```

**(f) MITL–TA — llmbt-uppaal**

```bash
# pilot: single task, single run, single method
docker run --rm -v $PWD/results:/app/results veribts/btltl-baselines:1.3 \
    /app/llmbt-uppaal run --bench hard --baselines ce --models gpt-4o --runs 1 --tasks ABC3 --out /app/results

# full (--dry-run prints the job matrix, no calls)
docker run --rm -v $PWD/results:/app/results veribts/btltl-baselines:1.3 \
    /app/llmbt-uppaal run --bench hard --out /app/results          # Table 3 "MITL–TA"
docker run --rm -v $PWD/results:/app/results veribts/btltl-baselines:1.3 \
    /app/llmbt-uppaal run --bench task --out /app/results          # Table 4 "MITL–TA"
```

## 6. Statistics (reproducing Tables 2–4)

The metrics reported by `/app/stats/*` are exactly the paper's
`Succ@ = succ@ ± s_succ@` and `Calls@ = calls@ ± s_calls@`
(the success/calls definitions in §4.4): per repetition, success rate over
the task set and mean calls per task, where **a task that fails at the call
limit contributes 10 calls**; across repetitions, mean ± sample standard
deviation (n−1).
The 70-task benchmark and the 10-task stress set are reported in separate
sections. Statistics re-run from raw files only — no LLM traffic.

**One command for everything:**

```bash
docker run --rm \
    -v $PWD/results:/app/results \
    -v $PWD/btrun:/data/btrun \
    -v $PWD/btbot:/app/BtBotRun/BtBotResults \
    -e BTRUN_DIR=/data/btrun \
    veribts/btltl-baselines:1.3 bash /app/stats/stats_all.sh /app/results
```

Per-instance scripts (same layout, one tool at a time):

```bash
# Table 3 / Table 4 rows:
docker run --rm -v $PWD/results:/app/results veribts/btltl-baselines:1.3 python3 /app/stats/stats_pdpl.py   /app/results   # Goal        (bt70 / bt10x)
docker run --rm -v $PWD/results:/app/results veribts/btltl-baselines:1.3 python3 /app/stats/stats_ltl4bt.py /app/results   # LTL–CSP     (bt70 / bt10)
docker run --rm -v $PWD/btrun:/data           veribts/btltl-baselines:1.3 python3 /app/stats/stats_btrun.py  /data         # LTL–BB      (hard / complex)
docker run --rm -v $PWD/btbot:/app/BtBotRun/BtBotResults veribts/btltl-baselines:1.3 bash /app/stats/stats_btbot.sh       # Examples    (main / complex)
docker run --rm -v $PWD/results:/app/results veribts/btltl-baselines:1.3 bash /app/stats/stats_ltl_direct.sh /app/results  # LTL–Direct  (hard / llmtask)
docker run --rm -v $PWD/results:/app/results veribts/btltl-baselines:1.3 bash /app/stats/stats_mitl_ta.sh   /app/results  # MITL–TA     (hard / task)
```

Console format (excerpt from a real run of the Goal instance, benchmark):

```
================= PDPL (pdpl_baselines-linux) =================
--- 70-task benchmark (bt70) ---
    model               baseline   tasks  runs |     success rate % |   avg LLM calls/task
    gpt-5.5             ce            70     5 |      100.00 ± 0.00 |          1.01 ± 0.01
    gpt-5.5             noce          70     5 |      100.00 ± 0.00 |          1.01 ± 0.01
    ...
```

Mapping to the paper's tables (row `ce` = VeriBTS, `noce` = NoFeedback,`withbt` = OnlyFailedBT):

| Paper table | Rows | Stats output section |
|---|---|---|
| Table 2 (`btbot`) — VeriBTS / NoFeedback / OnlyFailedBT | 3 methods × 3 models | `stats_btbot.sh` — part [1/3] `main` |
| Table 2 (`btbot`) — BtBot column | 3 models | `stats_btbot.sh` — part [3/3] `sketch` |
| Table 3 (`5instance`) Goal | 3 × 3 | `stats_pdpl.py` — `70-task benchmark (bt70)` |
| Table 3 MITL–TA | 3 × 3 | `stats_mitl_ta.sh` — hard section |
| Table 3 LTL–BB | 3 × 3 | `stats_btrun.py` — `results_hard` |
| Table 3 LTL–CSP | 3 × 3 | `stats_ltl4bt.py` — `70-task benchmark (bt70)` |
| Table 3 LTL–Direct | 3 × 3 | `stats_ltl_direct.sh` — hard section |
| Table 4 (`6stress`) Examples | 3 × 3 | `stats_btbot.sh` — part [2/3] `complex` |
| Table 4 Goal | 3 × 3 | `stats_pdpl.py` — `10-extreme-task benchmark (bt10x)` |
| Table 4 MITL–TA | 3 × 3 | `stats_mitl_ta.sh` — task section |
| Table 4 LTL–BB | 3 × 3 | `stats_btrun.py` — `results_complex` |
| Table 4 LTL–CSP | 3 × 3 | `stats_ltl4bt.py` — `10-complex-task benchmark (bt10)` |
| Table 4 LTL–Direct | 3 × 3 | `stats_ltl_direct.sh` — llmtask section |



## 7. Practical notes

* **Keys and cost.** The runners embed prepaid gateway keys for `4sapi.org`
  and connect **directly** (system proxies are ignored). All LLM cost of a
  replication run is consumed from these keys.
* **Call limit and error policy.** 10 successful LLM responses per task;
  transport errors (timeouts, 5xx, resets) are retried and do not count —
  identical to the protocol in §4.4.
* **UPPAAL lease (MITL–TA).** A 24-hour license lease is renewed online from
  `uppaal.veriaal.dk` on the first `verifyta` invocation; the image pre-sets
  the `USER` environment variable that `verifyta` requires. Nothing to
  configure.
* **Gateway IP drift.** If LLM calls report network errors, set
  `LLMBT_SAPI_IP=<IP>` (runner family) or `LTL4BT_GATEWAY=<domain>` /
  `LTL4BT_PROXY=host:port` (LTL–CSP family) — see each runner's `--help`.
* **Resume.** All runners append missing repetitions instead of overwriting;
  simply re-launch the same command.
* **Expected runtime.** On a 32-vCPU machine the full protocol (all six
  instances, benchmark + stress) takes several days wall-clock and is fully
  parallelizable per model/baseline; the statistics scripts run in seconds.

## 8. Troubleshooting

| Symptom | Fix |
|---|---|
| `permission denied` on self-tests | ensure the user is in the `docker` group; commands assume `docker` without `sudo` |
| LLM network errors | If LLM calls report network errors, set`LLMBT_SAPI_IP=<IP>` (runner family) or `LTL4BT_GATEWAY=<domain> LTL4BT_PROXY=host:port` (LTL–CSP family) — see each runner's `--help`. |
| MITL–TA self-test `verifyta engine check` fails | container has no direct internet to `uppaal.veriaal.dk` (lease renewal) — restore connectivity and re-run |
| Statistics print `(no result files found)` | results of that tool are not under the mounted path — check §3 layout and the mounts of §5/§6 |

