---
name: curate-source-to-publish
description: >-
  Use when the task involves 搜集/抓取多平台一手素材(X/Twitter、YouTube、通用网页)并整理、翻译、编辑、校验,再用
  reportlab 排版成中文 PDF 发布到 GitHub——"从来源到发布"的完整内容策展流水线。触发词:整理成文章/PDF、搜集资料、抓
  X 推文、YouTube 字幕/逐字稿、翻译编辑、做电子书、发布到 learning-vault。
---

# 从来源到发布:多平台内容策展流水线

把散落在 X / YouTube / 网页里的一手素材,做成一本**可溯源、署名规范、经过校验**的中文 PDF,发布到公开仓库。

三条铁律,开工前先背下来:

1. **一手素材优先,能下载的不抓取。** 素材已是成品就直接下载(实战:《制高点》原版 PDF 145 页,零抓取)。抓取只用于"内容埋在 JS 墙里"的场景。
2. **诚实性是红线。** 每句引文可溯源;拿不到的素材明说拿不到;病毒式假名言查无实据就标"存疑"。宁可附录空一节,不编一句原文。
3. **发布即公开。** 推公开库前,署名三处、README 链接、远端文件树逐一核对;`git push` 成功 ≠ 远端就位。

```
Phase 0 需求 → 1 搜集(raw/) → 2 整理(定结构) → 3 翻译 → 4 编辑(成章)
            → 5 校验(诚实性) → 6 排版(reportlab PDF) → 7 发布(learning-vault)
```

详细代码模板按需读,不必一次读完:

- X 时间线/文章页、YouTube 字幕/逐字稿抓取 → [references/x-youtube-scraping.md](references/x-youtube-scraping.md)
- reportlab 排版配方 + 发布检查单 → [references/pdf-publish-toolchain.md](references/pdf-publish-toolchain.md)

---

## Phase 0 明确需求

用户说"整理成文章/PDF"远不够开工。**一条消息问完四件事**,不要挤牙膏式来回:

| 问题 | 典型选项 | 影响 |
|---|---|---|
| 给谁看 | 自用 / 公开发布 | 公开 → Phase 7 全流程 + 三处署名 |
| 吃瓜还是干货 | 人物全档案 / 纯方法论 | 决定素材筛选口径和叙事口吻 |
| 目标体量 | 10 页 / 30 页 / 100 页 | 决定素材量(经验值:1 页 ≈ 2-3 条帖子的信息量) |
| 语言 | 纯中文 / 金句双语 | 双语块排版要预留样式(见 Phase 6) |

> ▸ 实战教训(土豆姐 @poteto 案例):第一版按"CP 全档案"口径做了人物关系大合集,用户反馈"不需要吃瓜"→ 整本重做成纯方法论版。开工前 30 秒的确认,省掉几小时返工。

---

## Phase 1 搜集:一手素材优先

### 第一步永远是判断素材形态

- 已是 PDF/EPUB/视频文件 → 直接 `curl -L -o` 下载,结束本 Phase。
- 内容埋在网页/帖子/视频里 → 走下面的抓取路径。

### 通用网页:先 curl 探底,空壳再上 playwright

```bash
UA="Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/126.0 Safari/537.36"
LEN=$(curl -sL -A "$UA" "$URL" | /usr/bin/python3 -c "
import sys,re
h=sys.stdin.read()
h=re.sub(r'<(script|style)[^>]*>.*?</\1>','',h,flags=re.S)
print(len(re.sub(r'<[^>]+>',' ',h).strip()))")
echo "正文文本量: $LEN 字符"   # < 500 → JS 空壳,curl 白搭,直接上 playwright
```

### X/Twitter:直连必死,渲染才活

urllib/requests 直连 x.com **100% 撞 JS 墙**,不要浪费时间。唯一可靠路径:

```
xvfb-run + playwright → viewport 拉高(1380x3000) → 滚动 2-3 屏 → 取 document.body.innerText
```

- 抓用户时间线(帖子流)或文章合集,完整代码模板见 [references/x-youtube-scraping.md](references/x-youtube-scraping.md) §1-2。
- 辅助路径:监听 playwright `response` 事件抓站点 JSON API(更结构化,但偶尔缺数据;innerText 为主、JSON 为辅)。

