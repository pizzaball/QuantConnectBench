# QuantConnect-Faithful

Faithful QuantConnect strategy checking using the **LEAN engine**, run
**locally**.


## The four stages

| Stage | Check | How |
|-------|-------|-----|
| 1. `compiled`   | valid Python syntax | `ast.parse()` |
| 2. `validated`  | `QCAlgorithm` subclass + `Initialize()` | regex |
| 3. `backtested` | runs in the real engine | `lean backtest` (Docker) |
| 4. `traded`     | LEAN reports ≥ 1 order | parsed from LEAN's result JSON |

## Setup

```bash
pip install -r requirements.txt        # installs the Lean CLI

# Create a Lean workspace — this is where your data lives.
# `lean init` initializes the CURRENT directory (it takes no path argument),
# so make the folder and cd into it first:
mkdir -p ~/lean-workspace
cd ~/lean-workspace
lean init
```

Verify the environment is ready before you rely on it:

```bash
python3 check.py --check-env
```

## Usage

```bash
# Point --workspace at your `lean init` workspace so LEAN finds the data:
python3 check.py examples/sma_crossover.py --workspace ~/lean-workspace

# Or point directly at a LEAN-format data folder:
python3 check.py my_strategy.py --data-folder /path/to/lean/data

cat my_strategy.py | python3 check.py - --workspace ~/lean-workspace
```

Example output:

```
  Compilation  (ast.parse)
  QC structure (QCAlgorithm + Initialize)
  Backtest ran (real LEAN engine)
  Trades executed

  Trades        : 6
  Total return  : +4.18%
  Annual return : +4.18%
  Sharpe ratio  : +0.51
  Max drawdown  : +10.30%
  Win rate      : +60.00%
```

Exit codes (designed for CI):

- `0` — compiled **and** validated **and** backtested **and** traded
- `1` — a strategy stage failed (syntax, structure, runtime error, or zero trades)
- `2` — environment not ready (Lean CLI or Docker missing) — couldn't run the engine

## As a library

```python
from qc_faithful import check

result = check(open("my_strategy.py").read(), workspace_dir="~/lean-workspace")
print(result["traded"], result["trade_count"], result["sharpe_ratio"])
```

## Benchmark: single-shot & agentic

Beyond checking one file, this repo includes an LLM benchmark harness in the
spirit of [QuantCode-Bench](https://github.com/LimexAILab/QuantCode-Bench). An LLM is asked to write a QuantConnect strategy for each task; the strategy is then graded by the pipeline:

```
compiled → validated → backtested (LEAN) → traded → judged (optional LLM)
```

`reward = 1.0` only if it compiles, validates, backtests, trades, and (when a
judge key is given) the judge agrees the code matches the task.

**Two modes, same generator:**

- **single-shot** — one generation attempt per task, no feedback.
- **agentic** — up to `--max-turns` attempts, each re-prompted with structured
  feedback derived from the stage it failed.

```bash
# Single-shot over the bundled tasks, scored by real LEAN:
python run_single_shot.py \
    --api-key  $OPENAI_API_KEY \
    --model    gpt-4o-mini \
    --tasks    data/benchmark_tasks_multiframe.json \
    --workspace ~/lean-workspace \
    --output   results/

# Agentic with up to 10 feedback turns, plus an LLM judge:
python run_agentic.py \
    --api-key  $OPENAI_API_KEY \
    --judge-key $OPENAI_API_KEY \
    --model    gpt-4o-mini \
    --max-turns 10 \
    --workspace ~/lean-workspace \
    --batch-size 2
```