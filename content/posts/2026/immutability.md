---
title: 'Immutability: Proof-by-Set-Theory'
date: '2026-03-04'
author: 'Connor Glosser'
tags: ['software', 'immutability', 'math', 'set theory']
showtoc: true
math: true
---

# Definitions

Let a _software component_ denote any function or module with a defined input domain.
For a component, $f$, define its **effective state space**, $S_f$, as

$$
\begin{equation}
  S_f = \mathrm{Input}_f \times \prod_{i \in \mathrm{deps}(f)} D_i.
\end{equation}
$$

Here, $\mathrm{deps}(f)$ denotes the set of all mutable bindings $f$ can observe or mutate, and $D_i$ their value domains.
For a [referentially transparent](https://en.wikipedia.org/wiki/Referential_transparency) component, $g$, $\mathrm{deps}(g) = \varnothing$ definitionally, so it follows that

$$
\begin{equation}
  S_g = \mathrm{Input}_g.
\end{equation}
$$

Next, let $\mathbf{P}$ denote the set of all software components, and $\mathbf{Q} \subset \mathbf{P}$ the subset of referentially transparent components.
The containment follows dierctly: every referentially transparent component qualifies as a component, but not the converse (i.e. not every component qualifies as referentially transparent).

Finally, let $B_\mathbf{X}$ denote the set of bugs across a collection of components, $\mathbf{X}$, and partition

$$
\begin{equation}
  B_\mathbf{P} = B_\mathbf{P}^\mathrm{(mut)} \cup B_\mathbf{P}^\mathrm{(other)}
\end{equation}
$$

where $B_\mathbf{P}^\mathrm{(mut)}$ contains all bugs manifesting from the observation or mutation of shared mutable state --- data races, aliasing errors, temporal coupling failures, order-of-update effects, etc.
By construction, $B_\mathbf{Q} \subseteq B_\mathbf{P}^\mathrm{(other)}$.

# Theorem 1: Categorial Bug Elimination

## Claim

$|B_\mathbf{Q}| < |B_\mathbf{P}|$ regardless of component count.

## Argument

$B_\mathbf{P}^\mathrm{(mut)} \neq \varnothing$ empirically --- the existence of data races, aliasing bugs, and temporal coupling defects in real systems is well-documented.
Since $B_\mathbf{Q} \subseteq B_\mathbf{P}^\mathrm{(other)}$, $B_\mathbf{P}^\mathrm{(other)} = B_\mathbf{P} / B_\mathbf{P}^\mathrm{(mut)}$, and $B_\mathbf{P}^\mathrm{(mut)} \neq \varnothing $ (i.e. $|B_\mathbf{P}^\mathrm{(mut)}| > 0$), it follows that

$$
\begin{equation}
  \left|B_\mathbf{Q}\right| \leq \left|B_\mathbf{P}\right| - \left|B_\mathbf{P}^\mathrm{(mut)}\right| \lt \left| B_\mathbf{P}\right|
\end{equation}
$$

This requires no assumption about bug distribution; it holds categorically.
In other words, by eliminating mutability (moving from $P$ to $Q$) we strictly eliminate bugs arising from mutability.

# Theorem 2: Proportional Reduction in Expected Bug Density

## Assumption

In the absence of knowledge about how bugs distribute over state space, assume the bug-probability-per-component scales proportionally with the component's effective state space volume:

$$
\begin{equation}
  P(\mathrm{bug} | f) \propto \left|S_f\right|
\end{equation}
$$

This follows from the principle of insufficient reason --- it assigns no _a priori_ preference to any region of state space as "more correct" or "more bug-free" which constitutes the least informitive defensible prior.

## Claim

Even holding component counts equal (i.e. if $|\mathbf{Q}| = |\mathbf{P}|$), expected bug _density_ satisfies

$$
\begin{equation}
  \frac{E[\text{bugs in Q}]}{|Q|} < \frac{E[\text{bugs in P}]}{|P|}.
\end{equation}
$$

## Argument

For any $f \in \mathbf{P}$ with a referentially transparent equivalent $g \in \mathbf{Q}$:

$$
\begin{align}
|S_g| &= \left|\mathrm{Input}_g\right| \\
|S_f| &= \left|\mathrm{Input}_f\right| \times \prod_{i \in \mathrm{deps}(f)} \left|D_i\right| 
\end{align}
$$

from which it follows $|S_g| < |S_f|$ whenever $\mathrm{deps}(f) \neq \varnothing$. The expected bug count across $\mathbf{Q}$ therefore satisfies

$$
\begin{equation}
E[\text{bugs in Q}] \propto \sum_{g \in \mathbf{Q}} \left|S_g\right| < \sum_{f \in \mathbf{P}} \left|S_f\right| \propto E[\text{bugs in P}].
\end{equation}
$$

The ratio of expected bugs equals the ratio of aggregate state space volumes, making the proportional reduction of bugs directly measurable in principle: a component whose mutable dependencies contribute a factor of $k$ to its state space volume yields a k-fold reduction in expected bug probability following referential transparency conversion.

### Note

This argument only provides a _lower_ bound on the actual reduction.
For mutable components that can alias or mutate shared state, pairwise interaction bugs scale as $O(n^2)$ in the number of mutable references, and concurrent interleavings scale factorially.
Referential transparency eliminates this entire interaction class, so we can expect real reductions to exceed the linear state-space ratio argument alone.

# Conclusion

Two independent arguments converge:

1. **Categorically**, referentially transparent software components cannot exhibit mutation-class bugs. Since that class demonstrably contains members, any system $\mathbf{Q}$ with fewer mutable components than $\mathbf{P}$ contains strictly fewer bugs, independent of magnitude.
2. **Quantitatively**, under a maximum-entropy prior, expected bug density scales with effective state space volume. Converting mutable components to referentially transparent equivalents reduces each component's state space by exactly the volume contributed by its mutable dependencies, yielding a proportional reduction in expected bugs even when total component count stays fixed.

