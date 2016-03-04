R Execution time benchmarking 101
================

Here's a common case: we do have several syntaxes to perform the same action, and we'd like to use the most efficient one. For small cases, it's usually better to use the most readable/easy one, but for large datasets performance is key. How do you measure and compare efficiently various alternatives?

In the following example, we will investigate different ways to filter a dataframe. First using various direct syntaxes: Since our data will contain NAs, we are going to compare `is.na` and `complete.cases` for NA removal.

-   the most common `dataframe[dataframe$columnName == filterValue, ]`, completed with `is.na` and with `complete.cases`
-   using the common `subset` function
-   the large logical to smaller integers version using which: `dataframe[which(dataframe$columnName == filterValue), ]`
-   using `dplyr`'s `filter` method, both with the pipe syntax and the common syntax (Does the pipe impact performances?)

For these tests, we will need to load a few packages:

-   [dplyr](https://cran.r-project.org/web/packages/dplyr/index.html), because we want to comapre it
-   [testthat](https://cran.r-project.org/web/packages/testthat/index.html) because we will run a few assertions to make sur eall our outputs are equivalent
-   [microbenchmark](https://cran.r-project.org/web/packages/microbenchmark/index.html) is very usefull to run, measure and report execution times
-   [ggplot](https://cran.r-project.org/web/packages/ggplot2/index.html) to plot the various execution times

``` r
library(dplyr)
library(testthat)
library(microbenchmark)
library(ggplot2)
```

To run our tests, we will create a medium dataframe, with 10 000 obserations and 7 variables. One of these will have NA and useful values, the one we are going to try to filter out.

``` r
nObs <- 10000
fieldMapping <- list(availability = "AVAILABILITY",  random1 = "var2")
colAvailability <- rep(NA, nObs) # Fill with NAs
colAvailability[rbinom(nObs, 1, 0.66) == 1] <- "ACTIVE" # 2/3 of the infos are set
testData <- data.frame(AVAILABILITY = colAvailability, var2 = rnorm(nObs), var3 = rnorm(nObs),
                       var4 = rnorm(nObs), var5 = rnorm(nObs), var6 = rnorm(nObs), var7 = rnorm(nObs))
```

For the first round of tests, we will filter directly the property, using the most straightforward and usual syntax. Then we will compare it with th eother syntaxes.

``` r
# Caveat: beware of NA for direct filtering
evalVanillaDirect          <- testData[testData$AVAILABILITY == "ACTIVE" & !is.na(testData$AVAILABILITY), ]
evalVanillaDirectCompCases <- testData[testData$AVAILABILITY == "ACTIVE" & complete.cases(testData$AVAILABILITY), ]
evalSubsetDirect           <- subset(testData, AVAILABILITY == "ACTIVE")
evalVanillaDirectWhich     <- testData[which(testData$AVAILABILITY == "ACTIVE"), ]
evalDplyrDirect            <- testData %>% filter(AVAILABILITY == "ACTIVE")
# Check all these syntaxes are the same
expect_equal(nrow(setdiff(evalVanillaDirect, evalVanillaDirectWhich)), 0)
expect_equal(nrow(setdiff(evalVanillaDirect, evalVanillaDirectCompCases)), 0)
expect_equal(nrow(setdiff(evalVanillaDirect, evalSubsetDirect)), 0)
expect_equal(nrow(setdiff(evalVanillaDirect, evalDplyrDirect)), 0)
```

Ok, but now, what happens when the name of the column is unknown, and defined by a variable? The syntax there became is a little bit more complex, and we might wonder how this could affect performances.

``` r
# Now use a variable for the field name
evalVanilla <- testData[testData[[fieldMapping$availability]] == "ACTIVE" & !is.na(testData[[fieldMapping$availability]]), ]
evalVanillaWhich <- testData[which(testData[[fieldMapping$availability]] == "ACTIVE"), ]
evalSubset <- subset(testData, testData[[fieldMapping$availability]] == "ACTIVE")
evalDplyr <- testData %>% filter_(paste(fieldMapping$availability, "=='ACTIVE'"))


# Check can be performed by microbenchmark's check argument, but we dont' want to test every single iteration
expect_equal(nrow(setdiff(evalVanilla, evalVanillaDirect)), 0)
expect_equal(nrow(setdiff(evalVanillaWhich, evalVanillaDirect)), 0)
expect_equal(nrow(setdiff(evalVanilla, evalSubset)), 0)
expect_equal(nrow(setdiff(evalVanilla, evalDplyr)), 0)
```

OK. By now we have a lot of different syntaxes to get the same result. Take a few seconds to guesstimate the performances.
Which one do you believe is the slowest one?
And which one would be the fastest one, if you had to use it without measuring it?

Now, let's measure and see how each one perform. To do this measurments, we could use the basic `system.time` function, which is handy. But if we want to see how fast is a synatx, but also how consistent it is: over many iterations, is the execution time consistent, and if not, how is it distributed. R provide everything to measure this, but there's a very useful package that already provide all the boilerplate: `microbenchmark`. The idea is that you provide several function calls, identified with names, and a number of iterations. (You could also provide a check method and it will verify that each call does return the same result, but in our case we're confident and don'T want to slow down the process). Microbenchmark then return you an object, you can view it like a data frame (for each iteration, you get the name and execution time), but it's clever than that, it also dump automatically the exuection time quartiles. Let's give it a try.

``` r
# Start the measures
nTestIterations <- 1000
benchDirect <- microbenchmark(
    directVanilla              = testData[testData$AVAILABILITY == "ACTIVE" & !is.na(testData$AVAILABILITY), ],
    directVanillaCompleteCases = testData[testData$AVAILABILITY == "ACTIVE" & complete.cases(testData$AVAILABILITY), ],
    directVanillaWhich         = testData[which(testData$AVAILABILITY == "ACTIVE"), ],
    directSubset               = subset(testData, AVAILABILITY == "ACTIVE"),
    directDplyr                = filter(testData, AVAILABILITY == "ACTIVE"),
    directDplyrPipe            = testData %>% filter(AVAILABILITY == "ACTIVE"),
    times = nTestIterations
)
# Output the quartiles
print(benchDirect, signif = 4)
```

    Unit: milliseconds
                           expr   min    lq     mean median    uq    max neval
                  directVanilla 1.742 1.872 3.359862  2.138 3.214 112.80  1000
     directVanillaCompleteCases 1.791 1.926 3.130762  2.161 3.179  90.56  1000
             directVanillaWhich 1.243 1.335 2.447887  1.496 2.437 232.50  1000
                   directSubset 1.776 1.918 3.309273  2.162 3.205  81.45  1000
                    directDplyr 1.372 1.576 2.830004  1.750 2.667 255.80  1000
                directDplyrPipe 1.479 1.697 2.671342  1.887 2.699  45.35  1000

Does these results match the guess you made previously?

And microbenchmark also provides you nice default plots to see how you execution time is distributed for all the given variations. You have the choice between box plots using the base plotting system, and violin plots leveraging ggplot2 (please take note of the log scale for the time axis):

``` r
boxplot(benchDirect)
```

![](execution-time-benchmarking_files/figure-markdown_github/Plot%20direct%20benchmark%20times-1.png)<!-- -->

``` r
autoplot(benchDirect)
```

![](execution-time-benchmarking_files/figure-markdown_github/Plot%20direct%20benchmark%20times-2.png)<!-- -->

Now, let's perform the same measures with the variable filtering, and see how slower it is from the direct filtering (or maybe it's not, let's see)

``` r
benchVariable <- microbenchmark(
    vanilla                    = testData[testData[[fieldMapping$availability]] == "ACTIVE" & !is.na(testData[[fieldMapping$availability]]), ],
    vanillaWhich               = testData[which(testData[[fieldMapping$availability]] == "ACTIVE"), ],
    subset                     = subset(testData, testData[[fieldMapping$availability]] == "ACTIVE"),
    dplyr                      = filter_(testData, paste(fieldMapping$availability, "=='ACTIVE'")),
    dplyrWithPipe              = testData %>% filter_(paste(fieldMapping$availability, "=='ACTIVE'")),
    times = nTestIterations
)
print(benchVariable, signif = 4)
```

    Unit: milliseconds
              expr   min    lq     mean median    uq   max neval
           vanilla 1.746 1.878 2.739639  2.080 2.784 60.87  1000
      vanillaWhich 1.240 1.337 1.976100  1.458 1.859 60.69  1000
            subset 1.795 1.932 2.670894  2.122 2.893 85.40  1000
             dplyr 1.431 1.671 2.107452  1.813 2.209  7.68  1000
     dplyrWithPipe 1.546 1.778 2.333462  1.966 2.407 52.41  1000

``` r
boxplot(benchVariable)
```

![](execution-time-benchmarking_files/figure-markdown_github/Measure%20variable%20filtering-1.png)<!-- -->

``` r
autoplot(benchVariable)
```

![](execution-time-benchmarking_files/figure-markdown_github/Measure%20variable%20filtering-2.png)<!-- -->

Feel free to submit pull-request to suggest other filtering syntaxes, other faster variants, or other performance evaluation tips!
