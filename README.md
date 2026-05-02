# claude-code-gemini-adapter
fix how to use gemini in claude code
# 解决 Gemini 无法接入 Claude 的问题
## 【避坑指南】Gemini 接入 Claude Code：解决 Gemini API 协议适配全记录

### 前言
最近在研究 Claude Code（Anthropic 官方推出的 CLI 工具），因为开了 Gemini 会员，所以打算利用 Google Gemini 3.1 Pro/Flash 来驱动 Claude。

本以为只需在 CC Switch 里面简单修改 BASE_URL 就 OK 了，没想到踩了一堆的坑，在 404、429、400 的错误循环中反复横跳。

如果你安装好了 Claude Code，想接入 Gemini 的 API，那会遇到一堆 bug，这篇文章很适合你。

---

## 一、为什么不能“直连”？（技术本质）
很多小伙伴（包括最开始的我）认为改个 `ANTHROPIC_BASE_URL` 就行了。但实际上，这是典型的**接口协议不兼容（Protocol Incompatibility）**：
- Claude Code 讲的是“Anthropic 语”：它发出的请求结构（Payload）是严格遵循 `messages` 格式的。
- Gemini 讲的是“Google 语”：它预期的字段是 `contents` 和 `parts`。

当你把地址强行指向 Google，就像是把圆形的插头往方形插座里捅，服务器看不懂你的诉求，自然会返回 **404 (Not Found)**。

---

## 二、三个大坑
### 坑 1：消失的 3.1-preview
如果你直接在 CC Switch 里面配置 Gemini：
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/7e6c3ca48ba6457e9d25ee5c7b024c3f.png#pic_center)
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/271aadbc77494a3eb71934f4e7057d19.png#pic_center)
- BASE_URL：`https://generativelanguage.googleapis.com`
- 模型名称：`gemini-3.1-pro` 和 `gemini-3-flash`

**问题**：Gemini 目前没有这两个模型，真实模型名称是 `gemini-3.1-pro-preview` 和 `gemini-3-flash-preview`。使用默认名称会直接报 404。

**对策**：模型名称改成 `gemini-3.1-pro-preview` 和 `gemini-3-flash-preview`。

### 坑 2：429 Rate Limit（频率限制）
Google 免费版对 Pro 模型的限制极其苛刻（每分钟 2-5 次）。

**对策**：测试阶段全部使用 `gemini-3-flash-preview`，配额更慷慨、响应更快，适合 CLI 环境。

### 坑 3：终端运行 Claude 仍报错
解决前两个坑，CC Switch 测速成功，但终端启动 Claude 报错：
```
There's an issue with the selected model (gemini-3-flash-preview). It may not exist or you may not have access to it. Run /model to pick a different model.
```

**原因**：Claude 和 Gemini 数据接口协议不兼容。
**对策**：安装 LiteLLM。

### 坑 4：400 参数冗余错误
装好 LiteLLM、配置 `GEMINI_API_KEY` 并启动后，重启 Claude 可能出现：
`gemini does not support parameters: ['context_management']`

启动命令：
```bash
litellm --model gemini/gemini-3-flash-preview --port 4000
```

报错详情：
```
API Error: 400 {"error":{"message":"litellm.UnsupportedParamsError: gemini does not support parameters: ['context_management'], for model=gemini-3-flash-preview. To drop these, set `litellm.drop_params=True` or for proxy: litellm_settings: drop_params: true. If you want to use these params dynamically send allowed_openai_params=['context_management'] in your request.","type":"None","param":null,"code":"400"}}
```

**原因**：Claude Code 会发送 Gemini 不支持的额外参数。
**终极解法**：启动 LiteLLM 时强制开启参数过滤。

```bash
litellm --model gemini/gemini-3-flash-preview --port 4000 --drop_params
```

---

## 三、核心解决步骤：LiteLLM 搭配 CC Switch
需要协议适配器（Adapter）实现请求格式转换，LiteLLM 是轻量高效的选择。

### 1. 环境准备（使用极速工具 uv）
```bash
# 1）初始化 uv 项目
uv init

# 2）添加依赖
uv add requests

# 安装 litellm 代理工具
uv tool install "litellm[proxy]"
```

### 2. 环境变量持久化
写入 shell 配置文件（如 ~/.zshrc）：
```bash
# 添加 Gemini API Key
echo 'export GEMINI_API_KEY="你的真实_Google_API_KEY"' >> ~/.zshrc

# 生效配置
source ~/.zshrc

# 检查是否生效
echo $GEMINI_API_KEY
```

### 3. 启动 LiteLLM
```bash
litellm --model gemini/gemini-3-flash-preview --port 4000 --drop_params
```

---

## 四、最终配置（CCSwitch）
在 CCSwitch 中指向本地适配服务，然后在终端启动claude就可以正常使用gemini模型了。

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://127.0.0.1:4000",
    "ANTHROPIC_API_KEY": "anything",
    "ANTHROPIC_MODEL": "gemini-3-flash-preview",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "gemini-3-flash-preview",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "gemini-3-flash-preview",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "gemini-3-flash-preview"
  },
  "includeCoAuthoredBy": false
}
```
