# 实例分析：基于MapReduce的Logistics回归


## 准备知识

* Logistics回归
* R和Python
* Hadoop Streaming






## 建立Logit回归模型

将购买（Purchase）作为本文的因变量，将购买CH”定义为0，“购买MM”定义为1，选取CH售价（PriceCH），MM售价（PriceMM），CH折扣（DiscCH），MM折扣（DiscMM），CH特殊指标（SpecialCH），MM特殊指标（SpecialMM），CH忠诚度（LoyalCH）几个指标作为自变量建立Logit回归模型。MapReduce中首先将数据打乱，随后分成不同的3块，以下分别给出3块回归的结果和最后Reduce后的结果，并和利用全部数据整体进行对比。


### Mapper部分

```
#! usr/bin/env python
import sys
import pandas as pd
from statsmodels.api import *
import numpy as np
Purchase = []
WeekofPurchase = []
StoreID = []
PriceCH = []
PriceMM = []
DiscCH = []
DiscMM = []
SpecialCH = []
SpecialMM = []
LoyalCH = []
Store7 = []
PctDiscMM =[]
PctDiscMM = []
Store = []

#打开文件
f = sys.stdin
#f=open('orangejuice.csv','r')
#读取数据
for line in f.readlines():
    vrb = line.split(',')
    Purchase.append(vrb[1])
    WeekofPurchase.append(float(vrb[2]))
    StoreID.append(float(vrb[3]))
    PriceCH.append(float(vrb[4]))
    PriceMM.append(float(vrb[5]))
    DiscCH.append(float(vrb[6]))
    DiscMM.append(float(vrb[7]))
    SpecialCH.append(float(vrb[8]))
    SpecialMM.append(float(vrb[9]))
    LoyalCH.append(float(vrb[10]))
    Store7.append(vrb[14])
    PctDiscMM.append(float(vrb[15]))
    PctDiscMM.append(float(vrb[16]))
    Store.append(float(vrb[18]))

f.close

#因变量选择PUrchase中的MM，如果是MM则取1否则为0
MM = pd.get_dummies(Purchase)#将分类型变量转化为0，1
MM = MM[MM.columns[1]]#选择MM为真的一列

#选取自变量
xdata = pd.DataFrame([PriceCH,PriceMM,DiscCH,DiscMM,SpecialCH,SpecialMM,LoyalCH])
xdata=xdata.T                      
xdata.columns=['PriceCH','PriceMM','DiscCH','DiscMM','SpecialCH','SpecialMM','LoyalCH']
xdata['intercept']=1.0

#打乱顺序
shuffle = np.random.permutation(1069)
MM = MM[shuffle]
xdata = xdata.ix[shuffle]

l = len(MM)
n=3#划分不同的几块，这里令n=3
step = l/n

#对前n-1块进行计算
for i in range(n-1):
    #print ("利用第{}行到{}行的数据进行建模。".format(step*i,step*(i+1)))
    logitmodel = Logit(MM[step*i:step*(i+1)],xdata[step*i:step*(i+1)])
    res = logitmodel.fit(disp=0)
    #print res.summary()
    for j in range(len(res.params)):
        print res.params [j]
    print ','

#对最后一块进行计算
#print ("利用第{}行到{}行的数据进行建模。".format(step*(n-1),l))
logitmodel = Logit(MM[step*(n-1):l],xdata[step*(n-1):l])
res = logitmodel.fit(disp=0)
#print res.summary()
for j in range(len(res.params)):
    print res.params [j]
```

### Reducer部分

```
import sys
import pandas as pd
#读取数据
f = sys.stdin
fields = f.read()
#整理数据
fields = fields.split(',')#每一块数据的结果是按“，”分割的，重新用“，”划分成列表
data = [field.split('\n') for field in fields]#每一块中自变量的系数
for i in range(len(data)):
    while '' in data[i]:#去除列表中的空格
        data[i].remove('')
data = pd.DataFrame(data)#重新整理成数据框
data.columns=['PriceCH','PriceMM','DiscCH','DiscMM','SpecialCH','SpecialMM','Lo\
yalCH','Intercept']
data = data.astype(float)#将数据框中的元素类型转化为float
print data
print data.mean()
```

### 实践结论
观察Logit全模型的估计结果，可以发现CH标价（PriceCH）、MM标价（PriceMM）、MM折扣（DiscMM）、CH品牌忠诚度（LoyalCH）、MM折扣比例（PctDiscMM）对Purchase有显著的影响。一定程度上对Purchase有解释作用，而其他变量对因变量解释性并不显著。
（感谢中央财经大学朱述政提供素材和案例。）