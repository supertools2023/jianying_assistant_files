# 插件开发指南

映映小助手支持通过 Python 编写插件来扩展功能。插件运行在独立的 Python 宿主进程中，通过 WebSocket 与 Flutter 主程序进行双向通信。

## 1. 目录结构与插件定义

映映小助手的插件系统位于项目的 `plugins/` 目录下。该目录包含公共 SDK 文件、共享依赖定义以及各个独立的插件子目录。

### 1.1 核心目录结构

```text
plugins/
├── sdk.py               # 插件 SDK 核心逻辑
├── plugin_host.py       # 插件宿主程序（内部使用）
├── requirements.txt      # 共享依赖定义（必须）
├── my_plugin/           # 插件 A（子目录形式）
│   ├── manifest.json    # 插件元数据（必须）
│   └── main.py         # 插件入口代码（必须）
└── another_plugin/      # 插件 B
    ├── manifest.json
    └── main.py
```

### 1.2 共享依赖 (requirements.txt)

位于 `plugins/requirements.txt` 的文件定义了所有插件共享的 Python 依赖包。当应用启动或用户强制重建环境时，系统会自动安装此文件中的所有依赖。

**注意事项:**
- 请将插件所需的第三方库（如 `requests`, `pandas` 等）添加到此文件中。
- 所有插件共享同一个虚拟环境，请注意依赖版本冲突。

### 1.3 插件定义 (manifest.json)

每个插件子目录必须包含一个 `manifest.json` 文件。

**示例:**
```json
{
  "id": "my_plugin",
  "name": "我的示例插件",
  "version": "1.0.0",
  "description": "这是一个演示插件",
  "author": "YourName",
  "entry": "main.py"
}
```

### 1.4 插件入口 (main.py)

插件必须提供一个 `setup(api)` 函数，这是插件加载时的入口。推荐使用异步函数 (`async def`) 来处理事件，以获得更好的响应性能。

```python
def setup(api):
    api.log("插件已加载")

    # 注册 UI 动作
    api.register_ui_action(
        action_id="my_action",
        label="点我一下",
        icon="magic"
    )

    # 监听 UI 动作 (推荐异步方式)
    @api.on("on_ui_action")
    async def on_action(params):
        if params.get("actionId") == "my_action":
            # 获取当前选中的草稿列表（如果在草稿管理页面）
            selected_drafts = params.get("selectedDrafts", [])
            if selected_drafts:
                api.log(f"选中了 {len(selected_drafts)} 个草稿")
                for draft in selected_drafts:
                    api.log(f"  - {draft['name']}: {draft['folderPath']}")
            else:
                api.log("未选中任何草稿")

            # 直接调用阻塞式 UI 接口（内部已处理同步等待）
            api.alert("你点击了按钮！")
            return {"status": "ok"}
```

---

## 2. SDK 接口说明 (`api` 对象)

`api` 对象（`PluginContext` 实例）提供了与主程序交互的所有方法。SDK 会自动识别您的处理器是同步还是异步，并在合适的环境中运行它们。

### 2.1 基础功能
- `api.log(message: str)`: 向主程序发送日志，显示在对应的插件日志窗口。
- `api.on(event_name: str)`: 装饰器，用于监听主程序发送的事件。
  - `on_ui_action`: 当用户点击插件注册的按钮时触发。推荐使用 `async def`。
    - `params` 参数包含：
      - `actionId`: 触发的动作标识
      - `selectedDrafts`: 当前选中的草稿列表（仅在草稿管理页面可用）
        - 每个草稿对象包含：
          - `name`: 草稿名称
          - `path`: 草稿draft_content.json的路径
          - `folderPath`: 草稿目录的路径
          - `isEncrypted`: 是否加密
  - `on_configure`: 当用户点击"设置"图标时触发。
- `api.on_teardown`: 装饰器，注册插件卸载时的回调函数（同步）。

