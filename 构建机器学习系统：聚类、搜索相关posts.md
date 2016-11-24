# 构建机器学习系统：聚类、搜索相关posts #

----------
## 1 介绍 ##
本章主要介绍了如何使用非监督的 **聚类** 方法对大量文档进行分类。

## 2 度量手段 ##
首先介绍了 **Levenshtein距离** 。具体操作方法为:比较两个文档之间的异同。每删除、添加一个单词则认为操作数加一。最终，能够使得一个文档转变为另一个文档的操作总数即为两个文档的距离。如，'How to format my hard disk'与'Hard disk format problems'的距离为5。

但是，这种度量方式存在明显的弊端，在上例中，'format'被操作了两次。 所以引入更好的距离度量规则是十分必要的。

于是在这里引入了 **'bag of words'** 。操作为：汇总所有文档中的词汇，再依据每个文档中词汇出现的频数构建词向量，词向量是稀疏的。之后，对生成的向量进行各种距离的求算，即可实现文档之间相似度的度量。

## 3.1 文档数据的预处理 ##
本节的目的是将已有的大量文档转化成文档-词矩阵。这里直接使用 **Scikit库** 的 **CountVecotorizer** 来实现。

首先导入**CountVecotorizer**模块：

    >>> from sklearn.feature_extraction.text import CountVecotorizer
    >>> vectorizer = CountVectorizer(min_df=1)

紧接着，我们在这里实例化了向量生成器，设置了**min-df参数**。此参数决定了生成器究竟如何处理文档，如果被设置成了整数。**频数** 低于此数值的词不会被文档-词模型采纳；如果被设置成了分数，则 **频率** 低于此数值的词不会被采纳。同时，也可以设置max_df参数来控制被采纳词所能出现的最高次数和最大频率。

    >>> print(vectorizer)
	CountVectorizer(analyzer=word, binary=False, charset=utf-8,
	charset_error=strict, dtype=<type 'long'>, input=content,
	lowercase=True, max_df=1.0, max_features=None, max_n=None,
	min_df=1, min_n=None, ngram_range=(1, 1), preprocessor=None,
	stop_words=None, strip_accents=None, token_pattern=(?u)\b\w\w+\b,
	tokenizer=None, vocabulary=None)

这里可以用print方法来查看生成器的参数设置信息。其中可以看到，analyzer参数被默认地设置为word，token\_pattern 决定了该生成器截取词汇的模式（以正则表达式给出）。在这里，例如像'cross\_validated'会被直接划分为'cross'和'validated'。

接下来对我们先前的两个文档'How to format my hard disk'与'Hard disk format problems'进行向量化处理。

    >>> content=['How to format my hard disk','Hard disk format problems']
    >>> x=vectorizer.fit_transform(content)
   
之后，打印对象x可以看到x的类型和基本信息

    <2x7 sparse matrix of type '<type 'numpy.int64'>'
		with 10 stored elements in Compressed Sparse Row format>

显示了，x为一个2*7的稀疏矩阵对象。

利用get\_feature\_names()方法则可以查看向量生成器所截获的特征词汇。

    >>> vectorizer.get_feature_names()
	[u'disk', u'format', u'hard', u'how', u'my', u'problems', u'to']

x对象可以转化成更加实用的的 **numpy.array对象** 。
    
    >>>print(X.toarray().transpose())
    array([[1, 1],
    [1, 1],
    [1, 1],
    [1, 0],
    [1, 0],
    [0, 1],
    [1, 0]], dtype=int64)

这样就可以对照上面的'feature names'清楚地看出向量中的词汇分布：左侧向量代表该文档中没有'problem'

