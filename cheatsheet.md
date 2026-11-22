# SQL to pandas cheat sheet

A reference for translating SQL patterns into pandas, with the mistakes that cost me the most time. Behaviours were checked by running them on pandas 3.0.5, and on 2.3.3 where noted.

## Pre-run checklist

Most wrong answers come from eight avoidable slips, so check these before you trust a result.

1. **Conditions:** use `&`, `|` and `~` with each condition in parentheses. Never `and`, `or` or `not`.
2. **Top N:** set `ascending=False`. Both `sort_values` and `rank` sort ascending by default.
3. **Order:** sort before `shift`, `rank` or `cumsum`. They follow the current row order, not a date.
4. **Column names:** `print(df.columns)` after every `agg` or `merge`. The names are the ones you defined, not the ones you expect.
5. **Assign back:** `round`, `rename`, `sort_values`, `drop` and `fillna` return a new object. Save the result.
6. **Grain:** the `groupby` keys are the partition. Ask whether you want one row per group or every row kept.
7. **Output:** the columns, their names and the sort order match what the problem asked for.
8. **Run and compare:** run the code on the sample data, compare with the expected output, and hand-check one row of any formula.

## SQL to pandas mapping

Each SQL clause has a direct pandas form. The last column is the catch that most often breaks it.

| SQL | pandas | Watch out |
| --- | --- | --- |
| `WHERE a > 1 AND b = 2` | `df.loc[(df['a'] > 1) & (df['b'] == 2)]` | Parentheses around each condition are required. |
| `WHERE x IN (...)` | `df[df['x'].isin([...])]` | `NOT IN` is `~df['x'].isin([...])`. |
| `WHERE x IS NULL` | `df[df['x'].isna()]` | `== None` and `== np.nan` never match. |
| `SELECT a, b` | `df[['a', 'b']]` | Double brackets give a DataFrame. Single brackets give a Series. |
| `SELECT a AS z` | `df.rename(columns={'a': 'z'})` | Returns a copy. Assign it. |
| `SELECT a * 2 AS z` | `df.assign(z=df['a'] * 2)` | Or `df['z'] = df['a'] * 2` to modify in place. |
| `DISTINCT` | `df.drop_duplicates(subset=['a'])` | Keeps the first row for each key. |
| `ORDER BY a DESC, b` | `df.sort_values(['a', 'b'], ascending=[False, True])` | The argument is `by`, not `columns`. Default is ascending. |
| `LIMIT n` | `df.head(n)` | For the top n by value, use `df.nlargest(n, 'col')`. |
| `BETWEEN x AND y` | `df['a'].between(x, y)` | Includes both ends. |
| `LIKE '%x%'` | `df['c'].str.contains('x', na=False)` | Treats the pattern as a regex by default. |
| `CASE WHEN` | `np.where(cond, 'yes', 'no')` or `np.select([c1, c2], [v1, v2], default=v3)` | `np.select` checks conditions in order. See the CASE WHEN section. |
| `GROUP BY` + aggregate | `df.groupby('k', as_index=False).agg(total=('v', 'sum'))` | Named aggregation: the new name is unquoted, the column and function are quoted. |
| `HAVING` | `agg_df[agg_df['total'] > 10]` | A normal filter applied after the aggregation. |
| `COUNT(*)` | `groupby('k').size()` | Counts every row, including nulls. |
| `COUNT(col)` | `groupby('k')['v'].count()` | Skips NaN. |
| `COUNT(DISTINCT col)` | `groupby('k')['v'].nunique()` | Skips NaN. |
| `INNER` or `LEFT JOIN` | `a.merge(b, how='left', on='k')` | Duplicate keys on the right multiply the left rows. |
| `LEFT JOIN ... WHERE b.k IS NULL` | `a.merge(b, how='left', on='k', indicator=True)` then keep `_merge == 'left_only'` | Or `a[~a['k'].isin(b['k'])]` if you need no columns from `b`. |
| `UNION ALL` | `pd.concat([a, b], ignore_index=True)` | Without `ignore_index`, the index repeats. |
| CTE or subquery | A named intermediate variable | One variable per step makes each step easy to print and check. |

## Window functions

A window function keeps every row and adds a value computed within a group. In pandas that is `groupby(...)` followed by a method that returns one value per row. pandas has no `ORDER BY` inside the window, so sort the frame first.

```python
df = df.sort_values(['product_id', 'year'])                  # the ORDER BY
df['prev'] = df.groupby('product_id')['spend'].shift(1)      # the PARTITION BY plus LAG
```

