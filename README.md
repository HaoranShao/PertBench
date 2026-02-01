# PertBench Green Agent

PertBench is an A2A-compatible benchmarking framework for the systematic evaluation of agent endpoints on dataset-grounded question-answering (QA) tasks. The framework is explicitly domain-agnostic: given a ground-truth dataset and a lightweight task specification—defining, for example, key columns, question rendering rules, and gold-label interpretation. PertBench can instantiate standardized benchmarks across diverse QA domains. In its current release, PertBench includes a concrete instantiation for *single-cell perturbation significance analysis*, built on multiple curated datasets. Beyond conventional accuracy metrics, PertBench explicitly quantifies paraphrase robustness by generating multiple template-based paraphrases for each evaluation unit and aggregating model responses into unit-level outcomes. This design enables more stable, fair, and reproducible comparisons across heterogeneous agents and model backends.

## What this benchmark evaluates

This benchmark tests whether an agent can correctly answer gene perturbation QA questions such as:
- "Does perturbing gene A significantly change expression of gene B in cell line X?"
- "Does perturbing gene A and perturbing gene B significantly change expression of gene B in cell line X?"
- “In X cells, gene A and gene B are perturbed and gene C expression is quantified. Does this perturbation result in a significant change in gene C expression compared with control cells?”

The benchmark is designed to measure:
- Correctness (accuracy vs gold labels)
- Robustness (invalid/ambiguous outputs)
- Efficiency (calls, tokens, estimated cost)

## Key features

- Multiple datasets supported (single & double perturbation)

- A2A-native integration: reads Purple Agent identity from its /.well-known/agent-card.json

- Automated scoring with micro-accuracy and coverage metrics

- Nuanced evaluation outputs:

- invalid rate / ambiguous rate (structured mode)

- per-dataset breakdown

- token usage + estimated USD cost

- Artifacts emitted as JSON/JSONL for easy auditing and leaderboard ingestion

- Deterministic runs via random_seed and consistent unit selection


## Repository Structure

```bash
GreenAgent/
  src/                   # Green Agent server implementation (A2A HTTP server)
  example/
    data/                # CSV datasets
    config/              # Spec JSON templates
  pyproject.toml
  uv.lock
  Dockerfile
  README.md
```

## Datasets and Specification Files

Datasets are located under `example/data/`:

- `adamson_psa_single.csv`

- `norman_psa_single.csv`

- `replogle_k562_psa_single.csv`

- `replogle_rpe1_psa_single.csv`

- `norman_psa_double.csv`

Specification files are located at `example/config/`:
- `perturbation_analysis_prompts_single.json` (for the 4 single datasets)
- `perturbation_analysis_prompts_double.json` (for the double dataset)
By default, the benchmark can run all datasets, or a specified dataset via scenario configuration.

## Evaluation methodology

**Inputs.**

The Green Agent receives an A2A assessment request from the AgentBeats runner and will:

1. Discover the Purple Agent endpoint.
2. Load the selected dataset(s) and prompt spec(s).
3. Sample evaluation units (optionally limited by max_units).
4. Query the Purple Agent and parse final answers (Yes / No / Invalid).
5. Aggregate and score results.

**Scoring.**

We report:

- Coverage: fraction of units that produced a valid label (Yes/No)
- Accuracy: fraction correct among covered units
- Ambiguous rate: ties / unresolved cases after aggregation
- Invalid rate: invalid answers divided by total answers across calls

We also compute micro accuracy across all datasets:

```python
micro_accuracy = total_correct / total_covered
```

**Cost estimation.**

If the Purple Agent returns usage metadata (input/output tokens, model), the Green Agent aggregates:

- calls, input/output/total tokens

- per-model token totals

- estimated cost using configured pricing (per-1M tokens)

*Note*: cost is an estimate based on token usage and the pricing table provided in config. Actual billing depends on your provider/account.

**Outputs (Artifacts).**

The Green Agent emits artifacts through the A2A TaskUpdater interface, including:

- `*.unit_results.jsonl`
Per-evaluation-unit details (gold label, parsed predictions, aggregation fields)

- `*.summary.json`
Per-dataset metrics + usage + estimated cost

