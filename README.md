# Optimizing R Shiny Apps

Shiny tips &amp; tricks for improving apps and solving common problems


### 1. Measure Where Your Shiny App Is Spending Its Time

Profvis allows you to measure the execution time and R memory consumption of R code. The package itself can generate a readable report that helps us identify inefficient parts of the code, and it can be used to test Shiny applications. You can see profvis in action here.

If we are only interested in measuring a code fragment rather than a complete application, we may want to consider simpler tools such as the tictoc package, which measures the time elapsed to run a particular code fragment.

### 2. Use Faster Functions

Once you’ve profiled your application, take a hard look at the functions consuming the most time. You may achieve significant performance gains by replacing the functions you routinely use with faster alternatives.

For example, a Shiny app might search a large vector of strings for ones starting with the characters “an”. Most R programmers would use a function such as grepl as shown below:
```r
  grepl("^an", cnames),
```

However, we don’t need the regular expression capabilities of grepl to find strings starting with a fixed pattern. We can tell grepl not to bother with regular expressions by adding the parameter fixed = TRUE. Even better, though, is to use the base R function startsWith. As you can see from the benchmarks below, both options are faster than the original grepl, but the simpler startsWith function performs the search more than 30 times faster.

Similarly, consider the following expressions:
```r
sum_value <- 0
for (i in 1:100) {
  sum_value <- sum_value + i ^ 2
}
```
versus
```r
sum_value <- sum((1:100) ^ 2)
```
Even a novice R programmer would likely use the second version because it takes advantage of the vectorized function sum.

When we create more complex functions for our Shiny apps, we should similarly look for vectorized operations to use instead of loops whenever possible. For example, the following code does a simple computation on two columns in a long data frame:

```r
frame <- data.frame (col1 = runif (10000, 0, 2),
                     col2 = rnorm (10000, 0, 2))

  for (i in 1:nrow(frame)) {
    if (frame[i, 'col1'] + frame[i, 'col2'] > 1) {
      output[i] <- "big"
    } else {
      output[i] <- "small"
    }
  }
```
However, an equivalent output can be obtained much faster by using ifelse which is a vectorized function:

```r
output <- ifelse(frame$col1 + frame$col2 > 1, "big", "small")
```
This vectorized version is easier to read and computes the same result about 100 times faster.


### 3. Pay Attention to Object Scoping Rules in Shiny Apps

* Global: Objects in global.R are loaded into R’s global environment. They persist even after an app stops. This matters in a normal R session, but not when the app is deployed to Shiny Server or Connect. 
* Application-level: Objects defined in app.R outside of the server function are similar to global objects, except that their lifetime is the same as the app; when the app stops, they go away. These objects can be shared across all Shiny sessions served by a single R process and may serve multiple users.
* Session-level: Objects defined within the server function are accessible only to one user session.


In general, the best practice is:

* Create objects that you wish to be shared among all users of the Shiny application in the global or app-level scopes (e.g., loading data that users will share).
* Create objects that you wish to be private to each user as session-level objects (e.g., generating a user avatar or displaying session settings).



