# Python Practice

Interview practice: data problems solved in **pandas**.

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
| [`CHEATSHEET.md`](CHEATSHEET.md) | this is a claude generated cheatsheet which lists the mistakes I often did while solving the problems and SQL to pandas reference: pre-run checklist, window functions, `CASE WHEN`, syntax gotchas, caveats, dates. |

The problems are paraphrased from common interview questions

## Run

```bash
pip install -r requirements.txt
jupyter notebook
```

Each notebook is self-contained: every problem builds its own data in the cell above its solution.