### 2.2 交互反馈
为了简化逻辑，大部分交互接口被封装为同步阻塞调用。

#### 2.2.1 基础对话框
- `api.show_notification(content: str, title: str = "插件通知", type: str = "info")`: 显示系统通知（非阻塞）。
  - `type`: 通知类型，支持 `info` (默认), `success` (成功/绿色), `warning` (警告/橙色), `error` (错误/红色)。
- `api.alert(content: str, title: str = "提示")`: 弹出警告对话框（同步阻塞）。
- `api.confirm(content: str, title: str = "确认") -> bool`: 弹出确认对话框（同步阻塞）。返回 `True` 表示用户点击确定，`False` 表示取消。
- `api.prompt(content: str, title: str = "输入", default_value: str = "", hint_text: str = "", multi_line: bool = False, min_lines: int = None, max_lines: int = None) -> str`: 弹出输入对话框（同步阻塞）。
  - `default_value`: 输入框的初始值。
  - `hint_text`: 输入框的占位符提示文本。
  - `multi_line`: 是否启用多行输入模式。
  - `min_lines`, `max_lines`: 在多行模式下控制输入框的最小/最大行数。
  - 返回用户输入的文本内容，如果用户取消则返回空字符串。

#### 2.2.2 自定义表单对话框
- `api.show_custom_form(config: dict) -> dict | None`: 通过 JSON 配置构造复杂的自定义输入界面（同步阻塞）。

**配置结构 (`config`):**
```python
{
    "title": "表单标题",  # 对话框标题
    "items": [           # 表单项列表
        {
            "type": "label",      # 组件类型
            "text": "说明文字"     # 标签文本
        },
        {
            "type": "input",           # 文本输入框
            "name": "username",        # 字段名（用于返回值的 key）
            "label": "用户名",         # 字段标签
            "value": "admin",          # 初始值
            "hint": "请输入用户名"     # 占位符（可选）
        },
        {
            "type": "combox",          # 下拉选择框
            "name": "mode",
            "label": "运行模式",
            "value": "fast",           # 当前选中值
            "options": ["fast", "safe", "debug"]  # 可选项列表
        },
        {
            "type": "select_file",     # 文件选择器
            "name": "background",
            "label": "背景图片",
            "value": "",               # 当前文件路径
            "extensions": ["png", "jpg"]  # 允许的文件扩展名（可选）
        },
        {
            "type": "select_dir",      # 目录选择器
            "name": "output",
            "label": "输出目录",
            "value": ""
        },
        {
            "type": "button",          # 动作按钮
            "label": "测试连接",
            "actionId": "test_conn"    # 动作标识（会触发 on_ui_action 事件）
        }
    ]
}
```

**返回值:**
- 用户点击确定：返回包含各字段当前值的字典，例如：
  ```python
  {
      "username": "admin",
      "mode": "safe",
      "background": "C:/images/bg.png",
      "output": "C:/exports"
  }
  ```
- 用户点击取消或关闭窗口：返回 `None`

**使用示例:**
```python
@api.on("on_ui_action")
async def handle_action(params):
    if params.get("actionId") == "open_settings":
        result = api.show_custom_form({
            "title": "插件配置",
            "items": [
                {"type": "label", "text": "基础设置"},
                {"type": "input", "name": "api_key", "label": "API 密钥", "value": ""},
                {"type": "combox", "name": "quality", "label": "输出质量",
                 "value": "high", "options": ["low", "medium", "high"]},
                {"type": "select_dir", "name": "cache", "label": "缓存目录"}
            ]
        })

        if result:
            api.log(f"用户配置: {result}")
            api.set_plugin_storage("settings", result)
        else:
            api.log("用户取消了配置")
```

**注意事项:**
- `button` 类型的组件点击时会触发 `on_ui_action` 事件，`params` 中包含对应的 `actionId`。
- 按钮点击不会关闭对话框，需要在事件处理器中手动更新表单或调用其他接口。
- 所有带 `name` 字段的组件都会出现在返回值中。

