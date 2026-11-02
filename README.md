# panel-debate

A Claude Code skill for multi-model adversarial review: a cheap prep seat builds a context pack, several premium models review blind one at a time, a head judge verifies and cross-examines, contested points go through rebuttal rounds, and a cheap model writes a plain-English summary. Sequential and token-efficient by design.

## Install

```bash
git clone https://github.com/quang30/panel-debate.git
cp -r panel-debate/panel-debate ~/.claude/skills/
```

Then read the **Setup** section in `panel-debate/SKILL.md`. The skill drives an engine script (`scripts/panel_debate.sh` + `scripts/panel_lib.sh`) that you supply or write for your own seats; it is not included here.

## Use

Say `debate this`, `debate two`, `debate three`, or `/panel-debate <target> [--rounds N]`.
