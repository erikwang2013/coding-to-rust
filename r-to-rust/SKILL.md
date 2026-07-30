---
name: r-to-rust
description: Use when migrating R codebases to Rust — covers data.frame to polars, dplyr to Iterator combinators, ggplot2 to plotters, S3 dispatch to traits, Shiny to Leptos/Axum, and incremental migration via extendr. Includes canonical code patterns, common mistakes, and reference implementations.
updated: 2026-07-30
---

# R to Rust Migration Guide

## Architecture Mapping

R's GNU R interpreter (lazy evaluation, S3/S4/R6 object systems, C/Fortran computational core via `.C`/`.Fortran`/`.Call`) maps to Rust's AOT-compiled native binary. Where R relies on a read-eval-print loop for interactive data exploration and copy-on-write semantics for safety, Rust provides zero-cost abstractions with explicit ownership. The R package ecosystem (CRAN, BioConductor) with its `install.packages()` workflow maps to Cargo and crates.io — each R package becomes a Cargo dependency with compile-time version resolution.

The fundamental transformation: R's `data.frame` (columnar, list-of-vectors) becomes `polars::DataFrame` with lazy evaluation and query optimization; R's formula interface (`y ~ x1 + x2`) becomes builder-pattern APIs or declarative macros; R's vectorized operations (automatically recycled across inputs) become explicit `ndarray` array operations or `Iterator` chains. For Shiny web applications, the R server function maps to an Axum handler with shared application state. For interactive exploration during migration, pair Rust binaries with `evcxr` (Rust REPL in Jupyter) or keep R as the visualization layer via the `extendr` FFI bridge.

| R Concept | Rust Equivalent |
|---|---|
| R interpreter (CRAN R) | rustc + LLVM (AOT) |
| CRAN packages | Cargo + crates.io |
| RStudio IDE | VS Code + rust-analyzer / CLion + Rust plugin |
| R Markdown / Quarto | mdBook + Rust code blocks / custom SSG |
| Shiny (web apps) | Leptos / Yew / Actix-web + HTMX |
| Rcpp (C++ bridge) | `cpp` crate / `bindgen` + `cc` crate |
| R6 / Reference classes | `struct` + `impl` with `&mut self` |
| S3 method dispatch | Trait-based dispatch (compile-time) or enum dispatch |
| Lazy evaluation / promises | Closures / `LazyCell` / `LazyLock` |
| `.RData` / `.RDS` | `serde` + bincode / parquet / arrow IPC |
| Global environment | No global mutable state — pass data explicitly |

R's strength is interactive data exploration. For migrated Rust code, pair with `evcxr` (Rust REPL) for exploratory workflows or expose Rust functions to R via the `extendr` crate for incremental migration.

## Type System Mapping

R is dynamically typed with a strong vectorized paradigm. Rust requires concrete types.

| R Type | Rust Type | Notes |
|---|---|---|
| `numeric` (double) | `f64` | R's default number type |
| `integer` | `i32` | R's 32-bit int |
| `logical` (TRUE/FALSE/NA) | `Option<bool>` | NA becomes `None` |
| `character` | `String` / `&str` | |
| `factor` | `enum` + lookup / `HashMap<String, u32>` | Encode levels as enum variants |
| `vector` (atomic) | `Vec<T>` / `ndarray::Array1<T>` | Vec for general; ndarray for numerical |
| `matrix` | `ndarray::Array2<T>` / `nalgebra::DMatrix<f64>` | ndarray for general N-dim; nalgebra for linear algebra |
| `array` (n-dim) | `ndarray::Array<T, D>` | |
| `list` | `Vec<Box<dyn Any>>` / `enum` / `Vec<serde_json::Value>` | Prefer enum for known types |
| `data.frame` | `polars::DataFrame` / `Vec<Struct>` | polars is the closest analogue |
| `tibble` | `polars::LazyFrame` | Lazy evaluation + columnar storage |
| `NULL` | `Option<T>` | |
| `NA` | `Option<T>` | R's missing value sentinel |
| `NaN` | `f64::NAN` | |
| `Inf` / `-Inf` | `f64::INFINITY` / `f64::NEG_INFINITY` | |
| `S4 class` | `struct` with validation in constructor | |
| `R6 class` | `struct` + `impl` with `&mut self` methods | Reference semantics preserved |
| `formula` y ~ x1 + x2 | Custom parser / declarative macro / builder pattern | |
| `environment` | Module scope / `HashMap<String, Value>` | |