## 3.2 词汇频数向量 ##
首先假设变量DIR存储了我们的数据根目录字符串，之后将这个路径文件迭代地下的数据加入到 **os.path** 变量中，之后利用 **open, read** 方法读取。将每个文档作为一个字符串存储在 **posts** 列表中，之后照例初始化向量生成器。

    >>> posts = [open(os.path.join(DIR, f)).read() for f in os.listdir(DIR)]
    >>> from sklearn.feature_extraction.text import CountVectorizer
    >>> vectorizer = CountVectorizer(min_df=1)

之后进行的将整个列表转化成文档-词矩阵。
    
    >>> X_train = vectorizer.fit_transform(posts)

之后不妨直接观察一下得到的矩阵的信息。

    >>> num_samples, num_features = X_train.shape
	>>> print("#samples: %d, #features: %d" % (num_samples,num_features)) 
	#samples: 5, #features: 25

可以看到，我们的矩阵中含有5条文档，正好对应了我们目录下面的5个文本文件，特征数为25可以知道，整个文档中一共含有25个词汇。

我们接下来的一个工作就是，创建一个新的文档。

	>>> new_post = "imaging databases"
	>>> new_post_vec = vectorizer.transform([new_post])

其中的 **transform** 方法是依照我们之前训练得到的矩阵，将'imaging databases'转化成稀疏向量的操作。但是，该方法返回的稀疏向量并非是一个 **numpy.array对象** 。

    >>> print(new_post_vec)
	(0, 7)1
	(0, 5)1

利用元组对象和数字直接标注了稀疏矩阵中非零元素的位置和内容。

当然，这里 **.toarray()** 方法也是容许的。

下面，我们就要进入相似性度量部分了，我们将利用到 **Scipy** 中的方法来构建合适的度量函数并计算
矩阵中新加入的向量和旧向量之间的相似度。我们先导入scipy模块并构建距离度量函数。

    >>> import scipy as so
    >>> def dist_raw(v1, v2):
    		delta=v1-v2
			return sp.linalg.norm(delta.toarray())

这里，我们选用 **sp.linalg.norm()** 方法来计算向量的欧几里得距离，欧几里得距离越小，说明两个向量之间的距离越小，即两个文档之间的相似度越高。接下来，我们的工作就是遍历所有的旧文档，并且记录下新文档和旧文档中的最小距离。

    >>> import sys
    >>> best_doc=None
    >>> best_dist=sys.maxint
    >>> best_i=None
    >>> for i in range(0, num_samples):
			post=post[i]
			if post==new_post:
				continue
			post_vec=X_train.getrow(i)
			d=dist(post_vec, new_post_vec)
			if d<best_dist:
				best_dist=d
				best_i=i

向循环加入适当的 **print** 语句，则可以将每次计算的结果输出到屏幕。

但是，我们这里采取的度量方法实际上是有缺陷的，例如选择如下两个文档：
    
    >>> print(X_train.getrow(3).toarray())
	[[0 0 0 0 1 1 0 1 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 0 0]]
	>>> print(X_train.getrow(4).toarray())
	[[0 0 0 0 3 3 0 3 0 0 0 0 0 0 0 0 0 0 0 0 0 3 0 0 0]]

它们的原文分别是： **'Imaging databases store data.'** 和 **'Imaging databases store data.Imaging databases store data.Imaging databases store data.'** 这两个文档从直观上来看，计算得到的距离必然很大，然而从实际内容上看，二者完全相同，距离应该为正。

## 3.3 频数向量归一化 ##

该步骤的工作实际上非常简单，只是简单地改造一下我们的 **dist_norm()** 函数：

    >>> def dist_norm(v1, v2):
			v1_normalized = v1/sp.linalg.norm(v1.toarray())
			v2_normalized = v2/sp.linalg.norm(v2.toarray())
			delta = v1_normalized - v2_normalized
			return sp.linalg.norm(delta.toarray())

注意，我们把输入向量的长度统一地缩放为1，这样上述的重复文档之间的距离就为0，避免了误差。

## 3.4 去除高频词汇 ##

