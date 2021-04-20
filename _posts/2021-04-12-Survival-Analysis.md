---
layout: post
title:  'Survial Analysis of Unemployment Data'
date:   2021-04-19
---

Survival analysis is the statistical analysis of duration data. Durations
can be anything from life until death in the medical sciences, run until
failure in engineering, or in this case, unemployment duration. In this
project we seek to understand the factors affecting the probability of
returning to the workforce.

{% marginfigure 'spell_hist' 'assets/unemp/spell_hist.png'
'Distribution of `spell`. A majority of respondents finished their
unemployment spell prior to 10x2 = 20 week mark, with relatively few
surpassing 15 two-week intervals.' %}

We begin the analysis by loading the data into our R workspace along with
the packages we're going to use. 'Data is loaded from the `Ecdat` package
in R, which can be found at
[https://cran.r-project.org/web/packages/Ecdat/Ecdat.pdf](https://cran.r-project.org/web/packages/Ecdat/Ecdat.pdf)
. The data was used in McCall, B. P. (1996) “Unemployment Insurance Rules,
Joblessness, and Part-time Work”, *Econometrica*, 64, 647–682.

{% marginfigure 'censored_hist' 'assets/unemp/censored_hist.png' "Distributions
of the four `censor` variables where I've changed the `1`s and `0`s into
booleans for clarity." %}

{% marginfigure 'age_hist' 'assets/unemp/age_hist.png' 'Distribution
of `age`. Bin width is set at 2.'  %}

```R
library(dplyr)
library(Ecdat)
library(survival)
library(ggplot2)

data(UnempDur); data <- UnempDur; rm(UnempDur)
glimpse(data)
# Rows: 3,343
# Columns: 11
# $ spell   <dbl> 5, 13, 21, 3, 9, 11, 1, 3, 7, 5, 7, 15, 2, 3, 1, 2, 14, 1, 1,…
# $ censor1 <dbl> 1, 1, 1, 1, 0, 0, 0, 1, 1, 0, 0, 0, 1, 0, 1, 1, 0, 1, 0, 0, 0…
# $ censor2 <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0…
# $ censor3 <dbl> 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0…
# $ censor4 <dbl> 0, 0, 0, 0, 0, 1, 0, 0, 0, 1, 1, 1, 0, 1, 0, 0, 0, 0, 0, 1, 1…
# $ age     <dbl> 41, 30, 36, 26, 22, 43, 24, 32, 35, 31, 26, 49, 39, 40, 20, 3…
# $ ui      <fct> no, yes, yes, yes, yes, yes, no, no, yes, yes, yes, yes, no, …
# $ reprate <dbl> 0.179, 0.520, 0.204, 0.448, 0.320, 0.187, 0.520, 0.373, 0.520…
# $ disrate <dbl> 0.045, 0.130, 0.051, 0.112, 0.080, 0.047, 0.130, 0.093, 0.130…
# $ logwage <dbl> 6.89568, 5.28827, 6.76734, 5.97889, 6.31536, 6.85435, 5.60947…
# $ tenure  <dbl> 3, 6, 1, 3, 0, 9, 1, 0, 2, 1, 2, 21, 0, 0, 0, 14, 10, 10, 0, …

```

{% marginfigure 'reprate_hist' "assets/unemp/reprate_hist.png"
"Distribution of `reprate`. deriving the unique value via
`length(unique(data$reprate))` actually shows that the variable, with 3.343
values, only has 419 unique values, suggesting that the replacement isn't
necessarily continuous."  %}

{% marginfigure 'disrate_hist' 'assets/unemp/disrate_hist.png' 'Distribution
of `age`. Bin width is set at 2.'  %}


Below is a table with a brief description of each variable. In particular,
note how `spell` give the duration of unemployment in two-week intervals,
for example, if `spell == 2`, a respondent has been unemployed for four
weeks. Except for `censor4`, the `censor` variables will not be needed in
this analysis, as we're only interested in whether or not respondents had
entered the workforce at the end of the survey.

|**Variable** |**Description**                                              |
|:------------|:------------------------------------------------------------|
|`spell`      | Unemployment duration (two-week intervals)                  |
|`censor1`    | `1` if reemployed at a full-time position.                  |
|`censor2`    | `1` if reemployed at a part-time position                   |
|`censor3`    | `1` if reemployed but later resigned from that reemployment | 
|`censor4`    | `1` if still unemployed                                     |
|`age`        | Age of respondent                                           |
|`ui`         | `1` if respondent has unemployment insurance                |
|`reprate`    | Eligeble replacement rate                                   |
|`disrate`    | Eligeble disregard rate                                     |
|`logwage`    | Logarithmic weekly earnings                                 |
|`tenure`     | Tenure at lost job                                          |


As a last step in the introduction, summary statistics are computed for
each variable. The `dsummary()` function is my own and placed in my
`.Rprofile`, which can be found at
[https://github.com/mhoirup/dotfiles/blob/main/.Rprofile](https://github.com/mhoirup/dotfiles/blob/main/.Rprofile).

```R
> dsummary(data)
# $categorical
#       var    type n.unique  mode prop.mode NAs
# 1 censor1 logical        2 FALSE     67.90   0
# 2 censor2 logical        2 FALSE     89.86   0
# 3 censor3 logical        2 FALSE     82.83   0
# 4 censor4 logical        2 FALSE     62.46   0
# 5      ui  factor        2   yes     55.28   0
# 
# $numeric
#       var    type   min  mean   max    sd NAs
# 1   spell numeric  1.00  6.25 28.00  5.61   0
# 2     age numeric 20.00 35.44 61.00 10.64   0
# 3 reprate numeric  0.07  0.45  2.06  0.11   0
# 4 disrate numeric  0.00  0.11  1.02  0.07   0
# 5 logwage numeric  2.71  5.69  7.60  0.54   0
# 6  tenure numeric  0.00  4.11 40.00  5.86   0

```

## Censoring and Flow Sampling

Censoring in the context of survival data is the incomplete measure of a
spell; a spell may still be ongoing at the time of the measurement, in
which case we have a censered observation. 


{% maincolumn 'assets/unemp/spell_censor4.png' "`Censor4` over the values
of `spell`. " %}

{% marginfigure 'logwage_hist' 'assets/unemp/logwage_hist.png' 'Distribution
of `age`. Bin width is set at 2.'  %}

## Survivor, Hazard and Cumulative Hazard Functions


{% marginfigure 'tenure_hist' 'assets/unemp/tenure_hist.png' 'Distribution
of `age`. Bin width is set at 2.'  %}

Let $T$ denote our response variable, in this case `spell`. For the
variable $T$, which quantifies the duration of moving from one state to
another, we associate with it the **survivor function** $S(t)=\Pr(T>t)$. The
survivor function is related to the CDF as $S(t)=1-F(t)$, and thus gives
the probability of continuing in the state from $t_j$ to $t_{j+1}$. Since
$S(t)$ is monotonically declining, we have $\lim_{t\to\infty}S(t)=0$ under
the assumption that spells ultimately do end. From $S(t)$, we can obtain
$E(T)$ as the area under the survival curve:

$$
    E(T)=\int_{0}^{\infty}[1-F(v)]\mathrm{d}v=\int_{0}^{\infty}S(v)\mathrm{d}v.
$$

While $S(t)$ gives the probability of continuing to the subsequent period
without transition, the **hazard function** $\lambda(t)$ gives the
probability of transition at period $t$. Here we see the
terminology's roots within the medical sciences once more, as $\lambda(t)$
gives the 'hazard' of transitioning from the state of life to the state of
death. The hazard function is defined as

$$
    \begin{aligned}
        \lambda(t)&=\lim_{\Delta t\to 0}\frac{\Pr(t\leqslant T<t+\Delta t\mid
        T\geqslant t)}{\Delta t}=\frac{f(t)}{S(t)} \\
        &=-\frac{\mathrm{d}\ln S(t)}{\mathrm{d}t}
    \end{aligned}
$$

where $f(t)=\mathrm{d}F(t)/\mathrm{d}t$ is the density function of $T$. Lastly, we have the
**cumulative hazard function**, in which we integrate $\lambda(t)$ over the
interval $[0,t]$, so that

$$
    \Lambda(t)=\int_{0}^{t}\lambda(v)\mathrm{d}v
$$

There's therefore no probabilistic interpretation of $\Lambda(t)$, as
it's essentially the summation of very small probabilities, and therefore
not bounded within the unit interval, but it's a quantity which increases
as $t\to\infty$, and can be viewed as the increased hazard of transitioning
states as time goes on.

### Function Definitions for Discrete Time

In many applications, such as this one, we observe $T$ in discrete time,
although the transition generating process is intristically continuous, in
which we have what is known as **grouped data**. In such cases, where the
data is grouped or the distribution of $T$ is actually discrete, we make
the following corrections to the definitions given above:  

$$
    \begin{gather}
    S(t)=\Pr(T\geqslant t)=\prod_{t_j\mid t_j\leqslant t} 1-\lambda(t_j) \\
    \lambda(t)=\Pr(T=t\mid T\geqslant t)=\frac{f(t)}{S(t)} \\
    \Lambda(t)=\sum_{t_j\mid t_j\leqslant t} \lambda(t_j)
    \end{gather}
$$

## Nonparametric Models

{% marginfigure 'kaplan_meier' 'assets/unemp/kaplan_meier.png'
'Kaplan-Meier estimator of the survivor function on `spell`. Shaded area
gives the 95% confidence bands. Note the sharp decline until around 8
two-week intervals, after which rate of declining probability diminishes a
bit.' %}

{% marginfigure 'kaplan_meier_mv' 'assets/unemp/kaplan_meier_mv.png'
'Kaplan-Meier estimator of the survivor function on `spell`. Shaded area
gives the 95% confidence bands. Note the sharp decline until around 8
two-week intervals, after which rate of declining probability diminishes a
bit.' %}

Sample estimates for both the survivor and the cumulative hazard functions
are readily available. For the former, we'll use the **Kaplan-Meier
estimator**. Let $t_j:j=1,\ldots,k$ be the set of discrete
state-change times of the spells in the sample (i.e. the unique values of
`spell`), where $d_j$ is the number of spells that end at time $t_j$, and
$m_j$ is the number of right-censored spells in the interval
$[t_j,t_{j+1})$. An observation is said to be **at risk** if it has not yet
transitioned or has been censored, where we define those the number of at risk
observations as $r_j=\sum_{l\mid l\geqslant j}^{}(d_l+m_l)$. We then have
the estimator

$$
    \hat{S}(t)=\prod_{j\mid t_j\leqslant t}1-\hat{\lambda}(t_j)=\prod_{t_j\mid
    t_j\leqslant t}\frac{r_j-d_j}{d_j}
$$

where $\hat{\lambda}(t_j)=d_j/r_j$ is our (discrete time) sample estimate for
the hazard function. $100(1-\alpha)$% confidence bands are obtained via the
**Greenwood estimate** of the standard deviation

$$
    \hat{\sigma}(t)=\sqrt{\hat{S}(t)^2\sum_{j\mid t_j\leqslant t}
    \frac{d_j}{r_j(r_j-d_j)}}
$$

$$
    S(t)\in\left[\hat{S}(t)\exp\left(-z_{\alpha/2}\cdot\hat{\sigma}(t)\right),
    \ \hat{S}(t)\exp\left(z_{\alpha/2}\cdot\hat{\sigma}(t)\right)\right]
$$


Nonparametric estimates are computed in R via

```R
# Univariate sample estimates
uvnonparam <- survfit(Surv(spell, censor4 == 0) ~ 1, data)
# Multivariate sample estimates stratified by ui
mvnonparam <- survfit(Surv(spell, censor4 == 0) ~ ui, data)

```

where the Kaplan-Meier estimate of the survivor function is obtained from
the `surv` attribute, and the confidence bands from the `upper` and `lower`
attributes from the resulting model object.

{% marginfigure 'nelson_aalen' 'assets/unemp/nelson_aalen.png'
'Kaplan-Meier estimator of the survivor function on `spell`. Shaded area
gives the 95% confidence bands. Note the sharp decline until around 8
two-week intervals, after which rate of declining probability diminishes a
bit.' %}

{% marginfigure 'nelson_aalen_mv' 'assets/unemp/nelson_aalen_mv.png'
'Kaplan-Meier estimator of the survivor function on `spell`. Shaded area
gives the 95% confidence bands. Note the sharp decline until around 8
two-week intervals, after which rate of declining probability diminishes a
bit.' %}


Similar to the Kaplan-Meier estimator, we have the **Nelson-Aalen
estimator** for the cumulative hazard being given as

$$
    \hat{\Lambda}(y)=\sum_{j\mid y_j\leqslant y}\hat{\lambda}_j=
    \sum_{j\mid y_j\leqslant y }\frac{d_j}{r_j}
$$

while the confidence intervals for the survivor function were available
within the `survfit` object, we have to compute them manually for $\hat{\Lambda}(t)$. 

## Proportional Hazard Models


$$
    \lambda(t\mid \boldsymbol{x})=\lambda_0(t,\alpha)\phi(\boldsymbol{x},\boldsymbol{\beta})
$$





```R
for (dist in c('weibull', 'exponential', 'gaussian')) {
    covariates <- regressors; reduced <- TRUE
    while (reduced) {
        unrestricted <- survreg(spec(covariates), data, dist = dist)
        cfs <- coeftest(unrestricted); rows <- nrow(progress)
        pvalues <- cfs[!(rownames(cfs) %in% c('(Intercept)', 'Log(scale)')), 4]
        idx <- sort(pvalues, index.return = TRUE, decreasing = TRUE)$ix
        for (var in covariates[idx]) {
            subset <- covariates[covariates != var]
            restricted <- survreg(spec(subset), data, dist = dist)
            lr_pvalue <- lmtest::lrtest(unrestricted, restricted)[[5]][2]
            new_params <- restricted$coefficients[-1]
            params <- unrestricted$coefficients[names(new_params)]
            differences <- abs(((new_params - params) / params)) * 100
            dmax <- c(names(which.max(differences)), max(differences))
            if (lr_pvalue > 0.05 && as.double(dmax[2]) < 20) {
                covariates <- subset
                progress <- rbind(progress, data.frame(
                    dist, var, lr_pvalue, delta_var = dmax[1], delta_p = dmax[2]
                ))
                break
            }
        }
        if (rows == nrow(progress)) break
    }
    # Finally we print a message which tells us (1) how many variables
    # are removed, (2) which were removed, and (3) the log-likelihood 
    # of the final specification.
    removed <- length(regressors) - length(covariates)
    omitted <- regressors[!(regressors %in% covariates)]
    message <- paste(dist, 'distribution: removed', removed, 'variable(s)',
        paste0('(', paste(omitted, collapse = ', '), ').'),
        '\n negative log-likelihood of final model:',
        round(logLik(unrestricted), 2), '\n\n')
    cat(message)
}

```
