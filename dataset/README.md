# Dataset roles and provenance

The project keeps the base-model evaluation set separate from the trained-model test set. These corpora are evaluation-only and are not part of `sft_dataset.jsonl`.

## Model evaluation

`model_evaluation/harmbench_test.jsonl` contains the 320 rows from HarmBench's official text test behavior split. This source file defines behaviors rather than pre-generated attack strings; the `prompt` field uses `ContextString`, followed by a blank line and `Behavior` when context is present, otherwise just `Behavior`. The original context and category fields are retained. Three source behavior IDs occur more than once; `prompt_id` remains unique per row.

- Source: [HarmBench](https://github.com/centerforaisafety/HarmBench), revision `8e1604d1171fe8a48d8febecd22f600e462bdcdd`
- Source file: `data/behavior_datasets/harmbench_behaviors_text_test.csv` (SHA-256 `75d257b3e7428c52eb7b0154318f455af3e01b09a3794b5e2f3d36054f3c0e29`)
- License: MIT; source notice is in `licenses/HarmBench-MIT-LICENSE.txt`
- Citation: Mazeika et al., *HarmBench: A Standardized Evaluation Framework for Automated Red Teaming and Robust Refusal* (2024). See the source repository for the citation entry.

This direct behavior set is not the full HarmBench red-team pipeline or its attack-generated test cases.

## Testing the trained model

`trained_model_test/advbench_test.jsonl` contains the `goal` prompts from AdvBench. The source `target` completions are intentionally omitted. AdvBench does not label an official test split; this project reserves the full 520-row source file for post-training evaluation.

- Source: [llm-attacks/llm-attacks](https://github.com/llm-attacks/llm-attacks), revision `098262edf85f807224e70ecd87b9d83716bf6b73`
- Source file: `data/advbench/harmful_behaviors.csv` (SHA-256 `6cd1a5c63c07610d7eb67307772ee5606017ee950b5770ab288a2c487489d3e1`)
- License: MIT; source notice is in `licenses/AdvBench-MIT-LICENSE.txt`
- Citation: Zou et al., *Universal and Transferable Adversarial Attacks on Aligned Language Models* (2023).

`trained_model_test/xstest_safe.jsonl` is the existing 123-prompt safe XSTest subset, normalized with stable IDs and role metadata for false-refusal measurement. The source subset was already in this repository as `OOD/xstest_safe.jsonl`; its exact upstream revision was not recorded.

## Data layout

- `model_evaluation/`: baseline evaluation prompts.
- `trained_model_test/`: held-out harmful-request prompts and safe prompts for false-refusal measurement.
- `OOD/`: existing OOD and XSTest source files retained for compatibility.
- `licenses/`: upstream MIT license notices for newly added HarmBench and AdvBench sources.
- `sft_dataset.jsonl`: training data only.

Every normalized row includes `prompt_id`, `prompt`, `technique`, `difficulty`, `target_behavior`, `source`, `source_id`, `source_split`, `evaluation_role`, `category`, and `category_method`.

## Prompt categories

`category` is a primary content category on every row in the three files above. HarmBench uses its original `semantic_category` value (also retained under that field). AdvBench uses a curated primary label mapped to the same taxonomy from the prompt's main requested harm; these labels are project annotations, not upstream AdvBench metadata. XSTest prompts use `safe` to identify the safe subset used for false-refusal measurement.

The taxonomy values are `chemical_biological`, `copyright`, `cybercrime_intrusion`, `harassment_bullying`, `harmful`, `illegal`, `misinformation_disinformation`, and `safe`. For consistent interpretation across files, `category_method` records whether a value came from HarmBench (`source_semantic_category`), project curation (`curated_project_annotation`), or the XSTest safe subset (`source_safe_subset`). AdvBench categories are primary labels; a prompt may involve secondary harms not represented by that single label.
