
---

> **任务目标**：改造 `foodManager.py` 中的 **“💬 AI 专属健康咨询”** 标签页，使其界面布局和交互逻辑**参考 Ollama 官方聊天界面**（你提供的 `image.png` 截图风格），但**保留现有翠绿（emerald）浅色主题**。
>
> ---
>
> ### 1. 界面布局（纯 Gradio 组件 + 自定义 CSS）
>
> 当前该标签页使用 `gr.ChatInterface`，请**替换**为自定义 `gr.Blocks` 布局，要求如下：
>
> - **整体采用左右两栏结构**（使用 `gr.Row`）：
>   - **左侧侧边栏**（`gr.Column(scale=1, min_width=200)`）：
>     - 顶部放置一个 **“➕ 新建聊天”** 按钮（`gr.Button`，样式为主色）。
>     - 下方放置一个 **历史会话列表**（使用 `gr.Accordion` 或 `gr.DataFrame`，推荐用 `gr.Dataframe` 或 `gr.Listbox` 配合 `gr.HTML` 渲染，建议用 `gr.Dataframe` 显示会话标题、最后更新时间）。
>     - 侧边栏高度应填满父容器，并可滚动。
>   - **右侧主聊天区**（`gr.Column(scale=4)`）：
>     - 顶部显示当前会话标题（如“聊天 - 2026-07-27 14:30”），使用 `gr.HTML` 或 `gr.Markdown` 展示。
>     - 中间区域为 **消息展示区**（`gr.Chatbot`，`height=500` 或 `flex-grow`），保留原有的头像设置（`avatar_images`）。
>     - 底部为输入框（`gr.Textbox`）+ 发送按钮（`gr.Button`）。
>
> ---
>
> ### 2. 数据存储与状态管理（基于 JSON 持久化）
>
> - **存储路径**：`user_data/{username}/chat_history.json`
> - **数据结构**（请按此格式设计）：
>   ```json
>   {
>     "sessions": [
>       {
>         "session_id": "uuid或时间戳",
>         "title": "会话摘要（取第一条用户消息，截断至20字）",
>         "created_at": "ISO时间",
>         "updated_at": "ISO时间",
>         "messages": [
>           {"role": "user", "content": "你好"},
>           {"role": "assistant", "content": "您好！我是您的营养医师..."}
>         ]
>       }
>     ],
>     "current_session_id": "当前打开的会话ID"
>   }
>   ```
> - **初始化**：如果用户没有 `chat_history.json`，自动创建一个空结构，并默认生成一个标题为“新对话”的空会话。
>
> ---
>
> ### 3. 核心交互逻辑（事件绑定）
>
> **3.1 “新建聊天”按钮**
> - 点击后，在当前 `chat_history.json` 中**新增一个空会话**（`messages=[]`），标题自动生成为“新对话 YYYY-MM-DD HH:MM”。
> - 将当前会话 ID 切换为新会话的 ID。
> - **清空右侧 `Chatbot` 组件**（重置为空列表）。
> - **刷新侧边栏的历史会话列表**（显示最新会话条目）。
> - *注意：旧会话内容必须保留在 JSON 文件中，不能丢失。*
>
> **3.2 点击侧边栏历史会话条目**
> - 用户点击某个历史会话时（可以用 `gr.Dataframe` 的 `select` 事件，或 `gr.Listbox` 的 `change` 事件）。
> - 从 `chat_history.json` 中读取该会话的 `messages` 列表。
> - **将右侧 `Chatbot` 的内容更新为该会话的完整消息历史**（按顺序显示）。
> - 更新顶部标题为对应会话的 `title`。
> - 更新 `current_session_id` 为当前选中的 ID。
>
> **3.3 发送消息（AI 对话）**
> - **保留现有的 `chat_with_ai` 函数不变**（输入输出格式不变），它仍然是流式生成器。
> - 用户点击发送或按回车时，需要执行以下步骤：
>   1. 获取当前会话 ID（从 `gr.State` 或 JSON 中读取 `current_session_id`）。
>   2. 将用户输入的消息**实时追加到**该会话的 `messages` 列表中（先存用户消息）。
>   3. 调用 `chat_with_ai`（传入 `user_message`、历史对话、`username`），接收流式输出。
>   4. 将 AI 的回复**逐条追加到**同一个会话的 `messages` 列表中。
>   5. 更新会话的 `updated_at` 时间戳。
>   6. 将更新后的整个会话写回 `chat_history.json`。
>   7. 同步刷新侧边栏（更新时间戳或标题）。
> - *重要*：每次消息接收完成后，需要**持久化存储**，保证刷新页面或重启程序后历史记录不丢失。
>
> **3.4 切换会话时的上下文加载**
> - 点击历史会话时，除了更新 `Chatbot` 的显示内容外，**不需要额外修改 `chat_with_ai` 的 system prompt 拼接逻辑**（因为 `chat_with_ai` 内部会通过 `history` 参数接收对话历史，我们只需将对应会话的 `messages` 列表按 `[user, assistant]` 交替格式传给 `chat_with_ai` 即可）。
> - 具体实现：在发送消息时，从 JSON 中加载当前会话的 `messages`，转换成 `history` 列表格式（`[("user msg", "assistant msg"), ...]`），传给 `chat_with_ai`。
>
> ---
>
> ### 4. 视觉与 CSS 约束
>
> - **配色方案**：沿用当前 `health_theme` 的翠绿（emerald）浅色主题，不要引入深色模式。
> - **侧边栏样式**：
>   - 背景色为极浅的灰色（`#f1f5f9`）或透明，与主聊天区区分。
>   - 会话列表条目采用卡片样式，悬停时有浅绿色背景反馈。
>   - 当前选中的会话条目使用绿色边框或浅绿色背景高亮。
> - **布局适配**：确保在 1280px 以上分辨率时侧边栏宽度约 220px，主聊天区自适应；小屏幕下（如 768px）可考虑隐藏侧边栏（暂不强制移动端适配，但要求不出现严重错位）。
> - **图标**：可以在按钮或标题中嵌入 Unicode 符号（如 `➕`、`💬`、`🕒`），不必引入额外图标库。
>
> ---
>
> ### 5. 代码改造指引（具体位置）
>
> - 定位到 `foodManager.py` 文件中 **`with gr.Tab("💬 AI 专属健康咨询"):`** 下方的代码块。
> - 将现有的 `gr.ChatInterface` 及其相关组件**整体替换**为新的自定义布局。
> - 新增函数：
>   - `load_user_chat_history(username)`：加载 JSON
>   - `save_user_chat_history(username, data)`：保存 JSON
>   - `get_current_session(username)`：获取当前会话
>   - `switch_session(username, session_id)`：切换会话并返回聊天记录
>   - `create_new_session(username)`：新建会话
> - 在 `main_block` 作用域内新增必要的 `gr.State`（如 `current_session_id`）。
> - 确保所有新增函数都位于 `chat_with_ai` 附近，并正确调用 `DietManager` 或其他已有模块。
>
> ---
>
> ### 6. 额外要求（重要）
>
> - **不要破坏现有功能**：其他标签页（今日概览、识别、档案、记录）保持不变。
> - **不要在 `chat_with_ai` 函数内部做侵入式修改**，保持其接口不变（只依赖 `user_message`, `history`, `username`）。
> - **确保多用户隔离**：不同用户的 `chat_history.json` 存储在不同子目录下，通过 `username` 区分。
> - **防错处理**：若 JSON 文件损坏或被意外删除，自动重建默认数据结构，避免程序崩溃。
> - 提供**必要的注释**，方便后续维护。
>
> ---
>
> 请根据以上要求，生成完整的代码修改方案（可直接替换原标签页部分），如果涉及新增工具函数，请一并提供。

---
