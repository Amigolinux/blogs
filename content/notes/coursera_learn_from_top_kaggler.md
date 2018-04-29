Title: How to Win a Data Science Competition: Learn from Top Kagglers | Coursera
Date: 2018-04-22 22:36
Modified: 2018-04-22 22:36
Slug: coursera-learn-from-top_kaggler
Authors: Joey Huang
Summary: How to Win a Data Science Competition: Learn from Top Kagglers 笔者
Status: draft

## 机器学习算法总结

### 机器学习算法的分类

* Linear: scikit-learn(@); vowpal wabbit
  Split space into 2 subspaces
* Tree-based: scikit-learn; XGBoost(@); LightGBM(@)
  Split space into boxes
* kNN: scikit-learn
  heavy rely on how to measure points "closeness"
* Neural Networks: Pytorch(@); TensorFlow; mxnet; Lasagne
  Produce smooth non-linear decision boundary

VOWPAL WABBIT: 处理超大型数据的 Linear 算法库
Pytorch: 更符合 Python/Unix 的设计哲学；更简单易懂；适合学术型研究。
TensorFlow: 完整的工具链，适合产品经应用；但构造模型不直观，对程序员来说难以理解。

参考链接：https://www.zhihu.com/question/65578911/answer/249995561

经验总结：
一般情况下，GBDT (Gradient Boost Decision Tree like XGBoost, LightGBM) 和 Neural Networks 是更强大的算法。但有时，可以考虑 Linear 和 kNN，它们在有些场景下可以实现更快的训练和更高的准确率。

### 不同的算法特点

不同算法适用范围不同，特别是每种算法的决策边界形状不同导致了不同的算法对不同的数据分布形状有不同的准确性和通用性。需要熟悉每种算法的决策边界的特征，然后根据数据的分布情况，选择合适的算法模型。

参阅链接：http://scikit-learn.org/stable/auto_examples/classification/plot_classifier_comparison.html

## 特征预处理

特征有不同的数据类型，典型地有数值型（numeric feature），类别（categories feature），文本（text feature），图像（image feature）等等。不同的特征有不同的预处理方法。不同的特征的处理方法还和使用的模型相关。

比如 Tatinac dateset 里，pclass 是一个 categories feature，分别用 1，2，3 表示不同等级的座位。如果我们使用 linear model 来拟合模型，座位级别和幸存之间的关系可能不是线性的，此时模型的拟合效果就不好。如下图所示：

