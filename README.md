
<!-- README.md is generated from README.Rmd. Please edit that file -->

# bench

[![Travis build
status](https://travis-ci.org/jimhester/bench.svg?branch=master)](https://travis-ci.org/jimhester/bench)
[![AppVeyor build
status](https://ci.appveyor.com/api/projects/status/github/jimhester/bench?branch=master&svg=true)](https://ci.appveyor.com/project/jimhester/bench)
[![Coverage
status](https://codecov.io/gh/jimhester/bench/branch/master/graph/badge.svg)](https://codecov.io/github/jimhester/bench?branch=master)

The goal of bench is to benchmark code, tracking both the execution
time, memory allocated and number and type of garbage collections.

## Installation

You can install the development version from
[GitHub](https://github.com/) with:

``` r
# install.packages("devtools")
devtools::install_github("jimhester/bench")
```

## Features

`bench::mark()` is used to benchmark one or a series of expressions, it
has a number of advantages over [alternatives](#alternatives).

  - Always uses the highest precision APIs available for each operating
    system (often nanoseconds).
  - Tracks memory allocations for each expression.
  - Tracks the number and type of R garbage collections per expression
    iteration.
  - Verifies equality of expression results, to avoid accidentally
    benchmarking inequivalent code.
  - Has two arguments, `setup` and `parameters`, to make running
    benchmarks over different size datasets easier.
  - Uses adaptive stopping by default, running each expression for a set
    amount of time rather than for a specific number of iterations.
  - Expressions are run in batches and summary statistics are calculated
    after filtering for iterations with garbage collections. This allows
    you to isolate the effects of garbage collection on running time
    (for more details see [Neal
    2014](https://radfordneal.wordpress.com/2014/02/02/inaccurate-results-from-microbenchmark/)).

The times and memory usage are returned as custom objects which have
human readable formatting for display (e.g. `104ns`) and comparisons
(e.g. `x$mem_alloc > "10MB"`).

There is also full support for [ggplot2](http://ggplot2.tidyverse.org/)
including custom scales and formatting.

## Usage

Benchmarks can be run with `bench::mark()`, which takes one or more
expressions to benchmark against each other.

``` r
set.seed(42)
dat <- data.frame(x = runif(10000, 1, 1000), y=runif(10000, 1, 1000))
```

`bench::mark()` will throw an error if the results are not equivalent,
so you don’t accidentally benchmark inequivalent code.

``` r
bench::mark(
  dat[dat$x > 500, ],
  dat[which(dat$x > 499), ],
  subset(dat, x > 500))
#> Error: All results must equal the first result:
#>   `dat[dat$x > 500, ]` does not equal `dat[which(dat$x > 499), ]`
```

Results are easy to interpret, with human readable units.

``` r
bench::mark(
  dat[dat$x > 500, ],
  dat[which(dat$x > 500), ],
  subset(dat, x > 500))
#> # A tibble: 3 x 14
#>   expression       min   mean  median    max `itr/sec` mem_alloc  n_gc n_itr total_time result   memory   time  gc     
#>   <chr>         <bch:> <bch:> <bch:t> <bch:>     <dbl> <bch:byt> <dbl> <int>   <bch:tm> <list>   <list>   <lis> <list> 
#> 1 dat[dat$x > …  302µs  394µs   344µs 1.56ms     2535.      416K    34   918      362ms <data.f… <Rprofm… <bch… <tibbl…
#> 2 dat[which(da…  237µs  316µs   269µs 1.28ms     3160.      357K    20  1099      348ms <data.f… <Rprofm… <bch… <tibbl…
#> 3 subset(dat, …  383µs  493µs   433µs 1.86ms     2027.      548K    21   797      393ms <data.f… <Rprofm… <bch… <tibbl…
```

The `bench::mark` argument `setup` allows you to run code before each
expression. The argument `parameters` allows you to define a `list()`
(or `data.frame()`) of parameters to assign before running `setup`. If
`parameters` is a `list()` all combinations of the parameters will be
enumerated by `expand.grid()`. This allows you to easily benchmark a set
of expressions across a wide variety of input sizes, among other things.

``` r
set.seed(42)

create_df <- function(rows, cols) {
  as.data.frame(setNames(
    replicate(cols, runif(rows, 1, 1000), simplify = FALSE),
    rep_len(c("x", letters), cols)))
}

results <- bench::mark(
  setup = dat <- create_df(rows, cols),
  parameters = list(rows = c(10000, 100000), cols = c(10, 100)),
  min_time = .5,
  min_iterations = 100,

  bracket = dat[dat$x > 500, ],
  which = dat[which(dat$x > 500), ],
  subset = subset(dat, x > 500))
#> Running benchmark with:
#>     rows  cols
#> 1  10000    10
#> 2 100000    10
#> 3  10000   100
#> 4 100000   100
```

## Plotting

`ggplot2::autoplot()` can be used to generate an informative default
plot.

``` r
ggplot2::autoplot(results)
```

<img src="man/figures/README-autoplot-1.png" width="100%" />

However you can also easily produce custom plots by unnesting the
results and working with the data directly.

``` r
library(bench)
library(tidyverse)
results %>%
  unnest() %>%
  filter(gc == "none") %>%
  ggplot(aes(x = mem_alloc, y = time, color = expression)) +
    geom_point() +
    geom_smooth(method = "lm", se = F) +
    scale_color_brewer(type = "qual", palette = 3)
```

<img src="man/figures/README-custom-plot-1.png" width="100%" />

## `system_time()`

**bench** also includes `system_time()`, a higher precision alternative
to
[system.time()](https://www.rdocumentation.org/packages/base/versions/3.5.0/topics/system.time).

``` r
bench::system_time({ i <- 1; while(i < 1e7) i <- i + 1 })
#> process    real 
#>   329ms   330ms
bench::system_time(Sys.sleep(.5))
#> process    real 
#>   117µs   505ms
```

## Alternatives

  - [rbenchmark](https://cran.r-project.org/package=rbenchmark)
  - [microbenchmark](https://cran.r-project.org/package=microbenchmark)
  - [tictoc](https://cran.r-project.org/package=tictoc)
  - [system.time()](https://www.rdocumentation.org/packages/base/versions/3.5.0/topics/system.time)
