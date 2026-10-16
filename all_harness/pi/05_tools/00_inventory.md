# Pi Concrete Tools 清单

> 状态：进行中。Tool System 框架已经完成；具体工具正在逐个共同阅读。

## 1. 当前内置工具

Pi Coding Agent 当前注册表包含 8 个内置工具名：

| 分类           | 工具           | 模型侧主要能力          | 当前状态  |
| ------------ | ------------ | ---------------- | ----- |
| 文件读取         | `read`       | 读取文本或图片，支持分页     | 已共同阅读 |
| 文件修改         | `write`      | 创建或完整覆盖文件        | 已共同阅读 |
| 文件修改         | `edit`       | 通过精确文本替换修改已有文件   | 已共同阅读 |
| 命令执行         | `bash`       | 执行 Shell 命令      | 已共同阅读 |
| Windows 命令执行 | `powershell` | 执行 PowerShell 命令 | 待共同阅读 |
| 内容搜索         | `grep`       | 在文件内容中搜索模式       | 待共同阅读 |
| 文件搜索         | `find`       | 按名称查找文件和目录       | 待共同阅读 |
| 目录浏览         | `ls`         | 列出目录内容           | 待共同阅读 |

主要代码入口：

~~~text
packages/coding-agent/src/core/tools/index.ts
~~~

## 2. 工具集合

Pi 并不总是把 8 个工具全部暴露给模型。

~~~text
createCodingTools()
  read / bash / edit / write

createReadOnlyTools()
  read / grep / find / ls

createAllTools()
  read / bash / powershell / edit / write / grep / find / ls
~~~

Registry 中存在不等于当前 Agent 一定可见；最终 Active Tools 仍由 AgentSession 和 Extension 配置决定。

## 3. 拆解结构

~~~text
05_tools/
├── 00_inventory.md
├── 01_read.md
├── 02_write.md
├── 03_edit.md
├── 04_shell_tools.md
└── 05_search_navigation_tools.md
~~~

`bash` 和 `powershell` 放在同一组比较；`grep`、`find`、`ls` 作为搜索与导航工具组比较。

## 4. 当前结论

Pi 的具体工具遵循一个共同模式：

~~~text
小型模型接口
        ↓
ToolDefinition 中的语义与执行逻辑
        ↓
可替换的 Operations 后端
        ↓
本地文件系统 / Shell 或宿主提供的远程环境
~~~

Operations 是宿主侧依赖注入接口，不是模型通过工具参数动态选择后端的能力。这个结论已在 `read`、`write`、`edit` 和 `bash` 中确认，后续需要检查其他工具是否保持一致。
