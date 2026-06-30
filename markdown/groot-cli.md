[← 返回主页](index.md) | [HTML 版本](../groot-cli.html)

---

# Groot 发版管理 CLI Skill

`发版执行引擎`

> 通过 HTTP 调用 Groot 服务端 API，在无浏览器/无交互环境下完成发版项目查询、发布触发、灰度放量管理及值班人查询。既是自动化发版流程的核心执行层，也是值班人身份校验的关键节点。

| 指标 | 值 |
|------|------|
| 大命令组 | **4** |
| 迭代版本 | **V11** |
| 认证优先 | **4 级** |
| API 驱动 | **REST** |

---

## 📋 项目概述

### 🎯 定位

**发版执行引擎 + 值班人校验工具**，是自动化发版流程的核心执行层。主 Skill（adfe-b-publish-automation）编排流程时，所有对 Groot 部署平台的操作均通过本 Skill 完成——项目列表查询、Stage/Prod 发布触发、灰度流量管控、部署状态轮询、值班人 oncall 校验。

### 📦 NPM 包信息

包名：`@adfe/groot-cli`（要求 >= 1.0.12，需要 oncall 子命令支持）

作者：yangshang | 版本：V11 | 通信协议：REST API over HTTPS

### 技术标签

`Node.js CLI` `REST API` `SSO Client Credentials` `Token 四级优先` `dry-run 预检` `状态轮询` `灰度管理` `值班人校验`

---

## 🗂️ 命令全景（4 组）

### 🔍 项目查询组

| 命令 | 功能 | 说明 |
|------|------|------|
| `list-projects` | 列出所有可部署项目 | 用于与上线计划做匹配，筛选出 A 类（Groot 可部署）项目 |
| `list-decouple-projects` | 列出解耦项目 | 解耦项目有独立的发版时间窗口（14:00-17:00） |
| `list-mrn-bundles` | 列出 MRN Bundle 列表 | 用于跨端应用的部署管理 |

### 🚀 发布管理组

| 命令 | 功能 | 关键参数 |
|------|------|----------|
| `publish` | 触发发布（test/stage/prod） | --env, --branch, --mis, --dry-run |
| `publish-status` | 查询发布状态 | --id（logInfo.id） |
| `cancel-publish` | 取消进行中的发布 | --id |

### 🎛️ 灰度管理组

| 命令 | 功能 |
|------|------|
| `gray full` | 全量放量 |
| `gray small-traffic` | 小流量放量 |
| `gray pause` | 暂停灰度 |
| `gray resume` | 恢复灰度 |
| `gray stop` | 停止灰度 |
| `gray immediate-full` | 立即全量（跳过灰度阶段） |
| `gray add-whitelist` | 添加灰度白名单 |

### 👤 值班人查询组

| 命令 | 功能 | 关键参数 |
|------|------|----------|
| `oncall` | 查询当前值班人 MIS | --format plain（纯文本输出，供脚本管道组合） |

oncall 命令是自动化流程启动时的第一个调用——校验当前执行环境的 MIS 与排班值班人一致后，才允许继续执行后续操作。

---

## ⚙️ 核心技术细节

### 🔐 四级认证优先级

groot-cli 的认证凭证按以下优先级依次尝试，首个可用即生效：

```
┌─────────────────────────────────────────────────────┐
│                  认证优先级架构                       │
├─────────────────────────────────────────────────────┤
│                                                     │
│   ┌──────────────────┐  ┌──────────────────────┐    │
│   │  优先级 1         │  │  优先级 2             │    │
│   │  环境变量          │  │  环境变量              │    │
│   │  GROOT_TOKEN      │  │  GROOT_COOKIE         │    │
│   └──────────────────┘  └──────────────────────┘    │
│                                                     │
│   ┌──────────────────┐  ┌──────────────────────┐    │
│   │  优先级 3         │  │  优先级 4             │    │
│   │  ~/.groot/token   │  │  ~/.groot/cookie      │    │
│   │  文件             │  │  文件                 │    │
│   └──────────────────┘  └──────────────────────┘    │
│                                                     │
└─────────────────────────────────────────────────────┘
```

在自动化场景中，推荐通过环境变量注入 Token（优先级最高），避免文件竞争和容器无状态问题。

### 🎫 Token 获取方式

```bash
# 通过 mtsso-client-credentials 获取 Groot API Token
mtsso-client-credentials --audience "ecbcdc79cf"

# 返回的 Token 设置为环境变量
export GROOT_TOKEN="<返回的 access_token>"

# 或直接在命令中使用
GROOT_TOKEN="xxx" groot list-projects
```

audience 值 `ecbcdc79cf` 是 Groot 服务在 SSO 中的注册标识。不同操作（list-projects vs publish）可能需要不同权限级别的 Token。

### 🛡️ CatClaw 环境安全校验

在 CatClaw（沙箱）环境下，groot-cli 通过 `GROOT_USER_TOKEN` 做操作人 MIS 比对，防止代他人操作：

```
# 安全校验流程
1. 从 GROOT_USER_TOKEN 解析出当前操作人的 MIS
2. 将操作人 MIS 与 publish --mis 参数进行比对
3. 若不一致 → 拒绝执行，抛出安全错误
4. 若一致 → 正常执行发布操作

# 目的：防止 A 用户的自动化任务以 B 用户身份触发发布
```

### 🌐 关键 API 端点

