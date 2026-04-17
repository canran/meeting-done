---
name: meeting-done
description: 「会议开完了」- 会议全生命周期管理自动化技能。触发条件：(1)用户说「开完会了」「会议结束了」触发会后整理分支；(2)用户说「帮我预定/预约/安排一个会议」触发会前预约分支。两个分支独立运行。
---

# 会议开完了 · meeting-done

> 会议全生命周期管理自动化技能。一句话触发，从预约、转录、整理、归档到提醒的完整闭环。

---

## ⚙️ 配置文件（使用前必读）

以下为个人化配置项，首次使用前需根据你的环境填写：

```yaml
# === 腾讯文档 ===
EXCEL_FILE_ID: ""      # 腾讯文档 Excel 索引表的 file_id（从 URL 中获取）
EXCEL_SHEET_ID: ""      # 索引表的工作表 ID

# === 邮件 ===
SMTP_EMAIL: ""          # 你的发件邮箱地址
SMTP_SCRIPT: "~/Library/Application Support/QClaw/openclaw/config/skills/imap-smtp-email/scripts/smtp.js"

# === 联系人库 ===
CONTACTS_FILE: "memory-tdai/联系人.md"   # 参会人姓名→邮箱映射文件

# === 会议模板 ===
TEMPLATE_FILE: "memory-tdai/会议模板/会议纪要模板.md"
```

### 联系人库格式（`memory-tdai/联系人.md`）

```markdown
# 联系人库

## 姓名
- 邮箱: xxx@example.com
- 备注: 部门/职位

## 张三
- 邮箱: zhangsan@example.com
```

---

## 📍 两套独立工作流

| 触发语 | 工作流 | 交互方式 |
|--------|--------|----------|
| 「开完会了」「会议结束了」 | **分支 A：会后整理** | 全自动 + 问三个问题（确认纪要、发邮件、预定下次） |
| 「帮我预定/预约/安排一个会议」 | **分支 B：会前预约** | 收集主题、时间后执行 |

---

## 🔷 分支 A：会后整理

**触发语：** 「开完会了」「会议结束了」

### A1. 自动拉取最近会议信息

```
tool: mcporter
action: invoke
provider: tencent-meeting-mcp
command: get_user_meeting_list
args:
  status: ended
  page_size: 1
  page: 0
```

提取：`meeting_id`、`subject`、`start_time`、`end_time`、`attendees`

---

### A2. 获取智能纪要 + 转录原文

**A2.1 AI 智能纪要：**
```
tool: mcporter
action: invoke
provider: tencent-meeting-mcp
command: get_meeting_ai_minutes
args:
  meeting_id: "{meeting_id}"
```

**A2.2 转录原文：**
```
tool: mcporter
action: invoke
provider: tencent-meeting-mcp
command: search_transcription
args:
  meeting_id: "{meeting_id}"
  keyword: ""
```

> 📌 智能纪要作为文档主体，转录原文作为附录参考。

---

### A2.5. 展示纪要预览，等你确认

**获取内容后，先展示预览，不直接写入文档：**

```
━━━━━━━━━━━━━━━━━━━━━━━━
📋 {会议主题}
━━━━━━━━━━━━━━━━━━━━━━━━
• 日期：{日期}
• 时间：{开始时间} - {结束时间}
• 参会人：{参会人}

📝 本次会议目标：
{从智能纪要提取}

📋 议程 & 结论：
{从智能纪要提取}

✅ 待办事项：
{从智能纪要提取}
━━━━━━━━━━━━━━━━━━━━━━━━
```

**等你说「可以」「OK」「确认」后，继续执行后续步骤。**
你说「修改xxx」→ 根据你的反馈调整，再次展示确认。

> ⚠️ 确认通过后才写入文档和 Excel，防止错误内容沉淀。

---

### A3. 匹配参会人邮箱

从 `memory-tdai/联系人.md` 匹配姓名→邮箱：
- 找到 → 使用对应邮箱
- 找不到 → 问你：「[{姓名}] 的邮箱没找到，请告诉我。」

> 记录到联系人库，下次无需再问。

---

### A4. 创建会议纪要文档

```
tool: mcporter
action: invoke
provider: tencent-docs
command: manage.create_file
args:
  name: "{会议主题} · {日期}"
  type: doc
  parent_folder_id: "~"
```

**写入内容：**

