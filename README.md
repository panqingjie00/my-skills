# My Skills

个人 Claude Code Skills 合集。

---

## 技能列表

### 1. Codex 接入 DeepSeek 指南

- **文件名**：`Codex接入DeepSeek指南.md`
- **简介**：在最新版 OpenAI Codex CLI 移除 `wire_api=chat` 支持后，使用 `deepseek-proxy` 本地代理实现 Responses API → Chat Completions API 协议翻译，让 DeepSeek 模型无缝接入 Codex，并集成 CC-Switch 一键切换。
- **适用场景**：DeepSeek 用户想让 Codex CLI 使用 DeepSeek V4 Pro 等模型。
- **前置条件**：Python 3.9+、CC-Switch、DeepSeek API Key。

**使用方法**：

```bash
# 安装并查看技能说明
claude code --skill Codex接入DeepSeek指南.md

# 或在对话中引用
/claude add panqingjie00/my-skills
```

**快速开始**：

1. `pip install deepseek-proxy`
2. 修改 `deepseek_proxy/__init__.py` 添加 `/responses` 路由（详见技能文件第 2 步）
3. 启动代理：`set DEEPSEEK_API_KEY=sk-xxx && python -m deepseek_proxy --port 8787`
4. 修改 `config.toml`：`wire_api = "responses"`，`base_url = "http://127.0.0.1:8787"`
5. `codex`

---

## 安装方式

克隆仓库到本地 `.claude/skills/` 目录：

```bash
git clone https://github.com/panqingjie00/my-skills.git ~/.claude/skills/my-skills
```

或通过 Claude Code 直接添加：

```bash
/claude add panqingjie00/my-skills
```

---

## 目录结构

```
my-skills/
├── README.md                          # 本文件
└── Codex接入DeepSeek指南.md            # Codex + DeepSeek 接入指南
```
