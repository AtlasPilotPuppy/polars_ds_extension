# Polars Extension for General Data Science Use

A Polars Plugin aiming to simplify common numerical/string data analysis procedures.

# Purpose

The goal of the package is to facilitate data processes and analysis that go beyond standard SQL queries. It incorproates parts of SciPy, NumPy, Scikit-learn, and NLP (NLTK), etc., and treats them as Polars queries so that they can be run in parallel, in group_by contexts, etc. 

This package does not attempt to replace any of the aforementioned libraries. Only a small subset of commonly used and suitable functions will be implemented. 

All the queries provided by this package should work with eager and lazy Polars. By relying on the power of Rust and Polars, we can thus keep the number of dependencies at a minimum (most dependency issues are checked at compile time and all builds must pass tests on all platforms), keep our code as clean as possible while getting better runtime performance in most cases.

See examples [here](./examples/basics.ipynb).

Read the docs [here](https://polars-ds-extension.readthedocs.io/en/latest/).

**Currently in Beta. Feel free to submit feature requests in the issues section of the repo. This library will only depend on python Polars and will try to be as stable as possible for polars?=0.20.6. Exceptions will be made when Polars's update forces changes in the plugins.**

Disclaimer: this package is not tested with Polars streaming mode and is not designed to work with data so big that data has to be streamed. The recommended usage will be for datasets of size 1k to 2-3mm rows. Performance problems within this range will be given more priority than performance problems for data outside this range.

## Getting Started
```bash
pip install polars_ds
```

and 

```python
import polars_ds as pld
```
when you want to use the namespaces provided by the package.

## Examples

In-dataframe statistical testing
```python
df.select(
    pl.col("group1").stats.ttest_ind(pl.col("group2"), equal_var = True).alias("t-test"),
    pl.col("category_1").stats.chi2(pl.col("category_2")).alias("chi2-test"),
    pl.col("category_1").stats.f_test(pl.col("group1")).alias("f-test")
)

shape: (1, 3)
┌───────────────────┬──────────────────────┬────────────────────┐
│ t-test            ┆ chi2-test            ┆ f-test             │
│ ---               ┆ ---                  ┆ ---                │
│ struct[2]         ┆ struct[2]            ┆ struct[2]          │
╞═══════════════════╪══════════════════════╪════════════════════╡
│ {-0.004,0.996809} ┆ {37.823816,0.386001} ┆ {1.354524,0.24719} │
└───────────────────┴──────────────────────┴────────────────────┘
```

Generating random numbers according to reference column
```python
df.with_columns(
    # Sample from normal distribution, using reference column "a" 's mean and std
    pl.col("a").stats.sample_normal().alias("test1") 
    # Sample from uniform distribution, with low = 0 and high = "a"'s max, and respect the nulls in "a"
    , pl.col("a").stats.sample_uniform(low = 0., high = None, respect_null=True).alias("test2")
).head()

shape: (5, 3)
┌───────────┬───────────┬──────────┐
│ a         ┆ test1     ┆ test2    │
│ ---       ┆ ---       ┆ ---      │
│ f64       ┆ f64       ┆ f64      │
╞═══════════╪═══════════╪══════════╡
│ null      ┆ 0.459357  ┆ null     │
│ null      ┆ 0.038007  ┆ null     │
│ -0.826518 ┆ 0.241963  ┆ 0.968385 │
│ 0.737955  ┆ -0.819475 ┆ 2.429615 │
│ 1.10397   ┆ -0.684289 ┆ 2.483368 │
└───────────┴───────────┴──────────┘
```

Blazingly fast string similarity comparisons. (Thanks to [RapidFuzz](https://docs.rs/rapidfuzz/latest/rapidfuzz/))
```python
df.select(
    pl.col("word").str2.levenshtein("asasasa", return_sim=True).alias("asasasa"),
    pl.col("word").str2.levenshtein("sasaaasss", return_sim=True).alias("sasaaasss"),
    pl.col("word").str2.levenshtein("asdasadadfa", return_sim=True).alias("asdasadadfa"),
    pl.col("word").str2.fuzz("apples").alias("LCS based Fuzz match - apples"),
    pl.col("word").str2.osa("apples", return_sim = True).alias("Optimal String Alignment - apples"),
    pl.col("word").str2.jw("apples").alias("Jaro-Winkler - apples"),
)
shape: (5, 6)
┌──────────┬───────────┬─────────────┬────────────────┬───────────────────────────┬────────────────┐
│ asasasa  ┆ sasaaasss ┆ asdasadadfa ┆ LCS based Fuzz ┆ Optimal String Alignment  ┆ Jaro-Winkler - │
│ ---      ┆ ---       ┆ ---         ┆ match - apples ┆ - apple…                  ┆ apples         │
│ f64      ┆ f64       ┆ f64         ┆ ---            ┆ ---                       ┆ ---            │
│          ┆           ┆             ┆ f64            ┆ f64                       ┆ f64            │
╞══════════╪═══════════╪═════════════╪════════════════╪═══════════════════════════╪════════════════╡
│ 0.142857 ┆ 0.111111  ┆ 0.090909    ┆ 0.833333       ┆ 0.833333                  ┆ 0.966667       │
│ 0.428571 ┆ 0.333333  ┆ 0.272727    ┆ 0.166667       ┆ 0.0                       ┆ 0.444444       │
│ 0.111111 ┆ 0.111111  ┆ 0.090909    ┆ 0.555556       ┆ 0.444444                  ┆ 0.5            │
│ 0.875    ┆ 0.666667  ┆ 0.545455    ┆ 0.25           ┆ 0.25                      ┆ 0.527778       │
│ 0.75     ┆ 0.777778  ┆ 0.454545    ┆ 0.25           ┆ 0.25                      ┆ 0.527778       │
└──────────┴───────────┴─────────────┴────────────────┴───────────────────────────┴────────────────┘
```

Even in-dataframe nearest neighbors queries! 😲
```python
df.with_columns(
    pl.col("id").num.knn_ptwise(
        pl.col("val1"), pl.col("val2"), 
        k = 3, dist = "haversine", parallel = True
    ).alias("nearest neighbor ids")
)

shape: (5, 6)
┌─────┬──────────┬──────────┬──────────┬──────────┬──────────────────────┐
│ id  ┆ val1     ┆ val2     ┆ val3     ┆ val4     ┆ nearest neighbor ids │
│ --- ┆ ---      ┆ ---      ┆ ---      ┆ ---      ┆ ---                  │
│ i64 ┆ f64      ┆ f64      ┆ f64      ┆ f64      ┆ list[u64]            │
╞═════╪══════════╪══════════╪══════════╪══════════╪══════════════════════╡
│ 0   ┆ 0.804226 ┆ 0.937055 ┆ 0.401005 ┆ 0.119566 ┆ [0, 3, … 0]          │
│ 1   ┆ 0.526691 ┆ 0.562369 ┆ 0.061444 ┆ 0.520291 ┆ [1, 4, … 4]          │
│ 2   ┆ 0.225055 ┆ 0.080344 ┆ 0.425962 ┆ 0.924262 ┆ [2, 1, … 1]          │
│ 3   ┆ 0.697264 ┆ 0.112253 ┆ 0.666238 ┆ 0.45823  ┆ [3, 1, … 0]          │
│ 4   ┆ 0.227807 ┆ 0.734995 ┆ 0.225657 ┆ 0.668077 ┆ [4, 4, … 0]          │
└─────┴──────────┴──────────┴──────────┴──────────┴──────────────────────┘
```

And a lot more!

# Credits

1. Rust Snowball Stemmer is taken from Tsoding's Seroost project (MIT). See [here](https://github.com/tsoding/seroost)
2. Some statistics functions are taken from Statrs (MIT). See [here](https://github.com/statrs-dev/statrs/tree/master)

# Other related Projects

1. Take a look at our friendly neighbor [functime](https://github.com/TracecatHQ/functime)
2. String similarity metrics is soooo fast and easy to use because of [RapidFuzz](https://github.com/maxbachmann/rapidfuzz-rs)