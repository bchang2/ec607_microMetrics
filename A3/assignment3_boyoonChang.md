---
title: "Micro-metrics, [Glen Waddell](https://glenwaddell.com)"
author: Boyoon Chang
date: "Winter 2020"
#date: "<br>27 March 2021"
header-includes:
  - \usepackage{mathtools}
  - \DeclarePairedDelimiter\floor{\lfloor}{\rfloor}
  - \usepackage{amssymb}
output: 
  html_document: 
    theme: flatly
    highlight: tango
    toc_float:
      collapsed: yes
      smooth_scroll: yes
    keep_md: true
---




> Can I propose something like each of is having a master file... like this one... that we each use to collect our thoughts as we work through the next ten weeks? 



# {.tabset .tabset-fade .tabset-pills}

## A3 - The visualization of treatment variation

> Due date: 26 January 2021 

First, simulate a DGP in which there is some treatment to be evaluated. Do this in a way that allows you to control whether it falls randomly on individuals or systematically with some observable characteristic. 

Second, consider how one might display the existing variation in treatment associated with individual characteristics. A nice visualization---something you could display on your webpage when you're in the job market. 

#### Framework of the Exercise and Brief Conclusion

First, I will introduce selection bias into the model. Then based upon the selection bias, I compare the average treatment effect estimate under the two scenarios, when treatment is random and when treatment is not random. The initial finding is that this particular selection bias that I introduced does not contribute to the inconsistency of the estimate of the average treatment effect. See the steps and specific results below. 

#### Results

- Introduce selection bias into the model: This implies that selection into treatment is non-random. 

The data generating process below suggests how the wage is determined. Motivated by Griliches (1976), I constructed a hypothetical model where the wage is determined by some function of IQ, experience, education, age and gender. Treatment would be a job training program and we are interested in looking at the average treatment effect of the program on wage. 

We think of two cases below based on randomness of the treatment. In particular, selection bias is introduced that is dependent on gender when the selection into treatment is non-random. That is, if male, subjects are selected into treatment when $\frac{IQ}{100} + 4 \cdot EXP+ e$ is greater than 3, whereas if female, subjects are selected into treatment when $\frac{IQ}{120}  + \frac{EDU}{5} - \frac{AGE}{45} + e$ is greater than 3. In other words, if male, selection into the job training program is positively related to $IQ$ and the $EXP$ which implies that as their IQ and experiences are high, they are more likely to be treated. If female, selection into the job training program is positively related to $IQ$ and $EDU$ but negatively correlated with $AGE$. This implies that they are more likely to be selected into treatment if their age is relatively low while their IQ and number of education are high. 



```r
# one_iter = sim_iter(300)
dgp_iter = function(n){
  tibble(
    i = 1:n,
    u = runif(n, -10, 10), 
    v = runif(n, -5,5),
    e = runif(n, -1,1),
    W = rnorm(n, 3,2),
    IQ = round(rnorm(n, mean = 120, sd =20),digits = 0), 
    EXP = rtruncnorm(rnorm(n, mean = 3, sd = 2)),
    EDU = sample(c(12:21), n, replace = TRUE),
    GENDER = sample(c("Male", "Female"), n , replace = TRUE),
    MALE = ifelse(GENDER == "Male", 1, 0),
    AGE = sample(c(24:80), n, replace = TRUE),
    X = 0.02*IQ + 0.05* EXP + 0.06*EDU + 
        0.25*ifelse(GENDER=="Male",1,0) + 
        0.07*AGE, 
    y0 = X + u,
    # suppose w being ITE an the arbitrary treatment effect imposed
    y1 = y0 + W + v, 
    t = y1 - y0, # 
    # dummy variable (random and non-random)
    d_r = sample(c(1,0), n, replace = TRUE),
    d_nr = ifelse( ifelse(MALE ==1, 
                   IQ / 100 + e + 4 * EXP ,
                   IQ / 120 + e + EDU/5 - AGE/45 ) > 3 , 1 , 0) %>% as.numeric(),
    y_r = y0 + d_r*t,
    y_nr = y0+ d_nr*t
  )
}
one_iter = dgp_iter(300)
prob_male_treat = one_iter %>% filter(MALE ==1 & (IQ / 100 + e + 4 * EXP > 3)) %>% summarize(counts = n()) /nrow(one_iter %>% filter(MALE==1)) %>% as.vector()
# print(prob_male_treat)
prob_female_treat = one_iter %>% filter(MALE ==0 & (IQ / 120 + e + EDU/5 - AGE/45 > 3)) %>% summarize(counts = n()) /nrow(one_iter %>% filter(MALE==0))%>% as.numeric()
# print(prob_female_treat)
```




```r
# plot
library(ggthemes)
one_iter$d_nr = factor(one_iter$d_nr, levels = c(0,1), labels = c("Control", "Treated"))
# plot treatment assignment and selection basis of female (female only)
ggplot(one_iter %>% filter(GENDER == "Female")) + 
  geom_boxplot(aes(x = IQ / 120 + e + EDU/5 - AGE/45 , y = d_nr, color = d_nr)) + 
  theme_light() + ylab("Group") + xlab("Selection Basis of Female")+
  theme(legend.title = element_blank())+scale_colour_brewer(palette="Set1")+ 
  geom_vline(xintercept = 3, linetype = "dashed" )+
  labs(title = "Treatment and Selection Basis of Female (Female Only)")
```

<img src="assignment3_boyoonChang_files/figure-html/a3plot-1.png" style="display: block; margin: auto;" />

```r
# plot treatment assignment and selection basis of male ( male only)
ggplot(one_iter %>% filter(GENDER == "Male")) + 
  geom_boxplot(aes(x = IQ / 100 + e + 4 * EXP, y = d_nr, color = d_nr))+ 
  theme_light() + ylab("Group") + xlab("Selection Basis of Male")+
  theme(legend.title = element_blank())+scale_colour_brewer(palette="Set1") + 
  geom_vline(xintercept = 3, linetype = "dashed" )+
  labs(title = "Treatment and Selection Basis of Male (Male Only)")
```

<img src="assignment3_boyoonChang_files/figure-html/a3plot-2.png" style="display: block; margin: auto;" />

```r
# plot treatment and IQ 
mean <- one_iter %>% group_by(d_nr) %>% summarise(mean=mean(IQ)) %>% ungroup()
ggplot(one_iter) + 
  geom_density(aes(x = IQ, color = d_nr)) + ylab("Frequency")+ theme_light() + 
  theme(legend.title = element_blank())+scale_colour_brewer(palette="Set1") + 
  geom_vline(data = mean, aes(xintercept=mean, color = d_nr), linetype = "dashed")
```

<img src="assignment3_boyoonChang_files/figure-html/a3plot-3.png" style="display: block; margin: auto;" />

```r
# plot treatment and EXP
mean <- one_iter %>% group_by(d_nr) %>% summarise(mean=mean(EXP)) %>% ungroup()
ggplot(one_iter) + 
  geom_density(aes(x = EXP, color = d_nr)) + ylab("Frequency")+ theme_light() + 
  theme(legend.title = element_blank())+scale_colour_brewer(palette="Set1") + 
  geom_vline(data = mean, aes(xintercept=mean, color = d_nr), linetype = "dashed")
```

