---
name: hermes-backup
description: "Pre-update backup: push skills/vault to GitHub, backup config."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [backup, update, safety]
---

# hermes-backup — pre-update safety backup

## When

Before running `hermes update`, to safeguard skills, vault, config, and env.

## Steps

### 1. Backup skills + vault to GitHub

```bash
cd /tmp/hermes-config
rm -rf vault skills
cp -r ~/Documents/ForHermes vault
cp -r ~/.hermes/skills/xunji-trains skills/
cp -r ~/.hermes/skills/xunji-diet skills/
cp -r ~/.hermes/skills/xunji-body skills/
git add -A
git commit -m "pre-update backup $(date +%Y-%m-%d)"
TOKEN=$(gh auth token)
git push "https://FortunateAdventurer:${TOKEN}@github.com/FortunateAdventurer/hermes-config.git" main
```

### 2. Backup config files

```bash
cp ~/.hermes/config.yaml ~/.hermes/config.yaml.bak
cp ~/.hermes/.env ~/.hermes/.env.bak
```

### 3. Run update

```bash
hermes update
```

## Recovery (if update breaks)

```bash
cp ~/.hermes/config.yaml.bak ~/.hermes/config.yaml
cp ~/.hermes/.env.bak ~/.hermes/.env
git clone https://github.com/FortunateAdventurer/hermes-config.git /tmp/hermes-config
cp -r /tmp/hermes-config/skills/* ~/.hermes/skills/
```
