---
layout: post
title: 我的 skill 工作流
categories: ai coding, ai
description: 从需求到上线的 AI Skill 流水线：工作区初始化、头脑风暴、审查、Comet 交付、提交与发布
keywords: AI, skill, comet, brainstorming, grilling
---

# 我的 skill 工作流

## 总览

![AI Skill 流水线总览](/images/20260706-ai-agent-skills-workflow.png)

## 分工

| 类型 | Skill | 管什么 |
|:---|:---|:---|
| 社区 | brainstorming、grilling | 一个设计，一个挑刺 |
| 流程组件 | comet 一条龙 | 基于 SDD 规范的交付链，封装了 openspec 和 superpowers |
| 自建 | new-demand、git-commit、git-commit--publish | 工作区管理、代码提交、测试部署 |

## 详细步骤

| 步骤 | 命令 | 作用 |
|:---:|:---|:---|
| 1 | `/new-demand 77330` | 初始化需求号为 77330 的工作空间：目录、代码分支、spec 仓库软链 |
| 2 | `brainstorming 运营系统支持二维码登陆-prd.docx` | 头脑风暴，跟 AI 讨论需求文档以及技术方案 |
| 3 | `/grilling 审查刚才的需求和技术设计` | 对头脑风暴结果二次审查，查漏补缺 |
| 4 | `/comet-*` | 发起变更 → 设计 → 构建 → 验证 → 归档（详见下表） |
| 5 | `git-commit` | 按规范生成 commit message，并提交 |
| 6 | `git-commit--publish` | 将改动 merge 到 test 分支，并运行测试流水线 |

## Comet 子阶段

| 子命令 | 阶段 | 作用 |
|:---|:---|:---|
| `/comet-open` | 开启 | 澄清需求，创建 change（proposal + design + tasks） |
| `/comet-design` | 深度设计 | 产出 Design Doc 与 delta spec |
| `/comet-build` | 计划与构建 | 制定计划并实施代码 |
| `/comet-verify` | 验证与收尾 | 验证实现符合设计，处理分支 |
| `/comet-archive` | 归档 | 合并主 spec，归档 change |