### Data Frame to Polars

```rust
use polars::prelude::*;
use std::fs::File;

// R:
// df <- read.csv("data.csv")
// df$new_col <- df$a + df$b
// filtered <- df[df$value > 10, ]

// Rust:
fn process_csv(path: &str) -> PolarsResult<DataFrame> {
    let mut df = CsvReadOptions::default()
        .try_into_reader_with_file_path(Some(path.into()))?
        .finish()?;

    // 添加新列
    let a: &Column = df.column("a")?;
    let b: &Column = df.column("b")?;
    let new_col: Column = (a.as_materialized_series()
        .f64()?
        .into_iter()
        .zip(b.as_materialized_series().f64()?.into_iter())
        .map(|(a, b)| match (a, b) {
            (Some(a), Some(b)) => Some(a + b),
            _ => None,
        })
        .collect::<Float64Chunked>()
        .into_column());
    df.with_column(new_col.with_name("new_col".into()))?;

    // 过滤行
    let mask = df.column("value")?
        .as_materialized_series()
        .f64()?
        .gt(10.0)?;
    let filtered = df.filter(&mask)?;

    Ok(filtered)
}
```

## Memory & Ownership Model

R uses a garbage collector with copy-on-write semantics. Rust uses ownership with explicit borrowing.

| R Memory Pattern | Rust Translation |
|---|---|
| Copy-on-write (automatic) | Explicit `.clone()` or `Cow<T>` |
| Garbage collection | Ownership — no GC overhead |
| Global assignment `<<-` | `Rc<RefCell<T>>` / `Arc<RwLock<T>>` (discouraged) |
| Environments as hash maps | `HashMap` + explicit scoping |
| `gc()` | Not needed — deterministic drop |
| `rm(x)` / `gc()` | `drop(x)` or scope exit |
| `tracemem()` (copy tracking) | Borrow checker guarantees at compile time |
| Lazy evaluation of arguments | `FnOnce() -> T` closures / `LazyLock` |
| `promises` package / future | Explicit `async` / `Future` |

### Copy-on-Write Awareness

```rust
// R — COW 自动发生:
// x <- 1:5
// y <- x        # 不复制（引用）
// y[1] <- 99    # 现在复制，修改 y

// Rust — 用户显式控制:
let x = vec![1, 2, 3, 4, 5];
let y = &x;              // 借用，不复制
// y[0] = 99;            // 编译错误: 不可变借用
let mut y = x.clone();   // 显式复制
y[0] = 99;               // OK
// 或使用 Cow:
use std::borrow::Cow;
let mut z = Cow::Borrowed(&x);
z.to_mut()[0] = 99;      // 仅在需要修改时复制
```

## Concurrency / Async Translation

R is fundamentally single-threaded (with some package exceptions). Rust provides full concurrency.

| R Concurrency | Rust Equivalent |
|---|---|
| `apply()` / `lapply()` | `Iterator::map()` (sequential) |
| `mclapply()` (parallel) | `rayon::par_iter().map()` |
| `future` + `promises` | `async` / `await` + tokio |
| `foreach` + `doParallel` | `rayon::scope` / `crossbeam::scope` |
| `%dopar%` (foreach backend) | `rayon::join` / `rayon::par_bridge` |
| `callr` (background R session) | `tokio::task::spawn_blocking` |
| `parallel::makeCluster()` | `rayon::ThreadPool` |
| `parallel::parApply` | `rayon::par_iter()` |
| `RcppParallel` | `rayon` (same underlying TBB-like approach) |
| `socketConnection` | `tokio::net::TcpStream` |

