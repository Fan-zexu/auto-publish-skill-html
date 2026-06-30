[← 返回主页](index.md) | [HTML 版本](../daxiang-sender.html)

---

# 大象消息发送 Skill

> `IM 消息通道`
>
> 统一的大象 IM 消息投递能力封装，基于大象开放平台 API 实现文本、Markdown、链接、文件、图片、名片、模板、富文本等全类型消息发送，支持 @人、@所有人、@机器人及动态消息更新，被多个团队和 Skill 复用的基础通信通道。

| 指标 | 值 |
|------|------|
| 消息类型 | 10+ |
| 迭代版本 | V6 |
| 限流策略 | QPS<5 |
| 认证机制 | OAuth2 |

---

## 📋 项目概述

### 🎯 Skill 定位

daxiang-sender 是一个**统一的大象 IM 消息投递能力封装**，通过大象开放平台 API 实现各类消息的发送。它不是一个独立运行的业务系统，而是作为基础通信能力被多个团队和 Skill 复用——例如自动化发版系统（adfe-b-publish-automation）通过调用 daxiang-sender 在发版群中发送催促消息和上线计划通知。

### 📦 核心脚本

整个 Skill 的核心实现是 `scripts/send.py`，一个 Python 3 脚本，依赖 `requests` 库进行 HTTP 请求和 `PyJWT` 库进行 Token 处理。脚本通过命令行参数接收消息类型、目标群/用户、凭证等信息，完成 OAuth2 认证后调用大象开放平台的消息发送接口。

### 技术栈一览

`Python 3` `requests` `PyJWT` `OAuth2 / OIDC` `大象开放平台 API` `CLI 参数化` `KMS 凭证管理` `QPS 限流` `文件中转模式`

### Skill 基本信息

| 属性 | 值 |
|------|------|
| 名称 | daxiang-sender |
| 版本 | V6 |
| 作者 | suhao20 |
| 更新者 | wuqiqi05 |
| 核心脚本 | scripts/send.py |
| 语言 | Python 3 |
| 依赖 | requests, PyJWT |

---

## 📨 消息类型全景

daxiang-sender 封装了大象开放平台支持的所有主流消息类型，通过统一的命令行参数接口对外暴露，调用方无需关心底层 API 细节。

| 类型 | 命令参数 | 说明 |
|------|----------|------|
| `text` | `--text` | 纯文本消息，最基础的消息类型 |
| `markdown` | `--markdown --text` | Markdown 格式消息，支持标题、列表、加粗等富文本排版 |
| `link` | `--link-url --link-title` | 链接卡片消息，展示为可点击的链接预览 |
| `multilink` | `--multilink-json` | 多链接卡片消息，一条消息中包含多个链接 |
| `file` | `--file` | 文件消息，发送本地文件到会话 |
| `image` | `--image` | 图片消息，发送本地图片到会话 |
| `vcard` | `--vcard` | 个人名片消息，展示用户的联系方式卡片 |
| `gvcard` | `--gvcard` | 群名片消息，展示群聊的邀请卡片 |
| `quote` | `--quote-id --text` | 引用回复消息，回复某条已发送的消息 |
| `custom` | `--template-id` | 模板消息，支持自定义按钮和交互组件 |
| `general` | `--general-json` | 富文本消息，JSON 定义的复杂富文本结构 |

---

## ⚙️ 核心技术细节

### 🔐 OAuth2 Token 管理

daxiang-sender 通过 OAuth2 Client Credentials 流程获取 accessToken，认证端点为 `https://ssosv.sankuai.com/sson/auth/oidc/v1/token`。脚本在每次发送消息前自动获取 Token，并在内存中缓存以减少重复请求。Token 过期后自动重新获取，全程无需人工介入。

```python
# OAuth2 Token 获取流程
def get_access_token(client_id, client_secret):
    url = "https://ssosv.sankuai.com/sson/auth/oidc/v1/token"
    data = {
        "grant_type": "client_credentials",
        "client_id": client_id,
        "client_secret": client_secret,
    }
    resp = requests.post(url, data=data)
    return resp.json()["access_token"]
```

### 📊 三级凭证优先级

为适配不同使用场景（CI/CD 管道、本地调试、团队共享配置），daxiang-sender 设计了三级凭证读取优先级机制，高优先级覆盖低优先级：

