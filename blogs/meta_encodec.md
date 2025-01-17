# 高保真神经网络音频编码器
>本文介绍了meta推出的音频AI Codec，其整体风格深受Google的SoundStream的影响。在其影响下改进了原有的肩背起，引入语言模型进一步降低码率，并提出了一种提升稳定性的训练策略。

论文题目：High Fidelity Neural Audio Compression 
作者：Alexandre Défossez, Jade Copet, Gabriel Symmaeve, Yossi Adi (Meta AI, FAIR Team)
![](https://pic4.zhimg.com/80/v2-6a02f19978157c97d0df84e10981bfb3_1440w.webp)

## 背景动机
* 与之前的AI Codec的动机相同，本文同样希望借助深度学习设计一款端到端多码率、立体声音频编码器，实现对语音和音乐的低码率压缩并高质量还原。
* 神经网络天然的抽象特征提取能力使其具有相比传统编码器更强的信号表征压缩能力，低码率的问题相对并不困难；
* 难点主要有两点：1. 音频的动态范围过大；2. 模型效率问题(计算复杂度和参数量)

## 本文贡献：
* 为解决音频动态范围过大的问题，使用庞大多样的训练集以及用鉴别器作为感知损失(这点似乎相比SoundStream)也并未见有什么突破；
* 限制在单核CPU上实时运行，并采用残差矢量量化(Residual Vector Quantization, RVQ)提高编码效率；
* 提出了语言模型进一步降低码率；
* 鉴别器采用多分辨率复数谱STFT鉴别器；
* 提出了一种balancer以保证GAN训练的稳定性

## 模型架构

![](https://pic4.zhimg.com/80/v2-664dcd30241529cebc9941621344986b_1440w.webp)

模型采用的基于GAN的模型，生成器采用时域编码器-量化器-解码器结构，鉴别器采用多分辨率的STFT鉴别器。

**编解码器**：编解码器采用SEANET，编码器由一层一维卷积对时域波形进行特征提取后经过B个用于降采样的残差单元(即convolurion blocks)，而后加入了两层LSTM用于序列建模，最后经过一层卷积得到音频的潜在表征。解码器则是编码器的镜像，其中残差单元的卷积被替换为反卷积用于上采样。根据文中采用的下采样因子(通过卷积的stride实现){2,4,5,8}，其编码器将音频下采样320倍(2x4x5x8=320)，即传输的一帧中压缩了320个采样点，因此在采样率为24kHz时1s的音频经编码器输出的时间维数为24000/320=75，48kHz时为48000/320=150。通过卷积的Padding和调整Nomalization去设置模型是否流式。

**量化器**：量化器采用残差矢量量化RVQ，关于RVQ的详细介绍参看 和 。每个码书包含1024个向量(entries)，对于采样率为24kHz的音频，最多使用32个码书，即最大码率为32x $\log_2(1024)$ /13.3=24kbps。为了支持多码率，训练过程中码书数量被设置为{2,4,8,16,32}，分别对应1.5kbps,3kbps,6kbps,12kbps,24kbps；且每个码率在训练时所使用的鉴别器是不同的。

**语言模型和熵编码**：此部分可选，使用Transormer语音模型对RVQ得到的索引映射到新隐藏空间的概率分布，对对应概率密度函数的累积分布函数进行Range Coder熵编码，从而进一步降低码率。

**鉴别器**：采用多分辨率复数STFT鉴别器，而非TTS中常见的多分辨率Mel谱鉴别器，也没加Multiple-period 鉴别器MPD(消融实验显示多分辨率复数STFT鉴别器性能更优，额外引入MPD有少量性能提升，但考虑训练时长舍弃)。每个分辨率的鉴别器有二维卷积组成，结构如图所示(注意：其中正文和图中的卷积核尺寸不一致，3x8 v.s. 3x9)。

![](https://pic4.zhimg.com/80/v2-25c2ddc01c52f98f76252a10ce791647_1440w.webp)

鉴别器采用hinge loss训练，为保证生成器和鉴别器训练平衡稳定，鉴别器以2/3的概率更新其参数

<img width="584" alt="" src="https://user-images.githubusercontent.com/36671362/198877068-c084ded9-3dbf-4be2-9983-cfb73623da56.png">

## 生成器的损失函数

包括重构损失、感知损失(实为对抗损失)以及RVQ的commitment loss三部分：

<img width="790" alt="" src="https://user-images.githubusercontent.com/36671362/198877006-55912e7b-9f31-4cf5-8928-198e9d5af0af.png">

**重构损失**包括时域和频域两部分，时域损失是波形的L1损失

<img width="203" alt="企业微信20221030-195824@2x" src="https://user-images.githubusercontent.com/36671362/198877201-57088460-9936-4c58-92dc-8501e35ae6d8.png">

频域损失是多时间尺度的Mel谱损失

<img width="696" alt="企业微信截图_6e363aab-30f3-4202-a8dd-5badb33f33bd" src="https://user-images.githubusercontent.com/36671362/198877207-d5ff7db0-89e8-4c27-88d8-6532138e53d3.png">

**对抗损失**采用hinge loss和特征匹配损失，分别为：

<img width="337" alt="" src="https://user-images.githubusercontent.com/36671362/198877051-41868da9-26e3-4977-859b-a8db785d4873.png">

<img width="476" alt="" src="https://user-images.githubusercontent.com/36671362/198877026-9892aeb6-fe39-4f4a-9172-61bc9afb7630.png">

**commitment loss**用于使VQ选择的向量满足量化后的变量与未量化的变量间最相近，采用欧式距离度量则为：

<img width="1005" alt="" src="https://user-images.githubusercontent.com/36671362/198876961-9fc7fce4-0523-452a-89ec-22a389ac12ec.png">

除了commitment loss之外，其他生成器loss均采用本文提出的平衡器以稳定训练，即在梯度更新时则反向传播 $\sum_{i}{\tilde{g}\_{i}}$ ,其中 $g_i$ 由第 $i$ 个损失函数对应的梯度 $g_i=\frac{\partial{l_i}}{\partial{x}}$ 对其指数平均项 $\langle ||g_i||\rangle_{\beta}$归一化后重加权得到，定义为

<img width="250" alt="" src="https://user-images.githubusercontent.com/36671362/198877082-b390c106-a71d-42a3-a6d7-31d559d4d6b8.png">


**训练参数**：300 epochs, with one epoch being 2,000 updates with the Adam optimizer with a batch size of 64 examples of 1 second each, a learning rate of 3 · 10−4 , $\beta_1$ = 0.5, and $\beta_2$ = 0.9. All the models are traind using 8 A100 GPUs. We use the balancer introduced in Section 3.4 with weights $\lambda_t$ = 0.1, $\lambda_f$ = 1, $\lambda_g$ = 3, $\lambda_{feat}$ = 3 for the 24 kHz models. 

## 数据与结果

数据源：speech:DNS Challenge 4 and the Common Voice dataset; general audio: AudioSet and FSD50K; music: Jamendo dataset

数据增广策略：多数据源混合；加混响；音量标准化并随机化增益-10～6 dB；无clip

结果：

![](https://pic2.zhimg.com/80/v2-a3998a9f1a04e2d579fc4920aef00fd1_1440w.webp)

![](https://pic2.zhimg.com/80/v2-e8d0d50ec0f44b00cba1afe1389e46c5_1440w.webp)


* 通道数对性能影响较小但显著影响实时率
* 残差模块和LSTM都显著影响实时率


![](https://pic4.zhimg.com/80/v2-cdf46c8c78206fa01ea153b5976dddeb_1440w.webp)
