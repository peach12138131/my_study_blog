---
title: "OpenManus前端安装"
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

感谢开源原项目地址：https://github.com/mannaandpoem/OpenManus.git


## 安装前端第一步，下载前端分支

```bash
git clone -b front-end https://github.com/mannaandpoem/OpenManus.git
```

## 安装前端第二步，安装依赖，环境

同之前安装方法一致


## 安装前端第三步，修改配置文件,config/config.toml



``` python
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

# Server configuration,端口看还剩下哪一些更改一下
[server]
host = "localhost"
port = 5172

```

# 重要步骤，替换app.py为以下代码

``` python
import asyncio
import os
import threading
import tomllib
import uuid
import webbrowser
from datetime import datetime
from functools import partial
from json import dumps
from pathlib import Path

from fastapi import Body, FastAPI, HTTPException, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import (
    FileResponse,
    HTMLResponse,
    JSONResponse,
    StreamingResponse,
)
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates
from pydantic import BaseModel


app = FastAPI()

app.mount("/static", StaticFiles(directory="static"), name="static")
templates = Jinja2Templates(directory="templates")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


class Task(BaseModel):
    id: str
    prompt: str
    created_at: datetime
    status: str
    steps: list = []

    def model_dump(self, *args, **kwargs):
        data = super().model_dump(*args, **kwargs)
        data["created_at"] = self.created_at.isoformat()
        return data


class TaskManager:
    def __init__(self):
        self.tasks = {}
        self.queues = {}
        self.process_tasks = {}  # 存储任务进程，用于停止任务

    def create_task(self, prompt: str) -> Task:
        task_id = str(uuid.uuid4())
        task = Task(
            id=task_id, prompt=prompt, created_at=datetime.now(), status="pending"
        )
        self.tasks[task_id] = task
        self.queues[task_id] = asyncio.Queue()
        return task

    async def update_task_step(
        self, task_id: str, step: int, result: str, step_type: str = "step"
    ):
        if task_id in self.tasks:
            task = self.tasks[task_id]
            task.steps.append({"step": step, "result": result, "type": step_type})
            await self.queues[task_id].put(
                {"type": step_type, "step": step, "result": result}
            )
            await self.queues[task_id].put(
                {"type": "status", "status": task.status, "steps": task.steps}
            )

    async def complete_task(self, task_id: str):
        if task_id in self.tasks:
            task = self.tasks[task_id]
            task.status = "completed"
            await self.queues[task_id].put(
                {"type": "status", "status": task.status, "steps": task.steps}
            )
            await self.queues[task_id].put({"type": "complete"})

    async def fail_task(self, task_id: str, error: str):
        if task_id in self.tasks:
            self.tasks[task_id].status = f"failed: {error}"
            await self.queues[task_id].put({"type": "error", "message": error})

     # 在类的其他方法后添加以下新方法
    async def stop_task(self, task_id: str):
        """停止正在运行的任务"""
        if task_id in self.tasks:
            task = self.tasks[task_id]
            if task.status == "running":
                task.status = "stopped"
                await self.queues[task_id].put(
                    {"type": "status", "status": task.status, "steps": task.steps}
                )
                await self.queues[task_id].put(
                    {"type": "stopped", "message": "Task was manually stopped"}
                )
                
                # 如果有关联的进程，尝试终止它
                if task_id in self.process_tasks and self.process_tasks[task_id]:
                    try:
                        self.process_tasks[task_id].cancel()
                    except Exception as e:
                        print(f"Error stopping task {task_id}: {str(e)}")
                    
                    self.process_tasks.pop(task_id, None)
                
                return True
        return False

    def clear_history(self, password: str):
        """清除所有历史任务记录"""
        if password == "25131":  # 硬编码密码进行验证
            # 保留当前运行的任务
            running_tasks = {
                task_id: task for task_id, task in self.tasks.items() 
                if task.status == "running"
            }
            
            # 保留队列中的正在运行的任务
            running_queues = {
                task_id: queue for task_id, queue in self.queues.items()
                if task_id in running_tasks
            }
            
            # 清除其他任务
            self.tasks = running_tasks
            self.queues = running_queues
            return True
        return False


task_manager = TaskManager()


@app.get("/", response_class=HTMLResponse)
async def index(request: Request):
    return templates.TemplateResponse("index.html", {"request": request})


@app.get("/download")
async def download_file(file_path: str):
    if not os.path.exists(file_path):
        raise HTTPException(status_code=404, detail="File not found")

    return FileResponse(file_path, filename=os.path.basename(file_path))


@app.post("/tasks")
async def create_task(prompt: str = Body(..., embed=True)):
    task = task_manager.create_task(prompt)
    asyncio.create_task(run_task(task.id, prompt))
    return {"task_id": task.id}


from app.agent.manus import Manus


async def run_task(task_id: str, prompt: str):
    try:
        task_manager.tasks[task_id].status = "running"

        current_task = asyncio.current_task()
        task_manager.process_tasks[task_id] = current_task

        agent = Manus(
            name="Manus",
            description="A versatile agent that can solve various tasks using multiple tools",
        )

        async def on_think(thought):
            await task_manager.update_task_step(task_id, 0, thought, "think")

        async def on_tool_execute(tool, input):
            await task_manager.update_task_step(
                task_id, 0, f"Executing tool: {tool}\nInput: {input}", "tool"
            )

        async def on_action(action):
            await task_manager.update_task_step(
                task_id, 0, f"Executing action: {action}", "act"
            )

        async def on_run(step, result):
            await task_manager.update_task_step(task_id, step, result, "run")

        from app.logger import logger

        class SSELogHandler:
            def __init__(self, task_id):
                self.task_id = task_id

            async def __call__(self, message):
                import re

                # Extract - Subsequent Content
                cleaned_message = re.sub(r"^.*? - ", "", message)

                event_type = "log"
                if "✨ Manus's thoughts:" in cleaned_message:
                    event_type = "think"
                elif "🛠️ Manus selected" in cleaned_message:
                    event_type = "tool"
                elif "🎯 Tool" in cleaned_message:
                    event_type = "act"
                elif "📝 Oops!" in cleaned_message:
                    event_type = "error"
                elif "🏁 Special tool" in cleaned_message:
                    event_type = "complete"

                await task_manager.update_task_step(
                    self.task_id, 0, cleaned_message, event_type
                )

        sse_handler = SSELogHandler(task_id)
        logger.add(sse_handler)

        result = await agent.run(prompt + f"\n\nThe initial directory is: ./workspace/{datetime.now().strftime('%Y-%m-%d_%H-%M')}")
        await task_manager.update_task_step(task_id, 1, result, "result")
        await task_manager.complete_task(task_id)

    except asyncio.CancelledError:
        await task_manager.update_task_step(task_id, 0, "Task was manually stopped", "stopped")
        task_manager.tasks[task_id].status = "stopped"
    except Exception as e:
        await task_manager.fail_task(task_id, str(e))
    finally:
        # 清理进程引用
        if task_id in task_manager.process_tasks:
            task_manager.process_tasks.pop(task_id, None)


@app.get("/tasks/{task_id}/events")
async def task_events(task_id: str):
    async def event_generator():
        if task_id not in task_manager.queues:
            yield f"event: error\ndata: {dumps({'message': 'Task not found'})}\n\n"
            return

        queue = task_manager.queues[task_id]

        task = task_manager.tasks.get(task_id)
        if task:
            yield f"event: status\ndata: {dumps({'type': 'status', 'status': task.status, 'steps': task.steps})}\n\n"

        while True:
            try:
                event = await queue.get()
                formatted_event = dumps(event)

                yield ": heartbeat\n\n"

                if event["type"] == "complete":
                    yield f"event: complete\ndata: {formatted_event}\n\n"
                    break
                elif event["type"] == "error":
                    yield f"event: error\ndata: {formatted_event}\n\n"
                    break
                elif event["type"] == "step":
                    task = task_manager.tasks.get(task_id)
                    if task:
                        yield f"event: status\ndata: {dumps({'type': 'status', 'status': task.status, 'steps': task.steps})}\n\n"
                    yield f"event: {event['type']}\ndata: {formatted_event}\n\n"
                elif event["type"] in ["think", "tool", "act", "run"]:
                    yield f"event: {event['type']}\ndata: {formatted_event}\n\n"
                else:
                    yield f"event: {event['type']}\ndata: {formatted_event}\n\n"

            except asyncio.CancelledError:
                print(f"Client disconnected for task {task_id}")
                break
            except Exception as e:
                print(f"Error in event stream: {str(e)}")
                yield f"event: error\ndata: {dumps({'message': str(e)})}\n\n"
                break

    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",
        },
    )


@app.get("/tasks")
async def get_tasks():
    sorted_tasks = sorted(
        task_manager.tasks.values(), key=lambda task: task.created_at, reverse=True
    )
    return JSONResponse(
        content=[task.model_dump() for task in sorted_tasks],
        headers={"Content-Type": "application/json"},
    )


@app.get("/tasks/{task_id}")
async def get_task(task_id: str):
    if task_id not in task_manager.tasks:
        raise HTTPException(status_code=404, detail="Task not found")
    return task_manager.tasks[task_id]


@app.get("/config/status")
async def check_config_status():
    config_path = Path(__file__).parent / "config" / "config.toml"
    example_config_path = Path(__file__).parent / "config" / "config.example.toml"

    if config_path.exists():
        return {"status": "exists"}
    elif example_config_path.exists():
        try:
            with open(example_config_path, "rb") as f:
                example_config = tomllib.load(f)
            return {"status": "missing", "example_config": example_config}
        except Exception as e:
            return {"status": "error", "message": str(e)}
    else:
        return {"status": "no_example"}


@app.post("/config/save")
async def save_config(config_data: dict = Body(...)):
    try:
        config_dir = Path(__file__).parent / "config"
        config_dir.mkdir(exist_ok=True)

        config_path = config_dir / "config.toml"

        toml_content = ""

        if "llm" in config_data:
            toml_content += "# Global LLM configuration\n[llm]\n"
            llm_config = config_data["llm"]
            for key, value in llm_config.items():
                if key != "vision":
                    if isinstance(value, str):
                        toml_content += f'{key} = "{value}"\n'
                    else:
                        toml_content += f"{key} = {value}\n"

        if "server" in config_data:
            toml_content += "\n# Server configuration\n[server]\n"
            server_config = config_data["server"]
            for key, value in server_config.items():
                if isinstance(value, str):
                    toml_content += f'{key} = "{value}"\n'
                else:
                    toml_content += f"{key} = {value}\n"

        with open(config_path, "w", encoding="utf-8") as f:
            f.write(toml_content)

        return {"status": "success"}
    except Exception as e:
        return {"status": "error", "message": str(e)}





@app.exception_handler(Exception)
async def generic_exception_handler(request: Request, exc: Exception):
    return JSONResponse(
        status_code=500, content={"message": f"Server error: {str(exc)}"}
    )




@app.post("/tasks/{task_id}/stop")
async def stop_task(task_id: str):
    """停止正在运行的任务"""
    result = await task_manager.stop_task(task_id)
    if result:
        return {"status": "success", "message": "Task stopped successfully"}
    else:
        raise HTTPException(status_code=404, detail="Task not found or already completed")

@app.post("/tasks/clear-history")
async def clear_task_history(password: str = Body(..., embed=True)):
    """清除所有历史任务记录"""
    result = task_manager.clear_history(password)
    if result:
        return {"status": "success", "message": "Task history cleared successfully"}
    else:
        raise HTTPException(status_code=403, detail="Invalid password")



def open_local_browser(config):
    webbrowser.open_new_tab(f"http://{config['host']}:{config['port']}")


def load_config():
    try:
        config_path = Path(__file__).parent / "config" / "config.toml"

        if not config_path.exists():
            return {"host": "localhost", "port": 5172}

        with open(config_path, "rb") as f:
            config = tomllib.load(f)

        return {"host": config["server"]["host"], "port": config["server"]["port"]}
    except FileNotFoundError:
        return {"host": "localhost", "port": 5172}
    except KeyError as e:
        print(
            f"The configuration file is missing necessary fields: {str(e)}, use default configuration"
        )
        return {"host": "localhost", "port": 5172}


if __name__ == "__main__":
    import uvicorn

    config = load_config()
    open_with_config = partial(open_local_browser, config)
    threading.Timer(3, open_with_config).start()
    uvicorn.run(app, host=config["host"], port=config["port"])

```


