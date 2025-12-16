# Phone Agent 核心操作指令深度解析
本文档旨在全面、无遗漏地解析 `phone_agent/actions/handler.py` 文件中定义的所有手机操作。该解析基于 `ActionHandler` 类的实现逻辑，面向需要理解大模型指令如何转化为底层原子操作的算法工程师和系统开发者。
## 1. 核心映射机制 (Architecture Overview)
`ActionHandler` 类维护了一个核心的映射表 `_get_handler`，它负责将模型输出的 `action_name` 路由到具体的 Python 方法。
### 坐标转换说明
大多数涉及屏幕交互的操作（Tap, Swipe等）都涉及坐标转换。模型输出的是归一化相对坐标（通常为 0-1000），而 ADB 需要屏幕绝对像素坐标。
`ActionHandler._convert_relative_to_absolute` 方法负责此转换：
$$ X_{abs} = \lfloor \frac{X_{rel}}{1000} \times Width \rfloor, \quad Y_{abs} = \lfloor \frac{Y_{rel}}{1000} \times Height \rfloor $$
## 2. 操作指令全集 (Comprehensive Action List)
以下是 `handler.py` 中定义的所有 14 种操作指令及其详细实现逻辑。
### 2.1 基础交互操作 (Basic Interactions)
#### 1. `Launch` (启动应用)
*   **方法**: `_handle_launch(self, action, width, height)`
*   **参数**: `app` (应用名称，如 "微信")
*   **实现逻辑**:
    1.  检查 `app` 参数是否存在。
    2.  调用 `adb.launch_app(app_name)`。
    3.  底层使用 `monkey -p <package_name>` 命令启动应用（相比 `am start` 更通用，不需要知道 Activity 名）。
    4.  如果应用不在预定义列表 (`APP_PACKAGES`) 中，返回失败。
#### 2. `Tap` (单机点击)
*   **方法**: `_handle_tap(self, action, width, height)`
*   **参数**:
    *   `element`: `[x, y]` (相对坐标)
    *   `message`: (可选) 敏感操作提示信息
*   **实现逻辑**:
    1.  校验坐标。
    2.  **安全拦截**: 如果指令包含 `message` 字段，触发 `confirmation_callback` 请求人工确认（例如支付、转账场景）。若用户拒绝，通过返回值终止任务。
    3.  调用 `adb.tap(x, y)` 发送 `input tap` 命令。
#### 3. `Swipe` (滑动)
*   **方法**: `_handle_swipe(self, action, width, height)`
*   **参数**:
    *   `start`: `[x1, y1]` (起始相对坐标)
    *   `end`: `[x2, y2]` (终点相对坐标)
*   **实现逻辑**:
    1.  将起点和终点坐标转化为绝对坐标。
    2.  调用 `adb.swipe(x1, y1, x2, y2)`。
    3.  底层会自动计算滑动持续时间（基于距离），通常在 1000ms - 2000ms 之间，以模拟自然的滑动轨迹。
#### 4. `Type` / `Type_Name` (文本输入)
*   **方法**: `_handle_type(self, action, width, height)`
*   **参数**: `text` (要输入的文本内容)
*   **实现逻辑**:
    1.  **切换输入法**: 自动将当前输入法切换为 `ADB Keyboard` (需要预先安装)。
    2.  **清空文本**: 发送清空指令，确保输入框干净。
    3.  **输入文本**: 通过 `ADB Keyboard` 注入文本（支持 Base64 编码以处理中文）。
    4.  **还原输入法**: 恢复用户原先使用的输入法。
    *此流程设计非常精细，解决了原生 ADB `input text` 不支持中文且无法清空输入框的痛点。*
