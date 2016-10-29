---
title: 离线AUC和线上CTR
date: 2016-10-29
tags: [AUC,离线评估,CTR,机器学习]
---

在做Rank系统时，我们常用ROC曲线的AUC来做离线评估。

AUC评估的是**序**,LOSS类指标（RMSE,log-loss等）评估的是**值**。

AUC更适合做Rank离线评估的原因有以下两点：
> 1. 将预测值都乘以二，线上排序不会改变。离线评估时，LOSS类指标会有变化，AUC不会变。
> 
> 2. 使用Pairwise训练产生的模型，预测时输出的是rank-score，并不是CTR的点回归，LOSS类指标一般都会比较差，不具参考意义。

但是AUC和线上CTR也不能完全对上，也存在很多坑。本文主要说说自己踩到的坑。。。

<!--more-->


# AUC的坑 #

### case 1 ###

| user  | rank  | item  | click | 基线打分 | 实验打分 |
| :---: | :---: | :---: | :---: | :---:   | :---:   |
|  u1   |  1    | i1    |  1    | 0.3     | 0.2     |
|  u1   |  2    | i2    |  0    | 0.2     | 0.1     |
|  u1   |  3    | i3    |  0    | 0.1     | 0.3     |

> 离线评估： 实验组AUC下降。

> 结论： 继续线下迭代。

> 等等。。。 u1真的最喜欢i1（基线），不喜欢i3(实验)? 也许是因为他没看i3呢？

> 离线评估数据中，是存在*position-bias*的。这里对position-bias稍微展开一下:

> > position-bias in show: 场景为用户在手机上浏览新闻，假设系统给用户加载了10条数据，手机一屏显示6条，如果用户没有进行翻页，那就只有top6的新闻真实曝光了，剩下的4条是伪曝光。对于伪曝光的数据，用户没点并不能说明他不感兴趣。*(用户真实的浏览行为更复杂，也许只看了前两条。需要做专门的click model，这个坑后续再填，这里只讲最简单的情况。)*

> > position-bias in click: 如果第一条和第三条内容差不多，同时都是用户感兴趣的新闻，一般用户都是从上往下浏览，那大多数情况就是用户点了第一条，没点第三条。同样的用户没点，不能说明他不感兴趣。

> 在实际项目中，在实验上线前，历史log中的Label（click）是按照基线组打分排序产生的，离线算AUC的话，是偏向基线的。
 
> 回到我们的例子中，假设用户u1对item的真实偏好程度为 *i3>i1>i2* 。

> 基线策略认为 *i1>i2>i3* ，所以给用户下发了 *i1,i2,i3*。 假设用户手机屏幕很小，只看见了i1和i2，他点击了i1。

> 但是，如果我们安装实验组打分给他下发 *i3，i1，i2* ,他就会看到i3和i1，有可能会产生两次点击。

> 这是我们站在上帝的角度来看问题，现实生活中，我们并不知道用户的真实偏好，只能从日志里去看，用户点击了i1没有点击i3，自然会觉得用户更喜欢i1，会产生一种基线就是最吼的错觉。

> 结论： **排序质量更优，AUC不一定上涨。**

> 解决方法： 同时使用Rank1-AUC(可能存在数据覆盖度不够的问题)。

### case 2 ###

| user  | rank  | item  | click | 基线打分 | 实验打分 |
| :---: | :---: | :---: | :---: | :---:   | :---:   |
|  u1   |  1    | i1    |  0    | 0.3     | 0.3     |
|  u1   |  2    | i2    |  1    | 0.2     | 0.2     |
|  u1   |  3    | i3    |  0    | 0.1     | 0.1     |
|  u2   |  1    | i1    |  0    | 0.3     | 0.03    |
|  u2   |  2    | i2    |  0    | 0.2     | 0.02    |
|  u2   |  3    | i3    |  0    | 0.1     | 0.01    |
> 优化点：实验组学出来u2基本不怎么点击，所以将他所有的打分都乘以0.1。
 
> 离线评估： 实验组AUC下降。
 
> 线上CTR： 线上排序没有变化，CTR肯定也基本持平。
 
> 原因分析： 单纯的环境特征(session, user, etc.)引入，能区分流量间的点击率高低，但是对单次请求没有区分性，无法提高线上排序质量。(这里只考虑最简单的情况，如果从去bias和特征组合的角度考虑，环境特征还是很重要的)
 
> 结论： **AUC上涨，线上CTR不一定有提升**。

> 解决方法： 同时使用Group-AUC (Group怎么划分是个问题)

### case 3 ###

| user  | rank  | item  | click | 基线打分 | 实验打分 |
| :---: | :---: | :---: | :---: | :---:   | :---:   |
|  u1   |  1    | i1    |  0    | 0.3     | 0.2     |
|  u1   |  2    | i2    |  1    | 0.2     | 0.3     |
|  u1   |  3    | i3    |  0    | 0.1     | 0.1     |
|  ---  |  ---  | i4    |  -    | 0.08    | 0.4     |
|  ---  |  ---  | i5    |  -    | 0.06    | 0.5     |
|  ---  |  ---  | i6    |  -    | 0.04    | 0.6     |

> 离线评估只能在历史展现过的数据上进行，对于未展现的数据不知道，存在survive bias。

> 在上升的例子中，我们先站在上帝的视角假设，用户u1对item的真实偏好程度为 *i2>i1>i3>i4>i5>i6*

> 历史数据中，只有i1,i2,i3被展示给用户了，我们拿过来做离线评估，会发现实验组的AUC有提升，很开心，终于有效果了，可以上线实验一把了。。。

> 但是上线后，发现效果反而下降了。原因是因为实验组把更差的i4,i5,i6展现给用户了 :(

> 结论： **离线评估集，不能代表全部数据**

> 解决方法： 统计上线后，实验组和基线组展现数据的diff，如果diff越大，说明survive bias越大。


# 写在最后 #

> 在排序系统里，AUC是一个比较置信的离线评估指标(我个人最喜欢)。但也不能迷信。。。

> ### idea很棒，离线AUC没有提升 ###

> 不要放弃。一般不会被允许进行线上实验的，但是不要轻易放弃。需要仔细分析，找出问题，不要被离线评估坑了(case1)。

> ### 离线AUC上涨，线上CTR没有提升 ###

> 0.review一下代码有没有bug

> 1.分析一下AUC提升的来源，看看是否对排序有帮助(case 2)

> 2.survive bias来背锅(case 3) 