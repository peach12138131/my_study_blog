# OpenManus 项目简介



# 项目概述


OpenManus 是一个基于大型语言模型(LLM)的智能代理框架，旨在创建能够使用各种工具执行复杂任务的AI助手。该项目采用模块化设计，包含多种代理类型、工具集合和执行流程，使开发者能够构建功能强大的AI应用。


![框架](/post1_openmanus/class_diagram.svg)




开源地址：https://github.com/mannaandpoem/OpenManus/tree/main

# 项目结构

项目主要由以下几个核心模块组成：

1. **Agent模块** - 定义各种类型的智能代理
2. **Tool模块** - 提供各种工具的实现
3. **Flow模块** - 管理代理的执行流程
4. **Prompt模块** - 存储各种代理的提示词模板
5. **基础设施** - 包括配置、日志、异常处理等

## 学习路径

### 1. 从基础设施开始

首先了解项目的基础设施，这些组件为整个项目提供支持：

- **config.py** - 配置管理，包括LLM设置
- **logger.py** - 日志系统
- **exceptions.py** - 异常处理
- **schema.py** - 核心数据模型和结构

这些文件定义了项目的基本数据结构和配置方式，是理解整个项目的基础。

### 2. 理解核心抽象

接下来，了解项目的核心抽象类：

- **llm.py** - LLM接口，处理与语言模型的交互
- **agent/base.py** - 代理的抽象基类
- **tool/base.py** - 工具的抽象基类
- **flow/base.py** - 流程的抽象基类

这些抽象类定义了项目的核心架构和扩展点。

### 3. 探索具体实现

然后，研究各种具体实现：

#### 代理实现
- **agent/react.py** - ReAct代理（推理-行动-观察循环）
- **agent/toolcall.py** - 工具调用代理
- **agent/planning.py** - 规划代理
- **agent/swe.py** - 软件工程代理
- **agent/manus.py** - Manus代理

#### 工具实现
- **tool/python_execute.py** - Python代码执行工具
- **tool/file_saver.py** - 文件保存工具
- **tool/browser_use_tool.py** - 浏览器使用工具
- **tool/google_search.py** - Google搜索工具
- **tool/terminate.py** - 终止工具
- **tool/planning.py** - 规划工具

#### 流程实现
- **flow/flow_factory.py** - 流程工厂
- **flow/planning.py** - 规划流程

### 4. 理解提示词模板

最后，了解各种代理使用的提示词模板：

- **prompt/manus.py**
- **prompt/planning.py**
- **prompt/swe.py**
- **prompt/toolcall.py**

## 依赖层级

![依赖层级](/post1_openmanus/dependency_diagram.svg)

项目的依赖层级可以分为以下几层：

### 底层（基础设施层）
- **config.py**
- **logger.py**
- **exceptions.py**
- **schema.py**

### 中间层（核心功能层）
- **llm.py**
- **tool/base.py**
- **agent/base.py**
- **flow/base.py**

### 上层（具体实现层）
- 各种具体的代理实现
- 各种具体的工具实现
- 各种具体的流程实现

### 应用层
- 最终的应用代理（如ManusAgent、SWEAgent等）

## 执行流程

![执行流程](/post1_openmanus/execution_flow.svg)

OpenManus的典型执行流程如下：

1. **初始化** - 创建代理实例，设置系统提示词
2. **接收输入** - 接收用户输入并添加到代理的内存中
3. **思考/规划** - 代理分析当前状态并决定下一步行动
4. **工具调用** - 如果需要，代理会调用相应的工具
5. **工具执行** - 执行工具并获取结果
6. **处理结果** - 处理工具执行结果并更新代理状态
7. **生成响应** - 生成对用户的响应
8. **判断完成** - 判断任务是否完成，如果未完成则返回步骤3

## 关键概念


### Agent（代理）
代理是框架的核心，负责理解用户需求、规划行动、调用工具和生成响应。不同类型的代理有不同的能力和专长。

### Tool（工具）
工具是代理可以使用的各种功能模块，如Python代码执行、文件保存、网页浏览等。工具提供了代理与外部世界交互的能力。

### Flow（流程）
流程管理多个代理之间的协作，定义了任务执行的整体流程和策略。

### Memory（内存）
内存存储代理的对话历史和状态，使代理能够保持上下文连贯性。

## 扩展建议

如果您想扩展这个框架，可以考虑：

1. **添加新工具** - 实现BaseTool抽象类，添加新的工具功能
2. **创建新代理** - 继承现有代理类或BaseAgent，实现特定领域的代理
3. **定义新流程** - 继承BaseFlow，创建新的执行流程
4. **优化提示词** - 改进现有提示词模板，提高代理性能

## 实践建议

1. **从简单开始** - 先尝试使用现有代理和工具
2. **逐步深入** - 然后尝试修改现有代理的行为
3. **添加新功能** - 最后尝试添加新的工具或代理

## 参考资源

