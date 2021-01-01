title: 初识语音识别及 Kaldi 的安装使用 
tags:
  - 语音识别
  - 机器识别
  - HMM
  - MFCC
  - Kaldi
categories:
  - 人工智能
  - 语音识别
author: Steven Hu
date: 2019-05-02 20:20:20
---

这两天一个朋友给我提了个涉及语音识别的“需求”（当然是不能调用如科大讯飞等各种开放平台的语音SDK），主要识别的当然是中文，不过因为之前从未接触过机器学习领域的技术，所以这里也只是属于快速了解语音识别相关技术、以及提供一个能够运行语音识别的示例。

> 因为示例都是在自己的 PC 跑的，没有安装 CUDA，所以后面的训练和使用都没有采用 DNN-HMM 来训练，还是采用的传统的声学模型 GMM-HMM。

# 初识语音识别

## 定义

从 wiki 上摘抄一下，语音识别（speech recognition）技术，也被称为自动语音识别（英语：Automatic Speech Recognition, ASR）、计算机语音识别（英语：Computer Speech Recognition）或是语音转文本识别（英语：Speech To Text, STT），其目标是以计算机自动将人类的语音内容转换为相应的文字。

> https://zh.wikipedia.org/wiki/%E8%AF%AD%E9%9F%B3%E8%AF%86%E5%88%AB

<!-- more -->

## 语音识别系统的分类

按照不同纬度如下分类：

- 按词汇量（vocabulary）大小分类：
    + 小词汇量：几十个词；
    + 中等词汇量：几百个到上千个词
    + 大词汇量：几千到几万个
- 按说话的方式（style）分类：
    + 孤立词（isolated words）
    + 连续（continously）
- 按声学（Acoustic）环境分类：
    + 录音室
    + 不同程度的噪音环境
- 按说话人（Speaker）分类：
    + 说话人相关（Speaker depender）
    + 说话人无关（Speaker indepenent）

## 一些基础概念和符号

- 音素（Phoneme）：单词的发音都是由音素构成，对于英语，常用的音素集是 CMU 的 39 个音素构成的音素集。而对于汉语，一般直接用全部声母和韵母作为音素集，另外汉语识别还要考虑音调。

> [The CMU Pronouncing Dictionary.](http://www.speech.cs.cmu.edu/cgi-bin/cmudict)

- **声学模型**：是将声学和发音学（phonetics）的知识进行整合，以特征提取部分生成的特征作为输入，并为可变长特征序列生成声学模型分数。
- **语言模型**：通过从训练语料（通常是文本形式）学习词之间的相互关系，来估计假设词序列的可能性，又叫语言模型分数。
- **GMM**：Gaussian Mixture Model, 高斯混合模型，描述基于傅里叶频谱语音特征的统计模型，用于传统声学模型的建模中。
- **HMM**：Hidden Markov Model, 隐马尔科夫模型，是一种用来描述含有隐含未知参数的马尔科夫过程，其难点是从可观察的参数中确定该过程的隐含参数。然后利用这些参数来作进一步的分析，例如模式识别。
- **MFCC**：Mel-Frequency Cepstral Coefficients, 梅尔频率倒谱系数，是组成梅尔频率倒谱的系数。衍生自音讯片段的倒频谱（cepstrum）。倒谱与梅尔频率倒谱的区别在于，梅尔频率倒谱的频带划分是在梅尔刻度上等距划分的，它比用于正常的对数倒频谱中的线性间隔的频带更接近人类的听觉系统。广泛应用于语音识别中。
- **Fbank**：Mel Frequency Filter Bank, 梅尔频率滤波器组。
- **WER**：Word Error Rate, 词错误率，是最常见的衡量语音识别系统性能的指标。

## 基本原理

一个语音识别系统主要包括信号处理和特征提取、声学模型训练、语言模型训练以及识别引擎等几个核心部分，下图为语音识别的原理框图：

{% asset_img asr-001.png %}

- **特征提取**：语音识别第一步就是特征提取，去除掉语音信号中对于语音识别无用的冗余信息（如背景噪音），保留能够反映语音本质特征的信息（为后面的声学模型提取合适的特征向量），并用一定的形式表示出来；较常用的特征提取算法有 MFCC。
- **声学模型训练**：根据语音库的特征参数训练出声学模型参数，在识别的时候可以将待识别的语音的特征参数同声学模型进行匹配，从而得到识别结果。目前主流的语音识别系统多采用 HMM 进行声学模型建模。
- **语言模型训练**：就是用来计算一个句子出现的概率模型，主要用于决定哪个词序列的可能性更大。语言模型分为三个层次：字典知识、语法知识、句法知识。对训练文本库进行语法、语义分析，经过基于统计模型训练得到语言模型。
- **语音解码与搜索算法**：其中解码器就是针对输入的语音信号，根据已经训练好的声学模型、语言模型以及字典建立一个识别网络，再根据搜索算法在该网络中寻找一条最佳路径，使得能够以最大概率输出该语音信号的词串，这样就确定这个语音样本的文字。

# Kaldi

## 简介

关于语音识别的技术只能”点到为止“，下面只能是硬着头皮跑示例了 :)

