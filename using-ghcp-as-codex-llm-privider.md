# 在 Codex 中使用 GHCP 作为模型提供方

> 基于 Codex App 26.924 / codex-cli 0.158 验证。

## 1. 准备 Token


安装 GitHub CLI
```bash
brew install gh
```

输入以下命令，然后按照提示完成 GitHub CLI 的登录。
```bash
gh auth login
```

运行以下命令获得 GitHub OAuth Token（`gho_` 开头）：
```bash
gh auth token
```
**请妥善保存该 Token，避免泄露，后续将用于配置 Codex。**

Codex 通过环境变量读取 GitHub Token（`gho_` 开头的 OAuth Token，可用 `gh auth token` 获取），写入 `~/.codex/.env`：

```bash
echo "COPILOT_GITHUB_TOKEN=$(gh auth token)" > ~/.codex/.env
```

## 2. 配置 `~/.codex/config.toml`

顶层配置必须写在所有 `[...]` 小节**之前**：

```toml
model = "gpt-6-astra"
model_provider = "ghcp-ent"
model_reasoning_effort = "xhigh"
web_search = "live"
model_catalog_json = "/Users/<your-username>/.codex/models-override.json"   # 见第 3 步，仅 gpt-5.6 / gpt-6 系列需要

[model_providers.ghcp-ent]
name = "GHCP"
base_url = "https://api.githubcopilot.com"
env_key = "COPILOT_GITHUB_TOKEN"
wire_api = "responses"
http_headers = { "Copilot-Integration-Id" = "vscode-chat", "Editor-Version" = "vscode/1.107.0", "Editor-Plugin-Version" = "copilot-chat/0.35.0", "User-Agent" = "GithubCopilot/0.35.0", "X-GitHub-Api-Version" = "2026-06-01" }
```

## 3. 为 gpt-5.6 / gpt-6 开启 Web Search

gpt-5.6、gpt-6 系列在内置模型目录中 `use_responses_lite = true`，Codex 不会发送 `web_search` 工具（gpt-5.5 不受影响，可跳过本步）。该字段无法在 `config.toml` 中覆盖，需要自定义模型目录：

```bash
# 导出内置模型目录（用空 CODEX_HOME，避免读取本地配置）
CODEX=/Applications/ChatGPT.app/Contents/Resources/codex-cli/CodexCLI.app/Contents/MacOS/codex
T=$(mktemp -d); CODEX_HOME=$T "$CODEX" debug models > /tmp/catalog.json; rm -rf $T

# 关闭目标模型的 use_responses_lite
python3 - <<'EOF'
import json, os
d = json.load(open('/tmp/catalog.json'))
for m in d['models']:
    if m['slug'] in ('gpt-6-astra', 'gpt-6-sol', 'gpt-6-luna'):
        m['use_responses_lite'] = False
json.dump(d, open(os.path.expanduser('~/.codex/models-override.json'), 'w'), indent=2, ensure_ascii=False)
EOF
```

> 该文件会整体替换内置目录，Codex 升级后需重新执行本步。

## 4. （可选）调整上下文窗口

gpt-6 默认 272K，最大 872K：

```toml
model_context_window = 872000
```

## 5. 重启并验证

完全退出并重启 Codex App，新建对话。命令行验证：

```bash
"$CODEX" exec --ephemeral --skip-git-repo-check -s read-only \
  "Use ONLY native web search. What is the latest Node.js LTS version? If no web search tool, reply WEB_SEARCH_UNAVAILABLE."
```

输出中出现 `web search:` 即表示成功。

## 参考

- [openai/codex#35988](https://github.com/openai/codex/issues/35988)：gpt-5.6 系列 web search 不可用
- [openai/codex#31875](https://github.com/openai/codex/issues/31875)：通过自定义模型目录关闭 `use_responses_lite`
- [openai/codex#34956](https://github.com/openai/codex/issues/34956)：搜索后端与模型提供方解耦的功能请求
