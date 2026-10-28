保存 FlagScale-Agent 的配置加载修复、OpenAI Responses 协议适配，以及可直接安装的完整迁移包，方便在新服务器上重新部署。

本仓库基于 [flagos-ai/FlagScale-Agent](https://github.com/flagos-ai/FlagScale-Agent) 的提交：

```text
4dff90ce84a7e8cfaac1af9a83f558baacc28d51
```

当前仓库以迁移包和补丁形式保存修复，项目源码在压缩包内。直接克隆官方仓库不会自动包含这里的改动。

## 下载

- [完整迁移包](migration-20261028/flagscale-agent-migration.tar.gz)
- [直接下载迁移包](https://github.com/zz774ever/flagscale-agent-debug/raw/refs/heads/main/migration-20261028/flagscale-agent-migration.tar.gz)
- [完整迁移与配置说明](migration-20261028/迁移与配置说明.md)
- [独立修复补丁](migration-20261028/flagscale-agent-responses.patch)
- [配置模板](migration-20261028/agent.yaml.example)
- [SHA-256 校验文件](migration-20261028/flagscale-agent-migration.tar.gz.sha256)

Windows 用户可以点击“直接下载迁移包”保存到电脑；下面的安装命令在准备运行 Agent 的 Linux 服务器终端执行。

## 在新服务器上安装

### 1. 下载、校验并安装

在新服务器终端依次执行：

```bash
cd ~
curl --fail --location --output flagscale-agent-migration.tar.gz https://github.com/zz774ever/flagscale-agent-debug/raw/refs/heads/main/migration-20261028/flagscale-agent-migration.tar.gz
curl --fail --location --output flagscale-agent-migration.tar.gz.sha256 https://github.com/zz774ever/flagscale-agent-debug/raw/refs/heads/main/migration-20261028/flagscale-agent-migration.tar.gz.sha256
sha256sum -c flagscale-agent-migration.tar.gz.sha256
tar -xzf flagscale-agent-migration.tar.gz
cd ~/flagscale-agent-migration/FlagScale-Agent
pip install -e .
```

校验应显示 `flagscale-agent-migration.tar.gz: OK`。某一步报错时，先解决该错误再继续执行后面的步骤。

如果压缩包已经放在服务器主目录，可以跳过下载步骤，直接解压和安装。迁移包包含完整修复后的源码，无需再次克隆上游仓库或叠加旧补丁。

`pip install -e .` 使用可编辑安装，安装后需要保留源码目录。如果移动了目录，在新位置重新安装。

### 2. 检查新参数

```bash
flagscale-agent --help
```

确认帮助中有 `--openai-api` 参数。如果没有，当前运行的程序尚未加载到本仓库保存的修复版本。

### 3. 创建配置文件

以下步骤使用默认配置目录 `~/.flagscale`：

```bash
unset FLAGSCALE_AGENT_CONFIG
unset FLAGSCALE_HOME
mkdir -p ~/.flagscale
```

如果已经有配置文件，先备份，再复制模板：

```bash
if [ -f ~/.flagscale/agent.yaml ]; then
    cp -p ~/.flagscale/agent.yaml ~/.flagscale/agent.yaml.before-responses
fi
cp ~/flagscale-agent-migration/agent.yaml.example ~/.flagscale/agent.yaml
chmod 600 ~/.flagscale/agent.yaml
nano ~/.flagscale/agent.yaml
```

将模板中的 `REPLACE_WITH_YOUR_API_KEY` 替换为自己的真实密钥。完整配置示例：

```yaml
provider: openai
model: gpt-6.1-sol
api_key: "替换为你的真实密钥"
base_url: https://api.codexzh.com/v1
openai_api: responses

max_iterations: 200
max_continuations: 200
max_context_tokens: 0
max_output_tokens: 8192
thinking_budget: 0
```

这段 YAML 要保存到文件中，不是作为命令输入终端。使用 nano 时，按 `Ctrl+O`、回车保存，再按 `Ctrl+X` 退出；也可以使用其他文本编辑器。

`model` 和 `base_url` 来自此次部署使用的配置。更换服务商时，填写其实际提供的模型名称和接口地址。`provider: openai` 表示使用 OpenAI SDK 适配器，请求实际发送到 `base_url` 指定的服务。

地址写到 `/v1` 即可，程序会追加 `/responses`。不要填写成 `/v1/responses/responses`。

仓库和迁移包使用密钥占位符，不包含部署时的真实密钥。填写真实密钥后的 `agent.yaml` 不要提交到公开仓库。

### 4. 测试连接与工具调用

先测试文本响应：

```bash
flagscale-agent --config ~/.flagscale/agent.yaml "只回复 OK"
```

再测试工具调用和结果回传：

```bash
flagscale-agent --config ~/.flagscale/agent.yaml "使用 shell 工具运行 uname -s，并告诉我输出"
```

第二条用于检查模型调用工具、执行命令、回传结果的完整过程。原项目的 Guard 仍会参与工具检查。

### 5. 开始对话

```bash
flagscale-agent --config ~/.flagscale/agent.yaml
```

配置文件已指定提供商和协议，以后启动时执行这一条即可。也可以显式指定：

```bash
flagscale-agent --config ~/.flagscale/agent.yaml --provider openai --openai-api=responses
```

## 本次修复解决了什么

### 配置被启动参数覆盖

原代码可能用命令行选项的默认值覆盖 YAML 中的提供商，或在指定 `--provider openai` 时重新读取默认模型、密钥和地址，使已有配置失效。

修复后，先合并 YAML 与显式命令行参数，再解析缺失字段的环境变量和默认值。未显式输入的启动参数会保留文件中的配置。明确切换提供商时，重新解析对应提供商的配置。

### HTTP 400：模型不支持 Chat Completions

旧工具补丁把工具定义改成了 Responses 的扁平结构，但实际调用仍然是 `/chat/completions`，消息和响应解析也没有完成协议转换。

本次补齐了：

- `/responses` 请求，以及 `input`、`max_output_tokens` 等请求字段。
- Responses 工具定义、函数调用和工具结果的转换。
- 普通响应和流式事件解析，包括多个工具的参数归属。
- 多轮请求所需的调用 ID 和推理状态保存。
- 流错误、未完成响应和连接中断的处理。
- 对原有 Chat Completions 协议的保留。

### 协议选项

| YAML 配置 | 行为 |
| --- | --- |
| `openai_api: responses` | 直接使用 Responses；当前部署使用此配置。 |
| `openai_api: chat_completions` | 固定使用 Chat Completions。 |
| `openai_api: auto` | 默认先尝试 Chat Completions；收到明确的模型或接口协议不支持的 HTTP 400 时切换 Responses，并在本次运行中继续使用。 |

命令行写 `--openai-api`，YAML 写 `openai_api`。命令行参数需要带值，例如 `--openai-api=responses`。

## 旧补丁和备份

旧补丁中“展平工具定义”的功能已经整合进 Responses 分支。新机器使用完整迁移包后，不需要再次应用旧补丁。

迁移包内保留 `flagscale_agent/react/providers/openai_provider.py.bak`。经比较，这个文件是基准提交中的上游原版备份，不是旧补丁修改后的副本。

修复没有移除或放宽原项目的 Guard。

## 修改文件

| 文件 | 用途 |
| --- | --- |
| `flagscale_agent/cli.py` | 修复默认参数覆盖配置，增加协议选项。 |
| `flagscale_agent/react/config.py` | 修复配置加载顺序，新增协议配置和校验。 |
| `flagscale_agent/react/providers/openai_provider.py` | 完成 Responses 请求、消息、工具和响应适配。 |
| `flagscale_agent/react/providers/__init__.py` | 将协议配置传给 Provider。 |
| `flagscale_agent/react/agent.py` | 处理新增流事件，保存推理状态并记录截断。 |
| `pyproject.toml` | OpenAI SDK 最低版本提高到 `1.66.0`。 |
| `tests/test_config_provider_override.py` | 配置加载与覆盖的回归测试。 |
| `tests/test_openai_responses.py` | 协议和多轮工具交互的回归测试。 |
| README 与配置示例 | 补充使用说明。 |

## 验证记录

完整补丁已在基准提交的原始源码上重新应用，重建后的 349 个文件与保存的源码副本一致。重建源码上的相关回归测试结果为 **121 passed**，覆盖配置加载、协议切换、请求格式、流式响应、并行工具参数、多轮工具交互及错误处理。

需要重跑这些测试时，进入解压后的源码目录执行：

```bash
pip install -e '.[dev]'
pytest -q tests/test_openai_responses.py tests/test_providers.py tests/test_config.py tests/test_config_provider_override.py tests/test_thinking_cap.py tests/test_empty_retry_thinking_fix.py
```

此前完整测试为 `1903 passed, 1 failed`，唯一失败是 `/tmp` 存储类型识别测试。该结果产生于最后的小幅请求过滤调整和新增三个针对性测试之前；最终版本重新通过了上述 121 项相关测试。

自动测试使用模拟 HTTP 与真实 OpenAI SDK；此次部署的真实中转已由使用者反馈可用。新服务器仍需要有效密钥、模型权限和可访问的服务地址。

## 可选：从上游源码应用补丁

已经通过迁移包安装的用户无需执行本节。这是另一种重建方式：

```bash
cd ~
git clone https://github.com/zz774ever/flagscale-agent-debug.git
git clone https://github.com/flagos-ai/FlagScale-Agent.git FlagScale-Agent-rebuilt
cd ~/FlagScale-Agent-rebuilt
git checkout --detach 4dff90ce84a7e8cfaac1af9a83f558baacc28d51
git apply --check ~/flagscale-agent-debug/migration-20261028/flagscale-agent-responses.patch
git apply ~/flagscale-agent-debug/migration-20261028/flagscale-agent-responses.patch
pip install -e .
```

随后按照前文创建 `~/.flagscale/agent.yaml` 并启动。配置模板也可从本仓库的 `migration-20261028/agent.yaml.example` 复制。

补丁针对上面固定的基准提交。未来更换上游版本时，需检查是否已包含相同修复及是否存在冲突。

## 常见问题

| 现象 | 检查方式 |
| --- | --- |
| 解压提示找不到压缩包 | 先完成下载，并确认文件在当前目录。 |
| `No such option: --openai-api` | 检查是否已安装迁移包内的修复源码，并用 `--help` 确认。 |
| `--openai-api requires an argument` | 使用 `--openai-api=responses`，或将协议写入 YAML。 |
| 模型不支持 Chat Completions | 确认实际加载的配置包含 `openai_api: responses`。 |
| API key 缺失或认证失败 | 将占位符换成真实密钥，核对服务商、密钥和权限。 |
| 找不到模型 | 核对服务商实际提供的模型名称与账号权限。 |
| DNS、连接失败或超时 | 检查服务器到中转的网络及代理配置。 |
| 工具被 Guard 拦截 | 阅读 Guard 的具体理由，按原项目流程处理。 |

## 上游与许可证

原项目：[flagos-ai/FlagScale-Agent](https://github.com/flagos-ai/FlagScale-Agent)。迁移包保留上游源码中的版权声明及 Apache License 2.0 许可证，详见包内 `FlagScale-Agent/LICENSE`。
