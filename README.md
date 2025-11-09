## 基于<https://github.com/zhaarey/apple-music-downloader>二次开发

## 1. 项目概述

本项目是一个综合性的后端服务系统，主要通过 Python 实现核心业务逻辑、任务调度、API 服务、邮件处理和通知功能。同时，项目集成了以 Go 语言编写的底层处理模块，用于执行特定的、可能与性能或特定系统级操作相关的任务。Python 应用作为上层调度者，负责管理和调用这些 Go模块，形成一个协同工作的完整系统。
<https://amdl.lyjw131.com>

## 2. 主要功能

* **API 服务**: 提供基于 HTTP 的 API 接口，用于外部系统或客户端的交互。
* **邮件处理**:
    * 定期检查指定邮箱，获取新邮件。
    * 根据邮件内容执行相应处理逻辑。
* **任务调度与管理**:
    * 通过任务队列 (`info/task_queue.json`) 管理待处理任务。
    * Python 脚本负责将任务分发给相应的处理单元，包括调用 Go 程序。
* **通知系统**:
    * 在特定事件发生或任务完成后，通过邮件或其他方式发送通知。
* **日志管理**:
    * 记录系统运行状态、错误信息和重要事件到日志文件 (`logs.log`)。
    * 提供日志管理功能，便于追踪和调试。
* **配置化管理**:
    * 通过 YAML (`config/config.yaml`, `config/source.yaml`, `config/users.yaml`) 和 JSON (`info/errors.json`, `info/api_token.json`) 文件管理系统配置，提高灵活性和可维护性。
* **底层处理能力 (Go)**:
    * 执行由 Go 语言实现的特定处理任务，可能涉及数据解密、内容处理或其他底层操作（具体功能由 Go 模块内部实现）。

## 3. 技术架构

### 3.1. 整体架构

系统采用混合语言架构，以 Python 为核心控制层，Go 为底层执行单元。

* **Python 应用层**:
    * **主控程序 (`main.py`)**: 项目的入口点，负责初始化配置、启动各个服务模块（如 API 服务、邮件检查器、任务处理器等）。
    * **后端服务 (`backend.py`)**: 可能基于 Flask/FastAPI 等框架实现，提供 API 接口，处理业务逻辑，并与其他 Python 模块交互。
    * **业务模块**: 包括 `email_checker.py` (邮件检查)、`notifications.py` (通知发送)、`logs_manager.py` (日志管理) 等。
    * **工具类 (`utils.py`)**: 提供公共函数和辅助方法。
* **Go 执行层**:
    * **Go 主程序 (`go/main.go`)**: 编译后的可执行文件，封装了特定的底层功能。
    * **JavaScript 代理脚本 (`go/agent.js`, `go/agent-arm64.js`)**: 这些 Node.js 脚本可能作为调用 Go 程序的中介。Python 通过执行这些 JS 脚本来间接触发 Go 程序。这种方式可以简化跨平台调用、参数传递或环境设置。
* **数据与配置**:
    * **配置文件**: `config/` 目录下的 YAML 文件。
    * **信息文件**: `info/` 目录下的 JSON 文件，用于存储 API 令牌、错误信息、任务队列等。
    * **日志文件**: `logs.log`。

[图片：项目整体架构图，展示 Python 各模块、Go 模块、配置文件、数据文件之间的关系和数据流向]

### 3.2. Python 与 Go 的交互原理

Python 部分与 Go 部分的交互是本项目的关键之一。由于 Go 模块的具体实现细节未深入分析，我们主要关注其调度接口。

1.  **调用方式**:
    * Python 的 `main.py` 或 `backend.py` (或其他相关业务模块) 会通过标准库 `subprocess` 来执行 Go 程序。
    * 考虑到 `agent.js` 和 `agent-arm64.js` 的存在，Python 更有可能是通过执行 Node.js 命令来间接调用 Go 程序，例如：
        ```python
        import subprocess
        # 假设 agent.js 是调用 Go 程序的入口
        # command 可能包含传递给 Go 程序的参数
        process = subprocess.run(['node', 'server/go/agent.js', 'arg1', 'arg2'], capture_output=True, text=True)
        if process.returncode == 0:
            result = process.stdout
            # 处理 Go 程序的输出
        else:
            error_message = process.stderr
            # 处理错误
        ```
    * `agent-arm64.js` 的存在暗示了系统考虑了不同 CPU 架构 (如 ARM64) 的兼容性，JS 脚本内部可能会根据环境选择或调用不同版本的 Go 可执行文件或进行特定配置。