我们在矩阵中的第二号文档和新输入文档的内容分别如下：
> Post2: ***Most imaging databases safe images permanently***
> 
> New Post: ***imaging databases***

这句话中的 ***most*** 一词，实际上是英语中各类语境中的高频词汇，这样的词汇并不能提供十分多的额外信息。那么从自然语言处理的角度看，它至少在文档中的权重是没有 ***image*** 来的大，这样的词常常被称作 **stop words** 而在预处理阶段的一个十分重要的任务，就是让向量生成器自动过滤掉这些高频的 **stop words** 

在python中，我们通过修改，CountVectorizer对象的参数来实现过滤：

    >>> vectorizer = CountVectorizer(min_df=1, stop_words='english')

在这里, **stopwords** 参数可以被赋予一个词汇列表，之后里面的词汇会被明确地指定为 **stop words** 并被忽略。**'english'** 是一个已经设置好的，具有318个常见词汇的列表。**vectorizer.get_stop_words()** 方法可以被用来查看向量生成器所使用的列表。

## 3.5 词干和NLTK模块 ##

还有一个问题被我们遗漏了： 词干的问题。 例如像'images' 和 'imaging'这两个词，他们代表的含义是十分近似的。但是显然，我们先前的方法不能察觉这种相似性，而只是机械地将二者视为两个词。为此，我们需要一个方法将每一个词尽可能地**转化成它的词干**，这就用到了python的另一个库————N.L.T.K。

NLTK为我们提供了多种不同的stemmer。每种语言的词干指定方法都是不同的，对于英语，我们只需选择SnowballStemmer。

下面我们就利用'images' 和 'imaging'这两个词来测试一下词干提取器。

    >>> import nltk.stem
    >>> s=nltk.stem.SnowballStemmer('english')
    >>> s.stem('imaging')
    u'imag'
	>>> s.stem('images')
	u'imag'
	>>> s.stem('imagine')
	u'imag'

- 注：词干提取工作不必仅仅针对有效的英语词汇进行，一些不完整的英文词汇片段实际上也可以进行词干提取。


## 3.6 利用NLTK的stemmer来扩展CountVectorizer ##

首先，为了能够更好地将stemmer加入到vectorizer中，我们在此覆盖build_analyzer方法（过程比较繁琐）

    >>> import nltk.stem
	>>> english_stemmer = nltk.stem.SnowballStemmer('english')

首先，我们在这里实例化了一个英文stemmer。

	>>> class StemmedCountVectorizer(CountVectorizer):
			def build_analyzer(self):
				analyzer = super(StemmedCountVectorizer, self).build_analyzer()
				return lambda doc: (english_stemmer.stem(w) for w in analyzer(doc))

之后，我们继承CountVectorizer类，简历StemmedCountVectorizer子类。我们仅在此写一个额外的方法————build_analyzer。这样，我们就可以使用我们新创建的Stemmer版本的向量生成器来处理文档了。

	>>> vectorizer = StemmedCountVectorizer(min_df=1, stop_words='english')


这个向量产生器子类将会实现三种功能：

- 将所有字母转化为小写字母
- 提取所有单个词汇
- 将提取的所有词汇转化为词干版本

之后，我们可以查看一下新的vectorizer的feature name属性：

    >>> [u'actual', u'capabl', u'contain', u'data', u'databas', u'imag', u'interest', u'learn', u'machin', u'perman', u'post', u'provid', u'safe', u'storag', u'store', u'stuff', u'toy']

可见，很多feature name被转化成了词根的形式。进一步考察词汇向量之间的距离，可以发现，距离被明显地缩小了。


## 3.7 思考高频词汇 ##

下面，我们来深入思考一下文档特征数字（对应特征下的频数，即词汇向量中的分量数值）的含义。特征数字代表了“词汇”在文档中出现的频数，之前我们只是简单地认为某个词汇的频数越高、则出现的可能性越大进而说明词汇对文档的重要性越高。但是，对于那些出现频率过高的词汇呢？

