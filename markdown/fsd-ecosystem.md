[← 返回主页](index.md) | [HTML 版本](../fsd-ecosystem.html)

---

# FSD 生态 — CLI + SSO 认证

`研发全流程平台`

> 覆盖部署、交付、测试、上线准备全链路的一站式 CLI 工具与认证基础设施

| 指标 | 值 |
|------|------|
| 大命令组 | **5** |
| 版本 | **V14+V3** |
| 覆盖 | **全流程** |
| 换票 | **自动** |

---

## 📋 项目概述

### fsd — 研发全流程管理 CLI

通过 `fsd` CLI 管理 FSD 平台的部署、交付自测、测试计划、上线准备全流程。以命令行方式封装 FSD 平台 API，让 AI Agent 和自动化脚本能够直接操作研发流程中的各个环节。

`版本: V14` `作者: lixuesong05` `CLI 工具`

### fsd-sso — FSD 体系的认证基础设施

为 FSD 体系提供 SSO 登录与 token 管理。负责统一处理 FSD CLI 所需的身份认证，包括登录态获取、token 缓存、过期检测和自动续期，是 fsd CLI 正常工作的前提条件。

`版本: V3` `作者: lixuesong05` `认证基础设施`

### 二者的关系

**fsd-sso 为 fsd CLI 提供登录态**，二者共享本地 token 存储。fsd CLI 的所有命令在执行前都会读取 fsd-sso 写入的本地 token。当 token 过期或不存在时，系统自动调用 fsd-sso 的登录流程获取新 token，实现无感续期。

```
┌─────────────────────────┐
│      fsd CLI            │
│    (业务命令执行)        │
└────────────┬────────────┘
             │ 读取 / 触发刷新
             ▼
┌─────────────────────────┐
│      fsd-sso            │
│   (登录 & Token 管理)    │
└────────────┬────────────┘
             │ 写入 / 读取
             ▼
┌─────────────────────────┐
│    本地 Token 存储       │
│     (~/.fsd/conf)       │
└─────────────────────────┘
```

---

## ⚙️ fsd CLI 命令全景（5 大命令组）

### 部署管理 — fsd deploy

骨干/备机/泳道部署、状态查询、失败分析。支持通过 `fsd deploy trigger` 触发部署，`fsd deploy status` 查询部署状态，`fsd deploy analyze` 分析部署失败原因。

### 交付管理 — fsd delivery

创建/触发/准出/finish 全链路。使用 `fsd delivery create` 创建交付单，`fsd delivery trigger` 触发交付流程，`fsd delivery check` 准出检测，`fsd delivery finish` 完成交付。

### 上线计划 — fsd pub

list/info/status/pre-rtag/rtag/merge。`fsd pub list` 搜索上线计划，`fsd pub info` 查看详情，`fsd pub status` 获取状态，`fsd pub pre-rtag` 备机验证，`fsd pub rtag` 打版，`fsd pub merge` 合入。

### 测试管理 — fsd test / fsd fstPlan

测试计划、FST 触发、准出检测。`fsd test trigger` 触发自动化测试，`fsd fstPlan create` 创建测试计划，`fsd fstPlan check` 检查测试准出条件是否满足。

### 实体管理 — fsd req/task/defect/pr

需求/任务/缺陷/PR 的 CRUD 操作。支持查询关联需求 `fsd req list`、创建任务 `fsd task create`、管理缺陷 `fsd defect list`、查看 PR 状态 `fsd pr status`。

### 命令组关系概览

```
                        研发全流程

┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  需求/任务    │  │  交付管理     │  │  部署管理     │
│ fsd req/task │  │ fsd delivery │  │ fsd deploy   │
└──────────────┘  └──────────────┘  └──────────────┘
                        │
                        ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  缺陷/PR     │  │  测试管理     │  │  上线计划     │
│fsd defect/pr │  │fsd test/fstPlan│ │  fsd pub     │
└──────────────┘  └──────────────┘  └──────────────┘
```

---

## 🔐 fsd-sso 认证机制

### 三件套命令

