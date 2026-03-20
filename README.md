# Recipe Virality Analysis 

*By Harshatha Prasanna and Jazely Tong*

---

## Introduction

What makes a recipe explode in popularity and can we predict it before it happens?
 
This project analyzes data from [Food.com](https://www.food.com/), a large recipe-sharing platform with decades of user interactions. Our dataset contains **83,782 recipes** and **731,927 user interactions** (ratings and reviews) from 2008 to 2018.
 
We define a recipe as **"viral"** using a threshold based on the **90th percentile of the number of ratings received within a recipe’s first 90 days** after its first rating. This captures early momentum and separates recipes with unusually strong early engagement from the rest.

Our central question:
 
> **Can we predict whether a recipe will go viral using only information available within the first 30 days of its posting?**
 
This matters because it gives content platforms and creators a window to identify and amplify promising recipes before they peak.
 
The columns most relevant to our analysis are:
 
| Column | Description |
|---|---|
| `id` | Unique recipe identifier |
| `n_steps` | Number of steps in the recipe |
| `n_ingredients` | Number of ingredients |
| `minutes` | Total cook/prep time |
| `avg_rating` | Average user rating (computed from interactions) |
| `ratings_30_days` | Number of ratings received in the first 30 days (engineered) |
| `is_viral` | 1 if a recipe meets the viral threshold based on 90-day ratings, else 0 (engineered) |
 
---

## Data Cleaning and Exploratory Data Analysis

### Cleaning Steps
 
1. **Left-merged** recipes with interactions on `recipe_id` to preserve all recipes even with no interactions.
2. **Replaced ratings of 0 with `NaN`**: Food.com uses 0 to indicate "no rating submitted" rather than a real rating of 0.
3. **Computed `avg_rating`** per recipe using only non-missing ratings and merged it back onto the recipes dataset.
4. **Converted interaction dates to datetime** and sorted chronologically within each recipe.
5. **Computed `days_since_first`**: the number of days elapsed since each recipe's first-ever interaction.
6. **Defined viral label** using the 90th percentile threshold of 90-day rating counts.

### Head of Cleaned DataFrame

The table below shows the head of our cleaned recipe-level DataFrame after merging in the average rating per recipe.

| id | name | minutes | n_steps | n_ingredients | avg_rating |
|---:|---|---:|---:|---:|---:|
| 333281 | 1 brownies in the world best ever | 40 | 10 | 9 | 4.0 |
| 453467 | 1 in canada chocolate chip cookies | 45 | 12 | 11 | 5.0 |
| 306168 | 412 broccoli casserole | 40 | 6 | 9 | 5.0 |
| 286009 | millionaire pound cake | 120 | 7 | 7 | 5.0 |
| 475785 | 2000 meatloaf | 90 | 17 | 13 | 5.0 |
 
 
### Univariate Analysis

After cleaning and preparing the dataset, we next explore the distributions of key variables to better understand the structure and variability in the data. In particular, we focus on how recipe popularity behaves in the first 90 days.

 
**Distribution of 90-Day Ratings**
 
<iframe
  src="assets/ratings-90-days.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>
 
The distribution is heavily right-skewed. Most recipes receive very few ratings in their first 90 days, while a smaller group receives substantially more. The log scale on the y-axis makes the long right tail easier to see, showing that highly popular recipes are outliers rather than typical cases.
 
**Distribution of Number of Ingredients**
 
<iframe
  src="assets/n-ingredients.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>
 
Ingredient counts are concentrated around the middle of the distribution, with most recipes using a moderate number of ingredients. This makes `n_ingredients` a straightforward structural feature to compare across recipes.
 
### Bivariate Analysis
 
**30-Day Ratings: Viral vs. Non-Viral**
 
<iframe
  src="assets/boxplot-viral.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>
 
Viral recipes accumulate more ratings in their first 30 days than non-viral recipes. The difference is clear in the distribution, suggesting that early engagement is strongly related to eventual virality.
 
**Number of Ingredients vs. Average Rating**
 
<iframe
  src="assets/scatter-ingredients-rating.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>
 
There is no strong visible relationship between number of ingredients and average rating. Ratings stay clustered in a fairly narrow range even as ingredient count changes.
 
### Interesting Aggregate
 
We grouped recipes by viral status and computed the mean of three key features:
 
| is_viral | ratings_30_days | n_ingredients | avg_rating |
|---|---:|---:|---:|
| 0 | 1.00 | 9.28 | 4.60 |
| 1 | 2.10 | 8.98 | 4.69 |
 
**The key takeaway:** Viral and non-viral recipes look fairly similar in ingredient count and average rating. The biggest difference is early engagement. Recipes labeled viral receive more ratings in the first 30 days, which supports the idea that early momentum is a major signal of later popularity.
 
---
 
## Missingness Analysis

About **6.4% of ratings are missing**. We ran two permutation tests to understand *why* ratings go missing:
 
**Test 1: Rating missingness vs. review missingness (p = 0.618):**  
No significant relationship. Whether or not a user left a written review does not appear to be associated with whether they left a rating.
 
**Test 2: Rating missingness vs. n_steps (p = 0.0):**  
Highly significant. Recipes with different numbers of steps show different missingness patterns in ratings. This suggests rating missingness depends on recipe complexity-related information. Since the p-value is very small, we reject the null hypothesis. This provides strong evidence that rating missingness depends on the number of steps in a recipe. Recipes with different numbers of steps have different missingness patterns.

<iframe
  src="assets/steps-missing.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

Although our results suggest the missingness is related to n_steps (which supports MAR), 
it is still possible that the rating column is MNAR (Missing Not At Random). 
For example, users might be less likely to leave a rating if they had a very bad or very good 
experience, meaning the missingness depends on the actual rating value that is missing. 
To better understand this, it would help to have more data on user satisfaction signals, 
which could explain the missingness using observed variables instead.
--- 

## Hypothesis Testing

**Question:** Do viral recipes genuinely receive more ratings in their first 30 days, or could the observed difference be due to random chance?
 
- **Null Hypothesis (H₀):** Viral and non-viral recipes have the same mean 30-day rating count. Any observed difference is due to random chance.
- **Alternative Hypothesis (H₁):** Viral recipes receive more 30-day ratings on average than non-viral recipes.
- **Test Statistic:** Difference in group means (viral − non-viral)
- **Significance Level:** α = 0.05
- **Method:** Permutation test with 10,000 shuffles

We use a permutation test because it does not rely on distributional assumptions and is appropriate for comparing group means. The difference in means is a natural choice of test statistic since our question focuses on whether early engagement differs between viral and non-viral recipes.

**Permutation Test Results**
 
<iframe
  src="assets/permutation-test.html"
  width="900"
  height="500"
  frameborder="0"
></iframe>
 
| Metric | Value |
|---|---:|
| Viral mean (30-day ratings) | 2.1034 |
| Non-viral mean (30-day ratings) | 1.0000 |
| Observed difference | 1.1034 |
| P-value | 0.0000 |
 
**Conclusion:** We reject H₀. There is strong statistical evidence to suggest that viral recipes tend to receive more ratings in their first 30 days than non-viral recipes, and that this difference is unlikely to be due to random chance.
 
---
 
## Framing the Prediction Problem
 
We frame this as a **binary classification** task:
 
> Given information available about a recipe within its first 30 days, predict whether it will be labeled `is_viral = 1`.
 
**Response variable:** `is_viral` (1 = viral, 0 = non-viral)
 
**Why this framing?** Predicting virality before it fully materializes helps identify recipes with strong future potential. All features used at prediction time are available by day 30, so we avoid using information from later periods.

For example, features such as the number of ratings, number of reviews, and engagement metrics within the first 30 days are used, while any information beyond this time frame is excluded.
 
**Evaluation metric:** We prioritize **F1-score on the viral class** over overall accuracy. Since the classes are imbalanced, accuracy alone could be misleading. F1-score better balances precision and recall for the group we care most about identifying.

---

## Baseline Model

Our baseline model uses **Logistic Regression** inside an `sklearn` Pipeline with three simple recipe-level features:

| Feature | Type | Description |
|---|---|---|
| `n_steps` | Quantitative | Number of recipe steps |
| `n_ingredients` | Quantitative | Number of ingredients |
| `minutes` | Quantitative | Total cook/prep time |

All three features are quantitative, so no categorical encoding was required. We standardized the features using `StandardScaler` before fitting the model.

**Baseline Results:**

| Metric | Non-Viral (0) | Viral (1) |
|---|---:|---:|
| Precision | 0.78 | 0.25 |
| Recall | 0.45 | 0.59 |
| F1-Score | 0.57 | 0.35 |
| **Overall Accuracy** |  | **0.4826** |


This baseline model performs poorly overall, especially on the viral class, where the F1-score is only **0.3510**. This is expected, since the model only uses simple recipe structure features and does not yet include early engagement information.
These results are evaluated on a held-out test set, allowing us to assess how well the model generalizes to unseen data. 
---

## Final Model and Conclusions

For the final model, we switched to a **Random Forest classifier** and added new features beyond the baseline:

| New Feature | Justification |
|---|---|
| `ratings_30_days` | Captures early momentum, which earlier analysis showed is strongly related to virality |
| `steps_per_ingredient` | Captures recipe complexity relative to ingredient count |
| `log_minutes` | Reduces skew in preparation time and may help the model handle wide variation in cooking time |

The final model used the following features:
- `n_steps`
- `n_ingredients`
- `minutes`
- `ratings_30_days`
- `steps_per_ingredient`
- `log_minutes`

All features used in the final model are quantitative, so no categorical encoding was required.

**Hyperparameter Tuning** was performed using `GridSearchCV` with 5-fold cross-validation, optimizing for F1-score on the viral class:

| Parameter | Values Tried | Best |
|---|---|---|
| `n_estimators` | 100, 200 | 100 |
| `max_depth` | 5, 10, None | 10 |
| `min_samples_split` | 2, 5, 10 | 5 |

**Final Model Results:**

| Metric | Non-Viral (0) | Viral (1) |
|---|---:|---:|
| Precision | 0.91 | 1.00 |
| Recall | 1.00 | 0.68 |
| F1-Score | 0.95 | 0.81 |
| **Overall Accuracy** |  | **0.9255** |

**Result Analysis:** The final model substantially improves on the baseline. The viral-class F1-score rises from **0.3510** in the baseline model to **0.8125** in the final model, while overall accuracy increases from **0.4826** to **0.9255**. This large improvement shows that early engagement information, especially `ratings_30_days`, is much more informative for predicting recipe virality than static recipe characteristics alone.

The final model performance is evaluated on a held-out test set, ensuring that the improvements reflect better generalization rather than overfitting.

> A precision of 1.00 on the viral class means that in this test set, every recipe predicted to be viral actually was viral. The tradeoff is that the model is conservative, so it misses some viral recipes instead of falsely labeling non-viral recipes as viral.

---

## Fairness Analysis

**Question:** Does our model perform equally well on "quick" recipes (≤ 35 minutes) versus "long" recipes (> 35 minutes)?

- **Group X:** Quick recipes (cook time ≤ 35 minutes, the median)
- **Group Y:** Long recipes (cook time > 35 minutes)
- **Metric:** Recall on the viral class
- **Null Hypothesis:** The model is fair. Any difference in viral recall between quick and long recipes is due to random chance.
- **Alternative Hypothesis:** The model is unfair. Its viral recall is lower for long recipes than for quick recipes.
- **Test Statistic:** Difference in recall (quick − long)
- **Significance Level:** α = 0.05

| Metric | Value |
|---|---:|
| Quick recipe recall | 0.6878 |
| Long recipe recall | 0.6801 |
| Observed difference | 0.0078 |
| P-value | 0.331 |

**Conclusion:** We fail to reject H₀. The observed difference in viral recall between quick and long recipes is small and consistent with random chance. Based on this test, we do not find evidence that the model performs worse on long recipes than on quick recipes.
---

## Discussion

This project set out to answer one question: **can we predict whether a recipe will go viral before it actually does?**

We found that virality appears to have little relationship with average rating. Viral and non-viral recipes are nearly identical in average rating (4.69 vs 4.60) and ingredient complexity (8.98 vs 9.28 ingredients). What separates them is purely **early momentum**, a recipe that gets 2 ratings in its first 30 days is on a fundamentally different trajectory than one that gets 1.

**The big picture takeaway:** On Food.com, recipes don't go viral because they're better, they go viral because they get noticed early. If you want to predict the next viral recipe, watch the first 30 days.

---
 
*Harshatha Prasanna & Jazely Tong — DSC 80, UC San Diego*
