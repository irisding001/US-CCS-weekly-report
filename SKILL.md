---
name: US-CCS-weekly-report
description: US CCS 团队周报生成器。拉取业绩数据，收集文字内容，生成 HTML 卡片并推送 GitHub Pages + 飞书通知。当用户提到"周报"、"weekly report"、"CCS 周报"、"填周报"、"写周报"时使用。
---

# US CCS Weekly Report

## 工作流程

### Step 1：获取认证信息

用户需提供：Cookie（含 `EGG_SESS`、`csrfToken`）、CSRF Token、uIdToken（可选）、日期范围（默认上周一~上周日）

### Step 2：拉取数据

```bash
cd C:/Users/irisding/.claude/skills/us-ccs-weekly-performance

# 本周跟进量
py scripts/fetch_data.py --start {start_yyyymmdd} --end {end_yyyymmdd} --cookies "{cookie}" --csrf "{csrf}"

# 上周跟进量（日期前推7天，用于环比）
py scripts/fetch_data.py --start {prev_start} --end {prev_end} --cookies "{cookie}" --csrf "{csrf}"

# Q2 累计 PC（4/1 ~ 本周日）
py scripts/fetch_data.py --start 20260401 --end {end_yyyymmdd} --cookies "{cookie}" --csrf "{csrf}"

# 转化率（需 uIdToken）
py scripts/fetch_bi_data.py --card new_leads_personal --start {start_dash} --end {end_dash} --uid-token {uid_token}
```

### Step 3：收集文字板块

逐项询问，可跳过：二（团队管理 / 招聘进展）/ 三（AI 使用情况 / AI 应用计划）/ 四（下周计划）

### Step 4：生成 HTML（分两段写入）

**避免 API Error: terminated，必须分两次写入。**

文件名：`weekly_report_{YYYY-MM-DD}.html`（日期 = 下周周一）
路径：`C:/Users/irisding/.claude/skills/us-ccs-weekly-performance/`

**第一段（Write 工具）**：`<!DOCTYPE html>` → `</script>`，结尾不加 `</body></html>`
- WEEKS/DATA 从上一份 `weekly_report_*.html` 读取，滑动追加当周

**第二段（Bash >> 追加）**：
```bash
cat >> "C:/Users/irisding/.claude/skills/us-ccs-weekly-performance/weekly_report_{date}.html" << 'EOF'
...二/三/四 HTML + </body></html>...
EOF
```

### Step 5：推送 GitHub Pages

```bash
cp "C:/Users/irisding/.claude/skills/us-ccs-weekly-performance/weekly_report_{date}.html" \
   "C:/Users/irisding/US-CCS-weekly-report/"
cd "C:/Users/irisding/US-CCS-weekly-report"
git add "weekly_report_{date}.html"
git commit -m "Add US CCS weekly report {start} ~ {end}"
git push origin main
```

地址：`https://irisding001.github.io/US-CCS-weekly-report/weekly_report_{date}.html`

### Step 6：飞书通知

卡片含两个按钮：**点击查看**（最新报告）+ **查看历史周报**（history.html）。

```bash
lark-cli --profile us-ccs im +messages-send \
  --user-id ou_423989c914515582660dfef99848b0e7 \
  --as bot --msg-type interactive \
  --content '{"config":{"wide_screen_mode":true},"header":{"title":{"tag":"plain_text","content":"US CCS Weekly Report | {MM-DD} ~ {MM-DD}"},"template":"blue"},"elements":[{"tag":"action","actions":[{"tag":"button","text":{"tag":"plain_text","content":"点击查看"},"type":"primary","url":"https://irisding001.github.io/US-CCS-weekly-report/"},{"tag":"button","text":{"tag":"plain_text","content":"查看历史周报"},"type":"default","url":"https://irisding001.github.io/US-CCS-weekly-report/history.html"}]}]}'
```

---

## HTML 设计规范

### 卡片结构

```
Header：US CCS Weekly Report- Irisding | 日期范围 | Week N
Compare Strip（4列）：总跟进量▲▼% / 总有效跟进量▲▼% / Weekly PC▲▼% / Q2季度总PC
数据表格列顺序（客经名后）：
  group-row: [Q2累计 span=1] [跟进量 span=3] [新Leads转化率 span=3]
  sub-row:   Q2 PC | 总跟进量 | 有效跟进量 | Weekly PC | 有效跟进率 | 跟进转化率 | 有效跟进转化率
趋势图：近6周团队转化率（有效跟进率 / 有效跟进转化率 / 跟进转化率）
二、团队运营 / 三、AI 应用 / 四、下周计划 / 底部 note
```

### 关键样式

- body：`display:flex; justify-content:center; padding:32px 16px`
- Weekly PC 列头：`pc-col`（橙色 `#ff8c00`）；Q2 PC 列头：`q2-col`（橙色背景白字），单元格文字橙色
- min/max 高亮：仅字体颜色（绿 `#16a34a` / 红 `#b91c1c`），`colCount=7`
- 趋势图：Y 轴 min:0 max:35；蓝 `#1456F0` / 橙 `#f59e0b` / 绿 `#10b981`
- WEEKS 格式：`MM-DD~MM-DD`（6周，周一~周日）
- 底部 note：`Q2季度总PC：4/1~{end_date}累计（外呼+SMS去重）`

> CSS 样式（q2、q2-col、g-q2 等）直接从上一份 `weekly_report_*.html` 复用，无需重写。

---

## 注意事项

- open_id 固定：`ou_423989c914515582660dfef99848b0e7`
- GitHub Pages 有 1~2 分钟延迟；git remote 已配置 SSH over 443，公司网络可直连
