
<!-- README.md is generated from README.Rmd. Please edit that file -->

# risks

<!-- badges: start -->

[![Lifecycle:
maturing](https://img.shields.io/badge/lifecycle-maturing-blue.svg)](https://www.tidyverse.org/lifecycle/#maturing)
[![Travis build
status](https://img.shields.io/travis/tgerke/risks/master?logo=travis&style=flat&label=Linux)](https://travis-ci.com/tgerke/risks)
[![Codecov test
coverage](https://codecov.io/gh/tgerke/risks/branch/master/graph/badge.svg)](https://codecov.io/gh/tgerke/risks?branch=master)
[![R CMD Check
Windows/MacOS/Ubuntu](https://github.com/tgerke/risks/workflows/R%20CMD%20Check%20Windows/MacOS/Ubuntu/badge.svg)](https://github.com/tgerke/risks/actions?query=workflow%3A%22R+CMD+Check+Windows%2FMacOS%2FUbuntu%22)
<!-- badges: end -->

The goal of risks is to … Further details are available at
[tgerke.github.io/risks/](https://tgerke.github.io/risks/).

## Installation

You can install the released version of risks from
[CRAN](https://CRAN.R-project.org) with:

``` r
install.packages("risks")
```

And the development version from [GitHub](https://github.com/) with:

``` r
# install.packages("devtools")
devtools::install_github("tgerke/risks")
```

# Summary

The `risks` packages provides a flexible interface to fitting regression
models for risk ratios and risk differences, as well as prevalence
ratios and prevalence differences. (For brevity, the vignette describes
“risk,” but “prevalence” can be substituted throughout.)

Emphasis is on a user-friendly approach that does not require advanced
programming or biostatistics skills while still providing users with
options for customization of model use and reporting as well as
comparisons between different approaches.

Implemented are Poisson models with robust covariance, binomial models,
binomial models aided in convergence by starting values obtained through
Poisson models, binomial models fitted via combinatorial expectation
maximization (optionally also with Poisson starting values), and
estimates obtained via marginal standardization after fitting logistic
models.

# Basic usage

We define a cohort of women with breast cancer, used by Spiegelman and
Hertzmark (Am J Epidemiol 2005) and Greenland (Am J Epidemiol 2004). The
the categorical exposure is `stage`, the binary outcome is `death`, and
the binary confounder is `receptor`.

``` r
knitr::opts_chunk$set(echo = TRUE, eval = FALSE)
```

``` r
library(risks)

# Newman SC. Biostatistical methods in epidemiology. New York, NY: Wiley, 2001, table 5.3
dat <- tibble(
  id = 1:192,
  death = c(rep(1, 54), rep(0, 138)),
  stage = c(rep("Stage I", 7),  rep("Stage II", 26), rep("Stage III", 21),
            rep("Stage I", 60), rep("Stage II", 70), rep("Stage III", 8)),
  receptor = c(rep("Low", 2),  rep("High", 5),  rep("Low", 9),  rep("High", 17),
               rep("Low", 12), rep("High", 9),  rep("Low", 10), rep("High", 50),
               rep("Low", 13), rep("High", 57), rep("Low", 2),  rep("High", 6)))
```

  
Using `risks` models to obtain (possibly multivariable-adjusted) risk
ratios or risk differences is similar to the standard code for logistic
models in R. No options for model `family` or `link` function can or
must be supplied:

``` r
fit_rr <- estimate_risk(formula = death ~ stage + receptor, data = dat)
summary(fit_rr)
```

  
By default, `risks` fits models for relative risk (`"rr"`). To obtain
risk differences, add the option `estimate = "rd"`:

``` r
fit_rd <- estimate_risk(formula = death ~ stage + receptor, data = dat, estimate = "rd")
summary(fit_rd)
```

For example, the risk of death was 57 percentage points higher in women
with stage III breast cancer compared to stage I (95% confidence
interval, 39 to 76 percentage points), adjusting for hormone receptor
status.

The model summary in `risks` includes to two additions compared to a
regular `glm` model:

  - In the first line of `summary(...)`, the type of risk regression
    model is displayed (in the example, “`Risk ratio model, fitted as
    binomial model...`”).
  - At the end of the output, confidence intervals for the model
    coefficients are printed.

  
`risks` provides an interface to `tidy()`, which returns a data frame
(or tibble) of all coefficients (risk differences), their standard
errors, and confidence intervals. Confidence intervals are included by
default.

``` r
tidy(fit_rd)
```

  
In accordance with R standards, coefficients for relative risks are
shown on the \(log_e(RR)\) scale. Exponentiated coefficients (risk
ratios, \(RR\)) are easily obtained via `tidy(..., exponentiate =
TRUE)`:

``` r
tidy(fit_rr, exponentiate = TRUE)
```

For example, the risk of death was 5.87 times higher in women with stage
III breast cancer compared to stage II (95% confidence interval, 2.76 to
12.48 times), adjusting for hormone receptor status.

  
Typical R functions that build on regression models can further process
fitted `risks` models. Examples:

  - `coef(fit)` returns model coefficients (*i.e.*, \(log(RR)\)s or RDs)
    as a numeric vector
  - `confint(fit, level = 0.9)` returns *90%* confidence intervals.
  - `predict(fit, type = "response")` returns predicted probabilities of
    the binary outcome.

# Models

What is the association between an exposure (perhaps men/women, age in
years, or underweight/lean/overweight/obese) and the risk of a binary
outcome (yes/no, dead/alive, disease/healthy), perhaps adjusting for
confounders (smoker/nonsmoker, years of completed education)? In a
cohort study, this association is best expressed as a risk ratio (RR) or
as a risk difference (RD). It is is well-recognized theoretically that
is it unneccessary to report an odds ratio (OR) in cohort studies,
because their interpretation differs from risk ratios and numerically
diverges the stronger the association is. Many regression models have
been proposed to directly estimate risk ratios and risk differences.
However, implementing them in standard software, including R, has
typically required more advanced programming skills.

`risks` implements all major regression models that have been proposed
for relative risks and risk differences. By default (`approach =
"auto"`), `risks` estimates the most efficient valid model that
converges; in more numerically challenging cases, it defaults to
computationally less efficient models while ensuring that the user
receives a valid result. Whenever data are sufficient to obtain odds
ratios from logistic models, `risks` is designed to successfully return
risk ratios and risk differences.

The following models are implemented in `risks`:

| \#<sup>1</sup> | `approach =` | RR  | RD  | Model                                                                                                               | Reference                                                                                                                                                                                                                               |
| -------------- | ------------ | --- | --- | ------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1              | `glm`        | Yes | Yes | Binomial model with a log or identity link                                                                          | Wacholder S. Binomial regression in GLIM: Estimating risk ratios and risk differences. [Am J Epidemiol 1986;123:174-184](https://www.ncbi.nlm.nih.gov/pubmed/3509965).                                                                  |
| 2              | `glm_start`  | Yes | Yes | Binomial model with a log or identity link, convergence-assisted by starting values from Poisson model              | Spiegelman D, Hertzmark E. Easy SAS calculations for risk or prevalence ratios and differences. [Am J Epidemiol 2005;162:199-200](https://www.ncbi.nlm.nih.gov/pubmed/15987728).                                                        |
| 3              | `glm_cem`    | Yes | —   | Binomial model with log-link fitted via combinatorial expectation maximization instead of Fisher scoring            | Donoghoe MW, Marschner IC. logbin: An R Package for Relative Risk Regression Using the Log-Binomial Model. [J Stat Softw 2018;86(9)](http://dx.doi.org/10.18637/jss.v086.i09).                                                          |
| 3              | `glm_cem`    | —   | Yes | Additive binomial model (identity link) fitted via combinatorial expectation maximization instead of Fisher scoring | Donoghoe MW, Marschner IC. Stable computational methods for additive binomial models with application to adjusted risk differences. [Comput Stat Data Anal 2014;80:184-96](https://doi.org/10.1016/j.csda.2014.06.019).                 |
| 4              | `margstd`    | Yes | Yes | Marginally standardized estimates using binomial model with a logit link (logistic model)                           | Localio AR, Margolis DJ, Berlin JA. Relative risks and confidence intervals were easily computed indirectly from multivariable logistic regression. [J Clin Epidemiol 2007;60(9):874-82](https://www.ncbi.nlm.nih.gov/pubmed/17689803). |
| –              | `robpoisson` | Yes | Yes | Log-linear (Poisson) model with robust/sandwich/empirical standard errors                                           | Zou G. A modified Poisson regression approach to prospective studies with binary data. [Am J Epidemiol 2004;159(7):702-6](https://www.ncbi.nlm.nih.gov/pubmed/15033648)                                                                 |
| –              | `logistic`   | No  | —   | Binomial model with logit link (*i.e.*, the logistic model), returning odds ratios                                  | Included for comparison purposes only.                                                                                                                                                                                                  |

<sup>1</sup> Indicates the priority with which the default modelling
strategy (`approach = "auto"`) attempts model fitting.

  
Which model was fitted is always indicated in the first line of the
output of `summary(...)` and in the `model` column of `tidy(...)`. In
methods sections of manuscripts, the approach can be described in detail
as follows:

> Risk ratios \[or risk differences\] were obtained via \[method listed
> in the first line of `summary(...)`\] using the `risks` R package
> (reference to this package and/or the article listed in the column
> “reference”).

For example:

> Risk ratios were obtained from binomial models with a log link,
> convergence-assisted by Poisson models (ref. Spiegelman and Hertzmark,
> AJE 2005), using the `risks` R package (reference).

# Advanced usage

## Model choice

By default, automatic model fitting according to the priority listed in
the table above is attempted. Alternatively, any of the options listed
under `approach =` in the table can be requested directly. However,
unlike with `approach = "auto"` (the default), the selected model may
not converge.

Selecting a binomial model with starting values from the Poisson model:

``` r
estimate_risk(formula = death ~ stage + receptor, data = dat, approach = "glm_start")
```

  
However, the binomial model without starting values does not converge:

``` r
estimate_risk(formula = death ~ stage + receptor, data = dat, approach = "glm")
```

## Marginal standardization

Marginal standardization, `approach = "margstd"`, makes use of the good
convergence properties of the logit link, which is also guaranteed to
result in probabilities within (0; 1). After fitting a logistic models,
predicted probabilities for each observation are obtained for the levels
of the exposure variable. Risk ratios and risk differences are
calculated by contrasting the predicted probabilities as ratios or
differences. Standard errors and confidence intervals are obtained via
bootstrapping the entire procedure. Standardization can only be done
over one exposure variable, and thus no cofficients are estimated for
other variables in the model. In addition, in order to derive contrasts,
continuous exposures are evaluated at discrete levels.

  - By default, the first categorical right-hand variable in the model
    formula will be assumed to be the exposure. The variable types
    `logical`, `factor`, and `character` are taken to represent
    categorical variables, as are variables of the type `numeric` with
    only two levels (e.g., `0` and `1`).
  - Using `variable =`, the user can specify a different variable.
  - Using `at =`, levels for contrasts and the order of levels can be
    specified. The first level is used as the reference with a risk
    ratio of 1 or a risk difference of 0. The option `at =` is required
    for continuous variables or `numeric` variables with more than two
    levels. A warning will be shown for continuous values if the
    requested levels exceed the range of data (extrapolation).
  - For models fitted via `approach = "margstd"`, standard
    errors/confidence intervals are obtained via bootstrapping. The
    default are 100 bootstrap repeats. This number should be increased
    to \>500 for more precise estimates. Use the option `bootrepeats =`
    in `summary()`, `tidy()`, or `confint()`.

We fit the same risk difference model as in section 2:

``` r
fit_margstd <- estimate_risk(formula = death ~ stage + receptor, data = dat, 
                             estimate = "rd", approach = "margstd")
summary(fit_margstd, bootrepeats = 500)
```

Consistent with earlier results, we observed that women with stage III
breast cancer have a 57 percentage points higher risk of death
(bootstrapped 95% confidence interval, 35 to 76 percentage points),
adjusting for hormone receptor status.

Note that coefficients and standard errors are only estimated for the
exposure variable. Model fit characteristics and predicted values stem
directly from the underlying logistic model.

Requesting a different exposure variable:

``` r
fit_margstd2 <- estimate_risk(formula = death ~ stage + receptor, data = dat, 
                             estimate = "rd", approach = "margstd", variable = "receptor")
summary(fit_margstd2, bootrepeats = 500)
```

## Model comparisons

With `approach = "all"`, all model types listed in the tables are
fitted. The fitted object, *e.g.*, `fit`, is one of the converged
models. A summary of the convergence status of all models is displayed
at the beginning of `summary(fit)`:

``` r
fit_all <- estimate_risk(formula = death ~ stage + receptor, data = dat, 
                         estimate = "rd", approach = "all")
summary(fit_all)
```

  
Individual models can be accessed as `fit$all_models[[1]]` through
`fit$all_models[[6]]` (or `[[7]]` if fitting a risk ratio model).
`tidy()` shows coefficients and confidence intervals from all models
that converged:

``` r
tidy(fit_all)
```

## Prediction

  - Checking maximum predicted probabilities (may be out-of-range for
    Poisson)

## Weights—TBD

## Clustered observations—TBD
