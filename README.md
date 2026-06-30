# 🤖 外卖广告B端自动化发版系统 — 技术深度分析

> 基于 AI Agent Skill 驱动的全流程自动化发版解决方案的技术分析文档集，可用于学习参考和简历素材包装。

## 📖 项目简介

本仓库是对美团外卖广告B端自动化发版系统（adfe-b-publish-automation）及其全部关联 Skill 的技术深度分析。系统以「AI Agent Skill 编排 + Cron 定时调度」的架构，将传统手工发版流程（每次 2-3 小时）完全自动化，覆盖 PR 合入检查、Stage 自动部署、备机验证催促、Prod 发版确认四大阶段。

每份文档提供 HTML（可视化交互）和 Markdown（纯文本阅读）两种格式，包含：项目概述、系统架构图、核心流程分析、关键技术细节、安全与防错机制、面试话术参考、简历要点（可直接引用）。

## 📂 文档结构

| 内容 | HTML 版本 | Markdown 版本 | 对应 Skill |
|------|-----------|---------------|-----------|
| **主控 Skill 全景分析** | [index.html](index.html) | [index.md](markdown/index.md) | adfe-b-publish-automation V56 |
| **上线计划详情查询** | [adfe-b-launch-plan-info.html](adfe-b-launch-plan-info.html) | [adfe-b-launch-plan-info.md](markdown/adfe-b-launch-plan-info.md) | adfe-b-launch-plan-info V8 |
| **上线计划搜索** | [adfe-b-launch-plan.html](adfe-b-launch-plan.html) | [adfe-b-launch-plan.md](markdown/adfe-b-launch-plan.md) | adfe-b-launch-plan V18 |
| **大象消息发送** | [daxiang-sender.html](daxiang-sender.html) | [daxiang-sender.md](markdown/daxiang-sender.md) | daxiang-sender V6 |
| **Groot 发版管理** | [groot-cli.html](groot-cli.html) | [groot-cli.md](markdown/groot-cli.md) | groot-cli V11 |
| **FSD 生态 (CLI + SSO)** | [fsd-ecosystem.html](fsd-ecosystem.html) | [fsd-ecosystem.md](markdown/fsd-ecosystem.md) | fsd V14 + fsd-sso V3 |

## 🏗️ 系统架构总览

```
┌─────────────────────────────────────────────────┐
│                   调度层                         │
│         Cron 定时任务 × 9 / 手动触发              │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│              控制层（主 Skill）                    │
│         adfe-b-publish-automation                │
│    流程编排 / 状态管理 / 异常处理 / 红线约束        │
└──────────────────────┬──────────────────────────┘
                       │ 编排调用
          ┌────────────┼────────────┐
          ▼            ▼            ▼
┌──────────────┐┌──────────────┐┌──────────────┐
│ launch-plan  ││ launch-plan  ││  daxiang-    │
│    -info     ││   (搜索)      ││   sender     │
│  详情查询     ││              ││  消息发送     │
└──────────────┘└──────────────┘└──────────────┘
          │            │            │
┌──────────────┐┌──────────────┐┌──────────────┐
│  groot-cli   ││    fsd       ││  fsd-sso     │
│  发版执行     ││  全流程CLI    ││  SSO认证      │
└──────────────┘└──────────────┘└──────────────┘
          │            │            │
          ▼            ▼            ▼
┌─────────────────────────────────────────────────┐
│                  外部系统                         │
│   大象IM / Groot部署 / FSD发版 / edopenapi / KMS  │
└─────────────────────────────────────────────────┘
```

## 🔑 核心技术亮点

**AI Agent 行为可靠性工程** — 五条绝对红线 + preflight checklist + 双重 RD 过滤，解决 AI Agent 在生产环境中的「自作聪明」问题。

**多层认证安全设计** — 四套认证体系（Groot / 大象 / FSD / edopenapi），KMS 动态凭证读取 + 命令行参数显式注入，解决沙箱环境凭证优先级覆盖问题。

**消息发送「文件中转」模式** — 从历史事故中提炼，通过 Python 中间脚本避免 Bash 对 IM @ 协议特殊字符的错误解析。

**Shell 驱动的确定性设计** — 日期推算、关键词匹配等判断逻辑全部由 Shell 脚本执行，AI 仅根据 exit code 决定分支，杜绝 AI 推算错误。

**优雅降级与自愈** — 结构变化检测、通用重试机制、Token 过期自动修复，保障系统在各种异常场景下不中断。

## 🎤 面试亮点提炼

每份文档都包含 2-5 个面试问答，覆盖以下高频话题：

- 项目背景与痛点分析
- AI Agent 行为约束的工程实践
- 多认证体系的设计与沙箱凭证覆盖问题
- 微服务编排架构（主 Skill + 关联 Skill）
- 防错体系设计（红线 / checklist / 双重过滤）
- 优雅降级与容错策略
- 上线效果与量化指标

## 📝 简历要点

每份文档末尾都有 3-6 条可直接粘贴到简历的项目经历描述，以及推荐的简历关键词标签（AI Agent、DevOps、CI/CD、微服务编排、防错设计等）。

## 🚀 在线访问

开启 GitHub Pages 后即可在线访问：`https://<username>.github.io/auto-publish-skill-html/`

## 📊 Skill 版本信息

| Skill | 版本 | 作者 | 角色 |
|-------|------|------|------|
| adfe-b-publish-automation | V56 | zhangshidong03 | 主控编排 |
| adfe-b-launch-plan-info | V8 | zhangshidong03 | 数据聚合 |
| adfe-b-launch-plan | V18 | zhangshidong03 | 计划搜索 |
| daxiang-sender | V6 | suhao20 | 消息投递 |
| groot-cli | V11 | yangshang | 发版执行 |
| fsd | V14 | lixuesong05 | 全流程 CLI |
| fsd-sso | V3 | lixuesong05 | SSO 认证 |

---

*基于 SkillHub 公开 Skill 分析整理 · 2026*
