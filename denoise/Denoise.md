[TOC]



# Rnnoise

## 频带划分

![image-20210423114648664](/Users/wxk/Library/Application Support/typora-user-images/image-20210423114648664.png)

采样率为F时，可观察频率范围 Hz : 0~F/2  若取采样点 S---FFT--->S/2 个bin  用S/2个bin均匀的代指 0~F/2 的频率

例如：采样率48000 Hz:0~24000 每次取960个点(20ms)则 FFT后能得到480个bin  每个bin代表24000/480=50个频率



Opus  ： 0    200  400  600  800   1k  1.2 1.4 1.6 2k 2.4 2.8 3.2 4k 4.8  5.6  6.8  8k  9.6  12k  15.6    20k

Bin    ：  0      4      8     12    16     20  ……………………………………………………………………    400

Ebands : 0     1       2      3      4       5   ……………………………………………………………………   100

**即我们找到了经过FFT变换后480长度的 bin 与 eband 的对应关系**

## 数据处理

分别从纯人声和纯噪声取出一帧的样本点：x(480) 、n(480) 都经过下列处理

1.经过随机的语音增强

2.通过一个二阶滤波器 
<img src="/Users/wxk/Library/Application Support/typora-user-images/image-20210423115306008.png" alt="image-20210423115306008" style="zoom:50%;" />

![image-20210423120417622](/Users/wxk/Library/Application Support/typora-user-images/image-20210423120417622.png)

3.计算频带能量：x(480)  --拼接前一帧数据--> x(960) --加窗--> x(960)  --FFT--> X(481) --计算能量（三角滤波）--> Ex(22)

![image-20210423132347871](/Users/wxk/Library/Application Support/typora-user-images/image-20210423132347871.png)

## 加噪音频提取特征

xn(480) = x(480)+n(480)

xn(480)  --拼接前一帧数据--> xn(960) --加窗--> xn(960)  --FFT--> XN(481) --计算能量--> Exn(22)

计算log(Exn)的DCT （信号的BFCC） （**22 feature**）

与前两帧计算出的22个feature 计算**6**个一阶差分和**6**个二阶差分 （**12 feature**）

![image-20210423135036462](/Users/wxk/Library/Application Support/typora-user-images/image-20210423135036462.png)

![image-20210423135509390](/Users/wxk/Library/Application Support/typora-user-images/image-20210423135509390.png)

对于第i行特征，计算出与其他7行的最小L2距离,mindist[i]  ，平稳度量特征为 spec_variability  (**1 feature**)
$$
mindist[i] = min_{j\neq i}(MSE(M[i],M[j])) \\
spec\_variability = \sum^7_{i=0} {mindist[i]}
$$
利用opus的函数找 pitch  （**1 feature**）

取出pitch部分       p(960)  --加窗-->  p(960)  --FFT-->  P(481) --计算能量--> Ep(22)

用Exn和Ep 计算corr能量   Exp(22)

计算Exp的DCT （即计算了加噪音频xn与pitch部分的BFCC）取前6个 （**6 feature**）

**总共 42 feature**

## 计算 target : 频带增益 , Vad

Vad : webrtc vad    （如果Ex 或者 Exn 很小，则Vad = 0  ,否则 Vad = 1）

Gains:
$$
Gain(i) = min(max(0,{\sqrt{\frac {Ex[i]}{Exn[i]}}}),1)
$$
**至此，一帧数据得到  input: 42维feature  target:（22维Gains + 1维 Vad**）

## 网络模型



![image-20210423142339528](/Users/wxk/Library/Application Support/typora-user-images/image-20210423142339528.png)

## 对带噪音频降噪

输入xn(480)  --拼接前一帧数据--> xn(960) --加窗--> xn(960)  --FFT--> XN(481) --计算能量--> Exn(22)

计算出pitch部分 的 P 和Ep  ； 以及两者的corr能量 Exp

计算出42维feature  输入网络得到Vad 和 Gains 

由于我们采用22频带，频带分辨率过于粗糙，对于音调谐波（ pitch harmonics）的噪声很难消除， 所以利用Gains和梳状滤波器对Exn进行处理

逆向的将22维的Exn能量分配回长度481的bin数组

利用IFFT变换回降噪后的音频样本点

# Conv-TasNet

## TCN 结构 （time-dilated convolutional network）

![image-20210423150223705](/Users/wxk/Library/Application Support/typora-user-images/image-20210423150223705.png)

## 模型结构

![image-20210423150315060](/Users/wxk/Library/Application Support/typora-user-images/image-20210423150315060.png)

