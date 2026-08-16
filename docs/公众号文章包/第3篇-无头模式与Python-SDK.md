# 把 AI Agent 变成自动化流水线：DeepSeek Harness 无头模式与 Python SDK

> 系列第 3 篇 · 首发于本公众号 · 2026 年 8 月

Web UI 适合人机交互，但真正值钱的是让 Agent 可以被脚本和 CI 调用。这一篇讲两条路：一条命令跑一次性任务的无头模式，和在 Python 代码里编排任务的 SDK。

Web UI 适合人机交互。但真正让 dsh 值钱的，是它**可以被脚本和 CI 调用**。这一章讲 `dsh` 命令本身。

## 4.1 命令语法：先搞清"谁的参数"

`dsh` 是启动器，它只解析自己的 flag，把剩下的参数交给 profile 里的应用插件解析。一句话：**launcher 的参数在前，应用的参数在后，遇到第一个启动器不认识的部分，就开始算应用的。**

```sh
dsh --profile web --port 8080       # --port 属于 web 应用
dsh --profile headless "run the tests"
dsh --profile web --help            # 打印 web 应用帮助
dsh --help                          # 打印启动器自己的帮助
```

## 4.2 入口模式一览

| 命令 | 作用 |
|---|---|
| `dsh web` | 启动 Web UI（`--profile web` 的别名） |
| `dsh --profile headless "任务"` | 无头模式：跑一个一次性会话，打印最终回复后退出 |
| `dsh --profile <name>` | 启动任意命名 profile |
| `dsh plugin --profile <name> <pnpm 参数...>` | 管理 profile 的插件（转发给 pnpm） |
| `dsh --profile web --dump-config` | 打印组合后的配置树（不启动） |
| `dsh --dump-default-config` | 打印默认配置 |

## 4.3 无头模式：一行命令跑一个任务

无头模式是脚本化的核心。它接受一个任务文本，创建并持久化一个全新会话，跑完后打印 agent 的最终文本回复并退出：

```sh
dsh --profile headless "fix the failing test in this workspace"
```

注意：

- 调用目录是默认工作区根；
- `web` 和 `headless` 两个 profile 首次使用会自动初始化；
- 会话是持久化的——跑过的任务会留下日志，这对批量任务和审计非常有价值。

## 4.4 用 `--patch` 加载你自己的配置

这是开发插件的日常命令。你写了一个插件和对应的 patch 文件，不想装进 profile，只想临时试一下：

```sh
dsh web --patch ./scratch-plugin/cordis.yml
```

多个 patch 可以叠加，按参数顺序应用。用 `--dump-config` 可以先看组合结果再启动。

## 4.5 `dsh plugin`：管理 Profile 的插件

前面说过，profile 的 manifest 不需要手写，`dsh plugin` 帮你维护：

```sh
dsh plugin --profile demo add ./hello-plugin          # 安装本地插件
dsh plugin --profile demo add github:you/hello-plugin # 从 GitHub 安装
dsh plugin --profile demo remove dsh-hello-plugin     # 移除
```

`dsh plugin` 本质上是在 profile 目录里转发 pnpm 命令，所以 `add`、`remove`、`install` 等 pnpm 子命令都可用。注意它只管理 profile 的依赖和 bundle 层，**不会动你的全局环境**。

## 4.6 实际场景：把它接进 CI

无头模式 + 退出码 + 会话日志，这三样组合起来就是一个完整的 agent 流水线：

```sh
# 示例：让 dsh 在 CI 里修测试并留档
DEEPSEEK_API_KEY=${{ secrets.DEEPSEEK_API_KEY }} \
  dsh --profile headless "inspect this repo, run the tests, fix failures, report what you changed"
```

进阶玩法：

- 用 `--patch` 按环境切换配置（比如 CI 里换更严格的沙箱策略）；
- 把会话日志目录接进你的日志系统；
- 用 Python SDK（下一章）在代码里编排多任务。

## 4.7 本章小结

- `dsh` 是启动器，先写 launcher flag，再写应用参数；
- `dsh --profile headless "任务"` 是无头执行的核心命令；
- `--patch` 让你临时叠加配置，`--dump-config` 让你预览配置树；
- `dsh plugin` 管理 profile 的插件，底层转发 pnpm；
- 无头模式天然适合 CI 和批量自动化。

下一章：Python SDK——不通过命令行，直接在程序里驱动 Harness。

命令行适合"跑一次"，Python SDK 适合"在程序里编排"。它让你在自己的 Python 代码里创建 harness、跑任务、拿结果，就像调用一个普通库。

## 5.1 环境要求（先看这条！）

- Python 3.10 或更高；
- Git；
- **Linux x64 / Linux arm64，或 macOS 14+（arm64）**；
- 一个 DeepSeek 兼容的 API 端点与凭据；
- 一个 agent 可以修改的隔离工作区。

**特别注意：官方明确说明 Python SDK 暂不支持 Windows agent**——因为底层持久化 PTY 后端需要 POSIX 终端环境。Windows 用户请用 WSL、Linux 服务器或容器来跑 SDK 示例。

## 5.2 安装

