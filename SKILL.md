---
name: meeting-done
description: 「会议开完了」- 会议全生命周期管理自动化技能。触发条件：(1)用户说「开完会了」「会议结束了」触发会后整理分支；(2)用户说「帮我预定/预约/安排一个会议」触发会前预约分支。两个分支独立运行。
---

# 会议开完了 · meeting-done

## 📍 两套独立工作流

| 触发语 | 工作流 | 交互方式 |
|--------|--------|----------|
| 「开完会了」「会议结束了」 | **分支 A：会后整理** | 全自动 + 问三个问题（确认纪要、发邮件、预定下次） |
| 「帮我预定/预约/安排一个会议」 | **分支 B：会前预约** | 收集主题、时间后执行 |

---

## 🔷 分支 A：会后整理（全自动模式）

### 触发条件
用户说：「开完会了」「会议结束了」「整理一下刚才的会议」

---

### A1. 自动拉取最近会议信息

**用户说「开完会了」后，AI 自动执行，不问任何问题：**

**Step 1：查询已结束会议列表**

```
bash -c 'TENCENT_MEETING_TOKEN=$(bash "{{QCLAW_HOME}}/skills/tencent-meeting-mcp/get-token.sh") mcporter call tencent-meeting-mcp get_user_ended_meetings --args "{\"start_time\":\"{今天 00:00:00+08:00}\",\"end_time\":\"{今天 23:59:59+08:00}\",\"page_size\":10,\"page_number\":1}" --output json'
```

> ⚠️ 时间参数必须使用 ISO 8601 格式（`2026-04-17T00:00:00+08:00`），不接受纯日期字符串。

**Step 2：判断结果**

从返回的 `meeting_info_list` 中找到**按实际结束时间最近**的会议（比较 `end_time`，不是预定结束时间）。

**Step 3：如果列表为空或找不到最近会议 → 问会议号**

> 「已结束列表里没找到刚结束的会议（API 可能还有延迟）。请提供会议号（9位数字），我直接查询。」

用户回复会议号后：

```
bash -c 'TENCENT_MEETING_TOKEN=$(bash "{{QCLAW_HOME}}/skills/tencent-meeting-mcp/get-token.sh") mcporter call tencent-meeting-mcp get_meeting_by_code --args "{\"meeting_code\":\"{会议号}\"}" --output json'
```

> ⚠️ **重要**：`get_user_ended_meetings` API 按**预定结束时间**过滤，实际提前离会的会议不会立即出现。如果用户说刚开完会但列表查不到，**直接问会议号用 `get_meeting_by_code` 查**，不要在分页上浪费时间。

**从返回结果中提取：**
- `meeting_id` → 会议ID
- `meeting_code` → 会议号
- `subject` → 会议主题
- `start_time` / `end_time` → 开始/结束时间

---

### A2. 获取智能纪要 + 转录原文

**同时获取两种内容：**

**A2.1 AI 智能纪要（v1.0.6 工具名）：**

```
bash -c 'TENCENT_MEETING_TOKEN=$(bash "{{QCLAW_HOME}}/skills/tencent-meeting-mcp/get-token.sh") mcporter call tencent-meeting-mcp get_smart_minutes --args "{\"meeting_id\":\"{meeting_id}\"}" --output json'
```

**A2.2 转录原文（v1.0.6 工具名）：**

```
bash -c 'TENCENT_MEETING_TOKEN=$(bash "{{QCLAW_HOME}}/skills/tencent-meeting-mcp/get-token.sh") mcporter call tencent-meeting-mcp search_transcripts --args "{\"meeting_id\":\"{meeting_id}\",\"keyword\":\"\"}" --output json'
```

> 📌 将智能纪要作为文档主体，转录原文作为参考材料（默认不写入正文，用户要求时再附）。

