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

### 4. Use Caching Operations

If you’ve used all of the previous tips and your application still runs slowly, it’s worth considering implementing caching operations. In 2018, RStudio introduced the ability to cache charts in the Shiny package. However, if you want to speed up repeated operations other than generating graphs, it is worth using a custom caching solution.

One of my favorite packages that I use for this case is memoise. Memoise saves the results of new invocations of functions while reusing the answers from previous invocations of those functions.

The memoise package currently offers 3 methods for storing cached objects:

1. cache_mem - storing cache in RAM (default)
2. cache_filesystem(path) - storing cache on the local disk
3. cache_s3(s3_bucket) - storage in the AWS S3 file database

The selected caching type is defined by the cache parameter in the memoise function.

If our Shiny application is served by a single R process and its RAM consumption is low, the simplest method is to use the first option, cache_mem, where the target function is defined and its answers cached in the global environment in RAM. All users will then use shared cache results, and the actions of one user will speed up the calculations of others. You can see a simple example below:

```r
library(memoise)

# Define an example expensive to calculate function
expensive_function <- function(x) {
    sum((1:x) ^ 2)
    Sys.sleep(5)    # make it seem to take even longer
  }

system.time(expensive_function(1000)) # Takes at least 5 seconds
    user  system elapsed 
  0.013   0.016   5.002 
system.time(expensive_function(1000)) # Still takes at least 5 seconds
   user  system elapsed 
  0.016   0.015   5.005 

# Now, let's cache results using memoise and its default cache_memory

memoised_expensive_function <- memoise(expensive_function)
system.time(memoised_expensive_function(1000)) # Takes at least 5 seconds
   user  system elapsed 
  0.016   0.015   5.001 
system.time(memoised_expensive_function(1000)) # Returns much faster
   user  system elapsed 
  0.015   0.000   0.015 
````

The danger associated with using in-memory caching, however, is that if you don’t manage the cached results, it will grow without bound and your Shiny application will eventually run out of memory. You can manage the cached results using the timeout and forget functions.

If the application is served by many processes running on one server, the best option to ensure cache sharing among all users is to use cache_filesystem and store objects locally on the disk. Again, you will want to manage the cache, but you will be limited only by your available disk space.

In the case of an extensive infrastructure using many servers, the easiest method will be to use cache_s3 which will store its cached values on a shared external file system – in this case, AWS S3.


