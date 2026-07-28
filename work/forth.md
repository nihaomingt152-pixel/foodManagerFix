
---

## 修改目标
为 `foodManager.py` 增加动态头像功能：  
- 用户登录后，根据个人档案中的性别（男/女）自动切换聊天界面的**用户头像**（`boy.jpg` / `girl.jpg`）；  
- 用户修改性别档案后，头像**实时更新**（无需重新登录）；  
- 医生头像固定为 `doctor.jpg`；  
- 头像文件缺失时自动降级（使用另一性别头像，若都缺失则显示默认占位）；  
- 使用**绝对路径**构造图片地址，保证在不同运行环境下正常显示。

---

## 需要修改的文件
- `foodManager.py`（项目根目录）

---

## 实施步骤

### 1. 在文件开头添加全局常量（在 `import` 之后）
```python
# 图片目录（绝对路径）
PICTURE_DIR = os.path.join(os.path.dirname(__file__), "picture")
```

### 2. 添加辅助函数 `update_chat_avatar(username)`
该函数返回一个 `gr.update` 对象，用于更新 `chat_bot` 的 `avatar_images` 属性。  
逻辑：  
- 若用户未登录或用户名无效，默认使用 `boy.jpg`（作为后备）。  
- 否则，从 `DietManager` 加载用户档案，读取 `gender` 字段（默认 `"男"`）。  
- 根据性别选择 `boy.jpg` 或 `girl.jpg`，构造绝对路径。  
- 若目标头像文件不存在，尝试使用另一性别头像；若仍不存在，则传空字符串（Gradio 会显示默认占位图标）。  
- 医生头像固定为 `doctor.jpg`，同样检查存在性。

**将以下代码插入到 `update_profile_handler` 函数之后（或任意函数定义区域）**：
```python
def update_chat_avatar(username):
    """返回 gr.update 用于动态修改 chat_bot 的头像"""
    doctor_path = os.path.join(PICTURE_DIR, "doctor.jpg")
    if not os.path.exists(doctor_path):
        doctor_path = ""  # 若医生图片缺失，留空

    # 默认使用男头像
    default_user_path = os.path.join(PICTURE_DIR, "boy.jpg")
    if not username:
        # 未登录时使用默认男头像
        return gr.update(avatar_images=(default_user_path if os.path.exists(default_user_path) else "", doctor_path))

    # 获取性别
    dm = DietManager.get_instance(username)
    gender = dm.profile.get("gender", "男")  # 默认男性
    selected = "boy.jpg" if gender == "男" else "girl.jpg"
    user_path = os.path.join(PICTURE_DIR, selected)

    # 若选择的文件不存在，尝试使用另一个性别头像作为降级
    if not os.path.exists(user_path):
        fallback = "girl.jpg" if gender == "男" else "boy.jpg"
        fallback_path = os.path.join(PICTURE_DIR, fallback)
        if os.path.exists(fallback_path):
            user_path = fallback_path
        else:
            user_path = ""  # 无可用头像，留空

    return gr.update(avatar_images=(user_path, doctor_path))
```

---

### 3. 修改登录与注册成功后的事件链
**登录按钮** (`login_btn.click`) 当前链为：
```python
login_btn.click(
    fn=handle_login,
    inputs=[login_username, login_password],
    outputs=[current_user, login_block, main_block, login_form, register_form, auth_message]
).then(
    fn=init_chat_state,
    inputs=[current_user],
    outputs=[current_session_id, session_list, session_ids_state, session_title, chat_bot]
)
```
**在此链末尾追加一个 `.then`**，用于更新头像：
```python
).then(
    fn=update_chat_avatar,
    inputs=[current_user],
    outputs=[chat_bot]
)
```

**注册按钮** (`reg_btn.click`) 同理，在 `init_chat_state` 之后追加：
```python
reg_btn.click(
    fn=handle_register,
    inputs=[reg_username, reg_password, reg_confirm],
    outputs=[current_user, login_block, main_block, login_form, register_form, auth_message]
).then(
    fn=init_chat_state,
    inputs=[current_user],
    outputs=[current_session_id, session_list, session_ids_state, session_title, chat_bot]
).then(
    fn=update_chat_avatar,
    inputs=[current_user],
    outputs=[chat_bot]
)
```

---

