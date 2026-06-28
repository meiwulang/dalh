---
name: tgb-publish
description: Publish stock market review content to tgb.cn (淘股吧) draft system via `sau tgb` CLI. Handles browser-automation login, cookie management, BBCode formatting, API submission, and draft verification. Use when the user asks to publish or submit a review to tgb.cn.
---

# tgb.cn Publishing Workflow (sau 模式)

通过 `sau tgb` CLI 完成登录、校验和草稿发布，与其他视频平台（抖音/B站/小红书/视频号/快手）保持一致的 sau 工作流。

## 功能概览

| 功能 | 命令入口 | 说明 |
| --- | --- | --- |
| 登录 | `sau tgb login --account <name>` | 浏览器自动化登录，用户扫码/输密码后自动保存 Cookie |
| 校验 | `sau tgb check --account <name>` | 检查 Cookie 是否有效 |
| 发布草稿 | `sau tgb upload-draft ...` | 提交草稿到 tgb.cn（不自动发布） |

## 默认工作流

1. `sau tgb check --account <name>` 校验 Cookie
2. 如果无效 → 提示用户在终端执行 `sau tgb login --account <name>`
3. 构建 BBCode 内容
4. `sau tgb upload-draft --account <name> --title "<title>" --content-file <file>`
5. 提示用户去后台手动发布

## 登录

```bash
cd ~/social-auto-upload && source .venv/bin/activate
sau tgb login --account tgb1
```

会弹出浏览器 → 用户在 tgb.cn 登录（扫码/输密码）→ Cookie 自动保存到 `~/social-auto-upload/cookies/tgb_tgb1.json`

## 校验

```bash
sau tgb check --account tgb1
```

返回 `valid` 或 `invalid`。

## 发布草稿

```bash
sau tgb upload-draft \
  --account tgb1 \
  --title "凤凰涅槃·每日复盘 | 2026.06.23" \
  --content-file /tmp/tgb_content.txt
```

或直接传内容：

```bash
sau tgb upload-draft \
  --account tgb1 \
  --title "凤凰涅槃·每日复盘 | 2026.06.23" \
  --content "[b]标题[/b][color=#F00F00]红色[/color]"
```

## Supported BBCode Tags

The kissEditor on tgb.cn supports these BBCode-style tags (NOT HTML):

| Tag | Purpose | Example |
|-----|---------|---------|
| `[b]text[/b]` | Bold | `[b]标题[/b]` |
| `[u]text[/u]` | Underline | `[u]操作纪律[/u]` |
| `[color=#RRGGBB]text[/color]` | Colored text | `[color=#F00F00]红色[/color]` |
| `[color=#0000FF]text[/color]` | Blue text | `[color=#0000FF]蓝色[/color]` |
| `[size=N]text[/size]` | Font size | `[size=3]大字号[/size]` |

## ⚠️ BBCode 格式铁律（2026-06-28 修正）

淘股吧 kissEditor API 对 BBCode 支持有限，以下规则必须遵守，否则返回 HTTP 500。

### ❌ 禁止使用的格式

| 禁止项 | 原因 | 替代方案 |
|--------|------|---------|
| Emoji 表情（🐉🟢🟡🔴等4字节Unicode） | API的URL编码无法处理4字节字符 | 用文字替代 `(强)` `(中)` `(弱)` |
| `[size=X]...[/size]` 字号标签 | 复杂BBCode嵌套导致API解析失败 | 只用 `[b]` 和 `[color]` |
| `[X/Y]` 格式（如 `[6/6]` `[13/7]`） | 被API错误解析为BBCode关闭标签 | 改写为 `6天6板` 或 `(6/6)` |
| 多层嵌套（超2层） | API标签栈解析有bug | 保持最多2层嵌套，每层严格配对 |
| 分隔线符号（过长） | 超长连续符号导致解析异常 | 限制在10字符以内 |

### ✅ 推荐的格式

```text
[b]标题文字[/b]                              ← 加粗
[color=#F00F00]红色强调文字[/color]          ← 红色
[color=#0000FF]蓝色强调文字[/color]          ← 蓝色
[color=#F00F00][b]红色加粗文字[/b][/color]   ← 2层嵌套（最多）
6天6板 或 (6/6)                             ← 连板统计（避免方括号）
```

### 内容长度限制

- 正常范围：4000-6000字符（含 `[b]` `[color]` 的纯BBCode）
- 极限制：约9700字符（简单格式无特殊字符）
- 超9500且含特殊字符 → HTTP 500

建议控制在 **5000-7000字符**，15个章节完整复盘。

### 提交失败排查流程

```
HTTP 500
  → 检查是否含 emoji（4字节Unicode）
    → 有则去掉
  → 检查是否含 [X/Y] 格式
    → 有则改为 (X/Y) 或 X天Y板
  → 检查是否含 [size] 标签
    → 有则去掉
  → 检查BBCode嵌套是否超2层
    → 简化嵌套
  → 检查字符是否超9500
    → 精简
```

## API Endpoint

```
POST https://www.tgb.cn/topic/saveUserDraft
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
```

## Required Cookies

The following cookies must be included in every request (from browser session):