![pclass](https://raw.githubusercontent.com/kamidox/blogs/master/images/kaggler_titanic_pclass.png)

此时，就需要对特征进行预处理，比如使用 One Hot Encoder 方法，把特征转换为对线性模型友好的新特征，如下图所示：

![one hot encoder](https://raw.githubusercontent.com/kamidox/blogs/master/images/kaggler_tatinic_pclass_ohe.png)

scikit-learn 里的 `sklearn.preprocessing.OneHotEncoder` 是处理 One Hot Encoder 的常用方法。然而，随机森林等 tree based 模型则不需要做这种预处理，它可以很好地对 categories feature 进行处理。这是因为 tree based 模型是把空间分成一个个盒子，而 linear based 模型是把空间分成两个子空间。

TODO: 针对 Titanic dataset 验证处理前后模型准确性的差异。

总结：

* 特征预处理常常是必要的
* 通过特征分析生成新特征是一个强大的工具
* 特征预处理和特征生成的策略和方法，通常和选择的模型相关

### 数值特征预处理

* tree-based 模型的数值特征预处理
* non-tree-based 模型的数值特征预处理
* 特征生成

#### 特征预处理

non-tree-based 模型往往对数值特征的数量级别比较敏感。比如 kNN 分类时，计算预测点到周围的点的距离，如果某个特征的数值范围很大（100 - 1000），则计算出来的距离就很大；而另外一个数值的距离单位较小（1 - 10），则计算出来的距离小得多，此时就会待预测点预测为距离较短的点。Linear model 也有类似的特性，特征数值的大小，会决定分隔平面的斜率。如下图所示：

![feature scale](https://raw.githubusercontent.com/kamidox/blogs/master/images/kaggler_numeric_feature_scale.png)

解决此类问题的关键，是要把数值型特征进行相应的缩放，使其在同一个数量级。常用的算法有：
* `sklearn.preprocessing.MinMaxScaler`：把特征转换为 [0, 1] 之间
* `sklearn.preprocessing.StandardScaler`：把特征转换为中间点为 0，方差为 1 的，满足正态分布的数值

此外，针对 linear model，还需要特别注意异常值（outliers）的处理。因为这些异常值往往会对 linear model 有较大的影响，从而导致模型准确性受损，如下图：

![outliers](https://raw.githubusercontent.com/kamidox/blogs/master/images/kaggler_outliers.png)

处理异常值的一个方法是，直接去掉这个异常值，比如我们取特征的值分布的 99% 作为有效值，去掉两端的异常值。另外一个方法是使用 rank 的方法来缩减异常值和正常值的距离。常用的是 Pandas 里的 `DataFrame.rank` 或 `Series.rank`，它们按照数值的大小排序，然后取序号来代替真实值。

```python
In [1]: import pandas as pd

In [2]: s = pd.Series([-100, 0, 1e5])

In [3]: s
Out[3]:
0      -100.0
1         0.0
2    100000.0
dtype: float64

In [5]: s.rank()
Out[5]:
0    1.0
1    2.0
2    3.0
dtype: float64

In [6]: s2 = pd.Series([10, 100, 1, 30, 1e8])

In [7]: s2
Out[7]:
0           10.0
1          100.0
2            1.0
3           30.0
4    100000000.0
dtype: float64

In [8]: s2.rank()
Out[8]:
0    2.0
1    4.0
2    1.0
3    3.0
4    5.0
dtype: float64
```

如果我们没有时间去手动处理异常值，使用 rank 的方法也是一个不错的选择，linear model, kNN, neural networks 都会从这种处理方法中受益。

还有一个处理方式是使用 `np.log(1 + x)` 或使用 `np.sqrt(x + 2/3)` 来缩放数据。这种转换实际上是把异常值往中间靠拢，neural networks 算法可能会从这种转换中显著地提高性能和准确性。

最后一个常用的预处理方法是，把不同预处理方法处理后的值串联起来作为新特征，或用不同的模型分别训练不同预处理方法处理的数据。这实际上类似于 ensemble 的思想。Linear models, kNN, nueral networks 等 non-tree-based 模型可以通过这种方法显著地提高性能。

TODO: 构造一个实例来验证上述说法。

#### 特征生成

特征生成是指在分析数据特征及训练目标后，根据业务特点创造出来的新特征，它会提高模型训练的准确性和有效性。特征生成往往需要对业务背景要有足够深刻的理解，特征生成的能力往往可能区分出一般的机器学习工程师和好的机器学习工程师。

比如，房子有面积特征和价格特征，我们可以生成一个新的特征叫单位面积的价格。再如，针对价格类的特征，如 2.49, 3.99 这种，可以把小数点后面提取出来作为一个新的特征，这个特征实际上是建立在消费者对价格的后缀的敏感度角度建立的。如 3.99 用户可能感觉比较便宜，还不到 4 ，这是利用认识错觉而建立起来的特征。

再如，为了区分网络蜘蛛和人，我们可以快速闪过一个文本，网络蜘蛛和人对这个文本会有不同的表现，网络蜘蛛会获取这个文本信息，而人是不可能在瞬间读取这个文本内容的。

#### 总结

* 针对不同的模型，数值特征的预处理方法不同
    * tree-based 模型不依赖于特征数值缩放
    * non-tree-based 模型严重依赖特征数值缩放
* 常用的数值缩放方法
    * `MinMaxScaler`
    * `StandardScaler`
    * `rank`
    * `np.log(1 + x)` 和 `np.sqrt(1 + x)`
* 特征生成
    * 先验知识，业务背景知识
    * 数据分析 EDA

### Categorical and ordinal feature 的预处理

* Categorical 特征和 ordinal 特征针对不同模型的处理方法
* Categorical 特征和 ordinal 特征有哪些不同
* 怎么样生成新特征

#### 不同特征的特点

* ordinal feature 与 categorical feature 的差异
  ordinal feature = order categorical feature
* ordinal feature 与 numerical feature 的差异
  在 Titanic dataset 里，pclass 有 1, 2, 3 三个级别，这个是 ordinal feature 还是 numerical feature 呢？答案是 ordinal feature。因为 numerical feature 的一个显著特点是数值等差，即 pclass = 1 和 pclass = 2 之间的“距离”与 pclass = 2 和 pclass = 3 之间的“距离”相同。但在我们的案例里，显然没有这个特点。所以，这是一个 ordinal feature。另外一些 ordinal feature 的例子是，驾照类型 A, B, C。再如教育程度，幼儿园，小学，中学，本科，硕士，博士等。

#### 预处理方法

** LabelEncoder**

把 categorical feature 映射成数字。这种方法最简单，但只适用于 tree-based model 。Non-tree-based model ，包括 linear mode, kNN, neural networks 不适用这种处理方法，因为模型无法把它当成一个数值来进行计算。想像一下 kNN 是根据距离来分类的，而 LabelEncoder 处理后的数值不能反应真实的“距离”。如下图所示，左图，linear model 无法很好地利用 LabelEncoder 处理后的 categorical feature，而 tree-based model 可以很好地利用处理后的数据。

![LabelEncoder](https://raw.githubusercontent.com/kamidox/blogs/master/images/kaggler_categorical_feature.png)

**概率分布处理**

除了 `LabelEncoder` 外，`Pandas.factorize` 方法也可以实现按照出现顺序进行数值编号。另外，还有一种方法是根据出现的概率，把 categorical feature 转换为数值，如 A 出现的概率为 50%，B 出现的概率为 30%，C 出现的概率为 20%，则使用 0.5, 0.3, 0.2 分别代表 A, B, C。 这种方法可以保留一些类别分布信息，对 tree-based model 和 linear model 都有帮助。Linear model 可以从概率和目标值的相关性里找出规则；tree-based model 可以从数值和目标的相关性，获得树的较少分裂次数。但是要注意，如果类别概率差不多，会造成模型无法区分出不同的概率，从而导致模型无法从这个新特征里获得有帮助的信息。此时可以使用 `rank` 方法来处理。

**one-hot encoding**

One-hot encoding 方法是给每个类别的值创建一个新特征，等于这个类别值的样本，此特征为 1 ，其他为 0。这种方法对 non-tree-based model 非常有用。需要注意，这种方法处理后的特征其实已经经过缩放了，最大值是 1 ，最小值是 0。

如果 dataset 里有少量的 numerical feature，但有大量的 one-hot encoding 处理后的特征，此时 tree-based 模型可能会得到较差的性能。在这种情况下，其一，tree-based 模型无法有效地利用 numerical feature；其二，tree-based 模型会因此导致分裂过多，训练速度变慢，但性能没有提升。

另外一个需要注意的事项，如果 categorical feature 的类别非常多，会导致创建很多新特征，样本在这些新特征的的值大部分为 0，这个会浪费存储空间，同时减慢模型训练速度。一个好的方法是使用 Sparse Matrix 来存储数据，如 `scipy.sparse.csr_matrix`。

**categorical feature interaction**

如果多个 categorical feature 的联合体对目标有影响，那么我们可以直接把这些 categorical feature 连接起来，组成一个新的 categorical feature，然后对这个新的 categorical feature 做 one-hot encoding 操作，这样就会生成很多新的特征。Linear model 会拟合出不同的参数，来适配这个新的组合特征，从而达到较好的模型准确率。如下图，把 pclass 和 sex 两个特征合并，生成一个新的 pclass_sex 的特征，然后再针对 pclass_sex 的特征执行 one-hot encoding 操作。

![categorical feature interaction](https://raw.githubusercontent.com/kamidox/blogs/master/images/kaggle_categorical_interaction.png)

需要注意，需要从业务角度判断，只把相关的 categorical feature 合并起来处理，而不是所有的 categorical feature 合并都会产生好的效果。

#### 总结

* Ordinal feature 是一种按照某种关系排序后的 categorical feature
* Label encoding 可以把 categorical feature 映射到数值
* Frequency encoding 可以把 categorical feature 映射到某出现的概率
* Label 和 frequency encodings 对 tree-based models 比较有效，但对 non-tree-based models 效果不好
* Non-tree-based models 常常使用 One-hot encoding 的方法来进行预处理
* Interfactions of categorical features 常常对 linear model 准确性有帮助

### 时间及地理坐标特征的预处理

#### 时间特征的处理

**周期性**

比如，我们可以根据样本的时间，添加小时，分钟，秒等新特征。或者添加星期，月，日等特征。这些特征会帮助模型抓住数据里的一些周期性特征。比如，模型会发现每周五晚上几点到几点会出现服务器访问高峰期这种模式。周期性划分时，不一定按照自然单位进行划分，还可以根据业务背景进行特定地划分，比如某种药是 3 天一个疗程，我们可以以 3 天为单位进行划分，这样可以根据时间特征，创建一个新的特征叫疗程数，来帮助模型抓住疗程数和疗效之间的关联关系。

**事件锚点**

另外一个处理方式是，可以把时间参数处理成和某个事件锚点的关联关系。如离周末的天数，或离十一长假的天数等等。下面是一个典型的使用 time since 方法创建的新特征，其中 date 是原始的日期特征，sales 是目标值，中间的都是创建出来的新特征。

![time since](https://raw.githubusercontent.com/kamidox/blogs/master/images/kaggler_time_since.png)

**事件间距**

还有一个处理方式是，可以根据不同的事件的发生时间，计算它们之间的时间间距。这个可以帮助模型发现事件发生间隔和目标值的关系。如电商交易里，可以计算出一年中用户的平均交易时间间隔，一个月的平均交易时间间隔。此外，还可以计算交易时间和打投诉电话的时间间隔，一般情况下如果这个间隔很短，说明用户在交易过程中遇到问题，会影响这个用户的下次交易行为，从而影响用户的交易总量。

**进一步预处理**

从时间特征里创建出的新特征，有些是 numerical feature，有些是 categorical feature，这些新特征需要用上述介绍的相应方法，结合我们使用的模型，进行二次预处理，以便这些新创建的特征能对我们的模型有帮助。

#### 地理特征处理

地理特征处理需要结合地图数据进行处理。比如需要预测房子的价格，除了房子本身的特征（面积，朝向，楼层，结构）外，房子所处的地理位置也很重要。怎么样利用位置信息呢？如果我们有额外的地理信息数据，可以计算房子离最近的医院，学校，公园的距离。如果没有额外的数据，可以从训练样本样本挖掘出新的特征，比如，可以把训练样本的房子分成一些相同的区域，然后然后找出最贵的房子，并计算这个房子到最贵的房子的距离。再如，可以找出一些特征的房子，比如年限最老的房子的区域，然后计算样本到这个区域中心点的距离。

另外一个技巧是，当一条街道把两个区域的房子价格区分开时，我们可以试着增加一个特征，这个特征是这个街道做适当的旋转，使得基于 tree-based 的模型可以在一次分裂即可区分出这两种价格，从而大大提高模型的训练速度和精度。如下图所示：

![coordinate](https://raw.githubusercontent.com/kamidox/blogs/master/images/kaggler_coordinate.png)

#### 总结

* 时间特征
    * 周期性
    * 事件锚点
    * 事件间距
* 地理位置
    * 距离最近的标志性地点，如公园，医院，学校等
    * 计算训练样本到局域性中心点的距离
    * 对训练样本周边进行聚合统计，如公园数量，医院数量，学校数量等等

### 缺失值的处理

缺失值包括 NaN, 空字符串，奇异点。有时缺失的值也会包含有用的信息，比如为什么会有缺失值？缺失代表什么意思？有时我们可以从这些含义的背后，创建出新的特征来。

#### 缺失值的识别

有时缺失的值并不是 NaN，可能是一些不容易发现的隐藏的值，如 -1。怎么样发现 -1 是缺失值呢？可以通过 EDA 来看数据的分布，从而发现缺失的值。如下图：

![hidden missing values](https://raw.githubusercontent.com/kamidox/blogs/master/images/kaggler_missing_value.png)

再如，有些异常值也可以当成缺失值来处理，如年份在 2050 年上映的电影，或者重量为 100 吨的桃子等。有时，某些 categorical feature 的类别只在测试数据里出现，但没在训练数据里出现。此时模型就会把这些只在测试数据里出现的值作为缺失值处理，进而影响模型的性能。一个处理方法是，使用类别数据出现的频率来进行编码，这样那些只在测试样本里出现的数据也会被正确地编码并处理。

#### 处理缺失值

有三种常用的方法来处理缺失值：

* 替换成常数，如 999， -1，0 等：这种方法对 non-tree-based 模型不友好，可能导致模型准确性下降
* 替换成平均值，中位数值：对 linear model 比较友好
* 重新构建出缺失值

缺失值的处理方法和问题背景密切相关。比如，我们统计了一年的温度，但是中间有段时间值是缺失的，如果我们把缺失的值替换成平均值，或替换成 0，可能都不是好的方法。比如我们创建了一个新的特征，这个特征是前后两天的温度差，这样的缺失值处理会误导我们的模型。

![wrong missing values handler](https://raw.githubusercontent.com/kamidox/blogs/master/images/kaggler_mean_temp.png)

#### 总结

* 如何填充缺失值，需要根据具体的问题具体分析
* 常用的缺失值处理，是替换成常数，平均值，中位数
* 数据提供者可能已经把缺失值填充成某个特殊的值，如 -999，此时可以通过画出数据的柱状图来检查
* 添加一个新特征叫 isnull，来表示这一样本是否包含缺失值，往往对模型有帮助
* 通常情况下，避免在生成新特征前去填充缺失值。因为填充的缺失值可能会影响生成的新特征的准确性。
* XGBoost 可以自己处理 NaN 值

### 文本特征处理

文本处理的原则是把文本转换为向量，有两种转换方式，一种是 Bag of words，另外一种是 word2vec。

![text to vector](https://raw.githubusercontent.com/kamidox/blogs/master/images/kaggler_text_2_vec.png)

#### Bag of words

Bag of words 的原理就是把每个词语作为一个特征，然后统计这个词在一篇文章里的出现次数。

![CountVectorizer](https://raw.githubusercontent.com/kamidox/blogs/master/images/kaggler_counter_vec.png)

scikit-learn 里的 `CountVectorizer` 可以很好地实现这种转换。

我们前面讨论过，Linear model, kNN, neural networs 等算法对数值的大小比较敏感，需要做等量缩放。Bag of words 里另外一个能实现同等数量级缩放的方法是 TFIDF 方法，它的全称是 Term Frequency - Inverse Document Frequency 。TF 指的是一个词语在一篇文章中的出现次数，IDF 是指这个词语在整个数据集里的出现的频率的 log 值，如果一个词语在所有的文章里都出现，则它的概率是 1，其 log 值则为 0，即它的权重为 0。这个也容易理解，一个单词在每篇文章中都出现，它不带来任何的有效信息。scikit-learn 里的 `TfidfVectorizer` 可以完成这样的任务。

另外一个常用的算法是 N-grams，它不单单统计一个单词，还处理这个单词的上下文信息。当只统计一个单词时，称为 unigrams，统计两个单词时，称为 bigrams，统计三个单词时称为 trigrams。一般情况下，不会超过三阶，因为统计的单词越多，特征数量会成指数增长。

![N-grams](https://raw.githubusercontent.com/kamidox/blogs/master/images/kaggler_ngrams.png)

有时，可以使用字符来代替单词来应用 N-grams 模型，这样做的好处是，可以让模型发现那些我们不认识，但广泛使用的缩写词。可以通过指定 `analyzer='char'` 来启用以字符为单位的分析，可以通过 `ngram_range` 参数来指定 N-grams 阶数。

在进行 Bag of words 处理前，还需要对文本进行预处理，典型地预处理步骤包括：

* 全部转换为小写
* 词形还原（lemmatization），如把 was -> is，having -> have 等
* 词干提取（stemming），去掉词尾，提取出词根，如 democracy, democratic, democratization -> democr，
* 去掉停止词，如把常用的 in, of, at, is 等词语去掉

NLTK 是进行英文文本预处理的理想工具。

#### Word2vec

Bag of words 的方法把文本置换为一个很大的稀疏向量。而 word2vec 使用的是另外一种方法，它把文本转换为一个通常只有几百维的向量。其更大的特点是，word2vec 能精确地表达词语之间的关系，即意思相近的词，其向量也相近。如下图：

![word2vec](https://raw.githubusercontent.com/kamidox/blogs/master/images/kaggler_word2vec.png)

使用 word2vec 处理的单词向量，存在如下关系：king + women - man = queen。

word2vec 算法的其他典型实现包括 Glove, FastText 等。训练 word2vec 不需要我们待处理的文本，而是用一个大而全的语料库，而且需要很长的时间。网络上有一些根据通用语料库训练好的模型。比如 Glove 预训练向量可以从 https://nlp.stanford.edu/projects/glove/ 下载。FastText 预训练向量可以从 https://github.com/facebookresearch/fastText/blob/master/pretrained-vectors.md 下载，它甚至还提供了多种语言的预训练向量，包括中文。

#### 总结

* 预处理（写换为小写，词形还原，词干提取，停止词）
* 使用 N-grams 可以处理文本的上下文信息
* 后处理，使用 TFIDF
* Bag of words 和 word2vec 往往会有不同的结果，可以使用 ensemble 方法结合起来使用

### 图像特征的处理

卷积神经网络是处理图像特征的最有利武器。关于卷积运算，可以搜索“如何通俗易懂地解释卷积”。知乎上有一个非常直观的解释 https://www.zhihu.com/question/22298352 。

使用卷积神经网络处理图像时，一个方法是使用针对问题领域预训练的神经网络模型，如针对病理图像分类的问题，可以使用 VGG, ResNet 等。这种方法往往对数据量比较小的问题，比自己从头训练神经网络模型效果更好。

TODO: 关于神经网络，需要学习完 PyTorch 后再回头重看这个视频。

训练一个神经网络需要大量的数据，数据量越大，模型准确度越好。有时，我们的数据量不够，图片太少，此时可以通过对图像进行旋转，旋转后的图像作为新的训练样本，这样就可以增加训练样本的数量，从而提高模型性能。比如针对房屋朝向的预测，有四种类别，分别是坐北朝南，坐南朝北，平面房顶，其他。我们可以通过旋转图片，从而得到更多的训练样本，可以旋转 180 度，得到一组新数据，旋转 90 度得到另外一组新数据。这样我们的训练样本就增加了 4 倍。

![increase dataset](https://raw.githubusercontent.com/kamidox/blogs/master/images/kaggler_inc_data.png)

## EDA

EDA 的全称是 Exploratory data analysis，即数据探索和分析。这个是每个机器学习任务的基础，我们需要熟悉数据，熟悉问题的背景。为后续建模打下基础。

### 对数据建立初步印象

* 获取领域信息，即问题的背景
* 对数据获取直觉信息
* 理解数据是怎么产生的

比如，发现人类的年龄是 360 ，这肯定是个错误的数据。后续对数据进行预处理时需要解决。再如，针对广告的数据，点击数一定小于等于展示数的，没有展示哪来的点击。有了这样的领域信息，我们也可以判断是否有错误的数据。因为错误的数据会误导我们的模型。针对这种情况，我们可以创建一个新的特征，称为 "is_correct"，我们的模型可以会从这个特征里学到东西，从而提高准确性。

此外，理解数据是怎么产生的，对构建 validation 模型有重要的帮助。比如，训练数据集和测试数据集不是用相同的算法产生的，此时我们就无法从训练数据集里取出一部分来作为模型验证的数据。

TODO: 随书的 w2_002_Building intuition about the data_EDA_video2.ipynb 是个很有趣的 EDA 示例。值得一看。

### 匿名数据分析

有时候，组织者为了隐藏一些敏感信息，会把数据的字段改为匿名字段，如 x1, x2 等。针对这种类型的数据，我们需要尽量去搞清楚这些数据的含义是什么，最少也需要搞清楚这些数据的类型，如 numeric feature, categorical feature 等，这样我们才能做合适的预处理。

w2_003_Exploring anonymized data.ipynb 这个 EDA 里从匿名数据里发现一个特征是出生年份。这个特征被 `StandardScaler` 缩放和平移了。通过一步步发现缩放的系数以及平移的系数，最终发现这是一个年龄。除此之外，这个 EDA 另外一个值得学习的是，使用 `RandomForestClassifier` 构建了一个基准模型，然后训练数据后，通过模型的 `feature_importances_` 去发现这些特征的重要程度。这个方法也有助于我们去分析最重要的特征，而舍弃那些不重要的特征。

匿名数据中，有用的函数：

* df.dtypes: Pandas 猜测的数据类型
* df.info(): 数据的概要信息
* df.value_counts(): 数据的分布情况
* df.isnull(): 数据是否包含空值

### 数据可视化

* 逐个分析特征
    * 柱状图（Histograms）查看数据的分布
    * 走势图（Plots）查看数据的规律
    * 统计图（Statistics）查看数据的统计信息
* 探索特征关联关系
    * 点状图（Scatter plot）
    * 相关性（Scatter plot）

#### 单特征分析

EDA 是一门艺术，可视化工具是我们的工具。有时候，一些图形会造成误解，比如下面的柱状图：

![histoograms](https://raw.githubusercontent.com/kamidox/blogs/master/images/kaggler_hist.png)

突出的部分是什么？可能是数据的均值，而数据的组织者使用均值来填充这些缺失值造成的。这就需要我们进一步计算并求证。如果真的是这个情况，我们需要反向处理，比如这样处理缺失值是否合理？还是我们要用另外的缺失值处理方式，比如填充成 -999 之类的。此外，我们还可以创建一个新的特征 is_missing ，把包含缺失值的样本标记一下，这种方法对 linear model 有很大的帮助。

以下这些是常用的数据分析函数：

* 柱状图：plt.hist()
* 数据走势图：plt.plot(x, '.')
* 数据统计信息：df.describe(), df.mean(), df.var()
* 其他工具函数：x.value_counts(), x.isnull()

#### 特征关联关系分析

我们可以使用 `plt.scatter(x1, x2, c=y)` 来画出 x1 和 x2 两个特征的关系，从中找出一些规律。此外，我们还可以把训练样本和测试样本都画在同一个点阵图里，查看他们的关系。比如下图，红色表示一个类别，蓝色表示另外一个类别，灰色表示的是测试数据。因为我们没有测试数据的类别情况，所以是灰色的。

![scatter](https://raw.githubusercontent.com/kamidox/blogs/master/images/kaggle_scatter.png)

我们可以看到，测试数据的一部分和红色数据重叠在一起，另外一部分和蓝色数据重叠在一起，这是好的现象，说明我们的训练数据很好地覆盖了这部分数据。但右上角有一部分灰色数据单独分布，没有和我们的训练数据重叠，这是不好的现象，说明我们的训练样本没有覆盖这部分数据，这样我们用这个训练数据做出来的模型对没有覆盖的这部分数据性能会很差。

再如下面的特征关系图，我们可以可以怎么样使用呢？

![scatter 2](https://raw.githubusercontent.com/kamidox/blogs/master/images/kaggle_scatter_2.png)

首先，这样的关系图说明 x1 和 x2 之间是什么关系？实际上，说明 $x1 <= 1 - x2$（在红色以下，而红色就是 $x1 + x2 = 1$）。那么如何使用这层关系呢？这个要根据业务模型来决定，但我们可以利用这个关系，创建一个新的特征，比如 x1 和 x2 的比值，这样的新特征会对 tree-based 模型有帮助。

以下这些是常用的数据分析函数：

* 数据分布图：plt.scatter(x1, x2, c=y)
* 多特征数据分布图：pd.scatter_matrix(df)
* 特征相关性：df.corr()
* 矩阵可视化：plt.matshow(df.values)
* 查看不同特征的均值变化趋势：df.mean().sort_values().plot(style=’.’)

TODO: 探索 `df.corr`，查找特征关联性。

#### 总结

这是一个 EDA checklist:

* 获得总是的领域知识
* 获取数据的直观感受
* 理解数据是怎么产生和采集的，从而帮助我们创建新的特征
* 逐个地探索特征
* 探索多个特征的关系
* 发现缺失值，重复值

#### 示例 Springleaf EDA

w2_007_Springleaf competition EDA II_EDA_Springleaf_screencast.ipynb 是一个值得一看的 EDA 示例，几个有意思的技巧总结如下：

**删除 ID 类特征**
我们可以把那些 ID 类的特征删除。因为 ID 类的特征会误导模型。ID 类特征的特点是，每个数据样本，它的值都是唯一的。可以使用下面的代码来找出这些 ID 类的特征：

```python
feats_counts = train.nunique(dropna = False)
feats_counts.sort_values()[:10]
```

**删除重复特征**
先对使用 `factorize()` 对数据进行 categorical 编码处理，然后针对所有的特征，两两比较是否相同，并且记录这些具有相同值的特征。最后把这些值相同的特征删除掉。

**决定数据类型**
决定数据类型有时不止一个方案，比如对 1, 2, 3 这样的数值，可以是 numerical feature ，也可以是 categorical feature。一个常用的方法是使用 `nunique = train.nunique(dropna=False)` 算出每个特征的值的个数，如果个数很大，很有可能是 numerical feature。如果个数很小，则可能是 categorical feature。除此之外，还可以使用 `plt.hist(nunique.astype(float)/train.shape[0], bins=100)` 画出柱状图，以便更形象地观察。

**其他**
有时我们通过 EDA 可以发现一些数据的模式，比如发现某些累加值，如第一个月销售量，前两个月累计销售量，前三个月累计销售量，linear mode 可以很好地处理这种值，但 tree-based 模型就会有问题。对 tree-based 模型更友好的是转换为每个月的销量。

## 模型检验和过拟合

经常参与 kaggle 竞赛的同学会发现一个有趣的现象，很多人在 public leaderboard 上评分很高，但是在真正竞赛结果出来时，在 private leaderboard 上却掉了下来。这里的原因，就是模型没有经过很好的检验，针对 public test data 过拟合，最终导致分数下降。

本章的主要内容：

* 什么是模型检验和过拟合
* 通过怎么样的数据切分，来建立稳定的模型检验系统
* 常用的数据切分方法
* 常见的模型检验问题

在 kaggle 竞赛中，模型的过拟合通常意见的模型过拟合不同。通常意义上，模型过拟合是指模型针对训练数据集很准确，但对测试数据集性能表现较差。而在 kaggle 竞赛中，过拟合是指模型对 public test dataset 分数较高，但对 private test dataset 没有达到预期的效果。从而导致排名从 public leaderboard 上较高的位置跌落到 private leaderboard 上较低的位置。

### 模型检验策略

为确保模型准确性，我们需要建议**稳定**的模型检验系统。我们一般把训练数据分成两部分，一部分是训练数据集（train dataset），另外一部分称为检验数据集（validation dataset）。常用的划分方法有：

* Holdout: 固定地划分一定比例的数据作为检验数据集。scikit-learn 的 `train_test_split` 以及 `ShuffleSplit` 是实现这种划分的理想函数。当数据量较大时，可以使用这种方法。
* K-fold: 把模型划分成 K 份，然后拿其中的 K - 1 份来当训练数据集，另外 1 份当检验数据集。重复上述过程，只至 K 份数据的每一份，都充分过检验数据集。最后求得 K 次模型准确性评分的平均值。scikit-learn 的 `KFold` 类可以实现这种划分。需要注意，这种划分，每次检验模型准确性，都需要进行 K 次训练和 K 次检验，需要花费额外的训练数据。K-fold 和进行 K 次 Holdout 有什么区别呢？区别在于，K-fold 可以确保样本中的每个数据都只有一次作为模型检验数据集，而 K 次 Holdout 的结果是，有些样本做了多次检验数据集，有些样本根本没有机会作为模型检验数据集。
* Leave-one-out: 每次从数据集中拿**一个**样本出来当模型检验数据集，其余的数据作为训练样本。重复上述过程，直到所有的样本都当过一次模型检验数据集。这个方法类似 K-fold，当 `KFold` 的参数 `n_splits` 为样本数据个数时，K-fold 就退化为 Leave-one-out 方法。scikit-learn 的 `LeaveOneOut` 可以实现这种数据划分。这种划分方法，适用于数据量较小，且模型训练速度足够快的场合。

此外，stratification 方法帮助我们建立更稳定的模型检验方法，特别是针对数据量较小，数据**不平衡**的时候。stratification 方法的核心思想是，确保 K-fold 划分后的数据集里，每份数据都有差不多的分布。比如针对二元分类，它保证 0 和 1 目标的样本分布概率接近。scikit-learn 里的 `StratifiedKFold` 可以实现这种数据划分。

### 数据划分策略

假设，我们有过去几个月销量数据，需要预测下个月的销量。针对这样的 time serial 数据集，我们有两种划分方法，一种是随机取出一部分作为模型检验数据集，另外一个方法是按时间序列，把某个时间之前的作为训练数据集，某个时间之后的作为检验数据集。这两种划分方法训练出来的模型有差别么？

![split strategy](https://raw.githubusercontent.com/kamidox/blogs/master/images/kaggler_split_strategy.png)

答案是**差别巨大**。为什么呢？

想像一下，模型训练过程中，我们是通过不断进行特征选择，优化参数，使得针对交叉验证数据集的评分越来越高。故，针对第一种方法，我们的模型将很擅长预测随机的缺失值。模型所选择出来的特征，也是对这种预测友好的。方法训练出来的模型，如果要去预测未来几个连续点的值，将会表现得很差。相反，第二种方法，我们的模型将被训练成擅长预测未来一段时间的多个缺失值。故，第二种方法对我们要解决的问题更有利。

长期来看，模型预测值将和训练数据集里的目标值的平均值接近。针对第一种情况，交叉验证数据集的平均值更接近训练数据集目标值的平均值，而测试数据集的平均值则离训练数据集的平均值较远。故，第一种情况下，模型对交叉验证数据集的预测结果要好于对测试数据集的预测结果。针对第二种情况，交叉验证数据集离训练数据集的目标值的平均值比较远，测试数据集离训练数据集的目标值平均值也比较远，故模型针对交叉验证数据集的预测和对测试数据集的预测的准确性将差不多，这有助于帮助我们建立一个更稳定的交叉验证模型。从而使算法具有更大的泛化性，避免过拟合。

所以，不恰当的交叉验证数据集的划分，将导致特征以及模型偏离我们的问题目标，从而造成我们意料之外的后果。**针对 time serial 数据集，我们应该使用 time-based split 方法来划分训练数据集和交叉验证数据集**。更一般化的结论，我们需要根据具体的问题，或者说针对 kaggle 竞赛的测试数据的产生机制，在我们模型交叉验证阶段，就模拟这种机制，以便我们的模型能更好地预测测试数据集。

常用的交叉验证数据集的划分策略有以下几种：

**按行随机选择**
这是最常用的一种划分方法。这种方法需要确保每行数据是独立的，不存在依赖关系。

**时间序列划分**
这种方法开头的例子里讨论过了。如果数据量足够多，我们可以使用“滑动窗口”交叉验证机制。如，使用第 1, 2, 3 周的数据作为训练数据集，第 4 周的数据作为交叉验证数据集。然后使用第 2, 3, 4 周的数据作为训练数据集，使用第 5 周的数据作为交叉验证数据集。

**基于 ID 的划分**
假设，我们有一个根据造影图片，预测病人疾病的任务。每个病人都有多个不同的图片，每个图片当作一个样本。我们在划分交叉验证数据集时，如果随机划分，那么可能把一个病人的部分图片划分到训练数据集里，另外一部分图片划分到交叉验证数据集里。这样的结果是造成了训练数据集和交叉验证数据集的数据“重合”，从而非常容易造成模型过拟合。正确地划分方法，应该是根据人来划分，同一个人的所有图片，要么都在训练数据集里，要么都在交叉验证数据集里。这就是基于 ID 的划分方法。其总体思想，是根据某个类别来划分，从而避免造成训练数据集和交叉验证数据集数据重叠。有时，这种基于 ID 的划分方法，ID 并不是现成的，需要我们自己去发现。比如，有些图片分类数据里，图片是连续拍摄的，这些图片的差异性很小，那么这些相似的图片应该被归到同一组里去。

**总之，我们划分出来的交叉验证数据集，需要尽量的模仿测试数据集的产生机制**。