**v1.0.6 工具速查：**
| 旧工具名 | v1.0.6 工具名 | 备注 |
|----------|-------------|------|
| `get_meeting_ai_minutes` | `get_smart_minutes` | 智能摘要 |
| `search_transcription` | `search_transcripts` | 转写原文 |
| `export_asr_details` | ~~已移除~~ | 不可用 |
| 新增 | `get_transcripts_paragraphs` | 分段转写 |
| 新增 | `get_transcripts_details` | 详细转写 |

---

### A2.5. 输出纪要预览，等用户确认

**获取内容后，先展示给你审核，不直接写入文档：**

> 「已拉取会议信息，生成纪要如下，请确认：」
>
> ```
> ━━━━━━━━━━━━━━━━━━━━━━━━
> 📋 {会议主题}
> ━━━━━━━━━━━━━━━━━━━━━━━━
> • 日期：{日期}
> • 时间：{开始时间} - {结束时间}
> • 参会人：{参会人}
>
> 📝 本次会议目标：
> {从智能纪要提取}
>
> 📋 议程 & 结论：
> {从智能纪要提取}
>
> ✅ 待办事项：
> {从智能纪要提取}
> ━━━━━━━━━━━━━━━━━━━━━━━━
> ```

**等待你的确认：**
- 你说「可以」「OK」「确认」→ 继续执行 A3 及后续步骤
- 你说「修改xxx」「不对」→ 根据你的反馈调整内容，调整后再次展示确认

> ⚠️ 确认通过后才写入文档和 Excel，防止内容错误沉淀

---

### A3. 匹配参会人邮箱

**自动匹配 `memory-tdai/联系人.md`：**

```
对于每个参会人姓名:
  1. 在联系人库中查找
  2. 找到 → 使用对应邮箱
  3. 找不到 → 记录到"待确认列表"
```

**如果有匹配不上的参会人：**

> 「参会人 [{姓名}] 的邮箱没找到，请告诉我邮箱地址。」

用户回复后：
- 记录到 `memory-tdai/联系人.md`
- 继续执行

---

### A4. 创建会议纪要文档

> ⚠️ **必须使用 `create_smartcanvas_by_mdx`**，不能用 `manage.create_file`（后者创建的是普通文档 tencentdoc，不支持 MDX 写入）。

```
bash -c 'TENCENT_MEETING_TOKEN=$(bash "{{QCLAW_HOME}}/skills/tencent-meeting-mcp/get-token.sh") mcporter call tencent-docs create_smartcanvas_by_mdx --args "{
  \"title\": \"{会议主题} · {日期}\",
  \"mdx\": \"---\ntitle: {会议主题} · {日期}\nicon: 📋\n---\n\n<Page>\n\n<Heading level=\\\"1\\\">\n    {会议主题}\n</Heading>\n\n<Callout icon=\\\"📅\\\" blockColor=\\\"light_blue\\\" borderColor=\\\"sky_blue\\\">\n    {日期} {开始时间} - {结束时间} &nbsp;|&nbsp; 会议号：{会议号} &nbsp;|&nbsp; 参会人：{参会人}\n</Callout>\n\n<Heading level=\\\"2\\\">\n    本次会议目标\n</Heading>\n\n<Paragraph>\n    {从智能纪要提取}\n</Paragraph>\n\n<Heading level=\\\"2\\\">\n    议程 & 结论\n</Heading>\n\n{从智能纪要提取：议程内容，使用 <NumberedList> 或 <BulletedList>}\n\n<Heading level=\\\"2\\\">\n    待办事项\n</Heading>\n\n{从智能纪要提取：待办，使用 <Table> (TableRow > TableCell)}\n\n<Heading level=\\\"2\\\">\n    下次会议\n</Heading>\n\n<Paragraph>\n    待定\n</Paragraph>\n\n</Page>\"
}" --output json'
```

> 📌 提取返回的 `file_id` 和 `url`。