#### 2.2.3 日志对话框
- `api.show_log_dialog(clear: bool = False)`: 调起日志对话框（非阻塞）。默认打开当前插件的日志窗口。
  - `clear`: 是否清空之前累积的日志。

### 2.3 文件与系统
- `api.select_file(title: str, allowed_extensions: list)`: 调用系统文件选择器（同步阻塞）。
- `api.select_directory(title: str)`: 调用系统目录选择器（同步阻塞）。
- `api.get_clipboard()`: 获取剪贴板文本。
- `api.set_clipboard(content: str)`: 设置剪贴板文本。

### 2.4 应用配置与导航
- `api.get_app_config() -> dict`: 获取应用的全局配置信息，包括剪映路径、服务器地址等。
- `api.navigate_to(target: str)`: 导航到应用的指定页面（非阻塞）。
  - 支持的目标页面：
    - `"home"`: 首页
    - `"drafts"`: 草稿管理页
    - `"materials"`: 素材库页
    - `"plugins"`: 插件管理页
    - `"settings"`: 设置页

### 2.5 剪映与草稿相关
- `api.is_jianying_running() -> bool`: 检查剪映是否正在运行。
- `api.get_current_draft_dir() -> str`: 获取当前选中的草稿目录路径。如果未选中草稿则返回空字符串。
- `api.read_draft_file(path: str) -> str`: 读取草稿文件（自动处理加密）。
  - `path`: 相对于草稿目录的文件路径（如 `"draft_content.json"`）。
- `api.write_draft_file(path: str, content: str, encrypt: bool = True)`: 写入草稿文件。
  - `encrypt`: 是否使用剪映的加密格式写入（默认为 `True`）。
- `api.get_jianying_info() -> dict`: 获取剪映安装路径、版本等信息。
  - 返回示例：
    ```python
    {
        "installPath": "C:/Program Files/JianyingPro",
        "version": "9.7.1",
        "exePath": "C:/Program Files/JianyingPro/JianyingPro.exe"
    }
    ```

### 2.6 数据持久化
- `api.get_plugin_storage(key: str) -> Any`: 获取该插件专属的持久化数据。
  - 返回之前通过 `set_plugin_storage` 保存的值，如果不存在则返回 `None`。
- `api.set_plugin_storage(key: str, value: Any)`: 存储该插件专属的持久化数据。
  - 支持任何可 JSON 序列化的数据类型（字符串、数字、列表、字典等）。
- `api.get_plugins_info() -> list`: 获取所有已安装插件的详细信息及状态。
  - 返回插件信息列表，每个插件包含：
    ```python
    {
        "id": "my_plugin",
        "isEnabled": True,
        "isRunning": True,
        "manifest": {
            "name": "我的插件",
            "version": "1.0.0",
            ...
        }
    }
    ```
- `api.get_plugin_info(plugin_id: str) -> dict`: 获取特定插件的信息。包含 `isEnabled`, `isRunning` 状态及 `manifest` 内容。

### 2.7 扩展服务 (ExtService)
允许插件注册自己的服务接口供其他插件调用，实现插件间的解耦通信。

- `api.register_ext_service(instance: Any)`: 向系统注册本插件的扩展服务对象（通常是自定义类的一个实例）。
  ```python
  class MyService:
      def greet(self, name: str) -> str:
          return f"Hello, {name}!"

  def setup(api):
      service = MyService()
      api.register_ext_service(service)
  ```

- `api.get_ext_service(plugin_id: str, service_type: Type[T] = Any) -> T`: 获取指定插件注册的服务对象。
  - `plugin_id`: 目标插件的 ID。
  - `service_type`: 期望的服务类型（用于 IDE 类型推导和运行时校验，可选）。
  - 返回服务对象实例，如果目标插件未运行或未注册服务则返回 `None`。
  - **重要限制**: 出于架构安全，禁止在插件的 `setup()` 过程中调用 `get_ext_service`。请在事件处理器或异步任务中获取。

  ```python
  @api.on("on_ui_action")
  async def handle_action(params):
      # 获取其他插件的服务
      other_service = api.get_ext_service("other_plugin")
      if other_service:
          result = other_service.greet("World")
          api.log(result)
      else:
          api.log("目标插件未运行或未提供服务")
  ```

