# Claude Code Skills

Personal skills for Claude Code (`~/.claude/skills/`), synced across machines via this repo.

## Usage on a new machine

```
git clone git@github.com:Skywork1234/claude-skills.git ~/.claude/skills
```

## Usage on a machine with existing skills

```
cd ~/.claude/skills
git init
git remote add origin git@github.com:Skywork1234/claude-skills.git
git add .
git commit -m "Add existing skills"
git branch -M main
git push -u origin main
```

## Keeping in sync

- After creating/editing a skill: `cd ~/.claude/skills && git add . && git commit -m "..." && git push`
- On another machine: `cd ~/.claude/skills && git pull`
