# Python Practice

Interview practice: SQL-style data problems solved in **pandas**.

## Why this repo

I practise the kind of data problems that come up in analyst interviews (joins, window functions, conditional aggregation, streaks) and solve them in pandas as well as SQL, so I can do them fluently in either. Each problem follows the same routine:

1. Read the problem and the expected output.
2. Solve it without hints.
3. Run it and compare with the expected output.
4. Write down what went wrong, and add it to the [cheat sheet](CHEATSHEET.md).

The repo is a working log, so the notebooks keep my own notes and corrections.

## What's here

| File | What it is |
|------|------------|
| [`notebooks/`](notebooks/) | One notebook per practice session, named by date. Each problem has the question, sample data, my solution and its output. |
| [`CHEATSHEET.md`](CHEATSHEET.md) | SQL to pandas reference: pre-run checklist, window functions, `CASE WHEN`, syntax gotchas, caveats, dates. |

The problems are paraphrased from common interview questions, and the sample data is my own. Problem 9 is an original problem.

## Log

| # | Problem | Topics | Notebook |
|---|---------|--------|----------|
| 1 | Pages with zero likes | anti-join, `merge(indicator=True)`, `isin` | [2026-11-21](notebooks/2026-11-21_python_practice.ipynb) |
| 2 | Top 2 senders by messages in a month | date range filter, groupby, top N | [2026-11-21](notebooks/2026-11-21_python_practice.ipynb) |
| 3 | Spread between max and min issuance per card | named aggregation, derived column | [2026-11-21](notebooks/2026-11-21_python_practice.ipynb) |
| 4 | Top 2 products per category | `groupby().rank()`, `ROW_NUMBER` vs `RANK` | [2026-11-21](notebooks/2026-11-21_python_practice.ipynb) |
| 5 | Year-over-year spend growth | sort then `shift` (LAG), percentage change | [2026-11-21](notebooks/2026-11-21_python_practice.ipynb) |
| 6 | Users active in two consecutive months | `pivot_table`, `dropna`, set intersection | [2026-11-21](notebooks/2026-11-21_python_practice.ipynb) |
| 7 | Each user's third transaction | window rank within a group | [2026-11-22](notebooks/2026-11-22_python_practice.ipynb) |
| 8 | Employees earning more than their manager | self-merge, `rename(columns=...)` | [2026-11-22](notebooks/2026-11-22_python_practice.ipynb) |
| 9 | Longest login streak | gaps and streaks: flag + `cumsum` | [2026-11-22](notebooks/2026-11-22_python_practice.ipynb) |
| 10 | Click-through rate by app | `np.where` flags, conditional aggregation | [2026-11-22](notebooks/2026-11-22_python_practice.ipynb) |

## Run

```bash
pip install -r requirements.txt
jupyter notebook
```

Each notebook is self-contained: every problem builds its own data in the cell above its solution.
