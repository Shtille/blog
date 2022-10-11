---
layout: post
title: "ESO rotation choice"
author: "Shtille"
categories: journal
tags: [ESO,math]
---

# Making simple math model for ESO rotation

## Introduction

Rotation is sequence of used skills. We are interested in choosing the best skill rotation to achive the maximum damage per second (DPS), and kill target as fast as possible. Thus we take following assumptions:
- we ignore light attack weaving, since it has no influence on rotation
- we ignore the fact if skill crits or not

All math will be common, but class skills will be used for templar.

### Base mechanics

There's a global cooldown on skill usage. So you cant use more than one skill per second. Thus skills with fractional cast time (like 0.8 or 1.8 seconds) will have its actual duration rounded to upper value.
There may be only 10 active skills and 2 ultimate skills (2 bars).
Most of dot skills (or simply dots) have duration above 10 seconds, thus we need one skill with highest damage to fill the gap between dots renewal. This skill is called spammable.

### List of abbreviations

- dps (DpS) - Damage per second
- dot (DoT) - Damage over time

### List of denotations

- $D_s$ - summary damage
- $T_s$ - summary duration
- $R_s=D_s/T_s$ - summary damage per second (dps)
- $\tau$ - duration of global cooldown (GCD), equal to 1 second

## Skills type overview

### Types of skills

1. Instant damage skill.
2. Damage over time skill.
	* Damage tick every X second.
	* Damage is over duration.
	* Combined.
3. Skill with cast time.
	* Cast time is less than second.
	* Cast time is greater than second.

### Types of skills amplification

1. No amplification.
2. Damage increased over time with arithmetic progression.
3. Damage is increased the more the less health the enemy has.

### Types of dots

1. Standard dot.
2. Dot with increasing tick damage.
3. Dot dealing the more damage the less health enemy has.

## Skill DPS calculation formulas

### Instant skill

$$ D_s = D_{base} $$

$$ T_s = \tau $$

$$ R_s = D_s / T_s = D_s / \tau $$

### Standard dot

$$ D_s = D_{base} + D_{duration} + D_{ticking} $$

where

$$ D_{ticking} = \sum_{i=1}^{N_{tick}} D_{tick} = D_{tick} N_{tick} $$

and $N_{tick}$ denotes number of ticks:

$$ N_{tick} = T_{duration} / T_{tick} $$

$$ T_s = T_{prepare} + T_{duration} $$

$$ R_s = D_s / T_s = \frac{D_{base} + D_{duration} + D_{tick} N_{tick}}{T_{prepare} + T_{duration}} \tag{1}\label{1} $$

### Dot with increasing tick damage

$$ D_s = D_{base} + D_{duration} + D_{ticking} $$

$$ N_{tick} = T_{duration} / T_{tick} $$

