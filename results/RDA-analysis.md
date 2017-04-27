---
title: "dbRDA analysis"
output: 
  html_notebook:
    toc: true
---





This document serves to re-assess Sajeewa's dbRDA analysis with the repeat
lengths that have been modified to avoid inconsistent allele calls. 

Modification in this case means that a tiny amount has been added or subtracted
to the repeat length to ensure that the alleles are all unique after division.

Below is my attempt at reproduction of Sajeewa's analysis.

## Packages and Data


```r
library('tidyverse')
```

```
## Loading tidyverse: ggplot2
## Loading tidyverse: tibble
## Loading tidyverse: tidyr
## Loading tidyverse: readr
## Loading tidyverse: purrr
## Loading tidyverse: dplyr
```

```
## Conflicts with tidy packages ----------------------------------------------
```

```
## filter(): dplyr, stats
## lag():    dplyr, stats
```

```r
library('poppr')
```

```
## Loading required package: adegenet
```

```
## Loading required package: ade4
```

```
## 
##    /// adegenet 2.1.0 is loaded ////////////
## 
##    > overview: '?adegenet'
##    > tutorials/doc/questions: 'adegenetWeb()' 
##    > bug reports/feature requests: adegenetIssues()
```

```
## This is poppr version 2.4.1.99.2. To get started, type package?poppr
## OMP parallel support: available
## 
## This version of poppr is under development.
## If you find any bugs, please report them at https://github.com/grunwaldlab/poppr/issues
```

```r
library('vegan')
```

```
## Loading required package: permute
```

```
## Loading required package: lattice
```

```
## This is vegan 2.4-3
```

```
## 
## Attaching package: 'vegan'
```

```
## The following object is masked from 'package:ade4':
## 
##     cca
```

```r
library('ggrepel')
```


So, here Sajeewa clone-corrected the data according to the combination of
Host, Source (aka Field), Region (aka State/Country), and Year. 



```r
load("data/sclerotinia_16_loci.rda")
setPop(dat)   <- ~Host/Source/Region/Year
setPop(dat11) <- ~Host/Source/Region/Year
dat11cc <- clonecorrect(dat11, ~Host/Source/Region/Year, keep = 1:4)
dat16cc <- clonecorrect(dat, ~Host/Source/Region/Year, keep = 1:4)
dat11cc
```

```
## 
## This is a genclone object
## -------------------------
## Genotype information:
## 
##    165 original multilocus genotypes 
##    318 haploid individuals
##     11 codominant loci
## 
## Population information:
## 
##      5 strata - MCG, Region, Source, Year, Host
##    128 populations defined - 
## GH_unk_NE_2003, GH_unk_NY_2003, G122_wmn_MN_2003, ..., unk_pmc_ND_2010, unk_wlc_ND_2010, unk_flds_France_2012
```

```r
dat16cc
```

```
## 
## This is a genclone object
## -------------------------
## Genotype information:
## 
##    215 original multilocus genotypes 
##    342 haploid individuals
##     16 codominant loci
## 
## Population information:
## 
##      5 strata - MCG, Region, Source, Year, Host
##    128 populations defined - 
## GH_unk_NE_2003, GH_unk_NY_2003, G122_wmn_MN_2003, ..., unk_pmc_ND_2010, unk_wlc_ND_2010, unk_flds_France_2012
```

```r
# Asserting that nothing messed up with the metadata.
stopifnot(identical(indNames(dat11cc), other(dat11cc)$meta$Isolate))
stopifnot(identical(indNames(dat16cc), other(dat16cc)$meta$Isolate))

# function to gather Environmental variables, but averaging Severity
# (aka Straw Test, Virulence). This function is necessary because the
# data tends to randomly shuffle when being processed vai dplyr functions
# for better or for worse. ¯\_(ツ)_/¯
# 
# @param DAT the full data set
# @param CC The clone-corrected data set
# @param wmn should the "Source" column be converted to a binary wmn
#   factor? When this is true, everything that is not wmn will be converted
#   to "other"
# 
# @return A data frame containing Severity, Host, Source, Year, Region, and MLG
makeENV <- function(DAT, CC, wmn = FALSE){
  # Creating the data frame with severity
  META   <- select(other(DAT)$meta, -Isolate)
  STRATA <- select(strata(DAT), -MCG)
  STRATA <- if (wmn) mutate(STRATA, Source = ifelse(Source == "wmn", "wmn", "other")) else STRATA
  MLL    <- data.frame(MLG = mll(DAT))
  sev    <- bind_cols(META, STRATA, MLL) %>%
    group_by(Host, Source, Year, Region, MLG) %>% 
    summarize(Severity = mean(Severity)) %>% # Get mean severity per MLG
    ungroup()
  # Ensuring the data is in the correct order
  META   <- select(other(CC)$meta, -Isolate)
  STRATA <- select(strata(CC), -MCG)
  MLL    <- data.frame(MLG = mll(CC))
  bind_cols(META, STRATA, MLL) %>% 
    left_join(sev)
}

# Get environmental variables for 11 loci
ENV11 <- makeENV(dat11, dat11cc)
```

