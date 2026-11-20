# GitGoodBench → Harbor Adapter

## Overview

GitGoodBench evaluates AI agents on version-control (Git) tasks mined from permissively licensed, open-source Python, Java, and Kotlin repositories on GitHub. **This adapter converts its merge conflict resolution scenario**: a real merge was started and stopped with conflicts, and the agent must resolve all conflicts and conclude the merge. The result is scored by exact match against the ground-truth merge commit.

| Split | HuggingFace dataset | Harbor tasks |
|-------|---------------------|--------------|
| `lite` | [`JetBrains/git_good_bench-lite`](https://huggingface.co/datasets/JetBrains/git_good_bench-lite) | **60** merge conflict tasks (20 each for Python, Java and Kotlin) |

Each `merge` sample of GitGoodBench Lite becomes one Harbor task (`jetbrains/gitgoodbench`). Lite is the split on which the paper reports its baseline. Only the `merge` samples are converted; the benchmark's other sample type, `file_commit_chain`, is outside the scope of this adapter. The datasets are licensed Apache-2.0; the source repositories carry Apache-2.0, MIT, or BSD licenses.

Two things differ from the original benchmark. The original baseline agent used custom tools such as `resolve_current_merge_conflict_with`, while Harbor agents work in a shell, so the part of the prompt that describes those tools is replaced by a note on where the checkout is and how to conclude the merge. The rest of the prompt, including the quoted conflict hunks, is unchanged. The ground-truth merge commit is also removed from the task image, so the agent cannot look it up.

## What is GitGoodBench?

GitGoodBench ([paper](https://aclanthology.org/2025.realm-1.19/), [repository](https://github.com/JetBrains-Research/git-good-bench)) was introduced by JetBrains Research at the 1st Workshop for Research on Agent Language Models (REALM 2025, co-located with ACL 2025). It targets a gap in software-engineering agent benchmarks such as SWE-bench, which overlook developer workflows around version control. A merge scenario counts as solved when no conflict remains and `git diff <ground-truth merge commit> current-scenario-branch` is empty.

The paper reports a 13.33% solve rate on the 60 Lite merge scenarios for a GPT-4o baseline with custom tools.

## Adapter Features

- Converts every `merge` sample of GitGoodBench Lite. The full dataset (`--split full`) and local exports (`--dataset-path`) are supported as well.
- Builds each scenario the way the original harness does (`ScenarioEnvironmentManager._setup_merge_conflict_scenario`): the image fetches the repository history from GitHub, checks out the first parent on `current-scenario-branch` and merges the second parent, which stops on the recorded conflicts.
- Keeps the answer out of the container. The image holds only the history the scenario needs, the ground-truth merge commit is purged, and during the agent phase the container can reach model APIs only (`network_mode = "allowlist"`).
- Scores like the original `Evaluator`, without network access: the reward is 1 when `current-scenario-branch` contains both merge parents and its tree equals the ground-truth merge tree. The verifier reads the commits straight from the object store instead of running Git in the agent's repository, so Git configuration, hooks or replacement refs left by the agent do not affect grading. Details are written to `/logs/verifier/details.json`.
- Ships an oracle that commits the ground-truth merge from `solution/ground_truth.bundle`, a Git bundle of a few KB. It needs no network access.

## Generated Task Structure

```
gitgoodbench/
├── {repo}-merge-{nnnn}/              # one task per merge sample
│   ├── task.toml                 # Task configuration
│   ├── instruction.md            # Instructions for the agent
│   ├── environment/              # Container definition
│   │   ├── Dockerfile
│   │   ├── setup_scenario.py     # Build-time scenario setup (deleted from the final image)
│   │   └── scenario.json         # Scenario description (deleted from the final image)
│   ├── solution/
│   │   ├── solve.sh              # Oracle solution
│   │   └── ground_truth.bundle   # Ground-truth merge commit (prerequisites: the merge parents)
│   └── tests/
│       ├── test.sh               # Verifier entry point
│       ├── evaluate.py           # Exact-match evaluation
│       └── config.json           # Merge parents and ground-truth tree id for the verifier
```

Task names follow `jetbrains/gitgoodbench__<task-id>`, where `<task-id>` is the sanitized original sample id (e.g. `spulec_freezegun_merge_0000` → `spulec-freezegun-merge-0000`).

The adapter code directory:
```
src/gitgoodbench/
├── README.md
├── adapter_metadata.json
├── parity_experiment.json
├── pyproject.toml
├── run_gitgoodbench.yaml
└── src/gitgoodbench/
    ├── __init__.py
    ├── adapter.py
    ├── main.py
    └── task-template/
        ├── task.toml
        ├── instruction.md        # the original merge prompt, rendered per task
        ├── environment/
        │   ├── Dockerfile
        │   └── setup_scenario.py
        ├── solution/
        │   └── solve.sh
        └── tests/
            ├── test.sh
            └── evaluate.py
```


## Run Evaluation / Harness
Harbor Registry & Datasets makes running adapter evaluation easy and flexible.

### Running with Datasets Registry

 Simply run

```bash
# Use oracle agent (reference solution)
harbor run -d jetbrains/gitgoodbench

# Use your specified agent and model
harbor run -d jetbrains/gitgoodbench -a <agent_name> -m "<model_name>"
```
from the harbor root to evaluate on the entire dataset (GitGoodBench Lite, 60 merge tasks).

> The dataset is not registered yet (this adapter is work in progress). Until then, prepare the task directories locally as described in [Usage](#usage-create-task-directories) and use the commands below.

### Using Job Configurations

```bash
# From the repository root
# Run a job with the default adapter configuration (oracle agent, GitGoodBench Lite)
harbor run -c src/gitgoodbench/run_gitgoodbench.yaml

# With a specific agent and model
harbor run -c src/gitgoodbench/run_gitgoodbench.yaml -a <agent_name> -m "<model_name>"

# Or run a job without configuration yaml but instead with locally prepared dataset path
harbor run -p datasets/gitgoodbench -a <agent_name> -m "<model_name>"

# Resume a previously started job
harbor job resume -p /path/to/jobs/directory
```

Results are saved in the `jobs/` directory by default (configurable via `jobs_dir` in the YAML config).

During the agent phase the tasks allow egress to `api.anthropic.com` and `api.openai.com` only. For another provider or a gateway, add its host at run time, e.g. `--allow-agent-host openrouter.ai`. The environment must support per-phase network policies (Docker, Daytona, E2B, Modal). Docker Desktop for macOS lacks the kernel support Harbor's egress control needs, and the run fails with `network_mode='allowlist' is not supported`; use OrbStack, a Linux Docker host or Daytona instead. The tasks request 10 GB of disk (`storage_mb = 10240`), the maximum a Daytona sandbox allows.

### Running Individual Trial

For quick testing or debugging a single task:

```bash
# Run a single task with oracle (pre-written solution)
harbor trial start -p datasets/gitgoodbench/spulec-freezegun-merge-0000

# Run a single task with a specific agent and model
harbor trial start -p datasets/gitgoodbench/spulec-freezegun-merge-0000 -a <agent_name> -m "<model_name>"
```

Trial outputs are saved in the `trials/` directory by default (configurable via `--trials-dir`).


## Usage: Create Task Directories

```bash
cd src/gitgoodbench
uv sync

# GitGoodBench Lite (60 merge tasks) -> datasets/gitgoodbench
uv run gitgoodbench

# Full GitGoodBench (450 merge tasks, not part of the registered dataset) -> datasets/gitgoodbench-full
uv run gitgoodbench --split full
```

Available flags:
- `--output-dir` — Directory to write generated tasks (defaults to `datasets/gitgoodbench`, or `datasets/gitgoodbench-full` for `--split full`, at the repo root)
- `--limit` — Generate only the first N tasks (tasks are sorted by task id)
- `--overwrite` — Overwrite existing tasks
- `--task-ids` — Only generate specific tasks (Harbor task names such as `jetbrains/gitgoodbench__spulec-freezegun-merge-0000`, Harbor task ids, or original sample ids); unknown ids are an error
- `--split` — `lite` (default) or `full`
- `--dataset-path` — Load samples from a local `.jsonl`/`.json`/`.parquet`/`.csv` export instead of HuggingFace

Generation needs `git` and access to GitHub. For each task the ground-truth merge commit is fetched once, which takes a second or two, to write `solution/ground_truth.bundle` and the ground-truth tree id in `tests/config.json`. If a fetch fails, no partial task is left behind and the command can be re-run.

The conflict hunks quoted in `instruction.md` come from replaying the merge with `git merge-tree`. The generator ignores the Git configuration of the machine it runs on, so the instructions come out the same everywhere. Generation stops with an error if a file the dataset lists is not in conflict or if the number of hunks differs. Two Lite tasks, `micrometer-metrics-micrometer-merge-0054` and `mockito-mockito-merge-0002`, have one more conflict that the dataset does not list: a file modified on one side and deleted on the other. It is recorded in `task.toml` under `metadata.unlisted_conflict_files`.

Tasks are written with one directory per task. Each task follows the structure shown in ["Generated Task Structure"](#generated-task-structure) above.

## Comparison with Original Benchmark (Parity)

The baseline agent of the paper cannot be run: its LLM client, agent loop and data source were removed from the released `src/agent_client`, which the benchmark's first author confirmed on the adapter PR. The original side of the parity experiment is therefore a reference harness that adds those pieces back and runs a standard CLI agent in the original scenario container. Scenario setup (`ScenarioEnvironmentManager`) and scoring (`Evaluator`) are the original code, which makes this a Scenario 2/3 adapter in the terms of the adapter guide. [`parity/README.md`](parity/README.md) describes the harness, how to run both sides and how they differ.

Both sides ran mini-swe-agent 2.4.6 with `deepseek-v4-1-flash-260910` (requested as `deepseek-v4.1-flash`), 3 runs over the 60 tasks. The metric is the solve rate.

| Agent | Model | Metric | Runs | Dataset Size | Original (mean ± SEM) | Harbor (mean ± SEM) |
|-------|-------|--------|------|--------------|-----------------------|---------------------|
| mini-swe-agent@2.4.6 | deepseek-v4-1-flash-260910 | Solve rate (%) | 3 | 60 (100%) | 40.56 ± 2.0031 | 41.67 ± 0.9623 |

Per-run solve rates: original 36.67, 41.67, 43.33; Harbor 40.00, 43.33, 41.67. The ranges overlap. Both sides get the same prompt, run the agent's commands with the same shell and have no access to the ground-truth merge commit. 52 of the 60 tasks have the same outcome in all three runs on both sides. By language the original side scores Java 25.0%, Kotlin 46.7%, Python 50.0% and the Harbor side Java 25.0%, Kotlin 50.0%, Python 50.0%.

Links:
- Original benchmark: [JetBrains-Research/git-good-bench](https://github.com/JetBrains-Research/git-good-bench) at `b11ae6cd92be2b8b96e237d379427e04ad59c455`
- Reference harness: [YanggWendy/git-good-bench](https://github.com/YanggWendy/git-good-bench/tree/0375616a91a58e495b370908918d01991bc6ebf4), branch `harbor-parity`, also kept as [`parity/reference-harness.patch`](parity/reference-harness.patch)
- [Adapter PR](https://github.com/harbor-framework/adapters/pull/78) and [dataset PR](https://github.com/harbor-framework/harbor-datasets/pull/277)
- Parity results: [HuggingFace parity-experiments PR](https://huggingface.co/datasets/harborframework/parity-experiments/discussions/281). The shared repository is over the Hub's file limit and does not accept uploads at the moment, so the files are at [yanggwendy/gitgoodbench-parity](https://huggingface.co/datasets/yanggwendy/gitgoodbench-parity/tree/main/adapters/gitgoodbench) in the same layout.

Oracle and negative controls, one run each:

| Side | Oracle (expected 100%) | Negative controls (expected 0%) |
|------|------------------------|----------------------------------|
| Original side (reference harness) | 60/60 | no-op agent: 0/2 (smoke samples) |
| Harbor adapter (Daytona) | 60/60 | all rejected: no-op agent, leftover conflict markers, deleted `.git`, correct files committed without merging, hostile Git configuration and hooks, forged history (`refs/replace`, commit-graph, tampered commit), reward file planted before grading |

The same controls run as unit tests in [`tests/test_gitgoodbench_verifier.py`](tests/test_gitgoodbench_verifier.py); [`tests/test_gitgoodbench.py`](tests/test_gitgoodbench.py) covers the generator.

Reproduction requirements and steps:
- Original side (reference harness; details in `parity/README.md`):
  ```bash
  git clone https://github.com/YanggWendy/git-good-bench.git && cd git-good-bench
  git checkout 0375616a91a58e495b370908918d01991bc6ebf4  # branch harbor-parity
  docker build -f docker/cli-agent.Dockerfile -t gitgoodbench-cli-agent .
  pip install docker datasets "mini-swe-agent==2.4.6" "litellm==1.91.1"
  export OPENAI_API_KEY=<key> OPENAI_BASE_URL=<OpenAI-compatible endpoint>
  python -m src.agent_client.run_cli_agent_benchmark --agent mini-swe-agent \
      --model openai/deepseek-v4.1-flash --image gitgoodbench-cli-agent \
      --hide-ground-truth --runs 3 --run-name mswea-deepseek-v4.1-flash-aligned
  python -m src.agent_client.summarize_cli_agent_results results/mswea-deepseek-v4.1-flash-aligned/results.jsonl
  ```
- Harbor side (3 attempts on Daytona; the job file allows the endpoint's host during the agent phase through `extra_allowed_hosts`):
  ```bash
  export OPENAI_API_KEY=<key> OPENAI_BASE_URL=<OpenAI-compatible endpoint>
  harbor run -c src/gitgoodbench/parity/run_gitgoodbench_parity.yaml
  ```
- The reported metric is the solve rate: the mean reward over all tasks, averaged over runs.

## Notes & Caveats

- **Tool interface.** The original baseline agent acted through custom tools; Harbor agents use a shell. The instruction is the original prompt with the tool section replaced by a note on where the checkout is and how to conclude the merge. It still gives the temporal order of the two parents, the conflicted files and every conflict hunk.
- **Scoring.** The original harness compares the tree of the branch head with the ground-truth merge commit. The adapter compares the same trees and also requires both merge parents in the branch history, so committing the right files without merging does not count. Only committed objects are checked; the index and working tree are ignored.
- **Verifier.** The verifier does not run Git inside the agent's repository. It reads the graded branch and its commits from the object store and checks each commit against its hash, so Git configuration, hooks, replacement refs, grafts or a rewritten commit-graph left by the agent do not change the result. Before grading it removes any reward file already in `/logs/verifier`. It runs in the task container, where the agent has root, so it does not protect against a replaced `git` or `python3` binary.
- **Leakage.** The original harness clones the complete repository, which contains the ground-truth merge commit. The task image holds only the history the scenario needs, and the agent phase can reach model APIs only. With open internet access an agent could fetch the merge commit from GitHub.
- **Build time and network.** Images fetch repository history from GitHub at build time. Large repositories take several minutes and a few GB of disk, and a sample whose commits disappear from GitHub will no longer build. The oracle and the verifier need no network.
- **Git version.** The image uses Git 2.39 from Debian bookworm with the `ort` merge strategy, like the Debian-based container of the original harness. Other Git versions can produce slightly different conflict hunks.

## Installation / Prerequisites

Adapters are managed as standalone uv Python packages. You can add or remove dependencies using `uv add` / `uv remove` from the adapter directory:

```bash
cd src/gitgoodbench
uv add datasets  # add a dependency
uv remove datasets  # remove a dependency
```

Environment setup:
- Docker installed and running
- Harbor 0.21.0 or newer installed and working (see main repository README); the tasks use task schema 1.4. To run mini-swe-agent use 0.26.0 or newer
- Python environment with dependencies:
  ```bash
  cd src/gitgoodbench
  uv sync
  ```
- Dataset-specific steps:
  - Access to HuggingFace to download `JetBrains/git_good_bench-lite` / `JetBrains/git_good_bench` (or a local export passed via `--dataset-path`)
  - `git` and internet access to GitHub for task generation and image builds
  - An environment provider with per-phase network policy support (see [Using Job Configurations](#using-job-configurations))
  - API keys for the evaluated agents/models

The unit tests for the generator and the verifier are in `tests/` and need `git`:

```bash
cd src/gitgoodbench
uv run --with pytest pytest tests
```

## Troubleshooting

- **Task generation fails with `could not fetch <commit>`** or **image build fails with `Could not fetch <commit>`**: GitHub is unreachable or rate limiting; re-run (generation skips finished tasks) or retry with lower concurrency. If the commit was removed upstream, the sample cannot be built.
- **`network_mode='allowlist' is not supported by docker environment`**: the Docker engine's kernel lacks nftables `fib` support, which Harbor's egress-control sidecar needs. On macOS use OrbStack instead of Docker Desktop, run on a Linux host, or use Daytona (`-e daytona`).
- **Agent cannot reach its model API**: only `api.anthropic.com` and `api.openai.com` are allowed during the agent phase; pass `--allow-agent-host <host>` for other providers or gateways.
- **Build timeouts or disk pressure on huge repositories**: lower `n_concurrent_trials`, increase `build_timeout_sec`, and clean up images with `docker system prune`.
- **Why did a trial score 0?**: see `verifier/details.json` in the trial directory: it records the branch head, which merge parents are missing from its history, the committed tree id against the expected one, and which conflicted files still contain conflict markers.

## Citation

```bibtex
@inproceedings{lindenbauer-etal-2025-gitgoodbench,
    title = "{G}it{G}ood{B}ench: A Novel Benchmark For Evaluating Agentic Performance On Git",
    author = "Lindenbauer, Tobias  and
      Bogomolov, Egor  and
      Zharov, Yaroslav",
    booktitle = "Proceedings of the 1st Workshop for Research on Agent Language Models (REALM 2025)",
    month = jul,
    year = "2025",
    address = "Vienna, Austria",
    publisher = "Association for Computational Linguistics",
    url = "https://aclanthology.org/2025.realm-1.19/",
    doi = "10.18653/v1/2025.realm-1.19",
    pages = "272--288"
}
```

## Authors & Contributions
This adapter is developed and maintained by [Wenting Yang](mailto:yanggwendy0213@gmail.com).

**Issues and Contributions:**
- Submit Issues and Pull Requests to the main repository
- Follow the project's coding style and commit guidelines

## Acknowledgement

API inference compute for running parity tests is generously supported by [2077AI](https://www.2077ai.com/) (https://www.2077ai.com/).
