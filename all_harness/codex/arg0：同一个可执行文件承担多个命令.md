---
created: 2026-10-28
tags:
  - 开发经验
  - 命令行
  - 进程
  - Codex
---

# arg0：同一个可执行文件承担多个命令

> 程序启动后，可以检查自己是通过什么调用名启动的，再选择执行哪段入口代码。这样一个可执行文件就能对外提供多个命令。

> 状态：前期资料记录。用户表示此前讲解尚未听懂，Codex 系统拆解尚未开始；本篇不标记为已共同理解。

下文核对的是本地 Codex 仓库 `c0e88f3c04219aaa032aaf4b20a8c3284cc81449` 中的实现。这里的“角色”是**程序功能入口**，与 Agent 的角色提示词不是同一件事。

## 1. 先用一个生活例子理解

想象有一个工具箱，里面同时有补丁工具和沙箱工具。给工具箱贴上不同的入口标签：

```text
用 codex 启动                → 进入普通 Codex 程序
用 apply_patch 启动          → 进入补丁工具
用 codex-linux-sandbox 启动  → 进入 Linux 沙箱工具
```

这些标签可以指向同一个可执行文件。每次调用仍然是一次独立的程序启动，可能形成不同进程；复用的是同一份可执行文件中的代码，并不是同一个进程同时变成了多个 Agent。

## 2. `argv[0]` 是什么？

程序启动时会收到一组参数，通常第 0 项 `argv[0]` 表示启动它时使用的名称或路径，其后才是传给程序的其他参数。

```text
apply_patch <其他参数>
    │            │
 argv[0]       后续参数
```

它是调用方提供的启动信息，并不一定等于可执行文件的真实磁盘路径，也不应被当作可信的身份或安全凭证。Codex 这里使用它来选择程序入口。

## 3. Codex 怎样使用这个技巧

Unix 环境下，Codex 会在临时目录中创建名为 `apply_patch` 的符号链接，指向当前可执行文件，并把该目录加入 `PATH`；Linux 下还会创建 `codex-linux-sandbox` 入口。这样调用对应名称时，可以重新启动同一份 Codex 可执行文件。

程序入口先调用 `arg0_dispatch()`。该函数读取 `argv[0]`，取出路径最后一段作为名字，然后决定：

```text
启动同一份可执行文件
        ↓
检查 argv[0] 的文件名部分
        ├─ apply_patch / applypatch → 补丁工具入口
        ├─ codex-linux-sandbox     → Linux 沙箱入口
        └─ 其他名称                → 继续普通启动流程
```

这是简化示意。源码还检查部分特殊的 `argv[1]`，用于其他辅助入口；实际路径并不全部依赖 `argv[0]`。Windows 的 `apply_patch` 则使用批处理脚本和特殊参数调用当前可执行文件，不应把 Unix 的符号链接方式直接套到 Windows。

## 4. 为什么这样做？

Codex 的源码注释说明，目标之一是方便以单个可执行文件部署，同时把部分功能暴露成不同的命令。程序按调用名分派，使辅助工具可以复用同一份可执行文件，而不要求用户为这些命令再单独安装一个完整程序。

这是一种进程启动与部署技巧。它不会告诉语言模型“你是哪个 Agent”，也不负责设置 Agent 的角色、任务或上下文。

## 5. 源码锚点

本地仓库：`/Users/zhouchenghao/python_file/codex`，版本 `c0e88f3c04219aaa032aaf4b20a8c3284cc81449`。

| 内容 | 源码 |
| --- | --- |
| CLI 启动时调用分派入口 | `codex-rs/cli/src/main.rs`，`main()` |
| 读取 `argv[0]` 并分派补丁与沙箱 | `codex-rs/arg0/src/lib.rs`，`arg0_dispatch()` |
| 普通入口和运行时包装 | 同文件 `arg0_dispatch_or_else()` |
| Unix 符号链接、PATH 与 Windows 批处理 | 同文件 `prepare_path_entry_for_codex_aliases()`、`prepare_aliases_in_home()` |
| Linux 沙箱调用名 | `codex-rs/sandboxing/src/landlock.rs`，`CODEX_LINUX_SANDBOX_ARG0` |

相关笔记：[[all_harness/codex/多 Agent 协作：上下文继承、消息与任务|多 Agent 协作：上下文继承、消息与任务]]。