![image-20210423151227621](/Users/wxk/Library/Application Support/typora-user-images/image-20210423151227621.png)

# TDCN++

TDCN++ 相对 TDCN 的改进：

1.将GLN层 input:B\*F\*L  mean : B\*1\*1

替换为FLN input:B\*F\*L mean : B\*F\*1

2.在每个conv-block前后加入一个线性层,并接入一个更长的 residual-connection

3.每个dense层后添加一个可学习的缩放参数

![image-20210423151400338](/Users/wxk/Library/Application Support/typora-user-images/image-20210423151400338.png)

![image-20210423151625031](/Users/wxk/Library/Application Support/typora-user-images/image-20210423151625031.png)

# 更多语音增强相关论文

## 最近被关注的模型

### [DCRN (2020)](https://arxiv.org/pdf/2011.03840.pdf)

![image-20210425145853589](/Users/wxk/Library/Application Support/typora-user-images/image-20210425145853589.png)

Sub-pixel Convolution ![image-20210425145925167](/Users/wxk/Library/Application Support/typora-user-images/image-20210425145925167.png)



![image-20210425154249666](/Users/wxk/Library/Application Support/typora-user-images/image-20210425154249666.png)

![image-20210425151038144](/Users/wxk/Library/Application Support/typora-user-images/image-20210425151038144.png)

![image-20210425175723443](/Users/wxk/Library/Application Support/typora-user-images/image-20210425175723443.png)

![image-20210425154442690](/Users/wxk/Library/Application Support/typora-user-images/image-20210425154442690.png)

![image-20210425160737466](/Users/wxk/Library/Application Support/typora-user-images/image-20210425160737466.png)



### 其他