### 2.8 多线程支持
- `api.create_thread(target: Callable, args: tuple = (), kwargs: dict = None, daemon: bool = True) -> threading.Thread`: 创建并启动一个受管理的线程。
  - 创建的线程会被自动记录，方便插件卸载时进行清理。
  - **重要**: 线程内的逻辑应该定期检查 `api.running` 属性，以便在插件卸载时优雅退出。

  ```python
  def background_task(api):
      while api.running:
          api.log("后台任务运行中...")
          time.sleep(5)
      api.log("后台任务已停止")

  def setup(api):
      api.create_thread(target=background_task, args=(api,))
  ```

- `api.running -> bool`: 只读属性，指示插件是否正在运行。当插件被禁用或卸载时，此值会变为 `False`。

- `api.on_teardown(func: Callable)`: 装饰器，注册插件卸载时的回调函数。
  ```python
  @api.on_teardown
  def cleanup():
      api.log("插件正在卸载，执行清理操作...")
      # 清理资源、关闭连接等
  ```

---

## 3. UI 动作注册与更新

### 3.1 注册 UI 动作
通过 `api.register_ui_action` 可以将按钮添加到主程序的 UI 中：

```python
api.register_ui_action(
    action_id="my_export",
    label="导出字幕",
    icon="export",
    location="draft_action_bar"
)
```

**参数说明:**
- `action_id` (str): 动作唯一标识，用于在 `on_ui_action` 事件中识别按钮点击。
- `label` (str): 按钮显示的文本。
- `icon` (str, 可选): 图标。
  - **内置图标**: `magic`, `clean`, `check`, `export`, `sync`, `rocket`, `star`, `refresh`, `add`, `settings`, `download`, `upload`, `delete`, `edit`, `search` 等。
  - **自定义图片**: 传入插件目录下的图片文件名（如 `icon.png`）。建议使用 40x40 像素的 PNG 图片。
- `location` (str): 显示位置。
  - `"draft_action_bar"`: 草稿管理页。按钮会同时出现在顶部的 Header 和底部的"插件功能"菜单中。
    - **重要**: 当按钮在草稿管理页被点击时，`on_ui_action` 的 `params` 参数会包含 `selectedDrafts` 字段，表示当前选中的草稿列表。
  - `"home_quick_actions"`: 首页快捷操作区。以大图标网格形式展示。
  - `"resource_hub_actions"`: 资源中心页。出现在顶部工具栏。

### 3.2 动态更新 UI 动作
在插件运行过程中，可以动态更新已注册按钮的显示内容：

- `api.update_ui_action(action_id: str, label: str = None, icon: str = None)`: 更新已注册的 UI 动作。

**参数说明:**
- `action_id` (str): 要更新的动作标识。
- `label` (str, 可选): 新的按钮文本。如果不传则保持原样。
- `icon` (str, 可选): 新的图标。如果不传则保持原样。

**使用场景示例:**
```python
@api.on("on_ui_action")
async def handle_action(params):
    if params.get("actionId") == "process_data":
        # 更新按钮状态为处理中
        api.update_ui_action("process_data", label="处理中...", icon="sync")

        # 执行耗时操作
        await process_long_task()

        # 恢复按钮状态
        api.update_ui_action("process_data", label="处理数据", icon="magic")
        api.show_notification("处理完成！", type="success")
```

---

## 4. 关键机制与注意事项