诚然，我们可以通过设置参数 **max df** 来滤掉这些词汇，比如说设置 max_df=90。然而对于一个在文章中出现了89次的词汇呢？

我们在这里提供一个方法： 我们只允许那些在某一篇文章中出现了很多次但是在其他的文档中几乎不出现的那些词出现很高的分量数值。 然而对于那些在所有文档中出现频率都很高的词汇，我们将直接过滤掉。

这个方案就是所谓的 **'TF-IDF'(term frequency - inverse document frequency)** 方案。 如下就是此方案的一个简单实现。

    >>> import scipy as sp
	>>> def tfidf(term, doc, docset):
			tf = float(doc.count(term))/sum(doc.count(w) for w in docset)
			# 计算指定词汇的term frequency——全体文档中的出现频率
			idf = math.log( float(len(docset))/(len([doc for doc in docset if term in doc])) )
			# 计算指定词汇的inverse document frequency——用出现了指定词汇的文档数量除以全体文档数量（这里的处理是为了idf值为正值）
			return tf * idf

接下来我们不妨查看一下此方案的效果：

    >>> a, abb, abc = ["a"], ["a", "b", "b"], ["a", "b", "c"]
	>>> D = [a, abb, abc]

我们首先设立了三个文档：a, abb, abc。 之后创建了D，将D作为文档-词矩阵。之后，我们计算此研究条件下的词汇a对于各个文档的tfidf值：

    >>> print(tfidf("a", a, D))
    >>> 0.0
	>>> print(tfidf("a", abc, D))
	>>> 0.0
	>>> print(tfidf("a", abb, D))
	>>> 0.0

可以看出，对于'a'这样的一个十分trivial的词汇，tfidf值在对于各个文档而言都是0.0。 然而词汇b和词汇c则有不同的结果：

    >>> print(tfidf("b", abb, D))
	>>>	0.270310072072
	>>> print(tfidf("c", abc, D))
	>>>	0.366204096223

但是在Scikit库中，我们有TfidfVectorizer对象可以实现利用TFIDF方法统计词汇频率。然而，为了加入词干机制，我们必须对TfidfVectorizer实施和上面相同的创建子类的方法。


## 3.8 文档-词模型的弱点 ##

1. 模型完全忽略了词汇之间的联系。'Car hits wall' 和 'Wall hits car' 的文档向量是完全相同的。
2. 模型不能正确地捕捉否定词给句子带来的差异。'I will eat ice cream'和'I will not eat ice cream' 的巨大差异在这个模型中无法体现。这个问题可以通过bigrams模式来解决（不分析单个词汇，而去分析二元词组）
3. 模型对于错误的拼写没有任何的敏感性（无法察觉错误拼写情况）。


## 4.1 聚类 ##

聚类分析一般有两种方法：
1. flat聚类：将数据分成一组聚类。这里的目标是提出一种划分方式，使得聚类之内的数据最相似而聚类之间不相似。此类方法一般需要事先指定聚类数目。
2. 层次聚类： 无需事先指定聚类数目。相似的数据被指定为一个聚类，相似的聚类再被指定成同一个父聚类直到最后仅仅剩下一个聚类，包含了所有的数据。

接下来我们将使用K-Means算法来实现flat聚类。

## 4.2 K均值算法 ##

先指定聚类中心，聚类中心的数量和用户指定的聚类数目是相同的。之后计算所有其他的向量和各个聚类中心的距离，并将向量归结到和它最近的聚类中心所代表的聚类中去。

得到聚类之后重新计算聚类中心，使得聚类中心位于聚类的中央（取向量的均值）。这样的话，向量所归属的聚类就有可能发生改变。所以要按照新的聚类中心来划分新的聚类。直到聚类中心不再明显地发生改变。

## 4.3 利用测试数据证实想法 ##