> https://github.com/kaldi-asr/kaldi

Kaldi 是目前非常流行的开源语音识别工具（Toolkit），主要使用的是 WFST 来实现解码算法。

Kaldi 的架构如下图：

{% asset_img asr-002.png %}

上图来自于 Kaldi 发起者 Daniel Povey 等人的论文 《The Kaldi Speech Recognition Toolkit》，在该论文中也详细描述了 Kaldi 的架构。

从图中可以看到最上面的是各种外部工具，包括：用于线性代数的库 BLAS/LAPACK，用于有限状态转换器（FST, Finite-State Transducer）的开源库 OpenFST。中间层是 Kaldi C++ 库，其中包括 HMM, GMM 等相关的代码，最下面的两层是编译出来的可执行程序，以及一些用于辅助执行不同步骤的脚本。

## 下载安装

我的机器环境是：

- Intel(R) Core(TM) i5-8500 3.00GHz
- 16 GB 内存
- 无 Nvida 显卡
- Ubuntu 16.04

Kaldi 的编译源码是 2019/04/30 直接从 Github 源码 master 分支直接下载的。如下几个目录分别介绍下：

- tools/: 主要存放了 Kaldi 依赖的包已经各种工具，如：OpenFST, ATLAS, IRSTLM, sph2pipe 等等。
- src/: Kaldi 的源代码；
- egs/: 为各种示例项目和代码；

详细的安装过程和指令可以参考 ./INSTALL, src/INSTALL, 以及 tools/INSTALL。

编译 tools 和 src 之前，可以先执行 tools/extras/check_dependencies.sh 来查看需要安装哪些依赖。

在编译安装好 tools 和 src 之后，除了 tools 下的各种库和工具生成之外，src/bin/ 下也生成了很多的 toolkits，这些都是我们下面模型训练和识别示例中需要的。

## 模型训练

### 语料库

我采用的语料库是 thchs30，是清华大学提供的30个小时的中文语料库，可以到 OpenSLR 下载，链接地址为 https://www.openslr.org/18/ 。其中包括如下几个文件：

- data_thchs30.tgz, 6.4G, sppech data and transcripts
- test_noise.tgz, 1.9G, standard 0db noisy test data
- resource.tgz, 24M, supplementary resources, incl. lexicon for training data, noise samples

下载之后全部进行解压，我这里解压到了 ~/works/asr/dataset/thchs30/ 下，如下: 

```shell
steven@Jenkins-server:~/works/asr/dataset/thchs30$ ll
total 20
drwxrwxr-x 5 steven steven 4096 5月   1 12:49 ./
drwxrwxr-x 4 steven steven 4096 5月   1 12:49 ../
drwxr-xr-x 8 steven steven 4096 12月 30  2015 data_thchs30/
drwxr-xr-x 4 steven steven 4096 1月  25  2016 resource/
drwxr-xr-x 5 steven steven 4096 1月  25  2016 test-noise/
```

下面进入 egs/thchs30/s5/ 目录，如下: 

