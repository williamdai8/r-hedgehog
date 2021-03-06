Hedgehog
========

<img src="vignettes/hedgehog-logo.png" width="307" align="right"/>

> Hedgehog will eat all your bugs.

[Hedgehog](http://hedgehog.qa/) is a modern property based testing
system in the spirit of QuickCheck, originally written in Haskell,
but now also available in R. One of the key benefits of Hedgehog is
integrated shrinking of counterexamples, which allows one to quickly
find the cause of bugs, given salient examples when incorrect
behaviour occurs.

Features
========

- Expressive property based testing.
- Integrated shrinking, shrinks obey invariants by construction.
- Generators can be combined to build complex and interesting
  structures.
- Abstract state machine testing.
- Full compatibility with [testthat][testthat] makes it easy to
  add property based testing, without disrupting your work flow.

Example
=======

To get a quick look of how Hedgehog feels, here's an example
showing some of the properties a function which reverses a vector
should have. We'll be testing the `rev` function from
`package:base`.


```r
test_that( "Reverse of reverse is identity",
  forall( gen.c( gen.element(1:100) ), function(xs) expect_equal(rev(rev(xs)), xs))
)
```

The property above tests that if I reverse a vector twice, the
result should be the same as the vector that I began with.
Hedgehog has generated 100 examples, and checked that this
property holds in all of these cases.

As one can see, there is not a big step from using vanilla `testthat`
to including hedgehog in one's process. Inside a `test_that` block,
one can add a `forall` and set expectations within it.

We use the term `forall` (which comes from predicate logic) to say
that we want the property to be true no matter what the input to
the tested function is. The first argument to `forall` is function
to generate random values (the generator); while the second is
the properties we wish to test.

The property above doesn't actually completely specify that the
`rev` function is accurate though, as one could replace `rev` with
the identity function and still observe this result. We will therefore
write one more property to thoroughly test this function.


```r
test_that( "Reversed of concatenation is flipped concatenation of reversed",
  forall( list( as = gen.c( gen.element(1:100) )
              , bs = gen.c( gen.element(1:100) ))
        , function(as,bs) expect_equal ( rev(c(as, bs)), c(rev(bs), rev(as)))
  )
)
```

This is now a well tested reverse function. Notice that the property
function now accepts two arguments: `as` and `bs`. A list of generators
in Hedgehog is treated as a generator of lists, and shrinks all members
independently. We do however do our best to make sure that properties
can be specified naturally if the generator is specified in this manner
as a list of generators.

Now let's look at an assertion which isn't true so we can see what our
counterexamples looks like


```r
test_that( "Reverse is identity",
  forall( gen.c( gen.element(1:100) ), function(xs) expect_equal(rev(xs), c(xs)))
)
```

```
## Error: Test failed: 'Reverse is identity'
## * Falsifiable after 1 tests, and 8 shrinks
## rev(xs) not equal to c(xs).
## 2/2 mismatches (average diff: 1)
## [1] 2 - 1 ==  1
## [2] 1 - 2 == -1
##
## Counterexample:
## [1] 1 2
```

This test says that the reverse of a vector should equal the vector,
which is obviously not true for all vectors. Here, hedgehog has run
this expectation with random input, and found it to not be true.
Instead of reporting it directly, it has shrunk the bad test case to
the smallest counterexample it could find: `c(1,2)`. Hedgehog then
reëmits this test error to `testthat`, which handles it as per usual
and displays it to the user.

Generators
==========

Hedgehog exports some basic generators and plenty of combinators for
making new generators. Here's an example which produces a floating
point value between -10 and 10, shrinking to the median 0.


```r
gen.unif( from = -10, to = 10 )
```

```
## Hedgehog generator:
## A generator is a function which produces random trees
## using a size parameter to scale it.
##
## Example:
## [1] -2.085815
## Shrinks:
## [1] 0
## [1] -0.08581477
## [1] -1.085815
```

Although only three possible shrinks are shown above, these are
actually just the first layer of a rose tree of possible shrinks.
This integrated shrinking property is a key component of hedgehog,
and gives us an excellent chance of reducing to the minimum possible
counterexample.


```r
test_that( "a is less than b + 1",
  forall(list(a = gen.element(1:100), b = gen.element(1:100)), function(a, b) expect_lt( a, b + 1 ))
)
```

```
## Error: Test failed: 'a is less than b + 1'
## * Falsifiable after 2 tests, and 10 shrinks
## 2 is not strictly less than b + 1. Difference: 0
##
## Counterexample:
## $a
## [1] 2
##
## $b
## [1] 1
```

The generators `gen.c`, `gen.element`, and `gen.unif`, are related to
standard R functions: `c`, to create a vector; `sample`, to sample
from a list or vector; and `runif`, to sample from a uniform
distribution. We try to maintain a relationship to R's well known
functions inside Hedgehog.

Generators can also be sequenced together, using the output of one
generator to create a new, more complex one. An example of this is a
list generator, which first randomly chooses a length, then generates
a list of said length.

One way to create larger generators in the `generate` function, which
acts upon an idiomatic `for` loop. For example, one can create a
generator of squares up to 100; and a generator of vectors with lengths
of squares with

```r
gen_squares   <- generate(for (i in gen.int(10)) i^2)
gen_sq_digits <- generate(for (i in gen_squares) {
  gen.c(of = i, gen.element(1:9))
})
```

In the following example, we'll create a generator which builds two
lists of length `n`, then turn them into a `data.frame` with `gen.with`.


```r
gen.df.of <- function(n)
  generate(for (x in
    list( as = gen.c(of = n, gen.element(1:10) )
        , bs = gen.c(of = n, gen.element(10:20) )
        )
    ) as.data.frame(x)
  )

test_that( "Number of rows is 5",
  forall( gen.df.of(5), function(df) expect_equal(nrow(df), 5))
)
```

While this is good, but we would also like to be able to create
`data.frames` with a varying number of rows. Here, we'll again
test a property which is false in order to show how hedgehog
will find the minimum shrink.


```r
gen.df <-
  generate(for (e in gen.element(1:100)) {
    gen.df.of(e)
  })

test_that( "All data frames are of length 1",
  forall( gen.df, function(x) expect_equal(nrow(x), 1))
)
```

```
## Error: Test failed: 'All data frames are of length 1'
## * Falsifiable after 1 tests, and 9 shrinks
## nrow(x) not equal to 1.
## 1/1 mismatches
## [1] 2 - 1 == 1
##
## Counterexample:
##   as bs
## 1  1 10
## 2  1 10
```

Technically, that we can sequence generators is this way implies they
are monads, and we provide a number combinators for manipulating them
in this manner. Indeed, `generate` is simply syntactic sugar for monadic
bind, sometimes referred to as "and then".

The `gen.with` function can be used to apply an arbitrary function to
the output of a generator, while `gen.and_then` is useful in chaining the
results of a generator.

  [testthat]: https://github.com/hadley/testthat
