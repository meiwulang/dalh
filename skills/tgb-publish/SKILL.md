---
name: tgb-publish
description: Publish stock market review content to tgb.cn (淘股吧) draft system. Handles BBCode formatting, API submission, and draft verification. Use when the user asks to publish or submit a review to tgb.cn.
---

# tgb.cn Publishing Workflow

Publish formatted stock review content to tgb.cn draft system via the `saveUserDraft` API.

## Supported BBCode Tags

The kissEditor on tgb.cn supports these BBCode-style tags (NOT HTML):

| Tag | Purpose | Example |
|-----|---------|---------|
| `[b]text[/b]` | Bold | `[b]标题[/b]` |
| `[u]text[/u]` | Underline | `[u]操作纪律[/u]` |
| `[color=#RRGGBB]text[/color]` | Colored text | `[color=#F00F00]红色[/color]` |
| `[color=#0000FF]text[/color]` | Blue text | `[color=#0000FF]蓝色[/color]` |
| `[size=N]text[/size]` | Font size | `[size=3]大字号[/size]` |

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

## Submission Flow

1. **Read cookies** from `~/.atomcode/skills/tgb-publish/cookies.txt`
2. **查重**：调用草稿列表API，检查是否已有同标题草稿
   - 如果有：获取其 `draftSeq`，更新而非新建
   - 如果没有：新建草稿（不传 `draftSeq`）
3. **Build BBCode content** with proper formatting tags
4. **URL-encode** both title and content (single encode only, using `urllib.parse.quote`)
5. **Submit via curl** using `--data-binary @file` to avoid shell escaping issues
6. **只存草稿，不自动发布** — 提示用户去后台手动发布
7. **Verify** by fetching the page and checking `var draftContent =` for expected tags

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
        '-b', cookies
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

## Verification

After submission, verify formatting survived by checking the stored content:

```bash
curl -s 'https://www.tgb.cn/topic/newTopic?davFlag=dv&draftSeq={draftSeq}' \
  -H 'user-agent: ...' -b 'cookies...' 2>/dev/null | \
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