#### 5. `Back` (返回)
*   **方法**: `_handle_back(self, action, width, height)`
*   **参数**: 无
*   **实现逻辑**: 调用 `adb.back()`，发送 `output keyevent 4`。
#### 6. `Home` (回主页)
*   **方法**: `_handle_home(self, action, width, height)`
*   **参数**: 无
*   **实现逻辑**: 调用 `adb.home()`，发送 `output keyevent KEYCODE_HOME`。
### 2.2 进阶交互操作 (Advanced Interactions)
#### 7. `Double Tap` (双击)
*   **方法**: `_handle_double_tap(self, action, width, height)`
*   **参数**: `element`: `[x, y]`
*   **实现逻辑**:
    1.  转化坐标。
    2.  调用 `adb.double_tap(x, y)`。
    3.  底层逻辑是发送两次 `input tap`，中间间隔极短（由配置决定，通常几十毫秒）。
#### 8. `Long Press` (长按)
*   **方法**: `_handle_long_press(self, action, width, height)`
*   **参数**: `element`: `[x, y]`
*   **实现逻辑**:
    1.  转化坐标。
    2.  调用 `adb.long_press(x, y)`。
    3.  底层通过 `input swipe x y x y duration` 实现长按（即起点终点相同，持续时间长的滑动）。默认长按时间为 3000ms。
#### 9. `Wait` (等待)
*   **方法**: `_handle_wait(self, action, width, height)`
*   **参数**: `duration` (字符串，如 "5 seconds")
*   **实现逻辑**:
    1.  解析字符串中的数字。
    2.  调用 `time.sleep()` 阻塞线程。
    3.  这是模型用于处理页面加载、广告播放等需要异步等待场景的机制。
### 2.3 辅助与人工介入操作 (Auxiliary & Intervention)
#### 10. `Take_over` (人工接管)
*   **方法**: `_handle_takeover(self, action, width, height)`
*   **参数**: `message` (请求接管的原因)
*   **实现逻辑**:
    1.  调用 `takeover_callback(message)`。
    2.  通常用于遇到验证码、复杂登录、人脸识别等 Agent 无法处理的场景，程序会挂起等待用户在终端确认完成后继续。
#### 11. `Note` (记录/笔记)
*   **方法**: `_handle_note(self, action, width, height)`
*   **参数**: 任意
*   **实现逻辑**:
    *   **当前状态**: 占位符 (Placeholder)。
    *   **用途**: 返回 `Success=True` 但不执行任何物理操作。设计用于让 Agent 记录某些观察到的信息（如价格、内容摘要），以便在后续步骤中使用。
#### 12. `Call_API` (API 调用)
*   **方法**: `_handle_call_api(self, action, width, height)`
*   **参数**: 任意
*   **实现逻辑**:
    *   **当前状态**: 占位符 (Placeholder)。
    *   **用途**: 未来扩展点，允许 Agent 调用外部 API（如搜索、计算器、翻译等）。
#### 13. `Interact` (用户交互)
*   **方法**: `_handle_interact(self, action, width, height)`
*   **参数**: 任意
*   **实现逻辑**:
    *   返回 `message="User interaction required"`，但不强制中断。
    *   这似乎是一个比 `Take_over` 更轻量级的交互请求信号，目前逻辑较为简单。
#### 14. `finish` (任务结束)
*   **逻辑**: 这是一个特殊的 Metadata Action，不在 `_get_handler` 中，而是在 `execute` 方法入口处直接判断。
*   **行为**: 标记当前任务 `should_finish=True`，并返回最终结果消息。
## 3. 解析与路由总结
`ActionHandler.execute` 方法是所有操作的总入口：
1.  **Metadata Check**: 首先检查 `_metadata` 是否为 "finish"，如果是直接返回。
2.  **Type Check**: 确保操作类型基于 "do"。
3.  **Dispatch**: 使用 `action_name` 查找上述 14 个 Handler 之一。
4.  **Try-Catch**: 统一捕获所有底层 ADB 异常，保证 Agent 不会因为单一指令失败而崩溃。
这份文档涵盖了源码中目前存在的所有分支逻辑，确保没有任何操作遗漏。
