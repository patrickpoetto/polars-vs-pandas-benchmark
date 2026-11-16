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
   pip install pandas polars numpy matplotlib seaborn psutil
   # optional, needed for the Parquet I/O cells:
   pip install pyarrow
   ```
2. Open `polars.ipynb` in Jupyter / JupyterLab / VS Code.
3. Run all cells from top to bottom. Dataset sizes are configurable near the top of the notebook if you want to try smaller or larger volumes.

Note: the notebook contains two benchmark passes (an earlier, more exhaustive run and a shorter follow-up run with a final results table and chart). A couple of cells reflect known rough edges from the original exploration — the Parquet cells raise an error if `pyarrow`/`fastparquet` isn't installed, and the complex-aggregation/window-function cell hits a couple of API mismatches (e.g. a `round`/`cumsum` call that doesn't match the installed library version) rather than producing timing numbers. These are left in place as-is since they're part of the recorded run; install the optional dependency or adjust the calls if you want those sections to execute cleanly.

## What the results show

On the executed runs (dataset sizes from 10k up to 1M rows), Polars was consistently faster than pandas, with the gap widening as dataset size grew:

- **I/O**: on a 100k-row CSV, Polars wrote ~9x faster and read ~4x faster than pandas; on 1M rows, Polars write was ~17x faster and read ~8x faster.
- **Basic operations** (`SELECT`/`FILTER`/`SORT`/`GROUP BY`): Polars was faster across the board, and the advantage grew with size — e.g. on the 1M-row dataset, `SORT` was ~8x faster and `GROUP BY` ~3x faster with Polars.
- **Joins**: on 1M rows, Polars completed inner joins ~4x faster and left joins ~10x faster than pandas, though in that run Polars reported higher peak memory for the join step — likely an artifact of how/when memory was sampled rather than a general rule.
- **Summary pass**: a consolidated run on 1M rows recorded an average Polars speedup of ~5.6x across `SELECT`/`FILTER`/`GROUP BY`, with `SELECT` showing the largest gain (~12x) and `FILTER` the smallest (~1.2x).

Overall takeaway consistent with the wider ecosystem experience: Polars' advantage is modest on small/simple operations but becomes substantial as data volume and query complexity increase, largely thanks to its multi-threaded, columnar execution engine. This is a toy benchmark on synthetic data, not a rigorous or production-grade study — treat the numbers as directional, not authoritative.

## License

MIT — see [LICENSE](LICENSE).
