# MCP-Scan: An MCP Security Scanner

<a href="https://discord.gg/dZuZfhKnJ4"><img src="https://img.shields.io/discord/1265409784409231483?style=plastic&logo=discord&color=blueviolet&logoColor=white" height=18/></a>

MCP-Scan is a security scanning tool designed to go over your installed MCP servers and check them for common security vulnerabilities like [prompt injections](https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks), [tool poisoning](https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks) and [cross-origin escalations](https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks).

## 代码组织与架构

### 核心模块结构

MCP-Scan项目采用模块化设计，核心模块如下：

```mermaid
graph TD
    A[MCP-Scan] --> B[cli.py 命令行界面]
    A --> C[MCPScanner.py 扫描引擎]
    A --> D[models.py 数据模型]
    A --> E[suppressIO.py IO控制]
    A --> F[version.py 版本信息]
    
    B --> C
    C --> D
    C --> E
```

### 组件关系与通信流程

```mermaid
graph TD
    subgraph 前端界面
        CLI[命令行接口 cli.py]
    end
    
    subgraph 核心引擎
        Scanner[MCPScanner类]
        Storage[StorageFile类]
        Format[格式化函数]
    end
    
    subgraph 数据层
        Models[数据模型 models.py]
        ConfigFiles[配置文件处理]
    end
    
    subgraph 外部通信
        Server[MCP服务器]
        API[验证API]
    end
    
    CLI --> |调用| Scanner
    Scanner --> |读取/保存| Storage
    Scanner --> |验证| API
    Scanner --> |连接| Server
    Scanner --> |格式化输出| Format
    Scanner --> |解析| ConfigFiles
    ConfigFiles --> |使用| Models
```

### 主要组件功能

1. **命令行接口 (cli.py)**
   - 处理用户命令行参数
   - 根据命令创建MCPScanner实例并调用相应方法

2. **扫描引擎 (MCPScanner.py)**
   - MCPScanner类：提供扫描、检查和白名单管理的核心功能
   - StorageFile类：管理本地存储的工具信息和白名单
   - 格式化函数：生成格式化的输出文本

3. **数据模型 (models.py)**
   - 定义配置文件结构的Pydantic模型
   - 支持多种客户端配置格式(Claude、VSCode等)
   - 服务器类型模型(SSE和Stdio)

4. **IO控制 (suppressIO.py)**
   - 控制MCP服务器输出的抑制

### 组件间通信

- **用户 → CLI**：通过命令行参数向程序传达意图
- **CLI → Scanner**：将参数传递给Scanner类
- **Scanner → 配置文件**：读取并解析配置文件
- **Scanner → MCP服务器**：使用mcp库连接服务器并获取工具信息
- **Scanner → 验证API**：将工具描述发送到验证服务器检查安全问题
- **Scanner → Storage**：检查工具描述变化和白名单状态
- **Scanner → 控制台**：格式化输出扫描结果

### 数据流向

```mermaid
flowchart LR
    A[用户] -->|命令| B[CLI]
    B -->|参数| C[Scanner]
    C -->|读取| D[配置文件]
    D -->|解析结果| C
    C -->|连接| E[MCP服务器]
    E -->|工具信息| C
    C -->|验证| F[验证API]
    F -->|安全评估| C
    C -->|存储| G[本地文件]
    G -->|历史数据| C
    C -->|结果| A
```

## 工作流程

以下时序图展示了MCP-Scan工具的工作流程：

### 扫描功能工作流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant CLI as 命令行接口
    participant Scanner as MCPScanner
    participant Storage as 存储文件
    participant Server as MCP服务器
    participant API as 验证API

    User->>CLI: 执行扫描命令
    CLI->>Scanner: 创建Scanner实例并调用start()
    
    loop 对每个配置文件路径
        Scanner->>Scanner: scan_config_file(path)
        Scanner->>+Server: 连接并检索工具、提示和资源
        Server-->>-Scanner: 返回工具、提示和资源列表
        
        loop 对每个工具
            Scanner->>+API: verify_server(tools, prompts, resources)
            API-->>-Scanner: 返回验证结果
            
            Scanner->>+Storage: check_and_update(server_name, tool, verified)
            Storage-->>-Scanner: 返回变更信息和历史数据
            
            Scanner->>+Storage: is_whitelisted(tool)
            Storage-->>-Scanner: 返回工具是否在白名单中
            
            Scanner->>Scanner: 格式化并输出工具信息
        end
        
        Scanner->>Scanner: 检测跨来源引用攻击
    end
    
    Scanner->>Storage: save()
    Scanner-->>CLI: 完成扫描
    CLI-->>User: 显示扫描结果
```

### 白名单功能工作流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant CLI as 命令行接口
    participant Scanner as MCPScanner
    participant Storage as 存储文件
    participant API as 验证API

    alt 添加工具到白名单
        User->>CLI: 执行whitelist命令(name, hash)
        CLI->>Scanner: whitelist(name, hash, local_only)
        Scanner->>Storage: add_to_whitelist(name, hash)
        Scanner->>Storage: save()
        
        alt !local_only
            Scanner->>API: upload_whitelist_entry(name, hash)
        end
        
        Scanner->>Storage: print_whitelist()
        Storage-->>Scanner: 白名单条目
        Scanner-->>CLI: 显示白名单信息
        CLI-->>User: 显示白名单结果
    else 重置白名单
        User->>CLI: 执行whitelist --reset命令
        CLI->>Scanner: reset_whitelist()
        Scanner->>Storage: reset_whitelist()
        Scanner->>Storage: save()
        Scanner-->>CLI: 完成重置
        CLI-->>User: 显示重置成功信息
    else 显示白名单
        User->>CLI: 执行whitelist命令(无参数)
        CLI->>Scanner: print_whitelist()
        Scanner->>Storage: print_whitelist()
        Storage-->>Scanner: 白名单条目
        Scanner-->>CLI: 显示白名单信息
        CLI-->>User: 显示当前白名单
    end
```

