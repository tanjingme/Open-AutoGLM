# AutoGLM-Phone-9B 与 Open-AutoGLM 交互机制深度解析

本文档旨在从源码角度深度解析 AutoGLM-Phone-9B（模型端）与 Open-AutoGLM（框架端）的交互全流程。面向 大模型算法工程师，重点阐述 Tokens 如何转化为手机上的物理操作。

## 1. 核心架构概览

整个交互过程是一个闭环控制系统（Closed-Loop Control System），遵循 **观察 (Observe) -> 思考 (Think) -> 行动 (Act)** 的范式。

*   **Model Service (Server)**: 部署了 AutoGLM-Phone-9B 的推理服务（vLLM/SGLang），负责接收多模态输入（Prompt + Image）并输出推理结果（Tokens）。
*   **Phone Agent (Client)**: 运行在本地的 Python 客户端，负责状态获取、Prompt 构建、结果解析以及 ADB 指令执行。

## 2. 交互全流程源码分析

### 2.1 观察阶段 (Observation)

框架首先获取手机的当前状态，这构成了模型的输入上下文。

*   **相关文件**: `phone_agent/agent.py`, `phone_agent/adb/device.py`, `phone_agent/adb/screenshot.py`
*   **关键逻辑**:
    1.  **截屏**: 调用 `get_screenshot` 获取当前屏幕截图。
    2.  **应用识别**: 调用 `get_current_app` 通过 `adb shell dumpsys window` 获取当前前台运行的 App 包名，并映射为人类可读的 App 名称（如 `com.tencent.mm` -> "微信"）。
    3.  **Prompt 构建**:
        *   **System Prompt**: 定义了模型的角色和可用操作集合（在 `phone_agent/config/prompts_ch.py` 中定义）。
        *   **User Message**: 将任务指令（如“打开微信”）和当前屏幕状态（Image + App Name）组合。

```python
# phone_agent/agent.py (伪代码)
screenshot = get_screenshot(device_id)
current_app = get_current_app(device_id)
screen_info = json.dumps({"current_app": current_app})
messages = [
    {"role": "system", "content": system_prompt},
    {"role": "user", "content": [
        {"type": "text", "text": f"{user_task}\n\n{screen_info}"},
        {"type": "image_url", "image_url": ...} # 传入截屏
    ]}
]
```

### 2.2 推理阶段 (Inference)

客户端将构建好的 Messages 发送给模型服务，并流式接收生成的 Tokens。

*   **相关文件**: `phone_agent/model/client.py`
*   **输入**: OpenAI 格式的 Chat Completion 请求。
*   **输出流格式**: 模型经过 Fine-tuning，被训练为输出特定的结构化文本。
    1.  **Thinking Chain**:通常包含在 `<think>...</think>` 中（或隐式输出），用于进行 GUI 理解和任务规划。
    2.  **Action Function**: 类似函数调用的字符串，例如 `do(action="Tap", element=[500, 1000])`。

*   **解析逻辑 (`ModelClient._parse_response`)**:
    代码实时监听流式输出，通过字符串匹配分离“思考”和“行动”。
    *   **Thinking**: `do(action=` 之前的所有内容。
    *   **Action**: 以 `do(action=` 或 `finish(message=` 开头的内容。

```python
# phone_agent/model/client.py
# 实时解析 Token 流
if "do(action=" in buffer:
    thinking = buffer.split("do(action=")[0] # 提取思维链
    action_str = "do(action=" + buffer.split("do(action=")[1] # 提取动作指令
```

### 2.3 解析阶段 (Translation)

将模型输出的非结构化字符串（Tokens）转化为结构化的 Python 字典。

*   **相关文件**: `phone_agent/actions/handler.py` -> `parse_action`
*   **处理过程**:
    1.  **提取**: 获取 `do(...)` 字符串。
    2.  **AST 解析**: 使用 Python 的 `ast.parse(mode='eval')` 安全地将字符串转化为 Python 对象。这里模型输出的格式必须严格符合 Python 函数调用语法。
    
    > **算法工程师注意**: 这里的“函数调用”并非真正的 Tool Use（Function Calling）API，而是模型被训练为直接输出符合这一语法的文本。

```python
# 示例模型输出
response_str = 'do(action="Tap", element=[500, 500])'

# 解析结果
action_dict = {
    "_metadata": "do",
    "action": "Tap",
    "element": [500, 500]
}
```

### 2.4 执行阶段 (Execution)

将结构化的 Action 字典转化为底层的 ADB 命令。

*   **相关文件**: `phone_agent/actions/handler.py` -> `ActionHandler`
*   **坐标转换**:
    模型输出的坐标是**归一化坐标**（通常是 0-1000 的相对坐标）。`ActionHandler` 需要将其映射回手机的真实分辨率。
    
    $$ X_{real} = \frac{X_{model}}{1000} \times Width_{screen} $$
    
*   **指令分发**:
    根据 `action` 字段（如 "Tap", "Swipe", "Type"）分发到对应的处理函数。

*   **ADB 调用 (`phone_agent/adb/device.py`)**:
    最终通过 `subprocess` 调用 ADB 二进制文件。

#### 关键操作映射表

| 模型 Action | Python Handler | 底层 ADB 命令 | 备注 |
| :--- | :--- | :--- | :--- |
| `Tap` | `_handle_tap` | `input tap x y` | 点击 |
| `Swipe` | `_handle_swipe` | `input swipe x1 y1 x2 y2 d` | 滑动 |
| `Type` | `_handle_type` | `ADB Keyboard` 注入 | 需切换输入法 |
| `Launch` | `_handle_launch` | `monkey -p package ...` | 启动应用 |

```python
# phone_agent/actions/handler.py
def _handle_tap(self, action, width, height):
    # 1. 获取相对坐标
    rel_x, rel_y = action["element"]
    # 2. 转换为绝对坐标
    abs_x = int(rel_x / 1000 * width)
    abs_y = int(rel_y / 1000 * height)
    # 3. 执行 ADB 命令
    subprocess.run(["adb", "shell", "input", "tap", str(abs_x), str(abs_y)])
```

## 3. 总结

`Tokens` 变 `Actions` 的核心路径：

1.  **Generation**: LLM 基于截屏生成符合 Python 函数语法的字符串 `do(action="Tap", element=[x, y])`。
2.  **Streaming Parse**: Python 客户端实时截取该字符串。
3.  **AST Eval**: 利用 `ast.literal_eval` 还原为字典数据结构。
4.  **Normalization**: 将 1000 系相对坐标映射为屏幕绝对像素。
5.  **Syscall**: 封装为 `adb shell input` 命令发送至 Android 系统内核。

这一过程不仅依赖模型强大的多模态理解能力（理解界面布局）和指令遵循能力（输出正确格式），也依赖框架端精确的坐标换算和鲁棒的 ADB 封装。
