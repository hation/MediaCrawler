# MediaCrawler 项目规则 — 常用命令封装

> 目的：用户用自然语言提出爬取需求时，直接按本规则映射为具体命令并执行，用户无需记忆任何命令行参数。

## 一、平台名称映射

| 用户说法 | `--platform` 值 |
|---|---|
| 小红书 / xhs / rednote | `xhs` |
| 抖音 / douyin / dy | `dy` |
| 快手 / kuaishou / ks | `ks` |
| B站 / bilibili / 哔哩哔哩 / bili | `bili` |
| 微博 / weibo / wb | `wb` |
| 贴吧 / 百度贴吧 / tieba | `tieba` |
| 知乎 / zhihu | `zhihu` |

## 二、三个核心场景（默认模板）

通用默认值（用户未指定时使用）：
- 登录方式 `--lt qrcode`
- 不开启无头模式 `--headless false`（首次登录 / 遇滑块验证码时必须保持 false，弹浏览器人工处理）
- 一级评论 `--get_comment true`，二级评论 `--get_sub_comment false`（用户明确要"二级/子评论/楼中楼"才改 true）
- 单条内容一级评论上限 `--max_comments_count_singlenotes 50`（用户指定的项目默认值）
- 存储格式 `--save_data_option jsonl`（不传即默认，无需额外指定）

### 场景1：关键词内容搜索
用户说「爬/查/搜 + 平台 + 关键词」→ 执行：

```bash
uv run main.py --platform <平台> --type search --keywords "<关键词1,关键词2>" \
  --get_comment true --get_sub_comment false --max_comments_count_singlenotes 50
```

- 关键词多个用英文逗号分隔
- 用户未提供关键词时，必须停下询问，**禁止**回退到默认关键词「编程副业,编程兼职」

### 场景2：指定创作者的全部内容
用户说「爬/抓 + 平台 + 某个创作者/博主/UP主/作者 + 所有/全部内容」→ 执行：

```bash
uv run main.py --platform <平台> --type creator --creator_id "<创作者ID或主页URL>" \
  --get_comment true --get_sub_comment false --max_comments_count_singlenotes 50
```

- **小红书创作者 ID 三种给法（代码已支持自动解析）**：
  - 完整主页链接（含 xsec_token）：最稳，直接使用
  - 24 位 hex user_id（如 `5acd5ca94eacab6236814d4d`）：自动打开主页提取 token 后爬取
  - 纯数字 user_id（如 `181944656`）：自动站内搜索定位主页；搜索不到时，让用户提供完整主页链接
- 其他平台（B站/抖音/微博等）`creator_id` 支持 ID 或主页 URL
- 贴吧支持主页 URL 自动归一化

### 场景3：内容的一级评论
- 一级评论由场景1/2 模板默认带上（`--get_comment true`）
- 用户单说要评论，视作在场景1/2 基础上开启评论，按对应场景执行
- 单条评论上限 50 条；用户可临时指定数量

## 三、执行规范

1. 解析用户需求时依次确认：**平台？场景？目标对象**（关键词 或 创作者ID/链接）
2. 必要信息缺失（平台 / 关键词 / 创作者ID）时先询问，不凭空猜测
3. 命令必须在工作目录 `/Users/xingan/Documents/software/aiengine/mediacrawler` 下执行
4. 抓取数量上限默认使用配置值（`CRAWLER_MAX_NOTES_COUNT=200`），用户明确要求数量时才传 `--crawler_max_notes_count`
5. 单次大规模任务（如 200 条 + 评论）执行前提醒风控风险，可建议缩小范围或关闭评论
6. 运行结束后告知用户数据保存路径（`data/{平台}/{格式}/文件名_日期.{后缀}`）
   - **一次爬取会话只生成 1 个 contents + 1 个 comments 文件**（AsyncFileWriter 已进程内单例化，时间戳固定）
   - creator 模式文件名会带 user_id：`creator_contents_{user_id}_{日期}_xhs.csv` / `creator_comments_{user_id}_{日期}_xhs.csv`，不同创作者互不覆盖
   - search 模式文件名带关键词：`search_contents_{日期}_{关键词}_xhs.csv`
7. 用户要求多平台同时爬取时，为每个平台按场景1 模板逐条执行，不要用一条命令混跑
