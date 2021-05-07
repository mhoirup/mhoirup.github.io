---
layout: post
title:  'Survival Analysis of Unemployment Data'
date:   2021-04-25
---

Survival analysis is the statistical analysis of duration data. Durations
can be anything from life until death in the medical sciences, run until
failure in engineering, or in this case, unemployment duration. In this
project we seek to understand the factors affecting the probability of
returning to the workforce.

{% marginfigure 'fig1' 'assets/unemp/spell_hist.png'
'**Figure 1**: Distribution of `spell`. A majority of respondents finished their
unemployment spell prior to 10x2 = 20 week mark, with relatively few
surpassing 15 two-week intervals.' %}

Full code can be found on my github at
[mhoirup/econometric-projects/unemployment](https://github.com/mhoirup/econometric-projects/tree/main/unemployment),
where files are split into `imports.R`, `exploratory.R` and
`analysis.R` to keep some semblance of organisation in the project structure.
We begin the analysis by loading the data into our R workspace along with
the packages we're going to use. The data itself is from [the Ecdat
package](https://cran.r-project.org/web/packages/Ecdat/Ecdat.pdf) and was
used in McCall (1996). The main methodological references are Cameron &
Trivedi (2005) and Wooldridge (2010).

{% marginfigure 'fig2' 'assets/unemp/censored_hist.png' "**Figure
2**: Distributions of the four `censor` variables where I've changed the
`1`s and `0`s into booleans for clarity. Here we note the distribution of
`censor4` which is of interest; 1.255 observations have been censored,
while 2.088 observations correspond to completed spells." %}

{% marginfigure 'fig3' 'assets/unemp/age_hist.png' '**Figure 3**: Distribution
of `age`. Bin width is set at 2 to reduce the number of bars in the graph.
A majority of respondents is under 40 years of age, 2.231 in fact, which
corresponds to 68% of all observations.'  %}

```R
library(dplyr)
library(Ecdat)
library(survival)
library(ggplot2)

data(UnempDur); data <- UnempDur; rm(UnempDur)
tibble::as.tibble(data)
# A tibble: 3,343 x 11
#    spell censor1 censor2 censor3 censor4   age ui    reprate disrate logwage tenure
#    <dbl>   <dbl>   <dbl>   <dbl>   <dbl> <dbl> <fct>   <dbl>   <dbl>   <dbl>  <dbl>
#  1     5       1       0       0       0    41 no      0.179   0.045    6.90      3
#  2    13       1       0       0       0    30 yes     0.52    0.13     5.29      6
#  3    21       1       0       0       0    36 yes     0.204   0.051    6.77      1
#  4     3       1       0       0       0    26 yes     0.448   0.112    5.98      3
#  5     9       0       0       1       0    22 yes     0.32    0.08     6.32      0
#  6    11       0       0       0       1    43 yes     0.187   0.047    6.85      9
#  7     1       0       0       0       0    24 no      0.52    0.13     5.61      1
#  8     3       1       0       0       0    32 no      0.373   0.093    6.16      0
#  9     7       1       0       0       0    35 yes     0.52    0.13     5.29      2
# 10     5       0       0       0       1    31 yes     0.52    0.13     5.29      1
# … with 3,333 more rows 
```

Below is a table with a brief description of each variable. In particular,
note how `spell` give the duration of unemployment in two-week intervals,
for example, if `spell == 2`, a respondent has been unemployed for four
weeks. Except for `censor4`, the `censor` variables will not be needed in
this analysis, as we're only interested in whether or not respondents had
entered the workforce at the end of the survey.

<table class='norulers'>
<tr><td><code>spell</code></td><td>Unemployment duration measured in two-week intervals.</td></tr>
<tr><td><code>censor1</code></td><td><code>1</code> if reemployed at a full-time position after ended spell.</td></tr>
<tr><td><code>censor2</code></td><td><code>1</code> if reemployed at a part-time position after ended spell.</td></tr>
<tr><td><code>censor3</code></td><td><code>1</code> if reemployed but later resigned from that reemployment..</td></tr>
<tr><td><code>censor4</code></td><td><code>1</code> if still unemployed at survey end.</td></tr>
<tr><td><code>age</code></td><td>Age of respondent.</td></tr>
<tr><td><code>ui</code></td><td><code>1</code> if respondent has unemployment insurance.</td></tr>
<tr><td><code>reprate</code></td><td>Eligible replacement rate.</td></tr>
<tr><td><code>disrate</code></td><td>Eligible disregard rate.</td></tr>
<tr><td><code>logwage</code></td><td>Logarithmic weekly earnings.</td></tr>
<tr><td><code>tenure</code></td><td>Tenure at lost job.</td></tr>
</table>

As a last step in the introduction, summary statistics are computed for
each variable using my own `dsummary()` function, the definition of which
can be found in [my .Rprofile](https://github.com/mhoirup/dotfiles/blob/main/.Rprofile).


{% marginnote 'table1' "**Table 1**: Variable summaries of both numerical
and categorical variables. Take special note of the distribution of `ui`,
whether a respondent has unemployment insurance, which appear to be
somewhat evenly split among observations" %}

<table class='norulers' style='width:51%;'>
<tr><th>Variable</th><th>Type</th><th>N Unique</th><th colspan='2'>Mode</th><th>Min</th><th>Mean</th><th>Max</th><th>SD</th></tr>
<tr><td><code>spell  </code></td><td><code>integer</code></td><td style='text-align:right;'> </td><td style='text-align:center;'><code>     </code></td><td>&zwnj;</td><td style='text-align:right;'>1.00  </td><td style='text-align:right;'>6.25  </td><td style='text-align:right;'>28.00 </td><td style='text-align:right;'>5.61  </td></tr>
<tr><td><code>censor1</code></td><td><code>logical</code></td><td style='text-align:right;'>2</td><td style='text-align:center;'><code>False</code></td><td>67.90%</td><td style='text-align:right;'>&zwnj;</td><td style='text-align:right;'>&zwnj;</td><td style='text-align:right;'>&zwnj;</td><td style='text-align:right;'>&zwnj;</td></tr>
<tr><td><code>censor2</code></td><td><code>logical</code></td><td style='text-align:right;'>2</td><td style='text-align:center;'><code>False</code></td><td>89.86%</td><td style='text-align:right;'>&zwnj;</td><td style='text-align:right;'>&zwnj;</td><td style='text-align:right;'>&zwnj;</td><td style='text-align:right;'>&zwnj;</td></tr>
<tr><td><code>censor3</code></td><td><code>logical</code></td><td style='text-align:right;'>2</td><td style='text-align:center;'><code>False</code></td><td>82.83%</td><td style='text-align:right;'>&zwnj;</td><td style='text-align:right;'>&zwnj;</td><td style='text-align:right;'>&zwnj;</td><td style='text-align:right;'>&zwnj;</td></tr>
<tr><td><code>censor4</code></td><td><code>logical</code></td><td style='text-align:right;'>2</td><td style='text-align:center;'><code>False</code></td><td>62.45%</td><td style='text-align:right;'>&zwnj;</td><td style='text-align:right;'>&zwnj;</td><td style='text-align:right;'>&zwnj;</td><td style='text-align:right;'>&zwnj;</td></tr>
<tr><td><code>age    </code></td><td><code>integer</code></td><td style='text-align:right;'> </td><td style='text-align:center;'><code>     </code></td><td>&zwnj;</td><td style='text-align:right;'>20.00 </td><td style='text-align:right;'>35.44 </td><td style='text-align:right;'>61.00 </td><td style='text-align:right;'>10.64 </td></tr>
<tr><td><code>ui     </code></td><td><code>factor </code></td><td style='text-align:right;'>2</td><td style='text-align:center;'><code>Yes  </code></td><td>55.28%</td><td style='text-align:right;'>&zwnj;</td><td style='text-align:right;'>&zwnj;</td><td style='text-align:right;'>&zwnj;</td><td style='text-align:right;'>&zwnj;</td></tr>
<tr><td><code>reprate</code></td><td><code>double </code></td><td style='text-align:right;'> </td><td style='text-align:center;'><code>     </code></td><td>&zwnj;</td><td style='text-align:right;'>0.07  </td><td style='text-align:right;'>0.45  </td><td style='text-align:right;'>2.06  </td><td style='text-align:right;'>0.11  </td></tr>
<tr><td><code>disrate</code></td><td><code>double </code></td><td style='text-align:right;'> </td><td style='text-align:center;'><code>     </code></td><td>&zwnj;</td><td style='text-align:right;'>0.00  </td><td style='text-align:right;'>0.11  </td><td style='text-align:right;'>1.02  </td><td style='text-align:right;'>0.07  </td></tr>
<tr><td><code>logwage</code></td><td><code>double </code></td><td style='text-align:right;'> </td><td style='text-align:center;'><code>     </code></td><td>&zwnj;</td><td style='text-align:right;'>2.71  </td><td style='text-align:right;'>5.69  </td><td style='text-align:right;'>7.60  </td><td style='text-align:right;'>0.54  </td></tr>
<tr><td><code>tenure </code></td><td><code>integer</code></td><td style='text-align:right;'> </td><td style='text-align:center;'><code>     </code></td><td>&zwnj;</td><td style='text-align:right;'>0.00  </td><td style='text-align:right;'>4.11  </td><td style='text-align:right;'>40.00 </td><td style='text-align:right;'>5.86  </td></tr>
</table>


{% marginfigure 'fig4' 'assets/unemp/rates_density.png' "**Figure 4**:
Densities of `reprate` and `disrate`. As can be seen in table 1, both
variables are of double type, however, examining their number unique
values, e.g. `length(unique(data$reprate))`, reveal
that neither are exactly continuous; `reprate` has 419 unique values while
`disrate` has 246. The range of both variables is too long to treat them as
discrete, so we're going to leave them as is." %}

## Censoring Mechanisms

**Censoring** refers to the inaccurate measurement of the observed duration $t$
due to the way data is sampled. Different types of censoring exist, which
is defined by the censoring time $c$, which denote the time at which a
realisation of the random variable $T$ was sampled. For an observation not
to be censored we require that $t$ is known, and therefore the implied
relation $0\leqslant t\leqslant c$. In the survival analysis literature we
typically deal with one of the following censoring mechanisms:

<table class='norulers'>
<tr><td><strong>right-<br>censoring</strong></td><td>Spell is observed from time 0 until censoring time $c$. Time $t$ is unknown and somewhere in the interval $(c,\infty)$.</td></tr>
<tr><td><strong>left-<br>censoring</strong></td><td>Spell has already ended at censor time $c$. Time $t$ is unknown but somewhere in the interval $(0,c)$.</td></tr>
<tr><td><strong>interval-<br>censoring</strong></td><td>Spell has ended at censor time $c$, but time $t$ is unknown. What is known is that $t$ is somewhere in the interval $[t_1^*,t_2^*]$.</td></tr>
</table>

Define the **censoring mechanism** as $C$, and let $T^\*$ denote the true
(as in, unaffected by censoring) version of the observed variable $T$. In
our sample we thus observe the (potentially) censored $T=\min(T^\*, C)$, as
well as a censoring indicator $\delta=I(T^\*<C)$, which equals 1 if the
observation has *not* been censored and 0 otherwise. For standard survival
analysis methods to be valid under the presence of censoring, we require
that $C$ is independent of $T^\*$, i.e. the parameters of the distribution
of $C$ are uninformative about the parameters of the distribution of $T^\*$.  

Multiple censoring mechanisms exists, with the type of mechanism depending
on the way the sampling is carried. In this case we have **type I**
censoring, whereby $C$ is effectively a fixed value, namely the time the
survey ends. Due to the fixed $C$ for all $t$, we have $\Pr(T\geqslant
t\mid \delta = 0)=\Pr(T\geqslant t)$ and $\Pr(T=t\mid \delta=0)=\Pr(T=t)$,
a property of all independent censoring mechanisms, which allows us to
treat $\delta$ as exogenous, meaning we don't have to model $C$. 

{% maincolumn 'assets/unemp/spell_censor4.png' "**Figure 4**: `censor4` over the values
of `spell`. Unsurprisingly, the length of the spell appear to correspond to
a greater chance of an observation being censored, which aligns with the
conventional wisdom that unemployment is harder to escape the longer one is
in it, making a spell exceed the sampling period. The correlation
coefficient between `censor4` and `spell` is 0.31, which adds evidence to
the previous statement." %}

{% marginfigure 'fig5' 'assets/unemp/logwage_hist.png' "**Figure 5**:
Density of `logwage`. Observations are centred around the 5.5
mark, with a distribution that appear to be, for the most part, bell shaped." %}

{% marginfigure 'fig6' 'assets/unemp/tenure_hist.png' "**Figure 6**:
Histogram of `tenure`. 2.587 observations, 77% of the data, had a tenure of
five years or less at their lost job, which is unsurprising, as typically
less experienced personnel is let of off first." %}

## Survivor, Hazard and Cumulative Hazard Functions

Let $T$ denote our response variable, in this case `spell`. For the
variable $T$, which quantifies the duration of moving from one state to
another, we associate with it the **survivor function** $S(t)=\Pr(T>t)$. The
survivor function is related to the CDF as $S(t)=1-F(t)$, and thus gives
the probability of continuing in the state from $t_j$ to $t_{j+1}$. Since
$S(t)$ is monotonically declining, we have $\lim_{t\to\infty}S(t)=0$ under
the assumption that spells ultimately do end. From $S(t)$, we can obtain
$E(T)$ as the area under the survival curve as
$E(T)=\int_{0}^{\infty}S(u)du$. 

While $S(t)$ gives the probability of *not* transitioning at period $t$,
the **hazard function** $h(t)$ gives the probability of transition at
period $t$. Here we see the terminology's roots within the medical sciences
once more, as $h(t)$ gives the 'hazard' of transitioning from the
state of life to the state of death. The hazard function is defined as

$$
    h(t)=\begin{cases}
    \lim\limits_{\Delta t\to 0}\dfrac{\Pr(t\leqslant T<t+\Delta t\mid
        T\geqslant t)}{\Delta t}\equiv -\dfrac{d\ln
        S(t)}{dt}& \text{for continuous $T$} \\
        \Pr(T=t\mid T\geqslant t) & \text{for discrete $T$}
    \end{cases}
$$

with the equality $h(t)=f(t)/S(t)$ holding in both cases, where
$f(t)=dF(t)/dt$ is the density/mass function of $T$.  Lastly,
we have the **cumulative hazard function**, in which we integrate
$h(t)$ over the interval $[0,t]$, so that

$$
    H(t)=\begin{cases}
    \int_{0}^{t}h(u)du& \text{for continuous }T \\
    \sum_{j\mid t_j\leqslant t}^{} h(t_j)& \text{for discrete $T$}
    \end{cases}
$$

There's therefore no probabilistic interpretation of $H(t)$, as
it's essentially the summation of very small probabilities, and therefore
not bounded within the unit interval, but it's a quantity which increases
as $t\to\infty$, and can be viewed as the increased hazard of transitioning
states as time goes on.

## Nonparametric Models

{% marginfigure 'fig8' 'assets/unemp/kaplan_meier.png' "**Figure 8**:
Kaplan-Meier estimator on `spell`, with the shaded
area giving the 95% confidence interval. Note the sharp decline until around
the 7x2 = 14 week mark, after which the rate of decline in 'survival'
appear to diminish a bit." %}

{% marginfigure 'fig9' 'assets/unemp/kaplan_meier_mv.png' "**Figure 9**:
Kaplan-Meier estimator on `spell` stratified by `ui`, with shaded area giving the
95% confidence interval. Unsurprisingly, those who have unemployment
insurance are more likely to stay unemployment than those without, as the
financial burden of unemployment is alleviated." %}

Sample estimates for $S(t)$ and $H(t)$ are readily available via the
**Kaplan-Meier** and **Nelson-Aalen** estimators, respectively. In
constructing these estimators, we make use of the following variables:  

<table class='norulers'>
<tr><td>$d_j$</td><td>The number of completed spells at time $t_j$.</td></tr>
<tr><td>$m_j$</td><td>The number of right-censored spells in the interval $[t_j,t_{j+1})$.</td></tr>
<tr><td>$r_j$</td><td>The number of spell that have neither ended nor been censored just before time $t_j$, which can be computed as $r_j=\sum_{l\mid l\geqslant j}(d_l+m_l)$.</td></tr>
</table>

Using the sample estimate of the hazard function
$\hat{h}(t_j)=d_j/r_j$, we can then derive the nonparametric models
as $\hat{S}(t)=\prod_{t_j\mid t_j\leqslant t}1-\hat{h}(t_j)$ for the
Kaplan-Meier estimate of the survivor function, and
$\hat{H}(t)=\sum_{t_j\mid t_j\leqslant t} \hat{h}(t_j)$ for the
Nelson-Aalen estimate of the cumulative hazard. Their estimation in R is
pretty straightforward:

{% maincolumn 'assets/unemp/nonparam_uni.png' "**Figure 4**: `censor4` over the values
of `spell`. Unsurprisingly, the length of the spell appear to correspond to
a greater chance of an observation being censored, which aligns with the
conventional wisdom that unemployment is harder to escape the longer one is
in it, making a spell exceed the sampling period. The correlation
coefficient between `censor4` and `spell` is 0.31, which adds evidence to
the previous statement." %}

{% maincolumn 'assets/unemp/nonparam_stratified.png' "**Figure 4**: `censor4` over the values
of `spell`. Unsurprisingly, the length of the spell appear to correspond to
a greater chance of an observation being censored, which aligns with the
conventional wisdom that unemployment is harder to escape the longer one is
in it, making a spell exceed the sampling period. The correlation
coefficient between `censor4` and `spell` is 0.31, which adds evidence to
the previous statement." %}

```R
# Univariate sample estimates
uvnonparam <- survfit(Surv(spell, censor4 == 0) ~ 1, data)
# Multivariate sample estimates stratified by ui
mvnonparam <- survfit(Surv(spell, censor4 == 0) ~ ui, data)
```

While the confidence intervals were readily available in the `survfit`
object for $\hat{S}(t)$, they must be manually computed for
$\hat{H}(t)$, which we do as  

```R
chazci <- function(model, alpha = .05) {
    lower <- model$cumhaz * exp(qnorm(alpha / 2, 0, 1) * model$std.chaz)
    upper <- model$cumhaz * exp(-qnorm(alpha / 2, 0, 1) * model$std.chaz)
    data.frame(lower, upper)
}
```

## Parametric Models for Survival Analysis

### Distributions for Duration Data

Here we'll introduce a selection of distributions we'll consider in our
parametric model of the survival times. We consider four distributions in
total, the exponential, Weibull, log-normal and log-logistic, but this list
is by no means exhaustive. We'll fit each distribution to the data and
estimate it's parameters via MLE, and then use those parameters to estimate
the survivor function and the cumulative hazard for that distribution on
`spell`.

<span style='font-weight:bold;font-size:1.1rem;margin-right:1rem'>Exponential</span>
Under the assumption that $T\sim \text{Exp}(\lambda)$, where $\lambda$ is
the rate (or inverse scale) parameter of the distribution, the hazard
function will be given as the constant $h(t)=\lambda$. Since the density
function of $T\sim \text{Exp}(\lambda)$ is $f(t)=\lambda\exp(-\lambda t)$,
we have $S(t)=\exp(-\lambda t)$. Additionally, we have the moments
$E(T)=1/\lambda$ and $\text{Var}(T)=1/\lambda^2$.

<span style='font-weight:bold;font-size:1.1rem;margin-right:1rem'>Weibull</span>
Under the assumption that $T\sim W(\lambda,p)$, where $\lambda$ and $p$ are
scale and shape parameters respectively, we have the hazard
$h(t)=\lambda^ppt^{p-1}$, density
$f(t)=\frac{p}{\lambda}(\frac{t}{\lambda})^{p-1}\exp[-(t/\lambda)^p]$
defined for $t\geqslant 0$ (otherwise zero), and the survivor function
$S(t)=\exp[-(\lambda t)^p]$. The Weibull distribution is closely related to
the exponential distribution, since, if $T\sim W(\lambda,p)$ then $T^p\sim
\text{Exp}(\lambda)$.

<span style='font-weight:bold;font-size:1.1rem;margin-right:1rem'>Log-Normal</span>
Under the assumption that $T\sim \text{Lognormal}(\mu,\sigma^2)$, where
$\mu$ and $\sigma$ are location and scale parameters respectively, we have
the hazard and density functions given as

$$
    \begin{align}
    h(t)&=\frac{\exp[-(\ln
    t-\mu)^2/2\sigma^2]}{t\sigma\sqrt{2\pi}[1-\Phi((\ln t-\mu)/\sigma)}
    \\[1em]
    f(t)&=\frac{1}{t\sigma\sqrt{2\pi}}\exp(-\frac{(\ln t-\mu)^2}{2\sigma})
    \end{align}
$$

where $\Phi(\cdot)$ is the standard Gaussian cdf. Additionally, under
lognormality we have the survivor function $S(t)=1-\Phi[(\ln t -\mu)/\sigma]$. 

<span style='font-weight:bold;font-size:1.1rem;margin-right:1rem'>Log-Logistic</span>
Under the assumption that $T\sim \text{Loglogistic}(\alpha, \beta)$, where
$\alpha$ and $\beta$ are scale and shape parameters respectively, we have
the hazard $h(t)=[(\beta/\alpha)(t/\alpha)^{\beta-1}]/[1+(t/\alpha)]$, the
density function
$f(t)=[(\beta/\alpha)(t/\alpha)^{\beta-1}]/[1+(t/\alpha)]^2$ and the
survivor function $S(t)=[1+(t/\alpha)^{\beta}]^{-1}$.

{% marginnote 'sn1' "Since the log-logistic distribution functions are
imported, we need to specify the actual R function in `fitdistr` rather
than giving a string as it's argument. This has the implication that we
also need to manually specify starting values for the optimisation
procedure. I tried a few different starting values for the parameters, and
all converged to the same values. In the code you'll find starting values
at 1 for both parameters - a nice generic value :-)" %} 

Each distribution is fitted to the data via `fitdistr` from [the MASS
package](https://cran.r-project.org/web/packages/MASS/MASS.pdf). For
example, the exponential distribution is fitted as
`MASS::fitdistr(data$spell, 'exponential')`. All distribution are available
base R, except for the log-logistic, which we instead import from [the
actuar package](https://cran.r-project.org/web/packages/actuar/actuar.pdf).
Parameter estimates were then extracted from the resulting object, and are
given in the table below with standard errors in parentheses.




<table class='norulers' style='width:35%;'>
<tr><th></th><th><strong>Location</strong></th><th><strong>Scale/Rate</strong></th><th><strong>Shape</strong></th></tr>
<tr><td>Exponential<td></td></td><td style='text-align:center;'><span style='color:#fafbfc;'>(</span>0.160<span style='color:#fafbfc'>)</span>(0.003)</td><td></td></tr>
<tr><td>Weibull</td><td></td><td style='text-align:center;'><span style='color:#fafbfc;'>(</span>1.184<span style='color:#fafbfc'>)</span>(0.016)</td><td style='text-align:center;'><span style='color:#fafbfc;'>(</span>6.649<span style='color:#fafbfc'>)</span>(0.103)</td></tr>
<tr><td>Log-Normal</td><td style='text-align:center;'><span style='color:#fafbfc;'>(</span>1.439<span style='color:#fafbfc'>)</span>(0.016)</td><td style='text-align:center;'><span style='color:#fafbfc;'>(</span>0.918<span style='color:#fafbfc'>)</span>(0.011)</td><td></td></tr>
<tr><td>Log-Logistic</td><td></td><td style='text-align:center;'><span style='color:#fafbfc;'>(</span>1.826<span style='color:#fafbfc'>)</span>(0.026)</td><td style='text-align:center;'><span style='color:#fafbfc;'>(</span>0.235<span style='color:#fafbfc'>)</span>(0.004)</td></tr>
</table>

{% maincolumn 'assets/unemp/pdfs.png' "**Figure 4**: `censor4` over the values
of `spell`. Unsurprisingly, the length of the spell appear to correspond to
a greater chance of an observation being censored, which aligns with the
conventional wisdom that unemployment is harder to escape the longer one is
in it, making a spell exceed the sampling period. The correlation
coefficient between `censor4` and `spell` is 0.31, which adds evidence to
the previous statement." %}