MLComp（http://mlcomp.org/datasets/379）和Scikit库有很好的交互，Scikit库中甚至有专门的为此网站的数据架设的读取方法。

我们将数据下载为ZIP格式压缩包解压后执行下列程序。

    >>> import sklearn.datasets
    >>> MLCOMP_DIR='F:\' # 设定你自己的379文件夹的根目录
    >>> data=sklearn.datasets.load_mlcomp('20news-18828', mlcomp_root=MLCOMP_DIR)
    >>> print(data.filenames)

之后，我们将要在载入的数据中选取训练数据和测试数据。

    >>> train_data=sklearn.datasets.load_mlcomp('20news-18828', 'train', mlcomp_root=MLCOMP_DIR)
    >>> test_data=sklearn.datasets.load_mlcomp('20news-18828', 'test', mlcomp_root=MLCOMP_DIR)
    
为了使得测试过程更加快速，我们在此限定了文本的主题范围。

    >>> groups = ['comp.graphics', 'comp.os.mswindows.misc', 'comp.sys.ibm.pc.hardware', 'comp.sys.ma c.hardware', 'comp.windows.x', 'sci.space']
    >>> train_data=sklearn.datasets.load_mlcomp('20news-18828', 'train', mlcomp_root=MLCOMP_DIR, categories=groups)
    

## 4.4 将文档聚类 ##

现实中的数据往往是杂乱无章的。甚至又可能存在无效字符而引发UnicodeDecodeError这样的错误。

这样我们就必须告诉vectorizer去忽略这些错误：

    >>> vectorizer = StemmedTfidfVectorizer(min_df=10, max_df=0.5, stop_words='english', charset_error='ignore')
    >>> vectorized = vectorizer.fit_transform(dataset.data)
    >>> num_samples, num_features = vectorized.shape
    >>> print("samples: %d, features: %d" % (num_samples, num_features))
	samples: 3414, #features: 4331

这样，我们的文档-词矩阵中就有了3414条文档，从中共提取出了4331个符合要求的词汇，形成了4331个特征。得到的文档-词矩阵正好是K-Means方法的输入，同时不要忘记设置聚类数目，这里我们直接设置为50。

    >>> num_clusters = 50
	>>> from sklearn.cluster import KMeans
	>>> km = KMeans(n_clusters=num_clusters, init='random', n_init=1, verbose=1)
	>>> km.fit(vectorized)

随后我们可以通过查看km.labels_变量来查看每个文档的聚类归属情况; km.cluster_centers_则提供了最终的聚类中心的信息。

## 4.5 用新的信息来测试模型 ##

首先，我们先创建一个新的文档：new_post

    >>> new_post='''
    Disk drive problems. Hi, I have a problem with my hard disk.
	After 1 year it is working only sporadically now.
	I tried to format it, but now it doesn't boot any more.
	Any ideas? Thanks.
	'''

之后用我们事先训练好的vectorizer和K-Means来创建文档向量、预测新文档的聚类情况。

	>>> new_post_vec = vectorizer.transform([new_post])
	>>> new_post_label = km.predict(new_post_vec)[0]

既然文档有了明确的聚类归属，那么就不需要再将新文档和所有聚类进行比较了。我们这里不妨提取文档所归属的聚类。

    >>> similar_indices = (km.labels_==new_post_label).nonzero()[0]

- 注：这里第一个括号其实是一个由bool值组成的向量，nonzero()方法则返回了这个向量中的True元素的索引。

接下来，我们将要写一个循环来生成一个同时包含文档和对应的相似度的列表。

    >>> similar = []
	>>> for i in similar_indices:
			dist = sp.linalg.norm((new_post_vec - vectorized[i]).toarray())
			similar.append((dist, dataset.data[i]))
	>>> similar = sorted(similar)
	>>> print(len(similar))
	44

这样，我们得到的similar列表就包含了文档和对应的相似度，并且列表是按照相似度的大小顺序排序的。

