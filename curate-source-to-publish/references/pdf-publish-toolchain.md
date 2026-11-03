# reportlab 中文 PDF 配方 + 发布检查单

配合 SKILL.md Phase 6/7 使用。

## §1 环境事实(本机,别浪费尝试)

- Python 一律 `/usr/bin/python3`;装包:`/usr/bin/python3 -m pip install --break-system-packages reportlab pypdf`
- **服务器唯一可用中文字体**:`/usr/share/fonts/truetype/wqy/wqy-zenhei.ttc`(subfontIndex=0)。别的字体路径不用试。
- wqy-zenhei 没有真正的斜体/粗体变体——视觉区分靠**灰色/字号/缩进**,不靠 italic/bold 渲染(正文加粗 reportlab 会用伪粗,效果尚可,金句一律用灰色区分)。

## §2 build_book.py 完整骨架

```python
#!/usr/bin/env python3
# build_book.py <book.md 素材> <out.pdf>   —— 按需改成数据驱动/直接内嵌段落
from datetime import date
from reportlab.lib import colors
from reportlab.lib.pagesizes import A4
from reportlab.lib.styles import ParagraphStyle
from reportlab.lib.units import mm
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont
from reportlab.platypus import (BaseDocTemplate, Frame, PageTemplate, Paragraph,
                                Spacer, Table, TableStyle, PageBreak)
from reportlab.platypus.tableofcontents import TableOfContents

# ---------- 元数据(署名三处共用,改这里即可) ----------
BOOK_TITLE = "书名"
SUBTITLE   = "副题"
AUTHOR_LINE = "由 FiroYu 的 Agent 整理"                    # 署名,逐字保留
REPO       = "github.com/FiroYu/learning-vault"            # 收录仓库
DATE_STR   = date.today().isoformat()

W, H = A4
M = 18 * mm                                                  # 四边距
CONTENT_W = W - 2 * M

pdfmetrics.registerFont(TTFont("WQY", "/usr/share/fonts/truetype/wqy/wqy-zenhei.ttc",
                               subfontIndex=0))
pdfmetrics.registerFontFamily("WQY", normal="WQY", bold="WQY", italic="WQY",
                              boldItalic="WQY")

# ---------- 样式表 ----------
st_cover   = ParagraphStyle("cover",  fontName="WQY", fontSize=26, leading=38,
                            alignment=1, textColor=colors.HexColor("#222222"))
st_cover_s = ParagraphStyle("csub",    fontName="WQY", fontSize=13, leading=20,
                            alignment=1, textColor=colors.HexColor("#666666"))
st_h1      = ParagraphStyle("H1",      fontName="WQY", fontSize=17, leading=24,
                            spaceBefore=14, spaceAfter=10)
st_body    = ParagraphStyle("body",    fontName="WQY", fontSize=10.5, leading=17,
                            firstLineIndent=21)
st_quote_en = ParagraphStyle("qen", parent=st_body, leftIndent=14*mm, rightIndent=8*mm,
                             textColor=colors.HexColor("#666666"), fontSize=9.5,
                             firstLineIndent=0)               # 金句英文原文
st_quote_zh = ParagraphStyle("qzh", parent=st_quote_en,
                             textColor=colors.HexColor("#444444"))  # 金句中文翻译
st_note    = ParagraphStyle("note", fontName="WQY", fontSize=9.5, leading=15,
                            textColor=colors.HexColor("#555555"))

# ---------- 页眉页脚(on_page 回调;封面不排) ----------
def on_page(canvas, doc):
    if doc.page == 1:
        return                                               # 封面干净
    canvas.saveState()
    canvas.setFont("WQY", 8)
    canvas.setFillColorRGB(0.45, 0.45, 0.45)
    y = H - 12 * mm
    canvas.drawString(M, y, BOOK_TITLE)                      # 页眉左: 书名
    canvas.drawRightString(W - M, y, AUTHOR_LINE)            # 页眉右: 署名
    canvas.drawCentredString(W / 2, 12 * mm,
        f"{doc.page} · 收录:{REPO}")                          # 页脚: 页码+收录信息
    canvas.restoreState()

# ---------- 文档骨架(带目录需要 multiBuild 两遍收敛页码) ----------
class BookDoc(BaseDocTemplate):
    def afterFlowable(self, flowable):
        if isinstance(flowable, Paragraph) and flowable.style.name == "H1":
            key = f"h1-{self.page}-{abs(hash(flowable.getPlainText())) % 10**8}"
            self.canv.bookmarkPage(key)
            self.notify("TOCEntry", (0, flowable.getPlainText(), self.page, key))

doc = BookDoc("out.pdf", pagesize=A4, width=W, height=H,
              leftMargin=M, rightMargin=M, topMargin=M, bottomMargin=M,
              title=BOOK_TITLE, author=f"FiroYu ({AUTHOR_LINE})")
frame = Frame(M, M, CONTENT_W, H - 2 * M, id="main")
doc.addPageTemplates([PageTemplate(id="all", frames=[frame], onPage=on_page)])

# ---------- 内容元素 ----------
def quote_block(en, zh):                                     # 金句双语块
    return [Paragraph(en, st_quote_en), Paragraph(zh, st_quote_zh),
            Spacer(1, 3 * mm)]

def editor_note(text):                                       # 编辑注框: 灰底+左竖线
    t = Table([[Paragraph(f"<b>编辑注</b>  {text}", st_note)]],
              colWidths=[CONTENT_W - 6 * mm])
    t.setStyle(TableStyle([
        ("BACKGROUND",   (0, 0), (-1, -1), colors.HexColor("#F2F2F2")),
        ("LINEBEFORE",   (0, 0), (0, -1), 2, colors.HexColor("#999999")),
        ("LEFTPADDING",  (0, 0), (-1, -1), 10),
        ("RIGHTPADDING", (0, 0), (-1, -1), 10),
        ("TOPPADDING",   (0, 0), (-1, -1), 6),
        ("BOTTOMPADDING",(0, 0), (-1, -1), 6),
    ]))
    return t

toc = TableOfContents()
toc.levelStyles = [ParagraphStyle("toc1", fontName="WQY", fontSize=11,
                                  leading=18, leftIndent=6)]

story = [
    # 封面: 大标题+副题+日期+落款(署名处之一)
    Spacer(1, 60 * mm),
    Paragraph(BOOK_TITLE, st_cover),
    Spacer(1, 6 * mm),
    Paragraph(SUBTITLE, st_cover_s),
    Spacer(1, 4 * mm),
    Paragraph(DATE_STR, st_cover_s),
    Spacer(1, 30 * mm),
    Paragraph(f"{AUTHOR_LINE}<br/>{REPO}", st_cover_s),       # 封面落款
    PageBreak(),
    Paragraph("目录", st_h1), toc, PageBreak(),               # 目录页
    # …… 每章: Paragraph(章题, st_h1) + 正文/金句块,章末 editor_note(...),PageBreak()
]
doc.multiBuild(story)                                        # TOC 必须两遍构建
print("built: out.pdf")
```