```
## Joining, by = c("Severity", "Region", "Source", "Year", "Host", "MLG")
```

```r
ENV11
```

```
## # A tibble: 318 × 6
##    Severity Region Source   Year   Host   MLG
##       <dbl> <fctr> <fctr> <fctr> <fctr> <int>
## 1       3.9     NE    unk   2003     GH   165
## 2       5.4     NE    unk   2003     GH   164
## 3       6.3     NY    unk   2003     GH    42
## 4       4.4     MN    wmn   2003   G122   165
## 5       4.7     MN    wmn   2003  Beryl   165
## 6       6.1     MI    wmn   2003  Beryl    30
## 7       5.5     MI    wmn   2003  Beryl    25
## 8       5.0     MI    wmn   2003  Beryl    40
## 9       5.2     MI    wmn   2003  Bunsi    27
## 10      5.3     MI    wmn   2003  Bunsi    20
## # ... with 308 more rows
```

```r
stopifnot(identical(ENV11$MLG, mll(dat11cc)))
ENV11 <- select(ENV11, -MLG)

# Get environmental variables for 16 loci
ENV16 <- makeENV(dat, dat16cc)
```

```
## Joining, by = c("Severity", "Region", "Source", "Year", "Host", "MLG")
```

```r
ENV16
```

```
## # A tibble: 342 × 6
##    Severity Region Source   Year   Host   MLG
##       <dbl> <fctr> <fctr> <fctr> <fctr> <int>
## 1       3.9     NE    unk   2003     GH   215
## 2       5.4     NE    unk   2003     GH   214
## 3       6.3     NY    unk   2003     GH    62
## 4       4.4     MN    wmn   2003   G122   211
## 5       4.7     MN    wmn   2003  Beryl   215
## 6       6.1     MI    wmn   2003  Beryl    17
## 7       5.5     MI    wmn   2003  Beryl    41
## 8       5.0     MI    wmn   2003  Beryl    49
## 9       5.2     MI    wmn   2003  Bunsi    15
## 10      5.3     MI    wmn   2003  Bunsi    28
## # ... with 332 more rows
```

```r
stopifnot(identical(ENV16$MLG, mll(dat16cc)))
ENV16 <- select(ENV16, -MLG)
```


## Functions to tie everything together

The analysis has a couple of steps

1. Model choice. Since we don't want to overparamaterize the model, we will use 
*vegan*'s built in model choice function `ordistep()` to to forward-backward 
selection of the appropriate model to fit our data. Yes, some (particularly 
Bayesians) believe that model choice is evil and the proper analysis will find 
the true pattern in the underlying data, but hey, I'm just trying to make sure 
that I'm not making overconfident judgements.

2. Plot the results of the model choice using ggplot2, overlaying the top 8 
explanitory vectors


