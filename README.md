
ruler: Rule Your Data
=====================

[![Travis-CI Build Status](https://travis-ci.org/echasnovski/ruler.svg?branch=master)](https://travis-ci.org/echasnovski/ruler) [![Coverage Status](https://img.shields.io/codecov/c/github/echasnovski/ruler/master.svg)](https://codecov.io/github/echasnovski/ruler?branch=master) [![packageversion](https://img.shields.io/badge/Package%20version-0.0.0.9000-green.svg?style=flat-square)](commits/master)

`ruler` offers a set of tools for creating tidy data validation reports using [dplyr](http://dplyr.tidyverse.org) grammar of data manipulation. It is designed to be flexible and extendable in terms of creating rules and using their output.

Installation
------------

You can install `ruler` from github with:

``` r
# install.packages("devtools")
devtools::install_github("echasnovski/ruler")
```

Usage
-----

### Check data properties

To check whether dimensions of `mtcars` obey some rules one can write the next dplyr pipeline:

``` r
mtcars %>% summarise(
  nrow_low = nrow(.) > 10, nrow_high = nrow(.) < 30,
  ncol = ncol(.) == 12
)
#>   nrow_low nrow_high  ncol
#> 1     TRUE     FALSE FALSE
```