### Vectorized Operations with Rayon

```rust
use rayon::prelude::*;
use ndarray::Array1;

// R:
// result <- sapply(1:1000000, function(x) sqrt(x) * log(x + 1))

// Rust — 顺序:
fn compute_sequential(n: usize) -> Vec<f64> {
    (1..=n).map(|x| (x as f64).sqrt() * (x as f64 + 1.0).ln()).collect()
}

// Rust — 并行（适合 CPU 密集型）:
fn compute_parallel(n: usize) -> Vec<f64> {
    (1..=n).into_par_iter()
        .map(|x| (x as f64).sqrt() * (x as f64 + 1.0).ln())
        .collect()
}

// 使用 ndarray 向量化（SIMD）:
fn compute_ndarray(n: usize) -> Array1<f64> {
    let x = Array1::linspace(1.0, n as f64, n);
    x.mapv(|v| v.sqrt() * (v + 1.0).ln())
}
```

## Build System & Dependencies

| R Tool | Rust Equivalent |
|---|---|
| `install.packages("pkg")` | `cargo add <crate>` |
| `library(pkg)` | `use crate_name;` |
| `devtools::install_github()` | `[dependencies]` with git URL |
| `renv` / `packrat` | `Cargo.lock` (deterministic builds) |
| `DESCRIPTION` file | `Cargo.toml` |
| `NAMESPACE` | `pub` / `pub(crate)` visibility |
| `R CMD check` | `cargo check` + `cargo clippy` |
| `R CMD build` | `cargo build --release` |
| `testthat` / `tinytest` | `#[cfg(test)] mod tests` + `cargo test` |
| `roxygen2` | `///` doc comments + `cargo doc` |
| `vignette` (Sweave) | mdBook / `cargo doc` with examples |
| R_HOME / library paths | `CARGO_HOME` / `target/` |
| BLAS / LAPACK linkage | `blas-src` / `lapack-src` feature flags |

### Cargo.toml for a Data Science Project

```toml
[package]
name = "my-analysis"
version = "0.1.0"
edition = "2021"

[dependencies]
polars = { version = "0.45", features = ["lazy", "csv", "parquet", "ipc", "ndarray"] }
ndarray = { version = "0.16", features = ["rayon"] }
ndarray-stats = "0.6"
nalgebra = "0.33"
linregress = "0.5"
rand = "0.8"
rand_distr = "0.4"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
csv = "1.3"
arrow = "53"
rayon = "1.10"
plotters = { version = "0.3", features = ["evcxr", "line_series"] }
tracing = "0.1"
thiserror = "2"
smartcore = "0.3"  # ML 库，类似 caret/tidymodels

[dev-dependencies]
approx = "0.5"  # 浮点数近似断言，类似 all.equal()
rstest = "0.22"
```

## Standard Library & Ecosystem Mapping