2.  **数据传递**:
    * **Python to Go**:
        * **命令行参数**: Python 在调用 `agent.js` (或直接调用 Go 可执行文件) 时，可以通过命令行参数传递数据。
        * **配置文件/数据文件**: Python 脚本可能会先生成一个包含任务信息或配置数据的临时文件，然后 Go 程序读取该文件进行处理。`info/task_queue.json` 可能是这种机制的一个体现，Python 将任务写入队列，Go 程序（或通过 JS 代理）读取并执行。
        * **标准输入 (stdin)**: Python 可以将数据通过管道传递给 Go 程序的标准输入。
    * **Go to Python**:
        * **标准输出 (stdout) / 标准错误 (stderr)**: Go 程序执行完毕后，可以通过标准输出返回结果数据，或通过标准错误返回错误信息。Python 的 `subprocess` 模块可以捕获这些输出。
        * **退出码 (Exit Code)**: Go 程序可以通过退出码告知 Python 其执行状态（成功或失败类型）。
        * **输出文件**: Go 程序可以将处理结果写入指定文件，Python 随后读取该文件。

3.  **任务协调**:
    * `info/task_queue.json` 文件可能扮演了异步任务队列的角色。Python 模块作为生产者，将需要 Go 处理的任务信息（如参数、待处理数据路径等）写入此 JSON 文件。
    * 一个独立的 Python 进程或 Go 进程（可能由 Python 启动和监控）会定期检查此队列，取出任务并执行。

## 4. 核心模块详解 (Python)

### 4.1. `main.py`
* **功能**: 项目的主入口和协调器。
* **技术原理**:
    * 加载 `config/config.yaml` 等核心配置文件。
    * 初始化日志系统 (`logs_manager.py`)。
    * 根据配置启动各个组件，例如：
        * 启动 `backend.py` 中的 API 服务 (如果存在)。
        * 启动 `email_checker.py` 中的邮件监控循环。
        * 启动任务处理器，该处理器可能轮询 `info/task_queue.json` 并分发任务（包括调用 Go 模块）。
    * 可能包含主事件循环或守护进程逻辑。

### 4.2. `backend.py`
* **功能**: 提供 API 接口，处理外部请求。
* **技术原理**:
    * 通常使用 Web 框架如 Flask 或 FastAPI。
    * 定义路由 (endpoints) 来接收 HTTP 请求。
    * 对请求进行验证和解析。
    * 调用其他业务逻辑模块 (如 `email_checker.py` 的功能、`notifications.py` 或通过 `subprocess` 调用 Go 程序) 来完成请求处理。
    * 将处理结果格式化为 JSON (或其他格式) 并返回给客户端。
    * 与 `info/api_token.json` 交互，进行 API 认证或调用外部 API。
    * 错误处理依赖 `info/errors.json` 中定义的错误码和信息。

### 4.3. `email_checker.py`
* **功能**: 监控和处理邮件。
* **技术原理**:
    * 使用 `imaplib`、`poplib` 等库连接邮件服务器 (IMAP/POP3)，配置信息来自 `config/config.yaml` 或 `config/source.yaml`。
    * 定期拉取新邮件。
    * 解析邮件内容 (发件人、主题、正文、附件)。
    * 根据预设规则或邮件内容，触发相应的业务逻辑，例如：
        * 将任务信息添加到 `info/task_queue.json`。
        * 直接调用其他 Python 模块。
        * 触发通知 (`notifications.py`)。

### 4.4. `logs_manager.py`
* **功能**: 管理应用程序的日志记录。
* **技术原理**:
    * 使用 Python 内置的 `logging` 模块。
    * 配置日志格式、级别 (DEBUG, INFO, WARNING, ERROR, CRITICAL)。
    * 将日志输出到 `logs.log` 文件。
    * 可能实现日志轮转 (log rotation) 以防止日志文件过大。
    * 提供接口供其他模块调用以记录日志。

### 4.5. `notifications.py`
* **功能**: 发送各种类型的通知。
* **技术原理**:
    * 支持多种通知渠道，最常见的是邮件通知 (使用 `smtplib` 库)。
    * 邮件服务器配置 (SMTP 服务器地址、端口、凭据) 来自 `config/config.yaml`。
    * 提供统一的接口，供其他模块在需要发送通知时调用。例如，任务完成、发生错误、收到特定邮件等。
    * 通知内容可以模板化，根据不同事件动态生成。

### 4.6. `utils.py`
* **功能**: 存放项目中多处用到的公共函数和类。
* **技术原理**:
    * 包含日期时间处理、文件操作、数据格式转换、字符串处理、加解密辅助函数等。
    * 旨在提高代码复用性，避免重复代码。

