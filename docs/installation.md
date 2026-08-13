# 🔧 环境配置与安装指南

本文档详细说明了从零开始配置 `Revit + MCP + AI 智能体` 工作流的完整步骤。

---

## 📋 目录

- [环境要求](#-环境要求)
- [核心组件说明](#-核心组件说明)
- [安装步骤](#-安装步骤)
  - [1. 下载 Revit 插件](#1-下载-revit-插件)
  - [2. 安装 Revit 插件](#2-安装-revit-插件)
  - [3. 安装 MCP Server](#3-安装-mcp-server)
  - [4. 安装 VS Code 与 Cline](#4-安装-vs-code-与-cline)
  - [5. 配置 Cline（连接大模型 API）](#5-配置-cline连接大模型-api)
  - [6. 配置 MCP Server 连接](#6-配置-mcp-server-连接)
  - [7. 启用 Revit 服务](#7-启用-revit-服务)
  - [8. 测试对话](#8-测试对话)
- [常见问题](#-常见问题)

---

## 💻 环境要求

| 组件 | 版本/要求 | 说明 |
| :--- | :--- | :--- |
| **Revit** | 2020 或更高版本 | 核心操作平台 |
| **Node.js** | 18+ | JavaScript 运行环境，用于运行 MCP Server |
| **Visual Studio Code** | 最新稳定版 | 承载 AI 客户端（Cline）的平台 |
| **Visual Studio** | 2022 或更高版本 | 安装时需要本地编译，**必须勾选** “使用 C++ 的桌面开发” 工作负载 |
| **大模型 API Key** | 如 DeepSeek、Claude、OpenAI 等 | 为 AI 智能体提供推理能力 |

> ⚠️ **特别提醒：** Visual Studio 的 C++ 工具链是安装 MCP Server 时的**必需依赖**，请务必提前安装。

---

## 🧩 核心组件说明

这套工作流由三个核心组件协同工作：

| 组件 | 作用 | 通俗理解 |
| :--- | :--- | :--- |
| **revit-mcp-plugin** | Revit 端插件，提供 Socket 服务 | 相当于 Revit 的 “电话接线员”，负责接收和响应外部指令 |
| **mcp-server-for-revit** | MCP 服务端，连接 AI 与 Revit | 相当于 “翻译官”，把 AI 的意图翻译成 Revit 能懂的命令 |
| **Cline（VS Code 扩展）** | AI 智能体对话入口 | 相当于你的 “AI 助手”，你在这里跟它说话 |

---

## 🚀 安装步骤

### 1. 下载 Revit 插件

原工作流由多个独立仓库组成，现已合并为单仓库方案，并直接提供编译好的 Release 包。

1. 访问 Release 页面：[mcp-servers-for-revit/releases](https://github.com/mcp-servers-for-revit/mcp-servers-for-revit/releases)
2. 根据你的 Revit **版本** 下载对应的压缩包。例如，Revit 2020 对应：
   - `mcp-servers-for-revit-v1.x.x-Revit2020.zip`

> 📌 **版本对应规则：** 文件名中的 `Revit2020`、`Revit2021` 等表示适用版本，请选择与你 Revit 版本一致的包。

### 2. 安装 Revit 插件

1. 解压下载好的 ZIP 包。
2. 将解压后的**全部内容**复制到 Revit 插件目录（Revit 2020 对应 `Addins\2020`，其他版本替换为对应年份），复制后的目录结构如下：

```text
Addins\2020
├── mcp-servers-for-revit.addin   ← 注意：只保留这一个 .addin 文件
└── revit_mcp_plugin
    ├── revit-mcp-plugin.dll
    └── Commands
        └── RevitMCPCommandSet
            └── 2020
                └── RevitMCPCommandSet.dll
    └── ... (其他文件夹)
```

> ⚠️ **注意：** 解压后的 Release 包中可能包含两个 `.addin` 文件（如 `mcp-servers-for-revit.addin` 和 `RevitMCPPlugin.addin`）。**请只保留 `mcp-servers-for-revit.addin`，删除另一个**，以避免冲突。

### 3. 安装 MCP Server

MCP Server 是连接 AI 与 Revit 的桥梁，通过 `npm`（Node.js 包管理器）全局安装。

1. **确保已安装 Visual Studio**，并勾选了 “**使用 C++ 的桌面开发**” 组件。
2. 打开命令提示符（Win + R，输入 `cmd`，回车）。
3. 执行安装命令：

   ```bash
   npm install -g mcp-server-for-revit
   ```

4. 验证安装是否成功：

   ```bash
   where.exe mcp-server-for-revit
   ```

   如果返回了文件路径（例如 `C:\Users\你的用户名\AppData\Roaming\npm\mcp-server-for-revit`），则表示安装成功。

### 4. 安装 VS Code 与 Cline

1. 如果你还没有 VS Code，请从 [code.visualstudio.com](https://code.visualstudio.com) 下载并安装。
2. 打开 VS Code，点击左侧的 **扩展（Extensions）** 图标（或按 `Ctrl+Shift+X`）。
3. 在搜索框中输入 **Cline**，找到后点击 **Install** 安装。
4. 安装完成后，VS Code 左侧边栏会出现 Cline 的机器人图标，点击即可打开对话窗口。

### 5. 配置 Cline（连接大模型 API）

Cline 需要接入一个大模型 API 才能工作。以 **DeepSeek** 为例：

1. 在 Cline 对话窗口顶部，点击 **齿轮图标** 进入设置。
2. 在 **API Provider** 中选择 `OpenAI Compatible`。
3. 填写以下信息：

| 配置项 | 值 |
| :--- | :--- |
| **Base URL** | `https://api.deepseek.com/v1` |
| **API Key** | 填入你的 DeepSeek API Key（可在 platform.deepseek.com 获取） |
| **Model ID** | `deepseek-chat` 或 `deepseek-reasoner`（根据你的需求选择） |

4. 点击 **Save** 保存配置。

> 💡 **其他模型参考：** 如果使用 Claude，API Provider 选择 `Anthropic`；使用 OpenAI 则选择 `OpenAI`，并填写对应的 Base URL 和 API Key。

### 6. 配置 MCP Server 连接

这一步让 Cline 能够找到并调用刚安装的 MCP Server。

1. 在 VS Code 中，点击 Cline 窗口底部的 **“Manage MCP Servers”** 按钮（或点击窗口右上角的齿轮，选择 `Configure MCP Servers`）。
2. 在打开的 `cline_mcp_settings.json` 文件中，输入以下配置：

   ```json
   {
     "mcpServers": {
       "revit-mcp": {
         "command": "node",
         "args": [
           "C:\\Users\\你的用户名\\AppData\\Roaming\\npm\\node_modules\\mcp-server-for-revit\\build\\index.js"
         ]
       }
     }
   }
   ```

   > ⚠️ **重要：** 请将路径中的 `你的用户名` 替换为你的实际 Windows 用户名（即 `where.exe mcp-server-for-revit` 返回路径中的用户名）。

3. 按 `Ctrl+S` 保存文件，然后重启 VS Code。

### 7. 启用 Revit 服务

1. 启动 Revit，并打开任意一个项目（`.rvt` 文件）。
2. 在 Revit 顶部菜单栏，找到 **“附加模块”**（Add-Ins）选项卡，你应该能看到 `mcp-servers-for-revit` 的按钮组。
3. 点击 **Settings** 按钮：
   - 在弹出的窗口中，勾选你需要的命令集（**建议全选**）。
   - 点击 **Save** 保存。
4. 点击 **Revit MCP Switch** 按钮，启动 Socket 服务。
   - 如果按钮变为 **绿色** 或显示 “Running”，说明服务启动成功。
   - 如果启动失败，检查是否有防火墙阻拦，或 Revit 是否以管理员权限运行。

### 8. 测试对话

在 VS Code 的 Cline 对话窗口中，输入以下测试指令：

```text
帮我获取当前 Revit 视图中的所有墙。
```

如果 Cline 正常响应并返回了墙的列表或数量，说明配置完全成功！🎉

---

## ❓ 常见问题

如果测试对话时报错或超时，请依次检查：

- Revit 中的 MCP Switch 是否已开启（绿色状态）
- MCP Server 路径配置是否正确
- 网络连接是否正常（API 调用需要网络）