| R Library | Rust Equivalent |
|---|---|
| `dplyr::filter()` | `polars::DataFrame::filter()` |
| `dplyr::select()` | `polars::DataFrame::select()` |
| `dplyr::mutate()` | `polars::DataFrame::with_column()` |
| `dplyr::arrange()` | `polars::DataFrame::sort()` |
| `dplyr::group_by()` + `summarise()` | `polars::GroupBy::agg()` |
| `dplyr::left_join()` | `polars::DataFrame::join()` |
| `tidyr::pivot_longer()` | `polars::DataFrame::melt()` |
| `tidyr::pivot_wider()` | `polars::DataFrame::pivot()` |
| `purrr::map()` | `Iterator::map()` |
| `purrr::map2()` | `Iterator::zip().map()` |
| `purrr::reduce()` | `Iterator::fold()` / `reduce()` |
| `purrr::walk()` | `for_each()` |
| `ggplot2` | `plotters` crate |
| `plotly` / interactive gg | `plotters` + WASM / `egui` plot widget |
| `stringr` | `String` methods + `regex` crate |
| `lubridate` | `chrono` / `time` crate |
| `forcats` (factors) | enum + match statements |
| `readr::read_csv()` | `csv::Reader` / `polars::CsvReader` |
| `readxl` | `calamine` crate |
| `haven` (SPSS/SAS/Stata) | `readstat` crate |
| `arrow` / `feather` | `arrow` crate / `polars` IPC |
| `jsonlite` | `serde_json` |
| `httr` / `httr2` | `reqwest` |
| `DBI` + `RPostgres` | `sqlx` / `diesel` / `tokio-postgres` |
| `lm()` / `glm()` | `linregress` / `ndarray-stats::fit_least_squares` / `smartcore::linear_regression` |
| `glmnet` (LASSO/Ridge) | `smartcore::linear::lasso` / `ndarray-linalg` |
| `randomForest` / `ranger` | `smartcore::ensemble::random_forest` |
| `xgboost` | `xgboost-rs` / `lightgbm` (FFI) |
| `caret` / `tidymodels` | `smartcore` (unified ML API) |
| `cluster` / kmeans | `smartcore::cluster::kmeans` / `linfa-clustering` |
| `broom` (tidy model output) | Custom `Display` / `Serialize` impls |
| `knitr` | mdBook + pre-processing script |
| `rmarkdown` | mdBook + evcxr code blocks |
| `shiny` | Leptos + Axum / Dioxus Fullstack |
| `plumber` (REST API) | `axum` / `actix-web` |
| `testthat::expect_equal()` | `assert_eq!()` |
| `testthat::expect_equivalent()` | `assert!((a - b).abs() < 1e-10)` / `approx::assert_abs_diff_eq!` |
| `microbenchmark` | `criterion` / `std::time::Instant` / `divan` |
| `profvis` | `perf` / `flamegraph` / `tokio-console` |

### Base R Builtins Mapping

