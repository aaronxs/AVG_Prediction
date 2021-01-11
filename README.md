---
title: "Batting Average Prediction"
always_allow_html: yes
author: "Aaron Situ"
# date: "November 10, 2020"
output:
  rmarkdown::github_document:
  # bookdown::html_document2:
    # number_sections: true
    toc: true
    toc_depth: 4
---

# Batting Average Prediction {#partB}

## Project Introduction & Data Source {#proj_intro}

The goal of this project was to predict player's full season batting average for the 2018 season given their batting stats in March/April 2018. The data included 309 MLB players from the 2018 season.

## Data Exploration {#data_explore}

### Data Distribution
The sample distribution of Full AVG was first examined to see what type of models will be suitable for analysis. Figure \@ref(fig:fig1) below shows the sample distribution is fairly symmetric, which suggests a linear regression model with Normal error is suitable for predicting Full AVG. Using a Linear regression model has the advantage of high interpretability, although it is also fairly restrictive as it assumes a linear relationship between the predictors and the response variable.   


```{r echo=FALSE, eval = TRUE, warning=FALSE, message = FALSE, results = "markup", tidy = TRUE}
# Batting Average Prediction ------------------------------------------------------------------
# Load package R ------------------------------
if(!require(pacman)) {install.packages("pacman"); library(pacman)}
pacman::p_load(tidyverse, magrittr, highcharter, pls, arrayhelpers, psych, lmtest, knitcitations, caret)
# clear the bibliographic environment
cleanbib()

# Load data ------------------------------
datt_orig <- read.csv("batting.csv", stringsAsFactors = F)

# * data for modeling --------------------
datt <- datt_orig %>%
  select(-playerid, -Name, -Team)
n_obs <- nrow(datt)

# Data exploration ------------------------------
# * shapiro-wilk test for normality --------------------
sh_test <- shapiro.test(datt_orig$FullSeason_AVG)

# * Breusch–Pagan test for constant variance --------------------
bp_test <- bptest(FullSeason_AVG~., data = datt)

# * full season average qq plot data --------------------
qq_input <- qqnorm(datt_orig$FullSeason_AVG, plot.it = FALSE, datax = TRUE)
qq_input <- data.frame(Name = datt_orig$Name,
                       x = qq_input$x,
                       y = qq_input$y,
                       stringsAsFactors = F)

# * correlation matrix --------------------
corr_input <- cor(datt_orig[, -c(1:3)])
# p values for correlations
corrP_input <- ggcorrplot::cor_pmat(datt_orig[, -c(1:3)])
corrP_input <- data.frame(corrP_input) %>%
  rownames_to_column("param_x") %>%
  gather(key = param_y, value = pval, -param_x) %>%
  # round correlation with 4 decimal places
  mutate(pval = round(pval, 4))

# ** input for correlation heat map --------------------
heat_lab <- colnames(datt)
heat_lab <- ifelse(startsWith(heat_lab, "MarApr_"),
                   str_remove_all(heat_lab, "MarApr_"),
                   str_replace_all(heat_lab, "Season_", " "))
heat_lab <- str_replace(heat_lab, "\\.", "%")

heat_input <- data.frame(corr_input) %>%
  rownames_to_column("param_x") %>%
  gather(key = param_y, value = crl, -param_x) %>%
  # round correlation with 4 decimal places
  mutate(crl = round(crl, 4)) %>%
  # combine p values
  left_join(corrP_input, by = c("param_x", "param_y")) %>%
  # create tool tip lavel
  mutate(tip_lab = paste0(param_x, " vs. ", param_y)) %>%
  # convert stats to factor for ordering purpose
  mutate(param_x = factor(param_x,
                          level = rev(colnames(datt)),
                          labels = rev(heat_lab)),
         param_y = factor(param_y,
                          level = colnames(datt),
                          labels = heat_lab))
# train & test split set up --------------------
set.seed(2019)
train <- sample(1:nrow(datt), nrow(datt)*0.9)

# training set
datt_train <- datt %>%
  dplyr::slice(train)

# test set
x_test <- datt %>%
  select(-FullSeason_AVG) %>%
  dplyr::slice(-train)
y_test <- datt$FullSeason_AVG[-train]

# Principal Components Regression --------------------
# * feature selection via LOOCV
pcr_cv <- pcr(FullSeason_AVG~., data = datt_train, validation ="LOO", scale=TRUE)

# * MSE for CV models --------------------
pcr_cv_mse <- array2df(RMSEP(pcr_cv, estimate = c("all"))$val) %>%
  set_colnames(c("mse", "estimate", "response", "model")) %>%
  # round mse to 5 decimal
  mutate(mse = round(mse, 5),
         # remove text in model
         model = factor(model, labels = 0:25)) %>%
  # convert data to wide
  spread(key = estimate, value = mse) %>%
  # model name
  mutate(mod_name = "Principal Components Regression")

# * best fitted model based on lowest RMSE --------------------
pcr_best <- pcr(FullSeason_AVG~., data = datt_train, scale=TRUE, ncomp = 5)

# * original predictors/latent variables correlations --------------------
pcr_load <- data.frame(round(pcr_best$loadings[1:25, ], 4))
pcr_load <- pcr_load %>%
  rownames_to_column("Stats") %>%
  # use cleaned name for stats
  mutate(Stats = heat_lab[-26]) %>%
  # only show if absolute value of correlation >= 0.2
  mutate_at(vars(Comp.1:Comp.5), function(x){ifelse(abs(x) < 0.15, "", x)}) %>%
  set_colnames(c("Stats", paste0("Latent ", 1:5)))

# * calculate MSE using best fitted model/test data -------------------------
pcr_pred <- predict(pcr_best, newdata = x_test, ncomp = 5, type = "response")
pcr_mse <- mean((pcr_pred - y_test)^2)

# Partial Least Squares --------------------
# * feature selection via LOOCV
pls_cv <- plsr(FullSeason_AVG~., data = datt_train, validation ="LOO", scale=TRUE)

# * MSE for CV models --------------------
pls_cv_mse <- array2df(RMSEP(pls_cv, estimate = c("all"))$val) %>%
  set_colnames(c("mse", "estimate", "response", "model")) %>%
  # round mse to 5 decimal
  mutate(mse = round(mse, 5),
         # remove text in model
         model = factor(model, labels = 0:25)) %>%
  # convert data to wide
  spread(key = estimate, value = mse) %>%
  # model name
  mutate(mod_name = "Partial Least Squares")

# * best fitted model based on lowest RMSE --------------------
pls_best <- plsr(FullSeason_AVG~., data = datt_train, scale=TRUE, ncomp = 2)

# * original predictors/latent variables correlations --------------------
pls_load <- data.frame(round(pls_best$loadings[1:25, ], 4))
pls_load <- pls_load %>%
  rownames_to_column("Stats") %>%
  # use cleaned name for stats
  mutate(Stats = heat_lab[-26]) %>%
  # only show if absolute value of correlation >= 0.2
  mutate_at(vars(Comp.1:Comp.2), function(x){ifelse(abs(x) < 0.15, "", x)}) %>%
  set_colnames(c("Stats", paste0("Latent ", 1:2)))

# * calculate MSE using best fitted model/test data -------------------------
pls_pred <- predict(pls_best, newdata = x_test, ncomp = 2, type = "response")
pls_mse <- mean((pls_pred - y_test)^2)

# Factor analysis regression --------------------
# * feature selection via LOOCV --------------------
# dataframe to store LOOCV results
fa_cv_mse <- data.frame(model = numeric(0), RMSE = numeric(0), Rsquared = numeric(0), MAE = numeric(0))

# perform LOOCV for 1 to 25 factors
for(i in 1:25){
fa_cv <- fa(r = datt_train[, -26], nfactors = i, rotate = "varimax", fm = "ml")

# training data for LOOCV
fa_train <- datt_train %>%
  select(FullSeason_AVG) %>%
  bind_cols(data.frame(fa_cv$scores))

# LOOCV
fa_lm_cv <- train(FullSeason_AVG ~., method = "lm", data = fa_train, trControl = trainControl(method = "LOOCV"))

# storing results
fa_cv_mse <- fa_cv_mse %>%
  bind_rows(c(model = i, fa_lm_cv$results[-1]))
}
fa_cv_mse1 <- fa_cv_mse %>%
  rename(CV = "RMSE") %>%
  # onvert model to factor
  mutate(model = factor(model),
         CV = round(CV, 5),
         mod_name = "Factor Regression Model")

# * best fitted model based on lowest RMSE --------------------
fa_best <- fa(r = datt_train[, -26], nfactors = which.min(fa_cv_mse$RMSE), rotate = "varimax", fm = "ml")
# * original predictors/latent variables correlations --------------------
fa_load <- data.frame(round(fa_best$loadings[1:25, ], 4))
fa_load <- fa_load %>%
  select(paste0("ML", 1:5)) %>%
  rownames_to_column("Stats") %>%
  # use cleaned name for stats
  mutate(Stats = heat_lab[-26]) %>%
  # only show if absolute value of correlation >= 0.2
  mutate_at(vars(ML1:ML5), function(x){ifelse(abs(x) < 0.15, "", x)}) %>%
  set_colnames(c("Stats", paste0("Latent ", 1:5)))

# data to fit linear regression
fa_train_best <- datt_train %>%
  select(FullSeason_AVG) %>%
  bind_cols(data.frame(fa_best$scores))
fa_lm_best <- lm(FullSeason_AVG ~ ., data = fa_train_best)

# * calculate MSE using best fitted model/test data -------------------------
fa_newdata <- data.frame(factor.scores(x_test, fa_best)$scores)
fa_pred <- predict(fa_lm_best, newdata = fa_newdata)
fa_mse <- mean((fa_pred - y_test)^2)

# ** input for predicted/observed vatting average --------------------
pred_input <- datt_orig %>%
  # test data only
  dplyr::slice(-train) %>%
  select(Name, FullSeason_AVG) %>%
  # combine with predictions from 3 models
  cbind(pcr_pred) %>%
  cbind(pls_pred) %>%
  cbind(fa_pred) %>%
  set_colnames(c("Name", "obs", "pcr", "pls", "fa")) %>%
  # round number to 3 decimal
  mutate_at(vars(-Name), round, digits = 3) %>%
  # add line break to name
  mutate(Name = str_replace_all(Name, " ", "<br>")) %>%
  # prediction error as % of observed value
  mutate(pcr_err = (pcr - obs),
         pls_err = (pls - obs),
         fa_err = (fa - obs)) %>%
  arrange(Name)
```