```r
# model choice for dbrda/capscale
# 
# @param bdist a distance matrix (here, we use Bruvo's distances, which is why
#        it's called 'bdist').
# @param ENV a data frame of environmental variables with the same number of
#        observatios as bdist
# @param CHOOSER the name of the function to perform the forwards-backwards selection
# @param ... arguments to be passed on to CHOOSER
# 
# @return a capscale object
choose_dbrda <- function(bdist, ENV, CHOOSER = "ordistep", ...){
  # Step 1: create null model
  mod0  <- capscale(bdist ~ 1, data = ENV, add = TRUE)
  # Step 2: create full model (all variables in ENV)
  mod1  <- capscale(bdist ~ ., data = ENV, add = TRUE)
  # Step 3: Run forward-backwards selection (this can take a while)
  CHOOSER   <- match.fun(CHOOSER)
  the_model <- CHOOSER(mod0, scope = formula(mod1), ...)
  # Return the best model with the anova results
  return(the_model)
}

# This was ganked from https://github.com/gavinsimpson/ggvegan/blob/59d233977a5b2d15d4de150b782fb1794aa1de8b/R/utils.R
# 
# @title Scale Vectors to Data
# @description Scale vector arrows to \code{fill} proportion of the data.
# @param arrows a two-column matrix-like object containing coordinates for the arrows/vectors on x and y axes.
# @param data a two-column matrix-like object containing coordinates of the data on the x and y axes.
# @param at numeric vector of length 2; location of the origin of the arrows.
# @param fill numeric; what proportion of the range of the data to fill
# @return a numeric multiplier that will scale the arrows
# @author Gavin L. Simpson
arrowMul <- function(arrows, data, at = c(0, 0), fill = 0.75) {
    u <- c(range(data[,1], range(data[,2])))
    u <- u - rep(at, each = 2)
    r <- c(range(arrows[, 1], na.rm = TRUE), range(arrows[, 2], na.rm = TRUE))
    rev <- sign(diff(u))[-2]
    if (rev[1] < 0)
        u[1:2] <- u[2:1]
    if (rev[2] < 0)
        u[3:4] <- u[4:3]
    u <- u/r
    u <- u[is.finite(u) & u > 0]
    fill * min(u)
}
# Plotting the dbRDA results
# 
# @param db a capscale object
# @return a ggplot2 object from the scores 
plot_dbrda <- function(db, arrows = 10){
  dbsum     <- scores(db, display = c("cn", "bp", "sites"), scaling = "sites")
  Centroids <- as.data.frame(dbsum$centroids)
  Centroids <- rownames_to_column(Centroids, var = "cent_type")
  Centroids <- mutate_(Centroids, .dots = list(Length = ~sqrt(CAP1^2 * CAP2^2)))
  # Centroids
  SampleCentroids <- rownames_to_column(data.frame(dbsum$sites), var = "isolate_names")
  
  terms   <- paste0("(", paste(attr(db$terms, "term.labels"), collapse = "|"), ")")
  mul     <- arrowMul(dbsum$biplot[, 1:2], dbsum$sites)
  Arrows  <- data.frame(dbsum$biplot * mul)
  Arrows  <- rownames_to_column(Arrows, var = "class")
  # Making the classes presentable
  Arrows$class <- gsub(terms, "\\1: ", Arrows$class)
  Arrows$class <- gsub(": $", "", Arrows$class)
  Arrows$class <- gsub("unk", "unknown", Arrows$class)
  Arrows  <- mutate_(Arrows, .dots = list(Length = ~sqrt(CAP1^2 * CAP2^2)))
  Arrows  <- arrange(Arrows, Length)
  Arrows  <- top_n(Arrows, arrows)
  ggplot(Centroids, aes(x = CAP1, y = CAP2))+
    geom_point(data = SampleCentroids, 
               # alpha = 1/2, 
               # fill = "white",
               # fill = "dark orange", 
               color = "grey45", 
               size = 2.5, 
               pch = 21)+
    coord_cartesian() +
    geom_segment(aes(x = 0, xend = CAP1, 
                     y = 0, yend = CAP2),
                 arrow = arrow(length = unit(0.3, "cm")), 
                 data = Arrows
                 ) + 
    geom_label_repel(aes(x = CAP1, y = CAP2, label = class), 
                     point.padding = unit(0.5, "lines"),
                     segment.color = "grey25",
                     data = Arrows) +
    ylab("Eigenvalue 1") +
    xlab("Eigenvalue 2")
}
```


# Calculations

Here come the calculations. Note, since I am verifying that we get the same
results from the 16 loci as we do for the 11 loci, we have to do this twice. 

1. calculate Bruvo's genetic distance.
2. model choice


```r
# 11 loci
dat11cc.bruvo <- dat11cc %>% bruvo.dist(replen = other(.)$REPLEN)
cap11cc       <- choose_dbrda(dat11cc.bruvo, ENV = ENV11, CHOOSER = "ordistep")
```

