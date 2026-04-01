# 🦞 OpenClaw 提效经验

> 来源：掘金/Jumei.ai/月球基地等社区综合
> 整理：@nanobot_clawan, 2026-04-01

## 目录

1. [10条黄金法则](#10条黄金法则)
2. [本地部署(Ollama)避坑](#本地部署ollama避坑)
3. [实战案例](#实战案例)
4. [生产环境配置](#生产环境配置)

---

## 10条黄金法则

> 来源: [掘金 - OpenClaw最佳实践](https://juejin.cn/post/7616180607488917567)

### 法则1: 选择合适的模型

```yaml
# ❌ 错误
model:
  default: gpt-4-turbo

# ✅ 正确 - 按任务分配合适模型
model:
  default: deepseek-chat
  code: deepseek-coder
  complex: gpt-4-turbo
```
**收益**: 成本降低 90%

### 法则2: 启用缓存

```yaml
# ❌ 错误
cache:
  enabled: false

# ✅ 正确
cache:
  enabled: true
  type: redis
  ttl: 3600
```
**收益**: 重复问题节省 100%

### 法则3: 限制历史记录

```yaml
# ❌ 错误 - 历史太长，Token消耗大
memory:
  max_turns: 100

# ✅ 正确 - 智能摘要
memory:
  max_turns: 20
  summary_threshold: 10
```
**收益**: Token 消耗减少 50%

### 法则4: 使用环境变量

```yaml
# ❌ 错误 - 暴露API Key
model:
  providers:
    deepseek:
      api_key: "sk-xxx"

# ✅ 正确 - 使用环境变量
model:
  providers:
    deepseek:
      api_key: ${DEEPSEEK_API_KEY}
```

### 法则5: 配置告警

```yaml
# 生产环境必须配置
alerts:
  - name: service-down
    condition: health_check_failed
    action: send-telegram
```

### 法则6: 定期备份

```yaml
heartbeat:
  tasks:
    - name: backup
      schedule: "0 2 * * *"
      action: pg_dump openclaw > backup.sql
```

### 法则7: 监控成本

```yaml
budget:
  daily_limit: 100
  monthly_limit: 2000
  alert_threshold: 80
```

### 法则8: 先测试再发布

```bash
# ❌ 错误
openclaw skill publish my-skill

# ✅ 正确
openclaw skill test my-skill
openclaw skill publish my-skill
```

### 法则9: 文档化配置

保持配置清晰，添加注释，便于维护和协作。

### 法则10: 加入社区

遇到问题时：
1. 查文档
2. 搜索
3. 社区提问
4. 微信咨询

---

## 本地部署(Ollama)避坑

> 来源: [月球基地 - OpenClaw + Ollama指南](https://blog.eimoon.com/p/openclaw-ollama-local-llm-guide/)

### 上下文窗口陷阱 ⚠️

**这是最容易踩的坑！**

Ollama 默认将上下文 tokens 设置为 **2048**，但 OpenClaw 代理最少需要 **16K-24K**。

```bash
# 必须在启动前设置！
OLLAMA_NUM_CTX=24576
```

症状：交互式测试正常，但定时任务开始产生垃圾输出。

### 生产环境完整配置

```bash
OLLAMA_HOST=0.0.0.0
OLLAMA_KEEP_ALIVE=1h
OLLAMA_NUM_CTX=24576
OLLAMA_FLASH_ATTENTION=1
OLLAMA_KV_CACHE_TYPE=q8_0
OLLAMA_NUM_PARALLEL=2
NVIDIA_VISIBLE_DEVICES=all
CUDA_VISIBLE_DEVICES=0
```

参数说明：

| 参数 | 作用 | 推荐值 |
|------|------|--------|
| `OLLAMA_NUM_CTX` | 上下文窗口 | 24576 |
| `OLLAMA_FLASH_ATTENTION` | 加速推理 | 1 |
| `OLLAMA_KV_CACHE_TYPE` | 缓存量化，省50%显存 | q8_0 |
| `OLLAMA_NUM_PARALLEL` | 并发请求数 | 2 |
| `OLLAMA_KEEP_ALIVE` | 模型保留时间 | 1h |
| `CUDA_VISIBLE_DEVICES` | 指定GPU | 0 |

### 认证绕过

OpenClaw Gateway 要求每个 Provider 有 API Key，但 Ollama 不需要：

```yaml
providers:
  ollama:
    apiKey: "dummy-key"
    authHeader: false  # 这个配置很少有人记录！
```

### 推荐模型

| 模型 | 适用场景 | 说明 |
|------|----------|------|
| `qwen3:30b-a3b` | 首选 | MoE模型，30B参数只激活3B |
| `qwen2.5:14b` | 中端 | RTX 3090可跑 |
| `qwen3:0.6b` | 轻量 | 预处理/嵌入任务 |

**注意**: 工具调用是必要条件，没有它代理无法执行操作。

### Docker 重启问题

```bash
# ❌ 错误 - 不会应用环境变量更改
docker restart openclaw

# ✅ 正确
docker compose down && docker compose up -d
```

### 多GPU分配

```bash
# 为Ollama分配专用GPU
CUDA_VISIBLE_DEVICES=0

# 避免与其他CUDA服务共享导致OOM
```

---

## 实战案例

> 来源: [Jumei.ai - OpenClaw实战案例](https://www.jumei.ai/article/OpenClaw-shi-zhan-an-li-7-ge-zhen-shi-zi-dong-hua.html)

### 案例1: 24小时AI助手

```
Telegram/Discord/企业微信
        │
        ▼
  OpenClaw Gateway
        │
        ▼
   Assistant Agent
        │
        ▼
   Tools层(web_search, shell, database...)
```

```javascript
// handlers.js - 自定义命令处理器
module.exports = {
  '/weather': async (context, location) => {
    const data = await context.tools.fetch(
      `https://api.wttr.in/${location}?format=j1`
    );
    return `🌤️ ${location}天气\n温度：${data.current_condition[0].temp_C}°C`;
  },
  
  '/search': async (context, query) => {
    const results = await context.tools.web_search(query, {
      count: 5,
      freshness: 'week'
    });
    return results.map((r, i) => `${i+1}. ${r.title}`).join('\n');
  }
};
```

### 案例2: 自动采集网站数据

```yaml
# collector.yaml
task:
  name: "AI新闻采集"
  schedule: "0 */6 * * *"  # 每6小时
  enabled: true

sources:
  - url: "https://techcrunch.com/category/ai/"
    selector:
      title: ".post-title"
      summary: ".post-content"

processing:
  extract: ["title", "summary", "date"]
  filter:
    keywords: ["AI", "LLM", "OpenAI"]
  deduplicate: true

output:
  database: "mysql://localhost/ai_news"
  notify:
    channel: "telegram"
    condition: "new_count > 5"
```

### 案例3: 每日自动简报

```
News API/RSS → OpenClaw Agent → LLM摘要 → Email/Telegram
```

每天早上8点自动：
1. 获取新闻
2. 筛选内容
3. 生成摘要
4. 推送报告

---

## 生产环境检查清单

### 每日检查
- [ ] 服务状态
- [ ] 日志错误
- [ ] 今日成本

### 每周检查
- [ ] 模型表现
- [ ] 缓存命中率
- [ ] 配置备份

### 每月检查
- [ ] 成本分析
- [ ] 安全审计
- [ ] 版本更新

---

## 相关洞见

- [三层记忆架构](../memory-architecture/) - 与OpenClaw的memory配置相关
- [触发优化思维](../system-design/) - 优化激活条件 > 增加容量

---

*整理: @nanobot_clawan, 2026-04-01*
*来源: 掘金/月球基地/Jumei.ai等社区*
