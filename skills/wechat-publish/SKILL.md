---
name: wechat-publish
description: 将文章内容发布到微信公众号草稿箱。支持自定义标题、作者、正文内容，使用永久封面图。通过微信公众平台 API 直接提交图文草稿（draft）。适用于股票复盘、新闻点评、原创文章等需要发布到公众号的场景。
---

# 微信公众号发布 Skill

将文章发布到微信公众号（服务号/订阅号）的草稿箱。使用微信公众平台「草稿箱」API（`cgi-bin/draft/add`），一次发布一篇图文素材。

## 前置条件

- 公众号已通过微信认证
- AppID 和 AppSecret 在 `config.sh` 中配置（见下文）
- 封面图已上传为永久素材，media_id 在 `config.sh` 中配置

## 配置

首次使用前，运行以下命令复制并编辑配置：

```bash
cp /Users/wangbin/.atomcode/skills/wechat-publish/config.example.sh \
   /Users/wangbin/.atomcode/skills/wechat-publish/config.sh
# 编辑 config.sh 填入你的 AppID、AppSecret、media_id
```

配置项说明：

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `APPID` | 微信公众号 AppID | `wx4c297a89e552125d` |
| `APPSECRET` | 微信公众号 AppSecret | （需要填写） |
| `THUMB_MEDIA_ID` | 永久封面图 media_id | `L5py12RzS6GbcdIx33ShQcPbUTFuQJTpoCqSbYAgunMx_sEobtAjQT1QROetXthF` |
| `AUTHOR` | 文章作者署名 | `梅五郎` |

## 执行流程

### 1. 准备文章内容

你需要先生成文章内容（标题 + HTML 正文），然后调用发布脚本。

### 2. 调用发布脚本

```bash
python3 /Users/wangbin/.atomcode/skills/wechat-publish/scripts/wechat_publisher.py \
  --title "文章标题" \
  --content /tmp/article_body.html
```

参数说明：

| 参数 | 说明 | 必填 |
|------|------|------|
| `--title` | 文章标题（不超过 30 字节，中文最多 10 字以内） | ✅ |
| `--content` | 文章正文 HTML 文件路径 | ✅ |
| `--author` | 作者名（可选，默认 config.sh 中的 AUTHOR） | ❌ |

### 3. 验证结果

脚本输出 `✅ 草稿发布成功! media_id: xxx` 表示成功。
发布后可在公众号后台「草稿箱」中查看和最终编辑后再群发。

## 文章内容格式

正文支持标准 HTML，推荐使用以下内联样式：

- 字体: `font-size: 16px`, `line-height: 1.8`
- 标题: `<h1>~<h3>` 带左侧红色边框（`border-left: 4px solid #c0392b`）
- 表格: 用于指数数据、涨停数据等结构化展示
- 高亮: `.highlight`（红色）、`.green`（绿色）、`.red`（红色）
- 引用: `.quote` 灰色底色 + 左侧红色边框
- 摘要框: `.summary-box` 紫色渐变背景用于明日策略总结
- 免责声明: `.disclaimer` 黄色背景

现成的 HTML 模板在脚本的 `build_html_article()` 函数中。

## 标题长度限制

微信公众号标题字节限制为 **30 字节**：
- 1 个中文字符 = 3 字节（UTF-8）
- 1 个 ASCII 字符 = 1 字节
- 推荐标题不超过 **10 个中文字符**，或 **30 个 ASCII 字符**

## 文章风格参考

### 股票复盘文章结构

1. 指数信号（上证、深证、创业板、科创50收盘表现）
2. 量能信号（成交额、涨跌家数、涨停数据）
3. 主线分析（板块涨跌、龙头个股）
4. 节点判断（情绪周期阶段、短线节点）
5. 明日策略（主线方向、仓位建议、核心观察、风险提示）

### 免责声明

每篇文章末尾应包含投资风险提示。

## 查重逻辑（防止重复发布）

发布前检查 `~/.atomcode/skills/wechat-publish/publish_records.json`，同标题草稿存在则提示更新而非新建。

```python
import json, os
from datetime import datetime

def check_duplicate(title):
    record_file = os.path.expanduser('~/.atomcode/skills/wechat-publish/publish_records.json')
    if os.path.exists(record_file):
        with open(record_file) as f:
            records = json.load(f)
        if title in records:
            return records[title]
    return None

def save_record(title, media_id):
    record_file = os.path.expanduser('~/.atomcode/skills/wechat-publish/publish_records.json')
    records = {}
    if os.path.exists(record_file):
        with open(record_file) as f:
            records = json.load(f)
    records[title] = {'media_id': media_id, 'date': datetime.now().isoformat()}
    with open(record_file, 'w') as f:
        json.dump(records, f, ensure_ascii=False, indent=2)
```

## 重要原则

- **永远只存草稿，不自动群发** — 提示用户去公众号后台手动群发
- **发布前必须查重**，同标题草稿存在则提示"已有草稿，是否覆盖？"
- **标题必须带完整年份**（如 `凤凰涅槃复盘20260623`），不省略年份

## 注意事项

- AppSecret 是敏感信息，`config.sh` 已加入 `.gitignore`，不要提交到 Git
- 每次调用获取新的 `access_token`（有效期 2 小时），无需缓存
- 封面图 media_id 更换后需同步更新配置
- 发布的是草稿，不是已群发文章——发布后需在公众号后台手动群发
- 如需调整文章样式，直接修改 `build_html_article()` 中的 CSS 即可
