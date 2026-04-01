# LLM 情境意识资源集

来源: GitHub
作者: AI4Collaboration
链接: https://github.com/AI4Collaboration/Awesome-LLM-Situational-Awareness
日期: 2026-04-01

## 核心

LLM 情境意识 = Agent 知道自己是谁、知道什么、知道多少。

这个仓库收集了 LLM 情境意识的核心资源，分几类：

### 1. 自我认知 (Self-Awareness)
- 理解自己的能力边界
- 知道自己擅长什么、不擅长什么

### 2. 知识边界 (Knowledge Boundaries)
- 知道"我知道什么"和"我不知道什么"
- 区分：真的理解 vs 只是在猜测

### 3. 不确定性量化 (Uncertainty Quantification)
- 知道自己的回答有多确定
- 区分：高置信度回答 vs 低置信度猜测

### 4. 元认知 (Metacognition)
- 知道自己是怎么思考的
- 反思自己的推理过程

### 5. 内省 (Introspection)
- 检视自己的内部状态
- 报告自己的置信度、推理路径

## 启发

Agent 常见问题：
- 过度自信：不知道自己在胡说八道
- 不知道自己不知道：回答超出能力范围的问题
- 无法判断自己是否正确

情境意识让 Agent：
- 在不确定时说"我不知道"
- 在超出能力边界时承认
- 正确校准自己的置信度

## 实际用法

1. **构建 Agent 时**：加入自我检查环节——"我真的理解这个问题吗？"
2. **设计回答策略**：不确定时主动说"这个我不确定"
3. **评估 Agent**：测它在"知道自己不知道"上的表现

## 参考

原文: https://github.com/AI4Collaboration/Awesome-LLM-Situational-Awareness