克隆仓库（拿内置示例），建虚拟环境，安装 SDK：

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
python -m venv .venv
. .venv/bin/activate
python -m pip install deepseek-harness-sdk
```

好消息：**安装后的运行时不需要系统再装 Node.js**，SDK 自带同版本内置运行时。

## 5.3 设置环境变量

```sh
export DEEPSEEK_API_KEY=sk-your-key-here
# 如果走 OpenAI 兼容代理，而不是默认官方端点：
# export DEEPSEEK_BASE_URL=http://127.0.0.1:8000/v1
# export DSH_MODEL=deepseek-v4-flash
# export DSH_SYSTEM_PROMPT='You are a helpful software engineer assistant.'
```

如果模型不是默认 DeepSeek 官方端点提供的，务必设置 `DEEPSEEK_BASE_URL`。

## 5.4 跑官方内置示例

仓库自带的 `examples/jsonrpc-agent/minimal.py` 是对 SDK 的轻量包装，针对隔离的工作区和会话目录跑一个任务：

```sh
python examples/jsonrpc-agent/minimal.py \
  --workspace /absolute/path/to/workspace \
  --session-root /absolute/path/to/sessions \
  --session-id example-001 \
  "Inspect the repository and fix the failing tests."
```

脚本会打印 agent 的最终回复。同时，会话目录会收到 JSONL 格式的日志——里面完整记录了组装后的模型请求和工具调用。这对调试和审计非常有用。

## 5.5 在自己的代码里用 SDK

```python
from pathlib import Path

from deepseek_harness import DeepSeekHarness

config = Path("examples/jsonrpc-agent/minimal.cordis.yml").resolve()
workspace = Path("/absolute/path/to/workspace").resolve()
sessions = Path("/absolute/path/to/sessions").resolve()

with DeepSeekHarness(
    provider="deepseek-official",
    model="deepseek-v4-flash",
    max_tokens=49_152,
    cwd=str(workspace),
    session_root=str(sessions),
    cordis=str(config),
) as harness:
    result = harness.run(
        "Inspect the repository and fix the failing tests.",
        session_id="example-001",
    )

print(result.final_response)
```

几个关键点：

- `DeepSeekHarness` 会**延迟启动内置运行时并持续复用**，直到退出上下文管理器；
- 复用同一个 harness 和 session id，会**保留该会话拥有的 Bash 进程**（工作目录、导出的变量、shell 函数都在）；
- **独立任务请用新的 session id**；只有确定要延续同一段对话和 shell 状态时，才复用原有 id；
- `cwd` 决定 agent 可访问的工作区，`session_root` 决定会话日志存哪。

## 5.6 这个示例组合的"配方"

看明白 `minimal` 组合里有什么，你就知道"最小 harness"长什么样：

| 属性 | 值 |
|---|---|
| 系统提示词 | `DSH_SYSTEM_PROMPT`，缺省为 "You are a helpful software engineer assistant." |
| 模型 | `--model` → `DSH_MODEL` → 默认 `deepseek-v4-flash` |
| 面向模型的工具 | 仅持久 `bash` 与 `str_replace_editor` |
| Bash 超时 | 300 秒 |
| 编辑器输出上限 | 16,000 字符 |
| 上下文压缩 | 关闭 |
| 文件系统 | 裸本地后端（编辑器用绝对路径） |
| 会话持久化 | `session_root` 下未压缩的 JSONL |
| 沙箱 | `danger-full-access`——只能在可丢弃的 checkout 或容器里跑 |

注意最后一行的警告：这个最小组合**没有任何沙箱限制**，Bash 和编辑器能改运行时进程可见的任何路径。生产环境请换上更严格的策略。

## 5.7 一个批量任务小例子

把上面的模式套进循环，就是一个最简单的批量自动化：

```python
from pathlib import Path
from deepseek_harness import DeepSeekHarness

tasks = {
    "repo-a": "Run the test suite and fix failures.",
    "repo-b": "Find TODO comments and turn them into issues.",
    "repo-c": "Update the dependency versions in package.json.",
}

with DeepSeekHarness(
    provider="deepseek-official",
    model="deepseek-v4-flash",
    cwd="/work",
    session_root="/sessions",
    cordis=Path("minimal.cordis.yml").resolve(),
) as harness:
    for repo, task in tasks.items():
        result = harness.run(task, session_id=repo)
        print(f"--- {repo}: {result.final_response[:200]}")
```

每个 repo 用独立 session id，彼此状态互不污染；`/sessions` 下留下每次任务的完整 JSONL 日志，方便事后排查。

## 5.8 本章小结

- Python SDK：`pip install deepseek-harness-sdk`，自带运行时，Python 3.10+；
- 平台注意：官方示例暂不支持 Windows agent，用 WSL/容器/Linux；
- `DeepSeekHarness` 上下文管理器 + `harness.run(task, session_id)` 是最小 API；
- session id 是状态边界：独立任务用新 id，延续对话才复用；
- 最小组合默认 `danger-full-access`，批量任务务必用隔离环境。

下一章进入重头戏：插件开发。

---

到这一步，你已经能让 Agent 批量修测试、升级依赖、写报告，而且每次任务都有完整 JSONL 日志可审计。

下一篇是重头戏：手把手写第一个插件，给 Agent 加一个它原本没有的能力。

全本《DeepSeek Harness 实战手册》微信读书搜索本公众号即可阅读；回复“手册”获取网页版全文。转载随意，注明出处即可。
