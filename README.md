# claude-skills

Public collection of [Claude Code](https://claude.com/claude-code) skills in the standard `SKILL.md` format.

## Skills

### [limit-break](limit-break/) · 限界突破

<img src="assets/limit-break-banner.webp" alt="A silhouette breaking through a glowing wall, shards and energy bursting out" width="896">

**English**

The boss looks unbeatable. Your best strategy has hit a wall. This is where Limit Break begins—not with a louder battle cry, but with a sharper question: what is actually holding the result back? This Claude Code skill turns an “impossible” goal into a testable hypothesis. It defines what success would mean, finds the dominant bottleneck, and challenges the assumptions that make the current path seem inevitable. Then it explores routes that work by different mechanisms: push the existing approach further, remove the bottleneck, or borrow a method from another field. The strongest candidate gets a small, decisive test under the real constraints. Results, costs, and counterexamples determine the next move. A failed attempt is useful evidence, not a cue to repeat the same attack with more force. Limit Break brings the energy of a last-chance RPG move to serious problem solving: aim high, change the mechanism, and let reality decide what broke through.

**中文**

眼前的难题像打不倒的最终 Boss，惯用招式已经撞上极限。限界突破不是喊一句“再努力”，而是先看清真正卡住结果的主导瓶颈：把看似不可能的目标写成可检验的假设，分清必须守住的约束与未经证实的惯例，再寻找机制不同的出路——把现有方法推到极致、绕开瓶颈，或借用其他领域的方法重构问题。选出最有潜力的一击，先用低成本实验检验关键假设；让实际结果、代价与反例决定继续推进还是换路。失败会留下线索，成功也只按证据宣告。像绝境中蓄满能量的 Limit Break 一击：目标够大胆，出手要换机制，突破了多少由事实裁决。

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