### 4. 修改退出登录事件
退出登录时应将头像重置为默认（男）。在 `logout_btn.click` 的现有链末尾，追加一个更新头像的调用（传入空用户名）：
```python
logout_btn.click(
    fn=handle_logout,
    inputs=None,
    outputs=[current_user, login_block, main_block, login_form, register_form, auth_message]
).then(
    fn=lambda: ("", pd.DataFrame({"💬 会话": [], "🕒 更新时间": []}), [], "", []),
    inputs=None,
    outputs=[current_session_id, session_list, session_ids_state, session_title, chat_bot]
).then(
    fn=update_chat_avatar,
    inputs=[gr.State("")],  # 传入空字符串表示未登录
    outputs=[chat_bot]
)
```
注意：`gr.State("")` 需要改为直接传入 `gr.State("")` 或使用 `fn=lambda: update_chat_avatar("")`，但最简单是在 `.then` 中指定 `inputs=[gr.State("")]`，但 `gr.State("")` 不是组件，我们可以用 `inputs=None`，并修改 `update_chat_avatar` 以处理 `username=None`。为统一，我们可让 `update_chat_avatar` 接受 `username=None`，在内部处理。建议在函数开头增加 `if not username:` 即可。所以退出时调用 `update_chat_avatar(None)`。  
可以改为：
```python
).then(
    fn=update_chat_avatar,
    inputs=[gr.State("")],  # 使用一个空状态
    outputs=[chat_bot]
)
```
但 `gr.State("")` 不是一个输入组件？实际上，可以在事件绑定中使用 `gr.State("")` 作为输入，但需要先定义该状态。更简单的方法：直接定义一个返回 `gr.update` 的函数，不依赖输入。但为保持一致性，我们可以在退出事件中直接调用 `update_chat_avatar` 并传入空字符串。因为 `update_chat_avatar` 接受 `username` 参数，我们可以使用 `fn=lambda: update_chat_avatar("")`，但这样无法指定 inputs。更合适的是使用 `inputs=[gr.State("")]`，但需要先定义 `empty_state = gr.State("")`，不过我们可以直接利用 `current_user` 在退出后变为空，但事件链中 `current_user` 已经更新为空，所以我们可以在 `.then` 中指定 `inputs=[current_user]`，因为 `current_user` 在此时已经是空字符串（因为 handle_logout 将其输出为空）。所以我们可以直接复用 `current_user`：
```python
).then(
    fn=update_chat_avatar,
    inputs=[current_user],  # 此时 current_user 已为空
    outputs=[chat_bot]
)
```
这样更简洁，因为 `handle_logout` 已经将 `current_user` 设为空字符串。

---

### 5. 修改档案更新事件
在 `profile_btn.click` 的现有链中，更新档案成功后，也应更新头像。`profile_btn.click` 目前是：
```python
profile_btn.click(
    fn=update_profile_handler,
    inputs=[...],
    outputs=profile_output
)
```
我们需要在其后追加 `.then`，调用 `update_chat_avatar`，确保头像随性别更新：
```python
profile_btn.click(
    fn=update_profile_handler,
    inputs=[current_user, name_input, height_input, profile_weight_input, age_input,
            gender_input, activity_input, goal_input, disease_input],
    outputs=profile_output
).then(
    fn=update_chat_avatar,
    inputs=[current_user],
    outputs=[chat_bot]
)
```

---

### 6. （可选）初始化聊天状态时不需要更新头像，因为登录时已经更新，但为了保险，可以在 `init_chat_state` 中不涉及头像。我们已通过登录事件更新。

---

## 注意事项
- 确保 `picture` 目录下存在 `boy.jpg`、`girl.jpg`、`doctor.jpg` 三个文件（您已确认有前两个，`doctor.jpg` 也存在）。
- 若图片文件缺失，代码会静默降级，不影响程序运行（符合您的选择 3B）。
- 头像更新仅影响当前已登录用户的 `chat_bot` 组件，新建会话或切换会话时头像保持不变（因为头像只与用户绑定）。

---

## 预期效果
- 登录时，男性用户看到 `boy.jpg`，女性用户看到 `girl.jpg`。
- 若用户修改性别并保存，聊天界面头像立即刷新为新性别对应的图片。
- 退出登录后，头像恢复为默认的 `boy.jpg`。

---

