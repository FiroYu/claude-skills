# claude-skills

Public collection of [Claude Code](https://claude.com/claude-code) skills in the standard `SKILL.md` format.

## Skills

### [limit-break](limit-break/) · 限界突破

<img src="assets/limit-break-banner-v2.webp" alt="A cloaked hero unleashing the innate skill Limit Break against a wall of despair" width="896">

**English**

The path ends here. The old method has failed, and the deadline is closing in. The hero draws one breath and releases the skill sealed until this moment: LIMIT BREAK. Choose the axis of the breakthrough—speed, altitude, depth, or range. Before striking, ask what an expert would demand: which work is missing, which original materials matter, which tools fit, and what evidence could settle the outcome. Then test paths that change the mechanism, including the one that looks too bold to try. Build something real. Let the source material challenge your assumptions and let results beyond your own judgment decide whether the move worked. If it fails, learn where the mechanism broke and strike again; if the battle returns, forge the lesson into a workflow that can run the next round. No victory is declared on conviction alone. The limit moves only as far as the evidence shows.

**中文**

旧路走到了尽头，期限步步逼近。勇者握紧剑柄，解放固有技能——「限界突破」。先选定要突破的方向：速度、高度、深度，或广度；再把专家会追问的缺口、原始材料、具体工具和外部判据写进任务。不要只把旧招使得更猛，还要寻找能绕开瓶颈、重构流程的不同机制，哪怕其中一条看起来太过大胆。拿出真正的作品或实验，让原始材料推翻臆测，让真实反馈裁决成败。失败了，就查清是哪项假设或机制失效，再出下一招；若战斗会反复到来，就把验证与行动锻造成持续运转的工作流。胜利无需夸口，突破到了哪里，只由证据作答。

Method source · 方法来源: [How to be more ambitious — Felix Rieseberg](https://felixrieseberg.com/how-to-be-more-ambitious-a-tutorial-for-humans-with-models/)

## Layout

Each folder is one skill:

```
<skill-name>/
└── SKILL.md
```

## Install

Copy a skill folder into your Claude Code skills directory:

```bash
git clone https://github.com/FiroYu/claude-skills.git
cp -r claude-skills/<skill-name> ~/.claude/skills/
```

The skill loads automatically on the next session.

## Scope

Only general-purpose, self-authored skills live here. Business-specific and personal skills stay in a private repo.
