
---

## 修改目标
让“个人档案与推荐设置”标签页的**所有输入框**实现实时同步：
1. **登录/注册后**：自动将用户已保存的档案数据填充到输入框（若未设置则使用默认值）。
2. **点击“更新个人档案”后**：输入框立即刷新为新保存的值。
3. **推荐摄入量文本框**（`profile_output`）同步更新（当前已有，保持不变）。
4. **性别修改后**：聊天头像同步更新（已实现，但需确保顺序正确）。

---

## 需要修改的文件
- `foodManager.py`（项目根目录）

---

## 实施步骤

### 步骤 1：新增辅助函数 `load_profile_to_form(username)`
将该函数插入到 `update_chat_avatar` 函数之后（或任意函数定义区域），用于批量更新档案输入框的值。

```python
def load_profile_to_form(username):
    """
    根据用户名加载个人档案，返回所有档案输入框的 gr.update 对象
    用于登录时自动填充 & 保存后刷新输入框
    """
    if not username:
        # 未登录时使用默认值（与界面初始值一致）
        return (
            gr.update(value="用户"),
            gr.update(value=170),
            gr.update(value=65),
            gr.update(value=20),
            gr.update(value="男"),
            gr.update(value="轻度活动"),
            gr.update(value="维持体重"),
            gr.update(value="健康")
        )
    
    dm = DietManager.get_instance(username)
    if dm is None:
        return load_profile_to_form("")
    
    p = dm.profile
    return (
        gr.update(value=p.get("name", username)),
        gr.update(value=p.get("height", 170)),
        gr.update(value=p.get("weight", 65)),
        gr.update(value=p.get("age", 20)),
        gr.update(value=p.get("gender", "男")),
        gr.update(value=p.get("activity", "轻度活动")),
        gr.update(value=p.get("goal", "维持体重")),
        gr.update(value=p.get("disease", "健康"))
    )
```

---

### 步骤 2：登录按钮事件链追加“加载档案到输入框”
找到 `login_btn.click` 的现有链，在其末尾追加 `.then` 调用 `load_profile_to_form`。

**修改前：**
```python
login_btn.click(
    fn=handle_login,
    inputs=[login_username, login_password],
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

**修改后（追加一个 `.then`）：**
```python
login_btn.click(
    fn=handle_login,
    inputs=[login_username, login_password],
    outputs=[current_user, login_block, main_block, login_form, register_form, auth_message]
).then(
    fn=init_chat_state,
    inputs=[current_user],
    outputs=[current_session_id, session_list, session_ids_state, session_title, chat_bot]
).then(
    fn=update_chat_avatar,
    inputs=[current_user],
    outputs=[chat_bot]
).then(
    fn=load_profile_to_form,
    inputs=[current_user],
    outputs=[name_input, height_input, profile_weight_input, age_input,
             gender_input, activity_input, goal_input, disease_input]
)
```

---

### 步骤 3：注册按钮事件链同样追加
`reg_btn.click` 做完全相同的追加。

**修改后：**
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
).then(
    fn=load_profile_to_form,
    inputs=[current_user],
    outputs=[name_input, height_input, profile_weight_input, age_input,
             gender_input, activity_input, goal_input, disease_input]
)
```

---

### 步骤 4：档案更新按钮事件链追加“刷新输入框”
`profile_btn.click` 目前是：
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

**修改为（在更新头像之后，追加刷新输入框）：**
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
).then(
    fn=load_profile_to_form,
    inputs=[current_user],
    outputs=[name_input, height_input, profile_weight_input, age_input,
             gender_input, activity_input, goal_input, disease_input]
)
```

---

### 步骤 5：退出登录时重置输入框为默认值
在 `logout_btn` 的链中，追加一个重置输入框的调用。当前退出链为：
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
    inputs=[current_user],
    outputs=[chat_bot]
)
```

**在末尾追加：**
```python
).then(
    fn=load_profile_to_form,
    inputs=[gr.State("")],  # 传入空字符串，重置为默认值
    outputs=[name_input, height_input, profile_weight_input, age_input,
             gender_input, activity_input, goal_input, disease_input]
)
```

---

## 预期效果
- ✅ 登录后：档案输入框自动显示用户已保存的数据（性别、身高、体重等）。
- ✅ 修改档案并保存：输入框立即显示新值，推荐摄入量文本框同步更新，头像同步切换。
- ✅ 退出登录：所有输入框重置为界面默认值（姓名“用户”、身高170等）。
- ✅ 新注册用户：输入框显示默认值（姓名=用户名，其余为默认），用户可随时修改。

---