# 重要步骤，替换templates/index.html为以下代码

``` html

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>OpenManus Local Version</title>
    <link rel="stylesheet" href="/static/style.css">
</head>
<body>
    <div class="container">
        <div class="history-panel">
            <div class="history-header">
                <h2>History Tasks</h2>
                <button id="clear-history-btn" class="clear-history-btn">Clear History</button>
            </div>
            <div id="task-list" class="task-list"></div>
        </div>

        <div class="main-panel">
            <div id="task-container" class="task-container">
                <div class="welcome-message">
                    <h1>Welcome to OpenManus Local Version</h1>
                    <p>Please enter a task prompt to start a new task</p>
                </div>
                <div id="log-container" class="step-container"></div>
            </div>

            <div id="input-container" class="input-container">
                <input
                    type="text"
                    id="prompt-input"
                    placeholder="Enter task prompt..."
                    onkeypress="if(event.keyCode === 13) createTask()"
                >
                <button onclick="createTask()">Create Task</button>
                <button id="stop-task-btn" class="stop-task-btn hidden">Stop Task</button>
            </div>
        </div>
    </div>

    <div id="status-bar" class="status-bar"></div>

    <!-- 原有的配置对话框 -->
    <div id="config-modal" class="config-modal">
        <div class="config-modal-content">
            <div class="config-modal-header">
                <h2>System Configuration</h2>
                <p>Please fill in the necessary configuration information to continue using the system</p>
            </div>

            <div class="config-modal-body">
                <div class="note-box">
                    <p>⚠️ Please ensure that the following configuration information is correct.</p>
                    <p>If you do not have an API key, please obtain one from the corresponding AI service provider.</p>
                </div>

                <div class="config-section">
                    <h3>LLM Configuration</h3>
                    <div class="form-group">
                        <label for="llm-model">Model Name <span class="required-mark">*</span></label>
                        <input type="text" id="llm-model" name="llm.model" placeholder="For example: claude-3-5-sonnet">
                    </div>
                    <div class="form-group">
                        <label for="llm-base-url">API Base URL <span class="required-mark">*</span></label>
                        <input type="text" id="llm-base-url" name="llm.base_url" placeholder="For example: https://api.openai.com/v1">
                    </div>
                    <div class="form-group">
                        <label for="llm-api-key">API Key <span class="required-mark">*</span></label>
                        <input type="password" id="llm-api-key" name="llm.api_key" placeholder="Your API key, for example: sk-...">
                        <span class="field-help">Must be your own valid API key, not the placeholder in the example</span>
                    </div>
                    <div class="form-group">
                        <label for="llm-max-tokens">Max Tokens</label>
                        <input type="number" id="llm-max-tokens" name="llm.max_tokens" placeholder="For example: 4096">
                    </div>
                    <div class="form-group">
                        <label for="llm-temperature">Temperature</label>
                        <input type="number" id="llm-temperature" name="llm.temperature" step="0.1" placeholder="For example: 0.0">
                    </div>
                </div>

                <div class="config-section">
                    <h3>Server Configuration</h3>
                    <div class="form-group">
                        <label for="server-host">Host <span class="required-mark">*</span></label>
                        <input type="text" id="server-host" name="server.host" placeholder="For example: localhost">
                    </div>
                    <div class="form-group">
                        <label for="server-port">Port <span class="required-mark">*</span></label>
                        <input type="number" id="server-port" name="server.port" placeholder="For example: 5172">
                    </div>
                </div>
            </div>

            <div class="config-modal-footer">
                <p id="config-error" class="config-error"></p>
                <div class="config-actions">
                    <button id="save-config-btn" class="primary-btn">Save Configuration</button>
                </div>
            </div>
        </div>
    </div>

    <!-- 添加历史记录删除密码对话框 -->
    <div id="password-modal" class="modal">
        <div class="modal-content">
            <div class="modal-header">
                <h3>Enter Password to Clear History</h3>
                <span class="close-modal" id="close-password-modal">&times;</span>
            </div>
            <div class="modal-body">
                <div class="form-group">
                    <label for="history-password">Password:</label>
                    <input type="password" id="history-password">
                </div>
                <p id="password-error" class="error-message hidden"></p>
            </div>
            <div class="modal-footer">
                <button id="confirm-clear-history" class="primary-btn">Confirm</button>
                <button id="cancel-clear-history" class="secondary-btn">Cancel</button>
            </div>
        </div>
    </div>

    <script src="/static/main.js"></script>
</body>
</html>

```