1. **唯一 ID 策略**: 系统优先使用插件 `manifest.json` 中定义的 `id` 作为唯一标识。如果未定义，则回退到文件夹名称。请确保您的插件 ID 在所有安装的插件中是唯一的。
2. **默认关闭状态**: 插件安装后默认不运行。只有当用户在"插件管理"界面手动开启后，才会调用 `setup(api)` 并注册 UI。
3. **按钮折叠**: 各个页面的 Header 区域最多展示 10 个图标按钮。超过此限制时，剩余按钮会自动收纳进"更多插件"菜单。
4. **共享宿主**: 所有 Python 插件运行在同一个进程中，请避免使用全局变量污染。
5. **多线程/协程**: SDK 支持在独立协程中处理事件回调，推荐使用 `asyncio` 编写非阻塞代码。
6. **热重载**: 主程序支持在不重启的情况下重新加载插件代码。

---

## 5. 性能优化建议

为了保证插件运行流畅及 App 的响应速度，请遵循以下建议：

1. **本地请求禁用代理**: 在使用 `requests` 库请求本地服务时，建议显式设置 `proxies={'http': None, 'https': None}`，以避免因系统全局代理导致的 1-2 秒延迟。
2. **地址优先使用 127.0.0.1**: 尽量避免使用 `localhost`，在 Windows 系统上可能存在 IPv6 优先解析导致的 DNS 延迟。
3. **及时刷新日志**: SDK 内部已优化日志刷新，但在执行长耗时同步操作前手动打印日志可以帮助用户感知进度。
4. **异步 UI 提示**: 对于耗时操作（如网络校验），建议先弹出一个"处理中..."的 `show_notification`，待结果返回后再更新提示内容。

---

## 6. 完整示例：草稿处理插件

以下示例展示了一个功能完整的插件，演示了如何使用 `show_custom_form` 创建复杂的配置界面，以及如何处理用户选中的草稿：