| 命令 | 功能 | 说明 |
|------|------|------|
| `fsd-sso login` | 登录获取 token | 根据环境自动选择登录方式（浏览器 SSO / MOA 换票） |
| `fsd-sso status` | 检查登录状态 | 输出当前 token 有效性、过期时间、用户身份 |
| `fsd-sso logout` | 注销登录态 | 清除本地 token 缓存 |

### 环境自动识别

fsd-sso 通过 `CATCLAW_PROFILE` 环境变量自动识别运行环境。在沙箱环境中，自动使用 MOA 换票方式获取 token，无需用户手动打开浏览器。非沙箱环境则使用标准浏览器 SSO 登录流程。

```javascript
// 环境识别逻辑
if (process.env.CATCLAW_PROFILE === 'sandbox') {
  // 沙箱环境 → MOA 换票，无需浏览器
  token = await moaTokenExchange();
} else {
  // 本地/CI → 浏览器 SSO 登录
  token = await browserSsoLogin();
}
```

### Token 优先级

系统按以下优先级查找可用 token，确保在各种环境下都能正确获取身份凭证：

1. **优先级 1（最高）— 环境变量 FSD_SSOID**
   允许用户或 CI 系统通过环境变量显式指定 token，跳过所有自动获取逻辑。

2. **优先级 2 — 本地 conf 文件**
   读取 ~/.fsd/conf 中缓存的 token，检查有效期，未过期则直接使用。

3. **优先级 3（兜底）— CatClaw MOA 换票**
   当前两种方式均不可用时，自动调用 MOA 换票接口获取新 token 并写入本地 conf。

### 代码集成 API

fsd-sso 对外暴露两个核心函数，供 fsd CLI 和其他工具集成使用：

```javascript
// 加载当前有效的 SSO Token
const token = await loadSsoToken();

// 构建包含 SSO Cookie 的请求头
const headers = buildFsdSsoCookieHeader(token);
// → { Cookie: 'ssoid=xxx; ...' }

// fsd CLI 内部每次请求 FSD API 前调用
const response = await fetch(fsdApiUrl, {
  headers: buildFsdSsoCookieHeader(await loadSsoToken())
});
```

### 401 自动修复机制

当 fsd CLI 请求 FSD API 返回 401（未授权）时，系统不会直接报错退出，而是执行以下自动修复流程：

1. **Step 1 — 检测 401 响应**
   fsd CLI 捕获到 HTTP 401 状态码，判断为 token 过期或无效。

2. **Step 2 — 自动判断环境**
   读取 CATCLAW_PROFILE，决定使用 MOA 换票（沙箱）还是 SSO 登录（本地）。

3. **Step 3 — 执行换票 / 登录**
   调用对应的认证方式获取新 token，写入本地 conf 文件。

4. **Step 4 — 自动重试原请求**
   携带新 token 重新发起原始请求，对调用方完全透明。

---

## 🔗 在主 Skill 中的使用场景

fsd 和 fsd-sso 作为能力层 Skill，被主 Skill（adfe-b-publish-automation）及其关联 Skill 封装调用。以下是具体的调用场景：

| 命令 | 封装 Skill | 使用场景 |
|------|-----------|---------|
| `fsd pub list` | adfe-b-launch-plan | 搜索上线计划，按关键词/日期/状态筛选当前生效的上线计划 |
| `fsd pub info` | adfe-b-launch-plan | 查询上线计划详情，获取包含的应用列表、负责人、部署状态 |
| `fsd pub status` | adfe-b-launch-plan-info | 获取每个应用的「合入集成分支」状态，用于 Step 1 PR 合入检查 |
| `fsd pub pre-rtag` | adfe-b-launch-plan-info | 获取每个应用的「备机验证」结果，用于 Step 3 备机验证检查 |
| `fsd-sso login` | 主 Skill 自动调用 | 当任何 fsd 命令返回"认证失败"时，自动执行登录后重试 |

### 调用链路示意