<img src="assignment3_boyoonChang_files/figure-html/a3plot-4.png" style="display: block; margin: auto;" />

```r
# plot treatment and EDU
mean <- one_iter %>% group_by(d_nr) %>% summarise(mean=mean(EDU)) %>% ungroup()
ggplot(one_iter) + 
  geom_density(aes(x = EDU, color = d_nr)) + ylab("Frequency")+ theme_light() + 
  theme(legend.title = element_blank())+scale_colour_brewer(palette="Set1") + 
  geom_vline(data = mean, aes(xintercept=mean, color = d_nr), linetype = "dashed")
```

<img src="assignment3_boyoonChang_files/figure-html/a3plot-5.png" style="display: block; margin: auto;" />

```r
# plot treatment and AGE
mean <- one_iter %>% group_by(d_nr) %>% summarise(mean=mean(AGE)) %>% ungroup()
ggplot(one_iter) + 
  geom_density(aes(x = AGE, color = d_nr)) + ylab("Frequency")+ theme_light() + 
  theme(legend.title = element_blank())+scale_colour_brewer(palette="Set1") + 
  geom_vline(data = mean, aes(xintercept=mean, color = d_nr), linetype = "dashed")
```

<img src="assignment3_boyoonChang_files/figure-html/a3plot-6.png" style="display: block; margin: auto;" />

```r
# plot treatment and GENDER
ggplot(one_iter) + 
  geom_bar(aes(x = d_nr, fill = GENDER)) + ylab("Frequency")+ theme_light() + 
  theme(legend.title = element_blank())+scale_fill_brewer(palette="Set1") 
```

<img src="assignment3_boyoonChang_files/figure-html/a3plot-7.png" style="display: block; margin: auto;" />

Contrary to the initial set-up to have the treatment more likely to be assigned to male than female, we see higher proportion of female in the treated group than in control. Perhaps data generating process creates $\frac{IQ}{120}  + \frac{EDU}{5} - \frac{AGE}{45} + e$ more likely to be higher than $\frac{IQ}{100} + 4 \cdot EXP+ e$. Notice that the probability of male in the sample to cross the threshold 3 of being assigned to treatment is 0.3860759 whereas female in the sample to cross the threshold of being assigned to treatment is 0.584507. The higher probability of female sample to cross the threshold may explain the higher proportion of female in treated group than in control.

Comparing other covariates, our treatment group seems to have higher IQ, greater number of experience and education, and are more likely to be younger. Below are the average treatment effect estimate without controlling for covariates and the plots showing the relationship of the selection basis and the wage level of the whole sample. As can be seen, we do not observe a stark differences in the wage level below and above the threshold. This could be because only one of the gender is influenced by the selection criteria and thus the other gender group that is not influenced by such criteria may have blurred the wage difference around the threshold.


```r
one_iter %<>% mutate(d_g = ifelse(GENDER=="Male",1,0))
mean <- one_iter %>% group_by(d_nr) %>% summarise(mean=mean(d_g)) %>% ungroup()
one_iter %>% group_by(d_nr) %>% summarize(mean = mean(y_nr)) 
```

<!--html_preserve--><table class="huxtable" style="border-collapse: collapse; border: 0px; margin-bottom: 2em; margin-top: 2em; ; margin-left: auto; margin-right: auto;  " id="tab:unnamed-chunk-2">
<col><col><tr>
<th style="vertical-align: top; text-align: left; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0pt 0.4pt 0.4pt;    padding: 6pt 6pt 6pt 6pt; font-weight: bold;">d_nr</th><th style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0.4pt 0.4pt 0pt;    padding: 6pt 6pt 6pt 6pt; font-weight: bold;">mean</th></tr>
<tr>
<td style="vertical-align: top; text-align: left; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0pt 0pt 0.4pt;    padding: 6pt 6pt 6pt 6pt; background-color: rgb(242, 242, 242); font-weight: normal;">Control</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0.4pt 0pt 0pt;    padding: 6pt 6pt 6pt 6pt; background-color: rgb(242, 242, 242); font-weight: normal;">7.85</td></tr>
<tr>
<td style="vertical-align: top; text-align: left; white-space: normal; border-style: solid solid solid solid; border-width: 0pt 0pt 0.4pt 0.4pt;    padding: 6pt 6pt 6pt 6pt; font-weight: normal;">Treated</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0pt 0.4pt 0.4pt 0pt;    padding: 6pt 6pt 6pt 6pt; font-weight: normal;">10&nbsp;&nbsp;&nbsp;</td></tr>
</table>
<!--/html_preserve-->

```r
# plot treatment and other variables
ggplot(one_iter) + 
  geom_boxplot(aes(x = d_nr, y = y_nr, color = d_nr)) + ylab("log Wage")+ theme_light() + 
  theme(legend.title = element_blank())+scale_colour_brewer(palette="Set1")+ 
  geom_vline(xintercept = 3, linetype = "dashed" )+
  labs(title = "Relationship between Log Wage and Treatment")
```

<img src="assignment3_boyoonChang_files/figure-html/unnamed-chunk-2-1.png" style="display: block; margin: auto;" />

```r
# plot log wage and selection basis of male
ggplot(one_iter) + 
  geom_point(aes(x = IQ / 100 + e + 4 * EXP, y = y_nr, color = d_nr)) + 
  ylab("log Wage")+ theme_light() + 
  geom_vline(xintercept = 3, linetype = "dashed" )+
  theme(legend.title = element_blank())+scale_colour_brewer(palette="Set1")+
  labs(title = "Relationship between Log Wage and Selection Basis of Male")+
  geom_smooth(aes(x = IQ / 100 + e + 4 * EXP, y = y_nr, color = d_nr),method = "loess") + 
  facet_wrap(~MALE)
```

<img src="assignment3_boyoonChang_files/figure-html/unnamed-chunk-2-2.png" style="display: block; margin: auto;" />

```r
# plot log wage and selection basis of female
ggplot(one_iter) + 
  geom_point(aes(x = IQ / 120 + e + EDU/5 - AGE/45 , y = y_nr, color = d_nr))+ 
  ylab("log Wage")+ theme_light() + 
  geom_vline(xintercept = 3, linetype = "dashed" )+
  theme(legend.title = element_blank())+scale_colour_brewer(palette="Set1")+
  labs(title = "Relationship between Log Wage and Selection Basis of Female") + 
  geom_smooth(aes(x = IQ / 120 + e + EDU/5 - AGE/45 , y = y_nr, color = d_nr),method = "loess")+
  facet_wrap(~MALE)
```

<img src="assignment3_boyoonChang_files/figure-html/unnamed-chunk-2-3.png" style="display: block; margin: auto;" />


- Simulation to estimate average treatment effect 