```python
import time

def setup(api):
    """插件入口函数"""
    api.log("草稿处理插件已加载")

    # 注册设置按钮（添加到草稿管理页）
    api.register_ui_action(
        action_id="open_advanced_settings",
        label="高级设置",
        icon="settings",
        location="draft_action_bar"
    )

    # 注册批量导出按钮（添加到草稿管理页）
    api.register_ui_action(
        action_id="batch_export",
        label="批量导出",
        icon="export",
        location="draft_action_bar"
    )

    @api.on("on_ui_action")
    async def handle_action(params):
        action_id = params.get("actionId")

        if action_id == "open_advanced_settings":
            # 从存储中读取之前的配置
            saved_config = api.get_plugin_storage("config") or {}

            # 构建表单配置
            form_config = {
                "title": "插件高级配置",
                "items": [
                    {"type": "label", "text": "=== 基础设置 ==="},
                    {
                        "type": "input",
                        "name": "server_url",
                        "label": "服务器地址",
                        "value": saved_config.get("server_url", "http://127.0.0.1:8000"),
                        "hint": "请输入完整的 HTTP 地址"
                    },
                    {
                        "type": "input",
                        "name": "api_key",
                        "label": "API 密钥",
                        "value": saved_config.get("api_key", ""),
                        "hint": "留空则使用默认密钥"
                    },
                    {"type": "label", "text": "=== 输出设置 ==="},
                    {
                        "type": "combox",
                        "name": "quality",
                        "label": "导出质量",
                        "value": saved_config.get("quality", "high"),
                        "options": ["low", "medium", "high", "ultra"]
                    },
                    {
                        "type": "select_dir",
                        "name": "output_dir",
                        "label": "默认输出目录",
                        "value": saved_config.get("output_dir", "")
                    },
                    {"type": "label", "text": "=== 高级功能 ==="},
                    {
                        "type": "select_file",
                        "name": "template_file",
                        "label": "模板文件",
                        "value": saved_config.get("template_file", ""),
                        "extensions": ["json", "yaml", "yml"]
                    },
                    {"type": "button", "label": "测试连接", "actionId": "test_connection"}
                ]
            }

            # 显示表单并等待用户操作
            result = api.show_custom_form(form_config)

            if result:
                # 用户点击了确定，保存配置
                api.set_plugin_storage("config", result)
                api.show_notification(
                    f"配置已保存！服务器: {result.get('server_url')}",
                    type="success"
                )
                api.log(f"新配置已保存: {result}")
            else:
                # 用户点击了取消
                api.log("用户取消了配置")

        elif action_id == "batch_export":
            # 获取当前选中的草稿列表
            selected_drafts = params.get("selectedDrafts", [])

            if not selected_drafts:
                api.alert("请先选择要导出的草稿！")
                return

            # 确认导出
            draft_names = [d['name'] for d in selected_drafts]
            confirm_msg = f"确定要导出以下 {len(selected_drafts)} 个草稿吗？\n\n" + "\n".join(f"• {name}" for name in draft_names[:5])
            if len(draft_names) > 5:
                confirm_msg += f"\n... 还有 {len(draft_names) - 5} 个草稿"

            if not api.confirm(confirm_msg, title="确认导出"):
                api.log("用户取消了批量导出")
                return

            # 更新按钮状态
            api.update_ui_action("batch_export", label="导出中...", icon="sync")

            # 批量处理草稿
            success_count = 0
            for idx, draft in enumerate(selected_drafts, 1):
                api.log(f"[{idx}/{len(selected_drafts)}] 正在处理: {draft['name']}")
                try:
                    # 读取草稿文件
                    draft_content = api.read_draft_file("draft_content.json")

                    # 这里可以添加实际的导出逻辑
                    # 例如: export_to_file(draft_content, output_dir)

                    success_count += 1
                    api.show_notification(f"已处理: {draft['name']} ({idx}/{len(selected_drafts)})", type="info")
                except Exception as e:
                    api.log(f"处理草稿 {draft['name']} 时出错: {str(e)}")

            # 恢复按钮状态
            api.update_ui_action("batch_export", label="批量导出", icon="export")

            # 显示结果
            api.alert(f"批量导出完成！\n\n成功: {success_count}\n失败: {len(selected_drafts) - success_count}", title="导出结果")

        elif action_id == "test_connection":
            # 处理表单中的按钮点击（按钮点击不会关闭表单）
            api.show_notification("正在测试连接...", type="info")

            # 模拟连接测试
            time.sleep(1)

            # 根据测试结果显示不同的通知
            try:
                # 这里可以添加实际的连接测试逻辑
                # 例如: requests.get(server_url, timeout=5)
                api.show_notification("连接测试成功！", type="success")
            except Exception as e:
                api.show_notification(f"连接失败: {str(e)}", type="error")

    @api.on_teardown
    def cleanup():
        """插件卸载时的清理操作"""
        api.log("高级配置插件正在卸载...")
```

**表单组件完整参考:**

| 组件类型 | 必需字段 | 可选字段 | 说明 |
|---------|---------|---------|------|
| `label` | `text` | - | 纯文本标签，用于分组或说明 |
| `input` | `name`, `label` | `value`, `hint` | 单行文本输入框 |
| `combox` | `name`, `label`, `options` | `value` | 下拉选择框 |
| `select_file` | `name`, `label` | `value`, `extensions` | 文件选择器 |
| `select_dir` | `name`, `label` | `value` | 目录选择器 |
| `button` | `label`, `actionId` | - | 动作按钮（触发 `on_ui_action` 事件）|

**关键要点:**
- 表单中的 `button` 组件点击时会触发 `on_ui_action` 事件，但不会关闭对话框
- 所有带 `name` 字段的组件值都会包含在返回结果中
- 使用 `label` 组件可以创建视觉分组，提升用户体验
- 配合 `api.get_plugin_storage` 和 `api.set_plugin_storage` 实现配置持久化

---

## 7. 选中草稿信息（selectedDrafts）

