---
title: "OpenManus添加推荐工具，修改观察长度"
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

# OpenManus 添加推荐工具，修改观察长度
1. **创建工具类文件**：在 `./app/tool/` 目录下创建工具的实现文件
2. **注册工具到智能体**：在 `./app/agent/manus.py` 中导入并添加工具
3. **更新提示词**：在 `./app/prompt/manus.py` 中更新提示词，添加工具的描述


先修改config.toml
# Global LLM configuration
[llm]
model = "claude-3-7-sonnet-20250219"
base_url = "https://api.anthropic.com/v1"
api_key = ""
max_tokens = 8192
temperature = 0.0

# Optional configuration for specific LLM models
[llm.vision]
model = "claude-3-7-sonnet-20250219"
base_url = "https://api.anthropic.com/v1"
api_key = ""


## 添加推荐工具

```python
import os
import json
import aiohttp
from typing import List, Dict, Any
from app.tool.base import BaseTool


class SupplierRecommend(BaseTool):
    name: str = "supplier_recommend"
    description: str = """
    Get supplier recommendations for one or more flight routes based on departure and arrival airports.
    This tool makes an API call to retrieve recommended suppliers for given routes.
    """
    parameters: dict = {
        "type": "object",
        "properties": {
            "routes": {
                "type": "array",
                "description": "(required) List of routes to get recommendations for. Each route must contain 'dep_icao' and 'arr_icao'.",
                "items": {
                    "type": "object",
                    "properties": {
                        "dep_icao": {
                            "type": "string",
                            "description": "Departure airport ICAO code (e.g., 'ZSSS')",
                        },
                        "arr_icao": {
                            "type": "string",
                            "description": "Arrival airport ICAO code (e.g., 'ZBAA')",
                        },
                    },
                    "required": ["dep_icao", "arr_icao"],
                },
            },
        },
        "required": ["routes"],
    }
    
    async def execute(self, routes: List[Dict[str, str]]) -> str:
        """
        Get supplier recommendations for one or more flight routes.
        
        Args:
            routes (List[Dict[str, str]]): List of routes, each containing 'dep_icao' and 'arr_icao'.
                Example: [{"dep_icao": "ZSSS", "arr_icao": "ZBAA"}, {"dep_icao": "ZSPD", "arr_icao": "ZGBH"}]
        
        Returns:
            str: A JSON string containing the supplier recommendations or an error message.
        """
        try:
            # API endpoint for supplier recommendations
            api_url = "http://devai.avi-go.com/avi-go-recommend/supplier-recommend"
            
            # Validate input
            for route in routes:
                if not isinstance(route, dict):
                    return f"Error: Each route must be a dictionary. Got {type(route)} instead."
                
                if "dep_icao" not in route or "arr_icao" not in route:
                    return "Error: Each route must contain 'dep_icao' and 'arr_icao' keys."
                
                if not isinstance(route["dep_icao"], str) or not isinstance(route["arr_icao"], str):
                    return "Error: ICAO codes must be strings."
                
                if len(route["dep_icao"]) != 4 or len(route["arr_icao"]) != 4:
                    return "Error: ICAO codes must be 4 characters long."
            
            # Make API request
            async with aiohttp.ClientSession() as session:
                async with session.post(api_url, json=routes) as response:
                    if response.status != 200:
                        return f"API Error: Status code {response.status}, {await response.text()}"
                    
                    result = await response.json()
                    return json.dumps(result, ensure_ascii=False)
        
        except aiohttp.ClientError as e:
            return f"Connection error: {str(e)}"
        except json.JSONDecodeError:
            return "Error: Failed to parse API response as JSON"
        except Exception as e:
            return f"Error getting supplier recommendations: {str(e)}"
        


```





然后在智能体中添加工具，修改步数，增加观察文本长度

```python

from typing import Any

from pydantic import Field

from app.agent.toolcall import ToolCallAgent
from app.prompt.manus import NEXT_STEP_PROMPT, SYSTEM_PROMPT
from app.tool import Terminate, ToolCollection
from app.tool.browser_use_tool import BrowserUseTool
from app.tool.file_saver import FileSaver
from app.tool.python_execute import PythonExecute
from app.tool.web_search import WebSearch
from app.tool.supplier_recommend import SupplierRecommend


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

    max_observe: int = 5000
    max_steps: int = 80

    # Add general-purpose tools to the tool collection
    available_tools: ToolCollection = Field(
        default_factory=lambda: ToolCollection(
            PythonExecute(), WebSearch(), BrowserUseTool(), FileSaver(), Terminate(),SupplierRecommend()
        )
    )

    async def _handle_special_tool(self, name: str, result: Any, **kwargs):
        if not self._is_special_tool(name):
            return
        else:
            await self.available_tools.get_tool(BrowserUseTool().name).cleanup()
            await super()._handle_special_tool(name, result, **kwargs)


```

然后将工具添加到prompt

```python

SYSTEM_PROMPT = "You are AviGo, an all-capable AI assistant, aimed at solving any task presented by the user. You have various tools at your disposal that you can call upon to efficiently complete complex requests. Whether it's programming, information retrieval, file processing, or web browsing, you can handle it all."

NEXT_STEP_PROMPT = """You can interact with the computer using PythonExecute, save important content and information files through FileSaver, open browsers with BrowserUseTool, and retrieve information using GoogleSearch.

PythonExecute: Execute Python code to interact with the computer system, data processing, automation tasks, etc.

FileSaver: Save files locally, such as txt, py, html, etc.

BrowserUseTool: Open, browse, and use web browsers.If you open a local HTML file, you must provide the absolute path to the file.

WebSearch: Perform web information retrieval.

SupplierRecommend: Get private jet charters supplier recommendations for one or more flight routes based on departure and arrival airports. 

Terminate: End the current interaction when the task is complete or when you need additional information from the user. Use this tool to signal that you've finished addressing the user's request or need clarification before proceeding further.

Based on user needs, proactively select the most appropriate tool or combination of tools. For complex tasks, you can break down the problem and use different tools step by step to solve it. After using each tool, clearly explain the execution results and suggest the next steps.

Always maintain a helpful, informative tone throughout the interaction. If you encounter any limitations or need more details, clearly communicate this to the user before terminating.
"""




```