# Recipe Virality Analysis 

**Contributors:** Harshatha Prasanna and Jazely Tong

---

## Introduction

What makes a recipe explode in popularity and can we predict it before it happens?
 
This project analyzes data from [Food.com](https://www.food.com/), a large recipe-sharing platform with decades of user interactions. Our dataset contains **83,782 recipes** and **731,927 user interactions** (ratings and reviews) from 2008 to 2018.
 
We define a recipe as **"viral"** if it falls in the **top 10% of recipes by number of ratings received within its first 90 days** of it first being posted. This captures early momentum, separating such recipes recipes from the rest.

Our central question:
 
> **Can we predict whether a recipe will go viral using only information available within the first 30 days of its posting?**
 
This matters because it gives content platforms and creator a 60-day window to identify and amplify promising recipes before they peak.
 
The columns most relevant to our analysis are:
 
| Column | Description |
|---|---|
| `id` | Unique recipe identifier |
| `n_steps` | Number of steps in the recipe |
| `n_ingredients` | Number of ingredients |
| `minutes` | Total cook/prep time |
| `avg_rating` | Average user rating (computed from interactions) |
| `ratings_30_days` | Number of ratings received in the first 30 days (engineered) |
| `is_viral` | 1 if top 10% by 90-day ratings, else 0 (engineered) |
 
---

## Data Cleaning and EDA

### Cleaning Steps
 
1. **Left-merged** recipes with interactions on `recipe_id` to preserve all recipes even with no interactions.
2. **Replaced ratings of 0 with `NaN`** — Food.com uses 0 to indicate "no rating submitted" rather than a rating of 0.
3. **Computed `avg_rating`** per recipe using only non-missing ratings and merged it back onto the recipes dataset.
4. **Converted interaction dates to datetime** and sorted chronologically within each recipe.
5. **Computed `days_since_first`** — the number of days elapsed since each recipe's first-ever interaction.
6. **Defined viral label** using the top 10% threshold of 90-day rating counts (~10% of recipes labeled viral, ~90% non-viral).
 
### Missingness Analysis
 
About **6.4% of ratings are missing** (15,036 out of ~231,637 interactions). We ran two permutation tests to understand *why* ratings go missing:
 
**Test 1: Rating missingness vs. review missingness (p = 0.665):**
No significant relationship. Whether or not a user left a written review has nothing to do with whether they left a rating. The missingness here appears **Missing Completely At Random (MCAR)** with respect to reviews.
 
**Test 2: Rating missingness vs. n_steps (p = 0.0):**
Highly significant. Recipes with more steps are systematically more likely to have missing ratings. This suggests **Missing At Random (MAR)** based on recipe complexity, users may attempt complex recipes, interact with the page, but not finish or rate them. This is a meaningful finding since the absence of a rating is itself informative about recipe complexity.
 
### Univariate Analysis
 
**Distribution of 90-Day Ratings**
 
<iframe
  src="assets/ratings-90-days.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>
 
The distribution is heavily right-skewed as most recipes receive only 1–3 ratings in their first 90 days, while a small number receive far more. The log scale on the y-axis makes this visible. This confirms that viral recipes are genuine outliers, not just slightly above average.
 
**Distribution of Number of Ingredients**
 
<iframe
  src="assets/n-ingredients.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>
 
Ingredient counts are roughly bell-shaped, centered around 9–10 ingredients. The distribution is well-behaved with no extreme outliers, making it a clean feature for modeling.
 
### Bivariate Analysis
 
**30-Day Ratings: Viral vs. Non-Viral**
 
<iframe
  src="assets/boxplot-viral.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>
 
Viral recipes accumulate significantly more ratings in their first 30 days than non-viral ones. The difference in medians is extreme making this is the clearest visual signal that viral recipes tend to have 
different early engagement patterns.
 
**Number of Ingredients vs. Average Rating**
 
<iframe
  src="assets/scatter-ingredients-rating.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>
 
No meaningful relationship. Whether a recipe has 3 ingredients or 30, average ratings cluster tightly around 4.5–5.0. Complexity alone does not predict quality perception.
 
### Interesting Aggregate
 
We grouped recipes by viral status and computed the mean of three key features:
 
| is_viral | ratings_30_days | n_ingredients | avg_rating |
|---|---|---|---|
| 0 (Non-viral) | 1.0 | 9.28 | 4.60 |
| 1 (Viral) | 2.1 | 8.98 | 4.69 |
 
**The key takeaway:** Viral recipes aren't better recipes. They don't have more elaborate ingredient lists, and they're barely rated higher (4.69 vs 4.60 which is essentially negligible). What separates viral from non-viral is purely **early momentum**. A recipe that gets 2 ratings in its first 30 days is on a fundamentally different trajectory than one that gets 1.
 
---
 
## Hypothesis Testing

**Question:** Do viral recipes genuinely receive more ratings in their first 30 days, or could the observed difference be due to random chance?
 
- **Null Hypothesis (H₀):** Viral and non-viral recipes have the same mean 30-day rating count. Any observed difference is due to random chance.
- **Alternative Hypothesis (H₁):** Viral recipes receive more 30-day ratings on average than non-viral recipes.
- **Test Statistic:** Difference in group means (viral − non-viral)
- **Significance Level:** α = 0.05
- **Method:** Permutation test with 10,000 shuffles
 
**Permutation Test Results**
 
<iframe
  src="assets/permutation-test.html"
  width="900"
  height="500"
  frameborder="0"
></iframe>
 
| Metric | Value |
|---|---|
| Viral mean (30-day ratings) | 2.1034 |
| Non-viral mean (30-day ratings) | 1.0000 |
| Observed difference | 1.1034 |
| P-value | 0.0000 |
 
**Conclusion:** We reject H₀. Across 10,000 random permutations of the viral labels, not a single one produced a difference as large as what we actually observed. The early engagement gap between viral and non-viral recipes is statistically real.
 
---
 
## Framing the Prediction Problem
 
We frame this as a **binary classification** task:
 
> Given information available about a recipe within its first 30 days, predict whether it will be in the top 10% of recipes by 90-day engagement (i.e., `is_viral = 1`).
 
**Response variable:** `is_viral` (1 = viral, 0 = non-viral)
 
**Why this framing?** Predicting virality *before* it fully materializes gives a 60-day early warning window. All features used at prediction time are observable by day 30, therefore we are not using future information.
 
**Evaluation metric:** We prioritize **F1-score on the viral class** over overall accuracy. Since only ~10% of recipes are viral, a model that always predicts "non-viral" would achieve 90% accuracy while being despite not giving us any useful information. F1-score balances precision and recall on the minority class we actually care about identifying.

---

## Baseline Model
Our baseline uses **Logistic Regression** with three features inside a `sklearn` Pipeline:
 
| Feature | Type | Description |
|---|---|---|
| `n_steps` | Quantitative | Number of recipe steps |
| `n_ingredients` | Quantitative | Number of ingredients |
| `ratings_30_days` | Quantitative | Ratings received in first 30 days |
 
All three features are quantitative and continuous. No ordinal or nominal encoding was required. Features were standardized using `StandardScaler` before fitting.
 
**Baseline Results:**
 
| Metric | Non-Viral (0) | Viral (1) |
|---|---|---|
| Precision | 0.91 | 1.00 |
| Recall | 1.00 | 0.68 |
| F1-Score | 0.95 | 0.81 |
| **Overall Accuracy** | | **0.9256** |
 
The baseline model already achieves a viral-class F1 of **0.81** and a precision of **1.00**, meaning that every recipe it labels as viral actually is viral. However when looking at recall, it misses about 32% of truly viral recipes. This is a strong baseline, suggesting the early engagement signal (`ratings_30_days`) is highly informative on its own.
 
---
 
## Final Model and Conclusions

For the final model, we switched to a **Random Forest classifier** and added two new engineered features:
 
| New Feature | Justification |
|---|---|
| `minutes` | Cook time may affect engagement, quick recipes get tried (and rated) faster |
| `avg_rating` | Quality signal; highly-rated recipes may get recommended more aggressively |
 
**Hyperparameter Tuning** was performed using `GridSearchCV` with 5-fold cross-validation, optimizing for F1-score on the viral class:
 
| Parameter | Values Tried | Best |
|---|---|---|
| `n_estimators` | 100, 200 | 100 |
| `max_depth` | 5, 10, None | 5 |
| `min_samples_split` | 2, 5, 10 | 2 |
 
**Final Model Results:**
 
| Metric | Non-Viral (0) | Viral (1) |
|---|---|---|
| Precision | 0.91 | 1.00 |
| Recall | 1.00 | 0.68 |
| F1-Score | 0.95 | 0.81 |
| **Overall Accuracy** | | **0.9256** |
 
**Result Analysis:** The final model's scores are identical to the baseline. This is a meaningful finding in itself, suggesting that the `ratings_30_days` feature is so dominant that once it's included, additional features and a more complex model do not provide measurable improvement. Furthermore, the Random Forest confirms the baseline wasn't underfitting.
 
> A precision of 1.00 on the viral class means: **when our model says a recipe will go viral, it has never been wrong.** The tradeoff is that the model tends to be convervative, it would rather miss a viral recipe than falsely flag a non-viral one.
 
---

## Fairness Analysis
 
**Question:** Does our model perform equally well on "quick" recipes (≤ 35 minutes) vs. "long" recipes (> 35 minutes)?
 
- **Group X:** Quick recipes (cook time ≤ 35 minutes, the median)
- **Group Y:** Long recipes (cook time > 35 minutes)
- **Metric:** Recall on the viral class
- **Null Hypothesis:** The model is fair, any difference in recall between quick and long recipes is due to chance
- **Alternative Hypothesis:** The model is unfair, quick recipes have higher viral recall than long recipes
- **Test Statistic:** Difference in recall (quick − long)
- **Significance Level:** α = 0.05
 
| Metric | Value |
|---|---|
| Quick recipe recall | 0.6878 |
| Long recipe recall | 0.6801 |
| Observed difference | 0.0078 |
| P-value | 0.669 |
 
**Conclusion:** We fail to reject H₀. The tiny 0.78% difference in recall between quick and long recipes is well within the range of random variation. Our model does not appear to systematically disadvantage longer recipes.
 
---

## Discussion

This project set out to answer one question: **can we predict whether a recipe will go viral before it actually does?** 

We found that virality has almost nothing to do with how good a recipe is. Viral and non-viral recipes are nearly identical in average rating (4.69 vs 4.60) and ingredient complexity (8.98 vs 9.28 ingredients). What separates them is purely **early momentum**, a recipe that gets 2 ratings in its first 30 days is on a fundamentally different trajectory than one that gets 1.

**The big picture takeaway:** On Food.com, recipes don't go viral because they're better, they go viral because they get noticed early. If you want to predict the next viral recipe, watch the first 30 days.

---
 
*Harshatha Prasanna & Jazely Tong — DSC 80, UC San Diego*