### YouTube:yt-dlp 首选,TreeWalker 兜底

```bash
yt-dlp --write-auto-sub --sub-langs "en.*" --skip-download -o "raw/yt_%(id)s.%(ext)s" "$VIDEO_URL"
```

被 bot-check 拦(数据中心 IP 常见,报 "Sign in to confirm you're not a bot")→ playwright 打开视频页点 "Show transcript",TreeWalker 遍历 segment 面板提取。完整模板见 [references/x-youtube-scraping.md](references/x-youtube-scraping.md) §3-4。

> ▸ 实战(Maven 工作坊):yt-dlp 被 bot-check 拦死,playwright + TreeWalker 拿到 461 段逐字稿——绕墙路径不只是备胎,常常是唯一路径。

### 镜像站/索引站的正确用法

ythx-101.github.io 这类镜像站**只当目录校对用**(核对帖号、日期、数量),**不当正文来源**——正文以自己实抓为准。

> ▸ 实战(帖号勘误):索引站给的帖号与实抓差一位。以实抓为准,附录勘误说明。索引站会错,实抓才是证据。

### 落盘规范

```
raw/
  x_poteto_18posts.txt              # {类型}_{标识}.txt,类型前缀: x_ / yt_ / web_ / pdf_
  yt_maven_workshop_transcript.txt
  web_leaf_sanren_articles.txt
```

一个来源一个文件。Phase 5 的溯源校验、附录的链接索引,全部依赖这一步的落盘。

---

## Phase 2 整理:重组,不是堆砌

素材堆砌 = 把 raw/ 按时间顺序粘一遍,没人看得下去。**按主题重组**:

```
序        为什么值得读、素材从哪来(1-2 页)
第 1..N 章 按主题切分,不按时间;X 帖做观点引入,长文/逐字稿做展开论证
附录 A    原文链接索引(编号 → URL → 抓取日期,逐条可溯源)
附录 B    勘误与存疑(Phase 5 的产出落这里)
```

> ▸ 实战(土豆姐 8 章):18 条 X 帖 + 461 段逐字稿,不是"帖子区 + 演讲区"两坨,而是按方法论主题切 8 章、30 页——素材的原始形态≠成书结构。

---

## Phase 3 翻译:分层处理

| 内容类型 | 处理方式 |
|---|---|
| 金句/推文原文 | **双语块**:英文原文 + 中文翻译并列展示(排版样式见 Phase 6) |
| 普通叙述 | 意译,不逐词硬译;口语气顺成书面语 |
| 人名/术语 | 首次出现标注原文,如:土豆姐(poteto,phglm 前维护者) |

原则:读者要的是"读懂",不是"对照逐字稿";但金句的力量在原文,砍掉原文等于降级。

---

## Phase 4 编辑:碎片 → 连贯叙述

编辑视角 = 把碎片组织成**别人愿意一口气读完**的叙述。

- **逐字稿**:去掉口水话("um""对吧""呃")和重复表达,提炼为编辑叙述,关键处穿插**直接引用**——不是全文照录。
- **每章末尾加"编辑注"**:2-3 句实操启示,告诉读者"这章能怎么用"。灰字框样式,与正文区分。
- 章与章之间要有过渡逻辑(递进/对比/总分),不是并列的素材抽屉。

> ▸ 实战:Maven 工作坊 461 段逐字稿,成书后每章只保留 5-10 处直接引用,其余全部转写为叙述——引文是盐,不是主菜。

---

## Phase 5 校验:诚实性红线

### 5.1 机械校验(pypdf,Phase 6 生成后必跑)

```python
from pypdf import PdfReader
r = PdfReader("out.pdf")
for i, pg in enumerate(r.pages):
    t = (pg.extract_text() or "").strip()
    if len(t) < 5:
        print(f"[空白页/乱码嫌疑] p{i+1} 仅 {len(t)} 字符")
```

### 5.2 内容诚实性(人工逐条过,不可跳过)

- [ ] 每条直接引文都能在 raw/ 里找到出处
- [ ] 拿不到的素材**明说**:附录标注"无法提取,未假装有内容"。实战:某演讲视频字幕拿不到,就写拿不到——不编。
- [ ] 病毒式流传的名言:溯源核实;查无实据 → 附录标"存疑"
- [ ] 帖号/ID 双源核对:索引站 vs 实抓不一致时,以实抓为准 + 附录勘误

