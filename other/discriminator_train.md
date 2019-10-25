[生成对抗网络GAN如果只训练一个网络会有效果么？](https://www.zhihu.com/question/276070438/answer/825816888)

作者：Chuang

链接：https://www.zhihu.com/question/276070438/answer/825816888

来源：知乎

著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

这其实是一个很有趣的问题。在实践过程中，如果把判别器（Discriminator）训练得太好了，看似能够在对抗中更加有效的拒绝生成器（Generator）生成的假样本，但是其实一样会产生诸多问题。

判别器最主要的作用就是为生成器提供下降梯度。如果判别器太差，则无法提供有效的梯度，同时**判别器越好，生成器梯度消失越严重。**

回顾一下，原始GAN中判别器要最小化如下损失函数，尽可能把真实样本分为正例，生成样本分为负例：

![公式][1] （公式1）

其中![[公式]](https://www.zhihu.com/equation?tex=P_r)是真实样本分布，![[公式]](https://www.zhihu.com/equation?tex=P_g)是由生成器产生的样本分布。

假设我们如果要学习得到一个最优网络，必然上式求导等于0，则会得到：

![[公式]][2](公式2)

从公式上就不难看出，此时最优的判别器，就是判断为真实图像概率对于判断为真和假的概率和的比值。当![[公式]][3]且![[公式]][4]，最优判别器就应该非常自信地给出概率0；如果![[公式]][5]说明该样本是真是假的可能性刚好一半一半，此时最优判别器也应该给出概率0.5。

在极端情况——判别器最优时，生成器的损失函数变成什么。给损失函数加上一个不依赖于生成器的项，使之变成

![[公式]][6]（公式3）

注意，它刚好是判别器损失函数的反。代入最优判别器即公式2，再进行简单的变换可以得到

![[公式]][7](公式4)

变换成这个样子是为了引入Kullback–Leibler divergence（简称KL散度）和Jensen-Shannon divergence（简称JS散度）这两个重要的相似度衡量指标。

KL散度和JS散度：

![[公式]][8](公式5：KL散度)

![[公式]][9](公式6：JS散度)

于是公式4就可以继续写成

![[公式]][10](公式7)

**根据原始GAN定义的判别器loss，我们可以得到最优判别器的形式；而在最优判别器的下，我们可以把原始GAN定义的生成器loss等价变换为最小化真实分布![[公式]](https://www.zhihu.com/equation?tex=P_r)与生成分布![[公式]](https://www.zhihu.com/equation?tex=P_g)之间的JS散度。我们越训练判别器，它就越接近最优，最小化生成器的loss也就会越近似于最小化![[公式]](https://www.zhihu.com/equation?tex=P_r)和![[公式]](https://www.zhihu.com/equation?tex=P_g)之间的JS散度。**

问题就出在这个JS散度上。我们会希望如果两个分布之间越接近它们的JS散度越小，我们通过优化JS散度就能将![[公式]](https://www.zhihu.com/equation?tex=P_g)“拉向”![[公式]](https://www.zhihu.com/equation?tex=P_r)，最终以假乱真。这个希望在两个分布有所重叠的时候是成立的，但是如果两个分布完全没有重叠的部分，或者它们重叠的部分可忽略，它们的JS散度是多少呢？

答案是![[公式]](https://www.zhihu.com/equation?tex=\log+2)，因为对于任意一个x只有四种可能：

![[公式]][11]且![[公式]][12]

![[公式]][13]且![[公式]][14]

![[公式]](https://www.zhihu.com/equation?tex=P_1%28x%29+%3D+0)且![[公式]](https://www.zhihu.com/equation?tex=P_2%28x%29+%5Cneq+0)

![[公式]](https://www.zhihu.com/equation?tex=P_1%28x%29+%5Cneq+0)且![[公式]](https://www.zhihu.com/equation?tex=P_2%28x%29+%3D+0)

第一种对计算JS散度无贡献，第二种情况由于重叠部分可忽略所以贡献也为0，第三种情况对公式7右边第一个项的贡献是![[公式]][16]，第四种情况与之类似，所以最终：![[公式]][15]。

换句话说，无论![[公式]](https://www.zhihu.com/equation?tex=P_r)跟![[公式]](https://www.zhihu.com/equation?tex=P_g+)是远在天边，还是近在眼前，只要它们俩没有一点重叠或者重叠部分可忽略，JS散度就固定是常数![[公式]](https://www.zhihu.com/equation?tex=\log+2)，**而这对于梯度下降方法意味着——梯度为0**！此时对于最优判别器来说，生成器肯定是得不到一丁点梯度信息的；即使对于接近最优的判别器来说，生成器也有很大机会面临梯度消失的问题。

但是![[公式]](https://www.zhihu.com/equation?tex=P_r)与![[公式]](https://www.zhihu.com/equation?tex=P_g)不重叠或重叠部分可忽略的可能性有多大？不严谨的答案是：非常大。比较严谨的答案是：**当![[公式]](https://www.zhihu.com/equation?tex=P_r)与![[公式]](https://www.zhihu.com/equation?tex=P_g)的支撑集（support）是高维空间中的低维流形（manifold）时，![[公式]](https://www.zhihu.com/equation?tex=P_r)与![[公式]](https://www.zhihu.com/equation?tex=P_g)重叠部分测度（measure）为0的概率为1。**

**所以其实在实践过程中，用一个很优的判别器去训练好一个网络的概率其实很小。**

**不然还有什么对抗的意义呢，对吧。**

ヾ(≧∇≦*)ゝ
[1]: https://www.zhihu.com/equation?tex=-\mathbb{E}_{x\sim+P_r}\log+D(x)+-+\mathbb{E}_{x\sim+P_g}\log(1-D(x))
[2]:https://www.zhihu.com/equation?tex=D^*(x)+%3D+\frac{P_r(x)}{P_r(x)+%2B+P_g(x)}
[3]:https://www.zhihu.com/equation?tex=P_r(x)+%3D+0
[4]:https://www.zhihu.com/equation?tex=P_g(x)+\neq+0
[5]:https://www.zhihu.com/equation?tex=P_r(x)+%3D+P_g(x)
[6]:https://www.zhihu.com/equation?tex=\mathbb{E}_{x\sim+P_r}\log+D(x)+%2B+\mathbb{E}_{x\sim+P_g}\log(1-D(x)
[7]: https://www.zhihu.com/equation?tex=\mathbb{E}_{x+\sim+P_r}+\log+\frac{P_r(x)}{\frac{1}{2}P_r(x)+%2B+P_g(x)}+%2B+\mathbb{E}_{x+\sim+P_g}+\log+\frac{P_g(x)}{\frac{1}{2}P_r(x)+%2B+P_g(x)}+-+2\log+2
[8]:https://www.zhihu.com/equation?tex=KL(P_1||P_2)+%3D+\mathbb{E}_{x+\sim+P_1}+\log+\frac{P_1}{P_2}
[9]:https://www.zhihu.com/equation?tex=JS(P_1+||+P_2)+%3D+\frac{1}{2}KL(P_1||\frac{P_1+%2B+P_2}{2})+%2B+\frac{1}{2}KL(P_2||\frac{P_1+%2B+P_2}{2})
[10]:https://www.zhihu.com/equation?tex=2JS(P_r+||+P_g)+-+2\log+2
[11]:https://www.zhihu.com/equation?tex=P_1(x)+%3D+0
[12]:https://www.zhihu.com/equation?tex=P_2(x)+%3D+0
[13]:https://www.zhihu.com/equation?tex=P_1(x)+\neq+0
[14]:https://www.zhihu.com/equation?tex=P_2(x)+\neq+0
[15]:https://www.zhihu.com/equation?tex=JS(P_1||P_2)+%3D+\log+2
[16]:https://www.zhihu.com/equation?tex=\log+\frac{P_2}{\frac{1}{2}(P_2+%2B+0)}+%3D+\log+2







> 参考：

郑华滨：令人拍案叫绝的Wasserstein GANzhuanlan.zhihu.com

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g74ozup1ynj305003ct8t.jpg)