| Base R Function | Rust Equivalent | Notes |
|---|---|---|
| `c(1, 2, 3)` | `vec![1, 2, 3]` / `ndarray::arr1(&[1., 2., 3.])` | Vector creation |
| `seq(1, 10, by=2)` | `(1..=10).step_by(2).collect::<Vec<_>>()` | Sequence generation |
| `rep(x, times=n)` | `vec![x; n]` or `std::iter::repeat(x).take(n).collect()` | Repeat values |
| `length(x)` | `v.len()` | O(1) for Vec |
| `dim(x)` | `a.shape()` (ndarray) | Array dimensions |
| `nrow(df)` / `ncol(df)` | `df.shape()` (polars: `.height()`, `.width()`) | Dimensions |
| `names(df)` | `df.get_column_names()` (polars) | Column names |
| `head(df, n)` / `tail(df, n)` | `df.head(Some(n))` / `df.tail(Some(n))` (polars) | First/last rows |
| `summary(df)` | `df.describe(None, None)` (polars) | Descriptive statistics |
| `str(df)` | `println!("{df:?}")` / `df.schema()` | Structure inspection |
| `class(x)` | `std::any::type_name::<T>()` (debug) | Type identity |
| `typeof(x)` | Compile-time: types known statically | Not needed; types are concrete |
| `is.na(x)` / `is.null(x)` | `x.is_none()` (Option) / `x.map_or(true, |v| v.is_nan())` | Missing value checks |
| `na.omit(df)` | `df.drop_nulls(None)` (polars) | Remove rows with nulls |
| `which(x == val)` | `v.iter().positions(|&e| e == val).collect()` | Index of matching elements |
| `match(x, table)` | `table.iter().position(|e| e == &x)` | Position lookup |
| `%in%` operator | `table.contains(&value)` | Membership test |
| `ifelse(test, yes, no)` | `if condition { yes } else { no }` (scalar) | Conditional; vectorized via `azip!` |
| `paste(a, b, sep="_")` | `format!("{a}_{b}")` | String concatenation |
| `grep(pattern, x)` | `x.iter().filter(|s| re.is_match(s)).collect()` | Pattern matching |
| `gsub(pattern, repl, x)` | `re.replace_all(&s, repl)` (regex crate) | String replacement |
| `substr(s, start, stop)` | `&s[start-1..stop]` (byte boundaries!) or `s.chars().skip().take().collect()` | Substring |
| `nchar(s)` | `s.chars().count()` (chars) / `s.len()` (bytes) | String length |
| `toupper(s)` / `tolower(s)` | `s.to_uppercase()` / `s.to_lowercase()` | Case conversion |
| `order(x)` / `sort(x)` | `v.sort()` / `v.sort_by()` (in-place) or `sorted()` (itertools) | Sorting |
| `rank(x)` | `arg_sort` + manual rank assignment (ndarray) / `rank()` (polars) | Ranking |
| `unique(x)` | `v.iter().unique().collect()` (itertools) / `df.unique(None, ...)` (polars) | Distinct values |
| `table(x)` | `HashMap::new()` + `.entry().or_insert(0)` counting | Frequency table |
| `cut(x, breaks)` | Manual bucket assignment with `match` / `polars::cut()` | Binning |
| `merge(x, y, by="id")` | `df_a.join(&df_b, ["id"], ["id"], JoinType::Inner)` (polars) | Join data frames |
| `rbind(df1, df2)` | `df1.vstack(&df2)` (polars) | Row bind |
| `cbind(df1, df2)` | `df1.hstack(df2.get_columns())` (polars) | Column bind |
| `aggregate(x ~ group, df, FUN=mean)` | `df.group_by(["group"]).agg(&[col("x").mean()])` (polars) | Split-apply-combine |
| `t.test(x, y)` | `statrs::stats_tests::t_test` (statrs crate) | Statistical tests |
| `lm(y ~ x, data=df)` | `linregress::FormulaRegressionBuilder::new().fit()` | Linear regression |
| `glm(y ~ x, family=binomial)` | `smartcore::linear::logistic_regression` | Generalized linear models |
| `predict(model, newdata)` | Trait `Predict::predict()` (custom or smartcore/linfa) | Model prediction |
| `set.seed(n)` | `let mut rng = StdRng::seed_from_u64(n)` (rand crate) | Set RNG seed |
| `sample(x, size, replace=F)` | `x.choose_multiple(&mut rng, size).cloned().collect()` | Random sampling |
| `rnorm(n, mean, sd)` | `Normal::new(mean, sd).unwrap().sample(&mut rng)` (rand_distr) | Random normal |
| `runif(n, min, max)` | `rng.gen_range(min..max)` (rand) | Random uniform |
| `cat("text\n")` / `message()` | `println!` / `tracing::info!` | Output / logging |
| `options(digits=3)` | `format!("{:.3}", value)` | Number formatting |
| `stop("error msg")` | `Err(anyhow::anyhow!("error msg"))?` / `bail!` | Error with message |
| `warning("msg")` | `tracing::warn!("msg")` | Warnings via tracing |
| `tryCatch(expr, error=handler)` | `match result { Ok(v) => ..., Err(e) => ... }` | Error handling |
| `do.call(fn, args_list)` | Closures or function pointers with unpacked args | Dynamic function call |
| `eval(parse(text=code))` | Not possible — no runtime code parsing | Use closures/traits instead |
| `formula(y ~ x1 + x2)` | Builder pattern / declarative macro | Formula parsing not built-in |

### dplyr Pipeline Translation

```rust
use polars::prelude::*;

// R with dplyr:
// result <- df |>
//   filter(value > 10) |>
//   group_by(category) |>
//   summarise(mean_val = mean(value), n = n()) |>
//   arrange(desc(mean_val))

// Rust with polars:
fn dplyr_pipeline(df: &DataFrame) -> PolarsResult<DataFrame> {
    df.clone()
        .lazy()
        .filter(col("value").gt(10.0))
        .group_by(&[col("category")])
        .agg(&[
            col("value").mean().alias("mean_val"),
            col("value").count().alias("n"),
        ])
        .sort(["mean_val"], SortMultipleOptions::default().with_order_descending(true))
        .collect()
}
```

