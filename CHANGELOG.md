# 工作备注 / 更新日志

## 2026-07-27 — 多用户系统 + AI 聊天重构 + 删除会话功能

### 第一阶段：多用户注册/登录系统（first.md）

**目标**：将单用户模式升级为多用户系统，每个用户数据完全隔离。

**改动要点**：

1. **用户管理系统**
   - 新增 `user_data/users.json` 存储注册用户信息
   - 密码使用 SHA-256 + 16 字节随机盐 + 1000 轮哈希加密
   - `register_user()` / `verify_login()` — 注册校验 + 登录验证
   - 首次启动自动备份旧数据到 `user_data/_old_backup/`

2. **DietManager 类重构**
   - 构造函数接收 `username` 参数
   - 数据路径改为 `user_data/{username}/records.json` 和 `user_data/{username}/profile.json`
   - 类级别实例缓存 `DietManager._instances`，线程安全 `get_instance()`

3. **核心函数改造**
   - 所有业务函数新增 `username` 参数（`main_predict`, `query_records`, `delete_record_handler`, `create_dashboard`, `export_report`, `chat_with_ai` 等）
   - 通过 `DietManager.get_instance(username)` 动态获取用户专属实例

4. **登录/注册界面**
   - 自定义绿色卡片式登录页（主色 `#059669`，圆角阴影）
   - 登录 ↔ 注册表单一键切换
   - 使用 `gr.State` 管理会话，通过 `visible` 属性控制登录/主界面切换
   - 右上角显示用户名 + 退出登录按钮

5. **安全措施**
   - `secrets.compare_digest` 防时序攻击
   - `secrets.token_hex(16)` 生成随机盐
   - 所有文件读写加线程锁

### 第二阶段：AI 聊天界面重构（second.md）

**目标**：将 `gr.ChatInterface` 替换为参考 Ollama 风格的自定义聊天布局。

**改动要点**：

1. **聊天历史持久化**
   - 新增 `user_data/{username}/chat_history.json`
   - 数据结构：多会话（sessions），每个会话含 `session_id`、`title`、`messages`、时间戳
   - 15 个新增函数管理完整的 CRUD 操作
   - 文件损坏自动重建，备份 `.corrupted` 文件

2. **界面布局**
   - 左侧边栏（220px）：新建聊天按钮 + 历史会话列表（`gr.Dataframe`）
   - 右侧主聊天区：会话标题 + `gr.Chatbot`（带头像） + 输入框 + 发送按钮
   - 自定义 CSS：翠绿色主题、悬停高亮、选中行样式

3. **交互逻辑**
   - 新建聊天：创建新会话 → 清空聊天区 → 刷新侧边栏
   - 切换会话：点击侧边栏条目 → 加载完整消息历史
   - 发送消息：**流式输出** AI 回复 → 结束后自动持久化到 JSON → 刷新侧边栏
   - 首条用户消息自动成为会话标题（截断至 20 字）
   - 登录时自动初始化聊天状态

4. **Gradio 6 兼容性修复**
   - `theme` / `css` 参数从 `Blocks()` 移至 `launch()`
   - 移除不支持的 `show_copy_button` 参数
   - `init_chat_state` 改用 `.then()` 链式调用，避免 `current_user.change` 跨层级更新导致 UI 卡死
   - `gr.Dataframe` 简化参数（移除 `interactive=False` 和 `datatype`，添加初始 `value`）

### 第三阶段：删除会话功能（third.md）

**目标**：为聊天侧边栏增加删除指定会话功能。

**方案选择**：1A（侧边栏底部按钮 + 选中删除）+ 2B（确认对话框）+ 3A（删当前→切首个）+ 4A（自动刷新列表）+ 5C（允许删最后一个）+ 6A（文字反馈）

**改动要点**：

1. **删除逻辑**
   - `delete_chat_session()` — 从 JSON 中移除会话
   - 删除当前会话 → 自动切换到第一个剩余会话
   - 删除非当前会话 → 当前会话保持不变
   - 删除最后一个会话 → 清空聊天区，显示"请新建会话"

2. **确认对话框**
   - 红色警告样式（`#fef2f2` 背景，`#dc2626` 文字）
   - 显示会话标题 + "此操作不可恢复" 警告
   - 「✅ 确认删除」/「❌ 取消」两个按钮

3. **界面新增**
   - `🗑️ 删除当前会话` 按钮（`variant="stop"`，红色系）
   - 确认对话框组件组（默认隐藏）
   - 操作反馈消息区域（绿色提示）

4. **边界处理**
   - `load_user_chat_history` 不再对空 sessions 自动重建（符合"允许空列表"预期）
   - 所有聊天 handler 加 try/except 保护
   - 退出登录时清理聊天组件状态

---

## 已知问题 & 后续优化方向

- [ ] 聊天界面在小屏幕（<768px）下的响应式适配
- [ ] 头像文件过大（2.1MB `user.png`）可压缩优化
- [ ] 登录页可增加"记住我"功能
- [ ] AI 聊天可支持切换不同模型
- [ ] 可增加会话重命名功能
- [ ] 营养数据库可扩展更多食物

---

## 技术债务

- `foodManager.py` 为单文件架构（~2300 行），后续可拆分为模块化结构
- 头像路径硬编码（`E:\4C_race\picture\`），可改为相对路径或配置项
- HGraph 进度图使用 Matplotlib 渲染，可考虑迁移到 Plotly 实现交互式图表