当插件的 UI 动作注册在草稿管理页（`location="draft_action_bar"`）时，用户点击按钮后，`on_ui_action` 事件的 `params` 参数中会包含 `selectedDrafts` 字段，表示当前选中的草稿列表。

### 7.1 数据结构

`selectedDrafts` 是一个列表，每个元素代表一个选中的草稿，包含以下字段：

```python
{
    "name": "我的视频项目",          # 草稿名称
    "path": "C:/JianyingPro/Drafts/1234567890/draft_content.json",
    "folderPath": "C:/JianyingPro/Drafts/1234567890", # 草稿目录路径
    "isEncrypted": true
}
```

### 7.2 使用场景

- **批量操作**: 对多个选中的草稿执行相同的操作（如批量导出、批量处理）
- **上下文操作**: 根据用户选中的草稿提供相关功能
- **信息展示**: 显示选中草稿的摘要信息

### 7.3 注意事项

1. **空列表处理**: 如果用户未选中任何草稿，`selectedDrafts` 会是空列表 `[]`，插件应相应地给出提示。

2. **页面限制**: `selectedDrafts` 仅在草稿管理页（`draft_action_bar`）可用，在其他位置注册的按钮不会包含此字段。

3. **异步处理**: 处理多个草稿时，建议使用异步方式并及时更新用户界面，避免阻塞主程序。

### 7.4 示例代码

```python
import os
import json

@api.on("on_ui_action")
async def handle_draft_action(params):
    action_id = params.get("actionId")
    selected_drafts = params.get("selectedDrafts", [])

    if action_id == "show_draft_info":
        if not selected_drafts:
            api.alert("请先选择要查看的草稿！")
            return

        # 构建草稿信息摘要
        info_lines = [f"共选中 {len(selected_drafts)} 个草稿：\n"]
        for draft in selected_drafts:
            from datetime import datetime
            info_lines.append(f"\n• {draft['name']}")
            info_lines.append(f"  路径: {draft['folderPath']}")

        # 显示在自定义表单中（支持复制）
        api.show_custom_form({
            "title": "草稿信息",
            "items": [
                {
                    "type": "label",
                    "text": "".join(info_lines)
                }
            ]
        })

    elif action_id == "batch_analyze":
        if not selected_drafts:
            api.alert("请先选择要分析的草稿！")
            return

        # 更新按钮状态
        api.update_ui_action("batch_analyze", label="分析中...", icon="sync")

        results = []
        for idx, draft in enumerate(selected_drafts, 1):
            draft_folder = draft['folderPath']
            api.log(f"正在分析草稿 {idx}/{len(selected_drafts)}: {draft['name']}")

            try:
                # 读取草稿内容进行分析（使用 folderPath）
                file_path = os.path.join(draft_folder, "draft_content.json")
                raw_content = api.read_draft_file(file_path)
                draft_json = json.loads(raw_content)

                # 这里可以添加实际的分析逻辑
                # 例如: 分析轨道数量、素材数量等
                track_count = len(draft_json.get("tracks", []))
                duration_us = draft_json.get("duration", 0)
                duration_sec = duration_us / 1000000  # 微秒转秒

                analysis_result = {
                    "name": draft['name'],
                    "track_count": track_count,
                    "duration": f"{int(duration_sec // 60):02d}:{int(duration_sec % 60):02d}"
                }
                results.append(analysis_result)

                api.show_notification(f"已完成: {draft['name']} ({idx}/{len(selected_drafts)})", type="info")

            except Exception as e:
                api.log(f"分析草稿 {draft['name']} 时出错: {str(e)}")

        # 恢复按钮状态
        api.update_ui_action("batch_analyze", label="批量分析", icon="search")

        # 显示分析结果
        if results:
            result_text = "\n".join(
                f"{r['name']}: {r['track_count']} 轨道, 时长 {r['duration']}"
                for r in results
            )
            api.alert(result_text, title="分析结果")
        else:
            api.alert("所有草稿分析失败，请查看日志了解详情。", title="分析结果")
```
