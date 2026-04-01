# Claude Code 记忆系统模式

来源: Developer Toolkit
作者: -
链接: https://developertoolkit.ai/en/shared-workflows/context-management/memory-patterns/
日期: 2026-04-01

## 核心

Claude Code 有六层记忆系统，理解这个层次才能正确使用：

```
Level 1: System CLAUDE.md     → 所有用户，IT管理
Level 2: Project ./CLAUDE.md   → 团队，通过git管理
Level 3: Project rules         → .claude/rules/*.md，团队
Level 4: User memory            → ~/.claude/CLAUDE.md，个人所有项目
Level 5: Local memory           → ./CLAUDE.local.md，个人此项目
Level 6: Auto memory           → Claude自动保存（亮点！）
```

### Auto Memory（亮点）

Claude 会**自动保存**有用的上下文：
- 项目模式
- 调试洞见
- 架构笔记
- 用户偏好

不需要用户手动触发，Claude 发现重要信息会自动记住。

**规则**：MEMORY.md 前200行会加载到每个会话，详细笔记按需读取。

### 应该记住什么

- 构建、测试、部署命令
- 非显而易见的约束（环境要求、服务依赖）
- 架构决策及理由
- 花时间诊断出来的错误模式
- 与语言默认不同的团队惯例

### 不应该记住什么

- 临时 workaround（很快会删除的）
- 所有细节（太大会被忽略）
- 过时的信息

**关键**：记忆太大 = 被忽略。和500行的CLAUDE.md一样，没人看。

## 启发

1. **分层记忆**：不同层有不同的生命周期和管理方式
2. **自动 > 手动**：Claude 自动记住比用户手动记更靠谱
3. **简洁 > 详细**：少而精的规则比多而全的文档更有用
4. **按需加载**：不需要每次都加载所有记忆

## 实际用法

1. **写 CLAUDE.md**：保持简洁，前200行最重要
2. **让 Claude 自动记**：发现重要信息时提示 Claude 保存
3. **定期清理**：删除过时信息，保持记忆新鲜
4. **分层管理**：个人偏好放个人层，团队规范放团队层

## 参考

原文: https://developertoolkit.ai/en/shared-workflows/context-management/memory-patterns/
