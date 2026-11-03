# X / YouTube 抓取代码模板

配合 SKILL.md Phase 1 使用。所有模板已按本机环境写死:`/usr/bin/python3`、xvfb-run 前缀、`--no-sandbox`。

## §0 运行纪律(内存红线,先读)

- **chromium 并发会爆内存,服务器红线 3.6G**:所有 playwright 任务**串行**,一次只跑一个实例。
- 跑完必清理并确认内存回落:

```bash
pkill -f chromium 2>/dev/null; sleep 1; free -m | head -2
```

- 统一启动方式(无头显示靠 xvfb):

```bash
xvfb-run -a -s "-screen 0 1280x1024x24" /usr/bin/python3 scrape.py <args>
```

- 依赖一次装齐:`/usr/bin/python3 -m pip install --break-system-packages playwright && playwright install chromium`

## §1 X 用户时间线(实战:@poteto 18 条帖)

**背景事实**:x.com 直连 urllib/requests 100% 撞 JS 墙,拿到的是空壳。唯一可靠路径 = playwright 渲染后取 `document.body.innerText`。viewport 拉高到 1380x3000,一屏装下更多帖子,滚动 2-3 屏覆盖时间线首屏内容。

```python
#!/usr/bin/env python3
# scrape_x_timeline.py <username> <out_path>
# 用法: xvfb-run -a /usr/bin/python3 scrape_x_timeline.py poteto raw/x_poteto_18posts.txt
import sys
from playwright.sync_api import sync_playwright

user, out_path = sys.argv[1], sys.argv[2]
UA = ("Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 "
      "(KHTML, like Gecko) Chrome/126.0.0.0 Safari/537.36")

screens = []
tweets_json = []          # 辅助路径:顺手收 JSON API 响应

with sync_playwright() as p:
    browser = p.chromium.launch(args=["--no-sandbox", "--disable-dev-shm-usage"])
    ctx = browser.new_context(
        viewport={"width": 1380, "height": 3000},   # 拉高 viewport 是关键
        user_agent=UA, locale="en-US",
    )
    page = ctx.new_page()

    def on_response(resp):                            # 结构化数据,能收就收
        if "/UserTweets" in resp.url:
            try:
                tweets_json.append(resp.json())
            except Exception:
                pass
    page.on("response", on_response)

    page.goto(f"https://x.com/{user}", wait_until="domcontentloaded", timeout=60000)
    page.wait_for_timeout(5000)                       # 等 X 的前端hydration,宁等勿抢

    for i in range(3):                                # 滚 3 屏
        screens.append(page.evaluate("document.body.innerText"))
        page.mouse.wheel(0, 3000)
        page.wait_for_timeout(2500)                   # 懒加载需要时间

    browser.close()

# 去重合并:相邻屏内容大量重叠,按行保序去重
seen, merged = set(), []
for scr in screens:
    for line in scr.splitlines():
        line = line.rstrip()
        if line and line not in seen:
            seen.add(line)
            merged.append(line)

text = "\n".join(merged)
with open(out_path, "w", encoding="utf-8") as f:
    f.write(text)
print(f"[innerText] {len(merged)} 行 -> {out_path}")
if tweets_json:
    import json
    jp = out_path.replace(".txt", ".json")
    with open(jp, "w", encoding="utf-8") as f:
        json.dump(tweets_json, f, ensure_ascii=False)
    print(f"[json api ] {len(tweets_json)} 个响应 -> {jp}")
```

**解析口径**:innerText 是页面真实文本(时间戳行、正文、互动数混排),交给后续 Phase 2 由人/模型按帖切分——比赌 DOM 结构稳。JSON 响应只做交叉校对(帖号、日期),不单独作正文来源。

## §2 X 文章合集页(实战:@leaf_sanren,1380x3000 滚 3 屏)

文章合集页比时间线好抓:正文都在。两个坑要处理——**标题锚点定位**每篇的起点、**尾部 "More from" 推荐区截断**。

```python
#!/usr/bin/env python3
# scrape_x_articles.py <username> <titles_comma_separated> <out_path>
import sys
from playwright.sync_api import sync_playwright

user, titles_raw, out_path = sys.argv[1], sys.argv[2], sys.argv[3]
titles = [t.strip() for t in titles_raw.split("|") if t.strip()]

with sync_playwright() as p:
    browser = p.chromium.launch(args=["--no-sandbox", "--disable-dev-shm-usage"])
    ctx = browser.new_context(
        viewport={"width": 1380, "height": 3000},
        user_agent="Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 "
                   "(KHTML, like Gecko) Chrome/126.0.0.0 Safari/537.36",
    )
    page = ctx.new_page()
    page.goto(f"https://x.com/{user}", wait_until="domcontentloaded", timeout=60000)
    page.wait_for_timeout(5000)

    screens = []
    for _ in range(3):
        screens.append(page.evaluate("document.body.innerText"))
        page.mouse.wheel(0, 3000)
        page.wait_for_timeout(2500)
    browser.close()

text = "\n".join(screens)
# 坑1: 尾部 "More from ..." 是推荐区,不是本合集正文
for marker in ("\nMore from ", "More from "):
    if marker in text:
        text = text.split(marker)[0]
        break

# 坑2: 用已知标题做锚点切出每篇正文(标题必须与页面渲染文本逐字一致)
pos = []
for t in titles:
    i = text.find(t)
    if i < 0:
        print(f"[警告] 标题未命中: {t}")     # 未命中宁缺毋滥,人工复查
    else:
        pos.append((i, t))
pos.sort()

with open(out_path, "w", encoding="utf-8") as f:
    for n, (i, t) in enumerate(pos):
        end = pos[n + 1][0] if n + 1 < len(pos) else len(text)
        f.write(f"\n{'='*60}\n## {t}\n{'='*60}\n{text[i:end].strip()}\n")
print(f"[ok] 命中 {len(pos)}/{len(titles)} 篇 -> {out_path}")
```