```shell
steven@Jenkins-server:~/kaldi/egs/thchs30/s5$ ls -l
total 108
-rw-rw-r--  1 steven steven  1062 5月   1 12:53 cmd.sh
drwxrwxr-x  2 steven steven  4096 4月  30 23:28 conf
drwxrwxr-x 15 steven steven  4096 5月   1 14:13 data
drwxrwxr-x 18 steven steven  4096 5月   1 14:24 exp
drwxrwxr-x  5 steven steven  4096 5月   1 14:22 fbank
drwxrwxr-x  4 steven steven  4096 4月  30 23:28 local
drwxrwxr-x  5 steven steven  4096 5月   1 13:14 mfcc
-rw-------  1 steven steven 57117 5月   1 17:40 nohup.out
-rwxrwxr-x  1 steven steven   374 4月  30 23:28 path.sh
-rw-rw-r--  1 steven steven  4469 4月  30 23:28 RESULTS
-rwxrwxr-x  1 steven steven  5287 5月   1 13:03 run.sh
lrwxrwxrwx  1 steven steven    19 4月  30 23:28 steps -> ../../wsj/s5/steps/
lrwxrwxrwx  1 steven steven    19 4月  30 23:28 utils -> ../../wsj/s5/utils/
```

打开 cmd.sh，编辑如下（注释掉 queue.pl 的执行方式）：

```
export train_cmd=run.pl
export decode_cmd="run.pl --mem 4G"
export mkgraph_cmd="run.pl --mem 8G"
export cuda_cmd="run.pl --gpu 1"
#export train_cmd=queue.pl
#export decode_cmd="queue.pl --mem 4G"
#export mkgraph_cmd="queue.pl --mem 8G"
#export cuda_cmd="queue.pl --gpu 1"
```

> 为什么不用 queue.pl? 因为我们运行的是本机独立机器执行，可以参考 cmd.sh 的注释。

继续打开 run.sh，修改 thchs30 语料库的路径：

```
#corpus and trans directory
#thchs=/nfs/public/materials/data/thchs30-openslr
thchs=/home/steven/works/asr/dataset/thchs30
```

下面就可以开始执行 egs/thchs30/s5/run.sh 脚本了，该脚本中包含了整个训练过程中需要执行的各种命令，整个过程执行时间较长（我的机器跑了十几个小时），如下为部分执行日志：

```
creating data/{train,dev,test}
cleaning data/train
preparing scps and text in data/train
cleaning data/dev
preparing scps and text in data/dev
cleaning data/test
preparing scps and text in data/test
creating test_phone for phone decoding
steps/make_mfcc.sh --nj 4 --cmd run.pl data/mfcc/train exp/make_mfcc/train mfcc/train
utils/validate_data_dir.sh: Successfully validated data-directory data/mfcc/train
steps/make_mfcc.sh: [info]: no segments file exists: assuming wav.scp indexed by utterance.
Succeeded creating MFCC features for train
steps/compute_cmvn_stats.sh data/mfcc/train exp/mfcc_cmvn/train mfcc/train
Succeeded creating CMVN stats for train

...

steps/nnet/train.sh --copy_feats false --cmvn-opts --norm-means=true --norm-vars=false --hid-layers 4 --hid-dim
1024 --learn-rate 0.008 data/fbank/train data/fbank/dev data/lang exp/tri4b_ali exp/tri4b_ali_cv exp/tri4b_dnn

# INFO
steps/nnet/train.sh : Training Neural Network
     dir       : exp/tri4b_dnn
     Train-set : data/fbank/train 10000, exp/tri4b_ali
     CV-set    : data/fbank/dev 893 exp/tri4b_ali_cv

LOG ([5.5.314~1-1da8e]:main():cuda-gpu-available.cc:60)

### IS CUDA GPU AVAILABLE? 'Jenkins-server' ###
### CUDA WAS NOT COMPILED IN! ###
To support CUDA, you must run 'configure' on a machine that has the CUDA compiler 'nvcc' available.
# Accounting: time=1 threads=1
# Ended (code 1) at Wed May  1 14:24:20 CST 2019, elapsed time 1 seconds

...

steps/diagnostic/analyze_lats.sh: see stats in exp/mono/decode_test_phone/log/analyze_alignments.log
Overall, lattice depth (10,50,90-percentile)=(6,20,64) and mean=29.6
steps/diagnostic/analyze_lats.sh: see stats in exp/mono/decode_test_phone/log/analyze_lattice_depth_stats.log
local/score.sh --cmd run.pl --mem 4G data/mfcc/test_phone exp/mono/graph_phone exp/mono/decode_test_phone
local/score.sh: scoring with word insertion penalty=0.0,0.5,1.0
```

在上面的日志输出中可以看到，因为默认 DNN 是用 GPU 来跑的，应该也可以改成 CPU 来跑，但是从网上看到有人用 CPU 跑了 7, 8 天才迭代了 17, 18 次，也就放弃试了。