Tick damage increases each tick by $D_{add}$ with [arithmetic progression](https://en.wikipedia.org/wiki/Arithmetic_progression):

$$ D_{tick}(i) = D_{tick} + D_{add}(i-1) $$

All ticks damage will be:

$$ D_{ticking} = \sum_{i=1}^{N_{tick}} D_{tick}(i) = \frac{2 D_{tick} + D_{add}(N_{tick}-1)}{2} N_{tick} $$

Mostly $D_{add}$ depends on $D_{tick}$:

$$ D_{add} = D_{tick} K_{tick} $$

$$ T_s = T_{prepare} + T_{duration} $$

$$ R_s = D_s / T_s $$

$$
R_s = \frac{D_{tick} N_{tick} (1 + K \frac{N_{tick}-1}{2})}{T_{prepare} + T_{duration}} \tag{2}\label{2}
$$

So $R_s$ is average DPS.

### Dot dealing the more damage the less health enemy has

Damage increase based on enemy's health starts at some enemy health threshold.

Skill description for this kind of skills would be like:
> Deals up to 500% more damage to enemies below 50% Health.

So lets make these denotations:
- $x$ - health of the enemy
- $x_0$ - initial health of the enemy
- $x_t$ - health of the enemy at threshold, where damage increase starts
- $K$ - amplification koefficient

In quoted example above there will be following values:

$$
K = 5 \\
x_t/x_0 = 0.5
$$

Dps at phase before threshold matches _standard dot_ section and described above $\eqref{1}$. Let's denote it $R_0$.
When enemy health reaches zero, dps has maximum value.

$$
\begin{cases}
R(x_0) = R_0 \\
R(x_t) = R_0 \\
R(0) = R_0 (1+K)
\end{cases}
$$

Dps amplification has linear character, thus we can calculate it function below threshold:

$$ R(x) = a x + b, \quad\text{$\nabla x \in [0,x_t]$} $$

$$
\begin{cases}
a x_t + b = R_0 \\
a 0 + b = R_0 (1+K)
\end{cases}
$$
$$
\begin{cases}
b = R_0 (1+K) \\
a x_t + R_0 + R_0 K = R_0
\end{cases}
$$
$$
\begin{cases}
a = -\frac{R_0 K}{x_t} \\
b = R_0 (1+K)
\end{cases}
$$
$$ R(x) = -\frac{R_0 K}{x_t} x + R_0 (1+K) $$

So

$$
R(x) = 
\begin{cases}
R_0 (1+K-K x/x_t), & \text{$x \in [0,x_t]$} \\
R_0, & \text{$x \in [x_t,x_0]$}
\end{cases}
$$

Average dps $R_s = \overline{R}$ can be calculated as:

$$ R_s = \frac{1}{t_0} \cdot \int_{0}^{t_0}R(t)dt $$

But it can be also calculated as:

$$ R_s = \frac{1}{x_0} \cdot \int_{0}^{x_0}R(x)dx $$

Divide it into two intervals:

$$
R_s x_0 = \int_{0}^{x_t}R(x)dx + \int_{x_t}^{x_0}R(x)dx
$$

$$
R_s x_0 = \int_{0}^{x_t}R_0 (1+K-K x/x_t)dx + \int_{x_t}^{x_0}R_0 dx
$$

$$
R_s x_0 = \left.R_{0}(1+K)x\right\rvert_{0}^{x_t} 
- \left.\frac{R_0 K}{2 x_t}x^2\right\rvert_{0}^{x_t} 
+ \left.R_0 x\right\rvert_{x_t}^{x_0}
$$

$$
R_s x_0 = R_0 (1+K) x_t - \frac{1}{2} R_0 K x_t + R_0 x_0 - R_0 x_t
$$

$$ R_s x_0 = \frac{1}{2} R_0 K x_t + R_0 x_0 $$

$$ R_s = \frac{1}{2} R_0 K \left(\frac{x_t}{x_0}\right) + R_0 $$

$$ R_s = R_0 \left(\frac{1}{2} K \frac{x_t}{x_0} + 1\right) \tag{3}\label{3} $$

## Choosing the spammable skill

Let's compare two skills defined by $(D_1, T_1)$ and $(D_2, T_2)$.

The minimal period contaning integer count of $T1$ and $T2$ is:

$$ T = T_1 N_1 = T_2 N_2 = LCM(T_1, T_2) $$

Thus:

$$
D_{s1} = D_1 N_1 = D_1 T/T_1 \\
D_{s2} = D_2 N_2 = D_2 T/T_2
$$

$$
\Delta D_s = D_{s2} - D_{s1} = D_2 T/T_2 - D_1 T/T_1
$$

$$
\Delta R_s = \left(\frac{D_2}{T_2} - \frac{D_1}{T_1}\right) \tag{4}\label{4}
$$

So:
- if $\Delta R_s > 0$ then $D_{s1} < D_{s2}$
- if $\Delta R_s < 0$ then $D_{s1} > D_{s2}$
- if $\Delta R_s = 0$ then $D_{s1} = D_{s2}$

Thus in set of two skills: if $D_{s1} > D_{s2}$ then the first skill is spammable, otherwise the second one.

To find the best spammable skill we should sort array of skills by condition above.

Mostly spammable skills will have 1 second duration.

## Determine if dot worthy usage

Let's assume we have system of two skills $(D_1, T_1)$, $(D_2, T_2)$. So denote:

- $C_1$ - cast time of the first skill
- $C_2$ - cast time of the second skill

Because of the global cooldown instant skills actually have 1 second cast time.

Dot skill is worthwhile if and only it does more damage than spammable during dot's cast time.

Spammable skill has lesser cooldown. So:
- if $T_1 < T_2$ the first skill is spammable, the second is dot;
- if $T_1 \ge T_2$ the first skill is dot, the second is spammable.

$$
\begin{cases}
R_1 C_2 < D_2, & \text{if $T_1 < T_2$} \\
R_2 C_1 < D_1, & \text{if $T_1 \ge T_2$}
\end{cases}
$$

$$
\begin{cases}
\frac{D_1}{T_1} C_2 < D_2, & \text{if $T_1 < T_2$} \\
\frac{D_2}{T_2} C_1 < D_1, & \text{if $T_1 \ge T_2$}
\end{cases}
$$

$$
\begin{cases}
D_1 C_2 < D_2 T_1, & \text{if $T_1 < T_2$} \\
D_2 C_1 < D_1 T_2, & \text{if $T_1 \ge T_2$} \tag{5}\label{5}
\end{cases}
$$

Let's call equation $\eqref{5}$ above _the expediency condition_.

## Dots comparison

### Two skills system damage calculation

Let's assume we have system (rotation) of two skills:
- spammable (the first) - $(D_1, T_1)$
- dot (the second) - $(D_2, T_2)$

The system period will be $T = LCM(T_1,T_2)$.

The system's damage will be:
$$
D_s = D_1 N_1 + D_2 N_2
$$

$$
\begin{cases}
N_2 T_2 = T \\
C_2 + N_1 T_1 = T
\end{cases}
$$

$$
\begin{cases}
N_2 = T/T_2 \\
N_1 = (T - C_2)/T_1
\end{cases}
$$

Then finally:

$$
D_s = D_1 \frac{T - C_2}{T_1} + D_2 \frac{T}{T_2} \tag{6}\label{6}
$$

### Two systems damage comparison

Let's assume that we have two following systems:
- spammable and the first dot
- spammable and the second dot

Let's denote:
- $(D_1, T_1)$ - spammable
- $(D_2, T_2)$ - the first dot
- $(D_3, T_3)$ - the second dot

The least common period is $T = LCM(T_2,T_3)$.

$$
\begin{cases}
N_2 T_2 = T \\
N_3 T_3 = T
\end{cases}
$$

According to $\eqref{6}$ the damage will be:

$$
\begin{cases}
D_s^{I} = D_1 \frac{T - N_2 C_2}{T_1} + D_2 \frac{T}{T_2} \\
D_s^{II} = D_1 \frac{T - N_3 C_3}{T_1} + D_3 \frac{T}{T_3}
\end{cases}
$$

$$
\Delta D_s = D_s^{II} - D_s^{I}
$$

- if $D_s^{I} < D_s^{II}$ then $\Delta D_s > 0$
- if $D_s^{I} > D_s^{II}$ then $\Delta D_s < 0$
- if $D_s^{I} = D_s^{II}$ then $\Delta D_s = 0$

$$
\Delta D_s = \left(D_1 \frac{T - N_3 C_3}{T_1} + D_3 \frac{T}{T_3}\right) -
\left(D_1 \frac{T - N_2 C_2}{T_1} + D_2 \frac{T}{T_2}\right)
$$

$$
\Delta D_s = \left(-\frac{D_1 N_3 C_3}{T_1} + D_3 \frac{T}{T_3}\right) -
\left(-\frac{D_1 N_2 C_2}{T_1} + D_2 \frac{T}{T_2}\right)
$$

$$
\Delta D_s = \frac{D_1}{T_1}\left(N_2 C_2 - N_3 C_3\right) +
\left(\frac{D_3}{T_3}T - \frac{D_2}{T_2}T\right)
$$

$$
\Delta D_s = \frac{D_1}{T_1}\left(\frac{T}{T_2} C_2 - \frac{T}{T_3} C_3\right) +
\left(\frac{D_3}{T_3} - \frac{D_2}{T_2}\right) T
$$

Divide $\Delta D_s$ by $T$ to make $\Delta R_s$:

$$
\Delta R_s = \frac{\Delta D_s}{T}
$$

Since $T > 0$:
- if $D_s^{I} < D_s^{II}$ then $\Delta R_s > 0$
- if $D_s^{I} > D_s^{II}$ then $\Delta R_s < 0$
- if $D_s^{I} = D_s^{II}$ then $\Delta R_s = 0$

$$
\Delta R_s = \frac{D_1}{T_1}\left(\frac{C_2}{T_2} - \frac{C_3}{T_3}\right) +
\left(\frac{D_3}{T_3} - \frac{D_2}{T_2}\right) \tag{7}\label{7}
$$

- if $\Delta R_s > 0$ then $D_s^{I} < D_s^{II}$
- if $\Delta R_s < 0$ then $D_s^{I} > D_s^{II}$
- if $\Delta R_s = 0$ then $D_s^{I} = D_s^{II}$

### Conclusion

To determine priority of dots usage we should sort list of dots with rule defined in $\eqref{7}$.

## Execute phase

There are spammable skills that deal the more damage the less health enemy has. Let's define this kind of skills as "execute spammable".

We've already calculated dps increase for this kind of skills: $\eqref{3}$.

### Choosing an execute spammable skill

We should choose the same way we did for ordinary spammable skills in $\eqref{4}$ taking into account dps increase formula in $\eqref{3}$:

$$
\begin{cases}
R_{10} = \frac{D_1}{T_1} \\
R_{20} = \frac{D_2}{T_2}
\end{cases}
$$

$$
\begin{cases}
R_{s1} = R_{10} \left(\frac{1}{2} K_1 \frac{x_{t1}}{x_0} + 1\right) \\
R_{s2} = R_{20} \left(\frac{1}{2} K_2 \frac{x_{t2}}{x_0} + 1\right)
\end{cases}
$$

$$
\Delta R_s = R_{s2} - R_{s1} =
\frac{D_2}{T_2} \left(\frac{1}{2} K_2 \frac{x_{t2}}{x_0} + 1\right) -
\frac{D_1}{T_1} \left(\frac{1}{2} K_1 \frac{x_{t1}}{x_0} + 1\right) \tag{8}\label{8}
$$

So:
- if $\Delta R_s > 0$ then $D_{s1} < D_{s2}$
- if $\Delta R_s < 0$ then $D_{s1} > D_{s2}$
- if $\Delta R_s = 0$ then $D_{s1} = D_{s2}$

To find the best execute spammable skill we should sort array of skills by condition $\eqref{8}$.

### When execute spammable becomes worth using

Assume that the first skill is spammable, the second one is execute spammable.
At some level of enemy's health execute spammable becomes more worthy and fully replaces spammable:

$$ R_1(x) \le R_2(x) $$

where $x$ is enemy's health.

$$
\begin{cases}
R_1(x) = R_1 \\
R_2(x) = R_2 \left(1+K-\frac{K}{x_t}x\right), & \text{$x \in [0,x_t]$}
\end{cases}
$$

$$ R_1 \le R_2(1+K) - R_2 \frac{K}{x_t} x $$

$$ R_2 \frac{K}{x_t} x \le R_2(1+K) - R_1 $$

Let's make some denotations:
- $f_t = \frac{x}{x_t}$
- $f_0 = \frac{x}{x_0}$

Then:

$$
f_t x_t = f_0 x_0 \\
f_t = \frac{x_0}{x_t} f_0
$$

Return to our inequality:

$$ R_2 K f_t \le R_2(1+K) - R_1 $$

$$ f_t \le \frac{R_2(1+K) - R_1}{R_2 K} $$

$$ f_0 \le \frac{R_2(1+K) - R_1}{R_2 K} \cdot \frac{x_t}{x_0} $$

$$ f_0 \le \left(\frac{1+K}{K} - \frac{R_1}{R_2}\cdot\frac{1}{K}\right)\cdot\frac{x_t}{x_0} $$

$$ f_0 \le \left(\frac{1+K}{K} - \frac{D_1}{D_2}\cdot\frac{T_2}{T_1}\cdot\frac{1}{K}\right)\cdot\frac{x_t}{x_0} \tag{9} $$

So $f_0$ is desired fraction of enemy's health.
From enemy's health fraction $f = f_0$ and below ordinary spammable usage is worthless.

### When to get rid of using dots

This condition follows from _the expediency condition_ $\eqref{5}$.

$$
D_1(x) C_2 \ge D_2 T_1
$$

$$
R_1(x) C_2 \ge R_2 T_2
$$

$$
R_1 \left(1+K-\frac{K}{x_t}x\right) C_2 \ge R_2 T_2
$$

$$
1+K-\frac{K}{x_t}x \ge \frac{R_2 T_2}{R_1 C_2}
$$

$$
\frac{K}{x_t}x \le 1+K - \frac{R_2 T_2}{R_1 C_2}
$$

Denote $f = \frac{x}{x_0}$, thus $x = f x_0$.

$$
\frac{K}{x_t} f x_0 \le 1+K - \frac{R_2 T_2}{R_1 C_2}
$$

$$
f \le f_0 = \frac{1}{K} \cdot \frac{x_t}{x_0} \cdot \left(1+K - \frac{D_2 T_1}{D_1 C_2}\right) \tag{10}
$$

So from enemy's health fraction $f = f_0$ and below dot usage is worthless.

## Conclusion

So with given list of skills we:

1. Calculated dps for each kind of skill.
2. Determined spammable skills and found best of it.
3. Compared dots and sorted it in dps descendant order.
4. Determined best execution spammable skill and when to use it.
5. Determined when to get rid of dots usage.