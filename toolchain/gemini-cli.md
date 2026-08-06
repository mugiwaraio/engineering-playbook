# Gemini CLI

Google 官方开源终端 AI agent。

## 安装

```bash
# 免安装试用
npx @google/gemini-cli

# npm 全局安装
npm install -g @google/gemini-cli

# Homebrew（macOS / Linux）
brew install gemini-cli
```

## 验证

```bash
gemini --version
```

## 认证

运行 `gemini`，三种方式：

1. **Google 账号 OAuth**（个人开发推荐）：选择 "Sign in with Google"，个人账号免费额度 60 次/分钟、1000 次/天。
2. **Gemini API key**（从 [AI Studio](https://aistudio.google.com/apikey) 获取）：

   ```bash
   export GEMINI_API_KEY="YOUR_API_KEY"
   gemini
   ```

3. **Vertex AI**：

   ```bash
   export GOOGLE_API_KEY="YOUR_API_KEY"
   export GOOGLE_GENAI_USE_VERTEXAI=true
   gemini
   ```

## 更新

三个发布渠道，按需选择：

```bash
npm install -g @google/gemini-cli@latest    # stable，每周二发布
npm install -g @google/gemini-cli@preview   # preview，每周二发布
npm install -g @google/gemini-cli@nightly   # nightly，每日发布
```

## 卸载

```bash
npm uninstall -g @google/gemini-cli
brew uninstall gemini-cli    # Homebrew 安装时
```

---

来源：[google-gemini/gemini-cli GitHub README](https://github.com/google-gemini/gemini-cli)。核实日期：2026-08-06。