| 功能 | 端点（基于 https://groot.sankuai.com） | 方法 |
|------|----------------------------------------|------|
| 项目列表 | /api/projects | GET |
| 触发发布 | /api/publish | POST |
| 发布状态 | /api/publish/status | GET |
| 取消发布 | /api/publish/cancel | POST |
| 灰度操作 | /api/gray/* | POST |
| 值班人查询 | /api/oncall | GET |

### 🧪 dry-run 预检机制

在正式部署前，系统先执行 `groot publish --dry-run` 进行预检。预检会验证：分支是否存在、项目配置是否正确、权限是否充足、是否有进行中的发布冲突。只有预检通过的项目才会进入正式部署流程，预检失败的项目直接标记为跳过并记录原因。

```bash
# 预检示例
groot publish --env stage --branch feature/xxx --mis zhangsan --dry-run

# 预检通过输出
✓ dry-run passed: ready to publish

# 预检失败输出
✗ dry-run failed: branch "feature/xxx" not found
```

### 🔄 状态轮询机制

`publish-status` 支持以下中文状态枚举，主 Skill 以 30 秒间隔轮询，最长 30 分钟超时：

`初始化` `等待中` `发布中` `成功` `失败` `已取消`

```javascript
// 状态轮询逻辑（主 Skill 中）
while (elapsed < 30min) {
    status = groot publish-status --id ${logId}
    if (status in ["成功", "失败", "已取消"]) break
    sleep 30s
}
```

---

## 🔗 在主 Skill 中的使用场景

1. **场景一 · 值班人校验**
   **groot oncall → 比对当前用户 MIS**
   每个 Cron 定时任务启动后的第一件事，调用 `groot oncall --format plain` 获取当天值班人 MIS，与当前执行环境的 MIS 做比对。匹配则继续执行，不匹配则静默退出——这是整个自动化流程的「入口守卫」。

2. **场景二 · Stage 自动部署**
   **groot publish --env stage --branch \<betaBranch\> --mis \<值班人mis\>**
   Step 2 中对 A 类项目（Groot 管理的项目）逐个触发 Stage 部署。先 dry-run 预检，通过后正式执行，间隔 3s 防并发冲突。从 stdout 解析出 `logInfo.id` 供后续状态轮询使用。

3. **场景三 · 部署状态轮询**
   **groot publish-status --id \<logId\>**
   所有项目触发部署后，以 30 秒间隔持续轮询每个项目的部署状态，最长等待 30 分钟。全部到达终态后一次性汇总结果（成功/失败/超时），通过大象私聊通知值班人。

4. **场景四 · 项目匹配**
   **groot list-projects → 与上线计划做交集**
   `groot list-projects` 返回 Groot 平台已注册的所有可部署项目列表，与上线计划中的前端应用做匹配。匹配上的归为 A 类（可自动部署），未匹配的归为 C 类（需人工处理），实现自动化范围的精确划分。

---

## 🎤 面试话术参考

> **Q：四级认证优先级是怎么考虑的？**
>
> **A：** groot-cli 的认证设计遵循「**灵活性 + 安全性**」的平衡原则。四级优先级从高到低是：环境变量 GROOT_TOKEN → 环境变量 GROOT_COOKIE → ~/.groot/token 文件 → ~/.groot/cookie 文件。环境变量优先级最高，是因为在 CI/CD 和容器化场景中，环境变量注入是最标准的凭证传递方式，且不依赖文件系统状态。文件方式作为兜底，适用于本地开发场景。Token 优先于 Cookie，是因为 Token 有明确的过期时间和权限范围（scope），比 Cookie 更适合机器间通信。这种设计让同一个 CLI 既能用于自动化流水线（环境变量注入），也能用于开发者本地调试（文件存储），覆盖了所有使用场景。

---

> **Q：值班人校验机制如何保证发版安全？**
>
> **A：** 值班人校验是整个自动化发版系统的「**第一道安全门**」。具体实现分两层：第一层是 **oncall 入口守卫**——每个 Cron 任务启动后第一件事就调用 `groot oncall --format plain` 获取当天值班人 MIS，与当前执行环境的 MIS 做比对，不匹配则静默退出，防止非值班人的自动化任务误触发发版。第二层是 **GROOT_USER_TOKEN 操作人校验**——在实际执行 publish 时，CLI 会从 GROOT_USER_TOKEN 解析出操作人身份，与 --mis 参数做二次比对，防止 A 用户的任务以 B 用户身份发布。两层校验结合，从「谁能启动流程」和「谁在执行操作」两个维度确保发版操作只能由当天值班人触发，即使 Token 泄露也无法代他人操作。

---

## 📝 简历要点（可直接引用）

### Groot CLI Skill 相关经历

- 设计并实现了 Groot 发版管理 CLI Skill，封装 Groot 服务端 REST API 为命令行工具，支持项目查询、发布触发、灰度管控、状态轮询和值班人校验五大能力，作为自动化发版系统的核心执行层。
- 设计了四级认证优先级机制（环境变量 Token → 环境变量 Cookie → 文件 Token → 文件 Cookie），兼顾 CI/CD 容器化场景和本地开发场景，通过 mtsso-client-credentials 动态获取 Token，实现凭证零硬编码。
- 实现了 CatClaw 沙箱环境下的操作人身份安全校验（GROOT_USER_TOKEN MIS 比对），配合 oncall 值班人入口守卫，构建了「谁能启动 + 谁在操作」的双层发版安全保障，杜绝了代他人操作的风险。

### 🏷️ 推荐简历关键词

`CLI 工具设计` `REST API 封装` `SSO 认证` `Token 管理` `灰度发布` `dry-run 预检` `安全校验` `发版自动化`

---

*基于 groot-cli Skill V11 分析整理 · 作者 yangshang · 2026*