**MDX 规范速查（必须遵守，否则文档组件无法解析）：**
- 块组件三段式多行（开标签 → 缩进内容 → 闭标签）
- 属性值用双引号：`level="1"`，禁止 `level={1}`
- `BulletedList` / `NumberedList`：每个元素是一个独立条目，不是容器
- `Mark bold` 布尔属性不写值，必须单行
- 禁止 Markdown 加粗 `**`，用 `<Mark bold>文本</Mark>`
- `Callout` 必须同时有 `blockColor` 和 `borderColor`
- 色值：`light_blue` / `sky_blue`（蓝）、`light_green` / `green`（绿）、`light_yellow` / `yellow`（黄）、`light_red` / `red`（红）、`light_purple` / `purple`（紫）、`light_orange` / `orange`（橙）

---

### A5. 写入 Excel 索引表

**固定资源：**
- `file_id: {{EXCEL_FILE_ID}}`
- `sheet_id: {{EXCEL_SHEET_ID}}`

> ⚠️ **所有 sheetengine 调用必须传 `sheet_id`**，虽然工具签名不暴露，但服务端强制要求。

**A5.1 读取现有行数：**

```
bash -c 'TENCENT_MEETING_TOKEN=$(bash "{{QCLAW_HOME}}/skills/tencent-meeting-mcp/get-token.sh") mcporter call tencent-sheetengine get_cell_data --args "{\"file_id\":\"{{EXCEL_FILE_ID}}\",\"sheet_id\":\"{{EXCEL_SHEET_ID}}\",\"start_row\":0,\"end_row\":99,\"start_col\":0,\"end_col\":6,\"return_csv\":true}" --output json'
```

解析 CSV，得出「最后一个非空行号 + 1」= `next_row`。

**A5.2 追加数据：**

```
bash -c 'TENCENT_MEETING_TOKEN=$(bash "{{QCLAW_HOME}}/skills/tencent-meeting-mcp/get-token.sh") mcporter call tencent-sheetengine set_range_value_by_csv --args "{
  \"file_id\": \"{{EXCEL_FILE_ID}}\",
  \"sheet_id\": \"{{EXCEL_SHEET_ID}}\",
  \"start_row\": {next_row},
  \"start_col\": 0,
  \"csv_data\": \"{日期},{主题},{开始时间},{结束时间},{参会人},📄 打开纪要,\"
}" --output json'
```

> ⚠️ **必须传 `start_row`**，不传会从第 0 行覆盖（表头被覆盖）。

**A5.3 设置超链接：**

```
bash -c 'TENCENT_MEETING_TOKEN=$(bash "{{QCLAW_HOME}}/skills/tencent-meeting-mcp/get-token.sh") mcporter call tencent-sheetengine set_link --args "{
  \"file_id\": \"{{EXCEL_FILE_ID}}\",
  \"sheet_id\": \"{{EXCEL_SHEET_ID}}\",
  \"row\": {next_row},
  \"col\": 5,
  \"display_text\": \"📄 打开纪要\",
  \"url\": \"{纪要文档URL}\"
}" --output json'
```

> ⚠️ 参数名是 `url`，不是 `link_id`。

---

### A6. 发送邮件确认

**在发送邮件前，先问用户确认：**

> 「会议纪要已整理完成，要发送邮件给 {参会人}（{邮箱}）吗？」

- 用户说「发」「确认」「好」→ 执行发送
- 用户说「不发」「不用了」→ 跳过发送，继续后续步骤

**发送邮件（必须通过 `email_gateway.sh` 入口）：**

