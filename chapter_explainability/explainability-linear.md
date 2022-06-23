# Explainability of Linear Models

We have just used linear models to predict heart disease using various risk factors.
Linear models (:numref:`chap_linear`) describe a target variable (e.g., heart disease or not) in terms of a linear combination of features (e.g., age, resting blood pressure, and serum cholesterol). Models such as linear regression, lasso regression, logistic regression, and softmax regression are the popular variants of linear models. The linearity assumption limits linear models' expressiveness but makes them easy to understand. Although linear models are often not perfect, they are extensively adopted in a multitude of domains especially when people are concerned more about what drives models' behaviors than predictive accuracy. Correctly interpreting linear models is a critical step to understand models and is fundamental to more advanced explanation methods. 


## Revisiting Linear Models 

To get started, let's revisit two popular linear models: linear regression (for regression problems with numeric targets) and logistic regression (for binary classifications).

Suppose that a data sample consists of features (predictive variables) $x_1,...x_p$ and a ground-truth target $y$. In linear regression, the predicted output for this sample is formulated as

$$\hat{y} = b + \sum_{j=1}^p w_j x_j,$$
:eqlabel:`eq_xdl-linear-reg`

where $w_*$ represents the coefficient (weight) that describes the mathematical relationship between each feature and the predicted target, and $b$ is the intercept (bias) that can also be viewed as the coefficient multiplied by a constant feature of value $1$.

Logistic regression is useful for binary classification problems where the targets are dichotomic (e.g., absence or presence). It converts the output into probabilities using a logistic function:

$$P(\hat{y}=1) = \frac{1}{1 + \exp{(-(b + \sum_{j=1}^p w_j x_j))}}.$$
:eqlabel:`eq_logistic-regression`

Essentially, the parameters $w_*$ and $b$ of both models are learned by minimizing the discrepancies between predicted and actual target values. More details on regularization, optimization, and evaluation can be found in :numref:`chap_linear` and :numref:`chap_classification`.

```{.python .input}
from d2l import torch as d2l
import numpy as np
import pandas as pd
from sklearn import linear_model
from sklearn.metrics import mean_absolute_error
```

## Explaining Linear Models

To explain linear regression and logistic regression models, we will focus on coefficients and features in both models. You will soon realize that simply reading off coefficient values may cause issues.

### Interpreting Coefficients

Core to linear models, a coefficient quantifies the variation of the output when a given feature is varied (keeping all other features fixed). Interpreting the coefficients of linear regression models is straightforward. To explain, let's train ridge regression, a linear regression model using linear least squares as  loss and $L_2$ norm as regularization. We apply this model to predicting a patient's cholesterol level on the same heart disease dataset in :numref:`subsec_heart-disease-dataset`.

```{.python .input}
data = d2l.HeartDiseaseData(target='chol')
regr = linear_model.Ridge(solver='sag')
regr.fit(data.X_train, np.squeeze(data.y_train))
print(f'Mean absolute error: \
{mean_absolute_error(regr.predict(data.X_test),np.squeeze(data.y_test)):.5f}')
print(f'Intercept: {regr.intercept_:.2f}')
print(f'Coefficients:')
for i in range(len(regr.coef_)):
    print(f'{data.feat_col[i]:>8}: {regr.coef_[i]:.2f}')
```

Any of these coefficients tells us the conditional dependency between a specific feature and the target when all the other features remain constant. Take :eqref:`eq_xdl-linear-reg` as an example. On one hand, increasing feature $x_j$ by one unit leads to a change of $\hat{y}$ by $w_j$ when all the other features are fixed. If $w_j$ is positive, $\hat{y}$ will be increased by $|w_j|$; otherwise, it will be decreased by the same amount. Note that if $x_j$ is a categorical variable, changing $x_j$ by one unit may mean switching it from one category to the other (e.g., from female=0 to male=1), which depends on how categories are encoded. Moreover, if features were transformed (e.g., in the log scale) before training, such transformation details are needed for interpreting the coefficients precisely. On the other hand, the intercept $b$ anchors the regression line or surface in the right place: it can be interpreted as the predicted output when setting all the features to zero, such as a "typical" input with mean feature values on the standardized (:eqref:`eq_standardization`) data. However on the raw data, oftentimes the intercept does not have a meaningful interpretation, especially when $x_j$ (e.g., blood pressure) cannot be 0.

In the following example, we increase a patient's age by one year and check the influence on the predicted cholesterol level. The empirical result exactly supports our mathematical analysis.

```{.python .input}
index = data.feat_col.index('age')
patient = data.X_test[0]
patient_ageplus1 = patient.copy()
patient_ageplus1[index] = patient_ageplus1[index] + 1
print(f'Increasing age by 1 leads to increasing cholesterol level by \
{regr.predict([patient_ageplus1])[0] - regr.predict([patient])[0]:.2f}.')
```

### Feature Importance

In our example, the coefficients of the linear regression model took into account the natural units of the raw data.
For example, *age* is expressed in "living years" while *resting blood pressure* is in "mmHg". Using natural units is convenient for intuitive explanations, especially for tangible concepts such as weight, height, temperature, and age. Given input features in different scales, it is inappropriate to directly compare the coefficients to inspect the relative importance of features.