```
## 
## Start: bdist ~ 1 
## 
##            Df    AIC      F Pr(>F)   
## + Severity  1 1787.5 1.3622  0.005 **
## + Year      7 1790.8 1.4330  0.005 **
## + Region   13 1793.2 1.4902  0.005 **
## + Source   24 1808.2 1.0700  0.005 **
## + Host     26 1810.2 1.0581  0.005 **
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Step: bdist ~ Severity 
## 
##            Df    AIC      F Pr(>F)   
## - Severity  1 1786.9 1.3622  0.005 **
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
##          Df    AIC      F Pr(>F)   
## + Year    7 1791.4 1.4253  0.005 **
## + Region 13 1794.0 1.4790  0.005 **
## + Source 24 1808.9 1.0624  0.005 **
## + Host   26 1810.7 1.0603  0.005 **
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Step: bdist ~ Severity + Year 
## 
##            Df    AIC      F Pr(>F)   
## - Year      7 1787.5 1.4253  0.005 **
## - Severity  1 1790.8 1.3112  0.005 **
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
##          Df    AIC      F Pr(>F)   
## + Region 13 1798.6 1.3854  0.005 **
## + Source 23 1811.6 1.0521  0.005 **
## + Host   26 1813.9 1.0578  0.005 **
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Step: bdist ~ Severity + Year + Region 
## 
##            Df    AIC      F Pr(>F)   
## - Severity  1 1798.0 1.2191  0.040 * 
## - Region   13 1791.4 1.3854  0.005 **
## - Year      7 1794.0 1.2581  0.005 **
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
##          Df    AIC      F Pr(>F)   
## + Host   26 1819.3 1.0739  0.005 **
## + Source 23 1818.3 1.0261  0.045 * 
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Step: bdist ~ Severity + Year + Region + Host 
## 
##            Df    AIC      F Pr(>F)   
## - Severity  1 1818.7 1.1541  0.060 . 
## - Host     26 1798.6 1.0739  0.005 **
## - Region   13 1813.9 1.3888  0.005 **
## - Year      7 1816.2 1.3362  0.005 **
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
##          Df    AIC      F Pr(>F)
## + Source 21 1835.5 1.0018   0.52
```

```r
# 16 loci
dat16cc.bruvo <- dat16cc %>% bruvo.dist(replen = other(.)$REPLEN)
cap16cc       <- choose_dbrda(dat16cc.bruvo, ENV = ENV16, CHOOSER = "ordistep")
```

```
## 
## Start: bdist ~ 1 
## 
##            Df    AIC      F Pr(>F)   
## + Year      7 1928.0 1.4098  0.005 **
## + Region   13 1930.8 1.4593  0.005 **
## + Source   24 1944.1 1.1237  0.005 **
## + Host     26 1946.7 1.0853  0.005 **
## + Severity  1 1924.7 1.3085  0.015 * 
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Step: bdist ~ Year 
## 
##        Df  AIC      F Pr(>F)   
## - Year  7 1924 1.4098  0.005 **
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
##            Df    AIC      F Pr(>F)   
## + Severity  1 1928.8 1.2208  0.005 **
## + Region   13 1935.3 1.3893  0.005 **
## + Source   23 1947.1 1.1100  0.005 **
## + Host     26 1950.1 1.0827  0.005 **
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Step: bdist ~ Year + Severity 
## 
##            Df    AIC      F Pr(>F)   
## - Severity  1 1928.0 1.2208  0.030 * 
## - Year      7 1924.7 1.3967  0.005 **
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
##          Df    AIC      F Pr(>F)   
## + Region 13 1936.1 1.3858  0.005 **
## + Source 23 1947.9 1.1034  0.005 **
## + Host   26 1950.8 1.0813  0.005 **
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Step: bdist ~ Year + Severity + Region 
## 
##            Df    AIC      F Pr(>F)   
## - Severity  1 1935.3 1.1830  0.040 * 
## - Region   13 1928.8 1.3858  0.005 **
## - Year      7 1931.5 1.2855  0.005 **
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
##          Df    AIC      F Pr(>F)   
## + Host   26 1956.7 1.0878  0.005 **
## + Source 23 1955.3 1.0524  0.010 **
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Step: bdist ~ Year + Severity + Region + Host 
## 
##            Df    AIC      F Pr(>F)   
## - Severity  1 1955.9 1.0817  0.125   
## - Host     26 1936.1 1.0878  0.005 **
## - Region   13 1950.8 1.3745  0.005 **
## - Year      7 1953.6 1.3645  0.005 **
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Step: bdist ~ Year + Region + Host 
## 
##            Df    AIC      F Pr(>F)
## + Severity  1 1956.7 1.0817  0.115
## + Source   21 1972.3 1.0140  0.360
## 
##          Df    AIC      F Pr(>F)   
## - Host   26 1935.3 1.0918  0.005 **
## - Region 13 1950.1 1.3829  0.005 **
## - Year    7 1952.8 1.3693  0.005 **
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
##            Df    AIC      F Pr(>F)  
## + Severity  1 1956.7 1.0817  0.085 .
## + Source   21 1972.3 1.0140  0.380  
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
```