```
bash '{{HOME}}/Library/Application Support/QClaw/openclaw/config/skills/imap-smtp-email/scripts/unix/email_gateway.sh' send \
  --account-email '{{YOUR_EMAIL}}' \
  --to '{收件人邮箱}' \
  --subject '📋 会议纪要 · {会议主题} · {日期}' \
  --html \
  --body '<!DOCTYPE html>
<html><head><meta charset="utf-8">
<style>
  body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; max-width: 640px; margin: 40px auto; color: #1a1a2e; }
  .header { background: linear-gradient(135deg, #667eea, #764ba2); color: white; padding: 28px 32px; border-radius: 12px 12px 0 0; }
  .body { background: #f8f9fc; padding: 28px 32px; border-radius: 0 0 12px 12px; border: 1px solid #e8eaf6; border-top: none; }
  .task { background: white; border-radius: 8px; padding: 13px 16px; margin-bottom: 8px; border: 1px solid #e8eaf6; }
  .task-owner { display: inline-block; background: #f0edfb; color: #667eea; font-size: 11px; font-weight: 600; padding: 2px 8px; border-radius: 20px; margin-top: 6px; }
  .doc-link { display: inline-block; background: #667eea; color: white; text-decoration: none; padding: 11px 24px; border-radius: 8px; font-size: 14px; font-weight: 600; margin-top: 8px; }
</style>
</head>
<body>
<div class="header">
  <h1>📋 {会议主题}</h1>
  <div style="opacity:0.88;font-size:13px;">{日期} {开始时间} - {结束时间} &nbsp;|&nbsp; 会议号：{会议号}</div>
</div>
<div class="body">
  <p style="margin-top:0;font-size:13px;color:#444;">{智能纪要摘要}</p>
  <div class="task">
    <div style="font-weight:600;font-size:13px;color:#1a1a2e;margin-bottom:6px;">{待办1}</div>
    <div><span class="task-owner">{负责人1}</span></div>
  </div>
  <div class="task">
    <div style="font-weight:600;font-size:13px;color:#1a1a2e;margin-bottom:6px;">{待办2}</div>
    <div><span class="task-owner">{负责人2}</span></div>
  </div>
  <a class="doc-link" href="{纪要文档URL}">🔗 在腾讯文档中查看完整纪要</a>
</div>
</body></html>'
```

---

### A7. 执行报告汇总

