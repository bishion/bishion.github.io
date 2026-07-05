---
layout: post
title: 飞书与 Cursor 互通
categories: cursor, feishu
description: 通过 cc-connect 与飞书 CLI，实现飞书与 Cursor 长连接互通，远程指挥 AI 写代码
keywords: cursor, 飞书, cc-connect, lark-cli, feishu
---

# 飞书与 Cursor 互通

## 背景

1. 跟 Cursor 唠出来的最终设计文档怎么分享？
2. 一个疑难 bug 的排查过程，怎么沉淀到知识库？
3. 羡慕朋友圈别人可以指挥龙虾写代码？
4. 想在下班后，让 Cursor 接着干活？

## 互通方案

通过 cc-connect 跟飞书保持长连接，将飞书消息转发给 Cursor，Cursor 处理完成后将结果返回到聊天框。如果 Cursor 需要用到飞书相关功能，则通过调用飞书 CLI 来实现。

### 整体架构

### 组件说明

| 软件/插件 | 说明 |
|---|---|
| 飞书 [下载](https://www.feishu.cn/download) | 其他 IM 软件也可以，具体可以参考 cc-connect 的文档 |
| Cursor [下载](https://cursor.com/cn/download) | 其他 AI agent 也可以，具体可以参考飞书 CLI 的文档 |
| nodejs [下载](https://nodejs.org/zh-cn/download) | 插件基座，下面三个核心插件都是基于 node 环境 |
| 飞书 CLI [官网](https://www.feishu.cn/feishu-cli) | 负责让 Cursor 操作飞书，具体功能可以参考官网 |
| cc-connect [地址](https://github.com/chenhg5/cc-connect/blob/main/README.zh-CN.md) | 通过 websocket 与飞书连接，实现飞书与 agent 对话，具体功能可以参考官网 |
| Cursor CLI [官网](https://cursor.com/cn/docs/cli/overview) | Cursor 的 CLI 模式，让 cc-connect 使用命令行调用 Cursor |

## 安装与配置

### 飞书 CLI

```shell
# cmd/powershell 命令行执行，执行过程中会提示绑定飞书应用
npx @larksuite/cli@latest install
```

### Cursor CLI

#### 安装命令

```shell
# powershell 命令行执行，执行过程中会提示登录 cursor
irm 'https://cursor.com/install?win32=true' | iex
```

### cc-connect

#### 安装命令

```shell
# cmd/powershell 命令行执行
npm install -g cc-connect
```

#### 配置过程

```shell
# 安装完成后，先执行 cc-connect 命令, 此时会报错说找不到 config.toml 文件，但是它会自动生成一个
cc-connect

# 在 ~/.cc-connect/ 下找到 config.toml
# 将 project.agent 从默认的 claudecode，改为 cursor，保存
[projects.agent]
type = "cursor"

# 执行 cc-connect web，此时浏览器会自动弹出配置页面，不过打不开，正常的
cc-connect web

# 再执行 cc-connect, 会有一堆报错信息提示飞书配置错误之类，不用管
# 此时浏览器页面会自动刷新恢复(如果没有，手动刷新即可)
cc-connect

# 接着，飞书/工作目录这些，就可以在「项目」菜单里配置了，参考附录一
```

## 常见问题

### ★ 在飞书中让 Cursor 操作文件时，报没有权限

> 正在尝试用 `lark-cli` 查询「法诉研发」知识库并上传设计文档。
>
> 当前环境下无法替你执行 `lark-cli`（命令会被拦截），所以不能由我直接把文档推到飞书。仓库里已经备好正文与说明，你需要在本机终端执行。

这是因为默认 Cursor CLI 会禁用终端命令行。可以去 `~/.cursor/cli-config.json` 中修改 allow 选项，常见配置参考附录，也可以在 Cursor IDE 里让 Cursor 自己加。

![Cursor CLI 权限配置](/images/20260705-feishu-cursor-cli-permission.png)

### cc-connect 没有自动登录，需要 API 令牌

令牌保存在 `~/.cc-connect/config.toml` 文件，在最下方，选 **[management]** 下的 token 配置。

![cc-connect API 令牌配置](/images/20260705-feishu-cursor-cc-connect-token.png)

## 附录

### 附录一：cc-connect 配置页面

![cc-connect 配置页面](/images/20260705-feishu-cursor-cc-connect-config.png)

![cc-connect 配置页面（续）](/images/20260705-feishu-cursor-cc-connect-config-2.png)

### 附录二：Cursor CLI 授权

```json
"permissions": {
    "allow": [
      "Shell(echo)",
      "Shell(cd)",
      "Shell(dir)",
      "Shell(type)",
      "Shell(where)",
      "Shell(tree)",
      "Shell(findstr)",
      "Shell(more)",
      "Shell(cls)",
      "Shell(mkdir)",
      "Shell(md)",
      "Shell(copy)",
      "Shell(xcopy)",
      "Shell(robocopy)",
      "Shell(ls)",
      "Shell(cat)",
      "Shell(pwd)",
      "Shell(which)",
      "Shell(touch)",
      "Shell(head)",
      "Shell(tail)",
      "Shell(grep)",
      "Shell(find)",
      "Shell(rg)",
      "Shell(Get-ChildItem)",
      "Shell(gci)",
      "Shell(Get-Content)",
      "Shell(gc)",
      "Shell(Set-Location)",
      "Shell(sl)",
      "Shell(Test-Path)",
      "Shell(Select-String)",
      "Shell(sls)",
      "Shell(Write-Output)",
      "Shell(New-Item)",
      "Shell(Join-Path)",
      "Shell(Split-Path)",
      "Shell(ConvertTo-Json)",
      "Shell(ConvertFrom-Json)",
      "Shell(Invoke-WebRequest)",
      "Shell(iwr)",
      "Shell(Compress-Archive)",
      "Shell(Expand-Archive)",
      "Shell(git)",
      "Shell(node)",
      "Shell(npm)",
      "Shell(npx)",
      "Shell(yarn)",
      "Shell(pnpm)",
      "Shell(python)",
      "Shell(py)",
      "Shell(pip)",
      "Shell(pip3)",
      "Shell(java)",
      "Shell(javac)",
      "Shell(mvn)",
      "Shell(gradle)",
      "Shell(dotnet)",
      "Shell(go)",
      "Shell(cargo)",
      "Shell(docker)",
      "Shell(curl)",
      "Shell(wget)",
      "Shell(tar)",
      "Shell(code)",
      "Shell(cursor)",
      "Shell(winget)",
      "Shell(choco)",
      "Shell(scoop)",
      "Shell(powershell)",
      "Shell(pwsh)",
      "Shell(cmd)",
      "Shell(lark-cli)"
    ],
    "deny": [
      "Shell(rm)",
      "Shell(del)",
      "Shell(rmdir)",
      "Shell(Remove-Item)",
      "Shell(ri)",
      "Shell(format)",
      "Shell(Format-Volume)",
      "Shell(shutdown)",
      "Shell(Stop-Computer)",
      "Shell(Restart-Computer)"
    ]
  },
```