# Plot the results

```r
cap11cc
```

```
## Call: capscale(formula = bdist ~ Severity + Year + Region + Host,
## data = ENV, add = TRUE)
## 
##               Inertia Proportion Rank
## Total         274.797      1.000     
## Constrained    48.358      0.176   47
## Unconstrained 226.439      0.824  270
## Inertia is Lingoes adjusted squared Bruvo distance 
## 
## Eigenvalues for constrained axes:
##  CAP1  CAP2  CAP3  CAP4  CAP5  CAP6  CAP7  CAP8  CAP9 CAP10 CAP11 CAP12 
## 5.304 3.249 2.473 2.199 1.712 1.422 1.308 1.066 1.036 1.013 0.895 0.879 
## CAP13 CAP14 CAP15 CAP16 CAP17 CAP18 CAP19 CAP20 CAP21 CAP22 CAP23 CAP24 
## 0.863 0.833 0.807 0.793 0.787 0.780 0.771 0.769 0.764 0.761 0.759 0.759 
## CAP25 CAP26 CAP27 CAP28 CAP29 CAP30 CAP31 CAP32 CAP33 CAP34 CAP35 CAP36 
## 0.757 0.756 0.755 0.754 0.751 0.750 0.749 0.743 0.742 0.734 0.729 0.728 
## CAP37 CAP38 CAP39 CAP40 CAP41 CAP42 CAP43 CAP44 CAP45 CAP46 CAP47 
## 0.720 0.713 0.711 0.709 0.703 0.682 0.668 0.662 0.645 0.634 0.563 
## 
## Eigenvalues for unconstrained axes:
##  MDS1  MDS2  MDS3  MDS4  MDS5  MDS6  MDS7  MDS8 
## 8.202 4.462 3.542 3.175 2.832 2.324 2.142 2.097 
## (Showed only 8 of all 270 unconstrained eigenvalues)
## 
## Constant added to distances: 0.7593611
```

```r
cap11cc$anova
```

```
##            Df    AIC      F Pr(>F)   
## + Severity  1 1787.5 1.3622  0.005 **
## + Year      7 1791.4 1.4253  0.005 **
## + Region   13 1798.6 1.3854  0.005 **
## + Host     26 1819.3 1.0739  0.005 **
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
```

```r
set.seed(99)
plot_dbrda(cap11cc) + 
  theme_classic(base_size = 16, base_family = "Helvetica") + 
  theme(axis.text = element_text(color = "black")) +
  theme(aspect.ratio = 1)
```

```
## Selecting by Length
```

