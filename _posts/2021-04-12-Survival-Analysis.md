---
layout: post
title:  'Survial Analysis of Unemployment Data'
date:   2021-04-21
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
used in McCall (1996).

{% marginfigure 'fig2' 'assets/unemp/censored_hist.png' "**Figure
2**: Distributions of the four `censor` variables where I've changed the
`1`s and `0`s into booleans for clarity. Here we note the distribution of
`censor4` which is of interest; 1.255 observations have been censored,
while 2.088 observations correspond to completed spells." %}



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


{% marginnote 'tableID-3' "**Table 1**: Summaries of categorical variables.
They're all binary variables, although of different data types, i.e.
boolean or character vectors. For all the `censor` variables we observe
non-censoring more often than not, while the availability of unemployment
insurance appears to be evenly split among observations." %}

|Variable  |Type      |N Unique |Most Frequent Value|Min   |Mean  |Max   |SD    |
|:---------|:---------|--------:|------------------:|-----:|-----:|-----:|-----:|
|`spell`   |`integer` |         |                   |1.00  |6.25  |28.00 |5.61  |
|`censor1` |`logical` |2        |  `FALSE` (67.90%) |      |      |      |      |
|`censor2` |`logical` |2        |  `FALSE` (89.86%) |      |      |      |      |
|`censor3` |`logical` |2        |  `FALSE` (82.83%) |      |      |      |      |
|`censor4` |`logical` |2        |  `FALSE` (62.46%) |      |      |      |      |
|`age`     |`integer` |         |                   |20.00 |35.44 |61.00 |10.64 |
|`ui`      |`factor`  |2        |  `yes` (55.28%)   |      |      |      |      |
|`reprate` |`double`  |         |                   |0.07  |0.45  |2.06  |0.11  |
|`disrate` |`double`  |         |                   |0.00  |0.11  |1.02  |0.07  |
|`logwage` |`double`  |         |                   |2.71  |5.69  |7.60  |0.54  |
|`tenure`  |`integer` |         |                   |0.00  |4.11  |40.00 |5.86  |

{% marginfigure 'fig3' 'assets/unemp/rates_density.png' "**Figure 3**: Densities of
`reprate` and `disrate`." %}

{% marginfigure 'fig3' 'assets/unemp/age_hist.png' '**Figure 3**: Distribution
of `age`. Bin width is set at 2 to reduce the number of bars in the graph.
A majority of respondents is under 40 years of age, 2.231 in fact, which
corresponds to 68% of all observations.'  %}

{% marginfigure 'reprate_hist' "assets/unemp/reprate_hist.png"
"Distribution of `reprate`. deriving the unique value via
`length(unique(data$reprate))` actually shows that the variable, with 3.343
observations, only has 419 unique values, suggesting that the replacement isn't
necessarily continuous. Without much more info about the rate, and
considering the relatively large amount of unique values, we're going to
treat `reprate` as continuous."  %}

## Censoring and Flow Sampling

In the context of survival analysis, censoring refers to the measurement of
duration that is either still ongoing at the end of measurement
(**right-censoring** ), had begun before measurement (**left-censoring** ),
or 

<table class='norulers'>
<tr><td><strong>Right</strong></td><td>Spell is observed from time 0 until censoring time $c$. Time $t$ is somewhere in the interval $(c,\infty)$.</td></tr>
<tr><td><strong>Left</strong></td><td>Spell has already ended at censor time $c$. Time $t$ is unknown, but lies somewhere in the interval $(0,c)$.</td></tr>
<tr><td><strong>Interval</strong></td><td>Spell has ended at censor time $c$, but time $t$ is unknown. What is known is that $t$ is somewhere in the interval $[t_1^*,t_2^*]$.</td></tr>
</table>

{% maincolumn 'assets/unemp/spell_censor4.png' "`censor4` over the values
of `spell`. Unsurprisingly, the length of the spell appear to correspond to
a greater chance of an observation being censored, which aligns with the
conventional wisdom that unemployment is harder to escape the longer one is
in it, making a spell exceed the sampling period. The correlation
coefficient between `censor4` and `spell` is 0.31, which adds evidence to
the previous statement." %}

{% marginfigure 'disrate_hist' 'assets/unemp/disrate_hist.png' ''  %}


{% marginfigure 'logwage_hist' 'assets/unemp/logwage_hist.png' ''  %}

## Survivor, Hazard and Cumulative Hazard Functions


{% marginfigure 'tenure_hist' 'assets/unemp/tenure_hist.png' ''  %}

Let $T$ denote our response variable, in this case `spell`. For the
variable $T$, which quantifies the duration of moving from one state to
another, we associate with it the **survivor function** $S(t)=\Pr(T>t)$. The
survivor function is related to the CDF as $S(t)=1-F(t)$, and thus gives
the probability of continuing in the state from $t_j$ to $t_{j+1}$. Since
$S(t)$ is monotonically declining, we have $\lim_{t\to\infty}S(t)=0$ under
the assumption that spells ultimately do end. From $S(t)$, we can obtain
$E(T)$ as the area under the survival curve:

$$
    E(T)=\int_{0}^{\infty}[1-F(v)]dv=\int_{0}^{\infty}S(v)dv.
$$

While $S(t)$ gives the probability of *not* transitioning at period $t$,
the **hazard function** $\lambda(t)$ gives the probability of transition at
period $t$. Here we see the terminology's roots within the medical sciences
once more, as $\lambda(t)$ gives the 'hazard' of transitioning from the
state of life to the state of death. The hazard function is defined as

$$
    \lambda(t)=\begin{cases}
    \lim\limits_{\Delta t\to 0}\dfrac{\Pr(t\leqslant T<t+\Delta t\mid
        T\geqslant t)}{\Delta t}\equiv -\dfrac{d\ln
        S(t)}{dt}& \text{for continuous $T$} \\
        \Pr(T=t\mid T\geqslant t) & \text{for discrete $T$}
    \end{cases}
$$

with the equality $\lambda(t)=f(t)/S(t)$ holding in both cases, where
$f(t)=dF(t)/dt$ is the density/mass function of $T$.  Lastly,
we have the **cumulative hazard function**, in which we integrate
$\lambda(t)$ over the interval $[0,t]$, so that

$$
    \Lambda(t)=\begin{cases}
    \int_{0}^{t}\lambda(v)dv& \text{for continuous }T \\
    \sum_{j\mid t_j\leqslant t}^{} \lambda(t_j)& \text{for discrete $T$}
    \end{cases}
$$

There's therefore no probabilistic interpretation of $\Lambda(t)$, as
it's essentially the summation of very small probabilities, and therefore
not bounded within the unit interval, but it's a quantity which increases
as $t\to\infty$, and can be viewed as the increased hazard of transitioning
states as time goes on.

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

Sample estimates for $S(t)$ and $\Lambda(t)$ are readily available.  Let
$d_j$ to be the number of spell ending at time $t_j$, let $m_j$ to be the number
of right-censored spell in the interval $[t_j,t_{j+1})$, and $r_j$ is the number of
spell that have neither ended or been censored (a quality known as being at
risk) at time $t_j$. The **Kaplan-Meier** estimate for $S(t)$ is then  

$$
    \hat{S}(t)=\prod_{j\mid t_j\leqslant
    t}1-\hat{\lambda}(t_j)=\prod_{j\mid j\leqslant t}\frac{r_j-d_j}{d_j}
$$

where $\hat{\lambda}(t_j)=d_j/r_j$ is our sample estimate for
the hazard function. Similarly, a nonparametric specification is available
for cumulative hazard function via the **Nelson-Aalen** estimator given as 

$$
    \hat{\Lambda}(y)=\sum_{j\mid y_j\leqslant y}\hat{\lambda}(t_j)=
    \sum_{j\mid y_j\leqslant y }\frac{d_j}{r_j}
$$

Nonparametric estimates are computed in R via

```R
# Univariate sample estimates
uvnonparam <- survfit(Surv(spell, censor4 == 0) ~ 1, data)
# Multivariate sample estimates stratified by ui
mvnonparam <- survfit(Surv(spell, censor4 == 0) ~ ui, data)

```

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

While the confidence intervals were readily available in the `survfit`
object for $\hat{S}(t)$, they must be manually computed for
$\hat{\Lambda}(t)$, which we do as  

```R
chazci <- function(model, alpha = .05) {
    lower <- model$cumhaz * exp(qnorm(alpha / 2, 0, 1) * model$std.chaz)
    upper <- model$cumhaz * exp(-qnorm(alpha / 2, 0, 1) * model$std.chaz)
    data.frame(lower, upper)
}

```


## Semiparametric and Parametric Specifications

In the following we'll introduce semi- and fully-parametric specifications
for the either explicit, or implicit, estimation of the conditional hazard
rate $\lambda(t\mid \boldsymbol{x})$. For the parametric specifications the
entertained distributions are the exponential, Weibull, log-normal and
log-logistic distribution. Only the exponential and Weibull distributions
can be used for the **proportional hazard** (PH) model, while all can be used
for the **accelerated failure time** (AFT) model.

### Proportional Hazard Models

The PH model formulates $\lambda(t\mid \boldsymbol{x})$ as factored into
separate functions 

$$
    \lambda(t\mid \boldsymbol{x})=\lambda_0(t)
    h(\boldsymbol{x},\boldsymbol{\beta})
$$

where $\lambda_0(t)$ is the **baseline hazard**, that is, the hazard
function as specified by the distribution, and
$h(\boldsymbol{x},\boldsymbol{\beta})$ is a scale factor, where typically
$h(\boldsymbol{x},\boldsymbol{\beta})=\exp(\boldsymbol{x}^{\small{\prime}}\boldsymbol{\beta})$.
For example, under the Weibull distribution we have $\lambda_0(t)=$ 


### Variate Selection and Model Fitting


```R
specification <- function(response, regressors, lhs = NULL, rhs = NULL,
    polynomials = NULL, intercept = TRUE) {
    # This function is to ease the tinkering with variables in the model
    # estimation procedure. Since a string can be converted into an R
    # formula, manipulating the formula can be boiled down to string
    # manipulation.
    # Initiate the formula by concatenating the regressors with '+' and
    # 'response ~'.
    spec <- paste(response, '~', paste(regressors, collapse = '+'))
    # Create any interaction terms where terms are specified by lhs and rhs.
    if (!is.null(lhs) && !is.null(rhs)) {
        M <- expand.grid(lhs, rhs)
        idx <- sapply(seq_len(nrow(M)), function(i) M[i, 1] != M[i, 2])
        spec <- paste(spec, '+', paste(M[idx, 1], M[idx, 2], sep = ':',
            collapse = '+'))
    }
    # Add to the formula any polynomial terms of order 2.
    if (!is.null(polynomials)) {
        spec <- paste(spec, paste(paste0(polynomials, '^2'), collapse = '+'))
    }
    # Optional: remove intercept.
    if (intercept == FALSE) spec <- paste(spec, '-1')
    as.formula(spec)
}

spec <- function(x) specification('Surv(spell, censor4 == 0)', x)
regressors <- names(data)[!grepl('censor|spell', names(data))]
progress <- data.frame() # Bad practice to fill out a dataframe without
# specifying its size first, but we don't know how many variables
# are insignificant.

for (dist in c('weibull', 'exponential', 'lognormal', 'loglogistic', 'coxph')) {
    covariates <- regressors; reduced <- TRUE
    while (reduced) {
        # The 'coxph' isn't really a distribution, but here we simply specify
        # the Cox model instead if dist == 'coxph'.
        if (dist == 'coxph') unrestricted <- coxph(spec(covariates), data)
        else unrestricted <- survreg(spec(covariates), data, dist = dist)
        # Get the p-values and sort them highest to lowest. Idx gives the
        # indices in covariates corresponding to the sorted list of p-values.
        # Also keeps track of the current numbers of rows in progress.
        cfs <- coeftest(unrestricted); rows <- nrow(progress)
        pvalues <- cfs[!(rownames(cfs) %in% c('(Intercept)', 'Log(scale)')), 4]
        idx <- sort(pvalues, index.return = TRUE, decreasing = TRUE)$ix
        for (var in covariates[idx]) {
            # Estimate the model without the candiate variate and compute the
            # p-value for the likelihood ratio test.
            subset <- covariates[covariates != var]
            if (dist == 'coxph') restricted <- coxph(spec(subset), data)
            else restricted <- survreg(spec(subset), data, dist = dist)
            lr_pvalue <- lmtest::lrtest(unrestricted, restricted)[[5]][2]
            # Here we check for the percentage change in the remaining
            # coefficients, with an acceptable limit being at 15% in either
            # direction.
            new_params <- restricted$coefficients[-1]
            params <- unrestricted$coefficients[names(new_params)]
            differences <- abs(((new_params - params) / params)) * 100
            dmax <- c(names(which.max(differences)), max(differences))
            if (lr_pvalue > 0.05 && as.double(dmax[2]) < 15) {
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
    nremoved <- length(regressors) - length(covariates)
    which_removed <- regressors[!(regressors %in% covariates)]
    message <- paste(dist, 'distribution: removed', nremoved, 'variable(s)',
        paste0('(', paste(which_removed, collapse = ', '), ').'),
        '\n negative log-likelihood of final model:',
        round(logLik(unrestricted), 2), '\n\n')
    cat(message)
}
# weibull distribution: removed 1 variable(s) (tenure). 
#  negative log-likelihood of final model: -6573 
# 
# exponential distribution: removed 1 variable(s) (tenure). 
#  negative log-likelihood of final model: -6583.05 
# 
# lognormal distribution: removed 1 variable(s) (tenure). 
#  negative log-likelihood of final model: -6338.08 
# 
# loglogistic distribution: removed 1 variable(s) (tenure). 
#  negative log-likelihood of final model: -6378.36 
# 
# coxph distribution: removed 1 variable(s) (tenure). 
#  negative log-likelihood of final model: -15118.03

regressors <- regressors[regressors != 'tenure']

```