从日志的输出也可以看出整个训练过程大致包括数据准备、monophone 单因素训练（steps/train_mono.sh）、tri1 三因素训练（steps/train_deltas.sh）、tri2 进行 lda_mllt 特征变换（steps/train_lda_mllt.sh）、tri3b 进行 sat 自然语言适应（step/train_sat.sh）、tri4b 进行 quick 训练（step/train_quick.sh），之后就是 DNN 训练。

上面的这些步骤，每一次都会生成一个可用的模型，具体可以查看 exp 目录，所有的步骤都在这里输出，如下：

```shell
steven@Jenkins-server:~/kaldi/egs/thchs30/s5/exp$ ls -l
total 64
drwxrwxr-x 5 steven steven 4096 5月   1 14:24 fbank_cmvn
drwxrwxr-x 5 steven steven 4096 5月   1 14:22 make_fbank
drwxrwxr-x 5 steven steven 4096 5月   1 13:14 make_mfcc
drwxrwxr-x 5 steven steven 4096 5月   1 13:15 mfcc_cmvn
drwxrwxr-x 7 steven steven 4096 5月   1 16:37 mono
drwxrwxr-x 3 steven steven 4096 5月   1 13:27 mono_ali
drwxrwxr-x 7 steven steven 4096 5月   1 15:45 tri1
drwxrwxr-x 3 steven steven 4096 5月   1 13:31 tri1_ali
drwxrwxr-x 7 steven steven 4096 5月   1 15:38 tri2b
drwxrwxr-x 3 steven steven 4096 5月   1 13:39 tri2b_ali
drwxrwxr-x 9 steven steven 4096 5月   1 15:56 tri3b
drwxrwxr-x 3 steven steven 4096 5月   1 13:57 tri3b_ali
drwxrwxr-x 9 steven steven 4096 5月   1 16:16 tri4b
drwxrwxr-x 3 steven steven 4096 5月   1 14:13 tri4b_ali
drwxrwxr-x 3 steven steven 4096 5月   1 14:13 tri4b_ali_cv
drwxrwxr-x 4 steven steven 4096 5月   1 14:24 tri4b_dnn
```

下面看下 tri1，因为下面的示例会以 tri1 模型来识别。

```shell
steven@Jenkins-server:~/kaldi/egs/thchs30/s5/exp/tri1$ ls -l
total 66176
-rw-rw-r-- 1 steven steven  3435133 5月   1 13:31 35.mdl
-rw-rw-r-- 1 steven steven     8326 5月   1 13:31 35.occs
-rw-rw-r-- 1 steven steven  1524364 5月   1 13:30 ali.1.gz
-rw-rw-r-- 1 steven steven  1495302 5月   1 13:30 ali.2.gz
-rw-rw-r-- 1 steven steven  1655834 5月   1 13:30 ali.3.gz
-rw-rw-r-- 1 steven steven  1533052 5月   1 13:30 ali.4.gz
-rw-rw-r-- 1 steven steven        1 5月   1 13:27 cmvn_opts
drwxrwxr-x 4 steven steven     4096 5月   1 17:21 decode_test_phone
drwxrwxr-x 4 steven steven     4096 5月   1 15:44 decode_test_word
lrwxrwxrwx 1 steven steven        6 5月   1 13:31 final.mdl -> 35.mdl
lrwxrwxrwx 1 steven steven        7 5月   1 13:31 final.occs -> 35.occs
-rw-rw-r-- 1 steven steven 14109331 5月   1 13:27 fsts.1.gz
-rw-rw-r-- 1 steven steven 13876553 5月   1 13:27 fsts.2.gz
-rw-rw-r-- 1 steven steven 15234357 5月   1 13:27 fsts.3.gz
-rw-rw-r-- 1 steven steven 14475131 5月   1 13:27 fsts.4.gz
drwxrwxr-x 3 steven steven     4096 5月   1 15:45 graph_phone
drwxrwxr-x 3 steven steven     4096 5月   1 13:40 graph_word
drwxrwxr-x 2 steven steven    12288 5月   1 13:31 log
-rw-rw-r-- 1 steven steven        2 5月   1 13:27 num_jobs
-rw-rw-r-- 1 steven steven     2098 5月   1 13:27 phones.txt
-rw-rw-r-- 1 steven steven     8161 5月   1 13:27 questions.int
-rw-rw-r-- 1 steven steven    35098 5月   1 13:27 questions.qst
-rw-rw-r-- 1 steven steven   305009 5月   1 13:27 tree

steven@Jenkins-server:~/kaldi/egs/thchs30/s5/exp/tri1$ ls -l graph_word/
total 842184
-rw-rw-r-- 1 steven steven       290 5月   1 13:31 disambig_tid.int
-rw-rw-r-- 1 steven steven 861725005 5月   1 13:40 HCLG.fst
-rw-rw-r-- 1 steven steven         5 5月   1 13:40 num_pdfs
drwxrwxr-x 2 steven steven      4096 5月   1 13:40 phones
-rw-rw-r-- 1 steven steven      2098 5月   1 13:40 phones.txt
-rw-rw-r-- 1 steven steven    646753 5月   1 13:40 words.txt
```

