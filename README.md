# Polars vs Pandas Benchmark

A hands-on performance comparison between [Polars](https://pola.rs/) and [pandas](https://pandas.pydata.org/), run on synthetically generated datasets of varying size.

## What's benchmarked

The notebook (`polars.ipynb`) measures execution time and memory usage for both libraries across:

- **I/O**: writing and reading CSV (and Parquet, where the optional engine is available)
- **Basic operations**: `SELECT`, `FILTER`, `SORT`, `GROUP BY`, tested at multiple dataset sizes to see how each library scales
- **Complex aggregations / window functions**: multi-level aggregations and cumulative window operations
- **Joins**: inner and left joins between a large primary dataset and a small lookup table
- **A stress test** on a very large (multi-million row) dataset

All data is synthetically generated inside the notebook (random numeric, categorical, date, and text columns) with a fixed random seed, so results are reproducible but not representative of any real-world dataset.

## How to run

1. Create an environment with the required packages:
   ```bash
   pip install pandas polars numpy matplotlib seaborn psutil pyarrow
   ```
2. Open `polars.ipynb` in Jupyter / JupyterLab / VS Code.
3. Run all cells from top to bottom (or `jupyter nbconvert --to notebook --execute polars.ipynb`). Dataset sizes are configurable near the top of each pass if you want to try smaller or larger volumes.

The notebook contains two benchmark passes (an earlier, more exhaustive run, and a shorter follow-up pass with its own results table and chart). Both passes now run end-to-end with no errors — verified by executing the full notebook from a clean virtual environment (Python 3.12, pandas 3.1.1, polars 2.1.0, numpy 2.5.5, pyarrow 26.0.1) on 2026-11-16. Total wall-clock time for the full notebook (56 + 6 benchmark results, including a 10M-row stress test) was about 3 minutes on an 8-core / 16GB machine.

Two things that were previously broken have been fixed:
- **Parquet I/O**: now runs correctly with `pyarrow` installed (it was simply missing from the original environment).
- **Complex aggregation / window functions**: the notebook originally called `DataFrame.round()` directly on a Polars groupby-agg result and `Expr.cumsum()` — both removed/renamed in current Polars. Fixed to round only the float columns via `.with_columns(pl.col(pl.Float64).round(4))` and to use `Expr.cum_sum()`. The row-number window expression was also adjusted to be 1-indexed (`pl.int_range(pl.len()) + 1`), matching pandas' `cumcount() + 1` in the same cell.
- **10M-row stress test**: the first pass's 10M-row stress test (cell 8) was already correctly sized and runs in well under a minute on a 16GB machine — it wasn't the problem. The real risk was the *second* pass, which generated an unused 50,000,000-row dataset (`molto_grande`) that is never referenced by any later cell in that pass — pure wasted memory/time with real potential to exhaust RAM or hang. It's been scoped down to 2,000,000 rows.

## What the results show

Numbers below are from the actual re-executed run described above (synthetic data, single machine, single run — expect run-to-run noise, especially on sub-millisecond operations).

**First pass** (28 operation/dataset comparisons, sizes 10k–10M rows):

- **I/O**: on a 100k-row CSV, Polars wrote ~14x faster and read ~1.8x faster than pandas; on 1M rows, Polars write was ~32x faster and read ~1.5x faster. Parquet showed a similar pattern (Polars write ~9x and ~6x faster at 100k/1M rows respectively).
- **Basic operations** (`SELECT`/`FILTER`/`SORT`/`GROUP BY`): Polars was faster in most cases and the advantage generally grew with size, e.g. `SORT` on the 1M-row dataset was ~5x faster with Polars. At the smallest size (10k rows) a couple of operations were close enough that measurement noise flipped the result — `FILTER` at 10k rows came out slightly faster in pandas this run, a reminder that sub-millisecond timings on tiny data aren't a reliable signal either way.
- **Complex aggregation / window functions** (now fixed and working): on the 1M-row dataset, the multi-level aggregation was ~2.7x faster with Polars, and the window-function test (cumulative sum, dense rank, row number, all partitioned by category) was ~7.8x faster with Polars.
- **Joins**: on 1M rows, Polars completed left joins ~2.9x faster than pandas; inner joins were close, with pandas winning that particular run (~0.6x) — again plausibly within noise for a join this size, not a reliable directional signal.
- **10M-row stress test**: creating a 10M-row Polars `DataFrame` from Python dict/array input was slower than pandas in this run (~17s vs ~13s) — object/string-heavy construction from plain Python containers is a case where pandas' maturity can win. But the subsequent chained operation (filter → group-by → aggregate → sort → head) on that same 10M-row data was ~2x faster with Polars (0.35s vs 0.72s), which is the more realistic "what you'd actually do with the library" comparison.
- Overall: Polars was faster in 25 of 28 comparisons (89%), with an average speedup of ~5.1x and a max of ~32x (CSV write).

**Second pass** (a shorter, independent re-run on the 1M-row dataset): Polars was faster in all 3 comparisons — `SELECT` ~7.2x, `FILTER` ~3.3x, `GROUP BY` ~1.9x faster — for an average of ~4.1x.

Overall takeaway, consistent with the wider ecosystem experience: Polars' advantage is small and sometimes noisy on tiny datasets or single-shot DataFrame construction, but becomes substantial and consistent on filter/sort/group-by/window-function workloads as data volume and query complexity increase, thanks to its multi-threaded, columnar execution engine. This is a toy benchmark on synthetic data run once on one machine, not a rigorous or production-grade study — treat the numbers as directional, not authoritative, and expect some variance if you re-run it yourself.

## License

MIT — see [LICENSE](LICENSE).
