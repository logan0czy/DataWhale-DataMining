# How to start a machine learning project from scratch?  
(mainly summarizing from [kaggle micro-course](https://www.kaggle.com/learn/overview))  
<span id='head'> </span>
**Contents**  
* Gather Data  
* [Prepare Data](#Prepare)  
    * EDA  
    * Feature Engineering
* [Select Model](#Select-Model)  
* Training  
* Evaluation  
* Tune Parameters  
* Get Prediction(Implementation)  

## Gather Data  
根据需要采集数据，过程中可能发生数据缺失或者受到污染的问题。  
<span id='Prepare'> </span>
## Prepare Data  
* [EDA](#EDA)  
* [Feature Engineering](#Feature-Engineering)  
    * [Missing Values](#Missing-Values)  
    * [Categorical Variables](#Categorical-Variables)  
    * [Numerical Variables](#Numerical-Variables)  
    * Feature Selection  
        * [Mutual Information](#Mutual-Information)  
        * [L1 Regression](#Lasso)
        * [Permutation](#Permutation)
        * [Partial Plots](#Partial-Plots)
        * [SHAP](#SHAP)
```python
建立data pipeline，避免发生data leakage的问题，同时避免了数据预处理的重复繁琐的操作在valid set/test set上遗漏相关步骤。
data -> model -> combine them! 其中pipeline允许嵌套使用

from sklearn.pipeline import Pipeline
my_pipe = Pipeline(steps=[
    ('name', process_fun),
    ('name', process_fun),
    ...
    ('name', model)
])

from sklearn.compose import ColumnTransformer
preprocessor = ColumnTransformer(transformers=[
        ('name', pipe/func, selected_cols),
        ('name', pipe/func, selected_cols)
])
```
<span id='EDA'> </span>
### [Exploration Data Analysis](#Prepare)  
拿到数据第一步需要对数据全貌，基本特性，各项特征的内容有一定的了解。  
1.描述  
```python
import pandas as pd

data = pd.read_csv(fpath, index_col= , parse_dates= )
# 基本信息
data.head()
data.dtypes() # string类型的数据也会被归入'object'类型中
data.describe() # 基础统计信息
# 缺失值
data.isnull().sum() / len(data)
```
2.选择部分直观上觉得有用的特征  
用少量特征和简单预测模型建立一个baseline  

3.baseline model上数据的进一步探索  
此部分用pandas库可以极大简化操作，提高数据分析效率，一些常用操作见[pandas常用操作整理](./Pandas_Operation.md)。此外，还有[常用可视化](./Visualization.md)  
<span id='Feature Engineering'> </span>
### [Feature Engineering](#Prepare)  
> 总体遵循--预处理操作将训练集和验证/测试集分离，避免data leakage问题。最好的方式make your data pipeline, 所有的预处理都放在pipeline内部进行  

<span id='Missing-Values'> </span>
**Missing Values**  
出现原因:  1.调查对象不愿提供  2.遗漏  3.数据逻辑上的先后关系，有的数据只能在特定情况下才有  
```python
# ---- drop ----
data.isnull.sum() / len(data)  # 缺失比例
data.dropna(axis= )
data.drop([])

# ---- imputation ----
from sklearn.impute import SimpleImputer
# SimpleImputer is based on mean value imputation
imputer = SimpleImputer()
data_impute = imputer.fit_transform(data)  # 'transform' for valid/test

# ---- imputation extension ----
# 数据项缺失本身可能就包含了一些信息，有时候是对预测有用的，所以在补全之外加上某特征是否有缺失的表示
data[col+'_bool'] = data[col].isnull()
```
<span id='Categorical-Variables'> </span>
**Categorical Variables**  
```python
# ---- label encoding ----
# 序列型类别ordinal variables，用有序数值打标签可以反应出它们的先后顺序，在诸如决策树模型中很有用
from sklearn.preprocessing import LabelEncoder

# ---- one-hot encoding ----
# 类别之间没有先后顺序，nominal variables
from sklearn.preprocessing import OneHotEncoder
encoder = OneHotEncoder(handle_unkown='ignore', sparse=False)
data_encode = encoder.fit_transform(data)  # 'transform' for valid/test

# ---- count encoding ----
# 用类别出现的次数作为编码，注意只能使用training data统计!!!
from category_encoders import CountEncoding

# ---- target encoding ----
# 各类别下target标签的均值来编码
from category_encoders import TargetEncoding

# ---- catboost encoding ----
#  同样面向target, 只不过是在当前数据之前的类别下target的均值
from category_encoders import CatBoostEncoding
```
<span id='Numerical-Variables'> </span>
**Numerical Variables**  
主要是数值分布不均的问题，常见长尾分布，用pandas绘制柱状图观察，数据取平方根（决策树有效）或者对数（使数值接近高斯分布，对深度学习模型有效）  

> 下列相关的操作都是基于在baseline model上的，比如Lasso regression, decision tree, random forests等，使用这些模型得到的基础训练结果可以为进一步的feature engineering提供信息，主要关注特征选择，找出重要特征以后还可以对这些特征进行进一步的特征组合  

[返回section目录](#Prepare)  
<span id='Mutual-Information'> </span>
**Multual Information**  
各特征与target的互信息  
```python
from sklearn.feature_selection import SelectKBest, f_classif
selector = SelectKBest(f_classif, K=10)
data_new = selector.fit_transform(data, target)  # 选取出互信息最大的K个特征

# 为了便于查看原始数据的筛选结果，需要对data_new逆变换
selected_data = pd.DataFrame(selector.inverse_transform(data_new), 
                                 index=data.index, 
                                 columns=data.columns)
```
<span id='Lasso'> </span>
**L1 Regression: Lasso**  
L1范数惩罚项进行的回归得到的结果为稀疏矩阵，主要用这个特性进行特征的筛选，惩罚系数越大，保留的特征越少，它与L2范数惩罚项不同(Ridge回归用于避免overfitting)  
最优的惩罚项系数还是要通过验证集来选择  
```python
# from sklearn.linear_model import Lasso
from sklearn.linear_model import LogisticRegression
from sklearn.feature_selection import SelectFromModel
train, valid, _ = train_test_split(data)
X, y = train[features], train[target]

# Set the regularization parameter C=1
logistic = LogisticRegression(C=1, penalty="l1", solver='liblinear', random_state=1).fit(X, y)
model = SelectFromModel(logistic, prefit=True)
X_new = model.transform(X)

# transform back
selected_features = pd.DataFrame(model.inverse_transform(X_new), 
                                 index=X.index,
                                 columns=X.columns)
```
<span id='Permutation'> </span>
**Permutation Importance**  
以验证数据集为参考，将训练好的模型应用在某一列特征随机乱序后的数据上，观察预测效果的改变
`fast to compute; widely used, easy to understand; consistent`  
```python
from sklearn.ensemble import RandomForestClassifier
**********
import eli5
**********
from eli5.sklearn import PermutationImportance
perm = PermutationImportance(model, random_state=1).fit(val_X, val_y)
eli5.show_weights(perm, feature_names = val_X.columns.tolist())
```
<span id='Partial-Plots'> </span>
**Partial Plots**  
how does one feature affect the prediction when other features remain unchanged?  
```python
from sklearn.ensemble import RandomForestClassifier
**********
# 1-D plot
from matplotlib import pyplot as plt
from pdpbox import pdp, get_dataset, info_plots
# Create the data that we will plot
pdp_goals = pdp.pdp_isolate(model=model, dataset=valid_X, model_features=feature_names, feature='col_name')
# plot it
pdp.pdp_plot(pdp_goals, 'col_name')
plt.show()

**********
# 2-D plot
features_to_plot = ['Goal Scored', 'Distance Covered (Kms)']
inter1  =  pdp.pdp_interact(model=model, dataset=valid_X, model_features=feature_names, features=features_to_plot)
pdp.pdp_interact_plot(pdp_interact_out=inter1, feature_names=features_to_plot, plot_type='contour')
plt.show()
```
<span id='SHAP'> </span>
**SHAP Values**  
相对于baseline value, raw data值对prediction正负贡献的量化  
`sum(SHAP values for all features) = pred_for_team - pred_for_baseline_values`
```python
from sklearn.ensemble import RandomForestClassifier
**********
row_to_show = 5
data_for_prediction = val_X.iloc[row_to_show]  # use 1 row of data here. Could use multiple rows if desired

import shap  # package used to calculate Shap values
# Create object that can calculate shap values
explainer = shap.TreeExplainer(model)
# Calculate Shap values
shap_values = explainer.shap_values(data_for_prediction)
# show contributions
shap.initjs()
shap.force_plot(explainer.expected_value[1], shap_values[1], data_for_prediction)  # [1]index for 'True' target,
                                                                                   # [0] for 'False'
# use Kernel SHAP to explain test set predictions
k_explainer = shap.KernelExplainer(my_model.predict_proba, train_X)
k_shap_values = k_explainer.shap_values(data_for_prediction)
shap.force_plot(k_explainer.expected_value[1], k_shap_values[1], data_for_prediction)
```
以上是用SHAP对单行数据进行分析得到的结果，SHAP还有一个有效的使用是不但能够知道数据集各特征对预测的影响程度，而且能够知道每种特征具体影响的分布(比如整体都不具有影响性，或者某些数据具有较强影响性，但是大部分影响比较中庸)--More details than permutation importance and partial dependence plots  
`summary plots`  
```python
# 以下程序在数据量不大时适用，另外程序运行对XGBoost有特定的优化
# Create object that can calculate shap values
explainer = shap.TreeExplainer(my_model)
# calculate shap values. This is what we will plot.
# Calculate shap_values for all of val_X rather than a single row, to have more data for plot.
shap_values = explainer.shap_values(val_X)
# Make plot. Index of [1] is explained in text below.
shap.summary_plot(shap_values[1], val_X)
```
`dependence contribution plots`  
和partial dependence plots类似，但是额外的提供某特征与另一特征之间的关联信息，如当前特征对预测的影响在对应值上是恒定的还是有分布，若有分布说明预测还受其它特征的共同作用。另外也可以看出当前特征一定趋势取值下对预测的影响。  
```python
# Create object that can calculate shap values
explainer = shap.TreeExplainer(my_model)
# calculate shap values. This is what we will plot.
shap_values = explainer.shap_values(X)
# make plot.
shap.dependence_plot('Ball Possession %', shap_values[1], X, interaction_index="Goal Scored")
```
[回到顶部](#head)  
**总体来说，特征工程一部分目的是提高模型的预测能力，另一部分也为模型结果提供了许多因素分析，reasonability**  

    Machine Learning Explainability
    - debugging
    - informing feature engineering
    - directing future data collection
    - informing human decision
    - build trust

<span id='Select-Model'> </span>
## [Select Model](#head)  
许多工作都应该从dirty work开始，scikit-learn提供了许多基本模型(Decision Tree, Random Forest, XGBoost, Lightgbm等)，先用这些库拟合得到baseline，再进行进一步的调优，模型调整。基本API是  
`define model and its parameters/random state -> fit training data -> predict on valid -> evaluate`  
研究不同模型的性能表现需要以验证集的指标为准，此外，为了避免单验证集上数据量限制原因使得其性能表现有有偶然性（即模型拟合了验证集上的特定特性），常用的是交叉验证方法，cross validation。  
```python
from sklearn.model_selection import train_test_split
from sklearn.model_selection import cross_val_score
scores = cross_val_score(my_pipeline/model, X, y, cv=5, scoring='metric_name')
scores.mean()
import sklearn.metrics
# compare based on metrics
```
额外提一下XGBoost(Extreme Gradient Boosting)，基于梯度下降从已有的模型参数得到更优参数生成新模型加入到model set，`ensemble`方式。  
重要参数：`n_estimators, early_stopping_rounds & eval_set, learning_rate, n_jobs`  
```python
from xgboost import XGBRegressor
my_model = XGBRegressor(n_estimators=1000, learning_rate=0.05, n_jobs=4)
my_model.fit(X_train, y_train, 
             early_stopping_rounds=5, 
             eval_set=[(X_valid, y_valid)], 
             verbose=False)
```