[Unet（2019）](https://ieeexplore.ieee.org/document/8969126?denied=)、[CRN（2018）](https://www.semanticscholar.org/paper/A-Convolutional-Recurrent-Neural-Network-for-Speech-Tan-Wang/d24d6db5beeab2b638dc0658e1510f633086b601?p2df)、[TCRN（2020）](https://arxiv.org/pdf/2002.00319v1.pdf)

## Gan网络

把生成器当做增强网络，用判别器区分干净语音和增强语音。

1.[SEGAN: Speech Enhancement Generative Adversarial Network](https://link.zhihu.com/?target=http%3A//pdfs.semanticscholar.org/307a/cb91ebc6333f044359aff9284bbe0d48e358.pdf) 2017

2.[Speech Enhancement Based on A New Architecture of Wasserstein Generative Adversarial Networks](https://link.zhihu.com/?target=https%3A//sci-hub.tw/10.1109/iscslp.2018.8706647) 2018

3.[Conditional Generative Adversarial Networks for Speech Enhancement and Noise-Robust Speaker Verification](https://link.zhihu.com/?target=https%3A//www.isca-speech.org/archive/Interspeech_2017/pdfs/1620.PDF) 2017

4.[Language and Noise Transfer in Speech Enhancement Generative Adversarial Network](https://link.zhihu.com/?target=https%3A//arxiv.org/abs/1712.06340%3Fcontext%3Dcs) 2017

5.[Exploring speech enhancement with generative adversarial networks for robust speech recognition](https://link.zhihu.com/?target=https%3A//arxiv.org/pdf/1711.05747.pdf) 2018

6.[Time-Frequency Masking-based Speech Enhancement using Generative Adversarial Network](https://link.zhihu.com/?target=https%3A//www.researchgate.net/publication/324888591_Time-Frequency_Masking-based_Speech_Enhancement_using_Generative_Adversarial_Network) 2018

7.[Adversarial Feature-Mapping for Speech Enhancement](https://link.zhihu.com/?target=https%3A//arxiv.org/pdf/1809.02251.pdf) 2019

8.[Sergan: Speech Enhancement Using Relativistic Generative Adversarial Networks with Gradient Penalty](https://link.zhihu.com/?target=https%3A//sci-hub.tw/10.1109/icassp.2019.8683799) 2019

9.[CP-GAN:Context Pyramid Generative Adversarial Network for Speech Enhancement 2020](https://link.zhihu.com/?target=https%3A//sci-hub.tw/10.1109/icassp40776.2020.9054060)

10.[Tdcgan: Temporal Dilated Convolutional Generative Adversarial Network for End-to-end Speech Enhancement](https://link.zhihu.com/?target=https%3A//arxiv.org/abs/2008.07787) 2020

## 卷积神经网络方面：全卷积、冗余卷积，时域、频域

1.[Single channel speech enhancement using convolutional neural network](https://link.zhihu.com/?target=http%3A//xueshu.baidu.com/s%3Fwd%3Dpaperuri%3A%288e20ebef36821bc701e5d55119d7aa73%29%26filter%3Dsc_long_sign%26tn%3DSE_xueshusource_2kduw22v%26sc_vurl%3Dhttp%3A%2F%2Fieeexplore.ieee.org%2Fdocument%2F7945915%2F%26ie%3Dutf-8%26sc_us%3D10353052094947466062) 2017

2.[A Fully Convolution Neural Network for Speech Enhancement](https://link.zhihu.com/?target=https%3A//www.isca-speech.org/archive/Interspeech_2017/pdfs/1465.PDF) 2017

3.[Raw Waveform-based Speech Enhancement by Fully Convolutional Networks](https://link.zhihu.com/?target=http%3A//pdfs.semanticscholar.org/9c05/e21d07734a986063bffe0a0b271a08eb30b6.pdf) 2017

4.[Speech Denoising with Deep Feature Losses](https://link.zhihu.com/?target=https%3A//arxiv.org/abs/1806.10522) 2018

5.[A New Framework for Supervised Speech Enhancement in the Time Domain](https://link.zhihu.com/?target=https%3A//web.cse.ohio-state.edu/~wang.77/papers/Pandey-Wang.interspeech18.pdf) 2018

6.[A Wavenet for Speech Denoising](https://link.zhihu.com/?target=https%3A//arxiv.org/pdf/1706.07162.pdf) 2018

7.[Fully Convolution Recurrnet Network for Speech Enhancement](https://ieeexplore.ieee.org/stamp/stamp.jsp?arnumber=9054230&casa_token=jn4YN4EWNT0AAAAA:ELKc_wnZ8ws50db0dS-ODujVDOr13dJYQMvs3jCZ-An5_zqBp-MVjHsLYvmTbTERFhnvOEimi2u9&tag=1) 2020

## DNN

主要是在频域内处理语音，通过短时傅里叶变换求得短时频谱，然后对短时频谱进行处理，利用含噪语音的相位进行重构增强语音。还有一些是DNN和传统语音增强方法进行结合的办法，把传统语音中的features换成DNN网络。

1.[Speech Enhancement In Multiple-Noise Conditions using Deep Neural Networks](https://link.zhihu.com/?target=https%3A//www.isca-speech.org/archive/Interspeech_2016/pdfs/0088.PDF) 2016

2.[NMF-based Speech Enhancement Incorporating Deep Neural Network](https://link.zhihu.com/?target=http%3A//pdfs.semanticscholar.org/8588/b415251e6f17f92a726171d957171fafcb81.pdf) 2014

3.[A Novel Single Channel Speech Enhancement Based on Joint Deep Neural Network and Wiener Filter](https://link.zhihu.com/?target=http%3A//xueshu.baidu.com/s%3Fwd%3Dpaperuri%3A%28de2ed9a93abb3edeacde73c2a321e400%29%26filter%3Dsc_long_sign%26tn%3DSE_xueshusource_2kduw22v%26sc_vurl%3Dhttp%3A%2F%2Fieeexplore.ieee.org%2Fdocument%2F7489830%2F%26ie%3Dutf-8%26sc_us%3D17707163339176136223) 2015

4.[An Experimental Study on Speech Enhancement Based on Deep Neural Networks](https://link.zhihu.com/?target=https%3A//ieeexplore.ieee.org/document/6665000/) 2014

5.[A Regression Approach to Speech Enhancement Based on Deep Neural Networks](https://ieeexplore.ieee.org/stamp/stamp.jsp?arnumber=6932438&casa_token=kZCN5q75DpQAAAAA:EBJGXuZ-8_beV9jWK2Br4MMLysNHgM_vrFjwzT-pK-K-5CvIVwTSX3vkminRgxmSCbSJci53hPok) 2015

## 基于RNN或者LSTM的语音增强

1.[Multiple-target deep learning for LSTM-RNN based speech enhancement](https://link.zhihu.com/?target=http%3A//ieeexplore.ieee.org/document/7895577/) 2017

2.[Densely Connected Progressive Learning for LSTM-Based Speech Enhancement](https://link.zhihu.com/?target=https%3A//ieeexplore.ieee.org/document/7895577) 2017

## 将语音合成应用于语音增强上面

1.[Speech Denoising by Parametric Resynthesis](https://link.zhihu.com/?target=https%3A//arxiv.org/pdf/1904.01537.pdf) 2019

