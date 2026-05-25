---
name: codex-deepseek-proxy
description: 使用本地代理（deepseek-proxy）让 OpenAI Codex CLI 接入 DeepSeek 模型的完整指南。解决 wire_api 协议不兼容问题，支持 CC-Switch 一键切换。
---

# Codex 接入 DeepSeek 完整指南（Windows-Claude Code）

## 前提条件

- Python 3.9+ 已安装
- CC-Switch 已安装
- 已有 DeepSeek API Key（格式：`sk-xxxxxxxx`）

---

## 第 1 步：安装 deepseek-proxy

```cmd
pip install deepseek-proxy
```

验证安装：

```cmd
pip show deepseek-proxy
```

记下 `Location` 路径，后续补丁需要用到。

---

## 第 2 步：修复路由（必须操作）

代理默认只监听 `/v1/responses`，但新版 Codex 请求的是 `/responses`（不带 `/v1` 前缀），需要手动补丁。

编辑文件（路径根据 `pip show deepseek-proxy` 的 Location 拼接 `\deepseek_proxy\__init__.py`）：

```
{Location}\deepseek_proxy\__init__.py
```

### 补丁 1：添加 `/responses` 路由

找到第 463 行附近：

```python
# 修改前
@app.route("/v1/responses", methods=["POST"])
def responses():

# 修改后（在 def 上方加一行 @app.route）
@app.route("/v1/responses", methods=["POST"])
@app.route("/responses", methods=["POST"])
def responses():
```

### 补丁 2：添加 `/models` 和 `/models/<id>` 路由

找到第 661-666 行附近：

```python
# 修改前
@app.route("/v1/models", methods=["GET"])
def list_models():
    logger.info(f"=== GET /v1/models {dict(request.args)} ===")
    return jsonify(_EMPTY_MODELS_LIST)

@app.route("/v1/models/<model_id>", methods=["GET"])
def get_model(model_id):
    logger.info(f"=== GET /v1/models/{model_id} {dict(request.args)} ===")
    return jsonify({**_MODEL_INFO, "id": model_id})

# 修改后（各加一行不带 /v1 前缀的路由）
@app.route("/v1/models", methods=["GET"])
@app.route("/models", methods=["GET"])
def list_models():
    logger.info(f"=== GET /v1/models {dict(request.args)} ===")
    return jsonify(_EMPTY_MODELS_LIST)

@app.route("/v1/models/<model_id>", methods=["GET"])
@app.route("/models/<model_id>", methods=["GET"])
def get_model(model_id):
    logger.info(f"=== GET /v1/models/{model_id} {dict(request.args)} ===")
    return jsonify({**_MODEL_INFO, "id": model_id})
```

---

## 第 3 步：启动代理

```cmd
set DEEPSEEK_API_KEY=sk-你的DeepSeek密钥
python -m deepseek_proxy --port 8787
```

看到以下输出表示启动成功：

```
DeepSeek Proxy starting
Listening on http://127.0.0.1:8787
DEEPSEEK_API_KEY set: True
Uvicorn running on http://127.0.0.1:8787 (Press CTRL+C to quit)
```

> **保持此窗口不要关闭。** 每次开机后需要重新执行此命令。

---

## 第 4 步：配置 Codex

### 4.1 修改 `config.toml`

位置：`%USERPROFILE%\.codex\config.toml`

```toml
model_provider = "custom"
model = "deepseek-v4-pro"
disable_response_storage = true

[model_providers.custom]
name = "custom"
wire_api = "responses"
requires_openai_auth = true
base_url = "http://127.0.0.1:8787"
```

### 4.2 修改 `auth.json`

位置：`%USERPROFILE%\.codex\auth.json`

```json
{
  "OPENAI_API_KEY": "sk-你的DeepSeek密钥"
}
```

> config.toml 中的 `OPENAI_API_KEY` 和启动代理时设置的 `DEEPSEEK_API_KEY` 是同一个密钥。

---

## 第 5 步：CC-Switch 集成

为避免 CC-Switch 切换供应商时覆盖手动配置，在 CC-Switch 中添加一个自定义供应商：

| 字段        | 值                            |
|------------|-------------------------------|
| name       | `DeepSeek（本地代理）`            |
| base_url   | `http://127.0.0.1:8787`       |
| wire_api   | `responses`                   |
| model      | `deepseek-v4-pro`             |

之后在 CC-Switch 中点击该供应商即可一键切换到 DeepSeek。

---

## 第 6 步：启动 Codex

```cmd
codex
```

---

## 数据流

```
Codex CLI/Desktop
    │
    ▼
CC-Switch（供应商路由）
    │
    ▼
http://127.0.0.1:8787（deepseek-proxy 本地代理）
    │  协议翻译：OpenAI Responses API → DeepSeek Chat Completions API
    ▼
https://api.deepseek.com/v1/chat/completions
```

---

## 日常使用

| 事项           | 操作                                                        |
|---------------|-------------------------------------------------------------|
| 每次开机后       | 先执行第 3 步启动代理，再启动 Codex                               |
| API Key 轮换    | 同时更新 `auth.json` 和启动代理时的 `DEEPSEEK_API_KEY` 环境变量     |
| 代理端口被占用    | 改用 `--port 其他端口`，同时更新 `config.toml` 中的 `base_url`      |
| 需要临时停用代理   | 关闭代理窗口，`config.toml` 中改回 `base_url = "https://api.deepseek.com"` 并改 `wire_api = "chat"`（仅限旧版 Codex） |
| 更新 deepseek-proxy | `pip install --upgrade deepseek-proxy`（升级后需重新打第 2 步的补丁） |

---

## 常见问题

### Q: 404 Not Found `/responses`

代理未打第 2 步的路由补丁，请确认 `__init__.py` 中已添加 `@app.route("/responses")`。

### Q: `wire_api = "chat"` 报错

新版 Codex 已移除 `chat` 模式支持，必须使用 `wire_api = "responses"` 并配合代理翻译。

### Q: 代理启动报错 `address already in use`

端口 8787 已被占用，先关闭已运行的代理，或换一个端口。

### Q: CC-Switch 切换后配置被覆盖

确认第 5 步已在 CC-Switch 中添加了指向本地代理的自定义供应商。切换回来后选中它即可恢复。





# 本流程中 新安装的项目、库，配置、生成的文件：  

| 项目                       | 位置                                                         | 类型                        |
| -------------------------- | ------------------------------------------------------------ | --------------------------- |
| deepseek-proxy             | D:\Program Files\Python310\lib\site-packages\deepseek_proxy\ | pip 包（代码+依赖）         |
| 依赖库 (flask, uvicorn 等) | D:\Program Files\Python310\lib\site-packages\                | pip 自动安装                |
| config.toml                | C:\Users\86178\.codex\config.toml                            | 修改了 wire_api 和 base_url |
| 指南文件                   | C:\Users\86178\Desktop\123\Codex接入DeepSeek指南.md          | 你让我输出的 markdown       |



  # 如何卸载 deepseek-proxy

```js
pip uninstall deepseek-proxy -y
```



  # 如果想把依赖也一起清理（可选，注意其他程序可能用到 flask/uvicorn）
```
pip uninstall flask uvicorn requests asgiref deepseek-proxy -y
```

# 如何恢复 Codex 配置

  把 config.toml 的这两行改回原来的值即可：

  

```
wire_api = "chat"                              # 改回 chat
base_url = "https://api.deepseek.com"          # 改回 DeepSeek 直连
```

  或者直接在 CC-Switch 里切回其他供应商，它会自动覆盖这两个配置项。