# 重要步骤，替换static/style.css为以下代码

``` css

:root {
    --primary-color: #007bff;
    --primary-hover: #0056b3;
    --success-color: #28a745;
    --error-color: #dc3545;
    --warning-color: #ff9800;
    --info-color: #2196f3;
    --text-color: #333;
    --text-light: #666;
    --bg-color: #f8f9fa;
    --border-color: #ddd;
}

* {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
}

body {
    font-family: 'PingFang SC', 'Microsoft YaHei', sans-serif;
    margin: 0;
    padding: 0;
    background-color: var(--bg-color);
    color: var(--text-color);
}

.container {
    display: flex;
    min-height: 100vh;
    min-width: 0;
    width: 90%;
    margin: 0 auto;
    padding: 20px;
    gap: 20px;
}

.card {
    background-color: white;
    border-radius: 8px;
    padding: 20px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.history-panel {
    width: 300px;
}

.main-panel {
    flex: 1;
    display: flex;
    flex-direction: column;
    gap: 20px;
}

.task-list {
    margin-top: 10px;
    max-height: calc(100vh - 160px);
    overflow-y: auto;
    overflow-x: hidden;
}

.task-container {
    @extend .card;
    width: 100%;
    max-width: 100%;
    position: relative;
    min-height: 300px;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    overflow: auto;
    height: 100%;
}

.welcome-message {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    text-align: center;
    color: var(--text-light);
    background: white;
    z-index: 1;
    padding: 20px;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.1);
    width: 100%;
    height: 100%;
}

.welcome-message h1 {
    font-size: 2rem;
    margin-bottom: 10px;
    color: var(--text-color);
}

.input-container {
    @extend .card;
    display: flex;
    gap: 10px;
}

#prompt-input {
    flex: 1;
    padding: 12px;
    border: 1px solid #ddd;
    border-radius: 4px;
    font-size: 1rem;
}

button {
    padding: 12px 24px;
    background-color: var(--primary-color);
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 1rem;
    transition: background-color 0.2s;
}

button:hover {
    background-color: var(--primary-hover);
}

.task-item {
    padding: 10px;
    margin-bottom: 10px;
    background-color: #f8f9fa;
    border-radius: 4px;
    cursor: pointer;
    transition: background-color 0.2s;
}

.task-item:hover {
    background-color: #e9ecef;
}

.task-item.active {
    background-color: var(--primary-color);
    color: white;
}

#input-container.bottom {
    margin-top: auto;
}

.task-card {
    background: #fff;
    padding: 15px;
    margin-bottom: 10px;
    border-radius: 8px;
    cursor: pointer;
    transition: all 0.2s;
}

.task-card:hover {
    transform: translateX(5px);
    box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.status-pending {
    color: var(--text-light);
}

.status-running {
    color: var(--primary-color);
}

.status-completed {
    color: var(--success-color);
}

.status-failed {
    color: var(--error-color);
}

.step-container {
    display: flex;
    flex-direction: column;
    gap: 10px;
    width: 100%;
    max-height: calc(100vh - 200px);
    overflow-y: auto;
    max-width: 100%;
    overflow-x: hidden;
}

.step-item {
    padding: 15px;
    background: white;
    border-radius: 8px;
    width: 100%;
    box-shadow: 0 2px 4px rgba(0,0,0,0.05);
    margin-bottom: 10px;
    opacity: 1;
    transform: none;
}

.step-item .log-line:not(.result) {
    opacity: 0.7;
    color: #666;
    font-size: 0.9em;
}

.step-item .log-line.result {
    opacity: 1;
    color: #333;
    font-size: 1em;
    background: #e8f5e9;
    border-left: 4px solid #4caf50;
    padding: 10px;
    border-radius: 4px;
}

.step-item.show {
    opacity: 1;
    transform: none;
}

.log-line {
    padding: 10px;
    border-radius: 4px;
    margin-bottom: 10px;
    display: flex;
    flex-direction: column;
    gap: 4px;
}

.log-line.think,
.step-item pre.think {
    background: var(--info-color-light);
    border-left: 4px solid var(--info-color);
}

.log-line.tool,
.step-item pre.tool {
    background: var(--warning-color-light);
    border-left: 4px solid var(--warning-color);
}

.log-line.result,
.step-item pre.result {
    background: var(--success-color-light);
    border-left: 4px solid var(--success-color);
}

.log-line.error,
.step-item pre.error {
    background: var(--error-color-light);
    border-left: 4px solid var(--error-color);
}

.log-line.info,
.step-item pre.info {
    background: var(--bg-color);
    border-left: 4px solid var(--text-light);
}

.log-prefix {
    font-weight: bold;
    white-space: nowrap;
    margin-bottom: 5px;
    color: #666;
}

.step-item pre {
    padding: 10px;
    border-radius: 4px;
    margin-left: 20px;
    overflow-x: hidden;
    font-family: 'Courier New', monospace;
    font-size: 0.9em;
    line-height: 1.4;
    white-space: pre-wrap;
    word-wrap: break-word;
    word-break: break-all;
    max-width: 100%;
    color: var(--text-color);
    background: var(--bg-color);

    &.log {
        background: var(--bg-color);
        border-left: 4px solid var(--text-light);
    }
    &.think {
        background: var(--info-color-light);
        border-left: 4px solid var(--info-color);
    }
    &.tool {
        background: var(--warning-color-light);
        border-left: 4px solid var(--warning-color);
    }
    &.result {
        background: var(--success-color-light);
        border-left: 4px solid var(--success-color);
    }
}

/* Step division line style */
.step-divider {
    display: flex;
    align-items: center;
    width: 100%;
    margin: 15px 0;
    position: relative;
}

.step-circle {
    width: 36px;
    height: 36px;
    border-radius: 50%;
    background-color: var(--primary-color);
    color: white;
    display: flex;
    align-items: center;
    justify-content: center;
    font-weight: bold;
    font-size: 1.2em;
    box-shadow: 0 2px 5px rgba(0,0,0,0.2);
    z-index: 2;
    flex-shrink: 0;
}

/* File interaction style */
.file-interaction {
    margin-top: 15px;
    padding: 10px;
    border-radius: 6px;
    background-color: #f5f7fa;
    border: 1px solid var(--border-color);
}

.download-link {
    display: inline-block;
    padding: 8px 16px;
    background-color: var(--primary-color);
    color: white;
    text-decoration: none;
    border-radius: 4px;
    font-size: 0.9em;
    transition: background-color 0.2s;
}

.download-link:hover {
    background-color: var(--primary-hover);
}

.preview-image {
    max-width: 100%;
    max-height: 200px;
    margin-bottom: 10px;
    border-radius: 4px;
    cursor: pointer;
    transition: transform 0.2s;
}

.preview-image:hover {
    transform: scale(1.02);
}

.audio-player {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.audio-player audio {
    width: 100%;
    margin-bottom: 10px;
}

.run-button {
    display: inline-block;
    padding: 8px 16px;
    background-color: var(--success-color);
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 0.9em;
    margin-right: 10px;
    transition: background-color 0.2s;
}

.run-button:hover {
    background-color: #218838;
}

/* Full screen image modal box */
.image-modal {
    display: none;
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.8);
    z-index: 1000;
    justify-content: center;
    align-items: center;
}

.image-modal.active {
    display: flex;
}

.modal-content {
    max-width: 90%;
    max-height: 90%;
}

.close-modal {
    position: absolute;
    top: 20px;
    right: 30px;
    color: white;
    font-size: 30px;
    cursor: pointer;
}

/* Python runs simulation modal boxes */
.python-modal {
    display: none;
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.8);
    z-index: 1000;
    justify-content: center;
    align-items: center;
}

.python-modal.active {
    display: flex;
}

.python-console {
    width: 80%;
    height: 80%;
    background-color: #1e1e1e;
    color: #f8f8f8;
    border-radius: 8px;
    padding: 15px;
    font-family: 'Courier New', monospace;
    overflow: auto;
}

.python-output {
    white-space: pre-wrap;
    line-height: 1.5;
}

.step-line {
    flex-grow: 1;
    height: 2px;
    background-color: var(--border-color);
    margin-left: 10px;
}

.step-info {
    margin-left: 15px;
    font-weight: bold;
    color: var(--text-light);
    font-size: 0.9em;
}

.step-item strong {
    display: block;
    margin-bottom: 8px;
    color: #007bff;
    font-size: 0.9em;
}

.step-item div {
    color: #444;
    line-height: 1.6;
}

.loading {
    padding: 15px;
    color: #666;
    text-align: center;
}

.ping {
    color: #ccc;
    text-align: center;
    margin: 5px 0;
}

.error {
    color: #dc3545;
    padding: 10px;
    background: #ffe6e6;
    border-radius: 4px;
    margin: 10px 0;
}

.complete {
    color: #28a745;
    padding: 10px;
    background: #e6ffe6;
    border-radius: 4px;
    margin: 10px 0;
}

pre {
    margin: 0;
    white-space: pre-wrap;
    word-break: break-word;
}

.complete pre {
    max-width: 100%;
    white-space: pre-wrap;
    word-break: break-word;
}

.config-modal {
    display: none;
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.5);
    z-index: 2000;
    justify-content: center;
    align-items: center;
}

.config-modal.active {
    display: flex;
}

.config-modal-content {
    background-color: white;
    border-radius: 8px;
    box-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
    width: 80%;
    max-width: 800px;
    max-height: 90vh;
    overflow-y: auto;
    display: flex;
    flex-direction: column;
}

.config-modal-header {
    padding: 20px;
    border-bottom: 1px solid var(--border-color);
    background-color: var(--info-color);
    color: white;
    border-top-left-radius: 8px;
    border-top-right-radius: 8px;
}

.config-modal-header h2 {
    margin-bottom: 10px;
    font-size: 1.8rem;
}

.config-modal-body {
    padding: 20px;
    overflow-y: auto;
    max-height: calc(90vh - 140px);
}

.config-modal-footer {
    padding: 20px;
    border-top: 1px solid var(--border-color);
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.config-section {
    margin-bottom: 25px;
    padding-bottom: 20px;
    border-bottom: 1px solid var(--border-color);
}

.config-section:last-child {
    border-bottom: none;
}

.config-section h3 {
    margin-bottom: 15px;
    color: var(--info-color);
}

.form-group {
    margin-bottom: 15px;
    transition: all 0.3s ease;
}

.form-group label {
    display: block;
    margin-bottom: 5px;
    font-weight: bold;
    color: var(--text-color);
}

.form-group input {
    width: 100%;
    padding: 10px;
    border: 1px solid var(--border-color);
    border-radius: 4px;
    font-size: 0.9rem;
    transition: border-color 0.3s ease, box-shadow 0.3s ease;
}

.form-group input:focus {
    outline: none;
    border-color: var(--info-color);
    box-shadow: 0 0 0 2px rgba(0, 123, 255, 0.25);
}

.config-actions {
    display: flex;
    gap: 10px;
}

.primary-btn {
    background-color: var(--info-color);
    transition: background-color 0.3s ease;
}

.primary-btn:hover {
    background-color: #0069d9;
}

.secondary-btn {
    background-color: var(--text-light);
}

.config-error {
    color: var(--error-color);
    margin-right: 15px;
    font-weight: bold;
    padding: 8px 0;
    transition: all 0.3s ease;
}

.form-group.error input {
    border-color: var(--error-color);
    box-shadow: 0 0 0 2px rgba(220, 53, 69, 0.25);
    background-color: rgba(220, 53, 69, 0.05);
}

.form-group.error label {
    color: var(--error-color);
}

.form-group .error-message {
    color: var(--error-color);
    font-size: 0.8rem;
    margin-top: 5px;
    display: block;
    animation: errorAppear 0.3s ease;
}

@keyframes errorAppear {
    from {
        opacity: 0;
        transform: translateY(-5px);
    }
    to {
        opacity: 1;
        transform: translateY(0);
    }
}

.note-box {
    background-color: #fff8e1;
    border-left: 4px solid #ffc107;
    padding: 15px;
    margin-bottom: 20px;
    border-radius: 4px;
}

.note-box p {
    margin: 0 0 8px 0;
    color: #856404;
}

.note-box p:last-child {
    margin-bottom: 0;
}

.field-help {
    font-size: 0.8rem;
    color: #666;
    margin-top: 4px;
    display: block;
}

.required-mark {
    color: var(--error-color);
    font-weight: bold;
}

/* 停止任务按钮样式 */
.stop-task-btn {
    background-color: var(--error-color);
    color: white;
}

.stop-task-btn:hover {
    background-color: #bd2130;
}

.hidden {
    display: none !important;
}

/* 任务停止状态样式 */
.stopped {
    color: #ff9800;
    padding: 10px;
    background: #fff3e0;
    border-radius: 4px;
    margin: 10px 0;
}

.status-stopped {
    color: #ff9800;
}

/* 状态栏样式 */
.status-bar {
    position: fixed;
    bottom: 10px;
    right: 10px;
    background-color: rgba(255, 255, 255, 0.8);
    padding: 5px 10px;
    border-radius: 4px;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
    z-index: 100;
}

/* 历史记录面板标题栏样式 */
.history-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 15px;
}

.clear-history-btn {
    padding: 5px 10px;
    font-size: 0.8rem;
    background-color: var(--error-color);
}

.clear-history-btn:hover {
    background-color: #bd2130;
}

/* 密码对话框样式 */
.modal {
    display: none;
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.5);
    z-index: 1000;
    justify-content: center;
    align-items: center;
}

.modal.active {
    display: flex;
}

.modal-content {
    background-color: white;
    border-radius: 8px;
    box-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
    width: 400px;
    max-width: 90%;
}

.modal-header {
    padding: 15px;
    border-bottom: 1px solid var(--border-color);
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.modal-header h3 {
    margin: 0;
    color: var(--text-color);
}

.modal-body {
    padding: 15px;
}

.modal-footer {
    padding: 15px;
    border-top: 1px solid var(--border-color);
    display: flex;
    justify-content: flex-end;
    gap: 10px;
}

.error-message {
    color: var(--error-color);
    margin-top: 10px;
    font-size: 0.9em;
}

.secondary-btn {
    background-color: #6c757d;
}

.secondary-btn:hover {
    background-color: #5a6268;
}


```

