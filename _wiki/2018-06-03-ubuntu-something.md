---
layout: wiki
title: ubuntu 一些事儿
categories: Linux
description: ubuntu 中的一些事项
keywords: Linux, ubuntu
---
# ctrl+alt+left/right 占用
gnome 中，该快捷键是切换到左/右工作区，但是日常开发中，很多IDE也会用到它。  
新版的 ubuntu 中，快捷键配置界面并没有它的选项。  
可以使用命令来修改：
```bash
gsettings set org.gnome.desktop.wm.keybindings switch-to-workspace-left "['']"
gsettings set org.gnome.desktop.wm.keybindings switch-to-workspace-right "['']"
```

# ubuntu18 无法安装docker（20180603）
在 ubuntu18 中执行 apt install docker-ce 时，总会提示不在包列表中  
原因： docker 还没有 ubuntu18 的下载源  
解决方式：
1. 使用 ubuntu17 的源
2. 使用snap命令：sudo snap install docker