## Canonical Patterns

### Pattern 1: Vectorized Function

```rust
// R:
// z_score <- function(x) { (x - mean(x, na.rm = TRUE)) / sd(x, na.rm = TRUE) }

// Rust — 单值:
fn z_score_single(x: f64, mean: f64, sd: f64) -> f64 {
    (x - mean) / sd
}

// Rust — 向量化（使用 ndarray）:
use ndarray::Array1;
use ndarray_stats::QuantileExt;

fn z_score(x: &Array1<f64>) -> Array1<f64> {
    // 忽略 NaN 计算均值和标准差
    let valid: Vec<f64> = x.iter().filter(|v| !v.is_nan()).copied().collect();
    let valid_arr = Array1::from_vec(valid);
    let mean = valid_arr.mean().unwrap_or(0.0);
    let sd = valid_arr.std(0.0);  // ddof = 0, 总体标准差
    if sd == 0.0 {
        return x.mapv(|_| 0.0);
    }
    x.mapv(|v| if v.is_nan() { f64::NAN } else { (v - mean) / sd })
}
```

### Pattern 2: Split-Apply-Combine (lapply / tapply)

```rust
// R:
// result <- tapply(df$value, df$group, mean)

// Rust — 使用 Iterator + HashMap:
use std::collections::HashMap;

fn tapply_mean(values: &[f64], groups: &[&str]) -> HashMap<String, f64> {
    let mut acc: HashMap<String, (f64, usize)> = HashMap::new();
    for (&v, &g) in values.iter().zip(groups.iter()) {
        let entry = acc.entry(g.to_string()).or_insert((0.0, 0));
        entry.0 += v;
        entry.1 += 1;
    }
    acc.into_iter()
        .map(|(k, (sum, count))| (k, sum / count as f64))
        .collect()
}

// 推荐：使用 polars:
// df.group_by(&["group"]).agg(&[col("value").mean()])
```

### Pattern 3: S3 Method Dispatch to Trait

```rust
// R S3:
// predict <- function(obj, ...) UseMethod("predict")
// predict.lm <- function(obj, newdata) { ... }
// predict.glm <- function(obj, newdata, type = "response") { ... }

// Rust — trait-based:
trait Predict {
    fn predict(&self, newdata: &Array2<f64>) -> Vec<f64>;
}

struct LinearModel { coefficients: Array1<f64> }

impl Predict for LinearModel {
    fn predict(&self, newdata: &Array2<f64>) -> Vec<f64> {
        newdata.dot(&self.coefficients).to_vec()
    }
}

struct LogisticModel { coefficients: Array1<f64> }

impl Predict for LogisticModel {
    fn predict(&self, newdata: &Array2<f64>) -> Vec<f64> {
        newdata.dot(&self.coefficients)
            .mapv(|x| 1.0 / (1.0 + (-x).exp()))
            .to_vec()
    }
}

// 统一调用:
fn compute_predictions<M: Predict>(models: &[M], data: &Array2<f64>) -> Vec<Vec<f64>> {
    models.iter().map(|m| m.predict(data)).collect()
}
```

### Pattern 4: Formula to Builder Pattern

```rust
// R:
// model <- lm(y ~ x1 + x2 + x1:x2, data = df)

// Rust — builder pattern:
use linregress::{FormulaRegressionBuilder, RegressionDataBuilder};

fn lm_formula() -> Result<(), Box<dyn std::error::Error>> {
    let data = vec![
        ("y", vec![1.0, 2.0, 3.0, 4.0]),
        ("x1", vec![1.0, 2.0, 3.0, 4.0]),
        ("x2", vec![2.0, 3.0, 4.0, 5.0]),
    ];
    let model = FormulaRegressionBuilder::new()
        .data(&data)
        .formula("y ~ x1 + x2")
        .fit()?;
    Ok(())
}
```

### Pattern 5: apply Family to Iterator Adapters

