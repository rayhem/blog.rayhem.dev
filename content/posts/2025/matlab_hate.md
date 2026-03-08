---
title: 'Things I Hate About MATLAB'
date: '2025-11-07'
lastmod: '2026-03-07'
tags: ['rant', 'software', 'MATLAB']
showtoc: true
---

# Functions

## Organization

One-function-per-file is, quite possibly, the only thing *worse* than header files for making code available.
1. The overhead of making a new function (create a new file, name it, name the function identically, realize the file is in the wrong spot, move it, ...) applies tremendous pressure _against_ making small helper functions.
2. Even if functions are closely related, you still have to open _multiple files_ to engage with them.
4. Touching _every file on the path_ to build a cache is apt to cause slowdowns, especially where latency matters (network filesystems).
    * This also makes for interesting times when you e.g. delete a file. Your choices are: MATLAB periodically rescan everything to refresh the cache (slow), or you trigger a cache update manually (annoying).

## Evaluation

In their infinite wisdom, MathWorks has elected to treat a function's _name_ as equivalent to a _function-evaluated-with-no-arguments_.
This is a parsing _nightmare_ that makes it impossible to do any sort of meaningful term rewriting, and it results in the following...strange behavior:
```matlab
>> cd * cd
Error using  *
Incorrect dimensions for matrix multiplication. Check that the number of columns in the first matrix matches the number of rows in the second matrix. To operate on each element of the
matrix individually, use TIMES (.*) for elementwise multiplication.

>> disp * disp
Error using disp
Too many output arguments.

>> numel * numel
Error using numel
Not enough input arguments.
```
* `cd` evaluates as `cd()` and, as output, produces a character (row) vector of the current working directory. This attempts to multiply (inner product) the same vector from the second `cd` which is also a row, producing the listed error. Also, what even _is_ an inner product of strings?
* `disp` produces no outputs. Multiplying values requires there to _be_ a value, hence when the interpreter tries to store the output of `disp` it can't and thus errors.
* `numel` requires at least one input. When the interpreter tries to evaluate the first `numel` it simply can't and throws this error.

All of this makes sense if you understand the rules of functions and their evaluation order, but, assuredly _none of this is what any reasonable user actually wanted_. Reasonable users want "Your expression sucks because times (\*) does not accept functions." Unfortunately, this will _never be fixed_ because of MATLAB's attitude towards legacy designs.

## Lambdas

MATLAB's anonymous functions are just gimped.
They support neither assignments nor multiple statements (which limits their utility to everything you can do in one statement), and they capture values at _definition_ time, not execution time (which limits their utility even more).

# Data types

## Typing

* Dynamic typing sucks.
* Weak typing sucks.

Enough said.

## Matrices

### Singleton-as-scalar

MATLAB treats *everything* as a matrix. So `3` is really `3: double(1, 1)`. This makes for some interesting gymnastics when you consider `3 * ones(2)`: this is a `1-by-1` matrix times a `2-by-2` matrix which violates normal matrix multiplication rules. MATLAB clearly enjoys exceptional behaviors and special rules.

## Row and column vectors

There is no such goddamn thing as _row_ and _column_ vectors[^1].
There are _vectors_.
There are operations defined on _vectors_.
Nothing in math requires a vector to have a "transpose"—it has a _dual_, but this is not at all the same thing.

# Ternary-if

```MATLAB
iif  = @(varargin) varargin{2*find([varargin{1:2:end}], 1, 'first')}();
```
Why does this exist?

[^1]: How would a "column vector" even work in a non-pointy-arrows vector space?
Fourier series are vectors; do they have rows or columns?