```
━━━━━━━━━━━━━━━━━━━━━━━━
📋 会议整理完成报告
━━━━━━━━━━━━━━━━━━━━━━━━
✅ 最近会议：{会议主题}
   🕐 {开始时间} - {结束时间}
   👥 参会人：{参会人}

✅ 会议纪要文档：{文档标题}
   🔗 {文档链接}

✅ Excel 索引表：已追加第 {行号} 行

✅ 邮件发送：已发送给 {参会人}
   📧 {邮箱}

━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### A8. 问第二个问题：要预定下次会议吗？

**输出执行报告后，问第二个问题：**

> 「要预定下次会议吗？如果要，告诉我时间。」

- 用户说时间 → 触发分支 B（会前预约），预约完成后更新 Excel"下次会议"列
- 用户说「不用了」→ 结束
- 用户不提 → 不追问，结束

---

## 🔷 分支 B：会前预约

### 触发条件
用户说：「帮我预定一个会议」「预约下周三的会议」「安排一个产品对齐会」

---

### B1. 信息收集（只问主题和时间）

```
AI：什么主题的会议？
用户：{主题}
AI：几点开始？时长多久？
用户：{时间}
```

> 参会人可选，不问；邮箱从联系人库匹配，匹配不上再问。

---

### B2. 预约腾讯会议 + 开启转录

```
bash -c 'TENCENT_MEETING_TOKEN=$(bash "{{QCLAW_HOME}}/skills/tencent-meeting-mcp/get-token.sh") mcporter call tencent-meeting-mcp schedule_meeting --args "{
  \"subject\": \"{会议主题}\",
  \"start_time\": \"{开始时间，ISO格式，如 2026-04-18T09:00:00+08:00}\",
  \"end_time\": \"{结束时间，ISO格式，如 2026-04-18T10:00:00+08:00}\",
  \"attendees\": [
    {\"user_id\": \"{姓名或邮箱}\"}
  ],
  \"settings\": {
    \"mute_type\": 2,
    \"allow_screen_sharing\": true,
    \"auto_record\": true
  }
}" --output json'
```

> ⚠️ 工具名是 `schedule_meeting`（不是 `create_meeting`），参数是 `subject`。
> `attendees` 接受 user_id（显示名）或邮箱；传显示名可能无法正确邀请。

---

### B3. 创建空文档占位

> ⚠️ **必须使用 `create_smartcanvas_by_mdx`**，不能用 `manage.create_file`（后者创建普通文档，不支持 MDX）。

占位文档内容应包含：
- 会议基本信息（日期、时间、参会人）
- 议程（从用户提供的主题推断，或写「待确认」）
- 上次纪要中的待对齐事项（如果用户刚开完会，可用 A8 上下文中的信息）

```
bash -c 'TENCENT_MEETING_TOKEN=$(bash "{{QCLAW_HOME}}/skills/tencent-meeting-mcp/get-token.sh") mcporter call tencent-docs create_smartcanvas_by_mdx --args "{
  \"title\": \"{会议主题} · {日期}\",
  \"mdx\": \"---\ntitle: {会议主题} · {日期}\nicon: 📋\n---\n\n<Page>\n\n<Heading level=\\\"1\\\">\n    {会议主题}\n</Heading>\n\n<Callout icon=\\\"📅\\\" blockColor=\\\"light_blue\\\" borderColor=\\\"sky_blue\\\">\n    {日期} {开始时间} - {结束时间} &nbsp;|&nbsp; 会议号：{会议号} &nbsp;|&nbsp; 参会人：{参会人}\n</Callout>\n\n<Heading level=\\\"2\\\">\n    本次会议目标\n</Heading>\n\n<Paragraph>\n    {根据主题描述本次会议目的}\n</Paragraph>\n\n<Heading level=\\\"2\\\">\n    议程\n</Heading>\n\n{用 <NumberedList> 列出议程}\n\n<Heading level=\\\"2\\\">\n    待对齐事项（来自上次纪要）\n</Heading>\n\n{从分支 A 上下文提取的待办 + 负责人}\n\n<Heading level=\\\"2\\\">\n    结论 & 待办\n</Heading>\n\n<Paragraph>\n    ⏳ 会后自动填充\n</Paragraph>\n\n</Page>\"
}" --output json'
```

> 📌 提取返回的 `file_id` 和 `url`。

---

### B4. 告知会议信息

```
✅ 会议已预约

📋 会议：{主题}
🕐 时间：{开始时间}
🔢 会议号：{会议号}
🔗 加入链接：{join_url}
📝 纪要文档：{文档链接}（会后自动填充）