```
┌─────────────────────────────────────────────┐
│         最高优先级                            │
│  ┌───────────────────────────────────────┐  │
│  │  命令行参数                             │  │
│  │  --client-id / --client-secret        │  │
│  └───────────────────────────────────────┘  │
│                    ↓ 未指定则降级             │
│  ┌───────────────────────────────────────┐  │
│  │  环境变量                               │  │
│  │  DX_CLIENT_ID / DX_CLIENT_SECRET      │  │
│  └───────────────────────────────────────┘  │
│                    ↓ 未指定则降级             │
│         最低优先级                            │
│  ┌───────────────────────────────────────┐  │
│  │  配置文件                               │  │
│  │  ~/.daxiang.json                      │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

这套优先级设计的核心考量是**沙箱环境兼容性**。AI Agent 沙箱环境会预设 `DX_CLIENT_ID` 环境变量指向沙箱 Bot，如果使用配置文件方式传入凭证，会被环境变量覆盖。因此在自动化发版系统中，始终通过命令行参数（最高优先级）显式传入从 KMS 动态获取的凭证，确保使用正确的机器人身份。

### 👥 @ 提及的实现机制

大象 IM 的 @ 功能需要同时满足两个条件才能生效：

**条件一：** 在消息的 `extension.at` 字段中设置需要 @ 的 uid 列表，这告诉大象服务端「这条消息要通知哪些人」。

**条件二：** 在 `body.text` 中插入 `[@姓名|mtdaxiang://profile?uid=xxx]` 格式的高亮文本，这让消息在客户端展示时呈现蓝色可点击的 @ 样式。

```python
# @ 协议文本格式
"[@张三|mtdaxiang://profile?uid=100001]"
"[@李四|mtdaxiang://profile?uid=100002]"

# 消息结构示例
{
  "body": {
    "text": "请 [@张三|mtdaxiang://profile?uid=100001] 尽快合入 PR"
  },
  "extension": {
    "at": ["100001"]
  }
}
```

缺少任一条件都会导致 @ 失效：只设 extension.at 不插入协议文本，对方收到通知但消息中看不到高亮；只插入协议文本不设 extension.at，消息中有蓝色文字但对方不会收到 @ 通知。

### 🎯 custom-at 模式

`--custom-at` 参数允许消息发送者自行控制 @ 文本在消息中的位置。默认模式下，send.py 会自动在消息末尾追加所有 @ 协议文本。开启 custom-at 后，调用方需要在 `--text` 内容中预先插入好 @ 协议文本的位置，send.py 仅负责设置 `extension.at` 字段。

这在自动化发版系统中被广泛使用——催促消息中需要在每个应用名后面 @ 对应的 RD，而非在消息末尾统一 @。

### 🔄 动态消息

daxiang-sender 支持后续**更新**和**撤回**已发送的消息。发送成功后返回 messageId，调用方可以使用这个 ID 对消息进行修改或撤回操作。典型应用场景：部署状态轮询过程中，先发一条"部署中..."的消息，部署完成后更新为最终结果，避免群内消息刷屏。

---

## 📤 在主 Skill 中的使用模式

### 📁 「文件中转」模式详解

这是自动化发版系统（adfe-b-publish-automation）调用 daxiang-sender 时的核心设计模式，从历史事故中提炼而来。

**问题根因：** 大象 @ 协议文本 `[@姓名|mtdaxiang://profile?uid=xxx]` 中包含 `|`（管道）、`://`、`[`、`]` 等 Bash 特殊字符。如果将消息内容直接 inline 到 shell 命令的 `--text` 参数中，Bash 会错误解析这些字符，导致消息被截断或命令执行失败。

### 三步文件中转法

1. **Step 1 · write — 消息内容写入临时文件**
   使用 AI Agent 的 write 工具将完整消息内容（包含 @ 协议文本）写入 `/tmp/msg.txt`。write 工具直接操作文件系统，内容不经过 Bash 解析，因此特殊字符被完整保留。

2. **Step 2 · write — @ 列表写入临时文件**
   将需要 @ 的人员信息写入 `/tmp/at_list.txt`，每行格式为 `uid:姓名`。这个文件用于 Python 中间脚本构建 `extension.at` 字段。

3. **Step 3 · bash — Python 中间脚本读文件并调用 send.py**
   Python 中间脚本从文件中读取消息内容和 @ 列表，通过 `subprocess.call` 以参数列表形式传给 send.py。由于 Python 的 subprocess 使用 exec 系统调用而非 shell 解析，消息内容中的特殊字符不会被二次解析。

```python
# Step 1: AI Agent write 工具写入消息文件（不经过 bash）
write → /tmp/msg_step1.txt
# 文件内容示例：
# 以下应用尚未合入，请尽快处理：
# - app-foo: [@张三|mtdaxiang://profile?uid=100001]

# Step 2: AI Agent write 工具写入 @ 列表文件
write → /tmp/at_list.txt
# 每行格式: uid:姓名
# 100001:张三
# 100002:李四

# Step 3: Python 中间脚本读取文件 → 调用 send.py
import subprocess

with open("/tmp/msg_step1.txt") as f:
    text = f.read().strip()

cmd = ["python3", send_py, "send",
       "--group", group_id,
       "--client-id", dx_client_id,
       "--text", text,
       "--custom-at"]

subprocess.call(cmd)  # 参数列表形式，不经过 shell
```

### 💡 为什么这样设计

这个设计的本质是**绕过 Bash 的字符串解析**。三个关键的"不经过"：

1. **消息内容不经过 Bash**：通过 AI Agent 的 write 工具直接写入文件，Bash 完全不参与。
2. **文件读取不经过 Bash**：Python 脚本用 `open()` 读取文件内容，Bash 完全不参与。
3. **参数传递不经过 Bash**：`subprocess.call(cmd_list)` 使用参数列表而非字符串形式，底层用 exec 而非 shell，Bash 完全不参与。

这套方案从历史事故中提炼，已被团队作为标准实践推广，所有需要发送包含特殊字符的大象消息的 Skill 都采用这个模式。

---

## 🎤 面试话术参考

**Q：消息发送为什么要通过文件中转，而不是直接把内容放在命令行参数里？**

> 这是从一个线上事故中提炼出来的设计。我们的大象 IM @ 协议文本格式是 `[@姓名|mtdaxiang://profile?uid=xxx]`，里面包含 `|`（管道符）、`://`、`[`、`]` 这些 Bash 特殊字符。如果直接把这些内容放在 shell 命令的 `--text` 参数里，Bash 会把 `|` 解释为管道、把 `[]` 解释为通配符，导致消息被截断甚至命令执行失败。
>
> 我的解决方案是**「三步文件中转法」**：第一步，用 AI Agent 的 write 工具把消息内容直接写入临时文件，这一步完全不经过 Bash；第二步，把 @ 人员列表也写入文件；第三步，用 Python 中间脚本读取文件，以 **subprocess.call 的参数列表形式**（而非字符串形式）调用 send.py。整个链路中消息内容的三次传递都绕过了 Bash 的字符串解析，从根本上杜绝了特殊字符被截断的问题。这套方案后来成了团队的标准实践，所有涉及大象消息发送的 Skill 都使用这个模式。

**Q：凭证的三级优先级是怎么设计的，为什么要这样分层？**

> daxiang-sender 设计了**命令行参数 > 环境变量 > 配置文件**三级凭证读取优先级。设计这套分层机制的核心驱动力是**沙箱环境兼容性**。
>
> **配置文件**（`~/.daxiang.json`）是最低优先级，适合本地开发调试场景，开发者把凭证写一次后续不用重复输入。**环境变量**（`DX_CLIENT_ID`）优先级高于配置文件，适合 CI/CD 管道等标准化环境。但问题出在 AI Agent 的沙箱环境——沙箱会预设 `DX_CLIENT_ID` 环境变量指向沙箱 Bot 的凭证。如果我们在配置文件里写了正确的生产凭证，会被沙箱的环境变量覆盖，导致「机器人不在该群」等鉴权错误。
>
> 因此**命令行参数**（`--client-id`）被设计为最高优先级。在自动化发版系统中，我们从 KMS 动态获取凭证，然后通过命令行参数显式注入，确保无论沙箱环境变量怎么设置，都能使用正确的机器人身份。这是一个典型的「环境感知型凭证管理」设计。

---

## 📝 简历要点（可直接引用）

### 项目经历描述

- 设计并实现了统一的大象 IM 消息投递 Skill（daxiang-sender），封装 10+ 种消息类型（文本/Markdown/链接/文件/图片/名片/模板/富文本等），通过 OAuth2 认证和三级凭证优先级机制适配多环境部署，被多个团队和 Skill 复用为基础通信通道。
- 针对 AI Agent 沙箱环境下 @ 协议特殊字符被 Bash 截断的线上事故，设计了「三步文件中转法」：消息写入临时文件 → Python 中间脚本读取 → subprocess 参数列表传递，全链路绕过 Bash 解析，方案已成为团队标准实践。
- 设计了命令行参数 > 环境变量 > 配置文件的三级凭证优先级机制，解决了沙箱环境变量覆盖导致的鉴权错误问题，结合 KMS 动态凭证读取实现了安全的自动化凭证管理。

### 推荐简历关键词

`大象开放平台` `OAuth2` `IM 消息通道` `Python CLI` `凭证管理` `文件中转模式` `沙箱兼容` `KMS` `防错设计`

---

*基于 daxiang-sender Skill V6 分析整理 · 2025*