```rust
// R:
// apply(mat, 1, sum)    # 行求和
// apply(mat, 2, mean)   # 列均值
// lapply(list, summary) # 列表映射
// sapply(vec, sqrt)     # 向量化

// Rust:
use ndarray::Array2;

fn apply_patterns() {
    let mat = Array2::from_shape_vec((3, 4), (1..=12).map(|v| v as f64).collect()).unwrap();

    // 行操作 (axis=0) — R apply(mat, 1, sum):
    let row_sums: Vec<f64> = mat.axis_iter(ndarray::Axis(0))
        .map(|row| row.sum())
        .collect();

    // 列操作 (axis=1) — R apply(mat, 2, mean):
    let col_means: Vec<f64> = mat.axis_iter(ndarray::Axis(1))
        .map(|col| col.mean().unwrap_or(0.0))
        .collect();

    // lapply 风格:
    let items = vec![1.0, 2.0, 3.0, 4.0];
    let squared: Vec<f64> = items.iter().map(|x| x * x).collect();

    // 并行版本 — mclapply:
    use rayon::prelude::*;
    let squared_par: Vec<f64> = items.par_iter().map(|x| x * x).collect();
}
```

## FFI & Incremental Migration

R and Rust can interoperate via the `extendr` crate, allowing incremental migration of R code to Rust.

| Strategy | Tool | When to Use |
|---|---|---|
| Call Rust from R | `extendr` crate | Replace slow R functions one at a time |
| Call R from Rust | Command line `Rscript` | Quick prototyping; not for prod |
| Rust as Rcpp replacement | `extendr` — `#[extendr]` proc macros | Drop-in replacement for Rcpp packages |
| Data exchange via Arrow | `arrow` crate in both R and Rust | Zero-copy data sharing between R and Rust |
| Standalone binary | Pure Rust CLI | Batch processing pipelines |
| IPC via REST/gRPC | axum/tonic | Micro-service split for heavy computation |

### extendr Example: Rust Function Callable from R

```rust
// Rust — 定义为 R 可调用的函数:
use extendr_api::prelude::*;

/// 计算向量化 z-score（替代 R 的 scale()）
/// @param x 数值向量
/// @export
#[extendr]
fn fast_zscore(x: &[f64]) -> Vec<f64> {
    let n = x.len();
    if n == 0 { return vec![]; }
    let mean: f64 = x.iter().sum::<f64>() / n as f64;
    let variance: f64 = x.iter().map(|v| (v - mean).powi(2)).sum::<f64>() / n as f64;
    let sd = variance.sqrt();
    if sd == 0.0 { return vec![0.0; n]; }
    x.iter().map(|v| (v - mean) / sd).collect()
}

// 注册函数供 R 调用:
#[extendr]
fn hello_from_rust() -> &'static str {
    "Hello from Rust!"
}

// 模块入口:
extendr_module! {
    mod myrustpkg;
    fn fast_zscore;
    fn hello_from_rust;
}
```

After building with `rextendr::document()`, the R code is simply:
```r
# R
library(myrustpkg)
result <- fast_zscore(c(1, 2, 3, 4, 5))
```

### Migration Order for R Projects

1. Profile the R code with `profvis`. Identify hot CPU-bound functions.
2. Rewrite the slowest pure-R functions (no external R package calls) as extendr functions.
3. Replace R's data.frame operations with polars via extendr — pass Arrow data, not R lists.
4. Move model fitting loops to Rust using smartcore or custom ndarray code.
5. Extract entire analysis pipelines into standalone Rust binaries, called from R via `system()` or HTTP API.
6. Only the final visualization/plotting layer stays in R (ggplot2 has no perfect Rust equivalent yet).

## Common Mistakes

### Mistake 1: Forgetting NA Handling

