## Feature Engineering Techniques

*2019-9-19*

*载于https://www.kaggle.com/c/ieee-fraud-detection/discussion/108575*

---

Engineering features is key to improving your LB score. Below are some ideas on how to engineer new features. Create a new feature and then evaluate it with a local validation scheme to see if it improves your model's CV (and thus LB). Keep beneficial features and discard the others.

If you create lots of new features at once, you can use forward feature selection, recursive feature elimination, LGBM importance, or permutation importance to determine which are useful.

### Train and Test

When performing Label Encoding below, you must encode train and test together as in

```python
df = pd.concat([train[col], test[col]], axis=0)
# Perform feature engineering here
train[col] = df[:len(train)]
test[col] = df[len(train):]
```

The other techniques you can choose to do together or separately. An example of separate is

```python
df = train
# Perform feature engineering here
df = test
# Perform feature engineering here
```

### NAN processing

if you give  `np.nan` to LGBM, then at each tree node split, it will split the non-NAN values and then send all the NANs to either the left child or right child depending on what's best. Therefore NANs get special treatment at every node and can become overfit. By simply converting all NAN to a negative number lower than all non-NAN values (such as -999),

```python
df[col].fillna(-999, inplace=True)
```

then LGBM will no longer overprocess NAN. Instead it will give it  the same attention as other numbers. Try both ways and see which gives the highest CV.

### Label Encode / Factorize / Memory reduction

Label encoding (factorizing) converts a (string, category, object) column to integers. Afterward you can cast it to int8, int16, or int32 depending on whether max is less than 128, less than 32768, or not. Factorizing reduces memory and turns NAN into a number (i.e. -1) which affects CV and LB as described above. Factorizing also gives you the choice to treat categorical variable as numeric described below.

```python
df[col],_ = df[col].factorize()

if df[col].max()<128:
    df[col] = df[col].astype('int8')
elif df[col].max()<32768:
    df[col] = df[col].astype('int16')
else:
    df[col].astype('int32')
```

Additionally for memory reduction, people use the popular `memory_reduce` function on the other columns. A simpler and safer approach is to convert all float64 to float32 and all int64 to int32 (It's best to avoid float16. You can use int8 and int16 if you like).

```python
for col in df.columns:
    if df[col].dtype=='float64':
        df[col] = df[col].astype('float32')
    if df[col].dtype=='int64':
        df[col] = df[col].astype('int32')
```

### Categorical Features

With categorical variables, you have the choice of telling LGBM that they are categorical  or you can tell LGBM to treat it as numerical (if you label encode it first). Either way, LGBM can extract the category classes. Try both ways and see which gives the highest CV. After label encoding do the following for category or leave it as int for numeric.

```python
df[col] = df[col].astype('category')
```

### Splitting

A single (string or numeric) column can be made into two columns by splitting. For example a string column `id_30` such as `Mac OS X 10_9_5` can be split into Operating System `Mac OS X` and Version `10_9_5`. Or for example number column `TransactionAmt`  "`1230.45`" can be split into Dollars "`1230`" and Cents "`45`". LGBM cannot see these pieces on its own, you need to split them.

### Combining / Transforming / Interaction

Two (string or numeric) columns can be combined into one column. For example `card1` and `card2` can become a new column with

```python
df['uid'] = df['card1'].astype(str)+'_'+df['card2'].astype(str)
```

This helps LGBM because by themselves `card1` and `card2` may not correlate with target and therefore LGBM won't split them at a tree node. But the interaction `uid = card1_card2` may correlate with target and now LGBM will split it. Numeric columns can combined with adding, subtracting, multiplying, etc. A numeric example is

```python
df['x1_x2'] = df['x1'] * df['x2']
```

### Frequency Encoding

Frequency encoding is a powerful technique that allows LGBM to see whether column values are rare or common. For example, if you want LGBM to "see" which credit cards are used infrequently, try

```python
temp = df['card1'].value_counts.to_dict()
df['card1_counts'] = df['card1'].map(temp)
```

### Aggregations / Group Statistics

Providing LGBM with group statistics allows LGBM to determine if a value is common or rare for a particular group. You calculate group statistics by providing pandas with 3 variables. You give it the group, variable of interest, and type of statistic. For example,

```python
temp = df.groupby('card1')['TransactionAmt'].agg(['mean']).rename({'mean':'TransactionAmt_card1_mean'}, axis=1)
df = pd.merge(df, temp, on='card1', how='left')
```

The feature here adds to each row what the average `TransactionAmt` is for that row's `card1` group. Therefore LGBM can now tell if a row has an abnormal `TransactionAmt` for their `card1` group.

### Normalize / Standardize

You can normalize columns against themselves. For example

```python
df[col] = (df[col] - df[col].mean()) / df[col].std()
```

Or you can normalize one column against another column. For example if you create a Group Statistic (described above) indicating the mean value for `D3` each week. Then you can remove time dependence by

```python
df['D3_remove_time'] = df['D3'] - df['D3_week_mean']
```

The new variable `D3_remove_time` no longer increases as we advance in time because we have normalized it against the affects of time.

### Outlier Removal / Relax / Smooth / PCA

Normally you want to remove anomalies from your data because them confuse your models. However is this competition, we want to find anomalies so use smoothing techniques carefully. The idea behind these methods is to determine and remove uncommon values. For example, by using frequency encoding of a variable, you can remove all values that appear less than 0.1% by replacing them with a new value like -9999 (note that you should use a different value than what you used for NAN).