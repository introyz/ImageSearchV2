# ImageSearchV2
使用深度学习网络（目标检测/特征提取/特征匹配）建立的图像精准检索系统(CBIR)

## 整体说明

1. 系统的实现背景基于：阿里天池的淘宝直播商品识别竞赛，

[直达链接]: https://tianchi.aliyun.com/competition/entrance/231772/tab/185	"点击跳转"

；项目中的演示数据也是出于这里。可以将该竞赛理解为：输入直播视频，在商品库中进行检索，然后输出最匹配的商品。

2. 该项目综合性程度很高的图像检索应用,包括:图像ROI区域检测与提取（采用了SSD）、ROI区域的分类（VGG-16与部分拼接网络）、ROI区域局部特征（LBP）/全局特征的提取（VGG-16与部分拼接网络，使用Triplet进行微调）和基于提取特征的检索算法（暴力匹配）的设计与实现。。

3. 系统检索效果上，在比赛官方的验证集上取得了Top20的成绩。

   在自己的本地数据集上，随机抽取的100件商品集合中，目前的精确匹配率可以达到35%。

4. 系统的不足：

   - 采用训练的数据集不是很多。官方提供的数据集大概180G，包含3W件商品（包括一个直播视频和5-7张商品展示图）；初期尝试本地算力不够的情况下，进行了“类均衡采样”，抽取了大概2W张图片（1W的视频帧和1W的商品展示图）；因此训练数据不是很多，导致特征提取网络的效果不是很好。

   - 没有实现很好的基于提取特征的检索算法：

     初版的检索算法是与商品图像特征库逐特征比较，选取欧氏距离最短的商品图片特征表示，该特征指代的商品图像即为匹配图像；

     后续改进引入了投票模型的思想：每个视频抽取10帧，每一帧中检测到的目标（可能多个）与商品图像特征库进行逐特征比较，选取欧氏距离最短的商品图片特征表示，随后按照该特征表示对应的商品进行投票，选择票数最多的商品图像为最终匹配图像。

5. 系统的下一步改进策略：

   - 可以选取更复杂的，表达能力更强的深度神经网络，如：Inception-V4；

   - 增加训练的数据量，包括：增加输入网络的数据量，以及使用更多的数据增强方法；

   - 增加训练数据量的之前，可以现在‘类均衡采样’的网络上进行初期训练，随后放到全量数据集上进行训练。

   - 改进检索算法，可以参考Match-RCNN的思想引入感知网络和匹配网络；也可以参考HardNet的思想；进行多方面尝试后，可以尝试进行模型融合。

   - 比赛官方也提供了“直播视频的语音文本”，可以进一步的引入文本特征，进行跨模态的特征提取和特征匹配。



## 系统效果演示

匹配正确的结果演示：

gif 图片过大，请见 pictures/匹配正确的.gif

匹配错误的结果演示：

gif 图片过大，请见 pictures/匹配错误的.gif



##  系统Tricks说明

1. 总体分析训练数据时,发现有很强的类别不均衡现象,因此进行了“类均衡采样”,后续训练使用类别均
   衡的样本,便于初期模型的快速收敛。
2. 对各个训练模型增加了“数据增强”的预处理结构,如:随机亮度/对比度调整。随机翻转/旋转/裁剪等。
3. 为了减少训练时间，模型训练时均采用了已训练好的模型,并固定部分层(主要是模型前段部分的参数)，训练模型靠近输出的层。
4. 图像ROI区域检测时为了大幅降低了训练时间，使用了预训练的SSD(PyTorch版),将衣物标签全部处理成预训练模型的“person”标签，随后固定SSD分类部分网络层,并在部分卷积层前增加了“BatchNorm层”，随后FineTune SSD的区域预测部分,大概10个epoch后,区域预测IOU=0.5时,单类别预测AP可达到93.7%。
5. 搭建分类网络时：抽取检测模型的VGG-16结构与参数，并在后面拼接上了部分Inception-V4结构(3*Inception-C，Avarage Pooling， FC，为了减少参数量，同时又增强分类能力)，固定VGG-16的参数，
   训练后续分类网络的结构。
