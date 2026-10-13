# Intrinsic vs. Extrinsic Safety Hardening for Small Language Models

An independent research project comparing two defensive approaches against adversarial prompting on Small Language Models (SLMs) under 10B parameters.

- **Intrinsic defense**: LoRA Supervised Fine-Tuning (SFT) on Mistral-7B-Instruct-v0.2
- **Extrinsic defense**: request screening and response moderation with Llama Guard 3 8B or Granite Guardian 3.1 2B

The designated evaluation files are HarmBench for baseline model evaluation, AdvBench for trained-model testing, and safe XSTest prompts for measuring false refusals.

---

## Existing Results

The figures below come from the earlier experiment and its original prompt corpus. They have not been recomputed with the designated datasets or the updated two-stage guard pipeline; rerun notebook 3 before treating them as a comparison of the current setup.

| Method | Defense Success Rate (DSR) | False Refusal Rate (FRR) |
|---|---|---|
| Baseline (no defense) | 6.49% | 0.00% |
| Llama Guard 3 (extrinsic) | 66.23% | 66.67% |
| LoRA SFT (intrinsic) | **90.91%** | **17.07%** |

These historical numbers should not be used to rank the current methods. The updated notebook screens requests before generation and checks responses afterward, and compares Llama Guard 3 with the smaller Granite Guardian. Each guard still runs alongside the target model during guarded evaluation; actual memory use depends on quantization and device placement.

---

## Repository Structure

```
.
├── dataset/
│   ├── model_evaluation/
│   │   └── harmbench_test.jsonl       # Baseline model evaluation
│   ├── trained_model_test/
│   │   ├── advbench_test.jsonl        # Held-out harmful-request test prompts
│   │   └── xstest_safe.jsonl          # Safe prompts for false-refusal measurement
│   ├── OOD/                           # Existing OOD and XSTest source corpora
│   ├── direct_injection.jsonl
│   ├── indirect_injection.jsonl
│   ├── persona_hijacking.jsonl
│   ├── hypothetical_framing.jsonl
│   └── sft_dataset.jsonl              # SFT training corpus
├── 1)_Baseline_Evaluation.ipynb
├── 2)_SFT_LoRA_Mistral_7B_v0_2.ipynb
├── 3)_Extrinsic_vs_Intrinsic_Mistral.ipynb
└── README.md
```

Dataset roles, source revisions, checksums, and license notices are documented in [`dataset/README.md`](dataset/README.md).
---

## Setup

### Requirements

All notebooks are designed to run on **Google Colab with an A100 GPU**.

### HuggingFace Access

Some models used in this project are gated and require HuggingFace approval:

| Model | Gated | Required in |
|---|---|---|
| mistralai/Mistral-7B-Instruct-v0.2 | No | Notebooks 1, 2, 3 |
| Qwen/Qwen3-8B | No | Notebook 1 |
| lmsys/vicuna-7b-v1.5 | No | Notebook 1 |
| meta-llama/Meta-Llama-3.1-8B-Instruct | **Yes** | Notebook 1 |
| meta-llama/Llama-Guard-3-8B | **Yes** | Notebook 3 (optional gated guard) |
| [ibm-granite/granite-guardian-3.1-2b](https://huggingface.co/ibm-granite/granite-guardian-3.1-2b) | No | Notebook 3 (ungated alternative) |

To request access, visit each gated model's page on [huggingface.co](https://huggingface.co) and accept the usage terms. In notebook 3, set RUN_LLAMAGUARD3 = False to skip the gated Llama Guard condition and run the ungated Granite Guardian comparison.

### Adding your HuggingFace Token to Colab

1. Open your Colab notebook
2. Click the key icon on the left sidebar (Secrets)
3. Add a new secret with the name `HF_TOKEN` and paste your token as the value
4. Enable notebook access for the secret

### Updating the GitHub path

In each notebook, update `GITHUB_RAW` to point to your repo:

```python
GITHUB_RAW = "https://raw.githubusercontent.com/YOUR_USERNAME/YOUR_REPO/main/dataset"
```

### Google Drive path

The notebooks use Google Drive to save and load the LoRA adapter and evaluation results. In Colab, select a GPU runtime, run the setup cells, and approve the Drive mount prompt. Notebook 3 can run its baseline and Granite Guardian conditions without an SFT adapter or Hugging Face token; it adds SFT when the adapter from notebook 2 exists and adds Llama Guard when `HF_TOKEN` is available and the account has model access. Notebook 3 starts with evaluation batches of 2 for GPU memory headroom; lower `EVAL_BATCH_SIZE` to 1 if the runtime runs out of memory. Update `BASE_DIR` in each notebook if your Drive folder differs:

```python
BASE_DIR = "/content/drive/MyDrive/YOUR_FOLDER"
```

---

## Notebooks

### 1. Baseline Evaluation (`1)_Baseline_Evaluation.ipynb`)

Evaluates four SLMs on the designated HarmBench model evaluation set without any defense applied. This establishes the baseline for comparison with the post-training model.

- HuggingFace login: **Required** (Llama 3.1 is gated)
- Dataset: dataset/model_evaluation/harmbench_test.jsonl loaded from GitHub
- Results: saved to Google Drive with each prompt category

### 2. SFT LoRA Training and Evaluation (`2)_SFT_LoRA_Mistral_7B_v0_2.ipynb`)

**Part A** fine-tunes Mistral-7B-Instruct-v0.2 using LoRA SFT on the adversarial safety corpus. Training runs in full BF16 precision to maximize weight update quality.

**Part B** evaluates the fine-tuned model on the designated AdvBench and safe XSTest test sets using 4-bit NF4 quantization with BF16 compute dtype.

- HuggingFace login: **Optional** (Mistral is not gated)
- Training data: dataset/sft_dataset.jsonl
- Test data: dataset/trained_model_test/advbench_test.jsonl and dataset/trained_model_test/xstest_safe.jsonl
- Adapter: saved to and loaded from Google Drive

LoRA configuration:
- Rank: 16
- Alpha: 32
- Target modules: q_proj, v_proj
- Dropout: 0.05
- Epochs: 5
- Learning rate: 2e-4
- Early stopping patience: 3

### 3. Extrinsic vs. Intrinsic Evaluation (`3)_Extrinsic_vs_Intrinsic_Mistral.ipynb`)

Compares the baseline, LoRA SFT, Llama Guard 3, and Granite Guardian. Both extrinsic guards screen requests before generation and moderate generated responses. Results separately report input blocks, output blocks, harmful-request refusal proxies, and safe-prompt false-refusal proxies, including per-category summaries.

- HuggingFace login: **Required only for Llama Guard 3**; Granite Guardian is ungated
- Datasets: designated HarmBench model evaluation and AdvBench/XSTest trained-model test files loaded from GitHub
- Results: saved to Google Drive

---

## Notes on Quantization and VRAM

Training (notebook 2 Part A) uses full BF16 precision with no quantization to maximize model quality during weight updates.

Evaluation (notebooks 2 Part B and 3) uses 4-bit NF4 quantization. Notebook 3 selects BF16 compute when the Colab GPU supports it and otherwise uses FP16, including on common T4 runtimes.

Notebook 3 quantizes the target and guard models to 4-bit for evaluation. Llama Guard 3 8B has a larger memory footprint; Granite Guardian 3.1 2B is included as a smaller alternative. Actual VRAM requirements vary by runtime and batch size. The refusal-text metrics are proxies, not semantic judgments of response safety.

---

## Author

**Mickhael Niklas**
Informatics Student, Pradita University
mickhaelniklas1@gmail.com
[github.com/Heatbless](https://github.com/Heatbless)