## §3 YouTube 字幕:yt-dlp 首选

```bash
VIDEO="https://www.youtube.com/watch?v=XXXXXXXXXXX"
yt-dlp --write-auto-sub --sub-langs "en.*" --skip-download \
       --convert-subs srt -o "raw/yt_%(id)s.%(ext)s" "$VIDEO"
```

auto-sub 转成 srt 后清洗成纯文本(去时间轴、去滚动重复行):

```python
#!/usr/bin/env python3
# srt_to_text.py <in.srt> <out.txt>
import re, sys

src, dst = sys.argv[1], sys.argv[2]
seen, out = set(), []
for ln in open(src, encoding="utf-8-sig"):
    ln = ln.strip()
    if not ln or ln.isdigit() or "-->" in ln or ln.startswith(("Kind:", "Language:")):
        continue
    ln = re.sub(r"<[^>]+>", "", ln)          # 去内联标签
    if ln and ln not in seen:                # srt 仍可能有整行重复
        seen.add(ln)
        out.append(ln)
open(dst, "w", encoding="utf-8").write("\n".join(out))
print(f"[ok] {len(out)} 行 -> {dst}")
```

## §4 YouTube 兜底:playwright + Show transcript + TreeWalker

**什么时候用**:yt-dlp 报 "Sign in to confirm you're not a bot"(数据中心 IP 常见)。playwright 带完整浏览器指纹,大概率能过。**为什么用 TreeWalker**:transcript 面板的元素结构常改,但文本节点永远在——按文本节点遍历比赌 `querySelector` 选择器稳。

```python
#!/usr/bin/env python3
# scrape_yt_transcript.py <video_url> <out_path>
import re, sys
from playwright.sync_api import sync_playwright

url, out_path = sys.argv[1], sys.argv[2]

with sync_playwright() as p:
    browser = p.chromium.launch(args=["--no-sandbox", "--disable-dev-shm-usage"])
    ctx = browser.new_context(
        viewport={"width": 1380, "height": 2000},
        user_agent="Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 "
                   "(KHTML, like Gecko) Chrome/126.0.0.0 Safari/537.36",
        locale="en-US",
    )
    page = ctx.new_page()
    page.goto(url, wait_until="domcontentloaded", timeout=90000)
    page.wait_for_timeout(5000)

    # 1) 展开 description,让 "Show transcript" 按钮露出
    try:
        page.get_by_role("button", name=re.compile("more|Expand", re.I)).first.click(timeout=5000)
        page.wait_for_timeout(1500)
    except Exception:
        pass                                   # 有些视频默认就展开

    # 2) 点 "Show transcript"(可能藏在 expanded 状态下)
    clicked = False
    for sel in [
        page.get_by_role("button", name=re.compile("Show transcript", re.I)),
        page.locator("button", has_text=re.compile("transcript", re.I)),
    ]:
        try:
            sel.first.click(timeout=8000)
            clicked = True
            break
        except Exception:
            continue
    if not clicked:
        sys.exit("[fail] 找不到 Show transcript 按钮——UI 又改了,人工开一次页面确认路径")
    page.wait_for_timeout(4000)               # 等面板加载 segments

    # 3) TreeWalker 遍历面板文本节点:时间戳与正文交替出现
    segs = page.evaluate("""
    () => {
      const root = document.querySelector(
        'ytd-transcript-renderer, [data-testid="transcript-segment-list"], ' +
        'ytd-engagement-panel-section-list-renderer[target-id*="transcript"]');
      if (!root) return null;
      const out = [];
      const walker = document.createTreeWalker(root, NodeFilter.SHOW_TEXT, {
        acceptNode: n => n.textContent.trim() ? NodeFilter.FILTER_ACCEPT
                                              : NodeFilter.FILTER_SKIP
      });
      let node;
      while ((node = walker.nextNode())) out.push(node.textContent.trim());
      return out;
    }""")
    browser.close()

if not segs:
    sys.exit("[fail] 面板存在但没取到文本,检查 root 选择器")

# 4) 配对还原:偶数位是时间戳(mm:ss / hh:mm:ss),其后是正文
lines, buf = [], []
ts_re = re.compile(r"^\d{1,2}:\d{2}(:\d{2})?$")
for s in segs:
    if ts_re.match(s) and buf:
        lines.append(" ".join(buf)); buf = []
    elif not ts_re.match(s):
        buf.append(s)
if buf:
    lines.append(" ".join(buf))

open(out_path, "w", encoding="utf-8").write("\n".join(lines))
print(f"[ok] {len(lines)} 段 -> {out_path}")   # 实战基准: Maven 工作坊 461 段
```

## §5 通用网页渲染兜底

curl 探底正文 < 500 字符(JS 空壳)时:

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    b = p.chromium.launch(args=["--no-sandbox", "--disable-dev-shm-usage"])
    pg = b.new_context(viewport={"width": 1280, "height": 3000}).new_page()
    pg.goto(url, wait_until="networkidle", timeout=60000)
    pg.wait_for_timeout(3000)
    text = pg.evaluate("document.body.innerText")
    b.close()
```

## §6 落盘命名与红线回顾

- `raw/` 目录,一来源一文件:`x_{标识}.txt` / `yt_{标识}.txt` / `web_{标识}.txt` / `pdf_{标识}.pdf`。
- 镜像站/索引站只校对帖号日期,不作正文来源;帖号冲突以实抓为准。
- 串行跑 chromium;结束 `pkill -f chromium`;`free -m` 确认内存回落再开下一个任务。
