---
title: "OpenManus 如何添加工具"
date: 2025-03-13
lastmod: 2025-03-13
draft: false
author: "openmanus"
comment: true


categories: ["Agent"]
tags: ["Agent", "OpenManus"]
featuredImage: ""
featuredImagePreview: ""
lightgallery: true

---

# OpenManus 工具添加指南

本指南将详细介绍如何为 OpenManus 智能体添加新工具。我们将以添加一个简单的计算器工具为例，展示整个流程。

## 工具添加流程概述

添加一个新工具到 OpenManus 智能体需要完成以下三个主要步骤：

1. **创建工具类文件**：在 `./app/tool/` 目录下创建工具的实现文件
2. **注册工具到智能体**：在 `./app/agent/manus.py` 中导入并添加工具
3. **更新提示词**：在 `./app/prompt/manus.py` 中更新提示词，添加工具的描述

下面我们将详细介绍每个步骤。

## 步骤 1：创建工具类文件

首先，我们需要在 `./app/tool/` 目录下创建一个新的 Python 文件来实现我们的工具。以计算器工具为例，我们创建 `calculator.py`。

### calculator.py 的结构

```python
from typing import Union, Dict, Any
from app.tool.base import BaseTool


class Calculator(BaseTool):
    name: str = "calculator"
    description: str = """Perform basic mathematical operations including addition, subtraction, multiplication, and division.
                 Use this tool when you need to perform numerical calculations. The tool accepts two numeric values and an operation, and returns the calculation result."""

    parameters: dict = {
        "type": "object",
        "properties": {
            "first_number": {
                "type": "number",
                "description": "(required) 第一个数值操作数。",
            },
            "second_number": {
                "type": "number",
                "description": "(required) 第二个数值操作数。",
            },
            "operation": {
                "type": "string",
                "description": "(required) 要执行的数学运算。",
                "enum": ["add", "subtract", "multiply", "divide"],
            },
        },
        "required": ["first_number", "second_number", "operation"],
    }
    
    async def execute(
        self, 
        first_number: Union[int, float], 
        second_number: Union[int, float], 
        operation: str
    ) -> Dict[str, Any]:
        """
        执行基本的数学计算操作。
        
        Args:
            first_number (Union[int, float]): 第一个数值操作数。
            second_number (Union[int, float]): 第二个数值操作数。
            operation (str): 要执行的数学运算，可以是 "add"、"subtract"、"multiply" 或 "divide"。
        
        Returns:
            Dict[str, Any]: 包含计算结果和运算说明的字典。
        """
        try:
            if operation == "add":
                result = first_number + second_number
                expression = f"{first_number} + {second_number}"
            elif operation == "subtract":
                result = first_number - second_number
                expression = f"{first_number} - {second_number}"
            elif operation == "multiply":
                result = first_number * second_number
                expression = f"{first_number} × {second_number}"
            elif operation == "divide":
                if second_number == 0:
                    raise ValueError("除数不能为零")
                result = first_number / second_number
                expression = f"{first_number} ÷ {second_number}"
            else:
                raise ValueError(f"不支持的操作: {operation}")
            
            return {
                "result": result,
                "expression": expression,
                "operation": operation
            }
        
        except Exception as e:
            return {
                "error": str(e),
                "first_number": first_number,
                "second_number": second_number,
                "operation": operation
            }
```

### 工具类的关键组成部分

1. **继承 BaseTool**：所有工具都必须继承自 `BaseTool` 基类
2. **定义工具属性**：
   - `name`：工具的唯一标识符
   - `description`：工具的详细描述，会显示在 AI 的工具选择界面
   - `parameters`：定义工具接受的参数，使用 JSONSchema 格式
3. **实现 execute 方法**：工具的核心逻辑，接收参数并返回结果

## 步骤 2：注册工具到智能体

创建好工具类后，我们需要将其导入并添加到 Manus 智能体的可用工具列表中。这需要修改 `./app/agent/manus.py` 文件。

### 修改 manus.py

```python
from pydantic import Field

from app.agent.toolcall import ToolCallAgent
from app.prompt.manus import NEXT_STEP_PROMPT, SYSTEM_PROMPT
from app.tool import Terminate, ToolCollection
from app.tool.browser_use_tool import BrowserUseTool
from app.tool.file_saver import FileSaver
from app.tool.google_search import GoogleSearch
from app.tool.python_execute import PythonExecute
# 导入新工具
from app.tool.calculator import Calculator

class Manus(ToolCallAgent):
    """
    A versatile general-purpose agent that uses planning to solve various tasks.

    This agent extends PlanningAgent with a comprehensive set of tools and capabilities,
    including Python execution, web browsing, file operations, and information retrieval
    to handle a wide range of user requests.
    """

    name: str = "Manus"
    description: str = (
        "A versatile agent that can solve various tasks using multiple tools"
    )

    system_prompt: str = SYSTEM_PROMPT
    next_step_prompt: str = NEXT_STEP_PROMPT

    # 在工具集合中添加新工具
    available_tools: ToolCollection = Field(
        default_factory=lambda: ToolCollection(
            PythonExecute(), GoogleSearch(), BrowserUseTool(), FileSaver(), Terminate(), Calculator()
        )
    )
```

### 关键修改点

1. **导入工具类**：添加 `from app.tool.calculator import Calculator` 导入语句
2. **添加到工具集合**：在 `ToolCollection` 中添加 `Calculator()` 实例

## 步骤 3：更新提示词

最后，我们需要更新 `./app/prompt/manus.py` 中的提示词，添加关于新工具的描述，以便 AI 知道何时使用这个工具。

### 修改 manus.py (prompt)

```python
SYSTEM_PROMPT = "You are OpenManus, an all-capable AI assistant, aimed at solving any task presented by the user. You have various tools at your disposal that you can call upon to efficiently complete complex requests. Whether it's programming, information retrieval, file processing, or web browsing, you can handle it all."

NEXT_STEP_PROMPT = """You can interact with the computer using PythonExecute, save important content and information files through FileSaver, open browsers with BrowserUseTool, and retrieve information using GoogleSearch. You can also perform mathematical calculations using Calculator.

PythonExecute: Execute Python code to interact with the computer system, data processing, automation tasks, etc.

FileSaver: Save files locally, such as txt, py, html, etc.

BrowserUseTool: Open, browse, and use web browsers. If you open a local HTML file, you must provide the absolute path to the file.

GoogleSearch: Perform web information retrieval.

Calculator: Perform basic mathematical operations including addition, subtraction, multiplication, and division.

Based on user needs, proactively select the most appropriate tool or combination of tools. For complex tasks, you can break down the problem and use different tools step by step to solve it. After using each tool, clearly explain the execution results and suggest the next steps.
"""
```

### 关键修改点

1. **添加工具描述**：在 `NEXT_STEP_PROMPT` 中添加 Calculator 工具的简要描述
2. **更新工具列表**：在开头的概述中提及新添加的 Calculator 工具

## 总结

通过以上三个步骤，我们成功地为 OpenManus 智能体添加了一个新的计算器工具。这个过程可以概括为：

1. 创建工具类文件，实现工具逻辑
2. 将工具注册到智能体的可用工具列表中
3. 更新提示词，添加工具的描述信息

这个流程适用于添加任何新工具到 OpenManus 智能体中。