```{r fig1, echo=FALSE, warning=FALSE, message = FALSE, fig.align="center", fig.cap = "Sample Distribution of Full Season Batting Average"}
# load("~/Ad-Hoc/Misc/Test_PP/Pred/PredData.RData")
# distribution of full season average --------------------
highcharter::hchart(density(datt$FullSeason_AVG), fillOpacity = 0.1) %>%
  # plot title
  highcharter::hc_title(text = "", style = list(fontSize = 0)) %>%
  # disable x axis label
  highcharter::hc_xAxis(title = list(text = "Full Season Batting Average", style = list(fontSize = 10))) %>%
  # disable y axis label
  highcharter::hc_yAxis(title = list(text = "", style = list(fontSize = 0))) %>%
  # disable legend
  highcharter::hc_legend(enabled = FALSE) %>%
  # disable tool tip
  highcharter::hc_tooltip(enabled = FALSE)
```

### Model Assumptions Assessment {#model_assess}

In order for the linear regression model to have the [desire properties and valid results](https://stats.stackexchange.com/questions/16381/what-is-a-complete-list-of-the-usual-assumptions-for-linear-regression), the following assumptions were assessed:

#### Normality of Full Season Batting Average with Constant Variance {#assume1}

To assess the Normality of Full AVG, i.e. whether it's reasonable to use Normal distribution model Full AVG, [Q-Q Plot](https://data.library.virginia.edu/understanding-q-q-plots/) was examined. Figure \@ref(fig:fig2) below plotted Full AVG against theoretical Normal quantiles. The plot shows the Full AVG aligns with Normal quantiles as they form an approximate diagonal line, although there are slight deviations in the lower tail of the sample. 

```{r fig2, echo=FALSE, warning=FALSE, message = FALSE, fig.align="center", fig.cap="Q-Q Plot of Full Season Batting Average"}
# QQ plot of full season average --------------------
# * tool tip --------------------
qq_tooltip_x <- c("Player", "Full Season Batting Average")
qq_tooltip_y <- sprintf("{point.%s}", c("Name", "x"))
qq_tip <- highcharter::tooltip_table(x = qq_tooltip_x, y = qq_tooltip_y)

# * create plot --------------------
highcharter::hchart(qq_input, "scatter", 
                            highcharter::hcaes(x = x, y = y),
                            marker = list(lineColor = "red",
                                          lineWidth = 1,
                                          radiu = 2,
                                          fillColor = "white")) %>%
  # plot title
  highcharter::hc_title(text = "", style = list(fontSize = 0)) %>%
  # disable x axis label
  highcharter::hc_xAxis(title = list(text = "Full Season Batting Average (Sorted)", style = list(fontSize = 10))) %>%
  # disable y axis label
  highcharter::hc_yAxis(title = list(text = "Theoretical Normal Quantiles", style = list(fontSize = 10))) %>%
  # disable legend
  highcharter::hc_legend(enabled = FALSE) %>%
  # tool tip format
  highcharter::hc_tooltip(useHTML = TRUE, pointFormat = qq_tip,
                          crosshairs = TRUE, shared = TRUE,
                          enabled = TRUE, headerFormat = "")
```

Furthermore, a [Shapiro–Wilk test](https://en.wikipedia.org/wiki/Shapiro%E2%80%93Wilk_test) was performed to check for Normality assumption and the results show no evidence against the Normality assumption at 5% significance (p-value = `r round(sh_test$p.value, 4)`).

To assess the constant variance assumptions, a [Breusch–Pagan test](https://en.wikipedia.org/wiki/Breusch%E2%80%93Pagan_test) was performed and the results show no evidence of non-constant variance of Full AVG at 5% significance (p-value = `r round(bp_test$p.value, 4)`).

#### Independence of Samples {#assume2}

The assumption of sample independence is valid in this case since it is reasonable to assume one player's performance did not affect the performance of another. 

#### Linearity and Multicollinearity {#assume3}

To assess linearity i.e. whether there was linear relationship between the predictors (March & April stats) and the response variable (Full AVG), their [correlation coefficients](https://en.wikipedia.org/wiki/Pearson_correlation_coefficient) were assessed. The values can be seen in the first column (or row) of Figure \@ref(fig:fig3) below. The colour of the cells represent the magnitude of correlation coefficient between the corresponding row and column variables. Negative values are represented by red and positive values are represented by green. 

The heat map shows Full AVG had significant (at 5%) correlation with March and April stats like batting average (AVG), hits (H), on-base percentage, slugging percentage (SLG), strikeout percentage (K%) and BABIP. This makes sense as these stats are either part of the batting average calculation or they use batting average in their calculation. Furthermore, it is known that the relationship between batting average and these stats are not linear e.g. [on-base percentage](https://library.fangraphs.com/offense/obp/) is related to batting average through hits and at bats. Therefore, the linearity assumption might not be valid in this case. 

The data also violated another assumption, namely [multicollinearity](https://en.wikipedia.org/wiki/Multicollinearity) i.e. when predictors are significantly correlated with each other. This violation is evident from the clusters of green and red cells in Figure \@ref(fig:fig3) and it makes sense as the stats are related to each other. Furthermore, they measure various aspects of the batters such as

- **Power** - slugging percentage, home runs, ISO, HRtoFB, FB%
- **Plate Discipline** - OContact%/ZContact%/Contact%, OSwing%/ZSwing%/Swing%, K%, BB%
- **Hit Tendency** - GB%, LD%, FB%, IFFB%, BABIP
- **Batting** - AVG, BABIP


```{r fig3, echo=FALSE, warning=FALSE, message = FALSE, fig.align="center", fig.cap="Pairwise Correlation Coefficient Heat Map", out.height="800px"}
# heatmap to summarize correlation matrix --------------------
# * tool tip --------------------
heat_tooltip_x <- c("Comparison", "Correlation", "P value")
heat_tooltip_y <- sprintf("{point.%s}", c("tip_lab", "crl", "pval"))
heat_tip <- highcharter::tooltip_table(x = heat_tooltip_x, y = heat_tooltip_y)

# * create plot --------------------
highcharter::hchart(heat_input, "heatmap",
                    highcharter::hcaes(x = param_x, y = param_y, value = crl),
                    borderColor = 'white', borderWidth = 1) %>%
  # highcharter::hc_chart(height = "575px") %>%
  # plot title
  highcharter::hc_title(text = "", style = list(fontSize = 0)) %>%
  # disable x axis label
  highcharter::hc_xAxis(title = list(text = "", style = list(fontSize = 0)), 
                        opposite = TRUE, tickWidth = 0,
                        labels = list(style = list(fontSize = "9px"))) %>%
  # disable y axis label
  highcharter::hc_yAxis(title = list(text = "", style = list(fontSize = 0)),
                        tickWidth = 0,
                        labels = list(style = list(fontSize = "9px"))) %>%
  # heatmap colour scale formatting
  highcharter::hc_colorAxis(min = -1, max = 1, labels = list(format = "{value}"), 
                            stops = highcharter::color_stops(n = 3, 
                                                             colors = c("#FF0000", "#FFFF99","#00B050"))) %>%
  # enable legend
  highcharter::hc_legend(title = list(text = "Correlation Coefficient", style = list(fontSize = "11px"))) %>%
  # tool tip format
  highcharter::hc_tooltip(useHTML = TRUE, pointFormat = heat_tip,
                          crosshairs = TRUE, shared = TRUE,
                          enabled = TRUE, headerFormat = "")
```

### Alternative Model {#altMod}

Because of the inherent relationship between the stats discussed above, *Dimension Reduction Methods (DRM)* discussed in `r c(james2013 = citet("10.1007/978-1-4614-7138-7"))`, namely *Principal Component Regression (PCR)* and *Partial Least Squares Regression (PLSR)*, appears to be a solution to the problem. PCR and PLSR work in the following fashion:

1. Derive a new (and usually smaller) set of latent predictors that are linearly related to the original predictors, i.e. March and April stats

2. Fit a linear regression model via least squares method using the transformed predictors

The key idea is that the smaller set of latent predictors is sufficient to explain the variability in the original predictor set and the relationship with the response variable, i.e. Full AVG. PCR and PLSR differs in the way they derive the latent predictors. PCR derives latent predictors that best describe the original predictors (via [Principal Component Analysis (PCA)](https://en.wikipedia.org/wiki/Principal_component_analysis)) without taking into account the response variable Full AVG. But as noted in `r c(james2013 = citet("10.1007/978-1-4614-7138-7"))`, this derivation of latent variables often won't be an issue. PLSR, on the other hand, derives the latent variables by considering both the original predictors and the response variable. 

It makes sense to use this approach because transforming the original predictors will lead to the reduction of correlation between the predictors. This will solve, or at the very least, alleviate the multicollinearity issue discussed in Section \@ref(assume3). Furthermore, the approach is suitable in the context of this data/problem because the predictors are the result of batters using their (unobservable) abilities in various aspect of the game. For example, Swing%/k% are the result of using plate discipline-related ability and home runs/slugging percentage are the result of using hit for power ability. Therefore, the latent variables can be view as the batters' abilities in different areas such as plate discipline and power. 

In fact, this is exactly the idea behind [Factor Analysis (FA)](https://en.wikipedia.org/wiki/Factor_analysis). FA is used in the derivation step of [Factor Regression Model (FRM)](https://en.wikipedia.org/wiki/Factor_regression_model) and FRM will also be considered in the analysis (motivated by this [article](https://medium.com/@jaynarayan94/multiple-linear-regression-factor-analysis-in-r-35a26a2575cc)).

## Method {#method}

### Model Selection
A key step in the DRM is to select the *optimal* number of latent variables; the model selection procedure will be as follows:

1. The data will be split into training/validation set (90/10 split).

2. The **final** model will be selected based on the lowest [Cross-Validation (CV)](https://en.wikipedia.org/wiki/Cross-validation_(statistics)) Root [Mean Squared Error (RMSE)](https://en.wikipedia.org/wiki/Mean_squared_error) (RMSE) using the raining set. Specifically Leave-one-out Cross-Validation (LOOCV) will be used.

3. Validation set will be used by the model with the lowest RMSE in each DRM (including **final** model) to examine the prediction RMSE.

## Results {#results}

### Dimension Reduction Method Models{#drmMod}

For each DRM, the number of latent variables were plotted against their CV RMSE from fitting a linear regression model (click on the legend icon to hide the corresponding DRM). 

```{r fig4, echo=FALSE, warning=FALSE, message = FALSE, fig.align="center", fig.cap="Cross-Validation Root Mean Squared Error", out.height="800px"}
# line plot to summarize cross-validation RMSE --------------------
mse_tooltip_x <- c("Model: ", "RMSE: ")
mse_tooltip_y <- sprintf("{point.%s}", c("mod_name", "CV"))
mse_tip <- highcharter::tooltip_table(x = mse_tooltip_x, y = mse_tooltip_y)

highcharter::hchart(pcr_cv_mse, "line",
                    name = "PCR", showInLegend = TRUE,
                    highcharter::hcaes(x = model, y = CV),
                                   color = "red") %>%
  highcharter::hc_add_series(pls_cv_mse, "line",
                             name = "PLSR", showInLegend = TRUE,
                             highcharter::hcaes(x = model, y = CV),
                             color = "blue") %>%
  highcharter::hc_add_series(fa_cv_mse1, "line",
                             name = "FA", showInLegend = TRUE,
                             highcharter::hcaes(x = model, y = CV),
                             color = "green") %>%
  # plot title
  highcharter::hc_title(text = "", style = list(fontSize = 0)) %>%
  # disable x axis label
  highcharter::hc_xAxis(title = list(text = "Number of Latent Variables", style = list(fontSize = 10))) %>%
  # disable y axis label
  highcharter::hc_yAxis(title = list(text = "Root Mean Squared Error (RMSE)", style = list(fontSize = 10))) %>%
  # disable legend
  highcharter::hc_legend(enable = TRUE) %>%
  # tool tip format
  highcharter::hc_tooltip(useHTML = TRUE, pointFormat = mse_tip,
                          crosshairs = TRUE, shared = FALSE,
                          enabled = TRUE, headerFormat = "")
```

Figure \@ref(fig:fig3) shows that 2 latent variables yields the lowest CV RMSE for PLSR (blue line). Furthermore, both PCR (red line) and FRM (green line) achieved lowest CV RMSE when the number of latent variables equals 5. 

#### Correlation Between Latent Variables {#drmModCor}
As discussed in Section \@ref(altMod), one advantage of the DRM is the ability to reduce correlations between predictors. The correlation coefficients matrices below show that this was achieved in all 3 cases as the correlations among the latent variables were either 0 or very close to 0. 

```{r echo=TRUE, warning=FALSE, message = FALSE, results = "markup"}
# Pairwise correlation between latent variables
# PCR with 5 latent variables
round(cor(pcr_best$scores), 5)

# PLSR with 2 latent variables
round(cor(pls_best$scores), 5)

# FRM with 5 latent variables
round(cor(fa_best$scores), 5)
```  

#### Variations Explained by Latent Variables {#drmModVar} 
Next, summarized below are the percentages of variance in the original predictors and Full AVG explained by the latent variables. The percentages represents the amount of information about the original predictors or Full AVG that was captured by the latent variables. The percent for Full AVG is also the $R^2$ from the linear regression fitted in Step 2 of DRM described in Section \@ref(altMod).

The latent variables in both PCR and FRM capture the majority of the information for the original predictors i.e. the stats from March and April. They explained around 75% and 68% of the variance, respectively. PLSR was able to capture the most information about Full AVG, i.e. $R^2$ at `r round(pls::R2(pls_best)$val[3], 4)` versus `r round(pls::R2(pcr_best)$val[6], 4)` for PCR and `r round(summary(fa_lm_best)$r.squared, 4)` for FRM.

```{r echo=TRUE, warning=FALSE, message = FALSE, results = "markup"}
# Percent of variance in Predictors/Response Explained
# PCR with 5 latent variables
summary(pcr_best)

# PLSR with 2 latent variables
summary(pls_best)

# Percent of variance in Predictorse Explained
# FRM with 5 latent variables
round(fa_best$Vaccounted[-1, ]*100, 2)
```  

#### Relationship between Original Predictors and Latent Variables {#drmModCor}
As explained in Section \@ref(altMod), DRM was suitable in this context because of how the stats are simply the results of players using various unobserved abilities (can be thought of as the latent variables) such as power and plate discipline. Below is a summary of correlation coefficient between the latent variables.

For PCR, Latent 1 from the best-fitted model was negatively correlated with all the *good* offense stats whereas Latent 2 correlated positively with all the power/swing stats such as HR, FB%, BB%, ISO, K% and all Swing%.

```{r echo=TRUE, warning=FALSE, message = FALSE, results = "markup"}
# PCR with 5 latent variables
pcr_load
```  

For the best fitted PLSR, Latent 1 was positively correlated with stats that are considered *good* while it was negatively correlated with K%, a *bad* stats. This suggests Latent 1 captured the impact of the players on offense. 

Latent 2, on the other hand, correlated positively with stats such as GB%, BABIP, AVG, and the Contact%. It was also negatively correlated with the power stats. It could be interpreted as the player's ability to make contact with the ball or hit for average.
```{r echo=TRUE, warning=FALSE, message = FALSE, results = "markup"}
# PLSR with 2 latent variables
pls_load
```  

In the best fitted FRM, it appeared Latent 1 captured the player's ability to hit for power as it correlated positively with the power stats such as HR, ISO, SLG and HRtoFB. 

Latent 3 to 5 captured the plate disciplines-related stats. Latent 3 and 4 were positively correlated with contact/hit stats whereas Latent 5 was correlated positively with swing stats. 
```{r echo=TRUE, warning=FALSE, message = FALSE, results = "markup"}
# FRM with 5 latent variables
fa_load
```  

### Final Model {#finalMod}

Below is the CV RMSE from models with the lowest CV RMSE within each DRM:

- **PCR with 5 latent variables**: `r round(min(pcr_cv_mse$CV), 5)`

- **PLSR with 2 latent variables**: `r round(min(pls_cv_mse$CV), 5)`

- **FRM with 5 latent variables**: `r round(min(fa_cv_mse$RMSE), 5)`

The PCR model with 5 latent variables had the CV RMSE, and hence it was selected as the **final** model.

### Prediction RMSE Assessment {#predErr}

Figure \@ref(fig:fig3) shows that 2 latent variables yields the lowest CV RMSE for PLSR (blue line). Furthermore, both PCR (red line) and FRM (green line) achieved lowest CV RMSE when the number of latent variables equals 5. 

Below is the plot of the predictions from the 3 models as well as the observed Full AVG for the players in the validation set (click on the legend icon to hide the corresponding line). The models were able to predict fairly well for players with Full AVG closer to the mean (`r round(mean(datt$FullSeason_AVG), 3)`). This makes sense as there was more data on batting averages closer to the mean. 

```{r fig5, echo=FALSE, warning=FALSE, message = FALSE, fig.align="center", fig.cap="DMR Predictions Using Validation Set", out.height="800px"}
# line plot to summarize cross-validation RMSE --------------------
pred_tooltip_x <- c("Player: ", "Observed Batting Average: ", "PCR Prediction: ", "PLSR Prediction: ", "FA Prediction: ")
pred_tooltip_y <- sprintf("{point.%s}", c("Name", "obs", "pcr", "pls", "fa"))
pred_tip <- highcharter::tooltip_table(x = pred_tooltip_x, y = pred_tooltip_y)

highcharter::hchart(pred_input, "line",
                    highcharter::hcaes(x = Name, y = obs),
                    name = "Observed<br>Batting Average", showInLegend = TRUE,
                    color = "black") %>%
  highcharter::hc_add_series(pred_input, "line",
                             highcharter::hcaes(x = Name, y = pcr),
                             name = "PCR Predictions", showInLegend = TRUE,
                             color = "red") %>%
  highcharter::hc_add_series(pred_input, "line",
                             highcharter::hcaes(x = Name, y = pls),
                             name = "PLSR Predictions", showInLegend = TRUE,
                             color = "blue") %>%
  highcharter::hc_add_series(pred_input, "line",
                             highcharter::hcaes(x = Name, y = fa),
                             name = "FA Predictions", showInLegend = TRUE,
                             color = "green") %>%
  # plot title
  highcharter::hc_title(text = "", style = list(fontSize = 0)) %>%
  # disable x axis label
  highcharter::hc_xAxis(title = list(text = "Player", style = list(fontSize = 10))) %>%
  # disable y axis label
  highcharter::hc_yAxis(title = list(text = "Batting Average", style = list(fontSize = 10))) %>%
  # disable legend
  highcharter::hc_legend(enabled = TRUE) %>%
  # tool tip format
  highcharter::hc_tooltip(useHTML = TRUE, pointFormat = pred_tip,
                          crosshairs = TRUE, shared = FALSE,
                          enabled = TRUE, headerFormat = "")
```  

Figure \@ref(fig:fig6) is another figure that summarizes the prediction error of the 3 models (click on the legend icon to hide the corresponding line). It show the distribution of the difference between the observed batting average and the predicted values from the 3 models using the validation set. The plot shows the prediction from FRM deviated more from the observed values while predictions from PCR and PLSR had similar deviations.  

```{r fig6, echo=FALSE, warning=FALSE, message = FALSE, fig.align="center", fig.cap="Distribution of Error as Percent of Observed Batting Average", out.height="800px"}
# distribution of prediction error --------------------
highcharter::highchart() %>%  
  highcharter::hc_chart(type=list('area')) %>%
  highcharter::hc_add_series(density(pred_input$pcr_err),
                             name = "PCR Prediction Error", showInLegend = TRUE,
                             fillOpacity = 0.1, color = "red") %>%
  highcharter::hc_add_series(density(pred_input$pls_err),
                             name = "PLSR Prediction Error", showInLegend = TRUE,
                             fillOpacity = 0.1, color = "blue") %>%
  highcharter::hc_add_series(density(pred_input$fa_err),
                             name = "FA Prediction Error", showInLegend = TRUE,
                             fillOpacity = 0.1, color = "green") %>%
  # plot title
  highcharter::hc_title(text = "", style = list(fontSize = 0)) %>%
  # disable x axis label
  highcharter::hc_xAxis(title = list(text = "Percent of Observed", style = list(fontSize = 10))) %>%
  # disable y axis label
  highcharter::hc_yAxis(title = list(text = "", style = list(fontSize = 0))) %>%
  # disable legend
  # disable tool tip
  highcharter::hc_tooltip(enabled = FALSE)
```  

In fact, the prediction RMSE of PCR with 5 latent variables (RMSE = 0.0008399) and PLSR with 2 latent variables (RMSE = 0.0008550) were similar. Prediction RMSE for FRM with 5 latent variables (RMSE = 0.0017583), on the other hand, was significantly higher than both PCR and PLSR.

## Discussion and Conclusion {#conclusion}

Although PCR was the "best" model based on the selection method employed, the model had a relatively poor fit ($R^2$ = `r round(pls::R2(pcr_best)$val[6], 4)`). This is perhaps because the linear relationship between the latent variable and the predictors/response variable assumed by the three methods was too restrictive. However, since the sample size is relatively small, perhaps by increasing the sample size, prediction accuracy and model fit would improve as well. Furthermore, non-linear or non-parametric dimension reduction methods suggested [here](https://www.analyticsvidhya.com/blog/2017/01/t-sne-implementation-r-python/) could also be explored for improvement in model prediction and fit.

## Appendix: Reference {#refs}

```{r echo=FALSE, warning=FALSE, message = FALSE,}
bibliography("text")
```

## Appendix: R Code {#code}

Below is the code used to perform the analyses:

```{r echo=TRUE, eval = FALSE, warning=FALSE, message = FALSE, results = "markup", tidy = TRUE}
# Batting Average Prediction ------------------------------------------------------------------
# Load package R ------------------------------
if(!require(pacman)) {install.packages("pacman"); library(pacman)}
pacman::p_load(tidyverse, magrittr, highcharter, pls, arrayhelpers, psych, ggcorrplot, lmtest, knitcitations)

# Load data ------------------------------
datt_orig <- read.csv("batting.csv", stringsAsFactors = F)

# * data for modeling --------------------
datt <- datt_orig %>%
  select(-playerid, -Name, -Team)
n_obs <- nrow(datt)

# Data exploration ------------------------------
# * shapiro-wilk test for normality --------------------
sh_test <- shapiro.test(datt_orig$FullSeason_AVG)

# * Breusch–Pagan test for constant variance --------------------
bp_test <- bptest(FullSeason_AVG~., data = datt)

# * full season average qq plot data --------------------
qq_input <- qqnorm(datt_orig$FullSeason_AVG, plot.it = FALSE, datax = TRUE)
qq_input <- data.frame(Name = datt_orig$Name,
                       x = qq_input$x,
                       y = qq_input$y,
                       stringsAsFactors = F)

# * correlation matrix --------------------
corr_input <- cor(datt_orig[, -c(1:3)])
# p values for correlations
corrP_input <- ggcorrplot::cor_pmat(datt_orig[, -c(1:3)])
corrP_input <- data.frame(corrP_input) %>%
  rownames_to_column("param_x") %>%
  gather(key = param_y, value = pval, -param_x) %>%
  # round correlation with 4 decimal places
  mutate(pval = round(pval, 4))

# ** input for correlation heat map --------------------
heat_lab <- colnames(datt)
heat_lab <- ifelse(startsWith(heat_lab, "MarApr_"),
                   str_remove_all(heat_lab, "MarApr_"),
                   str_replace_all(heat_lab, "Season_", " "))
heat_lab <- str_replace(heat_lab, "\\.", "%")

heat_input <- data.frame(corr_input) %>%
  rownames_to_column("param_x") %>%
  gather(key = param_y, value = crl, -param_x) %>%
  # round correlation with 4 decimal places
  mutate(crl = round(crl, 4)) %>%
  # combine p values
  left_join(corrP_input, by = c("param_x", "param_y")) %>%
  # create tool tip lavel
  mutate(tip_lab = paste0(param_x, " vs. ", param_y)) %>%
  # convert stats to factor for ordering purpose
  mutate(param_x = factor(param_x,
                          level = rev(colnames(datt)),
                          labels = rev(heat_lab)),
         param_y = factor(param_y,
                          level = colnames(datt),
                          labels = heat_lab))
# train & test split set up --------------------
set.seed(2019)
train <- sample(1:nrow(datt), nrow(datt)*0.9)

# training set
datt_train <- datt %>%
  dplyr::slice(train)

# test set
x_test <- datt %>%
  select(-FullSeason_AVG) %>%
  dplyr::slice(-train)
y_test <- datt$FullSeason_AVG[-train]

# Principal Components Regression --------------------
# * feature selection via LOOCV
pcr_cv <- pcr(FullSeason_AVG~., data = datt_train, validation ="LOO", scale=TRUE)

# * MSE for CV models --------------------
pcr_cv_mse <- array2df(RMSEP(pcr_cv, estimate = c("all"))$val) %>%
  set_colnames(c("mse", "estimate", "response", "model")) %>%
  # round mse to 5 decimal
  mutate(mse = round(mse, 5),
         # remove text in model
         model = factor(model, labels = 0:25)) %>%
  # convert data to wide
  spread(key = estimate, value = mse) %>%
  # model name
  mutate(mod_name = "Principal Components Regression")

# * best fitted model based on lowest RMSE --------------------
pcr_best <- pcr(FullSeason_AVG~., data = datt_train, scale=TRUE, ncomp = 5)

# * original predictors/latent variables correlations --------------------
pcr_load <- data.frame(round(pcr_best$loadings[1:25, ], 4))
pcr_load <- pcr_load %>%
  rownames_to_column("Stats") %>%
  # use cleaned name for stats
  mutate(Stats = heat_lab[-26]) %>%
  # only show if absolute value of correlation >= 0.2
  mutate_at(vars(Comp.1:Comp.5), function(x){ifelse(abs(x) < 0.15, "", x)}) %>%
  set_colnames(c("Stats", paste0("Latent ", 1:5)))

# * calculate MSE using best fitted model/test data -------------------------
pcr_pred <- predict(pcr_best, newdata = x_test, ncomp = 5, type = "response")
pcr_mse <- mean((pcr_pred - y_test)^2)

# Partial Least Squares --------------------
# * feature selection via LOOCV
pls_cv <- plsr(FullSeason_AVG~., data = datt_train, validation ="LOO", scale=TRUE)

# * MSE for CV models --------------------
pls_cv_mse <- array2df(RMSEP(pls_cv, estimate = c("all"))$val) %>%
  set_colnames(c("mse", "estimate", "response", "model")) %>%
  # round mse to 5 decimal
  mutate(mse = round(mse, 5),
         # remove text in model
         model = factor(model, labels = 0:25)) %>%
  # convert data to wide
  spread(key = estimate, value = mse) %>%
  # model name
  mutate(mod_name = "Partial Least Squares")

# * best fitted model based on lowest RMSE --------------------
pls_best <- plsr(FullSeason_AVG~., data = datt_train, scale=TRUE, ncomp = 2)

# * original predictors/latent variables correlations --------------------
pls_load <- data.frame(round(pls_best$loadings[1:25, ], 4))
pls_load <- pls_load %>%
  rownames_to_column("Stats") %>%
  # use cleaned name for stats
  mutate(Stats = heat_lab[-26]) %>%
  # only show if absolute value of correlation >= 0.2
  mutate_at(vars(Comp.1:Comp.2), function(x){ifelse(abs(x) < 0.15, "", x)}) %>%
  set_colnames(c("Stats", paste0("Latent ", 1:2)))

# * calculate MSE using best fitted model/test data -------------------------
pls_pred <- predict(pls_best, newdata = x_test, ncomp = 2, type = "response")
pls_mse <- mean((pls_pred - y_test)^2)

# Factor analysis regression --------------------
# * feature selection via LOOCV --------------------
# dataframe to store LOOCV results
fa_cv_mse <- data.frame(model = numeric(0), RMSE = numeric(0), Rsquared = numeric(0), MAE = numeric(0))

# perform LOOCV for 1 to 25 factors
for(i in 1:25){
fa_cv <- fa(r = datt_train[, -26], nfactors = i, rotate = "varimax", fm = "ml")

# training data for LOOCV
fa_train <- datt_train %>%
  select(FullSeason_AVG) %>%
  bind_cols(data.frame(fa_cv$scores))

# LOOCV
fa_lm_cv <- train(FullSeason_AVG ~., method = "lm", data = fa_train, trControl = trainControl(method = "LOOCV"))

# storing results
fa_cv_mse <- fa_cv_mse %>%
  bind_rows(c(model = i, fa_lm_cv$results[-1]))
}
fa_cv_mse1 <- fa_cv_mse %>%
  rename(CV = "RMSE") %>%
  # onvert model to factor
  mutate(model = factor(model),
         CV = round(CV, 5),
         mod_name = "Factor Regression Model")

# * best fitted model based on lowest RMSE --------------------
fa_best <- fa(r = datt_train[, -26], nfactors = which.min(fa_cv_mse$RMSE), rotate = "varimax", fm = "ml")
# * original predictors/latent variables correlations --------------------
fa_load <- data.frame(round(fa_best$loadings[1:25, ], 4))
fa_load <- fa_load %>%
  select(paste0("ML", 1:5)) %>%
  rownames_to_column("Stats") %>%
  # use cleaned name for stats
  mutate(Stats = heat_lab[-26]) %>%
  # only show if absolute value of correlation >= 0.2
  mutate_at(vars(ML1:ML5), function(x){ifelse(abs(x) < 0.15, "", x)}) %>%
  set_colnames(c("Stats", paste0("Latent ", 1:5)))

# data to fit linear regression
fa_train_best <- datt_train %>%
  select(FullSeason_AVG) %>%
  bind_cols(data.frame(fa_best$scores))
fa_lm_best <- lm(FullSeason_AVG ~ ., data = fa_train_best)

# * calculate MSE using best fitted model/test data -------------------------
fa_newdata <- data.frame(factor.scores(x_test, fa_best)$scores)
fa_pred <- predict(fa_lm_best, newdata = fa_newdata)
fa_mse <- mean((fa_pred - y_test)^2)

# ** input for predicted/observed vatting average --------------------
pred_input <- datt_orig %>%
  # test data only
  dplyr::slice(-train) %>%
  select(Name, FullSeason_AVG) %>%
  # combine with predictions from 3 models
  cbind(pcr_pred) %>%
  cbind(pls_pred) %>%
  cbind(fa_pred) %>%
  set_colnames(c("Name", "obs", "pcr", "pls", "fa")) %>%
  # round number to 3 decimal
  mutate_at(vars(-Name), round, digits = 3) %>%
  # add line break to name
  mutate(Name = str_replace_all(Name, " ", "<br>")) %>%
  # prediction error as % of observed value
  mutate(pcr_err = (pcr - obs),
         pls_err = (pls - obs),
         fa_err = (fa - obs)) %>%
  arrange(Name)
```
