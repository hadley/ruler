
ruler: Rule Your Data
=====================

[![Travis-CI Build Status](https://travis-ci.org/echasnovski/ruler.svg?branch=master)](https://travis-ci.org/echasnovski/ruler) [![Coverage Status](https://codecov.io/gh/echasnovski/ruler/graph/badge.svg)](https://codecov.io/github/echasnovski/ruler?branch=master)

`ruler` offers a set of tools for creating tidy data validation reports using [dplyr](http://dplyr.tidyverse.org) grammar of data manipulation. It is designed to be flexible and extendable in terms of creating rules and using their output.

It is recommended to have the solid knowledge of `dplyr` to fully use this package.

Installation
------------

You can install `ruler` from github with:

``` r
# install.packages("devtools")
devtools::install_github("echasnovski/ruler")
```

Overview
--------

**Rule** is a function which converts data unit of interest (data, group, column, row, cell) to logical value indicating whether this object satisfies certain condition.

**Rule pack** is a function which combines several rules into one functional block. The recommended way of creating rules is by creating packs right away with the use of `dplyr` and [magrittr](http://magrittr.tidyverse.org/)'s pipe operator.

**Exposing** data to rules means applying rules to data, collecting results in common format and attaching it to the data as an `exposure` attribute. In this way actual exposure can be done in multiple steps and also be a part of a general data preparation pipeline.

**Exposure** is a format designed to contain uniform information about validation of different data units. It also saves information about packs application for reproducibilty. Basically exposure is a list with two elements:

1.  **Packs info**: a [tibble](http://tibble.tidyverse.org/) with the following structure:
    -   *name* &lt;chr&gt; : Name of the pack. If not given it will be imputed during exposure.
    -   *type* &lt;chr&gt; : Name of pack type. Indicates which data unit pack checks.
    -   *fun* &lt;list&gt; : List (preferably unnamed) of rule pack functions.
    -   *remove\_obeyers* &lt;lgl&gt; : Whether rows about obeyers (data units that obey certain rule) were removed from report after applying pack.

2.  **Tidy data validation report**: a `tibble` with the following structure:
    -   *pack* &lt;chr&gt; : Name of rule pack from column 'name' in packs info.
    -   *rule* &lt;chr&gt; : Name of the rule defined in rule pack.
    -   *var* &lt;chr&gt; : Name of the variable which validation result is reported. Value '.all' is reserved and interpreted as 'all columns as a whole'. **Note** that *var* doesn't always represent the actual column in data frame: for group packs it represent the created group name.
    -   *id* &lt;int&gt; : Index of the row in tested data frame which validation result is reported. Value 0 is reserved and interpreted as 'all rows as a whole'.
    -   *value* &lt;lgl&gt; : Whether the described data unit obeys the rule.

There are four basic combinations of `var` and `id` values which define five basic data units:

-   `var == '.all'` and `id == 0`: Data as a whole.
-   `var != '.all'` and `id == 0`: Group (`var` shouldn't be an actual column name) or column (`var` should be an actual column name) as a whole.
-   `var == '.all'` and `id != 0`: Row as a whole.
-   `var != '.all'` and `id != 0`: Described cell.

With exposure attached to data one can perform different kinds of actions: exploration, assertion, imputation and so on.

Usage
-----

### Creating packs

#### Data packs

``` r
# List of two rule packs for checking data properties
my_data_packs <- data_packs(
  # data_dims is a pack name
  # Data should have 12 columns and 32 rows
  data_dims = . %>% summarise(
    # ncol and nrow are rule names
    ncol = ncol(.) == 12,
    nrow = nrow(.) == 32
  ),
  
  # Data after subsetting should have number of rows in between 10 and 30
  # Rules are applied separately
  vs_1 = . %>% filter(vs == 1) %>%
    summarise(
      nrow_low = nrow(.) > 10,
      nrow_high = nrow(.) < 30
    )
)
```

#### Group packs

``` r
# List of one nameless rule pack for checking group property
my_group_packs <- group_packs(
  # Name will be imputed during exposure
  . %>% group_by(vs, am) %>%
    # Group should have at least one row with 'cyl' == 6
    summarise(any_cyl_6 = any(cyl == 6)),
  
  # One should supply grouping variables
  .group_vars = c("vs", "am")
)
```

#### Column packs

``` r
# List of one rule pack for checking certain columns' property
my_col_packs <- col_packs(
  sum_bounds = . %>% summarise_at(
    # Check only columns with names starting with 'c'
    vars(starts_with("c")),
    # rules() is a dplyr::funs() with necessary name imputations
    # In column packs it should always be used instead of dplyr::funs()
    # Columns should have sum in between 300 and 400
    rules(sum_low = sum(.) > 300, sum_high = sum(.) < 400)
  )
)
```

#### Row packs

``` r
# List of one rule pack checking certain rows' property
z_score <- function(x) {(x - mean(x)) / sd(x)}

my_row_packs <- row_packs(
  row_mean = . %>% mutate(rowMean = rowMeans(.)) %>%
    # Row's mean should deviate from mean of row means by not more than 1 sd
    transmute(is_common_row_mean = abs(z_score(rowMean)) < 1) %>%
    # Check only rows 10-15
    # Values in 'id' column of report will be based on input data (i.e. 10-15)
    # and not on output data (1-6)
    slice(10:15)
)
```

#### Cell packs

``` r
# List of one cell pack checking certain cells property
is_integerish <- function(x) {all(x == as.integer(x))}

my_cell_packs <- cell_packs(
  my_cell_pack_1 = . %>% transmute_if(
    # Check only integer-like columns
    is_integerish,
    # Value should deviate from its column mean by not more than 1 sd
    rules(is_common = abs(z_score(.)) < 1)
  ) %>%
    # Check only rows 20-30
    slice(20:30)
)
```

### Exposing

By default exposing removes obeyers.

``` r
mtcars %>%
  expose(my_data_packs, my_group_packs) %>%
  get_exposure()
#>   Exposure
#> 
#> Packs info:
#> # A tibble: 3 x 4
#>            name       type              fun remove_obeyers
#>           <chr>      <chr>           <list>          <lgl>
#> 1     data_dims  data_pack  <S3: data_pack>           TRUE
#> 2          vs_1  data_pack  <S3: data_pack>           TRUE
#> 3 group_pack..1 group_pack <S3: group_pack>           TRUE
#> 
#> Tidy data validation report:
#> # A tibble: 3 x 5
#>            pack      rule   var    id value
#>           <chr>     <chr> <chr> <int> <lgl>
#> 1     data_dims      ncol  .all     0 FALSE
#> 2 group_pack..1 any_cyl_6   0.0     0 FALSE
#> 3 group_pack..1 any_cyl_6   1.1     0 FALSE
```

One can leave obeyers by setting `.remove_obeyers` to `FALSE`.

``` r
mtcars %>%
  expose(my_data_packs, my_group_packs, .remove_obeyers = FALSE) %>%
  get_report()
#> Tidy data validation report:
#> # A tibble: 8 x 5
#>            pack      rule   var    id value
#>           <chr>     <chr> <chr> <int> <lgl>
#> 1     data_dims      ncol  .all     0 FALSE
#> 2     data_dims      nrow  .all     0  TRUE
#> 3          vs_1  nrow_low  .all     0  TRUE
#> 4          vs_1 nrow_high  .all     0  TRUE
#> 5 group_pack..1 any_cyl_6   0.0     0 FALSE
#> 6 group_pack..1 any_cyl_6   0.1     0  TRUE
#> 7 group_pack..1 any_cyl_6   1.0     0  TRUE
#> 8 group_pack..1 any_cyl_6   1.1     0 FALSE
```

By default `expose()` guesses the pack type if common function is supplied. This behaviour has some edge cases but is useful for interactive use.

``` r
mtcars %>%
  expose(
    some_data_pack = . %>% summarise(nrow = nrow(.) == 10)
  ) %>%
  get_report()
#> Tidy data validation report:
#> # A tibble: 1 x 5
#>             pack  rule   var    id value
#>            <chr> <chr> <chr> <int> <lgl>
#> 1 some_data_pack  nrow  .all     0 FALSE
```

To write strict and robust code one can set `.guess` to `FALSE`.

``` r
mtcars %>%
  expose(
    some_data_pack = . %>% summarise(nrow = nrow(.) == 10),
    .guess = FALSE
  ) %>%
  get_report()
#> Error in expose_single.default(X[[i]], ...): There is unsupported class of rule pack.
```

### Acting after exposure

General action should be done with `act_after_exposure()`. Is takes two arguments:

-   `.trigger` - a function which takes the data with attached exposure and returns `TRUE` if some action should be made.
-   `.actor` - a function which takes the same argument as `.trigger` and performes some action. **Note** that this function is often created for creating side effects (printing, throwing error etc.) and should invisible return its input (to be able to use `act_after_exposure()` with pipe).

``` r
trigger_one_pack <- function(.tbl) {
  packs_number <- .tbl %>%
    get_packs_info() %>%
    nrow()
  
  packs_number > 1
}

actor_one_pack <- function(.tbl) {
  cat("More than one pack was applied.\n")
  
  invisible(.tbl)
}

mtcars %>%
  expose(my_col_packs, my_row_packs) %>%
  act_after_exposure(
    .trigger = trigger_one_pack,
    .actor = actor_one_pack
  ) %>%
  invisible()
#> More than one pack was applied.
```

`ruler` has function `assert_any_breaker()` which can notify about presence of any breaker in exposure.

``` r
mtcars %>%
  expose(my_col_packs, my_row_packs) %>%
  assert_any_breaker()
#>   Breakers report
#> # A tibble: 3 x 5
#>         pack               rule   var    id value
#>        <chr>              <chr> <chr> <int> <lgl>
#> 1 sum_bounds            sum_low   cyl     0 FALSE
#> 2 sum_bounds            sum_low  carb     0 FALSE
#> 3   row_mean is_common_row_mean  .all    15 FALSE
#> Error: assert_any_breaker: Some breakers found in exposure.
```