---

## Phase 6 排版:reportlab 中文 PDF

核心配方(完整可跑骨架见 [references/pdf-publish-toolchain.md](references/pdf-publish-toolchain.md) §2):

```python
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont
from reportlab.lib.pagesizes import A4

# 服务器唯一可用中文字体——先注册它,别的字体路径不用试
pdfmetrics.registerFont(TTFont('WQY', '/usr/share/fonts/truetype/wqy/wqy-zenhei.ttc', subfontIndex=0))
```

- **版面**:A4,四边 18mm;封面(大标题+副题+日期+落款)→ 目录 → 每章 `PageBreak` 起新页
- **页眉**:左书名、右"由 FiroYu 的 Agent 整理";**页脚**:页码 + 收录信息(仓库地址);`on_page` 回调跳过封面
- **金句块**:缩进 + 灰字,双语块英文行+中文行成对;**编辑注**:灰底 + 左竖线框(见 references 的样式代码)
- **生成后必须复验**:pypdf 重开,查页数 / 空白页 / 抽检 3-5 页文本是否正常(复验脚本在 references §3)——reportlab 不报错 ≠ 输出正常,中文字体丢注册时它会静默出空白页。

---

## Phase 7 发布:规则与事故预防

目标库:**FiroYu/learning-vault**(公开),定位 = 模型与 Agent 文献。

```
learning-vault/
  2026-10-27_poteto-agent-methodology/    # YYYY-MM-DD_主题/ 平铺,不搞嵌套分类
    poteto-agent-methodology.md
    poteto-agent-methodology.pdf
```

**署名规则(硬性,三处缺一不可)**,内容都含 `由 FiroYu 的 Agent 整理` + `github.com/FiroYu/learning-vault`:

1. md 文头引用块 + 文尾落款
2. PDF 封面落款
3. PDF 每页页眉 + 页脚

**发布后两步验证**:

```bash
# push 成功 ≠ 远端就位,必须 gh api 验文件树
gh api "repos/FiroYu/learning-vault/git/trees/HEAD?recursive=1" --jq '.tree[].path' | grep "$(date +%Y-%m-%d)"
```

- README 的"已有内容"区同步更新,链接指向**实际存在的路径**——贴完链接再 `gh api` 确认一次,404 链接 = 事故。
- 交付:大 PDF(>10MB)走 gofile;小 PDF 邮件附件 + gofile 双保险(命令在 references §5)。

---

## 工具链备忘(红线)

- **chromium 并发会爆内存**(服务器红线 3.6G):`xvfb-run` **串行**,一次一个实例;用完 `pkill -f chromium`, `free -m` 确认回落。
- **大 PDF(>10MB)交付**:gofile 新 API(`GET api.gofile.io/servers` → `POST {server}.gofile.io/contents/uploadfile`),完整命令见 references §5。
- **子进程跑 CC**:`claude --print --model glm-5.3 --dangerously-skip-permissions "$(cat task.md)"` — 新版 CC 可能不识别自定义模型名而**假退出**(退出码正常但什么都没干),跑完必须检查产物存在且非空。
- Python 一律 `/usr/bin/python3`;装包加 `--break-system-packages`(reportlab / pypdf / playwright)。

## 三个实战案例速查

| 案例 | 素材路径 | 产出 | 关键一招 |
|---|---|---|---|
| 土豆姐 @poteto Agent 方法论 | X 帖 18 条(playwright 实抓)+ Maven 工作坊逐字稿(yt-dlp 被拦 → TreeWalker 461 段) | 8 章 30 页 PDF → GitHub 开源库 | 逐字稿提炼为叙述 + 每章编辑注 |
| @leaf_sanren 文章合集 | x.com 直连 urllib 被 JS 墙 → xvfb playwright 渲染取 innerText(1380x3000 滚 3 屏) | reportlab PDF | 标题锚点定位每篇 + 尾部 "More from" 推荐区截断 |
| 《制高点》 | 原版 PDF 直接下载 | 145 页成品交付 | 能下载的不抓取 |