Next we look at how such non-random treatment plays out in simulation and estimation of the average treatment effect. Below I set sample size equal to 100 and performed 500 iterations. The results suggest that the treatment effect is fairly well estimated in the presence of selection bias. This is surprising since typically non-random treatment bias the average treatment effect. I would argue that the different selection basis between how each female and male are selected into treatment introduces arbitrary randomness into the model, which could have contributed to rather consistent estimate of the average treatment effect. We see greater consistency in average treatment effect estimate as we increase the sample size or the number of iterations.


```r
# a case where the treatment is endogenous
# iterating simulation process
library(truncnorm)
set.seed(123)
n = 1
sim_iter <- function(n){
  dgp_iter = tibble(
    i = 1:n,
    u = runif(n, -10, 10), 
    v = runif(n, -5,5),
    e = runif(n, -1,1),
    W = rnorm(n, 3,2),
    IQ = round(rnorm(n, mean = 120, sd =20),digits = 0), 
    EXP = rtruncnorm(rnorm(n, mean = 3, sd = 2)),
    EDU = sample(c(12:21), n, replace = TRUE),
    GENDER = sample(c("Male", "Female"), n , replace = TRUE),
    MALE = ifelse(GENDER == "Male", 1, 0),
    AGE = sample(c(24:80), n, replace = TRUE),
    X = 0.02*IQ + 0.05* EXP + 0.06*EDU + 
        0.25*ifelse(GENDER=="Male",1,0) + 
        0.07*AGE, 
    y0 = X + u,
    y1 = y0 + W + v, # suppose w being ITE an the arbitrary treatment effect imposed
    t = y1 - y0, # 
    d_r = sample(c(1,0), n, replace = TRUE),
    d_nr = ifelse( ifelse(MALE ==1, 
                   IQ / 100 + e + 4 * EXP ,
                   IQ / 120 + e + EDU/5 - AGE/45 ) > 3 , 1 , 0) %>% as.numeric(),
    y_r = y0 + d_r*t,
    y_nr = y0+ d_nr*t
  )
  bind_rows(
    lm(y_r ~ d_r , data = dgp_iter)%>% broom::tidy() %>% filter(term == "d_r"),
    lm(y_nr ~ d_nr , data = dgp_iter)%>% broom::tidy() %>% filter(term == "d_nr")
  ) %>% mutate(treatment = c("random", "nonrandom"))
}

p_load(furrr)
set.seed(1234)
plan(multiprocess, workers = 12, .progress = T)
invisible(future_options(seed = 1234L))
small_df = future_map_dfr(rep(100,500), sim_iter)
ggplot(data = small_df, aes(x= estimate, fill = treatment)) +
  geom_density(color = NA, alpha = 0.6) + 
  geom_hline(yintercept = 0) + 
  geom_vline(xintercept = 3, linetype = "dashed" )+
  labs(x = "Estimate", y = "Density")+ 
  scale_fill_viridis_d("Treatment", option="magma", end = 0.9) +
  theme_minimal(base_size = 12) +
  theme(legend.position= "bottom")
```

<img src="assignment3_boyoonChang_files/figure-html/unnamed-chunk-3-1.png" style="display: block; margin: auto;" />

```r
library(huxtable)
table = small_df %>%
  group_by(treatment) %>%
  summarize(
    "Mean Coef" = mean(estimate),
    "Median Coef" = median(estimate),
    "Std. Dev. Coef" = sd(estimate),
    "Rejection Rate" = mean(p.value < 0.05)
  ) %>%
  hux() %>% huxtable::add_colnames() %>%  ungroup()
table[-1,]
```

<!--html_preserve--><table class="huxtable" style="border-collapse: collapse; border: 0px; margin-bottom: 2em; margin-top: 2em; ; margin-left: auto; margin-right: auto;  " id="tab:unnamed-chunk-3">
<col><col><col><col><col><tr>
<th style="vertical-align: top; text-align: left; white-space: normal; padding: 6pt 6pt 6pt 6pt; font-weight: normal;">treatment</th><th style="vertical-align: top; text-align: right; white-space: normal; padding: 6pt 6pt 6pt 6pt; font-weight: normal;">Mean Coef</th><th style="vertical-align: top; text-align: right; white-space: normal; padding: 6pt 6pt 6pt 6pt; font-weight: normal;">Median Coef</th><th style="vertical-align: top; text-align: right; white-space: normal; padding: 6pt 6pt 6pt 6pt; font-weight: normal;">Std. Dev. Coef</th><th style="vertical-align: top; text-align: right; white-space: normal; padding: 6pt 6pt 6pt 6pt; font-weight: normal;">Rejection Rate</th></tr>
<tr>
<td style="vertical-align: top; text-align: left; white-space: normal; padding: 6pt 6pt 6pt 6pt; font-weight: normal;">nonrandom</td><td style="vertical-align: top; text-align: right; white-space: normal; padding: 6pt 6pt 6pt 6pt; font-weight: normal;">2.73</td><td style="vertical-align: top; text-align: right; white-space: normal; padding: 6pt 6pt 6pt 6pt; font-weight: normal;">2.65</td><td style="vertical-align: top; text-align: right; white-space: normal; padding: 6pt 6pt 6pt 6pt; font-weight: normal;">1.29</td><td style="vertical-align: top; text-align: right; white-space: normal; padding: 6pt 6pt 6pt 6pt; font-weight: normal;">0.536</td></tr>
<tr>
<td style="vertical-align: top; text-align: left; white-space: normal; padding: 6pt 6pt 6pt 6pt; font-weight: normal;">random</td><td style="vertical-align: top; text-align: right; white-space: normal; padding: 6pt 6pt 6pt 6pt; font-weight: normal;">2.95</td><td style="vertical-align: top; text-align: right; white-space: normal; padding: 6pt 6pt 6pt 6pt; font-weight: normal;">2.95</td><td style="vertical-align: top; text-align: right; white-space: normal; padding: 6pt 6pt 6pt 6pt; font-weight: normal;">1.43</td><td style="vertical-align: top; text-align: right; white-space: normal; padding: 6pt 6pt 6pt 6pt; font-weight: normal;">0.602</td></tr>
</table>
<!--/html_preserve-->




## A2 - Estimators

> Due date: 19 January 2020

_For each scenario below, first describe the intuition of the estimator---what is being compared, what problems it fixes, why it works. Second, the identification strategy, inclusive of the assumption required for the identification of a causal parameter.  Third, provide an example of the estimator in use, inclusive of the specific assumptions that would be operational in that example. (Examples can be from existing literature, or from your own research programme.) Fourth, comment on any particularly relevant considerations to be made with respect to the estimation of standard errors in each environment. (There may not be.) Responses will be evaluated based on accuracy, completeness and clarity._

---

A difference estimate of the effect of $\mathbb{1}(T_i=1)$ on $Y_i$.

- The intuition behind it: Suppose random selection of subjects to treatment group and control group. The difference estimator compares the average outcome of the treated group with the average outcome of the control group to get the average causal treatment effect. It is the difference in average outcome between the two groups.  

