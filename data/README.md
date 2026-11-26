# data/

Ignored by git, except for this file.

| Path | Created by | Content |
|---|---|---|
| `<tag>/raw/<condition>.jsonl` | `02_run_llm.py` | One line per round: prices, reply, parsed allocation, validity flag |
| `<tag>/meta/<condition>.jsonl` | `02_run_llm.py` | One line per subject: full transcript, token usage, comprehension answers, endpoint and role mode |
| `data.csv` | download | The authors' subject-level data, from the data link in the paper |
| `authors_*.csv` | `07_import_authors.py` | The authors' data in this project's format |

`<tag>` is `LLM_RUN_TAG` from `.env`: `gemma3n-e4b` for formal runs, `gemma3n-debug` for quantized test runs.
To compare with the gpt-4o-mini runs, copy that project's `data/raw` to `data/<tag>/extra/gpt-4o-mini/`.
