---
title: VS Code cli.js module not found error
tags:
  - vscode
  - wsl
categories:
  - Work
category_bar: true
math: true
date: 2026-07-13 02:01:28
---


{% note success %}
vscode更新中断,导致wsl code命令打不开
{% endnote %}

## 问题总结

**现象**
在 WSL2 里执行 `code .` 报错 `Cannot find module '.../fc3def6774/resources/app/out/cli.js'`,VS Code 无法启动。

**根本原因**
Windows 端 VS Code 自动更新时出了问题:实际安装目录下的版本文件夹已经更新为 `4fe60c8b1c`,但 `bin/code` 这个启动脚本没有同步更新,里面硬编码的还是上一个版本的 commit hash `fc3def6774...`,导致脚本去找一个已经不存在的路径。

**排查过程**
1. 确认 `code` 命令指向的脚本路径正常
2. 查看安装目录,发现只有 `4fe60c8b1c` 文件夹,没有脚本里写的 `fc3def6774`
3. 检查脚本内容,发现两处硬编码的旧版本 hash:
   - `VERSIONFOLDER="fc3def6774"`(短 hash,拼本地文件路径)
   - `COMMIT="fc3def6774c76082adf699d366f31a557ce5573f"`(完整 hash,用于给 WSL 下载对应的 Remote Server)

**踩的一个坑**
第一次用 `sed` 简单地把两处 `fc3def6774` 都替换成 `4fe60c8b1c`,导致 `COMMIT` 变成了"新版本短 hash + 旧版本后半段"拼出来的**假 hash**,下载 VS Code Server 时 404。

**正确修复**
从实际安装的版本文件夹里的 `product.json` 查到真正完整的 commit hash(`4fe60c8b1cdac1c4c174f2fb180d0d758272d713`),把脚本里的 `COMMIT` 整行替换成这个真实值,而不是简单拼接。

**根本性质**
这是 VS Code Windows 端自动更新过程中断/不完整导致的文件不一致问题(`bin/code` 脚本没跟着新版本一起更新)。当前的修复是手动纠正,属于治标;如果以后再遇到类似情况,建议直接卸载重装 VS Code,避免每次更新都要手动排查。

先观察,更新后再次出现问题再考虑是否需要卸载重装。