- 项目中的注释和文档字符串
- 各个模块的示例代码
- 相关的LLM和代理系统文献

# OpenManus 项目分析报告

## 1. 项目概述

OpenManus 是一个基于大型语言模型(LLM)的智能助手框架，它提供了一套灵活的架构，用于构建能够执行各种任务的AI助手。该项目的核心功能是通过工具调用(Tool Calling)机制，使AI助手能够与外部系统交互，执行如Python代码、网页浏览、文件操作和信息检索等任务。

### 主要特点

- **模块化架构**：项目采用了清晰的模块化设计，包括Agent、Tool、Flow等核心组件
- **工具调用机制**：支持AI助手调用各种工具来执行具体任务
- **流程控制**：通过Flow组件管理复杂任务的执行流程
- **可扩展性**：易于添加新的Agent类型和Tool实现

## 2. 代码阅读顺序建议

为了更好地理解项目，建议按照以下顺序阅读代码：

### 第一阶段：基础结构和核心概念

1. **`app/__init__.py`** - 项目入口点
2. **`app/config.py`** - 配置管理
3. **`app/schema.py`** - 核心数据结构定义
4. **`app/llm.py`** - 语言模型接口

### 第二阶段：核心抽象类

5. **`app/agent/base.py`** - Agent基类
6. **`app/tool/base.py`** - Tool基类
7. **`app/flow/base.py`** - Flow基类

### 第三阶段：具体实现

8. **`app/agent/react.py`** - ReAct模式Agent
9. **`app/agent/toolcall.py`** - 工具调用Agent
10. **`app/agent/manus.py`** - Manus主Agent实现
11. **`app/tool/tool_collection.py`** - 工具集合
12. **`app/tool/python_execute.py`** 等工具实现
13. **`app/flow/planning.py`** - 规划流程实现

### 第四阶段：提示模板和辅助功能

14. **`app/prompt/`** 目录下的提示模板
15. **`app/exceptions.py`** - 异常处理
16. **`app/logger.py`** - 日志功能

## 3. 代码依赖层级

项目的依赖层级从底层到顶层大致如下：

### 底层基础组件
- **配置管理** (`config.py`)
- **数据结构** (`schema.py`)
- **LLM接口** (`llm.py`)
- **日志和异常** (`logger.py`, `exceptions.py`)

### 中间抽象层
- **基础抽象类** (`agent/base.py`, `tool/base.py`, `flow/base.py`)
- **工具集合** (`tool/tool_collection.py`)

### 上层实现层
- **具体Agent实现** (`agent/react.py`, `agent/toolcall.py`, `agent/manus.py`)
- **具体工具实现** (`tool/python_execute.py`, `tool/browser_use_tool.py` 等)
- **流程实现** (`flow/planning.py`)

### 最上层应用层
- **提示模板** (`prompt/` 目录)
- **流程工厂** (`flow/flow_factory.py`)

## 4. 核心组件详解

### Agent 组件

Agent是项目的核心执行单元，负责决策和执行操作。项目实现了多种Agent：

- **BaseAgent**：所有Agent的抽象基类，提供状态管理、记忆管理等基础功能
- **ReActAgent**：实现ReAct模式的Agent，包含思考(think)和行动(act)两个核心方法
- **ToolCallAgent**：能够执行工具调用的Agent，是Manus的基础
- **Manus**：主要的Agent实现，集成了多种工具，能够处理各种任务

### Tool 组件

Tool是Agent用来执行具体操作的工具：

- **BaseTool**：所有工具的抽象基类
- **ToolCollection**：工具集合，管理多个工具
- **具体工具实现**：
  - PythonExecute：执行Python代码
  - BrowserUseTool：浏览器操作
  - GoogleSearch：网络搜索
  - FileSaver：文件保存
  - Terminate：终止执行

### Flow 组件

Flow负责管理复杂任务的执行流程：

- **BaseFlow**：流程的抽象基类
- **PlanningFlow**：实现基于规划的任务执行流程
- **FlowFactory**：创建不同类型Flow的工厂类

## 5. 关键流程分析

### Agent执行流程

1. Agent通过`run()`方法启动执行
2. 在每个执行步骤中，先调用`think()`方法决定下一步操作
3. 然后调用`act()`方法执行具体操作
4. 重复以上步骤直到达到最大步数或任务完成

### 工具调用流程

1. ToolCallAgent接收用户输入
2. 调用LLM生成包含工具调用的响应
3. 解析工具调用参数
4. 执行相应的工具
5. 将工具执行结果返回给LLM
6. 生成最终响应

### 规划执行流程

1. PlanningFlow接收任务请求
2. 创建初始计划
3. 逐步执行计划中的每个步骤
4. 为每个步骤选择合适的执行Agent
5. 标记步骤完成状态
6. 生成最终执行总结

## 6. 扩展建议

如果您想扩展项目功能，可以考虑：

1. 添加新的Tool实现
2. 创建特定领域的Agent
3. 实现新的Flow类型
4. 优化提示模板


