# Corn Yield Response to Nitrogen Fertilizer: An Experimental Data Analysis

## Project Overview

This project investigates the effect of nitrogen fertilizer on corn yield using data from a historical agricultural field experiment contained in the `agridat` R package. The objective was to determine whether nitrogen fertilizer significantly increases crop yield and to explore the relationship between fertilizer application rate and crop productivity.

The project combines classical statistical inference with regression modeling and follows a workflow similar to that used in modern A/B testing studies.

---

## Dataset Description

The dataset originates from agricultural field trials reported by Heady et al. and contains measurements of crop yield under different fertilizer combinations.

The original dataset contains the following variables:

| Variable | Description                  |
| -------- | ---------------------------- |
| crop     | Crop type                    |
| rep      | Replicate number             |
| N        | Nitrogen applied (lb/acre)   |
| P        | Phosphorus applied (lb/acre) |
| K        | Potassium applied (lb/acre)  |
| yield    | Observed crop yield          |

Four crop categories were present in the dataset:

* Corn
* Corn (second-year residual effects)
* Alfalfa
* Clover

Since yield units differ across crops, only the **corn** observations were used for the analysis to ensure meaningful comparisons.

---

## Data Preparation

The dataset contained missing yield values corresponding to treatment combinations that were not observed in the original experimental design.

Rows with missing yield values were removed before analysis.

After filtering:

* Total observations used: 456
* Corn observations used for analysis: 114

---

# Part I: A/B Testing Analysis

## Research Question

Does nitrogen fertilizer significantly increase corn yield?

### Control Group

Plots receiving no nitrogen:

N = 0

### Treatment Group

Plots receiving any amount of nitrogen:

N > 0

### Outcome Variable

Corn yield

---

## Exploratory Data Analysis

The average corn yield for each group was:

| Group            | Mean Yield |
| ---------------- | ---------- |
| No Nitrogen      | 26.65      |
| Nitrogen Applied | 97.21      |

Average yield increased substantially in plots receiving nitrogen fertilizer.

The difference in means was:

70.56 yield units

This corresponds to an increase of approximately 265%.

Boxplots and histograms showed a clear separation between treated and untreated plots, suggesting a strong treatment effect.

---

## Hypothesis Testing

### Null Hypothesis

H0: Nitrogen fertilizer does not affect average corn yield.

### Alternative Hypothesis

H1: Nitrogen fertilizer increases average corn yield.

A Welch two-sample t-test was used because the treatment and control groups had unequal sample sizes and potentially different variances.

### Results

t-statistic = 14.20

p-value = 7.10 × 10^-26

### Interpretation

The p-value is substantially below conventional significance thresholds (0.05, 0.01, and 0.001).

Therefore, the null hypothesis is rejected.

The data provide extremely strong evidence that nitrogen fertilizer increases corn yield.

---

## Confidence Interval

A 95% confidence interval was calculated for the difference in average yield between treatment and control groups.

95% Confidence Interval:

[50.89, 90.23]

### Interpretation

Based on the experimental data, the average yield increase associated with nitrogen fertilizer is estimated to lie between approximately 50.9 and 90.2 yield units.

Since the interval does not include zero, the evidence supports a positive treatment effect.

---

## Effect Size

Statistical significance alone does not indicate how large an effect is.

To quantify practical significance, Cohen's d was calculated.

Cohen's d = 1.83

### Interpretation

According to conventional guidelines:

* 0.2 = Small effect
* 0.5 = Medium effect
* 0.8 = Large effect

A value of 1.83 indicates an exceptionally large effect size.

This suggests that the increase in yield is not only statistically significant but also agriculturally meaningful.

---

# Part II: Dose-Response Analysis

After establishing that nitrogen fertilizer increases yield, the next question is:

How does yield change as nitrogen application increases?

Average yield at each nitrogen level was computed.

| Nitrogen (lb/acre) | Mean Yield |
| ------------------ | ---------- |
| 0                  | 26.65      |
| 40                 | 60.66      |
| 80                 | 86.63      |
| 120                | 103.59     |
| 160                | 104.09     |
| 200                | 97.66      |
| 240                | 101.79     |
| 280                | 106.03     |
| 320                | 105.27     |

The relationship revealed two distinct regions:

1. Rapid yield growth at low nitrogen levels.
2. A plateau at higher nitrogen levels.

This pattern is consistent with the agricultural concept of diminishing returns.

---

# Part III: Regression Modeling

## Linear Regression Using Nitrogen Only

A simple linear regression model was fit using nitrogen as the sole predictor.

Model:

Yield = a + bN

Results:

R² = 0.254

### Interpretation

Nitrogen alone explains approximately 25% of the observed variation in corn yield.

While nitrogen clearly influences yield, other factors also contribute substantially.

---

## Multiple Linear Regression

A multiple regression model was fit using nitrogen, phosphorus, and potassium.

Model:

Yield = a + bN + cP + dK

Results:

R² = 0.524

Coefficients:

* Nitrogen: 0.208
* Phosphorus: 0.219
* Potassium: 0.000

### Interpretation

Including phosphorus substantially improved predictive performance.

The increase in R² from 0.254 to 0.524 indicates that fertilizer response cannot be explained by nitrogen alone.

---

## Polynomial Regression

The dose-response curve suggested a nonlinear relationship between nitrogen and yield.

A quadratic regression model was fit:

Yield = a + bN + cN²

Results:

R² = 0.362

The quadratic coefficient was negative, indicating diminishing marginal returns at higher nitrogen levels.

This model performed better than a simple linear model, supporting the hypothesis that yield response is nonlinear.

---

# Key Findings

1. Nitrogen fertilizer significantly increases corn yield.
2. The average increase in yield was approximately 70.6 units.
3. The treatment effect was highly statistically significant (p ≈ 7 × 10^-26).
4. The estimated effect size was extremely large (Cohen's d = 1.83).
5. Yield increases rapidly at low nitrogen levels and plateaus at higher levels.
6. Nitrogen alone explains only part of the yield variation.
7. Including phosphorus improves predictive performance considerably.
8. A nonlinear model better captures fertilizer response than a simple linear model.

---

# Conclusion

This analysis demonstrates the value of combining experimental design, statistical inference, and predictive modeling to understand agricultural productivity.

The A/B testing framework established a strong causal relationship between nitrogen fertilizer and increased corn yield. Subsequent regression analyses showed that fertilizer response exhibits diminishing returns and that multiple nutrients contribute to crop performance.

Overall, the results suggest that nitrogen fertilizer is highly effective for increasing corn yield, although the benefit of additional nitrogen decreases at higher application rates.