- `gdp_user_id`
- `_c_WBKFRo`
- `_nb_ioWEgULi`
- `tgbuser` + `tgbpwd`
- `loginStatus`
- `JSESSIONID`
- `acw_tc`
- Various session tracking cookies

## Required Headers

```
accept: application/json, text/javascript, */*; q=0.01
content-type: application/x-www-form-urlencoded; charset=UTF-8
origin: https://www.tgb.cn
referer: https://www.tgb.cn/topic/newTopic?davFlag=dv&draftSeq={draftSeq}
user-agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36
x-requested-with: XMLHttpRequest
```

## POST Data Fields

| Field | Value | Notes |
|-------|-------|-------|
| `title` | URL-encoded title string | e.g., `616%E5%A4%8D%E7%9B%98` = "616复盘" |
| `content` | URL-encoded BBCode content | Use BBCode, NOT HTML |
| `type` | `T` | Draft type |
| `draftSeq` | Draft ID (number) | Omit to create new, include to update |

## 查重逻辑（防止重复发布）

```python
import urllib.parse, json, subprocess

def get_existing_draft(title, cookies):
    """查找同标题草稿，返回draftSeq或None"""
    # 方法1: 通过草稿列表API查询
    result = subprocess.run([
        'curl', '-s',
        'https://www.tgb.cn/topic/getUserDraftList?pageNo=1&pageSize=50',
        '-H', 'accept: application/json',
        '-H', 'x-requested-with: XMLHttpRequest',
        '-H', f'Cookie: {cookies}'
    ], capture_output=True, text=True)
    try:
        data = json.loads(result.stdout)
        for draft in data.get('data', {}).get('list', []):
            if draft.get('title') == title:
                return draft.get('draftSeq') or draft.get('id')
    except:
        pass

    # 方法2: 通过本地记录文件查询
    record_file = os.path.expanduser('~/.atomcode/skills/tgb-publish/publish_records.json')
    if os.path.exists(record_file):
        with open(record_file) as f:
            records = json.load(f)
        if title in records:
            return records[title].get('draftSeq')
    return None

def save_record(title, draft_seq):
    """保存发布记录，用于后续查重和更新"""
    record_file = os.path.expanduser('~/.atomcode/skills/tgb-publish/publish_records.json')
    records = {}
    if os.path.exists(record_file):
        with open(record_file) as f:
            records = json.load(f)
    records[title] = {'draftSeq': draft_seq, 'date': datetime.now().isoformat()}
    with open(record_file, 'w') as f:
        json.dump(records, f, ensure_ascii=False, indent=2)
```

## 发布记录文件

`~/.atomcode/skills/tgb-publish/publish_records.json` 记录每个标题对应的draftSeq，格式：

```json
{
  "凤凰涅槃·每日复盘 | 2026.06.23": {
    "draftSeq": 1055028,
    "date": "2026-06-23T14:03:00"
  }
}
```

## 重要原则

- **永远只存草稿，不自动发布为正式帖子**
- **发布前必须查重**，同标题草稿存在则更新（传draftSeq），不存在则新建
- **提示用户**：草稿已保存，请到 tgb.cn 后台手动发布

## Important Notes

- The API strips ALL HTML tags (`<b>`, `<u>`, `<span>`, `<font>` etc.). Use BBCode `[b]`, `[u]`, `[color]` instead.
- The API wraps content in `<p>` tags automatically. Do NOT include `<p>` tags in the submitted content.
- Verified working colors: `#F00F00` (red), `#0000FF` (blue). Other colors may be filtered.
- Use `--data-binary @file` NOT `--data-raw "..."` to avoid shell interpretation of special characters.
- Write the POST body to a temp file first using Python, then curl with `--data-binary @file`.
- If draft submission returns `502` or draft-list lookup returns an HTML `404`, first refresh cookie by running `sau tgb login --account <name>` and retry `upload-draft`; do not treat the old-cookie failure as a content/BBCode problem.
- `getUserDraftList` may be unavailable or route-filtered even when the main site is reachable. In that case, use the local `publish_records.json` for duplicate detection and verify via the returned `draftSeq` or by opening `newTopic?davFlag=dv&draftSeq=...`.

## Verification

After submission, verify formatting survived by checking the stored content:

```bash
curl -s 'https://www.tgb.cn/topic/newTopic?davFlag=dv&draftSeq={draftSeq}' \
  -H 'user-agent: ...' -H 'Cookie: ...' 2>/dev/null | \
  grep 'var draftContent =' | sed 's/.*var draftContent = "//' | sed 's/".*//' | \
  grep -o '<[a-z/][^>]*>' | sort | uniq -c | sort -rn
```

Expected tags: `<p>`, `</p>` (editor auto-wraps). BBCode formatting may be stored as HTML equivalents.

## Python Helper Template

```python
import urllib.parse

content = """[b]Title[/b]
[color=#F00F00]Red text[/color]
[u]Underline text[/u]"""

encoded = urllib.parse.quote(content, safe='')
data = f"title={urllib.parse.quote(title, safe='')}&content={encoded}&type=T&draftSeq={seq}"

with open('/tmp/tgb_post.txt', 'w') as f:
    f.write(data)
```

Then: `curl ... --data-binary @/tmp/tgb_post.txt`