- `aggregate.summary.json`
Aggregated metrics across datasets

- `leaderboard.json`
A compact, standardized summary for leaderboard ingestion (includes Purple Agent identity)

Example leaderboard.json fields:

- `participant`: role, endpoint, name/version from agent card

- `scores`: micro_accuracy, micro_covered_units

- `usage`: calls, tokens, estimated cost

- `per_dataset`: accuracy, coverage, invalid/ambiguous, usage

## Configuration

This benchmark is typically launched via an AgentBeats scenario TOML (example):

```
[green_agent]
endpoint = "http://127.0.0.1:9009"
cmd = "./start_green.sh"

[[participants]]
role = "qa_agent"
endpoint = "http://127.0.0.1:9010"
cmd = "./start_purple.sh"

[config]
dataset = "all"      # or a dataset id like "norman_psa_single"
max_units = 100     # the number of sample units to be tested

```

**Notes:**

- If `dataset="all"` and `max_units=100`, the benchmark applies `max_units` per dataset (i.e., each dataset runs up to 100 selected units), unless you implemented a global cap explicitly.

## Ground Truth Data and Question Generation

If you want to construct a benchmark using your own dataset, this is how you can do. The ground truth data files are in CSV format, consisting of all necessary keys for questions and corresponding answers. To form a prompt (or question), you must also prepare a JSON configuration file, containing the components below:

- `task_name`: The name of the task
- `input_mode`: The mode of input, either choose `structured` to form questions using different question templates, or choose `qa_pairs` to directly feed in question-answer pairs.
- `gold_label`: The gold label for the question units. **This should be exactly the column name of correct answers in the ground truth data file.**

If the `input_mode` is `structured`, you must also provide the following keys:

- `keys`: A list of keys that will be used to form questions. **Be sure that the keys are identical to the column names in the ground truth data file**
- `min_valid_answers_per_unit`: An integer threshold. A unit is considered covered/answerable only if the number of valid predictions (parsable as Final Answer: Yes/No) is at least this value. Only covered units are included in the accuracy (and consistency) denominators.  
- `model_input`: A list of template strings, where placeholders are written as {key} and key must be in `keys`.
- `tie`: Decide how to mark the majority vote when the number of Yes and No among valid predictions are equal. Choose from "Yes", "No", or "Ambiguous". If `tie` = "Ambiguous", it is always treated as incorrect, and `ambiguous_rate` will be reported.

If the `input_mode` is `qa_pairs`, you must also provide the following keys:

- `question`: A list of question strings.

Keys below are optional:

- `max_units`: The maximum number of units to evaluate. Set to an integer (e.g., 10) to run a small smoke test. Omit this field or set to null to evaluate all units.
- `unit_selection`: The method for unit selection. Choosing from "head"(default), "random", or "slice".
- `random_seed`: Random seed used when `unit_selection` = "random". If omitted, the sampling will be non-deterministic.
- `start_index`: The starting row index (0-based) used when `unit_selection` = "slice". Defaults to 0 if omitted.

## Example Data

### Perturbation Analysis Data

### `Adamson`
- Cell Line: `K562`
- Source: Adamson *et al.* , A Multiplexed Single-Cell CRISPR Screening Platform Enables Systematic Dissection of the Unfolded Protein Response. *Cell* **167**, 1867-1882.e21 (2016). DOI: [10.1016/j.cell.2016.11.048](https://doi.org/10.1016/j.cell.2016.11.048)
    
### `Norman`
- Cell Line: `K562`
- Source: Thomas M. Norman *et al.* ,Exploring genetic interaction manifolds constructed from rich single-cell phenotypes.*Science* **365**,786-793(2019).DOI:[10.1126/science.aax4438](https://www.science.org/doi/10.1126/science.aax4438)

### `Replogle`
- Cell Line: `K562`, `RPE1`
- Source: Replogle J.M. *et al.* , Mapping information-rich genotype-phenotype landscapes with genome-scale Perturb-seq. *Cell* **185**, 2559-2575.e28 (2022). DOI: [10.1016/j.cell.2022.05.013](https://ncbi.nlm.nih.gov/pubmed/35688146)
