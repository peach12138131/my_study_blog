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

然后将工具添加到prompt，版本更新后似乎无需更改

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


在llm.py中修改ask_tool函数，api错误后返回“try again”,以及重试时间
api错误后返回“try again”


```python
    @retry(
        wait=wait_random_exponential(min=120, max=480),
        stop=stop_after_attempt(10),
        retry=retry_if_exception_type(
            (OpenAIError, Exception, ValueError)
        ),  # Don't retry TokenLimitExceeded
    )
    async def ask_tool(
        self,
        messages: List[Union[dict, Message]],
        system_msgs: Optional[List[Union[dict, Message]]] = None,
        timeout: int = 300,
        tools: Optional[List[dict]] = None,
        tool_choice: TOOL_CHOICE_TYPE = ToolChoice.AUTO,  # type: ignore
        temperature: Optional[float] = None,
        **kwargs,
    ) -> ChatCompletionMessage | None:
        """
        Ask LLM using functions/tools and return the response.

        Args:
            messages: List of conversation messages
            system_msgs: Optional system messages to prepend
            timeout: Request timeout in seconds
            tools: List of tools to use
            tool_choice: Tool choice strategy
            temperature: Sampling temperature for the response
            **kwargs: Additional completion arguments

        Returns:
            ChatCompletionMessage: The model's response

        Raises:
            TokenLimitExceeded: If token limits are exceeded
            ValueError: If tools, tool_choice, or messages are invalid
            OpenAIError: If API call fails after retries
            Exception: For unexpected errors
        """
        try:
            # Validate tool_choice
            if tool_choice not in TOOL_CHOICE_VALUES:
                raise ValueError(f"Invalid tool_choice: {tool_choice}")

            # Check if the model supports images
            supports_images = self.model in MULTIMODAL_MODELS

            # Format messages
            if system_msgs:
                system_msgs = self.format_messages(system_msgs, supports_images)
                messages = system_msgs + self.format_messages(messages, supports_images)
            else:
                messages = self.format_messages(messages, supports_images)

            # Calculate input token count
            input_tokens = self.count_message_tokens(messages)

            # If there are tools, calculate token count for tool descriptions
            tools_tokens = 0
            if tools:
                for tool in tools:
                    tools_tokens += self.count_tokens(str(tool))

            input_tokens += tools_tokens

            # Check if token limits are exceeded
            if not self.check_token_limit(input_tokens):
                error_message = self.get_limit_error_message(input_tokens)
                # Raise a special exception that won't be retried
                raise TokenLimitExceeded(error_message)

            # Validate tools if provided
            if tools:
                for tool in tools:
                    if not isinstance(tool, dict) or "type" not in tool:
                        raise ValueError("Each tool must be a dict with 'type' field")

            # Set up the completion request
            params = {
                "model": self.model,
                "messages": messages,
                "tools": tools,
                "tool_choice": tool_choice,
                "timeout": timeout,
                **kwargs,
            }

            if self.model in REASONING_MODELS:
                params["max_completion_tokens"] = self.max_tokens
            else:
                params["max_tokens"] = self.max_tokens
                params["temperature"] = (
                    temperature if temperature is not None else self.temperature
                )

            response: ChatCompletion = await self.client.chat.completions.create(
                **params, stream=False
            )

            # Check if response is valid
            if not response.choices or not response.choices[0].message:
                print(response)
                # raise ValueError("Invalid or empty response from LLM")
                return None

            # Update token counts
            self.update_token_count(
                response.usage.prompt_tokens, response.usage.completion_tokens
            )

            print(f"测试response.choices[0].message: {response.choices[0].message}")
            return response.choices[0].message

        except TokenLimitExceeded:
            # Re-raise token limit errors without logging
            raise
        except ValueError as ve:
            logger.error(f"Validation error in ask_tool: {ve}")
            raise
        except OpenAIError as oe:
            logger.error(f"OpenAI API error: {oe}")
            if isinstance(oe, AuthenticationError):
                logger.error("Authentication failed. Check API key.")
            elif isinstance(oe, RateLimitError):
                logger.error("Rate limit exceeded. Consider increasing retry attempts.")
            elif isinstance(oe, APIError):
                logger.error(f"API error: {oe}")

                ##添加工具调用后审查
                from openai.types.chat import ChatCompletionMessage
                return ChatCompletionMessage(
                    content="tool use error please check carefully and try again",
                    role="assistant",
                    function_call=None,
                    tool_calls=None
                )
            raise
        except Exception as e:
            logger.error(f"Unexpected error in ask_tool: {e}")
            raise




```



在agent.manus.py中可以修改执行步骤