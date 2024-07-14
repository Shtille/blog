---
layout: post
title: "ESO DPS change on one skill adjustment"
author: "Shtille"
categories: journal
tags: [ESO,math]
---

Lets figure out how our total DPS will change when one skill damage output changes.
Check the [ESO rotation choice article]({{ 'eso-rotation-choice' | relative_url }}) to recall the denotations and formulas.

## General case

### List of denotations

- $N$ - number of different sources of damage
- $X$ - amount of parse dummy target health points
- $T_a$ - time required to kill dummy target with the first skill version
- $T_b$ - time required to kill dummy target with the second skill version
- $R_a=X/T_a$ - summary damage per second (dps) with the first skill version
- $R_b=X/T_b$ - summary damage per second (dps) with the second skill version

### Initial scenario math

$X_i$ will be the amount of damage caused by a separate _i_-th source.
$R_i$ will be DPS of a separate _i_-th source.

$$ X=\sum_{i=1}^{N} X_i $$

$$ X_i=R_i T_a $$

$$ X=\sum_{i=1}^{N} R_i T_a = T_a \sum_{i=1}^{N} R_i $$

$$ R_a=\sum_{i=1}^{N} R_i $$

Let's assume that our changing skill has index 1, so:

$$ R_a=R_1 + \sum_{i=2}^{N} R_i $$

Let's denote that summ as $R_s$:

$$ R_s=\sum_{i=2}^{N} R_i $$

$$ R_a=R_1+R_s $$

### Target scenario

DPS of a single skill (we've chosen it as the first skill) changes.
Time required to kill dummy target changes to $T_b$.

$$ X=\sum_{i=1}^{N} Y_i $$

$$ Y_i=R_{i}^{*} T_b $$

$$ X=\sum_{i=1}^{N} R_{i}^{*} T_b = T_b \sum_{i=1}^{N} R_{i}^{*} $$

$$ R_b=\sum_{i=1}^{N} R_{i}^{*} $$

As we set peviously, only first skill DPS changes, so:

$$ R_{i}^{*}=R_i, \quad\text{$\nabla i \in [2,N]$} $$

And taking into account initial scenario:

$$ R_b = R_{1}^{*} + R_s $$

Let's assume that skill damage output increases in $K$ times, so:

$$ \frac{R_{1}^{*}}{R_1}=K $$

So our formula will be:

$$
\begin{cases}
R_a = R_1 + R_s \\
R_b = K R_1 + R_s
\end{cases} \tag{1}\label{1}
$$

As we get our parse log we have contribution of each damage source as fractional part.

$$ \frac{X_i}{X}=\frac{R_i}{R_a}=f_i $$

We need only the first (changing) skill fractional part contribution $f$:

$$ \frac{R_1}{R_a}=f $$

$$ \frac{R_s}{R_a}=1-f $$

$$
\begin{cases}
R_1=f R_a \\
R_s=(1-f) R_a
\end{cases} \tag{2}\label{2}
$$

Push results into the $\eqref{1}$:

$$ R_b = K f R_a + (1-f) R_a $$

$$ \frac{R_b}{R_a}=1+(K-1)f \tag{3}\label{3} $$

There are two cases:

1) Skill damage increases, $K>1$
2) Skill damage decreases, $K<1$

## Our practical research

As we get from PTS (public test server) patch notes, the arcanist skill [Cephaliarch's Flail](https://eso-hub.com/en/skills/arcanist/herald-of-the-tome/cephaliarchs-flail) will lose execute capability after game update 43.

### Skill DPS change calculation

- $D_1$ - skill damage before change
- $D_2$ - skill damage after change

Base skill damage:

$$ D_{base} = D n $$

Skill with execute damage increase:

$$ D_{exec} = D \frac{n}{2} + \frac{3}{2} D \frac{n}{2} = \frac{5}{4} D n $$

As in upcoming update there will be no execute component, DPS change will be

$$ \frac{D_2}{D_1}=\frac{D_{base}}{D_{exec}}=\frac{4}{5} $$

$$ K=\frac{4}{5} \tag{4}\label{4} $$

### Summary DPS change calculation

According to typical arcanist parsers, [Cephaliarch's Flail](https://eso-hub.com/en/skills/arcanist/herald-of-the-tome/cephaliarchs-flail) has *7.6%* of total damage output, so $f$ will be:

$$ f=0.076 \tag{5}\label{5} $$

So let's put equations $\eqref{4}$ and $\eqref{5}$ into $\eqref{3}$:

$$ \frac{R_b}{R_a}=1+(K-1)f=0.9848 $$

With typical arcanist parser DPS of 130k, the DPS after update 43 will be 128k. So DPS loss will be 2k. That's not critical. Arcanist still will rock! :)