To make coefficients comparable, we need to ensure that features are unitless. 
To achieve this, we can certainly standardize the raw data and then retrain the model to reduce all the coefficients to the same unit of measure. As a simpler alternative, we can multiply the coefficients by the standard deviation of the  features, which is similar to feature standardization:

$$
\hat{y} = b+\sum_{j}^{p} w_j  x_j = b +\sum_{j}^{p} (w_j  \sigma_j) \cdot \left(\frac{x_j}{\sigma_j}\right).
$$

```{.python .input}
d2l.plot_hbar(data.feat_col, regr.coef_ * data.X_train.std(axis=0))
```

### Correlated Features

If two features depend on or associate with each another, we call them *correlated features* (also known as *multicollinearity* or *collinearity*). Correlated features do not bring additional information (or just very little). Instead, they might increase the model complexity and make it difficult to find a unique solution for linear models. Also, correlated features make the coefficient interpretation unreliable because they tend to change in unison. When features are correlated, changes in one are associated with variations in the other. The stronger the correlation, the more difficult it is to change one feature without changing the other. Therefore, for any two highly correlated features, practitioners may keep only one of them in the dataset.

As such, it is good practice to check the correlation between features in your dataset, which can be measured by the Pearson, Kendall, or Spearman correlation coefficient :cite:`asuero2006correlation`. Correlation can be negative, positive, or zero. A positive correlation signifies that both features move in the same direction, while a negative one means that they move in the opposite direction. A correlation of zero indicates no relationship between two features.

```{.python .input}
print(f"{data.df['age'].corr(data.df['chol'], method='pearson'):.2f}")
```

Some models are robust to correlated features. For instance, most tree-based models (e.g., decision trees and random forests) make no assumptions about the relationships between features. If two features are heavily correlated, one of them will be automatically ignored because splitting on it will have a low Gini or entropy value :cite:`Breiman.2001`. Some sophisticated post-hoc explanation methods such as SHAP :cite:`Lundberg.Lee.2017` can better deal with correlated features because it satisfies the *symmetry* property: if two tokens contribute equally to all possible coalitions, their contribution values are the same. We will cover it in more detail later.

### Odds Ratio

Interpretation of logistic regression is trickier than that of linear regression: the former has a nonlinear logistic function so coefficients do not influence output probability linearly. Since the logistic function is hard to interpret, we will seek for other explanation methods. First, let's re-train a logistic regression model for heart disease prediction.

```{.python .input}
data = d2l.HeartDiseaseData()
lr = linear_model.LogisticRegression(solver='sag', max_iter=20000)
lr.fit(data.X_train, np.squeeze(data.y_train))
print(f'Validation accuracy: \
    {lr.score(data.X_test, np.squeeze(data.y_test)):.5f}') 
```

Recall :eqref:`eq_logistic-regression` that logistic regression outputs probabilities. Let's define *odds* of an event to be the probability that the event occurs divided by the probability that the event does not occur. In fact, odds are widely used in areas such as gambling and medical statistics. According to :eqref:`eq_logistic-regression`, the odds of heart disease is


$$
\text{odds} = \frac{P(\hat{y}=1)}{1-P(\hat{y}=1) } = \exp \left(b + \sum_{j=1}^p w_j x_j\right).
$$

We will use *odds ratio* to explain logistic regression models, which is the ratio between two events: increasing $x_j$ by one unit vs. keeping it fixed:

$$\frac{\text{odds} (x_j+1)}{\text{odds}} = \frac{\exp (b + w_1 x_1 + \ldots+w_j (x_j+1)+\ldots+ w_p x_p)}{\exp (b + w_1 x_1 + \ldots+w_j x_j+\ldots+ w_p x_p)}  = \exp(w_j).
$$

Odds ratio interprets logistic regression handily: increasing $x_j$ by one unit will scale the odds ratio by a factor of $\exp(w_j)$. Let's compute them for our logistic regression model in heart disease prediction.

```{.python .input}
pd.DataFrame(list(zip(data.feat_col, np.around(np.exp(lr.coef_[0]), 3))), 
                  columns=['Feature', 'Odds Ratio']).transpose()
```

Now we can explain our logistic regression model using odds ratio for each feature.
For example, on one hand, an increase in a patient's age nearly does not change the odds of having heart disease vs. not having heart disease (the odds ratio is around $1$); on the other hand, for chest pain type (0: typical angina, 1: atypical angina, 2: non-anginal pain, 3: asymptomatic), changing the categorical type from "typical angina" to "atypical angina" will at least double the heart disease odds ratio (the odds ratio is over $2$). Note that such interpretations are on the model behavior and do not imply real-world causality.

## Summary
Linear models encompass a range of variants, such as ridge regression and logistic regression. Interpreting linear models is an essential yet non-trivial endeavor. We introduced how to correctly interpret two popular linear models: linear regression and logistic regression. When the underlying patterns are not linear, more complex models such as neural networks shall be used. In this case, linear models can still be used as surrogate models to explain the prediction of given instances :cite:`ribeiro2016should`. We will discuss them soon.


## Exercises

1. When can we view coefficients as feature importance?
1. How to interpret the feature *sex* in the logistic regression model?
1. Calculate the feature correlations between features for the heart disease dataset.