### 4.7. 配置文件 (`config/`)
* **`config.yaml`**:
    * **作用**: 存储全局性的配置，如应用设置、数据库连接（如果使用）、邮件服务器设置、API 密钥（不推荐直接存储，更安全的做法是使用环境变量或专门的密钥管理服务）、日志级别等。
    * **原理**: Python 通过 `PyYAML` 库解析此文件，将配置加载到内存中供各模块使用。
* **`source.yaml`**:
    * **作用**: 可能用于定义数据源、特定任务的参数来源或其他与外部资源相关的配置。例如，邮件检查的目标邮箱列表、特定 API 的端点 URL 等。
    * **原理**: 与 `config.yaml` 类似，由 Python 解析和使用。
* **`users.yaml`**:
    * **作用**: 可能存储用户信息，如系统用户列表、权限配置、或者与邮件通知相关的用户邮箱等。
    * **原理**: 解析后用于用户认证、权限控制或通知目标确定。

### 4.8. 信息文件 (`info/`)
* **`errors.json`**:
    * **作用**: 定义标准化的错误代码和错误信息。当程序发生特定错误时，可以查询此文件获取对应的错误描述，便于统一错误处理和对外反馈。
    * **原理**: Python 程序在捕获异常或识别到错误条件时，加载此 JSON 文件，根据错误标识符查找具体的错误信息。
* **`api_token.json`**:
    * **作用**: 存储访问外部服务所需的 API 令牌或密钥。这是一种将敏感信息与代码分离的方式，但仍需注意此文件的权限管理。
    * **原理**: 需要调用外部 API 的模块（如 `backend.py`）会读取此文件获取认证令牌。
* **`task_queue.json`**:
    * **作用**: 一个简单的基于文件的任务队列。当需要异步处理或将任务传递给其他进程（尤其是 Go 进程）时，可以将任务描述（如任务类型、参数、数据ID）以 JSON 对象的形式追加到此文件的数组中。
    * **原理**:
        * **生产者**: Python 模块（如 `email_checker.py` 或 `backend.py` 的 API 处理函数）在接收到任务请求后，将任务信息序列化为 JSON 并写入此文件。文件锁机制可能被用于防止并发写入冲突。
        * **消费者**: 另一个 Python 进程或由 `agent.js` 启动的 Go 进程会定期轮询此文件，读取并移除（或标记为已处理）任务，然后执行。

## 5. Python 与 Go 的调度流程示例

假设一个场景：用户通过 API 提交了一个需要 Go 程序处理的数据。

1.  **请求到达**: 客户端向 `backend.py` 提供的 API 端点发送请求，请求中包含待处理的数据或数据标识。
2.  **Python 处理与任务生成**:
    * `backend.py` 接收请求，进行验证和初步处理。
    * 确定此任务需要 Go 模块处理。
    * `backend.py` 将任务参数（如数据ID、处理类型、回调信息等）构造成一个 JSON 对象。
    * 此 JSON 对象被追加到 `info/task_queue.json` 文件中。
3.  **任务拾取与 Go 调用 (可能通过 Python 监控进程)**:
    * 一个独立的 Python 监控进程（或 `main.py` 中的一个线程/协程）定期检查 `info/task_queue.json`。
    * 发现新任务后，读取任务信息。
    * 根据任务信息，准备调用 Go 模块的参数。
    * 通过 `subprocess.run(['node', 'server/go/agent.js', '--task_id', task_id, '--data_path', data_path, ...])` 来执行 `agent.js`。
4.  **`agent.js` 代理执行**:
    * `agent.js` (Node.js 脚本) 解析收到的命令行参数。
    * 它可能进行一些环境设置或参数转换。
    * 最终，`agent.js` 调用编译好的 Go 可执行文件 (`go/main.go` 的产物)，并将必要的参数传递给 Go 程序。
5.  **Go 程序执行**:
    * Go 程序接收参数，执行其核心逻辑（如数据解密、复杂计算等）。
    * 处理完成后，将结果输出到标准输出，或写入一个约定的结果文件。
    * 以特定的退出码结束。
6.  **结果返回与后处理**:
    * `agent.js` 捕获 Go 程序的输出和退出码，可能进行格式化后，再通过自己的标准输出返回。
    * Python 的 `subprocess.run` 调用捕获 `agent.js` 的输出和退出码。
    * Python 监控进程解析结果。
    * 如果成功，可能会更新任务状态，将结果存入数据库或文件，并通过 `notifications.py` 发送完成通知。
    * 如果失败，记录错误到 `logs.log`，并可能发送错误通知。

## 6. 配置

你需要修改 `config/source.yaml` 中的获取/解密服务器地址来连接至你的 AMDL Wrapper