```markdown
# {会议主题}

**日期：** {日期}
**时间：** {开始时间} - {结束时间}
**参会人：** {参会人}
**会议号：** {会议号}

---

## 📌 本次会议目标
（从智能纪要中提取）

## 📋 议程
（从智能纪要中提取）

## ✅ 结论 & 待办事项
| 事项 | 负责人 | 截止时间 |
|------|--------|----------|
（从智能纪要待办事项中提取）

## 📝 转录原文（参考）
（转录原文内容）

## 📅 下次会议
（待定）
```

---

### A5. 写入 Excel 索引表

```
tool: mcporter invoke tencent-sheetengine.set_range_value_by_csv
csv_data: "{日期},{主题},{开始时间},{结束时间},{参会人},📄 打开纪要,"
file_id: {EXCEL_FILE_ID}
sheet_id: {EXCEL_SHEET_ID}
start_col: 0
```

```
tool: mcporter invoke tencent-sheetengine.set_link
file_id: {EXCEL_FILE_ID}
row: {row_count + 1}
col: 5
display_text: "📄 打开纪要"
link_id: "{纪要文档URL}"
```

---

### A6. 发送邮件确认

**发送前先问你：**

> 「要发送邮件给 {参会人}（{邮箱}）吗？」

确认后执行：
```
tool: exec
command: node {SMTP_SCRIPT} send \
  --to {参会人邮箱} \
  --subject "📋 会议纪要 · {会议主题} · {日期}" \
  --body "{邮件正文}"
```

---

### A7. 输出执行报告

```
━━━━━━━━━━━━━━━━━━━━━━━━
📋 会议整理完成报告
━━━━━━━━━━━━━━━━━━━━━━━━
✅ 会议纪要文档：{文档链接}
✅ Excel 索引表：已追加第 {行号} 行
✅ 邮件发送：已发送给 {参会人}
━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### A8. 问：要预定下次会议吗？

> 「要预定下次会议吗？如果要，告诉我时间。」

- 用户说时间 → 触发分支 B，预约完成后更新 Excel"下次会议"列
- 用户说「不用了」→ 结束

---

## 🔷 分支 B：会前预约

**触发语：** 「帮我预定一个会议」

### B1. 收集主题和时间

```
AI：什么主题的会议？
用户：{主题}
AI：几点开始？时长多久？
用户：{时间}
```

---

### B2. 预约腾讯会议 + 开启转录

```
tool: mcporter invoke tencent-meeting-mcp.create_meeting
subject: "{会议主题}"
start_time: "{ISO格式}"
duration: 60
attendees: ["{邮箱}"]
enable_transcription: true
```

---

### B3. 创建纪要文档占位

```
tool: mcporter invoke tencent-docs.manage.create_file
name: "{会议主题} · {日期}"
type: doc
parent_folder_id: "~"
```

---

### B4. 会前提醒（提前5分钟 + 上次纪要）

**B4.1 从 Excel 索引表获取上次纪要：**
```
tool: mcporter invoke tencent-sheetengine.get_range_value
file_id: {EXCEL_FILE_ID}
sheet_id: {EXCEL_SHEET_ID}
start_row: 1
start_col: 0
end_row: 1
end_col: 6
```

**B4.2 获取上次纪要内容：**
```
tool: mcporter invoke tencent-docs.manage.get_doc_content
file_id: "{上次纪要file_id}"
```

**B4.3 设置提醒（提前5分钟）：**
```
tool: cron action: add
job:
  schedule:
    kind: at
    at: "{会议开始时间 - 5分钟}"
  payload:
    kind: agentTurn
    message: |
      ⏰ 会议即将在5分钟后开始！

      📋 本次会议：{主题}
      🔢 会议号：{会议号}
      🔗 加入：{join_url}

      📝 上次会议纪要：{上次主题}
      📌 待对齐事项：{从上次纪要提取的待办事项}
  sessionTarget: main
```

---

### B5. 执行报告

```
━━━━━━━━━━━━━━━━━━━━━━━━
📋 会议预约完成报告
━━━━━━━━━━━━━━━━━━━━━━━━
✅ 腾讯会议：{主题}，会议号：{会议号}
✅ 纪要文档（占位）：{文档链接}
⏰ 会前提醒：已设置（提前5分钟，含上次纪要）
━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## ⚠️ 关键原则

1. **分支 A 半自动**：自动拉取会议 → 展示预览确认 → 写入文档/Excel → 问两个问题
2. **三个确认点**：纪要预览确认 → 发邮件确认 → 要预定下次会议吗
3. **参会人邮箱**：自动匹配联系人库，匹配不上才问
4. **会议内容**：智能纪要 + 转录原文都要获取
5. **先审核再沉淀**：确认通过后才写入文档
6. **下次会议可选**：AI 主动问，用户决定