# 重要步骤，替换static/main.js为以下代码

``` js
let currentEventSource = null;
let currentTaskId = null; // 添加变量跟踪当前任务ID
let exampleApiKey = '';

// 显示和隐藏停止任务按钮
function toggleStopTaskButton(show) {
    const stopButton = document.getElementById('stop-task-btn');
    if (stopButton) {
        if (show) {
            stopButton.classList.remove('hidden');
        } else {
            stopButton.classList.add('hidden');
        }
    }
}

// 停止当前运行的任务
function stopCurrentTask() {
    if (!currentTaskId) {
        console.warn('No active task to stop');
        return;
    }
    
    fetch(`/tasks/${currentTaskId}/stop`, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        }
    })
    .then(response => {
        if (!response.ok) {
            return response.json().then(err => { throw new Error(err.detail || 'Failed to stop task') });
        }
        return response.json();
    })
    .then(data => {
        console.log('Task stopped:', data);
        // 界面更新会通过SSE事件自动处理
    })
    .catch(error => {
        console.error('Error stopping task:', error);
        alert(`Failed to stop task: ${error.message}`);
    });
}

// 显示密码对话框
function showPasswordModal() {
    const modal = document.getElementById('password-modal');
    if (modal) {
        modal.classList.add('active');
        document.getElementById('history-password').value = '';
        document.getElementById('password-error').classList.add('hidden');
        document.getElementById('history-password').focus();
    }
}

// 隐藏密码对话框
function hidePasswordModal() {
    const modal = document.getElementById('password-modal');
    if (modal) {
        modal.classList.remove('active');
    }
}

// 清除历史记录
function clearTaskHistory() {
    const password = document.getElementById('history-password').value;
    
    fetch('/tasks/clear-history', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({ password })
    })
    .then(response => {
        if (!response.ok) {
            return response.json().then(err => { throw new Error(err.detail || 'Failed to clear history') });
        }
        return response.json();
    })
    .then(data => {
        hidePasswordModal();
        loadHistory(); // 重新加载历史记录
        alert('Task history cleared successfully');
    })
    .catch(error => {
        console.error('Error clearing history:', error);
        const errorElem = document.getElementById('password-error');
        errorElem.textContent = `Error: ${error.message}`;
        errorElem.classList.remove('hidden');
    });
}

function checkConfigStatus() {
    fetch('/config/status')
    .then(response => response.json())
    .then(data => {
        if (data.status === 'missing') {
            showConfigModal(data.example_config);
        } else if (data.status === 'no_example') {
            alert('Error: Missing configuration example file! Please ensure that the config/config.example.toml file exists.');
        } else if (data.status === 'error') {
            alert('Configuration check error:' + data.message);
        }
    })
    .catch(error => {
        console.error('Configuration check failed:', error);
    });
}

// Display configuration pop-up and fill in sample configurations
function showConfigModal(exampleConfig) {
    const configModal = document.getElementById('config-modal');
    if (!configModal) return;

    configModal.classList.add('active');

    if (exampleConfig) {
        fillConfigForm(exampleConfig);
    }

    const saveButton = document.getElementById('save-config-btn');
    if (saveButton) {
        saveButton.onclick = saveConfig;
    }
}

// Use example configuration to fill in the form
function fillConfigForm(exampleConfig) {
    if (exampleConfig.llm) {
        const llm = exampleConfig.llm;

        setInputValue('llm-model', llm.model);
        setInputValue('llm-base-url', llm.base_url);
        setInputValue('llm-api-key', llm.api_key);

        exampleApiKey = llm.api_key || '';

        setInputValue('llm-max-tokens', llm.max_tokens);
        setInputValue('llm-temperature', llm.temperature);
    }

    if (exampleConfig.server) {
        setInputValue('server-host', exampleConfig.server.host);
        setInputValue('server-port', exampleConfig.server.port);
    }
}

function setInputValue(id, value) {
    const input = document.getElementById(id);
    if (input && value !== undefined) {
        input.value = value;
    }
}

function saveConfig() {
    const configData = collectFormData();

    const requiredFields = [
        { id: 'llm-model', name: 'Model Name' },
        { id: 'llm-base-url', name: 'API Base URL' },
        { id: 'llm-api-key', name: 'API Key' },
        { id: 'server-host', name: 'Server Host' },
        { id: 'server-port', name: 'Server Port' }
    ];

    let missingFields = [];
    requiredFields.forEach(field => {
        if (!document.getElementById(field.id).value.trim()) {
            missingFields.push(field.name);
        }
    });

    if (missingFields.length > 0) {
        document.getElementById('config-error').textContent =
            `Please fill in the necessary configuration information: ${missingFields.join(', ')}`;
        return;
    }

    // Check if the API key is the same as the example configuration
    const apiKey = document.getElementById('llm-api-key').value.trim();
    if (apiKey === exampleApiKey && exampleApiKey.includes('sk-')) {
        document.getElementById('config-error').textContent =
            `Please enter your own API key`;
        document.getElementById('llm-api-key').parentElement.classList.add('error');
        return;
    } else {
        document.getElementById('llm-api-key').parentElement.classList.remove('error');
    }

    fetch('/config/save', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify(configData)
    })
    .then(response => response.json())
    .then(data => {
        if (data.status === 'success') {
            document.getElementById('config-modal').classList.remove('active');

            alert('Configuration saved successfully! The application will use the new configuration on next startup.');

            window.location.reload();
        } else {
            document.getElementById('config-error').textContent =
                `Save failed: ${data.message}`;
        }
    })
    .catch(error => {
        document.getElementById('config-error').textContent =
            `Request error: ${error.message}`;
    });
}

// Collect form data
function collectFormData() {
    const configData = {
        llm: {
            model: document.getElementById('llm-model').value,
            base_url: document.getElementById('llm-base-url').value,
            api_key: document.getElementById('llm-api-key').value
        },
        server: {
            host: document.getElementById('server-host').value,
            port: parseInt(document.getElementById('server-port').value || '5172')
        }
    };

    const maxTokens = document.getElementById('llm-max-tokens').value;
    if (maxTokens) {
        configData.llm.max_tokens = parseInt(maxTokens);
    }

    const temperature = document.getElementById('llm-temperature').value;
    if (temperature) {
        configData.llm.temperature = parseFloat(temperature);
    }

    return configData;
}

function createTask() {
    const promptInput = document.getElementById('prompt-input');
    const prompt = promptInput.value.trim();

    if (!prompt) {
        alert("Please enter a valid prompt");
        promptInput.focus();
        return;
    }

    if (currentEventSource) {
        currentEventSource.close();
        currentEventSource = null;
    }

    const container = document.getElementById('task-container');
    container.innerHTML = '<div class="loading">Initializing task...</div>';
    document.getElementById('input-container').classList.add('bottom');

    fetch('/tasks', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({ prompt })
    })
    .then(response => {
        if (!response.ok) {
            return response.json().then(err => { throw new Error(err.detail || 'Request failed') });
        }
        return response.json();
    })
    .then(data => {
        if (!data.task_id) {
            throw new Error('Invalid task ID');
        }
        currentTaskId = data.task_id; // 保存当前任务ID
        toggleStopTaskButton(true); // 显示停止按钮
        setupSSE(data.task_id);
        loadHistory();
        promptInput.value = '';
    })
    .catch(error => {
        container.innerHTML = `<div class="error">Error: ${error.message}</div>`;
        console.error('Failed to create task:', error);
        toggleStopTaskButton(false); // 隐藏停止按钮
    });
}

function setupSSE(taskId) {
    let retryCount = 0;
    const maxRetries = 3;
    const retryDelay = 2000;
    let lastResultContent = '';

    const container = document.getElementById('task-container');

    function connect() {
        const eventSource = new EventSource(`/tasks/${taskId}/events`);
        currentEventSource = eventSource;

        let heartbeatTimer = setInterval(() => {
            container.innerHTML += '<div class="ping">·</div>';
        }, 5000);

        // Initial polling
        fetch(`/tasks/${taskId}`)
            .then(response => response.json())
            .then(task => {
                updateTaskStatus(task);
            })
            .catch(error => {
                console.error('Initial status fetch failed:', error);
            });

        const handleEvent = (event, type) => {
            clearInterval(heartbeatTimer);
            try {
                const data = JSON.parse(event.data);
                container.querySelector('.loading')?.remove();
                container.classList.add('active');

                const stepContainer = ensureStepContainer(container);
                const { formattedContent, timestamp } = formatStepContent(data, type);
                const step = createStepElement(type, formattedContent, timestamp);

                stepContainer.appendChild(step);
                autoScroll(stepContainer);

                fetch(`/tasks/${taskId}`)
                    .then(response => response.json())
                    .then(task => {
                        updateTaskStatus(task);
                    })
                    .catch(error => {
                        console.error('Status update failed:', error);
                    });
            } catch (e) {
                console.error(`Error handling ${type} event:`, e);
            }
        };

        const eventTypes = ['think', 'tool', 'act', 'log', 'run', 'message'];
        eventTypes.forEach(type => {
            eventSource.addEventListener(type, (event) => handleEvent(event, type));
        });

        // 处理任务停止事件
        eventSource.addEventListener('stopped', (event) => {
            clearInterval(heartbeatTimer);
            try {
                const data = JSON.parse(event.data);
                container.innerHTML += `
                    <div class="stopped">
                        <div>⚠️ Task stopped manually</div>
                        <pre>${data.message || ''}</pre>
                    </div>
                `;

                fetch(`/tasks/${taskId}`)
                    .then(response => response.json())
                    .then(task => {
                        updateTaskStatus(task);
                    })
                    .catch(error => {
                        console.error('Status update after stop failed:', error);
                    });

                eventSource.close();
                currentEventSource = null;
                currentTaskId = null;
                toggleStopTaskButton(false); // 隐藏停止按钮
            } catch (e) {
                console.error('Error handling stopped event:', e);
            }
        });

        eventSource.addEventListener('complete', (event) => {
            clearInterval(heartbeatTimer);
            try {
                const data = JSON.parse(event.data);
                lastResultContent = data.result || '';

                container.innerHTML += `
                    <div class="complete">
                        <div>✅ Task completed</div>
                        <pre>${lastResultContent}</pre>
                    </div>
                `;

                fetch(`/tasks/${taskId}`)
                    .then(response => response.json())
                    .then(task => {
                        updateTaskStatus(task);
                    })
                    .catch(error => {
                        console.error('Final status update failed:', error);
                    });

                eventSource.close();
                currentEventSource = null;
                currentTaskId = null;
                toggleStopTaskButton(false); // 隐藏停止按钮
            } catch (e) {
                console.error('Error handling complete event:', e);
            }
        });

        eventSource.addEventListener('error', (event) => {
            clearInterval(heartbeatTimer);
            try {
                let errorMessage = "Unknown error";
                if (event.data) {
                    try {
                        const data = JSON.parse(event.data);
                        errorMessage = data.message || errorMessage;
                    } catch (parseError) {
                        errorMessage = event.data;
                    }
                }
                
                container.innerHTML += `
                    <div class="error">
                        ❌ Error: ${errorMessage}
                    </div>
                `;
                eventSource.close();
                currentEventSource = null;
                currentTaskId = null;
                toggleStopTaskButton(false); // 隐藏停止按钮
            } catch (e) {
                console.error('Error handling failed:', e);
            }
        });

        eventSource.onerror = (err) => {
            if (eventSource.readyState === EventSource.CLOSED) return;

            console.error('SSE connection error:', err);
            clearInterval(heartbeatTimer);
            eventSource.close();

            fetch(`/tasks/${taskId}`)
                .then(response => response.json())
                .then(task => {
                    if (task.status === 'completed' || task.status.startsWith('failed') || task.status === 'stopped') {
                        updateTaskStatus(task);
                        if (task.status === 'completed') {
                            container.innerHTML += `
                                <div class="complete">
                                    <div>✅ Task completed</div>
                                </div>
                            `;
                        } else if (task.status === 'stopped') {
                            container.innerHTML += `
                                <div class="stopped">
                                    <div>⚠️ Task stopped manually</div>
                                </div>
                            `;
                        } else {
                            container.innerHTML += `
                                <div class="error">
                                    ❌ Error: ${task.status.replace('failed: ', '') || 'Task failed'}
                                </div>
                            `;
                        }
                        currentTaskId = null;
                        toggleStopTaskButton(false); // 隐藏停止按钮
                    } else if (retryCount < maxRetries) {
                        retryCount++;
                        container.innerHTML += `
                            <div class="warning">
                                ⚠ Connection lost, retrying in ${retryDelay/1000} seconds (${retryCount}/${maxRetries})...
                            </div>
                        `;
                        setTimeout(connect, retryDelay);
                    } else {
                        container.innerHTML += `
                            <div class="error">
                                ⚠ Connection lost, please try refreshing the page
                            </div>
                        `;
                        currentTaskId = null;
                        toggleStopTaskButton(false); // 隐藏停止按钮
                    }
                })
                .catch(error => {
                    console.error('Task status check failed:', error);
                    if (retryCount < maxRetries) {
                        retryCount++;
                        setTimeout(connect, retryDelay);
                    } else {
                        currentTaskId = null;
                        toggleStopTaskButton(false); // 隐藏停止按钮
                    }
                });
        };
    }

    connect();
}

function loadHistory() {
    fetch('/tasks')
    .then(response => {
        if (!response.ok) {
            return response.text().then(text => {
                throw new Error(`request failure: ${response.status} - ${text.substring(0, 100)}`);
            });
        }
        return response.json();
    })
    .then(tasks => {
        const listContainer = document.getElementById('task-list');
        listContainer.innerHTML = tasks.map(task => `
            <div class="task-card" data-task-id="${task.id}">
                <div>${task.prompt}</div>
                <div class="task-meta">
                    ${new Date(task.created_at).toLocaleString()} -
                    <span class="status status-${task.status ? task.status.toLowerCase() : 'unknown'}">
                        ${task.status || 'Unknown state'}
                    </span>
                </div>
            </div>
        `).join('');
    })
    .catch(error => {
        console.error('Failed to load history records:', error);
        const listContainer = document.getElementById('task-list');
        listContainer.innerHTML = `<div class="error">Load Fail: ${error.message}</div>`;
    });
}


function ensureStepContainer(container) {
    let stepContainer = container.querySelector('.step-container');
    if (!stepContainer) {
        container.innerHTML = '<div class="step-container"></div>';
        stepContainer = container.querySelector('.step-container');
    }
    return stepContainer;
}

function formatStepContent(data, eventType) {
    return {
        formattedContent: data.result,
        timestamp: new Date().toLocaleTimeString()
    };
}

function createStepElement(type, content, timestamp) {
    const step = document.createElement('div');

    // Executing step
    const stepRegex = /Executing step (\d+)\/(\d+)/;
    if (type === 'log' && stepRegex.test(content)) {
        const match = content.match(stepRegex);
        const currentStep = parseInt(match[1]);
        const totalSteps = parseInt(match[2]);

        step.className = 'step-divider';
        step.innerHTML = `
            <div class="step-circle">${currentStep}</div>
            <div class="step-line"></div>
            <div class="step-info">${currentStep}/${totalSteps}</div>
        `;
    } else if (type === 'act') {
        // Check if it contains information about file saving
        const saveRegex = /Content successfully saved to (.+)/;
        const match = content.match(saveRegex);

        step.className = `step-item ${type}`;

        if (match && match[1]) {
            const filePath = match[1].trim();
            const fileName = filePath.split('/').pop();
            const fileExtension = fileName.split('.').pop().toLowerCase();

            // Handling different types of files
            let fileInteractionHtml = '';

            if (['jpg', 'jpeg', 'png', 'gif', 'svg', 'webp'].includes(fileExtension)) {
                fileInteractionHtml = `
                    <div class="file-interaction image-preview">
                        <img src="${filePath}" alt="${fileName}" class="preview-image" onclick="showFullImage('${filePath}')">
                        <a href="/download?file_path=${filePath}" download="${fileName}" class="download-link">⬇️ 下载图片</a>
                    </div>
                `;
            } else if (['mp3', 'wav', 'ogg'].includes(fileExtension)) {
                fileInteractionHtml = `
                    <div class="file-interaction audio-player">
                        <audio controls src="${filePath}"></audio>
                        <a href="/download?file_path=${filePath}" download="${fileName}" class="download-link">⬇️ 下载音频</a>
                    </div>
                `;
            } else if (['html', 'js', 'py'].includes(fileExtension)) {
                fileInteractionHtml = `
                    <div class="file-interaction code-file">
                        <button onclick="simulateRunPython('${filePath}')" class="run-button">▶️ 模拟运行</button>
                        <a href="/download?file_path=${filePath}" download="${fileName}" class="download-link">⬇️ 下载文件</a>
                    </div>
                `;
            } else {
                fileInteractionHtml = `
                    <div class="file-interaction">
                        <a href="/download?file_path=${filePath}" download="${fileName}" class="download-link">⬇️ 下载文件: ${fileName}</a>
                    </div>
                `;
            }

            step.innerHTML = `
                <div class="log-line">
                    <span class="log-prefix">${getEventIcon(type)} [${timestamp}] ${getEventLabel(type)}:</span>
                    <pre>${content}</pre>
                    ${fileInteractionHtml}
                </div>
            `;
        } else {
            step.innerHTML = `
                <div class="log-line">
                    <span class="log-prefix">${getEventIcon(type)} [${timestamp}] ${getEventLabel(type)}:</span>
                    <pre>${content}</pre>
                </div>
            `;
        }
    } else {
        step.className = `step-item ${type}`;
        step.innerHTML = `
            <div class="log-line">
                <span class="log-prefix">${getEventIcon(type)} [${timestamp}] ${getEventLabel(type)}:</span>
                <pre>${content}</pre>
            </div>
        `;
    }
    return step;
}

function autoScroll(element) {
    requestAnimationFrame(() => {
        element.scrollTo({
            top: element.scrollHeight,
            behavior: 'smooth'
        });
    });
    setTimeout(() => {
        element.scrollTop = element.scrollHeight;
    }, 100);
}


function getEventIcon(eventType) {
    const icons = {
        'think': '🤔',
        'tool': '🛠️',
        'act': '🚀',
        'result': '🏁',
        'error': '❌',
        'complete': '✅',
        'log': '📝',
        'run': '⚙️',
        'stopped': '⚠️'
    };
    return icons[eventType] || 'ℹ️';
}

function getEventLabel(eventType) {
    const labels = {
        'think': 'Thinking',
        'tool': 'Using Tool',
        'act': 'Action',
        'result': 'Result',
        'error': 'Error',
        'complete': 'Complete',
        'log': 'Log',
        'run': 'Running',
        'stopped': 'Stopped'
    };
    return labels[eventType] || 'Info';
}

function updateTaskStatus(task) {
    const statusBar = document.getElementById('status-bar');
    if (!statusBar) return;

    if (task.status === 'completed') {
        statusBar.innerHTML = `<span class="status-complete">✅ Task completed</span>`;
        toggleStopTaskButton(false); // 隐藏停止按钮

        if (currentEventSource) {
            currentEventSource.close();
            currentEventSource = null;
        }
        currentTaskId = null;
    } else if (task.status === 'stopped') {
        statusBar.innerHTML = `<span class="status-stopped">⚠️ Task stopped</span>`;
        toggleStopTaskButton(false); // 隐藏停止按钮

        if (currentEventSource) {
            currentEventSource.close();
            currentEventSource = null;
        }
        currentTaskId = null;
    } else if (task.status.startsWith('failed')) {
        statusBar.innerHTML = `<span class="status-error">❌ Task failed: ${task.status.replace('failed: ', '') || 'Unknown error'}</span>`;
        toggleStopTaskButton(false); // 隐藏停止按钮

        if (currentEventSource) {
            currentEventSource.close();
            currentEventSource = null;
        }
        currentTaskId = null;
    } else if (task.status === 'running') {
        statusBar.innerHTML = `<span class="status-running">⚙️ Task running</span>`;
        toggleStopTaskButton(true); // 显示停止按钮
        currentTaskId = task.id;
    } else {
        statusBar.innerHTML = `<span class="status-running">⚙️ Task status: ${task.status}</span>`;
    }
}

// Display full screen image
function showFullImage(imageSrc) {
    const modal = document.getElementById('image-modal');
    if (!modal) {
        const modalDiv = document.createElement('div');
        modalDiv.id = 'image-modal';
        modalDiv.className = 'image-modal';
        modalDiv.innerHTML = `
            <span class="close-modal">&times;</span>
            <img src="${imageSrc}" class="modal-content" id="full-image">
        `;
        document.body.appendChild(modalDiv);

        const closeBtn = modalDiv.querySelector('.close-modal');
        closeBtn.addEventListener('click', () => {
            modalDiv.classList.remove('active');
        });

        modalDiv.addEventListener('click', (e) => {
            if (e.target === modalDiv) {
                modalDiv.classList.remove('active');
            }
        });

        setTimeout(() => modalDiv.classList.add('active'), 10);
    } else {
        document.getElementById('full-image').src = imageSrc;
        modal.classList.add('active');
    }
}

// Simulate running Python files
function simulateRunPython(filePath) {
    let modal = document.getElementById('python-modal');
    if (!modal) {
        modal = document.createElement('div');
        modal.id = 'python-modal';
        modal.className = 'python-modal';
        modal.innerHTML = `
            <div class="python-console">
                <div class="close-modal">&times;</div>
                <div class="python-output">Loading Python file contents...</div>
            </div>
        `;
        document.body.appendChild(modal);

        const closeBtn = modal.querySelector('.close-modal');
        closeBtn.addEventListener('click', () => {
            modal.classList.remove('active');
        });
    }

    modal.classList.add('active');

    // Load Python file content
    fetch(filePath)
        .then(response => response.text())
        .then(code => {
            const outputDiv = modal.querySelector('.python-output');
            outputDiv.innerHTML = '';

            const codeElement = document.createElement('pre');
            codeElement.textContent = code;
            codeElement.style.marginBottom = '20px';
            codeElement.style.padding = '10px';
            codeElement.style.borderBottom = '1px solid #444';
            outputDiv.appendChild(codeElement);

            // Add simulation run results
            const resultElement = document.createElement('div');
            resultElement.innerHTML = `
                <div style="color: #4CAF50; margin-top: 10px; margin-bottom: 10px;">
                    > Simulated operation output:</div>
                <pre style="color: #f8f8f8;">
