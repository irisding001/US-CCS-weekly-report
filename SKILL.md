---
name: US-CCS-weekly-report
description: US CCS 团队周报生成器。自动拉取业绩数据，引导用户逐板块填写周报内容，生成 HTML 卡片文件并发送飞书消息。当用户提到"周报"、"weekly report"、"CCS 周报"、"填周报"、"写周报"时，必须使用此 skill。
---

# US CCS Weekly Report

生成 HTML 周报卡片 + 发送飞书私信（给自己）。

## HTML 模板规范

生成文件命名：`weekly_report_{YYYY-MM-DD}.html`（日期取**下周周一**，即报告提交日。例：报告周期 05-25~05-31，文件名用 2026-06-01）
保存路径：`C:/Users/irisding/.claude/skills/us-ccs-weekly-performance/`

### 卡片结构

```
Header：US CCS Weekly Report- Irisding | 日期范围 | Week N
Compare Strip：与上周对比（总跟进量 / 总有效跟进量 / 总PC ▲▼%）+ Q2季度总PC（独立卡片，4/1~本周日累计）
数据表格：跟进量（总跟进量、有效跟进量、总PC）+ 新 Leads 转化率（有效跟进率、跟进转化率、有效跟进转化率）
趋势图：团队近6周转化率趋势（3条折线：有效跟进率 / 有效跟进转化率 / 跟进转化率）
二、团队运营（团队管理 + 招聘进展）
三、AI 应用（使用情况 + 应用计划）
四、下周计划
```

### 关键设计细节

- **参考文件**：`C:/Users/irisding/.claude/skills/my-ccs-weekly-performance/preview_card.html`（样式基准）
- body：`display: flex; justify-content: center; padding: 32px 16px;`
- 表格第一列（客经名）：`text-align: center`
- min/max 高亮：**仅字体颜色**，绿色 `#16a34a`，红色 `#b91c1c`，不加背景色填充
- 总PC列：`pc-col` 橙色列头（`#ff8c00`）
- 趋势图 Y 轴：`min: 0, max: 35`；3条线颜色：蓝 `#1456F0` / 橙 `#f59e0b` / 绿 `#10b981`
- 周期格式：`MM-DD~MM-DD`（周一到周日，6 周）

### Q2 季度总PC

- **定义**：自 4月1日起至本周日，团队总转化有资产数（外呼+SMS去重），即 `intervened_transferred_num` 字段
- **数据来源**：Source A API，日期范围改为 `start_date=每年04-01` / `end_date=本周周日`
- **展示位置**：Compare Strip 第4个卡片（总PC 右侧），不做环比，只显示累计数值
- **CSS 样式**：`.cmp.q2` — 橙色边框/背景，值用 `#ff8c00`
  ```css
  .cmp.q2 { border-color: #ffd899; background: #fffaf0; }
  .cmp.q2 .val { color: #ff8c00; }
  ```
- **Compare Strip 布局**：`grid-template-columns: repeat(4, 1fr)`（原来3列改为4列）
- **底部 note** 补充说明：`Q2季度总PC：4/1~{end_date}累计（外呼+SMS去重）`

### WEEKS 数组规则

每周必须是**周一到周日**，例：
```javascript
const WEEKS = ['04-13~04-19','04-20~04-26','04-27~05-03','05-04~05-10','05-11~05-17','05-18~05-24'];
```

---

## 工作流程

### Step 1：获取认证信息 & 日期范围

询问用户提供：
- Cookie 字符串（含 `EGG_SESS`、`csrfToken`）
- CSRF Token
- uIdToken（BI Dashboard JWT，**可选**；若无则跳过转化率，或用户手动填写）
- 确认日期范围（默认上一完整周，周一~周日）

### Step 2：调用 `us-ccs-weekly-performance` skill

