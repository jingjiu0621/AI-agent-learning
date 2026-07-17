# 2.1 tokenization — Tokenizer 原理与优化

## 概念定义

Tokenizer 是 LLM 的第一层组件，负责将原始文本（人类可读的字符串）转换为模型可处理的 Token ID 序列。它本质上是一个**编码-解码系统**：编码时将文本映射到整数序列，解码时将整数序列还原为文本。Tokenization 的质量直接影响模型的理解能力、推理效率和成本。

## 核心角色

| 角色 | 说明 |
|------|------|
| 编码器 | 将文本 → Token ID 序列（预处理） |
| 解码器 | 将 Token ID 序列 → 文本（后处理） |
| 词表 | Token → ID 映射的完整字典 |

## 为什么 Tokenizer 重要

1. **Token 数量直接影响成本**：API 按 Token 计费，不同的 Tokenizer 对同一文本的 Token 计数差异可达数倍
2. **Token 切分影响理解质量**：生僻词被切碎成多个 Token 会损失语义信息
3. **词表大小影响模型性能**：词表太小（更多的 OOV）vs 词表太大（更高的计算开销）

## Tokenizer 类型演进

| 类型 | 代表 | 方法 | 特点 |
|------|------|------|------|
| Word-based | 早期 NLP | 按空格/标点切分 | 词表巨大，OOV 问题严重 |
| Character-based | — | 按字符切分 | 序列太长，丢失单词级语义 |
| Subword（BPE） | GPT 系列 | 从字符开始合并高频共现 | 平衡词表大小和序列长度 |
| Subword（WordPiece） | BERT | 类似 BPE 但基于概率 | 更倾向于完整词 |
| Subword（Unigram） | SentencePiece | 基于概率模型 | 支持多语言 |

## 内容结构

| 子主题 | 核心问题 | 难度 |
|--------|---------|------|
| 2.1.1 bpe | BPE 算法如何从字节逐步合并为子词？ | ★★★☆☆ |
| 2.1.2 wordpiece | WordPiece/SentencePiece 与 BPE 有何不同？ | ★★★☆☆ |
| 2.1.3 tiktoken | OpenAI 的 tiktoken 库如何实现超高速 Token 计数？ | ★★★☆☆ |
| 2.1.4 special-tokens | [CLS], [SEP], <|endoftext|> 这些特殊 Token 起什么作用？ | ★★☆☆☆ |
| 2.1.5 token-efficiency | 如何减少 Token 数以控制成本？ | ★★★☆☆ |
| 2.1.6 multilingual-token | 多语言 Tokenizer 面临什么独特挑战？ | ★★★☆☆ |