```rust
// 错误 — 忽略 NA 值导致静默错误:
fn mean_naive(values: &[f64]) -> f64 {
    values.iter().sum::<f64>() / values.len() as f64
    // 如果 values 包含 NaN，结果也是 NaN — 但 R 的 mean(na.rm=T) 会跳过
}

// 正确 — 处理缺失值:
fn mean_robust(values: &[Option<f64>]) -> Option<f64> {
    let (sum, count) = values.iter()
        .filter_map(|&v| v)
        .fold((0.0f64, 0usize), |(s, c), v| (s + v, c + 1));
    if count == 0 { None } else { Some(sum / count as f64) }
}
```

### Mistake 2: Using Vec<Vec<f64>> Instead of ndarray for Matrices

```rust
// 错误 — 双层 Vec 效率极低（指针追逐，缓存不友好）:
let matrix: Vec<Vec<f64>> = vec![vec![1.0; 1000]; 1000];
let sum: f64 = matrix.iter().flatten().sum();

// 正确 — 列优先连续内存:
let matrix = Array2::<f64>::from_elem((1000, 1000), 1.0);
let sum: f64 = matrix.sum();
// ndarray 自动 SIMD 向量化，且内存连续
```

### Mistake 3: 1-Indexing in Algorithm Ports

```rust
// 错误 — R 代码中的 1:5 直接翻译:
// R: for (i in 1:n) { x[i] <- x[i] * 2 }
// Rust 错误翻译:
for i in 1..=n {  // 越界！Rust 数组从 0 开始
    x[i] *= 2.0;
}

// 正确:
for i in 0..n {
    x[i] *= 2.0;
}
// 或使用迭代器（更符合 Rust 习惯）:
for val in x.iter_mut() {
    *val *= 2.0;
}
```

### Mistake 4: Implicit Recycling in Vector Operations

```rust
// R 自动长度回收:
// c(1, 2, 3, 4) + c(1, 2)  =>  c(2, 4, 4, 6)

// Rust — 必须显式处理或报错:
fn add_with_recycle(a: &[f64], b: &[f64]) -> Result<Vec<f64>, &'static str> {
    if a.len() % b.len() != 0 {
        return Err("长度不是倍数关系，无法回收");
    }
    Ok(a.iter()
        .enumerate()
        .map(|(i, &av)| av + b[i % b.len()])
        .collect())
}
```

### Mistake 5: Using Mutable Global State Like R's .GlobalEnv

```rust
// 错误 — R 风格的全局变量:
static mut DATA: Vec<DataFrame> = Vec::new();

// 正确 — 参数化传递:
struct AnalysisContext {
    data: Vec<DataFrame>,
    parameters: Config,
}

fn run_pipeline(ctx: &AnalysisContext) -> Result<Report, Error> {
    // 所有数据通过 ctx 传入
    // ...
}
```

## Reference Implementations

| Project | Description | Migration Pattern |
|---|---|---|
| [extendr](https://github.com/extendr/extendr) | Rust extension framework for R | R-to-Rust FFI; drop-in Rcpp replacement |
| [polars](https://github.com/pola-rs/polars) | DataFrame library in Rust | Full data.frame replacement; faster than data.table |
| [prql-compiler](https://github.com/PRQL/prql) | Pipeline query language compiled to SQL | dplyr-to-SQL pipeline analogue written in Rust |
| [savvy](https://github.com/yutannihilation/savvy) | Simple R-FFI for Rust | Lighter alternative to extendr |
| [smartcore](https://github.com/smartcorelib/smartcore) | ML library in Rust | Replacement for caret/tidymodels |
| [linfa](https://github.com/rust-ml/linfa) | ML toolkit in Rust | Similar scope to tidymodels |
| [evcxr](https://github.com/evcxr/evcxr) | Rust REPL for Jupyter | Interactive exploration, similar to R console |

## Cross-Reference

- **python-to-rust** — For migrating pandas/numpy/scipy-based workflows to Rust (ndarray/polars)
- **julia-to-rust** — For migrating Julia's scientific computing and numerical workflows to Rust
- **go-to-rust** — For migrating API servers that serve R model results
- **nodejs-to-rust** — For migrating Shiny web app backends
- **c-to-rust** — For migrating C/Fortran statistical libraries that R wraps via .C/.Fortran
