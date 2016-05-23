osmdatar
========

R package for downloading OSM data. Current status (on test data of highways only):

| method                     | Computation time (s) |
|----------------------------|----------------------|
| osmplotr                   | 1.86                 |
| hrbrmstr                   | 1.47                 |
| Rcpp (-&gt;`sp` in `R`)    | 0.25                 |
| Rcpp (-&gt;`sp` in `Rcpp`) | 0.08                 |

------------------------------------------------------------------------

Install
-------

``` r
Sys.setenv ('PKG_CXXFLAGS'='-std=c++11')
setwd ("..")
#devtools::document ("osmdatar")
devtools::load_all ("osmdatar")
setwd ("./osmdatar")
Sys.unsetenv ('PKG_CXXFLAGS')
```

------------------------------------------------------------------------

Speed comparisons
-----------------

The `osmplotr` package uses `XML` to process the API query, and `osmar` to convert the result to `sp` structures. The [overpass](https://github.com/hrbrmstr/overpass/) repo of [hrbrmstr](https://github.com/hrbrmstr) uses `xml2` and thus all cribbed functions are called `_xml2_`. All of my new `Rcpp` functions described below are then appended with `_3`. In all cases, the `process_xml_...` functions convert an xml document to the same SpatialLinesDataFrame of highways.

``` r
library (microbenchmark)
```

------------------------------------------------------------------------

1. Osmar / osmplotr
-------------------

``` r
bbox <- matrix (c (-0.12, 51.51, -0.11, 51.52), nrow=2, ncol=2) 
doc <- get_xml_doc (bbox=bbox)
mb <- microbenchmark ( obj <- process_xml_doc (doc), times=100L)
tt <- formatC (mean (mb$time) / 1e9, format="f", digits=2)
```

``` r
cat ("Mean time to convert with osmar =", tt, "s\n")
```

    ## Mean time to convert with osmar = 1.86 s

------------------------------------------------------------------------

2. hrbrmstr
-----------

``` r
doc2 <- get_xml2_doc (bbox=bbox)
mb2 <- microbenchmark ( obj2 <- process_xml2_doc (doc2), times=100L )
tt2 <- formatC (mean (mb2$time) / 1e9, format="f", digits=2)
```

``` r
cat ("Mean time to convert with hrbrmstr code =", tt2, "\n")
```

    ## Mean time to convert with hrbrmstr code = 1.47

The code of **hrbrmstr** using `dplyr` is (only) around 20% faster than using `osmar`. And then for the C++ version ...

------------------------------------------------------------------------

3. Rcpp routines returning lists
--------------------------------

The function calls *should* compile properly within the `load_all` call, but in case they don't they can be loaded manually here:

``` r
Sys.setenv ('PKG_CXXFLAGS'='-std=c++11')
Rcpp::sourceCpp('src/get_highways.cpp')
Sys.unsetenv ('PKG_CXXFLAGS')
```

(One reason this might be necessary is because `devtools::document` fails to insert the necessary line `useDynLib(osmdatar)` in `NAMESPACE`. Re-inserting this manually should fix the problem.)

This section details `Rcpp` routines which return lists of coordinates for ways. These lists are then converted within `R` to `sp` objects. The tests cover 4 versions, the first of which uses the `Rcpp` function `get_highways`, which returns a list of matrices of 2 columns. The subsequent 3 use `get_highways_with_id`, which returns a list of matrices of 3 columns, the first of which contains the OSM way ID.

The differences between the corresponding `R` functions are:

1.  `process_xml_doc3a` uses a simple loop.
2.  `process_xml_doc3b` uses a simple loop (and so time differences arises only through the third ID column from Rcpp)
3.  `process_xml_doc3c` is same as 2/b, but uses `lapply`
4.  `process_xml_doc3d` uses the `dplyr::groupby` approach of **hrbrmstr**

``` r
txt <- get_xml_doc3 (bbox=bbox)
mb3a <- microbenchmark ( obj3 <- process_xml_doc3a (txt), times=100L )
mb3b <- microbenchmark ( obj3 <- process_xml_doc3b (txt), times=100L )
mb3c <- microbenchmark ( obj3 <- process_xml_doc3c (txt), times=100L )
mb3d <- microbenchmark ( obj3 <- process_xml_doc3d (txt), times=100L )
tt3a <- formatC (mean (mb3a$time) / 1e9, format="f", digits=2)
tt3b <- formatC (mean (mb3b$time) / 1e9, format="f", digits=2)
tt3c <- formatC (mean (mb3c$time) / 1e9, format="f", digits=2)
tt3d <- formatC (mean (mb3d$time) / 1e9, format="f", digits=2)
```

``` r
cat ("Mean times to convert with Rcpp code = (", 
     tt3a, ", ", tt3b, ", ", tt3c, ", ", tt3d, ")\n", sep="")
```

    ## Mean times to convert with Rcpp code = (0.25, 0.25, 0.25, 0.35)

The `dplyr` approach of **hrbrmstr** is thus considerably slower than simply using direct loops, while the addition of an extra ID column to the `Rcpp` output does not slow things down at all. It is possible that the `Rcpp` routine could be sped up a little, but this is unlikely to make any difference compared to the necessity of looping / `dplyr`-ing within `R`. The slowness simply comes from the fact that so-called `Spatial___DataFrame` structures are actually **lists**, yet do not redefine any assignment operators for list elements, and so inevitably require loops.

------------------------------------------------------------------------

Rcpp with internal S4 objects
-----------------------------

Finally, the `sp` S4 objects can be constructed within `Rcpp`. The first version constructs `sp::SpatialLines` objects which are then converted in `R` to `sp::SpatialLinesDataFrame` objects (using `process_xml_doc3e`). The second and final version constructs `sp::SpatialLinesDataFrame` objects entirely within `Rcpp` (using `process_xml_docf`).

``` r
txt <- get_xml_doc3 (bbox=bbox)
mb3e <- microbenchmark ( obj3 <- process_xml_doc3e (txt), times=100L )
mb3f <- microbenchmark ( obj3 <- process_xml_doc3f (txt), times=100L )
tt3e <- formatC (mean (mb3e$time) / 1e9, format="f", digits=2)
tt3f <- formatC (mean (mb3f$time) / 1e9, format="f", digits=2)
```

``` r
cat ("Mean time to convert with Rcpp+sp code = (", tt3e, ", ", tt3f, 
     ")\n", sep="")
```

    ## Mean time to convert with Rcpp+sp code = (0.11, 0.8)