要点:

- **字体注册必须是第一件事**;丢注册时 reportlab 不一定报错,而是静默产出空白/豆腐块页——所以 §3 复验是硬性步骤。
- `doc.multiBuild(story)` 而非 `build`:TableOfContents 页码要两遍才收敛。
- 封面落款、页眉右、页脚三处署名引用同一组常量 `AUTHOR_LINE` / `REPO`,只改一处配置。

## §3 生成后必须复验(pypdf,跳过=裸奔)

```python
#!/usr/bin/env python3
# verify_pdf.py <out.pdf>
import random, sys
from pypdf import PdfReader

r = PdfReader(sys.argv[1])
n = len(r.pages)
print(f"页数: {n}")

texts = [(p.extract_text() or "").strip() for p in r.pages]
blank = [i + 1 for i, t in enumerate(texts) if len(t) < 5]
print("空白页(<5字符):", blank if blank else "无")

sample = random.sample(range(1, n), min(5, n - 1))            # 抽检 3-5 页正文
for i in sorted(sample):
    print(f"p{i+1}: {texts[i][:60].replace(chr(10), ' ')}")

assert "FiroYu" in texts[0], "封面缺署名!"                     # 诚实性+署名双查
assert any("learning-vault" in t for t in texts[:5]), "页脚缺收录信息!"
print("PASS")
```

## §4 发布检查单(逐项打勾才算发布完成)

目标库 **FiroYu/learning-vault**(公开),定位 = 模型与 Agent 文献。

- [ ] 目录:`YYYY-MM-DD_主题/` 平铺于仓库根(例:`2026-10-27_poteto-agent-methodology/`),内含 `主题.md` + `主题.pdf`(+ 可选 raw 索引;附录已有链接索引时可不放 raw)。
- [ ] md 署名①文头引用块 + ②文尾落款:

```markdown
> 本手册由 FiroYu 的 Agent 整理自公开素材,来源与勘误见附录。
> 收录仓库:[FiroYu/learning-vault](https://github.com/FiroYu/learning-vault)

(正文……)

---
*由 FiroYu 的 Agent 整理 · 收录于 [FiroYu/learning-vault](https://github.com/FiroYu/learning-vault)*
```

- [ ] PDF 署名③封面落款 ④每页页眉 ⑤每页页脚——都含 `由 FiroYu 的 Agent 整理` 与 `github.com/FiroYu/learning-vault`。
- [ ] §3 复验 PASS(页数 / 无空白页 / 抽检文本正常 / 署名在场)。
- [ ] `git add` + `commit` + `push`。
- [ ] **gh api 验证远端文件树**(push 成功 ≠ 远端就位):

```bash
gh api "repos/FiroYu/learning-vault/git/trees/HEAD?recursive=1" \
  --jq '.tree[].path' | grep "$(date +%Y-%m-%d)"
```

- [ ] README"已有内容"区同步新增一行,链接指向**实际存在的路径**(贴完再 `gh api` 查一次该路径;404 链接 = 事故)。
- [ ] 交付:大 PDF(>10MB)走 §5 gofile;小 PDF 邮件附件 + gofile 双保险。

## §5 大文件交付:gofile(新 API,两步)

```bash
PDF="out.pdf"   # >10MB 走这里
SERVER=$(curl -s https://api.gofile.io/servers | /usr/bin/python3 \
         -c "import sys,json;print(json.load(sys.stdin)['data']['servers'][0]['name'])")
curl -s -F "file=@${PDF}" "https://${SERVER}.gofile.io/contents/uploadfile" \
  | /usr/bin/python3 -c "import sys,json;d=json.load(sys.stdin);print(d['status'],d['data']['downloadPage'])"
```

注意是**新 API**:先 `GET api.gofile.io/servers` 拿 server 名,再 `POST {server}.gofile.io/contents/uploadfile`;旧域名 `gofile.io/upload` 已失效,直接 POST 根域会 404。

## §6 用 CC 子进程跑排版/长任务的坑

```bash
claude --print --model glm-5.3 --dangerously-skip-permissions "$(cat task.md)"
```

- **坑**:新版 CC 可能不识别自定义模型名 → 正常退出码但什么都没干(假退出)。
- **铁律**:跑完必须验证产物存在且非空,再认定成功:

```bash
[ -s "$OUT_FILE" ] && echo "产物 OK: $(du -h "$OUT_FILE")" || echo "假退出,产物缺失,需重跑"
```
