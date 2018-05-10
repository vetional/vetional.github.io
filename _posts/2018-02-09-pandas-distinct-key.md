---
layout: post
title: "Pandas distinct"
comments: true
description: "Pandas distinct values by column"
keywords: "pandas distinct"
---


```python

import pandas as pd
df = pd.read_csv("all_words.txt", header=None, names=['key', 'val'])
df.describe()
```



```python
df = df.drop_duplicates(subset='key', keep="last")
df.describe()
```
