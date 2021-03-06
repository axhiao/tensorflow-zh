# Vector Representations of Words <a class="md-anchor" id="AUTOGENERATED-vector-representations-of-words"></a>

本教程将讲解如下论文所描述之word2vec模型
[Mikolov et al.](http://papers.nips.cc/paper/5021-distributed-representations-of-words-and-phrases-and-their-compositionality.pdf)
此模型用于学习词汇的向量表示，亦称为词嵌套(word embeddings)。

## Highlights <a class="md-anchor" id="AUTOGENERATED-highlights"></a>

此一教程将着重强调使用TensorFlow建起word2vec模型之过程中最为核心及有趣的部分。

* 首先，本文给出将词汇表达为向量的原因；
* 第二，本文给出对此一模型的直观理解及模型的训练方法（其中包含必要的数学推导）；
* 然后，本文展示word2vec模型在TensorFlow中的一个简单实现；
* 最后，本文探讨如何扩展此一简单模型，使之适应不同数据集。

本文稍后将逐行讲解代码，不过如果您想先睹为快，可以直接参考以下链接：
[tensorflow/g3doc/tutorials/word2vec/word2vec_basic.py](https://tensorflow.googlesource.com/tensorflow/+/master/tensorflow/g3doc/tutorials/word2vec/word2vec_basic.py)
这一简单范例包含下载数据，简单训练模型及将结果可视化所需之代码。在您已经充分阅读并运行这一范例之后，可以进阶至：
[tensorflow/models/embedding/word2vec.py](https://tensorflow.googlesource.com/tensorflow/+/master/tensorflow/models/embedding/word2vec.py)
这一更加复杂的范例将展示TensorFlow的一些高阶应用，例如如何利用线程将数据导入文本模型，如何在训练过程中进行检验等等。

不过，首先我们要了解词嵌套的重要意义。如果您已经是一位词嵌套达人，则尽可以跳过本节，直接进入下文相关章节以了解模型细节。

## 目的: 为什么要学习词嵌套? <a class="md-anchor" id="AUTOGENERATED-motivation--why-learn-word-embeddings-"></a>

图像及音频处理系统通常处理大量高维数据集；其中，图像数据一般编码为单一原始像素强度所组成之向量，而音频数据则可根据功率谱密度系数编码为向量。对于物体或语音识别一类的任务，我们所需的全部信息都已经存在于原始数据中（人类本身就是依赖原始数据进行日常的物体或语音识别的）。然而，自然语言处理系统传统上将词汇作为离散的单一符号，例如‘cat'一词或可被表示为’Id537‘，而’dog'一词则可能为‘Id143’。此类符号编码毫无规律，无法提供不同词汇之间可能存在的关联信息。换句话说，在处理关于‘dogs’一词的信息时，模型将无法利用已知的关于‘cats'的信息（例如，它们都是动物，有四条腿，可作为宠物等等）。可见，将词汇表达为上述的独一离散符号将进一步导致数据稀疏，使我们在训练统计模型时不得不寻求更多的数据。而词汇的向量表示将克服上述的难题。

<div style="width:100%; margin:auto; margin-bottom:10px; margin-top:20px;">
<img style="width:100%" src="img/audio-image-text.png" alt>
</div>

[向量空间模型](https://en.wikipedia.org/wiki/Vector_space_model) (VSMs)
将词汇表达（嵌套）于一个连续的向量空间中，语义近似的词汇将被映射为相邻的数据点。向量空间模型在自然语言处理领域中有着漫长且丰富的历史，不过几乎所有利用这一模型的方法都依赖于
[分布式假设](https://en.wikipedia.org/wiki/Distributional_semantics#Distributional_Hypothesis),
其核心思想为出现于相同上下文情境中的词汇都有相类似的语义。采用这一假设的研究方法大致分为以下两类：
*基于计数的方法* (e.g.
[潜在语义分析](https://en.wikipedia.org/wiki/Latent_semantic_analysis)),
及 *预测方法* (e.g.
[神经概率化语言模型](http://www.scholarpedia.org/article/Neural_net_language_models)).

其中分野在如下论文中有详细阐述
[Baroni et al.](http://clic.cimec.unitn.it/marco/publications/acl2014/baroni-etal-countpredict-acl2014.pdf),
不过简而言之：基于计数的方法计算某词汇与其邻接词汇在一个大型语料库中共同出现的频度及其他统计量，然后将这些统计量映射到一个小型且稠密的向量中。而预测方法则试图直接从某词汇的临近词汇对其进行预测，在此过程中利用已经学习到的小且稠密的嵌套向量。

Word2vec是一种可以进行高效率词嵌套学习的预测模型。其两种变体分别为：连续词袋模型（CBOW）及Skip-Gram模型。从算法角度看，这两种方法极为类似，其区别为CBOW根据源上下文词汇（‘the cat sits on the’）来预测目标词汇（例如，‘mat’），而Skip-Gram模型反之。Skip-Gram模型采取CBOW的逆过程的动机在于：CBOW算法对于很多分布式信息进行了平滑处理（例如将一整段上下文信息视为一个单一观察量）；对于小型的数据集，这一处理无可厚非。相形之下，Skip-Gram将每个（上下文-目标词汇）对，视为一个新观察量，而这一处理对于大型数据集极为有效。本教程余下部分将着重讲解Skip-Gram模型。



## Scaling up with Noise-Contrastive Training <a class="md-anchor" id="AUTOGENERATED-scaling-up-with-noise-contrastive-training"></a>

Neural probabilistic language models are traditionally trained using the
[maximum likelihood](https://en.wikipedia.org/wiki/Maximum_likelihood) (ML)
principle  to maximize the probability of the next word \\(w_t\\) (for "target")
given the previous words \\(h\\) (for "history") in terms of a
[*softmax* function](https://en.wikipedia.org/wiki/Softmax_function),

$$
\begin{align}
P(w_t | h) &= \text{softmax}(\exp \{ \text{score}(w_t, h) \}) \\
           &= \frac{\exp \{ \text{score}(w_t, h) \} }
             {\sum_\text{Word w' in Vocab} \exp \{ \text{score}(w', h) \} }.
\end{align}
$$

where \\(\text{score}(w\_t, h)\\) computes the compatibility of word \\(w\_t\\)
with the context \\(h\\) (a dot product is commonly used). We train this model
by maximizing its log-likelihood on the training set, i.e. by maximizing

$$
\begin{align}
 J_\text{ML} &= \log P(w_t | h) \\
  &= \text{score}(w_t, h) -
     \log \left( \sum_\text{Word w' in Vocab} \exp \{ \text{score}(w', h) \} \right)
\end{align}
$$

This yields a properly normalized probabilistic model for language modeling.
However this is very expensive, because we need to compute and normalize each
probability using the score for all other \\(V\\) words \\(w'\\) in the current
context \\(h\\), *at every training step*.

<div style="width:60%; margin:auto; margin-bottom:10px; margin-top:20px;">
<img style="width:100%" src="img/softmax-nplm.png" alt>
</div>

On the other hand, for feature learning in word2vec we do not need a full
probabilistic model. The CBOW and skip-gram models are instead trained using a
binary classification objective (logistic regression) to discriminate the real
target words \\(w_t\\) from \\(k\\) imaginary (noise) words \\(\tilde w\\), in the
same context. We illustrate this below for a CBOW model. For skip-gram the
direction is simply inverted.

<div style="width:60%; margin:auto; margin-bottom:10px; margin-top:20px;">
<img style="width:100%" src="img/nce-nplm.png" alt>
</div>

Mathematically, the objective (for each example) is to maximize

$$J_\text{NEG} = \log Q_\theta(D=1 |w_t, h) +
  k \mathop{\mathbb{E}}_{\tilde w \sim P_\text{noise}}
     \left[ \log Q_\theta(D = 0 |\tilde w, h) \right]$$

where \\(Q_\theta(D=1 | w, h)\\) is the binary logistic regression probability
under the model of seeing the word \\(w\\) in the context \\(h\\) in the dataset
\\(D\\), calculated in terms of the learned embedding vectors \\(\theta\\). In
practice we approximate the expectation by drawing \\(k\\) constrastive words
from the noise distribution (i.e. we compute a
[Monte Carlo average](https://en.wikipedia.org/wiki/Monte_Carlo_integration)).

This objective is maximized when the model assigns high probabilities
to the real words, and low probabilities to noise words. Technically, this is
called
[Negative Sampling](http://papers.nips.cc/paper/5021-distributed-representations-of-words-and-phrases-and-their-compositionality.pdf),
and there is good mathematical motivation for using this loss function:
The updates it proposes approximate the updates of the softmax function in the
limit. But computationally it is especially appealing because computing the
loss function now scales only with the number of *noise words* that we
select (\\(k\\)), and not *all words* in the vocabulary (\\(V\\)). This makes it
much faster to train. We will actually make use of the very similar
[noise-contrastive estimation (NCE)](http://papers.nips.cc/paper/5165-learning-word-embeddings-efficiently-with-noise-contrastive-estimation.pdf)
loss, for which TensorFlow has a handy helper function `tf.nn.nce_loss()`.

Let's get an intuitive feel for how this would work in practice!

## The Skip-gram Model <a class="md-anchor" id="AUTOGENERATED-the-skip-gram-model"></a>

As an example, let's consider the dataset

`the quick brown fox jumped over the lazy dog`

We first form a dataset of words and the contexts in which they appear. We
could define 'context' in any way that makes sense, and in fact people have
looked at syntactic contexts (i.e. the syntactic dependents of the current
target word, see e.g.
[Levy et al.](https://levyomer.files.wordpress.com/2014/04/dependency-based-word-embeddings-acl-2014.pdf)),
words-to-the-left of the target, words-to-the-right of the target, etc. For now,
let's stick to the vanilla definition and define 'context' as the window
of words to the left and to the right of a target word. Using a window
size of 1, we then have the dataset

`([the, brown], quick), ([quick, fox], brown), ([brown, jumped], fox), ...`

of `(context, target)` pairs. Recall that skip-gram inverts contexts and
targets, and tries to predict each context word from its target word, so the
task becomes to predict 'the' and 'brown' from 'quick', 'quick' and 'fox' from
'brown', etc. Therefore our dataset becomes

`(quick, the), (quick, brown), (brown, quick), (brown, fox), ...`

of `(input, output)` pairs.  The objective function is defined over the entire
dataset, but we typically optimize this with
[stochastic gradient descent](https://en.wikipedia.org/wiki/Stochastic_gradient_descent)
(SGD) using one example at a time (or a 'minibatch' of `batch_size` examples,
where typically `16 <= batch_size <= 512`). So let's look at one step of
this process.

Let's imagine at training step \\(t\\) we observe the first training case above,
where the goal is to predict `the` from `quick`. We select `num_noise` number
of noisy (contrastive) examples by drawing from some noise distribution,
typically the unigram distribution, \\(P(w)\\). For simplicity let's say
`num_noise=1` and we select `sheep` as a noisy example. Next we compute the
loss for this pair of observed and noisy examples, i.e. the objective at time
step \\(t\\) becomes

$$J^{(t)}_\text{NEG} = \log Q_\theta(D=1 | \text{the, quick}) +
  \log(Q_\theta(D=0 | \text{sheep, quick}))$$.

The goal is to make an update to the embedding parameters \\(\theta\\) to improve
(in this case, maximize) this objective function.  We do this by deriving the
gradient of the loss with respect to the embedding parameters \\(\theta\\), i.e.
\\(\frac{\partial}{\partial \theta} J_\text{NEG}\\) (luckily TensorFlow provides
easy helper functions for doing this!). We then perform an update to the
embeddings by taking a small step in the direction of the gradient. When this
process is repeated over the entire training set, this has the effect of
'moving' the embedding vectors around for each word until the model is
successful at discriminating real words from noise words.

We can visualize the learned vectors by projecting them down to 2 dimensions
using for instance something like the
[t-SNE dimensionality reduction technique](http://lvdmaaten.github.io/tsne/).
When we inspect these visualizations it becomes apparent that the vectors
capture some general, and in fact quite useful, semantic information about
words and their relationships to one another. It was very interesting when we
first discovered that certain directions in the induced vector space specialize
towards certain semantic relationships, e.g. *male-female*, *gender* and
even *country-capital* relationships between words, as illustrated in the figure
below (see also for example
[Mikolov et al., 2013](http://www.aclweb.org/anthology/N13-1090)).

<div style="width:100%; margin:auto; margin-bottom:10px; margin-top:20px;">
<img style="width:100%" src="img/linear-relationships.png" alt>
</div>

This explains why these vectors are also useful as features for many canonical
NLP prediction tasks, such as part-of-speech tagging or named entity recognition
(see for example the original work by
[Collobert et al.](http://arxiv.org/pdf/1103.0398v1.pdf), or follow-up work by
[Turian et al.](http://www.aclweb.org/anthology/P10-1040)).

But for now, let's just use them to draw pretty pictures!

## Building the Graph <a class="md-anchor" id="AUTOGENERATED-building-the-graph"></a>

This is all about embeddings, so let's define our embedding matrix.
This is just a big random matrix to start.  We'll initialize the values to be
uniform in the unit cube.

```python
embeddings = tf.Variable(
    tf.random_uniform([vocabulary_size, embedding_size], -1.0, 1.0))
```

The noise-contrastive estimation loss is defined in terms a logistic regression
model. For this, we need to define the weights and biases for each word in the
vocabulary (also called the `output weights` as opposed to the `input
embeddings`). So let's define that.

```python
nce_weights = tf.Variable(
  tf.truncated_normal([vocabulary_size, embedding_size],
                      stddev=1.0 / math.sqrt(embedding_size)))
nce_biases = tf.Variable(tf.zeros([vocabulary_size]))
```

Now that we have the parameters in place, we can define our skip-gram model
graph. For simplicity, let's suppose we've already integerized our text corpus
with a vocabulary so that each word is represented as an integer (see
[tensorflow/g3doc/tutorials/word2vec/word2vec_basic.py](https://tensorflow.googlesource.com/tensorflow/+/master/tensorflow/g3doc/tutorials/word2vec/word2vec_basic.py)
for the details). The skip-gram model takes two inputs. One is a batch full of
integers representing the source context words, the other is for the target
words. Let's create placeholder nodes for these inputs, so that we can feed in
data later.

```python
# Placeholders for inputs
train_inputs = tf.placeholder(tf.int32, shape=[batch_size])
train_labels = tf.placeholder(tf.int32, shape=[batch_size, 1])
```

Now what we need to do is look up the vector for each of the source words in
the batch.  TensorFlow has handy helpers that make this easy.

```python
embed = tf.nn.embedding_lookup(embeddings, train_inputs)
```

Ok, now that we have the embeddings for each word, we'd like to try to predict
the target word using the noise-contrastive training objective.

```python
# Compute the NCE loss, using a sample of the negative labels each time.
loss = tf.reduce_mean(
  tf.nn.nce_loss(nce_weights, nce_biases, embed, train_labels,
                 num_sampled, vocabulary_size))
```

Now that we have a loss node, we need to add the nodes required to compute
gradients and update the parameters, etc. For this we will use stochastic
gradient descent, and TensorFlow has handy helpers to make this easy as well.

```python
# We use the SGD optimizer.
optimizer = tf.train.GradientDescentOptimizer(learning_rate=1.0).minimize(loss)
```

## Training the Model <a class="md-anchor" id="AUTOGENERATED-training-the-model"></a>

Training the model is then as simple as using a `feed_dict` to push data into
the placeholders and calling
[`session.run`](../../api_docs/python/client.md#Session.run) with this new data
in a loop.

```python
for inputs, labels in generate_batch(...):
  feed_dict = {training_inputs: inputs, training_labels: labels}
  _, cur_loss = session.run([optimizer, loss], feed_dict=feed_dict)
```

See the full example code in
[tensorflow/g3doc/tutorials/word2vec/word2vec_basic.py](./word2vec_basic.py).

## Visualizing the Learned Embeddings <a class="md-anchor" id="AUTOGENERATED-visualizing-the-learned-embeddings"></a>

After training has finished we can visualize the learned embeddings using
t-SNE.

<div style="width:100%; margin:auto; margin-bottom:10px; margin-top:20px;">
<img style="width:100%" src="img/tsne.png" alt>
</div>

Et voila! As expected, words that are similar end up clustering nearby each
other. For a more heavyweight implementation of word2vec that showcases more of
the advanced features of TensorFlow, see the implementation in
[tensorflow/models/embedding/word2vec.py](https://tensorflow.googlesource.com/tensorflow/+/master/tensorflow/models/embedding/word2vec.py).

## Evaluating Embeddings: Analogical Reasoning <a class="md-anchor" id="AUTOGENERATED-evaluating-embeddings--analogical-reasoning"></a>

Embeddings are useful for a wide variety of prediction tasks in NLP. Short of
training a full-blown part-of-speech model or named-entity model, one simple way
to evaluate embeddings is to directly use them to predict syntactic and semantic
relationships like `king is to queen as father is to ?`. This is called
*analogical reasoning* and the task was introduced by
[Mikolov and colleagues](http://msr-waypoint.com/en-us/um/people/gzweig/Pubs/NAACL2013Regularities.pdf),
and the dataset can be downloaded from here:
https://word2vec.googlecode.com/svn/trunk/questions-words.txt.

To see how we do this evaluation, have a look at the `build_eval_graph()` and
`eval()` functions in
[tensorflow/models/embedding/word2vec.py](https://tensorflow.googlesource.com/tensorflow/+/master/tensorflow/models/embedding/word2vec.py).

The choice of hyperparameters can strongly influence the accuracy on this task.
To achieve state-of-the-art performance on this task requires training over a
very large dataset, carefully tuning the hyperparameters and making use of
tricks like subsampling the data, which is out of the scope of this tutorial.


## Optimizing the Implementation <a class="md-anchor" id="AUTOGENERATED-optimizing-the-implementation"></a>

Our vanilla implementation showcases the flexibility of TensorFlow. For
example, changing the training objective is as simple as swapping out the call
to `tf.nn.nce_loss()` for an off-the-shelf alternative such as
`tf.nn.sampled_softmax_loss()`. If you have a new idea for a loss function, you
can manually write an expression for the new objective in TensorFlow and let
the optimizer compute its derivatives. This flexibility is invaluable in the
exploratory phase of machine learning model development, where we are trying
out several different ideas and iterating quickly.

Once you have a model structure you're satisfied with, it may be worth
optimizing your implementation to run more efficiently (and cover more data in
less time).  For example, the naive code we used in this tutorial would suffer
compromised speed because we use Python for reading and feeding data items --
each of which require very little work on the TensorFlow back-end.  If you find
your model is seriously bottlenecked on input data, you may want to implement a
custom data reader for your problem, as described in
[New Data Formats](../../how_tos/new_data_formats/index.md).  For the case of Skip-Gram
modeling, we've actually already done this for you as an example in
[tensorflow/models/embedding/word2vec.py](https://tensorflow.googlesource.com/tensorflow/+/master/tensorflow/models/embedding/word2vec.py).

If your model is no longer I/O bound but you want still more performance, you
can take things further by writing your own TensorFlow Ops, as described in
[Adding a New Op](../../how_tos/adding_an_op/index.md).  Again we've provided an
example of this for the Skip-Gram case
[tensorflow/models/embedding/word2vec_optimized.py](https://tensorflow.googlesource.com/tensorflow/+/master/tensorflow/models/embedding/word2vec_optimized.py).
Feel free to benchmark these against each other to measure performance
improvements at each stage.

## Conclusion <a class="md-anchor" id="AUTOGENERATED-conclusion"></a>

In this tutorial we covered the word2vec model, a computationally efficient
model for learning word embeddings. We motivated why embeddings are useful,
discussed efficient training techniques and showed how to implement all of this
in TensorFlow. Overall, we hope that this has show-cased how TensorFlow affords
you the flexibility you need for early experimentation, and the control you
later need for bespoke optimized implementation.