#This is the result of Python code simulation run
#The actual operational results may vary

# Running ${filePath.split('/').pop()}...
print("Hello from Python Simulated environment!")

# Code execution completed
</pre>
            `;
            outputDiv.appendChild(resultElement);
        })
        .catch(error => {
            console.error('Error loading Python file:', error);
            const outputDiv = modal.querySelector('.python-output');
            outputDiv.innerHTML = `Error loading file: ${error.message}`;
        });
}

document.addEventListener('DOMContentLoaded', () => {
    // Check configuration status
    checkConfigStatus();

    loadHistory();

    document.getElementById('prompt-input').addEventListener('keydown', (e) => {
        if (e.key === 'Enter' && !e.shiftKey) {
            e.preventDefault();
            createTask();
        }
    });

    const historyToggle = document.getElementById('history-toggle');
    if (historyToggle) {
        historyToggle.addEventListener('click', () => {
            const historyPanel = document.getElementById('history-panel');
            if (historyPanel) {
                historyPanel.classList.toggle('open');
                historyToggle.classList.toggle('active');
            }
        });
    }

    const clearButton = document.getElementById('clear-btn');
    if (clearButton) {
        clearButton.addEventListener('click', () => {
            document.getElementById('prompt-input').value = '';
            document.getElementById('prompt-input').focus();
        });
    }

    // 停止任务按钮事件
    const stopButton = document.getElementById('stop-task-btn');
    if (stopButton) {
        stopButton.addEventListener('click', stopCurrentTask);
    }

    // 清除历史记录按钮事件
    const clearHistoryBtn = document.getElementById('clear-history-btn');
    if (clearHistoryBtn) {
        clearHistoryBtn.addEventListener('click', showPasswordModal);
    }

    // 密码对话框确认按钮事件
    const confirmClearBtn = document.getElementById('confirm-clear-history');
    if (confirmClearBtn) {
        confirmClearBtn.addEventListener('click', clearTaskHistory);
    }

    // 密码对话框取消按钮事件
    const cancelClearBtn = document.getElementById('cancel-clear-history');
    if (cancelClearBtn) {
        cancelClearBtn.addEventListener('click', hidePasswordModal);
    }

    // 密码对话框关闭按钮事件
    const closePasswordBtn = document.getElementById('close-password-modal');
    if (closePasswordBtn) {
        closePasswordBtn.addEventListener('click', hidePasswordModal);
    }

    // 密码输入框回车事件
    const passwordInput = document.getElementById('history-password');
    if (passwordInput) {
        passwordInput.addEventListener('keydown', (e) => {
            if (e.key === 'Enter') {
                e.preventDefault();
                clearTaskHistory();
            }
        });
    }

    // Add keyboard event listener to close modal boxes
    document.addEventListener('keydown', (e) => {
        if (e.key === 'Escape') {
            const imageModal = document.getElementById('image-modal');
            if (imageModal && imageModal.classList.contains('active')) {
                imageModal.classList.remove('active');
            }

            const pythonModal = document.getElementById('python-modal');
            if (pythonModal && pythonModal.classList.contains('active')) {
                pythonModal.classList.remove('active');
            }

            // 添加关闭密码对话框的快捷键支持
            const passwordModal = document.getElementById('password-modal');
            if (passwordModal && passwordModal.classList.contains('active')) {
                hidePasswordModal();
            }

            // Do not close the configuration pop-up, because the configuration is required
        }
    });
});


```