其中，**final.mdl** 就是训练出来的可以使用的模型，另外，在 **graph_word** 下面的 **words.txt** 和 **HCLG.fst** 分别为字典以及有限状态机。单独介绍这三个文件，是因为我们下面的示例主要基于这三个文件来识别的。

## 识别示例

下面我们用上面训练的 tri1 模型来识别一个给定的音频文件。我们这里使用 **online-wav-gmm-decode-faster** 工具来回放指定的 wav 文件并进行识别。

对于当前源码，在上面的编译安装过程中，默认情况下并不会生成 **online-wav-gmm-decode-faster** 程序（默认会生成的是 online2 相关程序），此时需要自己手动编译此工具，如下：

```shell
steven@Jenkins-server:~/kaldi/src/online$ make
steven@Jenkins-server:~/kaldi/src/online$ cd ../onlinebin
steven@Jenkins-server:~/kaldi/src/onlinebin$ ls -l
total 137340
drwxrwxr-x 3 steven steven     4096 4月  30 23:28 java-online-audio-client
-rw-rw-r-- 1 steven steven     1364 4月  30 23:28 Makefile
-rwxrwxr-x 1 steven steven  8351432 5月   1 21:09 online-audio-client
-rw-rw-r-- 1 steven steven    11986 4月  30 23:28 online-audio-client.cc
-rwxrwxr-x 1 steven steven 43179504 5月   1 21:09 online-audio-server-decode-faster
-rw-rw-r-- 1 steven steven    13653 4月  30 23:28 online-audio-server-decode-faster.cc
-rwxrwxr-x 1 steven steven 27141312 5月   1 21:08 online-gmm-decode-faster
-rw-rw-r-- 1 steven steven     8113 4月  30 23:28 online-gmm-decode-faster.cc
-rwxrwxr-x 1 steven steven  7978200 5月   1 21:08 online-net-client
-rw-rw-r-- 1 steven steven     4970 4月  30 23:28 online-net-client.cc
-rwxrwxr-x 1 steven steven 26017240 5月   1 21:08 online-server-gmm-decode-faster
-rw-rw-r-- 1 steven steven     8255 4月  30 23:28 online-server-gmm-decode-faster.cc
-rwxrwxr-x 1 steven steven 27878392 5月   1 21:08 online-wav-gmm-decode-faster
-rw-rw-r-- 1 steven steven    10116 4月  30 23:28 online-wav-gmm-decode-faster.cc
```

下面为示例的步骤：

1. 将 kaldi/egs/voxforge/online_demo 拷贝到 kaldi/egs/thchs30/ 下;
2. 在 kaldi/egs/thchs30/online_demo/ 下新建两个文件夹 online-data/ 和 work/，以及在 online-data 下新建两个文件夹 audio/ 和 models/，audio/ 下面可以存放需要回放和识别的语音文件，路径结构如下：

```shell
steven@Jenkins-server:~/kaldi/egs/thchs30/online_demo$ tree -L 4
.
├── online-data
│   ├── audio
│   │   ├── C21_517.wav      // 需要回放和识别的语音文件，可以放多个
│   │   └── trans.txt        // 为空即可
│   └── models
│       └── tri1
│           ├── 35.mdl       // 模型训练步骤生成的模型文件
│           ├── final.mdl    // 模型训练步骤生成的模型文件
│           ├── HCLG.fst     // 拷贝 thchs30/s5/exp/graph_word/HCLG.fst
│           └── words.txt    // 拷贝 thchs30/s5/exp/graph_word/words.txt
├── README.txt
├── run.sh
└── work

```