$$
\begin{aligned}
\textrm{A difference estimator}
&= 
E(Y_{1i}|T_i=1) - E(Y_{0i}|T_i=0)\\
&= E(Y_{1i}|T_i=1) - E(Y_{0i}|T_i=1) + E(Y_{0i}|T_i=1)- E(Y_{0i}|T_i=0)\\
&= E(Y_{1i}-Y_{0i}|T_i=1) + E(Y_{0i}|T_i=1)- E(Y_{0i}|T_i=0)\\
&= \textrm{average treatment effect on the treated} + \textrm{selection bias (pre-treatment)}\\
\end{aligned}
$$

- The identification assumption(s): 
(a)  There is no selection bias in the absence of the treatment. Some individual characteristics do not govern the placement of the treatment, i.e., individuals are randomly selected to the treated or control group. This allows for the control group to be used as the counter-factual for the treated group ($E(Y_{0i}|T_i=1)=E(Y_{0i}|T_i=0)$). 
(b)  It assumes that there are no other reasons except the treatment for the change in mean outcomes before and after the treatment. That is, the treatment stands for the impact of all factors that are different between the two groups. 
(c)  The average outcome of the treated group remains constant throughout earlier and later periods of the treatment. The average outcome of the control group remains constant before and after the treatment. We don't consider the time component in difference estimator.  


- An example: Provided that the control group and treatment group select into treatment randomly, difference estimator in RCT setting could provide average causal treatment effect.

- Anything particular about standard errors: We may want to adjust for standard error if we see the variation of the outcome in treatment group different from the variation of the outcome in control group. However, here we may have too few clusters (two clusters), which could be problematic. 


<br>
A matching-type estimator of the effect of  $\mathbb{1}(T_i=1)$ on $Y_i$.

- The intuition behind it: Suppose the selection into treatment is non-random but is well approximated by observable covariates ($X$) or by $f(X)$. A matching-type estimator first pairs up the individuals from the treated and control group that have similar distributions on the covariates. Then for each matching pair, treatment effect is calculated by taking the difference in the outcome. Lastly, by taking the average of those differences, we can reasonably argue that a matching-type estimate of the treatment effect is the average causal treatment effect. A matching-type estimator helps to mitigate the fundamental problem of causal inference by using an individual with similar characteristics as a counterfactual for the treated individual. 

