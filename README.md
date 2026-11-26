# Economic Rationality of Gemma 3n

This repository runs the LLM experiments of

> Chen, Y., Liu, T. X., Shan, Y. and Zhong, S. (2023). The emergence of economic rationality of GPT. *PNAS* 120(51), e2316205120.

on Google's open-weight model **Gemma 3n E4B** (`google/gemma-3n-E4B-it`).

The code is adapted from the gpt-4o-mini replication of the same paper: [ZQ-77777/gpt-rationality-replication](https://github.com/ZQ-77777/gpt-rationality-replication). The design, prompts (Appendix A of the paper, verbatim), parser and analysis are the same; the changes are listed below.

Author: Zhang Qi

## Status

Code ready; no Gemma 3n results yet.

## What changed from the gpt-4o-mini version

| Change | Why |
|---|---|
| The model is served locally (vLLM, BF16) instead of through a hosted API | No hosted provider serves Gemma 3n any more (OpenRouter and NVIDIA's free endpoint have stopped). Local weights also fix the model version and precision |
| `LLM_ROLE_MODE=system_as_user` | Gemma's chat template has no system role, and turns must alternate user/assistant starting with user. The paper's system message is sent as the first user turn; the paper's assistant message and every later turn are unchanged. The stored transcript keeps the paper's layout |
| `LLM_RUN_TAG` | Keeps each model's data and results apart: `data/<tag>/`, `results_<tag>/` |
| Each subject records the endpoint and role mode | Traceability |
| Permanent API errors (400, 401, 403, 404) stop at once instead of being retried | A wrong model name or key no longer waits through eight retries |

## Hardware

Formal runs need BF16, which requires an Ampere-or-newer GPU (for example A30, L20, L4, A100, RTX 30/40 series) with at least 24 GB of memory. A free Colab T4 has no native BF16, and Gemma 3n is known to overflow in FP16, so a T4 is used only for debugging with the quantized Ollama model.

## Setup

```bash
pip install -r requirements.txt          # analysis and client
pip install vllm                          # only on the GPU machine that serves the model
cp .env.example .env                      # then edit; never commit .env
python scripts/00_selftest.py             # must print ALL PASSED
```

The model is gated on Hugging Face: accept the Gemma terms on the model page and create a Read token. Record the commit id you use in `archive/model_revisions.md`.

## Running

**1. Debug run (quantized, data go to `data/gemma3n-debug/`).** Use block B of `.env.example`, or open `notebooks/gemma3n_smoke_colab.ipynb` in Colab with a T4 GPU.

```bash
ollama pull gemma3n:e4b
python scripts/01_smoke_test.py --condition baseline
```

**2. Formal run (BF16, data go to `data/gemma3n-e4b/`).** Use block A of `.env.example`.

```bash
vllm serve google/gemma-3n-E4B-it --revision <commit> --dtype bfloat16 \
     --max-model-len 32768 --served-model-name gemma-3n-e4b-it --port 8000 &
python -c "import config as C; print(C.MODEL, C.RUN_TAG, C.RAW_DIR, C.ROLE_MODE)"   # check before running
python scripts/01_smoke_test.py --condition baseline
python scripts/02_run_llm.py --conditions baseline temp_0.5 temp_1 price_framing discrete --workers 8
python scripts/02_run_llm.py --conditions female male young_child elderly elementary_edu college_edu asian african_american --reps 30 --workers 8
```

On a SLURM cluster, start the server and the run in the same job script, and wait for `curl -s localhost:8000/v1/models` to answer before starting `02_run_llm.py`.

**3. Analysis.** With the same `.env`:

```bash
python scripts/07_import_authors.py data/data.csv   # authors' human and GPT-3.5 data, once
python scripts/03_indices.py
python scripts/04_estimate.py
python scripts/05_tests.py
python scripts/06_figures.py
python scripts/12_offmenu.py
python scripts/14_parse_review.py                   # exports replies read by weaker parser rules for hand checking
```

Results are written to `results_<tag>/`.

## Repository layout

```
config.py              design constants, conditions, API, run tag and role mode (read from .env)
src/
  prices.py            budget lines (seeded per subject, identical across conditions)
  prompts_paper.py     the paper's prompts, verbatim from Appendix A (default)
  prompts.py           paraphrased prompt set (sensitivity check)
  prompt_api.py        selects the prompt set
  parse.py             reply -> allocation (parser version 3)
  parse_v2.py          parser version 2, kept for scripts/14_parse_review.py
  llm_client.py        OpenAI-compatible client; role adaptation for Gemma
  experiment.py        one subject = one 25-round conversation
  rp.py                GARP, CCEI, HMI, MPI, MCI, Bronars
  demand.py            Spearman test of downward-sloping demand
  estimate.py          two-limit Tobit for CES (alpha, rho)
  stats.py             Welch t, Levene, OLS with HC1 standard errors
  dataio.py            loading model data and the authors' data
scripts/               00-12 and 14, as in the gpt-4o-mini repository
notebooks/             Colab notebook for the debug run
archive/               model commit ids, slicing configurations, MatFormer Lab notebook
data/                  ignored by git; see data/README.md
```

## Data not in this repository

- `.env` holds keys and settings. Never commit it.
- `data/` holds raw transcripts and the authors' file. The authors' `data.csv` is downloaded from the link in the paper and is not redistributed here.