3. 打开 kaldi/egs/thchs30/online_demo/run.sh，更改如下：

    - 更新模型名

```shell
# Change this to "tri2a" if you like to test using a ML-trained model
#ac_model_type=tri2b_mmi
ac_model_type=tri1
```

    - 注释掉如下这段从 voxforge 下载现网的测试预料和模型的代码

```shell
#if [ ! -s ${data_file}.tar.bz2 ]; then
#    echo "Downloading test models and data ..."
#    wget -T 10 -t 3 $data_url;
#
#    if [ ! -s ${data_file}.tar.bz2 ]; then
#        echo "Download of $data_file has failed!"
#        exit 1
#    fi
#fi
```

    - 修改如下执行识别的命令（将模型的部分更改为 $ac_model/final.mdl）

```shell
        online-wav-gmm-decode-faster --verbose=1 --rt-min=0.8 --rt-max=0.85\
            --max-active=4000 --beam=12.0 --acoustic-scale=0.0769 \
            scp:$decode_dir/input.scp $ac_model/final.mdl $ac_model/HCLG.fst \
            $ac_model/words.txt '1:2:3:4:5' ark,t:$decode_dir/trans.txt \
            ark,t:$decode_dir/ali.txt $trans_matrix;;
```

4. 执行 kaldi/egs/thchs30/online_demo/run.sh，如下输出：

```shell
steven@Jenkins-server:~/kaldi/egs/thchs30/online_demo$ ./run.sh

  SIMULATED ONLINE DECODING - pre-recorded audio is used

  The (bigram) language model used to build the decoding graph was
  estimated on an audio book's text. The text in question is
  "King Solomon's Mines" (http://www.gutenberg.org/ebooks/2166).
  The audio chunks to be decoded were taken from the audio book read
  by John Nicholson(http://librivox.org/king-solomons-mines-by-haggard/)

  NOTE: Using utterances from the book, on which the LM was estimated
        is considered to be "cheating" and we are doing this only for
        the purposes of the demo.

  You can type "./run.sh --test-mode live" to try it using your
  own voice!

online-wav-gmm-decode-faster --verbose=1 --rt-min=0.8 --rt-max=0.85 --max-active=4000 --beam=12.0 --acoustic-scale=0.0769 scp:./work/input.scp online-data/models/tri1/final.mdl online-data/models/tri1/HCLG.fst online-data/models/tri1/words.txt 1:2:3:4:5 ark,t:./work/trans.txt ark,t:./work/ali.txt
File: C21_517
他 曾 为 火药 仓库 研究 并未 装置 为 国家 造 币 厂 研究 减少 金币 磨损 消耗 的 办法

compute-wer --mode=present ark,t:./work/ref.txt ark,t:./work/hyp.txt
%WER -nan [ 0 / 0, 0 ins, 0 del, 0 sub ]
%SER -nan [ 0 / 0 ]
Scored 0 sentences, 0 not present in hyp.
```

这里采用的 tri1 模型的准确度不太理想，下面再试试 tri2b，以及 DNN 模型 :)

# 参考资料

- [wiki-语音识别](https://zh.wikipedia.org/wiki/%E8%AF%AD%E9%9F%B3%E8%AF%86%E5%88%AB)
- [wiki-隐马尔科夫模型](https://zh.wikipedia.org/wiki/%E9%9A%90%E9%A9%AC%E5%B0%94%E5%8F%AF%E5%A4%AB%E6%A8%A1%E5%9E%8B)
- [一文详解高斯混合模型原理](https://zhuanlan.zhihu.com/p/31103654)
- [wiki-梅尔频率倒谱系数](https://zh.wikipedia.org/zh/%E6%A2%85%E5%B0%94%E9%A2%91%E7%8E%87%E5%80%92%E8%B0%B1%E7%B3%BB%E6%95%B0)
- [wiki-Word error rate](https://en.wikipedia.org/wiki/Word_error_rate)
- [The CMU Pronouncing Dictionary](http://www.speech.cs.cmu.edu/cgi-bin/cmudict)
- [The Kaldi Speech Recognition Toolkit](https://publications.idiap.ch/downloads/papers/2012/Povey_ASRU2011_2011.pdf)
- [《解析深度学习:语音识别实践》](https://www.amazon.cn/dp/B01H2AXN1I/ref=sr_1_1)
