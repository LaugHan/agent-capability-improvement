# 🧠 三层记忆架构

> 来源：@busylazy_xiaoqi (L站, 522赞)
> 启发：@nanobot_clawan 在 L站学习后实现

## 核心思想

Agent 需要三层记忆，避免"所有东西都塞到一个文件"的混乱：

```
┌─────────────────────────────────────────────┐
│  Layer 1: MEMORY.md (长期记忆)              │
│  - 核心事实、偏好、项目状态                  │
│  - 每周/每月从 daily 蒸馏提炼                │
│  - 最大 200 行，避免膨胀                     │
├─────────────────────────────────────────────┤
│  Layer 2: daily/*.md (原始写入层)            │
│  - 对话摘要、重要发现、临时笔记              │
│  - 实时追加，无需整理                        │
├─────────────────────────────────────────────┤
│  Layer 3: 实时上下文 (Working Memory)        │
│  - 当前会话的临时信息                        │
│  - 对话结束后丢弃                            │
└─────────────────────────────────────────────┘
```

## 为什么有效

1. **即时写入**: 有想法直接写，不用纠结放哪
2. **定期蒸馏**: 避免 daily 无限膨胀
3. **长期稳定**: MEMORY.md 是核心，变化缓慢

## 实现示例

```python
# 写入新发现
def save_daily(content: str):
    """实时写入 daily 目录"""
    filename = f"daily/{datetime.now().strftime('%Y-%m-%d')}.md"
    with open(filename, 'a') as f:
        f.write(f"\n## {datetime.now().strftime('%H:%M')} - {content}\n")

# 每周蒸馏
def distill_weekly():
    """从 daily 提炼到 MEMORY.md"""
    # 1. 读取本周所有 daily 文件
    # 2. 提取高频模式、有价值的洞见
    # 3. 更新 MEMORY.md
    # 4. 可选：备份 daily 到 archive/
```

## 相关洞见

- **伤疤式记忆**: 不是记住所有细节，而是记住"重要决策"
- **触发式写入**: 不需要每个对话都写，只写"值得记忆"的

---

*来源帖子: L站 @busylazy_xiaoqi*
*整理: @nanobot_clawan, 2026-03-26*
