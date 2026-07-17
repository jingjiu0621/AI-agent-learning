# WordPiece / SentencePiece / Unigram LM

## 简单介绍

除了 BPE，还有几种重要的 Tokenizer 方案：WordPiece（BERT 使用）、SentencePiece（支持多语言、原生字节级）、Unigram LM（基于概率的语言模型方法）。它们在核心思路上与 BPE 不同，各有优劣。

## WordPiece（BERT 方案）

### 基本原理

WordPiece 与 BPE 类似，也是从字符开始逐步合并，但合并的依据不是**频率**，而是**概率增益**：

```
合并标准: 某个 pair 的联合概率 / 两个 Token 各自概率的乘积

Score(a,b) = P(ab) / (P(a) * P(b))

如果 Score > 1，说明 a 和 b 的共现不是巧合，应该合并
```

### 与 BPE 的区别

| 维度 | BPE | WordPiece |
|------|-----|-----------|
| 合并依据 | 最高频共现 | 最大概率增益 |
| 结果倾向 | 多字词更常见 | 完整词更常见 |
| 代表模型 | GPT, Llama | BERT, DistilBERT |
| 词表标记 | 无 | ## 表示非词首（如 "playing" → "play" + "##ing"）|

## SentencePiece

### 核心创新

SentencePiece 把**原始文本视为 Unicode 字符序列**，不依赖空格分词：

```
传统 Tokenizer:
  先按空格/标点分词 → 再切分 Token
  "I love NLP" → ["I", "love", "NLP"] → [ID1, ID2, ID3]

SentencePiece:
  直接处理原始文本 → 子词切分（不依赖空格）
  "I love NLP" → [ID1, ID5, ID32, ID7]
```

### 两种训练算法

SentencePiece 是一个框架，可以训练 BPE 或 Unigram：

```
SentencePiece + BPE:   类似标准 BPE 但不需要空格预分词
SentencePiece + Unigram: 基于概率的裁剪式方法（见下文）
```

### 优势

- **多语言友好**：对 CJK（中日韩）等没有空格的文字效果极好
- **前后端分离**：训练时不需要语言特定的预分词器
- **可逆**：可以完美还原原始文本

## Unigram LM

### 基本原理

与 BPE 和 WordPiece 的自底向上合并不同，Unigram 是**自顶向下裁剪**：

```
1. 从非常大的候选词表开始（所有可能的子词序列）
2. 用 EM 算法估计每个子词的概率
3. 移除使似然损失最小的子词
4. 重复直到达到目标词表大小
```

### 核心优势

- **全局最优**：BPE/WordPiece 的贪心合并可能不是全局最优，Unigram 通过概率建模更灵活
- **多候选输出**：Unigram 可以为同一个文本输出多种 Tokenization 方式（带概率），便于训练增强

## 三者对比

| 方案 | 算法类型 | 合并依据 | 代表模型 | 多语言 | 预分词需求 |
|------|---------|---------|---------|--------|-----------|
| BPE | 自底向上合并 | 频率 | GPT, Llama, Claude | 差 | 需要 |
| WordPiece | 自底向上合并 | 概率增益 | BERT, DistilBERT | 中 | 需要 |
| SentencePiece | 框架（BPE或Unigram） | 不限 | Llama 2, Gemma, T5 | 好 | 不需要 |
| Unigram LM | 自顶向下裁剪 | 概率 | XLNet, ALBERT, T5 | 好 | 不需要 |

## 工程启示

- SentencePiece + Unigram 是目前多语言场景的最佳选择
- BPE 在英语场景下仍然效果出色且实现简单
- Tokenizer 的选择通常无法在模型训练后更改——需要在训练前确定
- Agent 开发中如果涉及多语言输入（尤其是中文、日文），推荐使用 SentencePiece 类 Tokenizer 的模型