### 检查功能工作流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant CLI as 命令行接口
    participant Scanner as MCPScanner
    participant Server as MCP服务器
    participant Storage as 存储文件

    User->>CLI: 执行inspect命令
    CLI->>Scanner: 创建Scanner实例并调用inspect()
    
    loop 对每个配置文件路径
        Scanner->>Scanner: scan_config_file(path)
        Scanner->>+Server: 连接并检索工具、提示和资源
        Server-->>-Scanner: 返回工具、提示和资源列表
        
        loop 对每个工具/提示/资源
            Scanner->>Scanner: format_inspect_tool_line(item)
            Scanner->>Scanner: 输出格式化信息
        end
    end
    
    Scanner->>Storage: save()
    Scanner-->>CLI: 完成检查
    CLI-->>User: 显示检查结果
```

## Quick Start
To run MCP-Scan, use the following command:

```bash
uvx mcp-scan@latest
```

### Example Output
![mcp-scan-output](https://invariantlabs.ai/images/mcp-scan-output.png)


## Features

- Scanning of Claude, Cursor, Windsurf, and other file-based MCP client configurations
- Scanning for prompt injection attacks in tool descriptions and [tool poisoning attacks](https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks) using [Invariant Guardrails](https://github.com/invariantlabs-ai/invariant?tab=readme-ov-file#analyzer)
- Detection of cross-origin escalation attacks ([tool shadowing](https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks))
- _Tool Pinning_ to detect and prevent [MCP rug pull attacks](https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks), i.e. detects changes to MCP tools via hashing
- Inspecting the tool descriptions of installed tools via `uvx mcp-scan@latest inspect`

## How It Works
MCP-Scan searches through your configuration files to find MCP server configurations. It connects to these servers and retrieves tool descriptions.

It then scans tool descriptions, both with local checks and by invoking Invariant Guardrailing via an API. For this, tool names and descriptions are shared with invariantlabs.ai. By using MCP-Scan, you agree to the invariantlabs.ai [terms of use](https://explorer.invariantlabs.ai/terms) and [privacy policy](https://invariantlabs.ai/privacy-policy).

Invariant Labs is collecting data for security research purposes (only about tool descriptions and how they change over time, not your user data). Don't use MCP-scan if you don't want to share your tools.

MCP-scan does not store or log any usage data, i.e. the contents and results of your MCP tool calls.

## CLI parameters

```

usage: uvx mcp-scan@latest [--storage-file STORAGE_FILE] [--base-url BASE_URL]

help
    Prints this help message

whitelist 
    Whitelists a tool, or prints the current whitelist if no arguments are provided

    [NAME HASH]
        Adds tool name and hash to whitelist
        
    [--reset]
        Resets the whitelist
    
    [--local-only]
        Do not contribute to the global whitelist.
        Defaults to False
    
scan 
    Scan MCP servers
    Default command, when no arguments are provided

    [FILE1] [FILE2] [FILE3] ...
        Different file locations to scan. This can include custom file locations as long as they are in an expected format, including Claude, Cursor or VSCode format.
        Defaults to well known locations, depending on your OS
        
    [--checks-per-server CHECKS_PER_SERVER]
        Number of checks to perform on each server, values greater than 1 help catch non-deterministic behavior
        Defaults to 1
    [--server-timeout SERVER_TIMEOUT]
        Number of seconds to wait while trying a mcp server
        Defaults to 10
    [--suppress-mcpserver-io]
        Suppress the output of the mcp server
        Defaults to True

inspect
    Prints the tool descriptions of the installed tools

    [FILE1] [FILE2] [FILE3] ...
        Different file locations to scan. This can include custom file locations as long as they are in an expected format, including Claude, Cursor or VSCode format.
        Defaults to well known locations, depending on your OS

    [--server-timeout SERVER_TIMEOUT]
        Number of seconds to wait while trying a mcp server
        Defaults to 10
    [--suppress-mcpserver-io]
        Suppress the output of the mcp server
        Defaults to True
```

## Contributing

We welcome contributions to MCP-Scan. If you have suggestions, bug reports, or feature requests, please open an issue on our GitHub repository.

## Development Setup
To run this package from source, follow these steps:

```
uv run pip install -e .
uv run -m src.mcp_scan.cli
```

## Including MCP-scan results in your own project / registry

If you want to include MCP-scan results in your own project or registry, please reach out to the team via `mcpscan@invariantlabs.ai`, and we can help you with that.

## Further Reading
- [Introducing MCP-Scan](https://invariantlabs.ai/blog/introducing-mcp-scan)
- [MCP Security Notification Tool Poisoning Attacks](https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks)
- [WhatsApp MCP Exploited](https://invariantlabs.ai/blog/whatsapp-mcp-exploited)
- [MCP Prompt Injection](https://simonwillison.net/2025/Apr/9/mcp-prompt-injection/)

## Changelog
- `0.1.4.0` initial public release
- `0.1.4.1` `inspect` command, reworked output
- `0.1.4.2` added SSE support
- `0.1.4.3` added VSCode MCP support, better support for non-MacOS, improved error handling, better output formatting
- `0.1.4.4-5` fixes
- `0.1.4.6` whitelisting of tools