- The identification assumption(s): 
(a) Treated and control group share the same covariates and these covariates are observable.
(b) The selection into treatment is non-random and is well approximated by $X$ or $f(X)$.
(c) Conditional independence assumption (CIA): Conditional on observed characteristics $X$, the assignment of treatment is independent of the treatment-specific outcomes so that selection bias disappears. Recall CIA is satisfied if $X$ includes all variables that affect both selection into treatment and outcomes. 
(d) Overlap assumption: Conditional on observed characteristics $X$, the probability of being assigned to treated group is in between 0 and 1 ($Pr(T=1|X=x') \in (0,1)$). This implies that the probability of observing individuals in control group at each level of $X$ is positive. Notice that if $Pr(T=1|X=x')=1$, the there is no good matching individual in control group to be used as a counterfactual.

- An example: Nearest-neighbor matching, propensity score methods are some of the matching estimators. 

- Anything particular about standard errors: If variation in outcome across matching groups are different, we may account for it by using clustered standard error.


<br>
A difference-in-difference estimate of the effect of  $\mathbb{1}(T_i=1)$ on $Y_i$. 

- The intuition behind it: Difference-in-difference estimator is the time dimension added version of difference estimator and is used to estimate the average treatment effect. By including time dimension, the estimator addresses serial correlation of the outcomes. A difference-in-difference estimator compares the differences in average outcome in the treated group and control group across time. Suppose that the treated and control group share the same time trend. The difference-in-difference estimator isolates such trend component from the outcomes of the treated individuals. It does so by subtracting the differences in post-treatment and pre-treatment means of the control group from the difference in the post-treatment and pre-treatment means of the treated group. Note that this process also removes unobservable characteristics that are fixed across time within each group that could have contributed to the difference in outcome of the two groups.   

- The identification assumption(s): All the OLS model assumptions apply for DiD if we run a regression for DiD estimator. In addition, DiD assumes parallel trend assumption. This implies that both treatment and control group share the same trend over time and thus their difference in outcome is constant if treatment did not occur. 

- An example: Card and Krueger (1994) article about change in minimum wage on employment in the fast food sector. Having the change in employment in Pennsylvania to serve as a base, it controls for any bias caused by variables common to New Jersey and Pennsylvania. A simple statistical formulation of the model is $Y_{it} = \beta_0 + \beta_1 S_t + \beta_2 T_i + \beta_3 (S_t\times T_i) + \varepsilon_{it}$, where $S_t$ is time dummy ( $S_t = 0$ if $t=1$ and $S_t=1$ if $t=2$), and $T_i$ is the treatment dummy. The difference-in-difference estimate would then be $\beta_4$ which represents $(E(Y_{i2}|T_i=1)- E(Y_{i1}|T_i=1))-(E(Y_{i2}|T_i=0)- E(Y_{i1}|T_i=0))$. 

- Anything particular about standard errors: It is helpful to use robust standard errors to account for autocorrelation or correlation within identical group before and after the treatment. 


<br>
Instrumenting for an endogenous regressor, $X_i$, with an instrument, $Z_i$, in order to retrieve an estimate of the causal effect of $X_i$ on $Y_i$. 

- The intuition behind it: We control for the bad variation of $X_i$ by regressing it with respect to $Z_i$. We use the fitted $X_i$ to estimate the causal effect of $X_i$ on $Y_i$. IV methods solve for bias from measurement error in regression models. Recall that a regression coefficients is biased toward zero if the regressors contain large noise or measurement error. IV also solves for omitted variable bias in a sense that it isolates irrelevant variation of the endogenous regressors. 

- The identification assumption(s): The instruments should be relevant in that its variation must be related to the variation in the instrumented variable $X_i$. The instruments have no effect on $Y_i$ except through $X_i$. The instruments should be exogenous or predetermined in that they are uncorrelated with the disturbances.  

- An example: Consider a case where we are interested in the returns on schooling. The outcome variable is wage and the variable of interest is education. Here, education is defined to be years of schooling. Suppose education is endogenous, i.e. it is correlated with the error term. OLS estimator no longer produces consistent estimate since exogeneity assumption is violated. Thus we introduce an instrument variable, arguably a valid one to isolate bad variation from education. In general, 2SLS is used for IV estimator so at the first stage, education is regressed on the instrument to get the fitted value of education. Then on the second stage, wage is regressed on the fitted value of education. As long as the bad variation in education is well controlled for at the first stage, 2SLS produces consistent estimate for the average treatment effect. 

- Anything particular about standard errors: Standard error in IV is reported to be in general larger than OLS standard error although IV estimators generate a consistent estimate in contrast to OLS with endogenous regressors. If the instruments are weak, i.e. they have very low correlation with the regressors, the instrumental variable estimators or 2SLS estimators could lead to large or inconsistent standard errors. 


<br>
A regression-discontinuity approach to estimating the effect of  $\mathbb{1}(T_i=1)$ on $Y_i$.

- The intuition behind it: Suppose researchers have some knowledge that the non-random treatment is governed at least partly by some threshold value ($c$) of an observed covariate $X_i$ and that passing this threshold induces a change in potential outcome $Y_i$. RD then regards the discontinuity of the mean outcome along the covariate at the cutoff value as causal effect of treatment. Regression discontinuity comes in sharp and fuzzy. In sharp RD, we look at the discontinuity in conditional expectation of the outcome given the covariate to calculate an average causal effect of the treatment. The probability of the treatment changes from 0 to 1 (either not treated or treated) as $X_i$ moves across some threshold $c$. Then RD estimate the local average treatment effect by comparing the mean of the outcome that is right above and right below that threshold $c$. In fuzzy RD, the probability of the treatment that is strictly less than 1 changes as $X_i$ crosses the cutoff. This implies that the effect of $X_i$ crossing the cutoff has influence on the outcome as well as the probability of treatment. Therefore the treatment effect is defined by the ratio of these two effects. Fuzzy RD is similar to IV in a sense that it estimates a change in probability of treatment for variable $X_i-c$ and use the fitted probability of treatment to estimate the change in outcome.  If we extrapolate $E(Y_{0i}|X_i)$ and $E(Y_{1i}|X_i)$ not only upon the threshold $c$ but for the entire $X_i$ horizon, and compare the pre- and post-treatment differences in outcome means for those below the threshold to those above it, regression discontinuity is similar to the difference-in-difference estimator. The fuzzy RD estimates the local average treatment effect of the compliers.

- The identification assumption(s):
(a) Researchers know the assignment mechanism is some function of observable variable $X$. 
(b) Conditional independence assumption: Conditional on the covariates, there is no variation in the treatment.
(c) We observe a discontinuous change in the probability of treatment at some cut-off point. 
(d) $E(Y_{1i}|X_i=x)$ and $E(Y_{0i}|X_i=x)$ are continuous in $x$. This assumption is necessary to extrapolate $E(Y_{0i}|X_i=x)$ around the neighborhood of the discontinuity to use it for counterfactual of the average outcome of treated individuals around that threshold.
(e) Monotonicity assumption: Suppose $T_i(X=x^\star)$ denotes the potential treatment of $i$ with threshold $x^{\star}$. In fuzzy RD, $T_i(X)$ is non-increasing in $x^\star$ at $x^\star = c$. That is, increasing $x^\star$ marginally from $c$ to $c+\varepsilon$ doesn't make someone more likely to be treated, i.e. there is no defiers.   

- An example: Suppose that we are interested in looking at the effect of Ph.D. degree on wage. For the sake of simplicity, suppose that students with GRE test score above 160 get the degree and those below the score don't. The basic idea is that RD splits individuals into two groups below and above the score of 160 and get the difference in the outcome to estimate the average treatment effect. Thus the underlying assumption is that the ability and covariates of these two groups are similar enough that the wage comparison for those on either side of the GRE score of 160 gives us high internal validity. However, recall that the estimator is only so useful to estimate "local treatment effect" around the cutoff, otherwise the control group's outcome wouldn't be a good counterfactual to the treatment group. For example, comparing wages of individuals with GRE score very much far away from the cutoff, say those with score of 130 with those with 170, does not tell anything about whether that person holds a Ph.D. degree.  

- Anything particular about standard errors: Bandwidth selection could affect the estimates and standard errors. 

## A1 - OVB simulation

> Due date: 12 January 2020
>
> Please have your simulation ready to share in class on Tuesday.

When people make the claim that _correlation does not imply causation_, they usually mean that the existence of some correlation between $y$ and $x_1$ does not imply that variation in $x_1$ _causes_ variation in $y$.

- That is, they tend to be acknowledging that you can have correlation without causation

**Assignment** Propose and simulate a data-generating process in which (**i**) causation runs from $x_1$ to $y$ but at the same time (**ii**) the correlation of $y$ and $x_1$ is zero. Write it in an Rmd file and make the argument visually.

- Yes, at the end of this you should have simulated something (good) and demonstrated that the lack of correlation does not imply lack of causation (which is like a party trick).


### Formal Definition of Correlation

$$
\begin{aligned}
  r_{xy} 
  &= \frac{s_{xy}}{s_{x}s_{y}} \\
  &= \frac{(n-1)^{-1}\sum_{i=1}^{n}(x_i-\bar{x})(y_i-\bar{y})}
  {(n-1)^{-1}\sqrt{\sum_{i=1}^n(x_i-\bar{x})^2\sum_{i=1}^{n}(y_i-\bar{y})^2}}\\
  &= \frac{\sum_{i=1}^{n}(x_i-\bar{x})(y_i-\bar{y})}
  {\sqrt{\sum_{i=1}^n(x_i-\bar{x})^2\sum_{i=1}^{n}(y_i-\bar{y})^2}}
\end{aligned}
$$
To make correlation equal to zero, it should either be that the numerator which is the covariance between $x$ and $y$ is sufficiently small to approach zero, or that the denominator is sufficiently large which could happen if either the variation of $x$ or $y$ is very large. In the example that follows, I considered a case where the variance of $y$ is large that is driven by some omitted variable $z$.  


```r
## Looking at correlation
fun_iter_corr <- function(iter, n = 30){
  iter_df <- tibble(
    e = rnorm(n, 0, 1),
    z = rnorm(n, -100, 10),
    x = rnorm(n, 1, 0.5), 
    y = x + z + e
  )
  corr<- cor(iter_df$x, iter_df$y)
}

sim_df<-sapply(1:1000, fun_iter_corr)
sim_df<-as.data.frame(sim_df)


ggplot() + 
  geom_density(data = sim_df, aes(x = sim_df), color = "black")+
  xlab("correlation")  +
  geom_vline(xintercept = 0, linetype = "longdash", color = "red") 
```

<img src="assignment3_boyoonChang_files/figure-html/unnamed-chunk-4-1.png" style="display: block; margin: auto;" />


### Correlation in Broader Sense

More broadly, suppose we define the correlation being the statistical significance of $x_1$ on $y$. In other words, if the null hypothesis that the coefficient estimate of $x_1$ on $y$ is zero is rejected at some alpha percent significance level, then $x_1$ is defined to be correlated with $y$. In other words, if we fail to reject the null hypothesis, then this implies that the coefficient estimate of $x_1$ on $y$ is not statistically different from zero, i.e. $x_1$ is not correlated with $y$. 

Suppose that $y$ is the sum of $x_1$ and $\varepsilon$, i.e. random disturbances. When the magnitude and the standard error of the random disturbance, $\varepsilon$, is relatively greater than the causal factor, $x_1$, the coefficient estimate of $x_1$ could be reported as not statistically significant. See the example below:


```r
# first iteration
data1 = tibble(e = rnorm(100, 1000, 500),
               x = rnorm(100, 1, 0.5), 
               y = x + e)
summary(lm(y~x, data = data1))
```

```
#> 
#> Call:
#> lm(formula = y ~ x, data = data1)
#> 
#> Residuals:
#>      Min       1Q   Median       3Q      Max 
#> -1235.05  -367.98   -81.49   319.67  1347.87 
#> 
#> Coefficients:
#>             Estimate Std. Error t value Pr(>|t|)    
#> (Intercept)  1094.61      94.16   11.62   <2e-16 ***
#> x             -79.76      84.89   -0.94     0.35    
#> ---
#> Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
#> 
#> Residual standard error: 475.8 on 98 degrees of freedom
#> Multiple R-squared:  0.008928,	Adjusted R-squared:  -0.001185 
#> F-statistic: 0.8828 on 1 and 98 DF,  p-value: 0.3497
```

```r
ggplot(aes(x = x, y = y), data = data1)+geom_point()
```

<img src="assignment3_boyoonChang_files/figure-html/unnamed-chunk-5-1.png" style="display: block; margin: auto;" />

```r
cor(data1$x, data1$y)
```

```
#> [1] -0.0944869
```

```r
# function
fun_iter_l <- function(iter, n = 30) {
  iter_df <- tibble(
    e = rnorm(n, 0, 15), 
    x = rnorm(n, 1, 0.5), 
    y = x + e
  )
  lm <- lm_robust(y ~ x, data = iter_df, se_type = "classical")
  bind_rows(tidy(lm)) %>% 
    select(1:5) %>% filter(term == "x") %>% 
    mutate(se_type = c("classical"), i = iter, variation="large")
}

fun_iter_s <- function(iter, n = 30, a, b) {
  # Generate data
  iter_df <- tibble(
    e = rnorm(n, 0, 0.5), 
    x = rnorm(n, 1, 0.5), 
    y = x + e
  )
  # Estimate models
  lm <- lm_robust(y ~ x, data = iter_df, se_type = "classical")
  # Stack and return results
  bind_rows(tidy(lm)) %>%
    select(1:5) %>% filter(term == "x") %>%
    mutate(se_type = c("classical"), i = iter, variation="small")
}


# perform 100 iteration
p_load(purrr)
set.seed(1234)
sim_list_l <- map(1:100, fun_iter_l)
sim_list_s <- map(1:100, fun_iter_s)
sim_df <- bind_rows(sim_list_l, sim_list_s)
sim_df_s <- bind_rows(sim_list_s)

#plotting
ggplot(data = sim_df, aes(x = std.error, fill = variation)) +
  geom_density(color = NA) +
  geom_hline(yintercept = 0) +
  xlab("Standard error") +
  ylab("Density") +
  scale_fill_viridis(
    "", labels = c("var(e) large", "var(e) small"), discrete = T,
    option = "B", begin = 0.25, end = 0.85, alpha = 0.9
  ) +
   theme(legend.position = c(0.8, 0.8))
```

<img src="assignment3_boyoonChang_files/figure-html/unnamed-chunk-5-2.png" style="display: block; margin: auto;" />

```r
ggplot(data = sim_df, aes(x = statistic, fill = variation)) +
  geom_density(color = NA) +
  geom_hline(yintercept = 0) +
  geom_vline(xintercept = qt(0.975, df = 28), linetype = "longdash", color = "red") +
  xlab("t statistic") +
  ylab("Density") +
  scale_fill_viridis(
    "", labels = c("var(e) large", "var(e) small"), discrete = T,
    option = "B", begin = 0.25, end = 0.85, alpha = 0.9
  ) +
  theme(legend.position = c(0.8, 0.8))
```

<img src="assignment3_boyoonChang_files/figure-html/unnamed-chunk-5-3.png" style="display: block; margin: auto;" />

The first graph above shows the distribution of the standard error of $\hat{\beta}$ from 100 iterations (i.e. simulation). The distribution of the standard error of $\hat{\beta}$ when the variation of the disturbance is large tend to locate on the farther right to the distribution of the standard error of $\hat{\beta}$ when the variation of the disturbance is small. This implies that the volatility of the disturbance could result in large variation in the standard error of the point estimate of $\beta$. If such is the case, it is likely that the t-stat calculated based on it is reported to be small, which results in higher likelihood of not rejecting the null hypothesis that $\beta$ is zero, i.e. no correlation between $y$ and $x_1$. 

The next graph shows the distribution of t-statistic of $\hat{\beta}$ from 100 iterations. The red dotted line denotes the 95$\%$ confidence interval. When the t-statistic for each 100 point estimate of $\beta$ falls outside the red dotted line, this indicates that we are likely to reject the null hypothesis that $\beta$ is zero. Notice that when the variance of disturbance term is small, we are more likely to conclude that $\beta$ estimate being different from 0. In other words, we are more likely to conclude causal inference from the regression. However, when the variation of the disturbance is large, the t-statistic of 100 point estimate of $\beta$ is likely to locate within the confidence interval, which may lead us to conclude that there is insufficient evidence to conclude causal relationship between $y$ and $x_1$. 



### Other cases

- When the model is misspecified: when the model is linearly specified in x when the true functional form of x is quadratic.

```r
df2 = tibble(x = rnorm(100, 2, 50), 
             z = rnorm(100, 3, 1000),
             e = rnorm(100, 0, 1),
             y1 = x^2 + z + e,
             y2 = x^2 + e,
             y3 = x + e)
ggplot(data = df2, aes(x = x, y = y1)) + geom_point()
```

<img src="assignment3_boyoonChang_files/figure-html/unnamed-chunk-6-1.png" style="display: block; margin: auto;" />

```r
summary(lm(y1~x, data = df2))
```

```
#> 
#> Call:
#> lm(formula = y1 ~ x, data = df2)
#> 
#> Residuals:
#>     Min      1Q  Median      3Q     Max 
#> -4421.6 -2256.9  -926.2  1130.8 13149.6 
#> 
#> Coefficients:
#>             Estimate Std. Error t value Pr(>|t|)    
#> (Intercept) 2428.594    347.568   6.987 3.41e-10 ***
#> x              2.522      6.986   0.361    0.719    
#> ---
#> Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
#> 
#> Residual standard error: 3471 on 98 degrees of freedom
#> Multiple R-squared:  0.001328,	Adjusted R-squared:  -0.008863 
#> F-statistic: 0.1303 on 1 and 98 DF,  p-value: 0.7189
```

```r
# ggplot(data = df2, aes(x = x, y = y2)) + geom_point()
# summary(lm(y2~x, data = df2))
```
- When the sample that we are using from the data is truncated in a way that does not show a correlation




## Ideas

### Idea 1

1. What is the question being asked of the data?
<br>  - here

1. Why do I care about it? Why should anyone else care?
<br>  - here

1. What methodologies are being used to answer the question?
<br>  - here

1. If a causal claim is being made, what must I assume in order to interpret the relationship as causal? 
<br>  - here

1. What are the main findings?
<br>  - here

---

### Idea 2

1. What is the question being asked of the data?
1. Why do I care about it? Why should anyone else care?
1. What methodologies are being used to answer the question?
1. If a causal claim is being made, what must I assume in order to interpret the relationship as causal? 
1. What are the main findings?

---

Etc.


## Sim Example

_When your intuition is exhausted or your confidence is lacking, you need a tool. When your intuition is on point but you also need a confidence boost, you need a tool. When you are writing estimators and you wish to demonstrate its properties, you need a tool._ 

---

> You took Ed Rubin's class... so I know you've seen the material below. I'm providing it here with some editing, so it's in this format, but do consider consulting the material from that class directly. 

---

### The recipe

1. Define a data-generating process (DGP)
1. Define an estimator or estimators, setting up the test/conditions you're looking for
1. Set seed and run many iterations of
<br>  a. Drawing a sample of size n from the DGP
<br>  b. Conducting the exercise
<br>  c. Record outcomes
1. Communicate results

---

### The data-generating process

$$
\begin{align}
  \text{Y}_{i} = 1 + e^{0.5 \text{X}_{i}} + \varepsilon_i
\end{align}
$$
where $\text{X}_{i}\sim\mathop{\text{Uniform}}(0, 10)$ and $\varepsilon_i\sim\mathop{N}(0,15)$.






```r
library(pacman)
p_load(dplyr)
# Choose a size
n <- 1000
# Generate data
dgp_df <- tibble(
  ε = rnorm(n, sd = 15),
  x = runif(n, min = 0, max = 10),
  y = 1 + exp(0.5 * x) + ε
)
```

<!--html_preserve--><table class="huxtable" style="border-collapse: collapse; border: 0px; margin-bottom: 2em; margin-top: 2em; ; margin-left: auto; margin-right: auto;  " id="tab:sim_dply, printed">
<col><col><col><tr>
<th style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0pt 0.4pt 0.4pt;    padding: 6pt 6pt 6pt 6pt; font-weight: bold;">e</th><th style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0pt 0.4pt 0pt;    padding: 6pt 6pt 6pt 6pt; font-weight: bold;">x</th><th style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0.4pt 0.4pt 0pt;    padding: 6pt 6pt 6pt 6pt; font-weight: bold;">y</th></tr>
<tr>
<td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0pt 0pt 0.4pt;    padding: 6pt 6pt 6pt 6pt; background-color: rgb(242, 242, 242); font-weight: normal;">8.78</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0pt 0pt 0pt;    padding: 6pt 6pt 6pt 6pt; background-color: rgb(242, 242, 242); font-weight: normal;">9.53</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0.4pt 0pt 0pt;    padding: 6pt 6pt 6pt 6pt; background-color: rgb(242, 242, 242); font-weight: normal;">127&nbsp;&nbsp;</td></tr>
<tr>
<td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0pt 0pt 0pt 0.4pt;    padding: 6pt 6pt 6pt 6pt; font-weight: normal;">10.6&nbsp;</td><td style="vertical-align: top; text-align: right; white-space: normal; padding: 6pt 6pt 6pt 6pt; font-weight: normal;">6.22</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0pt 0.4pt 0pt 0pt;    padding: 6pt 6pt 6pt 6pt; font-weight: normal;">34&nbsp;&nbsp;</td></tr>
<tr>
<td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0pt 0pt 0pt 0.4pt;    padding: 6pt 6pt 6pt 6pt; background-color: rgb(242, 242, 242); font-weight: normal;">-1.64</td><td style="vertical-align: top; text-align: right; white-space: normal; padding: 6pt 6pt 6pt 6pt; background-color: rgb(242, 242, 242); font-weight: normal;">5.32</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0pt 0.4pt 0pt 0pt;    padding: 6pt 6pt 6pt 6pt; background-color: rgb(242, 242, 242); font-weight: normal;">13.6</td></tr>
<tr>
<td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0pt 0pt 0pt 0.4pt;    padding: 6pt 6pt 6pt 6pt; font-weight: normal;">-6.8&nbsp;</td><td style="vertical-align: top; text-align: right; white-space: normal; padding: 6pt 6pt 6pt 6pt; font-weight: normal;">8.92</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0pt 0.4pt 0pt 0pt;    padding: 6pt 6pt 6pt 6pt; font-weight: normal;">80.7</td></tr>
<tr>
<td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0pt 0pt 0pt 0.4pt;    padding: 6pt 6pt 6pt 6pt; background-color: rgb(242, 242, 242); font-weight: normal;">9.09</td><td style="vertical-align: top; text-align: right; white-space: normal; padding: 6pt 6pt 6pt 6pt; background-color: rgb(242, 242, 242); font-weight: normal;">1.96</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0pt 0.4pt 0pt 0pt;    padding: 6pt 6pt 6pt 6pt; background-color: rgb(242, 242, 242); font-weight: normal;">12.8</td></tr>
<tr>
<td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0pt 0pt 0.4pt 0.4pt;    padding: 6pt 6pt 6pt 6pt; font-weight: normal;">-27.3&nbsp;</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0pt 0pt 0.4pt 0pt;    padding: 6pt 6pt 6pt 6pt; font-weight: normal;">8.84</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0pt 0.4pt 0.4pt 0pt;    padding: 6pt 6pt 6pt 6pt; font-weight: normal;">57&nbsp;&nbsp;</td></tr>
</table>
<!--/html_preserve-->


**The CEF (in orange), and the population least-squares regression line (in purple)**

<img src="assignment3_boyoonChang_files/figure-html/sim_pop_plot3-1.png" style="display: block; margin: auto;" />

---

**Iterating**

To make iterating easier, let's wrap our DGP in a function.


```r
fun_iter <- function(iter, n = 30) {
  # Generate data
  iter_df <- tibble(
    ε = rnorm(n, sd = 15),
    x = runif(n, min = 0, max = 10),
    y = 1 + exp(0.5 * x) + ε
  )
}
```
We still need to run a regression and draw inference

---

### Inference

We will use `lm_robust()` from the `estimatr` package for OLS and inference.

- `se_type = "classical"` provides homoskedasticity-assuming SEs
- `se_type = "HC2"` provides heteroskedasticity-robust SEs
- `lm()` works for "spherical" standard errors but cannot calculate het-robust standard errors


```r
lm_robust(y ~ x, data = dgp_df, se_type = "classical") %>% tidy() %>% select(1:5)
```

<!--html_preserve--><table class="huxtable" style="border-collapse: collapse; border: 0px; margin-bottom: 2em; margin-top: 2em; ; margin-left: auto; margin-right: auto;  " id="tab:ex_lm_robust">
<col><col><col><col><col><tr>
<th style="vertical-align: top; text-align: left; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0pt 0.4pt 0.4pt;    padding: 6pt 6pt 6pt 6pt; font-weight: bold;">term</th><th style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0pt 0.4pt 0pt;    padding: 6pt 6pt 6pt 6pt; font-weight: bold;">estimate</th><th style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0pt 0.4pt 0pt;    padding: 6pt 6pt 6pt 6pt; font-weight: bold;">std.error</th><th style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0pt 0.4pt 0pt;    padding: 6pt 6pt 6pt 6pt; font-weight: bold;">statistic</th><th style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0.4pt 0.4pt 0pt;    padding: 6pt 6pt 6pt 6pt; font-weight: bold;">p.value</th></tr>
<tr>
<td style="vertical-align: top; text-align: left; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0pt 0pt 0.4pt;    padding: 6pt 6pt 6pt 6pt; background-color: rgb(242, 242, 242); font-weight: normal;">(Intercept)</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0pt 0pt 0pt;    padding: 6pt 6pt 6pt 6pt; background-color: rgb(242, 242, 242); font-weight: normal;">-21.1</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0pt 0pt 0pt;    padding: 6pt 6pt 6pt 6pt; background-color: rgb(242, 242, 242); font-weight: normal;">1.47&nbsp;</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0pt 0pt 0pt;    padding: 6pt 6pt 6pt 6pt; background-color: rgb(242, 242, 242); font-weight: normal;">-14.3</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0.4pt 0pt 0pt;    padding: 6pt 6pt 6pt 6pt; background-color: rgb(242, 242, 242); font-weight: normal;">1.38e-42&nbsp;</td></tr>
<tr>
<td style="vertical-align: top; text-align: left; white-space: normal; border-style: solid solid solid solid; border-width: 0pt 0pt 0.4pt 0.4pt;    padding: 6pt 6pt 6pt 6pt; font-weight: normal;">x</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0pt 0pt 0.4pt 0pt;    padding: 6pt 6pt 6pt 6pt; font-weight: normal;">10.5</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0pt 0pt 0.4pt 0pt;    padding: 6pt 6pt 6pt 6pt; font-weight: normal;">0.258</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0pt 0pt 0.4pt 0pt;    padding: 6pt 6pt 6pt 6pt; font-weight: normal;">40.7</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0pt 0.4pt 0.4pt 0pt;    padding: 6pt 6pt 6pt 6pt; font-weight: normal;">6.56e-214</td></tr>
</table>
<!--/html_preserve-->

```r
lm_robust(y ~ x, data = dgp_df, se_type = "HC2") %>% tidy() %>% select(1:5)
```

<!--html_preserve--><table class="huxtable" style="border-collapse: collapse; border: 0px; margin-bottom: 2em; margin-top: 2em; ; margin-left: auto; margin-right: auto;  " id="tab:ex_lm_robust">
<col><col><col><col><col><tr>
<th style="vertical-align: top; text-align: left; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0pt 0.4pt 0.4pt;    padding: 6pt 6pt 6pt 6pt; font-weight: bold;">term</th><th style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0pt 0.4pt 0pt;    padding: 6pt 6pt 6pt 6pt; font-weight: bold;">estimate</th><th style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0pt 0.4pt 0pt;    padding: 6pt 6pt 6pt 6pt; font-weight: bold;">std.error</th><th style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0pt 0.4pt 0pt;    padding: 6pt 6pt 6pt 6pt; font-weight: bold;">statistic</th><th style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0.4pt 0.4pt 0pt;    padding: 6pt 6pt 6pt 6pt; font-weight: bold;">p.value</th></tr>
<tr>
<td style="vertical-align: top; text-align: left; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0pt 0pt 0.4pt;    padding: 6pt 6pt 6pt 6pt; background-color: rgb(242, 242, 242); font-weight: normal;">(Intercept)</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0pt 0pt 0pt;    padding: 6pt 6pt 6pt 6pt; background-color: rgb(242, 242, 242); font-weight: normal;">-21.1</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0pt 0pt 0pt;    padding: 6pt 6pt 6pt 6pt; background-color: rgb(242, 242, 242); font-weight: normal;">1.43</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0pt 0pt 0pt;    padding: 6pt 6pt 6pt 6pt; background-color: rgb(242, 242, 242); font-weight: normal;">-14.7</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0.4pt 0.4pt 0pt 0pt;    padding: 6pt 6pt 6pt 6pt; background-color: rgb(242, 242, 242); font-weight: normal;">1.11e-44&nbsp;</td></tr>
<tr>
<td style="vertical-align: top; text-align: left; white-space: normal; border-style: solid solid solid solid; border-width: 0pt 0pt 0.4pt 0.4pt;    padding: 6pt 6pt 6pt 6pt; font-weight: normal;">x</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0pt 0pt 0.4pt 0pt;    padding: 6pt 6pt 6pt 6pt; font-weight: normal;">10.5</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0pt 0pt 0.4pt 0pt;    padding: 6pt 6pt 6pt 6pt; font-weight: normal;">0.31</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0pt 0pt 0.4pt 0pt;    padding: 6pt 6pt 6pt 6pt; font-weight: normal;">33.8</td><td style="vertical-align: top; text-align: right; white-space: normal; border-style: solid solid solid solid; border-width: 0pt 0.4pt 0.4pt 0pt;    padding: 6pt 6pt 6pt 6pt; font-weight: normal;">8.79e-168</td></tr>
</table>
<!--/html_preserve-->

---

Now add these estimators to our iteration function...


```r
fun_iter <- function(iter, n = 30) {
  # Generate data
  iter_df <- tibble(
    ε = rnorm(n, sd = 15),
    x = runif(n, min = 0, max = 10),
    y = 1 + exp(0.5 * x) + ε
  )
  # Estimate models
  lm1 <- lm_robust(y ~ x, data = iter_df, se_type = "classical")
  lm2 <- lm_robust(y ~ x, data = iter_df, se_type = "HC2")
  # Stack and return results
  bind_rows(tidy(lm1), tidy(lm2)) %>%
    select(1:5) %>% filter(term == "x") %>%
    mutate(se_type = c("classical", "HC2"), i = iter)
}
```


With that function in hand, let's run it 1,000 times.


There are a lot of ways to run a single function over a list/vector of values.

- `lapply()`, _e.g._, `lapply(X = 1:3, FUN = sqrt)`
- `for()`, _e.g._, `for (x in 1:3) sqrt(x)`
- `map()` from `purrr`, _e.g._, `map(1:3, sqrt)`

Let's go with `map()` from the `purrr` package because it easily parallelizes across platforms using the `furrr` package.

**Alternative 1: 1,000 iterations**


```r
# Packages
p_load(purrr)
# Set seed
set.seed(12345)
# Run 1,000 iterations
sim_list <- map(1:1e3, fun_iter)
```

**Alternative 2: Parallelized 1,000 iterations**


```r
# Packages
p_load(purrr, furrr)
# Set options
set.seed(123)
# Tell R to parallelize
plan(multiprocess)
# Run 10,000 iterations
sim_list <- future_map(
  1:1e3, fun_iter,
  .options = future_options(seed = T)
)
```

The `furrr` package (`future` + `purrr`) makes parallelization easy

Our `fun_iter()` function returns a `data.frame`, and `future_map()` returns a `list` (of the returned objects).

So `sim_list` is going to be a `list` of `data.frame` objects. We can bind them into one `data.frame` with `bind_rows()`.


```r
# Bind list together
sim_df <- bind_rows(sim_list)
```

---

### And the results?

Comparing the distributions of standard errors for the coefficient on $x$

<img src="assignment3_boyoonChang_files/figure-html/sim_plot1-1.png" style="display: block; margin: auto;" />


Comparing the distributions of $t$ statistics for the coefficient on $x$

<img src="assignment3_boyoonChang_files/figure-html/sim_plot2-1.png" style="display: block; margin: auto;" />





