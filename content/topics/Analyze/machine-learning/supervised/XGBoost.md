---
title: "Extreme gradient boosting with XGBoost in R"
description: "Explanation and usage of XGBoost in R"
keywords: "XGBoost, Extreme Gradient Boosting, Machine Learning, R, decision tree, model "
draft: false
weight: 2
author: "Niels Rahder"
authorlink: "https://tilburgsciencehub.com/contributors/nielsrahder/" 
---

## Overview

In this building block, you will delve deep into the inner workings of XGBoost, a top-tier machine learning algorithm known for its efficiency and effectiveness.

Navigating through this guide, you will gain hands-on experience and broaden your understanding of:

- The 'under the hood' process of how XGboost works.
- How to use XGBoost in `R`, what the hyperparameters are, and how to tune them using a grid search. 
- How to interpret the results. 
- How to visualize the algorithm. 

Step into the world of XGBoost, where you'll learn to build, fine-tune, and interpret highly reliable models, ready to make impactful predictions across various industries.

## What is XGBoost?

Extreme gradient boosting is an highly effective and widely used machine learning algorithm developed by [Chen and Guestrin 2016](https://dl.acm.org/doi/10.1145/2939672.2939785). The algorithm works by iteratively building a collection of decision trees, where newer trees are used to correct the errors made by previous trees (think of it as taking small steps in order to come closer to the "ultimate truth"). Additionally, the algorithm also makes use of regularization (penalizing complexity of individual trees, and the total number of trees in the collection) to prevent overfitting. 

This method has proven to be effective in working with large and complex datasets, and is used to address a wide range of problems (both classification and regression) (for example, at the KDDCup in 2015, XGBoost was used by every team ranked in the top 10!).

### How does XGBoost work?

The algorithm starts by calculating the residual values for each data point based on an initial estimate. For instance, given the variables `age` and `degree`, we compute the residual values relative to `salary`, for which the value `49` will serve as our initial estimation:


  
  | **Salary** | **age** | **degree** | **Residual** |
  | ---------- | ------- | --------- | -----------   |
  |     40    |     20   |    yes    |     -9      | 
  |     46     |    28   |    yes    |    -3       |
  |     50     |    31   |    no     |     1       | 
  |     54     |   34    |    yes    |     5       |
  |     61     |   38    |    no     |     12      |
  

Next, the algorithm calculates the similarity score for the entire tree and the individual splits, especially focusing on an arbitrarily chosen mean value of 24 (the mean between the first two age values) using the following formula:  

 
<div style="text-align: center;">
{{<katex>}}
\text{Similarity score} = \frac{{(\sum \text{Residual})^2}}{{\text{\# of Residual} + \lambda}}
{{</katex>}}
</div>

<br>

<div style="text-align: center;">
{{<katex>}}
\text{Similarity score root node} (\lambda = 1) = \frac{(-9 - 3 + 1 + 5 + 12)^2}{6} = \frac {36}{6} = 6 
{{</katex>}}
</div>

<br>

<div style="text-align: center;">
{{<katex>}}
\text{Similarity score left node} (\lambda = 1) = \frac{(-9 )^2}{2} = 40.5 
{{</katex>}}
</div>

<br>

<div style="text-align: center;">
{{<katex>}}
\text{Similarity score right node} (\lambda = 1) = \frac{(- 3 + 1 + 5 + 12 )^2}{5} = 112.5
{{</katex>}}
</div>

<br>

{{% warning %}}
λ is a regularization parameter which is used to prune the tree. For now, we set it to the default value of 1
{{% /warning %}}

Following the calculation of the individual similarity scores, the gain is calculated as:

<div style="text-align: center;">
{{<katex>}}
\text{Gain} = \text{sim(left node)} + \text{sim(right node)} - \text{sim(root node)}
{{</katex>}}
</div>

<br>

<div style="text-align: center;">
{{<katex>}}
\text{Gain} = 40.5 + 112.5 - 6 = 147
{{</katex>}}
</div>

<br>
 
The algorithm then compares this value to gain-values generated by other split options within the dataset to determine which division optimally enhances the separation of classes. For example, when we choose the split value of 29.5 (the mean value between the second and third ages), the similarity score of the root node remains unchanged, while the similarity scores of the left and right nodes change to:

<div style="text-align: center;">
{{<katex>}}
\text{Similarity score left node} ( \lambda = 1) = \frac{(-9 -3)^2}{3} = 72
{{</katex>}}
</div>

<br>

<div style="text-align: center;">
{{<katex>}}
\text{Similarity score right node} ( \lambda = 1) = \frac{(1  + 5 + 12)^2}{4} = 162
{{</katex>}}
</div>




<br>


The gain-value therefore becomes:
<br> 

<div style="text-align: center;">
{{<katex>}}
\text{Gain} = {72 +  162 - 6 } =228
{{</katex>}}
</div>

<br> 

Which is a higher value than that of the split based on our initial values, and is therefore used as the more favorable option. The same process is then repeated for all possible split combinations in order to find the best option. After this process is completed for all possible split options the final tree (with the residual values) becomes:

<p align = "center">
<img src = "../images/tree_structure.png" width="400">
</p>

The ouput value for each datapoint is then calculated via the following formula: 

<div style="text-align: center;">
{{<katex>}}
\text{Output value} = {\text{initial estimate} +  \alpha * \text{output of tree} } 
{{</katex>}}
</div>




<br> 

<div style="text-align: center;">
{{<katex>}}
\text{Output value (age = 20)} = {49 +  0.3 * -6 } = 47.2
{{</katex>}}
</div>

{{% warning %}}
ɑ is the learning rate of the model, for now we set it to the default value of 0.3 
{{% /warning %}}


From this we can see that our output value for the average salary of a 20 year old has decreased from the initial prediction of 49 to our new prediction of 47.2, which is closer to the true value of 40! An overview of all output values can be found below, from these output values the new residual values `Res2` are constructed. Therefore we can conclude that the average residual has decreased! 

This process is then repeated until the model cannot improve anymore  


| **Salary**  | **age** | **degree** | **Residual** | **Output** | **Residual 2** |
| --------- | ---------- | --------- | --------- |  --------- | --------- |
|     40   | 20   | yes |  -9   | 47.2 | -7.2 |
|   46   |  28    | yes |  -3   | 47.2 | -7.2 |
|   50   |  31    | no |  1   | 50.95 | -0.95 |
|    54   |  34   | yes |  5   | 50.5 | 3.5 |
|   61   |  38    | no |  12   | 50.95 | 10.05 |

<br> 

An additional factor we need to take into account is gamma (γ), which is an arbitrary regularization parameter that controls the complexity of individual trees in the boosting process. It is a measure of how much an additional split will need to reduce loss in order to be added to the collection. 
A higher gamma value encourages simpler trees with fewer splits, which helps to prevent overfitting and lead to a more generalized model. A lower gamma, on the other hand, allows for more splits and potentially more complex trees which may lead to better fit on the training data but could increase the risk of overfitting.  

The tree is pruned when: 

<div style="text-align: center;">
{{<katex>}}
\text{gain-value - }\gamma < 0
{{</katex>}}
</div>
<br>

In this case the branch is removed. In other words the gain-value always needs to be higher than gamma in order for a split to be used. 

<br> 

## Using XGBoost in R

You can get started by using the [`xgboost` package in R](https://cran.r-project.org/web/packages/xgboost/xgboost.pdf). 

In what follows, we use [this dataset from Kaggle](https://www.kaggle.com/code/ekaterinadranitsyna/russian-housing-evaluation-model/input) for illustration. With this dataset we will try to predict the house price based on a number of features (e.g., number of bedrooms, stories, type of building etc).

In order to work with the algorithm you first need to install the `XGBoost` package, load the data and remove some columns via the following commands:

{{% codeblock %}}
```R
#install the xgboost package
install.packages("xgboost")

# Load necessary packages
library(xgboost)
library(tidyverse)
library(caret)

# Open the dataset 
data <- read.csv("../all_v2.csv", nrows = 2000)

#remove columns
data <- data %>%
  select(-date, -time, -geo_lat, -geo_lon, -region)
```
{{% /codeblock %}}

{{% warning %}}
Because XGBoost only works with numeric vectors, so you need to one-hot encode you data.
{{%/ warning %}}

Luckily, this is straightforward to do using the `dplyr` package. One-hot encoding is a process that transforms categorical data into a format that machine learning algorithms can understand. In the code below each category value in a column is transformed into its own column where the presence of the category is marked with a 1, and its absence is marked with a 0.

{{% codeblock %}}
```R

data <- data %>%
  mutate(building_type_1 = as.integer(building_type == 1),
        building_type_2 = as.integer(building_type == 2),
        building_type_3 = as.integer(building_type == 3),
        building_type_4 = as.integer(building_type == 4),
        building_type_5 = as.integer(building_type == 5)) %>%
  select(-building_type)
```
{{% /codeblock %}}

Now that we have one-hot encoded our data we separate the target variable `price` from the input variables. Subsequently, the data is divided into training and testing sets using an 80/20 split ratio. Finally the data is converted into separate data matrices in order for `XGBoost` to use it.

{{% codeblock %}}
```R
#separate target variable (price) from input variables
X <- data %>% select(-price)  
Y <- data$price 

set.seed(123) #set seed for reproducibility

train_indices <- sample(nrow(data), 0.8 * nrow(data)) #sample 80% for training set
X_train <- X[train_indices, ] #extract features for training set
y_train <- Y[train_indices] #extract target values for training set
X_test <- X[-train_indices, ] 
y_test <- Y[-train_indices]

#convert both sets to a DMatrix format, in order for xgboost to work with the data 
dtrain <- xgb.DMatrix(data = as.matrix(X_train), label = y_train)
dtest <- xgb.DMatrix(data = as.matrix(X_test), label = y_test)
```
{{% /codeblock %}}

### Hyperparameters

Now that we have created our training data and test data we need to set up our hyperparameters (think of these as our knobs and handles for the algorithm). These hyperparameters for the XGBoost algorithm consist out of:

  - `max_depth`: Specifies the maximum depth of a tree.
  - `min_child_weight`: Minimum sum of instance weight needed in a child to further split it.
  - `gamma`: Regularization term. Specifies a penalty for each split in a tree.
  - `eta`: Learning rate, which controls the weights on different features to make the boosting process more conservative (or aggressive).
  - `colsample_bytree`: Fraction of features sampled for building each tree (to introduce randomness and variety).
  - `objective`: Specifies the object (regression, or classification).
  - `eval_metric`: Specifies the model performance (e.g. RMSE).
  - `nrounds`: Number of boosting rounds or trees to be built.

Let us now (arbitrarily) set up some hyperparameters:  

{{% codeblock %}}

```R
params <- list(
  max_depth = 4,
  min_child_weight = 1, 
  gamma = 1,
  eta = 0.1, 
  colsample_bytree = 0.8, 
  objective = "reg:squarederror", # Set the objective for regression
  eval_metric = "rmse", # Use RMSE for evaluation
  nrounds = 100 
)

```
{{% /codeblock %}}

{{% tip %}}
You can also use objective = `binary:logistic` for classification.
{{% /tip %}}


With the parameters specified we can now pass them to the `xgb.train` command in order to train the model on the `dtrain` data!
{{% codeblock %}}

```R
xgb_model <- xgb.train(
  params = params,
  data = dtrain,
  nrounds = params$nrounds, 
  early_stopping_rounds = 20, # stop iteration when there test set does not improve for 20 rounds
  watchlist = list(train = dtrain, test = dtest),
  verbose = 1 # print progress to screen
)

```
{{% /codeblock %}}

Finally, let us make some predictions on the new dataset, view the original prices and the predicted prices next to each other, and assess the Mean Absolute Error (MAE).

{{% codeblock %}}

```R
predictions <- predict(xgb_model, newdata = dtest)

results <- data.frame(
  Actual_Price = y_test,
  Predicted_Price = predictions
)

mae <- mean(abs(predictions - y_test))
mea

```
{{% /codeblock %}}

As can be seen the predictions of the algorithm and the original prices still differ significantly (MEA is almost 2 million). A possible reason for this could be that the hyperparameters were arbitrarily chosen. In order to perform a more structured way to choose our hyperparameter we can perform a grid search. 

### Hyperparameter optimization using grid search 
Selecting the correct hyperparameters can prove to be quite a time-consuming task. Fortunately, there exists a technique in the form of grid search that can help alleviate this burden. 

Utilizing a grid search involves selecting a predefined range of potential hyperparameters for examination. The grid search systematically tests each specified combination, identifying the set of parameters that yield the highest level of optimization. Additionally, we make use of a 5-fold cross validation in order to robustly assess the models performance across different subsets of the data. 

{{% codeblock %}}
```R
ctrl <- trainControl(
  method = "cv", # Use cross-validation
  number = 5,  # 5-fold cross-validation
  verboseIter = TRUE, # print progress messages 
  allowParallel = TRUE # allow for prarallel processing (to speed up computations)
)
```
{{% /codeblock %}}

Following which we define the hyperparameter grid: 

{{% codeblock %}}

```R
hyper_grid <- expand.grid(
  nrounds = c(100, 200),
  max_depth = c(3, 6, 8),
  eta = c(0.001, 0.01, 0.1),
  gamma = c(0, 0.1, 0.5),
  colsample_bytree = c(0.6, 0.8, 1),
  min_child_weight = c(1, 3, 5),
  subsample = c(1)
)

```
{{% /codeblock %}}

In order to run the grid search we run the following code. With `X` and `Y` representing the input features and the target variable, respectively (this could take some time).
{{% codeblock %}}

```R
# Perform grid search
xgb_grid <- train(
  X, Y,
  method = "xgbTree", # specify that xgboost is used
  trControl = ctrl, # specify the 5-fold cross validation
  tuneGrid = hyper_grid, 
  metric = "RMSE" # specify the evaluation method
)

# Print the best model
print(xgb_grid$bestTune)
```
{{% /codeblock %}}

from which the output is: 

|            | **nrounds** | **max_depth** | **eta** | **gamma** | **colsample_bytree** | **min_child_weight** | **subsample** |
| --------- | ---------- | ------------- | ------- | --------- | -------------------- | ------------------- | -------------- |
| 445   | 100        | 8             | 0.1     | 0         | 1                    | 1                   | 1              |

indicating the following: 
- `nrounds ` = 100 means that the training process will involve 100 boosting rounds or iterations. 
- The value of 3 for `max_depth` means that the maximum depth of each decision tree in the ensemble was limited to 8 levels.
- `eta` = 0.1 indicates that a relatively small learning rate was applied making the model learn slow (but steady!)
- the value of 0 for `gamma` means that there was no minimum loss reduction required to make a further split. In other words, the algorithm could split nodes even if the split didn't lead to a reduction in the loss function. 
- `colsample_bytree` = 1 indicated that all available features are considered during training
- a new split in a tree node was required to have a minimum sum of instance weights of 1, as indicated by `min_child_weight`
- the full `subsample` was used

## Assessment

for the full set of combinations the command below can be used. This will also return all other indicators like RMSE, R2, MAE, R2SD, MAESD. 
{{% codeblock %}}

```R
print(xgb_grid$results)
```
{{% /codeblock %}}

Additionally, the importance of individual features can also be assessed. This can be done via the following code: 

{{% codeblock %}}

```R
importance_matrix <- xgb.importance(feature_names = colnames(X), model = xgb_model)
xgb.plot.importance(importance_matrix)

```
{{% /codeblock %}}

Which results in the following graph, from which it can be see that `kitchen_area` is the most important feature in predicting house prices. 

<p align = "center">
<img src = "../images/Features_XGB.png" width="500">
</p>


## Visualization

In order to visualize the created tree (using the hyperparameters found in the grid search), let's run the next line of code:
{{% codeblock %}}

```R
xgb.plot.multi.trees(feature_names = names(data), 
                     model = xgb_model) #the size of this plot can be adjusted by limiting the max_depth variable
```
{{% /codeblock %}}

<p align = "center">
<img src = "../images/tree_plot.png" width="400">
</p>

{{% summary %}}

- **Understanding XGBoost:**
  - Dive into the mechanics of XGBoost's functionality.
  - Explore its application within the `R`.

- **Setting Up and Tuning the Algorithm:**
  - Define and understand the hyperparameters.
  - Employ a systematic grid search for optimal model performance.

- **Interpreting and Visualizing Outcomes:**
  - Analyze model results for meaningful insights.
  - Visualize your algorithm's efficacy and decision process.

{{% /summary %}}