```
┌─────────────────────────────────────┐
│          触发层                      │
│       Cron 定时任务                  │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│          编排层                      │
│   adfe-b-publish-automation         │
│          (主 Skill)                  │
└──────────────────┬──────────────────┘
                   │ 编排调用
                   ▼
┌────────────────────┐  ┌─────────────────────┐
│      封装层         │  │       封装层         │
│ adfe-b-launch-plan │  │adfe-b-launch-plan-info│
│ pub list / pub info│  │pub status / pub pre-rtag│
└─────────┬──────────┘  └──────────┬──────────┘
          │                        │
          └────────────┬───────────┘
                       │ CLI 调用
                       ▼
┌────────────────────┐  ┌──────────────┐
│      能力层         │  │   能力层      │
│     fsd CLI        │  │   fsd-sso    │
│  (实际执行命令)     │  │  (认证保障)   │
└────────────────────┘  └──────────────┘
```

---

## 🎤 面试话术参考

> **Q：fsd 和 fsd-sso 的分层设计有什么好处？**

> **A：** 我们把 FSD 体系的 CLI 工具拆分为两个独立的 Skill：**fsd 负责业务命令**（部署、交付、测试、上线计划），**fsd-sso 负责认证基础设施**（登录、token 管理、环境适配）。这种分层设计有三个好处：
>
> 第一，**关注点分离**。fsd 只关心"做什么"（部署一个应用、查一个上线计划），不需要关心"以谁的身份做"。认证逻辑完全封装在 fsd-sso 中，对业务层透明。
>
> 第二，**独立迭代**。认证机制的变更（比如 SSO 升级、MOA 换票协议变化）只需要升级 fsd-sso，不影响 fsd 的任何业务命令。反过来，fsd 新增命令也不需要动认证层。
>
> 第三，**复用性**。fsd-sso 提供的 `loadSsoToken()` 和 `buildFsdSsoCookieHeader()` 不仅被 fsd 使用，也可以被其他需要 FSD 认证的工具直接复用，避免重复实现登录逻辑。

---

> **Q：401 自动修复机制是怎么实现的？**

> **A：** 这是一个典型的**「拦截 → 修复 → 重试」三段式自愈模式**。当 fsd CLI 请求 FSD 平台 API 返回 HTTP 401 时，系统不会直接报错退出，而是进入自动修复流程：
>
> 首先，**环境判断**。读取 `CATCLAW_PROFILE` 环境变量，判断当前是沙箱环境还是本地环境。沙箱环境使用 MOA 换票（无需浏览器交互），本地环境使用标准 SSO 登录。
>
> 然后，**执行换票/登录**。调用 `fsd-sso login`，获取新的 token 并写入本地 `~/.fsd/conf` 文件。
>
> 最后，**透明重试**。携带新 token 重新发起原始请求。整个过程对上层调用方完全透明——上层只是调了一个 `fsd pub list`，感知不到中间经历了 401 → 换票 → 重试的过程。
>
> 这个设计的核心价值在于，自动化流程中的 **token 过期不再是阻塞事件**。以前 token 一过期，整个 Cron 任务就会失败，需要人工介入重新登录。现在系统能自愈，只有在自动修复也失败的极端情况下才会通知值班人。

---

## 📝 简历要点（可直接引用）

### FSD 生态相关项目经历

- 设计并实现了 FSD CLI 与 SSO 认证的分层架构，将业务命令（部署/交付/测试/上线）与认证基础设施解耦为独立模块，支持独立迭代和跨工具复用，累计迭代至 V14+V3 版本。
- 实现了 401 自动修复机制（拦截 → 环境判断 → 换票/登录 → 透明重试），使自动化流程中的 token 过期从阻塞事件降级为无感自愈，消除了人工介入需求。
- 设计了三级 Token 优先级策略（环境变量 → 本地缓存 → MOA 换票），结合 CATCLAW_PROFILE 环境自动识别，确保 CLI 工具在本地开发、CI 流水线、沙箱环境三种场景下均能正确获取身份凭证。

### 推荐简历关键词

`FSD 平台` `CLI 工具` `SSO 认证` `Token 管理` `自动换票` `分层架构` `401 自愈` `研发效能` `DevOps` `MOA 换票` `环境适配` `微服务编排`

---

*基于 fsd V14 + fsd-sso V3 分析整理 · 2025-07*