6. 训练分类网络大概20个epoch后，分类的mAP超过50%后就停止训练了；由于分类是为了更好的图像检索，对具体类别是否正确要求不高，因此针对分类网络的结果进行“类别合并”，进一步提升查全率。
7. 特征提取模型的搭建：复制分类模型结构及其参数，并去除最后一个FC层；随后采用多输入网络的思想(Triplet)，FineTune特征提取模型；目的是为了让同一个商品图片提取的特征基本相似,不同商品图片提取的特征差异较大。
8. 检索算法的设计与实现：经数据分析，90%的商品是单件商品，也就是单张图片的ROI区域唯一，因此特征检索具体可分为三部分进行：首先是“去重/去遮挡”，提取得分最高的商品特征表示；接着是比较粗暴的特征向量相似度度量(欧氏距离，由于Triplet损失函数采用的也是欧氏距离)，这里需要引入感知网络和匹配网路；最后是组合商品的搜索。



## 系统各部分模型训练说明

1. 目标检测部分：

   由于SSD的预训练网络是在PACAL_VOC_2012上得出的，而它的网络结构设计上，从先验框的获取到边界框的预测和分类（Neck部分/Dense Prediction），都与类别数目是强相关的。因此将SSD的Neck部分复用到新的数据集上，会比较困难，而重新训练Neck部分必然会很耗费时间和算力，即训练时间可能得有个3-7天（单卡：GTX 1060）。

   接着在观察了自己的数据集（衣物）和PACAL_VOC_2012数据集后，发现衣物的主体特征跟PACAL_VOC_2012数据集中“Person”类别的主体特征特别相似；即便是使用未经FineTune的SSD在自己的数据集进行测试，单‘Person’类别，IOU=0.5时的AP都可以达到30%左右。

   因此，综上述，为了充分利用预训练的网络，在SSD网络结构上固定VVG-16部分（Backbone）的参数，只FineTune后续的的边界框预测网络和分类网络。在处理自己的数据集时，将衣物的标签都转化成了‘Person’类型，强化SSD对该类型的预测。

   最后结果是显著的：训练数据集是‘类均衡采样’后的2W张图片，机器配置是单卡：GTX 1060；在采用冲量和学习率衰减的SGD优化方法训练12H后，LOSS就下降到了1以下；在随机采样的2K张验证集上，最好的单类别AP=93.7%，此时IOU=0.5。

   

2. 分类网络部分：

   此时的分类网络要处理的数据，可以理解成是目标检测后的目标区域。

   官方数据集中的类别有23类，但实际可能不需要区分的这么详细，如：长裤、中裤、短裤完全可以放到一个类别里。

   增加分类网络的主要原因是为了对组合商品进行判定，经数据分析商品组合通常是固定的某几个类别的组合，如：T恤与短裤的组合。因此一张图片的多个目标区里，出现了上述组合，则该商品是组合商品的概率就很高。

   通常的特征提取网络可以当成从某个预训练网络的中间部分截取特征，如：去掉分类网络最后的FC层，将FC层之前的输出当作固定维数的特征向量提取出来，表示当前图片。

   

   因此综上述，对分类网络有下面三个要求：

   分类网络的设计要有很好的表达能力，VGG-16部分可能是不够的，需要增加网络的深度；

   再增加网络深度的同时，也要保证参数量不能激烈爆增，适当的控制训练成本；

   同时分类网络训练好后，可以用于后续的特征提取网路。

   故最后，分类网络前段部分采用的是预训练的VGG-16（带BN层），后续拼接的是InceptionV-4的部分结构（3*Inception-C, Avarage Pooling, FC），两者之间衔接层采用的是：1乘1卷积核（用于降纬），MaxPool和Relu。

   

   分类的训练，在mAP>50%时就停止了（大概耗时10H），随后让分类网络在全部的验证集上跑了一遍，根据分类结果，进行了一定的类别合并；类别合并后再次预测，mAP可达到89%左右。

   

3. 特征提取网络部分：

   此时的特征提取网络要处理的数据，可以理解成是目标检测后的目标区域。

   复制了分类网络，并去除了最后的FC层，采用了Triplet网络的思想进行训练，目的是为了让同一个商品图片提取的特征基本相似,不同商品图片提取的特征差异较大。

   Triplet网络中，anchor选择的直播视频的某一帧，正样本选择的跟该直播视频匹配的商品的某一张图片，负样本选择的是跟该直播视频中的商品处于同一分类，但不是同一商品的直播视频的某一帧。

   

4. 检索网络部分：

   对这部分的了解比较少，后续准备引入：Match-RCNN或者是HardNet
