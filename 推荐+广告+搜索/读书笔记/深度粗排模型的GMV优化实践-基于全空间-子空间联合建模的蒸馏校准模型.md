---
alias:
发布时间: 2021-12-13
出品方: 阿里
价值: ⭐⭐⭐⭐
---

环节:: 粗排
关键词:: #Reco/双塔模型 #Reco/全链路优化 #Reco/多任务学习 #ML/蒸馏学习

---

# [深度粗排模型的GMV优化实践：基于全空间-子空间联合建模的蒸馏校准模型](https://mp.weixin.qq.com/s/_Ya9OM6NbMvKe8Lo_8QVmw)
#TODO


**阿里淘宝**：10亿（召回）--> 粗排（1万）--> 精排（10）

主要思路是：精排知识向粗排蒸馏。在粗排阶段，模型预估出ctr和cvr后，==如何组合ctr、cvr、price也是影响效果的关键因素==。

* 通常来说，会对ctr和cvr分别校准，再将ctr*cvr与price组合得到最终排序分数。
* 在这里能发现，更本质的需求是预估校准后的ctcvr。
* 我们基于蒸馏校准模型，引入了全空间建模的思路，并在现有的子空间建模的框架下进行了改进，以全空间-子空间联合建模的方式==直接预估出蒸馏校准后的ctcvr==。
* 在新模型中，如何平衡ctr任务与ctcvr任务是一个需要解决的问题，我们采用了冲突梯度投影算法，并结合应用场景，做了适配与改进。

## 基础模型：双塔

采用ctr和cvr任务分别用**独立数据源交替训练**的方式，也就是对ctr和cvr在**两个空间上分别建模**，

* 其中ctr的数据源是**曝光样本，点击是正样本，未点击的负样本有采样**；
* cvr的数据源是**点击样本，成交是正样本**

❓❓❓其实这里就是最大的问题，为什么要用pv->click和click->pay两个独立数据源？为什么不直接在pv->pay来训练cvr? 用两个独立数据源训练有什么优势吗？


考虑到粗排场景的高性能要求，采用了双塔内积。

* ==ctr和cvr任务各自有独立的user net==，user net由self attention和user+query attention构成的attention net来提取特征
* ==共享item net，==
* ==共享双塔底层的embedding层。==
* 在双塔内积之外，还有包含==user、query、display特征的bias net==来提高预估的准度。
* 由于item net与user、query无关，可以提前计算好，模型在线上提供服务时，只需要对一个query计算一次user net和bias net，通过向量内积和加法计算出排序分数

![image.png | 400](assets/image-20211213172716-fcvr3m9.png)


## 蒸馏校准模型

### 蒸馏

teacher模型：采用==已经训练好的精排多层模型（并非和student共同训练）==，指导粗排student模型的学习

![image.png](assets/image-20211213173619-6qs2svs.png)

* 第一项是粗排loss，
* 第二项是蒸馏loss，其中**rlogits来自精排（因为精排模型固定，因此是常量）**。需要注意的是，**rloigts是由校准后的精排打分**，反推回来的。
* T是温度超参。

精排的分数处于不同区间范围时，对应区间上数据的auc会有所不同，==特别在分数最低的一部分数据上auc最低，我们认为在这部分数据上精排的置信度比较低==，**这个置信度会反映到loss上进行权重调节（how??? T不是一个温度超参吗？难道还能随预估值变化而变化？）**。


### 校准

**假设ctr任务中负样本的采样率为r**，在训练过程中可以对正负样本设置不同的权重，来将预测概率的分布恢复到采样前。只需要注意到采样前后似然项不变，就能推导出正负样本权重设置为r:1时，预测概率可以被校准回真实分布。为了保持一致性，我们需要在训练过程中就进行校准，而不是事后校准。


## 全空间-子空间联合建模

蒸馏校准模型的训练方式是，对ctr和cvr任务分别用**pv->click**和**click->pay**两个**独立数据源**交替训练，也就是在**两个空间上分别建模**，其中click->pay是全空间pv->pay的子空间。对于一条pv样本来说，训练时模型没有同时对它打出ctr和cvr的分数，但这却是线上应用时的场景，也就是需要同时打出这两个分数再相乘。在这里，离线训练与在线预测存在**bias**


## 多任务平衡

用“冲突梯度平衡”动态设置训练各任务的权重。