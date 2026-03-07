+++
title = 'Things I Hate About MATLAB'
date = '2025-11-07'
tags = ['rant', 'software', 'MATLAB']
topics = ['software']
featured = true
weight = 3
draft = false
+++

# Functions

## Evaluation

In their infinite wisdom, MathWorks has elected to treat a function's _name_ as equivalent to a _function-evaluated-with-no-arguments_. This is a parsing _nightmare_ that makes it impossible to do any sort of meaningful term rewriting, and it results in the following...strange behavior:
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

# Data types

## Typing

* Dynamic typing sucks.
* Weak typing sucks.

## Matrices

### Singleton-as-scalar

MATLAB treats *everything* as a matrix. So `3` is really `3: double(1, 1)`. This makes for some interesting gymnastics when you consider `3 * ones(2)`: this is a `1-by-1` matrix times a `2-by-2` matrix which violates normal matrix multiplication rules. MATLAB clearly enjoys exceptional behaviors and special rules.

## Row and column vectors

There is no such goddamn thing as a row and column vector. There are _vectors_. There are operations defined on _vectors_. Nothing in math requires a vector to have a "transpose"—it has a _dual_, but this is not the same thing.

# Ternary-if

```MATLAB
iif  = @(varargin) varargin{2*find([varargin{1:2:end}], 1, 'first')}();
```
Why does this exist?