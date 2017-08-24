---
layout: post
title: "Pandas apply functions on groups"
comments: true
description: "using custom aggregate fucntions with pandas apply"
keywords: "pandas apply aggregate"
---

### Groupby

```python

# Defining a small dataframe
raw_data = {'regiment': ['Nighthawks', 'Nighthawks', 'Nighthawks', 'Nighthawks', 'Dragoons', 'Dragoons', 'Dragoons', 'Dragoons', 'Scouts', 'Scouts', 'Scouts', 'Scouts'],
        'company': ['1st', '1st', '2nd', '2nd', '1st', '1st', '2nd', '2nd','1st', '1st', '2nd', '2nd'],
        'name': ['Miller', 'Jacobson', 'Ali', 'Miller', 'Cooze', 'Jacon', 'Ryaner', 'Sone', 'Sloan', 'Piger', 'Riani', 'Ali'],
        'preTestScore': [4, 24, 31, 2, 3, 4, 24, 31, 2, 3, 2, 3],
        'postTestScore': [25, 94, 57, 62, 70, 25, 94, 57, 62, 70, 62, 70]}
df = pd.DataFrame(raw_data, columns = ['regiment', 'company', 'name', 'preTestScore', 'postTestScore'])
df
```
Let's do a groupby on regiment and see how many people in each regiment have similar names.

```python
dd = defaultdict(int)
def fns(name):
    global dd
    print ( "\t" + name, dd)
    dd[name] += 1
    return dd

for name, reg in df.groupby('regiment'):
    print (name)
    reg['name'].apply(fns)
    print (dd)
    dd.clear()

```