执行完整数据拉取流程，获得：
- 跟进量数据（3人 + 团队合计，含与上周对比）
- **Q2季度总PC**：额外调用 Source A，日期范围 `20260401~{end_date}`，取团队 `intervened_transferred_num` 加总
- 新 Leads 个人转化率（有效跟进率 / 跟进转化率 / 有效跟进转化率）— 需 uIdToken

> **网络提示**：Python urllib 在部分 VPN 环境下 SSL 握手超时，改用 `curl` + `py` 解析 JSON 可绕过。
> 拉取顺序：先拉本周 → 上周 → Q2累计，每次单独请求避免连接超时。

### Step 3：收集文字板块（二/三/四）

每次只问一个子项，等用户回复后再继续：

1. **二-团队管理**：出勤、培训、问题跟进等
2. **二-招聘进展**：面试、offer、岗位状态等
3. **三-AI 工具使用情况**：工具、场景、效果
4. **三-AI 应用计划**：下阶段方向
5. **四-下周计划**：自由填写

用户可随时说"跳过"留空。

### Step 4：生成 HTML 文件

根据上述数据和模板规范，生成完整 HTML 文件：
- 补充当周和前5周的 DATA 数组（向用户确认历史数据，或从上一份报告沿用）
- 写入 `weekly_report_{start_date}.html`（保存路径：`C:/Users/irisding/.claude/skills/us-ccs-weekly-performance/`）

### Step 5：推送到 GitHub Pages

将 HTML 文件复制到本地 repo 并推送：

```bash
cp "C:/Users/irisding/.claude/skills/us-ccs-weekly-performance/weekly_report_{start_date}.html" \
   "C:/Users/irisding/US-CCS-weekly-report/"

cd "C:/Users/irisding/US-CCS-weekly-report"
git add "weekly_report_{start_date}.html"
git commit -m "Add US CCS weekly report {start_date} ~ {end_date}"
git push origin main
```

**注意**：git remote 已配置为 SSH over 443（`git@github.com:irisding001/US-CCS-weekly-report.git`），公司网络下可直接推送，无需 VPN。

推送成功后 GitHub Pages 地址：
`https://irisding001.github.io/US-CCS-weekly-report/weekly_report_{start_date}.html`

### Step 6：发送飞书卡片通知

通过 `lark-im` skill，以 **bot 身份**（`--as bot`）、**`--profile us-ccs`** 发送 interactive card 给 `irisding`（us-ccs app open_id: `ou_423989c914515582660dfef99848b0e7`）：

```json
{
  "config": {"wide_screen_mode": true},
  "header": {
    "title": {"tag": "plain_text", "content": "📊 US CCS Weekly Report 已更新 | {start_MM_DD} ~ {end_MM_DD}"},
    "template": "blue"
  },
  "elements": [
    {
      "tag": "action",
      "actions": [
        {
          "tag": "button",
          "text": {"tag": "plain_text", "content": "点击查看完整报告"},
          "type": "primary",
          "url": "https://irisding001.github.io/US-CCS-weekly-report/weekly_report_{start_date}.html"
        }
      ]
    }
  ]
}
```

命令示例：
```bash
lark-cli --profile us-ccs im +messages-send \
  --user-id ou_423989c914515582660dfef99848b0e7 \
  --as bot \
  --msg-type interactive \
  --content '{...card json...}'
```

---

## 注意事项

- 日期使用**下周周一**作为文件名（= 报告周期结束次日，即提交日）
- 历史趋势数据：优先从上一份 `weekly_report_*.html` 的 DATA 数组读取，追加当周数据
- 若上周数据缺失，向用户确认后手动填入
- GitHub Pages 更新有 1~2 分钟延迟，推送成功后稍等再刷新链接
- irisding open_id（**us-ccs app**）：`ou_423989c914515582660dfef99848b0e7`（已固定，无需每次查询）
- 卡片用 **button** 元素（非 lark_md 链接），URL 指向具体 HTML 文件名
- HTML 样式：section-label 蓝色大字（15px #1456F0），sub-title 蓝色粗体，正文列表黑色（#333）
- 根路径 `index.html` 已部署，自动跳转最新报告