| SQL | pandas | Watch out |
| --- | --- | --- |
| `LAG(x, 1) OVER (PARTITION BY k ORDER BY d)` | `df.groupby('k')['x'].shift(1)` | Sort by `k, d` first. There is no `.lag()`. |
| `LEAD(x, 1)` | `df.groupby('k')['x'].shift(-1)` | The last row of each group is NaN. |
| `ROW_NUMBER()` | `df.groupby('k')['x'].rank(method='first', ascending=False)` | Ties are broken by row order, so exactly one row gets each number. |
| `RANK()` | `rank(method='min', ascending=False)` | Ties share a rank and leave gaps: 1, 1, 3. |
| `DENSE_RANK()` | `rank(method='dense', ascending=False)` | Ties share a rank with no gaps: 1, 1, 2. |
| `SUM(x) OVER (PARTITION BY k)` | `df.groupby('k')['x'].transform('sum')` | Same length as the frame, so you can assign it as a column. |
| Share of group total | `df['x'] / df.groupby('k')['x'].transform('sum')` | Multiply by 100 for a percentage. |
| Running total | `df.groupby('k')['x'].cumsum()` | Sort by the time column first. |
| `FIRST_VALUE(x)` | `df.groupby('k')['x'].transform('first')` | Depends on the current row order. |
| Moving average | `df.groupby('k')['x'].rolling(3).mean().reset_index(level=0, drop=True)` | `rolling` after `groupby` adds the group key to the index. Drop it before assigning back. |
| `NTILE(4)` | `pd.qcut(df['x'], 4, labels=False)` | Gives 0 to 3, not 1 to 4. |
| Growth vs previous row | `df.groupby('k')['x'].pct_change()` | Uses the previous row, so a missing period gives a wrong rate. |

To keep only the top 2 per group, rank first and filter on the rank column with `<= 2`. Using `== 2` returns only the runner-up. To get exactly the third row per group, filter with `== 3`.

### Aggregation or window?

`groupby` alone only splits the rows into groups. What you call next decides the result.

| | Aggregation | Transformation (window-style) |
|---|---|---|
| Examples | `sum`, `count`, `min`, `max`, `agg(...)` | `shift`, `rank`, `cumsum`, `pct_change`, `transform(...)` |
| Each group returns | One value | One value per row in the group |
| Result shape | One row per group | Same rows and index as the original |
| SQL equivalent | `GROUP BY` | `OVER (PARTITION BY ...)` |

## CASE WHEN

The condition goes inside `np.where` or `np.select`, and the result is assigned to a column. Putting the same condition inside `.loc[...]` filters rows instead.

```python
df['click_flag'] = np.where(df['event_type'] == 'click', 1, 0)

df['period'] = np.select(
    [df['timestamp'].dt.year == 2022, df['timestamp'].dt.year == 2021],
    ['current', 'previous'],
    default='older'
)
```

| SQL pattern | pandas | Watch out |
| --- | --- | --- |
| Two-way `CASE WHEN c THEN a ELSE b END` | `np.where(cond, a, b)` | Both branches should be the same type. |
| Multi-way `WHEN ... WHEN ... ELSE` | `np.select(conditions, values, default=...)` | First match wins, so list the most specific condition first. |
| Exact value lookup | `df['x'].map({'A': 4, 'B': 3})` | Unmatched values become NaN. Chain `.fillna(0)` for an ELSE. |
| `ELSE` keeps the existing value | `np.where(cond, new, df['col'])` or `df.loc[cond, 'col'] = new` | Updates only the rows that match. |
| Keep where true, replace elsewhere | `s.where(cond, other)` | The opposite is `s.mask(cond, other)`, which replaces where true. |
| Numeric buckets | `pd.cut(s, bins, labels=...)` | Intervals are open on the left and closed on the right by default. |
| `SUM(CASE WHEN ...)` | Flag column, then `sum` in a named aggregation | `(cond).astype(int)` and `np.where(cond, 1, 0)` give the same flag. |

- **NaN falls through to the else branch.** A NaN compared with `>= 80` is False, so it ends up in the default. If missing values need their own label, test `s.isna()` first in the `np.select` list.
- **Mixed types in `np.where` can fail.** `np.where(cond, 'big', 0)` raised a `DTypePromotionError`. Use `'big'` and `'0'`, or numbers on both sides.
- **`map` returns floats** when some values fail to match, because NaN forces the column to float.

## Syntax gotchas

The patterns below cause most syntax errors. The error table after them tells you what Python is complaining about.

- **Filtering with `.loc`:** `df.loc[condition, ['col1', 'col2']]`. The condition comes first, then one list of columns. Count your brackets.
- **Several conditions:** `(a) & (b)`, never `a and b`.
- **Named aggregation:** `.agg(total=('spend', 'sum'))`. The new name is unquoted. The column and the function are strings.
- **After `groupby().agg()`, the keys become the index.** Add `.reset_index()` or use `as_index=False`.
- **`reset_index(name='x')` works only on a Series**, such as `df.groupby('k')['v'].count().reset_index(name='x')`. On a DataFrame, use `rename`.
- **`rename` needs `columns=`.** Pass one dict: `df.rename(columns={'a': 'z', 'b': 'y'})`. Without `columns=`, it relabels the rows. Several separate dicts raise a `TypeError`.
- **Methods return copies.** `df.sort_values(...)` on its own line does nothing to `df`. Write `df = df.sort_values(...)`.
- **Don't name a column like a method.** `df.diff`, `df.count` and `df.rank` are methods. Use `df['diff']`, or a name such as `difference`.
- **Write long chains inside parentheses**, one step per line, so you can comment out any step to debug it.
- **Setting values:** `df.loc[cond, 'col'] = value`. Chained forms such as `df[cond]['col'] = value` change a copy and the original stays the same.