已开启文字转录，会后说「开完会了」触发整理流程。
```

---

### B5. 会前提醒（提前5分钟 + 上次纪要）

> ⚠️ 如果用户在分支 A 中刚完成会议（即 A8 触发分支 B），上次纪要信息已在上下文中，**无需再次查询 API**，直接使用。

**B5.1（如无上下文）从 Excel 获取上次纪要：**

```
bash -c 'TENCENT_MEETING_TOKEN=$(bash "{{QCLAW_HOME}}/skills/tencent-meeting-mcp/get-token.sh") mcporter call tencent-sheetengine get_cell_data --args "{\"file_id\":\"{{EXCEL_FILE_ID}}\",\"sheet_id\":\"{{EXCEL_SHEET_ID}}\",\"start_row\":1,\"end_row\":1,\"start_col\":0,\"end_col\":6,\"return_csv\":true}" --output json'
```

> 📌 提取：上次主题（col 1）、上次纪要链接（col 5）。

**B5.2（如无上下文）获取上次纪要内容：**

```
bash -c 'TENCENT_MEETING_TOKEN=$(bash "{{QCLAW_HOME}}/skills/tencent-meeting-mcp/get-token.sh") mcporter call tencent-docs get_content --args "{\"file_id\":\"{上次纪要file_id}\"}" --output json'
```

> 📌 从纪要中提取「待办事项」和「下次讨论」相关内容。

**B5.3 设置会前提醒（提前5分钟）：**

> ⚠️ `payload.kind` 必须是 `systemEvent`（main session 不支持 agentTurn）。

```
tool: cron
action: add
job:
  name: "{会议主题}-会前提醒"
  schedule:
    kind: at
    at: "{会议开始时间 - 5分钟，ISO格式}"
  payload:
    kind: systemEvent
    text: |
      ⏰ 会议即将在5分钟后开始！

      ━━━━━━━━━━━━━━━━━━━━━━━━
      📋 本次会议
      ━━━━━━━━━━━━━━━━━━━━━━━━
      • 主题：{本次主题}
      • 时间：{开始时间}
      • 会议号：{会议号}
      • 加入链接：{join_url}

      ━━━━━━━━━━━━━━━━━━━━━━━━
      📝 上次会议纪要
      ━━━━━━━━━━━━━━━━━━━━━━━━
      • 主题：{上次主题}
      • 纪要链接：{上次纪要链接}

      📌 待对齐事项：
      {从上次纪要提取的待办/下次讨论内容}

      ━━━━━━━━━━━━━━━━━━━━━━━━
  sessionTarget: main
  enabled: true
  deleteAfterRun: true
```

---

### B6. 执行报告汇总

```
━━━━━━━━━━━━━━━━━━━━━━━━
📋 会议预约完成报告
━━━━━━━━━━━━━━━━━━━━━━━━
✅ 腾讯会议：{主题}
   🔢 会议号：{会议号}
   🔗 加入链接：{join_url}
   📝 已开启文字转录

✅ 纪要文档（占位）：{文档链接}

⏰ 会前提醒：已设置（提前5分钟，含上次纪要 + 待对齐事项）

━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 📎 固定资源

| 资源 | 路径/ID | 备注 |
|------|---------|------|
| 会议索引表 | `file_id: {{EXCEL_FILE_ID}}`, `sheet_id: {{EXCEL_SHEET_ID}}` | ⚠️ sheetengine 所有调用必须传 `sheet_id`（工具签名不暴露，服务端强制要求） |
| 联系人库 | `memory-tdai/联系人.md` | 参会人邮箱自动匹配 |
| Token 脚本 | `{{QCLAW_HOME}}/skills/tencent-meeting-mcp/get-token.sh` | 见 Token 传递规范 |
| 邮件入口 | `imap-smtp-email/scripts/unix/email_gateway.sh` | 必须通过此入口，禁止直接调 smtp.js |
| 用户邮箱 | `{{YOUR_EMAIL}}` | SMTP 已配置 |

---

## ⚠️ 关键原则

1. **分支 A 半自动**：用户说「开完会了」→ 自动拉取会议 → 生成纪要 → 先展示给你确认 → 确认后写入文档/Excel/邮件
2. **三个确认点**：(1) 纪要预览确认 (2) 发邮件确认 (3) 要预定下次会议吗
3. **参会人邮箱**：自动匹配联系人库，匹配不上才问
4. **会议内容**：智能纪要 + 转录原文都要
5. **Excel 追加行**：必须先读 → 算行数 → 指定 `start_row` 写入，不传 `start_row` 会覆盖表头
6. **Token 传递**：必须内联，不传 `start_row` 会覆盖表头；必须用 `bash -c` 方式
7. **文档创建**：用 `create_smartcanvas_by_mdx`，不用 `manage.create_file`
8. **MDX 格式**：严格遵循三段式多行、双引号属性、无表达式语法
9. **Cron payload**：main session 必须用 `systemEvent`
10. **先审核再沉淀**：确认通过的纪要才写入文档，防止错误内容沉淀
