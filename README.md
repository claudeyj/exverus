# <p align=center> [ICML 2026] ExVerus: Verus Proof Repair via Counterexample Reasoning

[![Arxiv Paper](https://img.shields.io/badge/Arxiv-Paper-brightred)](https://arxiv.org/abs/2603.25810)
[![OpenReview](https://img.shields.io/badge/OpenReview-Forum-8c1b13)](https://openreview.net/forum?id=FNDkXA0OUJ)
![](https://img.shields.io/badge/PRs-welcome-brightgreen) ![](https://img.shields.io/github/stars/claudeyj/exverus?style=social)
[![Website](https://img.shields.io/badge/Website-exverus--proof.github.io-blue)](https://exverus-proof.github.io/)

## About
Existing LLM-based verification approaches treat proof generation as a
static, end-to-end prediction over source code, relying on limited verifier
feedback with no access to concrete program behavior. _ExVerus_ is a
counterexample-guided framework that instead lets an LLM reason about proofs
using behavioral feedback: when a [Verus](https://github.com/verus-lang/verus)
proof fails, ExVerus generates and validates source-level counterexamples,
then guides the LLM to generalize them into inductive invariants that block
the failure — significantly improving proof accuracy, robustness, and token
efficiency over the state-of-the-art prompting-based Verus proof generator.

This repository is the artifact for the paper, containing the Python repair
pipeline, Rust/Verus-Syn helper tools, benchmark copies, prompts, and analysis
scripts used by the paper experiments.

> [!IMPORTANT]
> Full paper-scale reproduction requires LLM API calls, Verus, Rust tooling, and
> substantial compute. The commands below focus on the core experiment modes in
> small/debug form rather than launching the full sweep.



![poster](./exverus_poster.svg)

## Repository Contents

| Path | Contents |
| --- | --- |
| `vinv/` | ExVerus pipeline, LLM clients, prompts, analysis utilities |
| `tool/rs/convert/` | Rust helpers for loop extraction and CEX instrumentation |
| `cleaned-verusbench/` | Cleaned VerusBench tasks |
| `Benchmarks/` | Additional paper benchmarks: DafnyBench, HumanEval, LCBench, ObfsBench, etc. |
| `verus-proof-synthesis/` | AutoVerus support code reused for initial proof generation and Verus utilities |
| `scripts/` | Historical batch scripts used for large experiment sweeps |
| `results/` | Generated outputs, repair traces, status files, and cost reports |

## Paper Experiment Map

The main paper experiments correspond to these artifact modes:

| Paper result | Artifact mode |
| --- | --- |
| Main ExVerus repair | `vinv pipeline repair --cex-generation-strategy z3 --cex-generalization-strategy mut_val` |
| Iterative refinement baseline | `vinv pipeline repair --ablation` |
| Direct generalization ablation | `--cex-generalization-strategy simple` |
| VerusBench | `--source CLEANED_VB` |
| DafnyBench + LCBench + HumanEval | `--source THREEBENCH` |
| Additional benchmarks, including ObfsBench | `--source ADDITIONAL` |

See [`Benchmarks/README.md`](Benchmarks/README.md) for which benchmark folder each `--source`
value pulls tasks from, task counts, and citations.

The paper evaluates five model families: DeepSeek-V3.1, GPT-4o, Qwen3-Coder,
o4-mini, and Claude Sonnet 4.5. The default paper repair budget is 10 repair
attempts and 10 counterexamples.

## Setup

Initialize the AutoVerus dependency and install Python dependencies from this
directory:

```bash
git submodule update --init --recursive
uv sync
```

Make Verus available and source the local environment setup:

```bash
# Either put `verus` on PATH, or set one of these:
export VERUS_PATH=/path/to/verus
# export VERUS_BIN=/path/to/verus
source ./setup.sh
```

The paper implementation used Verus `0.2025.07.12.0b6f3cb`. The Rust helper
tools in `tool/rs/convert` are built automatically on first use and require
`cargo`. `setup.sh` also exports `LYNETTE_PATH` to the bundled AutoVerus
Lynette binary path; build it with
`cargo build --manifest-path verus-proof-synthesis/utils/lynette/source/Cargo.toml`
if a command needs the standalone binary.

Configure at least one LLM provider key:

```bash
export OPENAI_API_KEY=...
export OPENROUTER_API_KEY=...
export ANTHROPIC_API_KEY=...
```

ExVerus generates the AutoVerus first-stage configuration under
`results/autoverus_configs/` from these environment variables and the local
`verus-proof-synthesis` submodule paths. To use a hand-written AutoVerus config
instead, set `AUTOVERUS_CONFIG_FILE=/path/to/config_<model>.json`.

`OLD_VERUS_PATH` is optional and only needed by commands that explicitly request
the old Verus executable.

Sanity-check the local setup:

```bash
uv run vinv check
uv run vinv check --strict
```

If this reports that Verus is missing Rust toolchain `1.88.0`, install it with:

```bash
rustup install 1.88.0-x86_64-unknown-linux-gnu
```

If `uv` cannot write to its default cache on a restricted machine, use a local
or temporary cache:

```bash
UV_CACHE_DIR=/tmp/uv-cache uv run vinv check
```

## Core Experiment Commands

### 1. Main ExVerus Repair

This is the core paper pipeline: Z3Py counterexample generation plus
mutation/validation repair. The command below uses `--debug` to run a small
VerusBench subset.

```bash
uv run vinv pipeline repair \
  --model gpt-4o \
  --source CLEANED_VB \
  --cex-generation-strategy z3 \
  --cex-generalization-strategy mut_val \
  --num-cex 10 \
  --max-repair-attempts 10 \
  --debug \
  --num-workers 1
```

### 2. Iterative Refinement Baseline

This mode repairs from verifier feedback without the ExVerus CEX-guided mutation
pipeline.

```bash
uv run vinv pipeline repair \
  --model gpt-4o \
  --source CLEANED_VB \
  --ablation \
  --max-repair-attempts 10 \
  --debug \
  --num-workers 1
```

### 3. Direct Generalization Ablation

This keeps counterexample generation but uses direct/simple generalization
instead of the `mut_val` repair strategy. It is the closest CLI-level ablation
to the paper's no-mutator comparison.

```bash
uv run vinv pipeline repair \
  --model gpt-4o \
  --source CLEANED_VB \
  --cex-generation-strategy z3 \
  --cex-generalization-strategy simple \
  --num-cex 10 \
  --max-repair-attempts 10 \
  --debug \
  --num-workers 1
```

## Useful Single-File Helper

To test the Rust loop-harness conversion helper directly:

```bash
uv run vinv assume convert path/to/input.rs --out /tmp/assume
uv run vinv assume convert path/to/input.rs --out /tmp/assert --use-assert
```

## Outputs

Debug runs write under `results/pipeline_debug/<model>/<source>/`. Non-debug
runs write under `results/pipeline/<model>/<source>/`.

Each task directory may contain:

- `init_gen/`: initial proof generation output and verification status
- `cex_repair_<generation>_<generalization>_<num_cex>/`: CEX-guided repair trace
- `repair_status_*.json`: per-task repair result
- `llm_cost_report_*.json`: per-task token and cost accounting
- `aggregated_llm_cost_*.json`: merged cost report for the run

## Full Paper Reproduction

The full paper tables require running the same modes across all benchmark
sources, model families, and ablations. The historical sweep scripts in
`scripts/` show the exact large-run structure, but they are intentionally not
listed here as quick-start commands because they launch expensive batch LLM
experiments.

For paper-scale runs, use the core modes above and vary:

- `--model`: `deepseek/deepseek-chat-v3.1`, `gpt-4o`, `qwen/qwen3-coder`,
  `o4-mini`, or a Claude Sonnet model configured in the LLM client
- `--source`: `CLEANED_VB` or `ADDITIONAL`
- `--cex-generalization-strategy`: `mut_val` for ExVerus, `simple` for the
  direct generalization ablation
- `--num-cex`: `10` for the main setting, `1` for the CEX-count sensitivity

## Citation

If you use this artifact, please cite:

```bibtex
@inproceedings{
yang2026exverus,
title={ExVerus: Verus Proof Repair via Counterexample Reasoning},
author={Jun Yang and Yuechun Sun and Yi Wu and Rodrigo Caridad and Yongwei Yuan and Jianan Yao and Shan Lu and Kexin Pei},
booktitle={Forty-third International Conference on Machine Learning},
year={2026},
url={https://openreview.net/forum?id=FNDkXA0OUJ}
}
```
