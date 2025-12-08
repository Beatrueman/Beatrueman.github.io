---
title: NLP语义匹配学习赛记录
date: {{ date }}
tags: [机器学习]
categories: [机器学习]
---

# 赛题介绍

## 背景

本次项目来源于阿里云天池的日常学习赛：[【NLP系列学习赛】语音助手：对话短文本语义匹配_学习赛_天池大赛-阿里云天池的排行榜](https://tianchi.aliyun.com/competition/entrance/532329/rankingList)

背景是OPPO公司有一个语音助手，这个语音助手需要根据对话来识别意图。赛题要求是参赛队伍需要根据脱敏后的短文本query-pair，来预测他们是否属于同一语义，最终提交一个预测概率文件来进行评测。

简单来说就是提供了两句话，需要判断这两句话是不是同一个意思（语义匹配）。

比如用下面的几个训练样本来举例：

```shell
肖战的粉丝叫什么名字 肖战的粉丝叫什么 1

王者荣耀里面打野谁最厉害 王者荣耀什么英雄最好玩 0

我想换个手机 我要换手机 1

我是张睿 我想张睿 0

不想 不想说 0
```

可以看到意思完全不同的两句话，最终真值标签输出0，表示不匹配。而意思相同的两句话，最终真值标签输出1。

## 提交

最终需要提交一份预测结果文件，结果文件中每行为一个0-1的预测值，代表query-lair语义匹配的概率，与测试数据每行一一对应。

```shell
0.001
0.999
```

## 评估

![image](https://gitee.com/beatrueman/images/raw/master/20251208211020703.png)

# 赛题解析

本赛题属于自然语言处理（NLP）领域的文本匹配/语义相似度分类任务。给定两段脱敏后的短文本（query-pair），系统需要判断它们是否表达相同或相近的语义，并输出二分类结果。

## 难点与挑战

- 文本脱敏：原始词语被替换为数字ID，模型就无法直接使用词义信息，只能从数字序列中学习潜在的语义模式。

  如果直接提供了文本，那问题非常简单，直接使用bert-base-chinese这种预训练中文模型就能得到很好的结果，但是这种脱敏数据，由于不知道词表，就无法使用开源的预训练模型。
- 句子成对输入：任务需要同时理解两段文本并进行比较，而不仅仅是单句分类。
- 数字ID分布不均：高频词和低频词的数据差异很大，需要合理的enmedding初始化策略，避免低频数字影响训练效果
- 句子长度和复杂关系：简单模型难以捕获远距离依赖，需要使用能够建模全局信息的网络结果（比如Transformer、BiLSTM等）

## 数据集结构分析

提供了4个.tsv数据集，学习赛采用`gaiic_track3_round1_testB_20210317.tsv`​来做为测试集。

```shell
gaiic_track3_round1_testA_20210228.tsv
gaiic_track3_round1_testB_20210317.tsv
gaiic_track3_round1_train_20210228.tsv # 训练样本10万
gaiic_track3_round2_train_20210407.tsv # 复赛训练样本30万
```

对于训练集，每行为一个训练样本，由query-pair和真值组成，每行格式如下：

- query-pair格式：query以中文为主，中间可能带有少量英文单词（如英文缩写、品牌词、设备型号等），采用UTF-8编码，未分词，两个query之间使用\\t分割。
- 真值：真值可为0或1，其中1代表query-pair语义相匹配，0则代表不匹配，真值与query-pair之间也用\\t分割。

```gaiic_track3_round1_train_20210228.tsv
72 29 68 69 70 533 1661 1877	28 12 347 72 29 369 16	1
12 453 317 43 1163 317 59 355 1024	8 9 1847 6 1163 317 59	0
12 19 1162 126 53 66	12 19 79 389 126 53 66	1
275 552 553 433 881 338 1104 101 202 2343 14825	995 551 550 1660 2830 1075 662 935	0
421 330 62 12 80 81 82 76	202 62 12 80 838 76	1
177 455 456 3474 964 1364 55 1364	133 134 2246	1
29 168 12 19 1003 719 23 29 263 276	29 23 12 115 115 263 16	1
```

# 方案介绍

针对上述难点，本方案设计如下：

1. **双塔 Transformer 编码**

    分别编码两段文本，能够捕获全局语义特征，解决句子长度和复杂关系的问题。
2. **向量融合进行分类**

    将两段文本编码向量进行融合，再输入 MLP 分类器，实现句子成对匹配。
3. **结合频度表初始化 embedding**

    针对脱敏数字 ID，通过高频 ID 初始化 embedding，帮助模型快速学习有效特征，缓解数据稀疏问题。
4. **合理训练策略**

    合并训练集、划分验证集、使用早停机制和批量训练，充分利用数据和计算资源，提高模型稳定性和精度。

利用这套方案，最终在比赛平台上获得了0.84的最终成绩

![image](https://gitee.com/beatrueman/images/raw/master/20251208211020705.png)

## 代码结构

![mermaid-diagram 3](https://gitee.com/beatrueman/images/raw/master/20251208211020706.png)

```gaiic_track3_round1_train_20210228.tsv
# ================= 配置 =================
config = {
    "max_len": 128,
    "batch_size": 128,
    "epochs": 8,
    "lr": 2e-4,
    "device": "cuda" if torch.cuda.is_available() else "cpu",
    "embed_dim": 512,
    "num_heads": 8,
    "num_layers": 4,
}
```

### 张量转换：PairDataset

功能：

- 读取句子对数据（q1，q2）
- 处理成固定长度序列（padding、截断）
- 生成attention mask

#### __init__()

构造q1、q2、label以及最大序列长度max_len（默认 64，每条句子只保留一半长度，因为双塔模型会拼接两条句子，保留一半长度虽然有丢失信息的风险，但是这么做可以确保两条句子信息对等，提高显存效率，因为序列越长显存占用越高）

max_len = q1 + q2 + [CLS/SEP/padding]

#### 序列填充函数：pad_seq()

功能：保证每条输入序列长度固定，返回half_len + 2的list

- [0]相当于[CLS]token，表示序列开头

- [1]相当于[SEP]token，表示序列结尾
- [2]相当于[PAD]，作为填充，保持固定长度

#### 获取样本函数：__getitem__()

功能：返回对应数据集的训练格式。

- 如果是训练集，返回（q1,a1,q2,a2）
- 如果是测试集，返回（q1,a1,q2,a2,label）

这里a1,a2表示`attention_mask`​，1表示有效token，0表示`padding token`​。Transformer 在编码时用 `src_key_padding_mask=(attn_mask == 0)`​ 屏蔽 padding。

### 双塔Transformer：SiameseTransformer

两条句子分别经过相同编码器 -> 得到向量 -> 融合 -> 分类

#### __init__()

```
self.embedding = nn.Embedding(vocab_size, embed_dim)
```

功能：为每一个数字ID创建一个随机向量（长度为 `embed_dim`​），也就是 `vocab_size`​ × `embed_dim`​ 的矩阵。之后通过 `init_embedding_from_freq`​ 用高频ID对应的更有意义的向量进行替换，以帮助模型更快学到有效语义。

- vocab_size：数字ID的总数量
- embed_dim：每个token被映射的向量维度

```
encoder_layer = nn.TransformerEncoderLayer(
    d_model=embed_dim,  # 输入向量维度
    nhead=num_heads,    # 多头注意力的头数
    dim_feedforward=embed_dim * 4, # 前馈网络隐藏层大小
    batch_first=True, # 输入维度[batch, seq_len, embed_dim]
    dropout=0.1 # 防止过拟合：随机丢弃部分神经元，也就是随机将10%的神经元输出置为0
)
self.encoder = nn.TransformerEncoder(encoder_layer, num_layers=num_layers)
```

功能：定义Transformer编码器

```
self.mlp = nn.Sequential(
    nn.Linear(embed_dim * 2, embed_dim), # 两个句子的向量拼接【输入 1024 (512*2)，输出 512】
    nn.ReLU(), # ReLU激活函数
    nn.Linear(embed_dim, 2) # 分类：线性层输出2个logit（对应0/1）【输入 512，输出 2 (类别数)】
)
```

功能：融合两条句子的编码向量并做分类

#### encode

功能：把一句话从数字序列编码成固定长度向量

1. 将数字序列转换为向量序列emb
2. 把emb用Transformer编码，提示其在attm_mask==0不参与计算，输出为out
3. 然后取out的第一个token的向量作为句子表示（从 Transformer 输出的一长串向量中，只取出一个代表整句话的向量）

#### forward

功能：模型前向传播

1. 对q1进行encode，得到句子向量v1
2. 对q2进行encode，得到句子向量v2
3. 使用torch.cat拼接两个向量
4. 使用mlp进行分类输出2个logit

### 训练：train

1. 设置训练模式（model.train()）、优化器Adam与损失函数（交叉熵CrossEntropyLoss）
2. 设置早停，当连续patience轮验证准确率没有提升，就停止训练，降低过拟合风险。

    `best_val_auc`​：记录验证集的最好auc

    `patience`​：早停阈值，如果连续 `patience`​ 轮验证准确率没有提升，就停止训练

    `early_stop_counter`​：记录连续验证没有提升的轮数
3. 开始epoch循环

    每次循环遍历训练集的每个batch，把数据转移到GPU上

    接着前向传播：`logits = model(...)`​ → 模型输出每个类别的分数

    计算损失Loss

    最后反向传播与梯度更新

    - `optimizer.zero_grad()`​ 清除上一步梯度
    - `loss.backward()`​ 计算梯度
    - `optimizer.step()`​ 更新模型参数

    累计batch损失，打印输出
4. 验证阶段：model.eval()

    遍历验证集，计算每一轮验证集的AUC。

1. 早停判断

    如果当前验证准确率比历史最好值高，就更新best_val_acc，重置早停计数器并保存当前模型。否则早停计数器加1，若连续patience轮没提升就停止训练。

### 预测：predict

遍历测试集，输入两条句子及其attention_mask，得到模型输出logits。

然后通过softmax将分数转换为概率，取标签1（匹配）的概率。

## 频度表初始化：init_embedding_from_freq

为了应对脱敏文本的问题，这里参考了论坛中的一位参赛选手的方案：[AUC 0.9094 BERT经典方案_天池技术圈-阿里云天池](https://tianchi.aliyun.com/forum/post/935989)

即让这些脱敏数字ID变得相对来说有意义，让模型一开始就对高频数字有”合理的表示“，帮助模型更快学习语义。

为此，我统计了训练集里所有数字ID的频率，然后从[https://lingua.mtsu.edu/chinese-computing/statistics/char/list.php?Which=MO&amp;utm_source=chatgpt.com](https://lingua.mtsu.edu/chinese-computing/statistics/char/list.php?Which=MO&utm_source=chatgpt.com)中下载了一份汉字频度表文件，取前max_chars个最常用字符，保存在top_chars中。

接着为top_chars生成随机向量（char_vectors），最后将最频繁出现的数字ID对应的embedding替换为char_vectors，这样高频ID就有了合理的表示，低频ID仍然保持随机初始化。

![IMG_0506](https://gitee.com/beatrueman/images/raw/master/20251208211020707.jpg)

## 数据集划分

正式比赛分为了初赛和复赛，提供了两个训练集。对于学习赛，我将两个训练集合并为一个大的训练集，训练样本有10+40w条。

```
df_train = pd.concat([df_train1, df_train2], ignore_index=True)
```

这里我划分了训练集和验证集。对于整个数据集，90%作为训练集，10%作为验证集来评估模型效果、调参和早停。

```
df_tr, df_val = train_test_split(
    df_train, 
    test_size=0.1, 
    random_state=42, 
    stratify=df_train['label'] # 保持正负样本比例一致，防止验证集偏斜，提高验证结果可靠性
)
```

统计vocab_size，因为Embedding层需要知道有多少词/数字ID。刚开始没有统计这个，导致ID超过embedding行数就会报错：某个索引值超出了目标张量的大小。

```
C:\cb\pytorch_1000000000000\work\aten\src\ATen\native\cuda\Indexing.cu:1290: block: [103,0,0], thread: [60,0,0] Assertion `srcIndex < srcSelectDimSize` failed.
C:\cb\pytorch_1000000000000\work\aten\src\ATen\native\cuda\Indexing.cu:1290: block: [103,0,0], thread: [61,0,0] Assertion `srcIndex < srcSelectDimSize` failed.
  File "c:\Users\bearx\Desktop\project\code\train\train.py", line 154, in <module>
    predictions = predict(model, test_loader, device)
  File "c:\Users\bearx\Desktop\project\code\train\train.py", line 111, in predict
    probs = torch.softmax(out, dim=1)[:, 1]  # 取概率
RuntimeError: CUDA error: device-side assert triggered
CUDA kernel errors might be asynchronously reported at some other API call, so the stacktrace below might be incorrect.
For debugging consider passing CUDA_LAUNCH_BLOCKING=1.
Compile with `TORCH_USE_CUDA_DSA` to enable device-side assertions.
```

## 提分尝试

提分路径：0.78（BiLSTM） => 0.73（BERT）=> 0.77（频度表映射+预训练base-bert-chinese） => 0.84（双塔Transformer）

### BiLSTM

BiLSTM（Bidirectional Long Short-Term Memory）：双向长短期记忆网络是一种RNN变体，能够处理序列文本数据（比如文本）并捕获前后依赖信息。

它擅长捕捉长距离依赖，经常用于文本分类、句子相似度匹配等NLP任务。

```
class BiLSTM(nn.Module):
    def __init__(self, vocab_size, embed_dim=128, hidden_dim=128):
        super().__init__()
        # 1. Embedding 层，将数字ID映射为向量
        self.embedding = nn.Embedding(vocab_size + 10, embed_dim, padding_idx=0)

        # 2. BiLSTM 层
        # batch_first=True 表示输入 shape = (batch, seq_len, embed_dim)
        # bidirectional=True 表示双向LSTM
        self.lstm = nn.LSTM(embed_dim, hidden_dim, batch_first=True, bidirectional=True)

        # 3. 全连接层用于分类
        self.fc = nn.Sequential(
            nn.Linear(hidden_dim * 4, 256),  # hidden_dim*4：两条句子 * 双向输出
            nn.ReLU(),
            nn.Dropout(0.3),
            nn.Linear(256, 2)  # 输出2类
        )
		
	# 将每条句子变成一个固定长度向量
    def encode(self, x):
        x = self.embedding(x)      # 转为向量
        out, _ = self.lstm(x)      # BiLSTM 编码
        out, _ = torch.max(out, dim=1)  # 取序列维度的最大值（Max Pooling）
        return out
	
	# 前向传播
	def forward(self, q1, q2):
        v1 = self.encode(q1)       # 编码第一条句子
        v2 = self.encode(q2)       # 编码第二条句子
        x = torch.cat([v1, v2], dim=1)  # 拼接两条句子向量
        return self.fc(x)           # 分类输出
```

利用BiLSTM获取了0.78的成绩

![image](https://gitee.com/beatrueman/images/raw/master/20251208211020708.png)

放弃BiLSTM，选择Transformer的原因：

- 对于很长的数字ID序列，BiLSTM捕获全局模式不如Transformer强。Transformer通过自注意力机制，可以让每个token直接与所有token建立联系
- BiLSTM层数少（`"embed_dim": 128`​）、参数少，表达能力有限，难以学习到复杂的语义结构。Transformer可以堆叠多层encoder，embed_dim高（`"embed_dim": 512`​）
- 在相同的数据与训练流程下，BiLSTM 双塔的最佳得分约为 0.78，而 Transformer 双塔可以稳定达到 0.84。

### BERT

在最初尝试中，我将脱敏后的数字 ID 序列直接转换为 tensor 输入 BERT。但这种做法与 BERT 的设计机制完全不匹配，导致模型性能显著下降，最终准确率只有 **0.73**。

主要原因：

- BERT的词表是基于中文字符和词训练的

  BERT接收的不是数字ID，而是汉字，并且词表中包含万个token的语义分布。但是脱敏后的数字ID并没有出现在BERT词表里，所以模型无法区分句子之间的差异。
- BERT的enbedding是预训练好的中文语义空间

  比如很多中文都有固定语义的embedding，但数字ID完全破坏了这个模式，也就是说预训练只是全部失效

在意识到这些问题后，参考了论坛里一位参赛选手的方案，选择了一份汉字频度表，更改了输入，也就是把数字ID频率和汉字频度结合，embedding后让数字ID拥有一定意义，并且使用bert-base-chinese预训练模型，最终的分数达到0.78。

## 优化思路

1. 跑一轮epoch耗时过长，加上T4显卡是租的，按时间收费。所以目前仅跑了8个epoch，训练时验证集AUC一直在升高。其实可以再多跑几个epoch，结合早停机制，AUC应该还能继续提高。
2. 模型融合：将多个不同类型的模型进行融合，也就是说把各个模型的输出预测值做平均或者加权平均等。
3. 特征融合：目前只是concat(v1,v2)输入给MLP，可以加入差值（v1-v2）与乘积（v1 * v2）可能会提升判断力。

# 心得体会

以前虽然经常听到 Transformer、NLP 这些名词，但始终没有真正动手实践过。上学期的数据挖掘课程的O2O预测项目让我第一次接触到完整的机器学习流程，也感受到了它的魅力。而这学期通过这个 NLP 的小项目，我初步理解了处理文本类任务的基本思路，也认识了常用的模型和方法。

这次的提分过程相比去年做O2O预测要轻松一些——一方面得益于 AI 的帮助，另一方面论坛里大佬们的“点拨”也非常关键。整体提分还算顺利，虽然中间也遇到一些曲折。尤其是从 0.78 提升到 0.84 这段，我第一次真正感受到 Transformer 的强大，也深刻意识到显卡对深度学习实验的重要性。在那个阶段，我常常觉得做机器学习像是在做“黑盒测试”，也难怪很多人说是在“炼丹”——训练一次结果出来后，如果想继续提升，你必须同时考虑模型结构、参数、数据、预处理等多个因素，而且一时很难判断问题究竟出在哪里。也可能是因为目前能力有限，提分的方向常常没有特别明确的思路。

这次的 NLP 实践其实只是对 Transformer、BERT、BiLSTM 这些经典模型有了一个初步认识，它们内部的具体机制和细节我还没完全掌握。不过这次的体验激起了我继续深入学习的兴趣，未来如果有机会，我希望能够系统地理解这些模型背后的原理。

‍