| Error message | What it means | Fix |
| --- | --- | --- |
| `ValueError: The truth value of a Series is ambiguous` | You used `and`, `or`, `not` or `if` on a Series. | Use `&`, `\|`, `~` with parentheses. |
| `KeyError: 'name'` | That column does not exist, often because of a rename or aggregation. | Print `df.columns`. |
| `AttributeError: ... has no attribute 'lag'` | The method name does not exist in pandas. | Use `shift`. Check this cheat sheet for the right name. |
| `TypeError: ... got an unexpected keyword argument` | Wrong parameter name. | `help(df.sort_values)` or `df.sort_values?` shows the real ones. |
| `ValueError: Index contains duplicate entries, cannot reshape` | `pivot` met two values for one cell. | Use `pivot_table` with an `aggfunc`. |
| `MergeError: Merge keys are not unique` | You asked for a one-to-one or many-to-one merge and the data isn't. | Deduplicate, or fix the join key. |
| `SettingWithCopyWarning` (pandas 2.x) | You assigned into a slice of another frame. | Use `.loc[...]`, or `.copy()` the slice first. Pandas 3 no longer warns, but the original is still not changed. |

## Caveats by command

These behaviours differ from SQL or from what you would guess.

| Command | Caveat |
| --- | --- |
| `merge` | Duplicate keys on one side multiply rows. Check `len()` before and after, or pass `validate='m:1'` to make pandas raise an error. |
| `merge` on NaN keys | Unlike SQL, pandas matches NaN keys to each other. Drop null keys first if that is not what you want. |
| `shift`, `pct_change` | They use the previous row, not the previous period. If a year is missing, the comparison is silently wrong. The first row of each group is NaN. |
| `shift` dtype | A column of integers becomes floats once NaN appears (`1500.0`, not `1500`). |
| `groupby` | Rows with a NaN key are dropped by default. Use `dropna=False` to keep them. |
| `count`, `size`, `nunique` | `count` skips NaN, `size` counts every row, `nunique` counts distinct non-null values. With one NaN in a group of two, `count` gives 1 and `size` gives 2. |
| `sum` of all-NaN | Returns `0`, where SQL gives NULL. Use `sum(min_count=1)` to get NaN. |
| `round` | Rounds halves to the nearest even number. `0.5` becomes `0`, `2.5` becomes `2`, and `0.125.round(2)` is `0.12`. SQL usually rounds half up, so exact halves can differ. |
| `rank`, `nlargest` | Ties are the catch. `rank(method=...)` decides how they share positions. `nlargest(1, ...)` returns the first tied row only, unless you pass `keep='all'`. |
| `sort_values` | The order of tied rows is not guaranteed. Add a second sort key when the output must be stable. |
| `drop_duplicates` | Keeps the first row of each duplicate set. Sort first if you want the latest or the largest. |
| `pivot` | Fails if one cell would get two values. `pivot_table` aggregates them. |
| `str.contains` | The pattern is a regex. On a column with missing values, pass `na=False`. Depending on the version, a mask containing NaN raises an error when you filter with it. |
| `between` | Inclusive on both ends, like SQL. |
| Date `<=` string | `d <= '2022-12-31'` drops a row at `2022-12-31 14:00`. Use `d < '2023-01-01'`. |
| `merge` with a Series | The Series needs a `name`. It becomes the column name. |

## Dates

Convert to datetime first with `pd.to_datetime(df['col'])`. Comparing a string column against a date string gives text comparisons, not date comparisons.

| Need | pandas | Watch out |
| --- | --- | --- |
| Year, month, day | `df['d'].dt.year`, `.dt.month`, `.dt.day` | `dt.month` ignores the year, so filter the year first or use a period. |
| Year-month bucket | `df['d'].dt.to_period('M')` | Good as a groupby key. |
| `DATE_TRUNC('month', d)` | `df['d'].dt.to_period('M').dt.to_timestamp()` | Gives the first day of the month. |
| Text like `2022-08` | `df['d'].dt.strftime('%Y-%m')` | The result is a string, not a date. |
| Day of week | `df['d'].dt.dayofweek` | Monday is 0. |
| One calendar month | `(d >= '2022-08-01') & (d < '2022-09-01')` | Use a strict `<` on the first day of the next month. |
| Days between | `(df['end'] - df['start']).dt.days` | Subtracting gives a Timedelta, not a number. |
| Add days | `df['d'] + pd.Timedelta(days=7)` | |
| Add months | `df['d'] + pd.DateOffset(months=1)` | Jan 31 plus one month is Feb 28, not an error. |