![plot of chunk resultplot](./figures/RDA-analysis///resultplot-1.png)

```r
ggsave(filename = "results/figures/publication/Figure7Z.svg", width = 88, height = 88, units = "mm", scale = 1.5)
cap16cc
```

```
## Call: capscale(formula = bdist ~ Year + Region + Host, data = ENV,
## add = TRUE)
## 
##                Inertia Proportion Rank
## Total         276.6746     1.0000     
## Constrained    44.5875     0.1612   46
## Unconstrained 232.0871     0.8388  295
## Inertia is Lingoes adjusted squared Bruvo distance 
## 
## Eigenvalues for constrained axes:
##  CAP1  CAP2  CAP3  CAP4  CAP5  CAP6  CAP7  CAP8  CAP9 CAP10 CAP11 CAP12 
## 5.227 2.587 2.087 1.885 1.612 1.399 1.104 1.025 0.983 0.913 0.903 0.875 
## CAP13 CAP14 CAP15 CAP16 CAP17 CAP18 CAP19 CAP20 CAP21 CAP22 CAP23 CAP24 
## 0.842 0.811 0.785 0.763 0.748 0.741 0.737 0.733 0.724 0.720 0.718 0.717 
## CAP25 CAP26 CAP27 CAP28 CAP29 CAP30 CAP31 CAP32 CAP33 CAP34 CAP35 CAP36 
## 0.717 0.717 0.715 0.713 0.711 0.709 0.706 0.704 0.702 0.701 0.695 0.690 
## CAP37 CAP38 CAP39 CAP40 CAP41 CAP42 CAP43 CAP44 CAP45 CAP46 
## 0.688 0.682 0.680 0.677 0.667 0.658 0.634 0.627 0.596 0.558 
## 
## Eigenvalues for unconstrained axes:
##  MDS1  MDS2  MDS3  MDS4  MDS5  MDS6  MDS7  MDS8 
## 7.342 3.739 3.258 3.005 2.249 2.056 1.816 1.720 
## (Showed only 8 of all 295 unconstrained eigenvalues)
## 
## Constant added to distances: 0.7174933
```

```r
cap16cc$anova
```

```
##            Df    AIC      F Pr(>F)   
## + Year      7 1928.0 1.4098  0.005 **
## + Severity  1 1928.8 1.2208  0.005 **
## + Region   13 1936.1 1.3858  0.005 **
## + Host     26 1956.7 1.0878  0.005 **
## - Severity  1 1955.9 1.0817  0.125   
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
```

```r
plot_dbrda(cap16cc) +
  theme_classic(base_size = 16, base_family = "Helvetica") + 
  theme(axis.text = element_text(color = "black")) +
  theme(aspect.ratio = 1)
```

```
## Selecting by Length
```

![plot of chunk resultplot](./figures/RDA-analysis///resultplot-2.png)

## Variance partitioning

First, we want to examine how the entire data fits with the model. We will use
the alleles as the representative of the entire data.


```r
dat11raw <- genind2df(dat11cc, usepop = FALSE) %>% mutate_all(funs(as.integer(.)))
dat16raw <- genind2df(dat16cc, usepop = FALSE) %>% mutate_all(funs(as.integer(.)))
vp11     <- varpart(dat11cc.bruvo, ~Severity + Year + Region + Host, data = ENV11, comm = dat11raw, add = TRUE)
plot(vp11, Xnames = c("Full Model", "No Model"))
```

![plot of chunk unnamed-chunk-1](./figures/RDA-analysis///unnamed-chunk-1-1.png)

```r
try(vp16     <- varpart(dat16cc.bruvo, ~Year + Region + Host, data = ENV16, comm = dat16raw, add = TRUE))
```


```r
vp11 <- varpart(dat11cc.bruvo, ~Severity, ~Year, ~Region, ~Host, data = ENV11, add = TRUE)
vp16 <- varpart(dat16cc.bruvo, ~Year, ~Region, ~Host, data = ENV16, add = TRUE)

plot(vp11, digits = 2, Xnames = c("Severity", "Year", "Region", "Host"))
```

![plot of chunk unnamed-chunk-2](./figures/RDA-analysis///unnamed-chunk-2-1.png)

```r
plot(vp16, digits = 2, Xnames = c("Year", "Region", "Host"))
```

![plot of chunk unnamed-chunk-2](./figures/RDA-analysis///unnamed-chunk-2-2.png)


```r
wmn11 <- seppop(dat11, ~Source)$wmn %>% setPop(~Region)
wmn11_sub <- popsub(wmn11, blacklist = character(0))
min_span_net <- bruvo.msn(wmn11_sub, replen = other(wmn11)$REPLEN, add = TRUE, loss = TRUE, showplot = FALSE, include.ties = TRUE)
set.seed(67)
plot_poppr_msn(wmn11,
               min_span_net,
               inds = "none",
               mlg = TRUE,
               gadj = 6,
               nodelab = 100,
               nodebase = 1.15,
               palette = funky,
               cutoff = (0.5/11)*3,
               quantiles = FALSE,
               beforecut = TRUE,
               layfun = igraph::layout_with_lgl)
```

![plot of chunk unnamed-chunk-3](./figures/RDA-analysis///unnamed-chunk-3-1.png)
