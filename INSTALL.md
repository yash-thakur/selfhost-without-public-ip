# Installing the `dokploy-cloudflared-cgnat` skill

This is a local, self-contained skill (just `SKILL.md` + docs — no scripts or dependencies). Pick
the install path that matches how your agent discovers skills.

The skill source lives at:

```
/Users/yash/.kiro/crew/workspace/skills/dokploy-cloudflared-cgnat/
```

---

## Option A — Native Kiro skills directory (`~/.kiro/skills`)

Your native skills live under `~/.kiro/skills` (as a dir or a symlink per skill). Symlink this one
in so edits in the workspace stay live:

```bash
ln -s /Users/yash/.kiro/crew/workspace/skills/dokploy-cloudflared-cgnat ~/.kiro/skills/dokploy-cloudflared-cgnat
```

Or copy it if you'd rather it be standalone:

```bash
cp -R /Users/yash/.kiro/crew/workspace/skills/dokploy-cloudflared-cgnat ~/.kiro/skills/
```

Verify it's picked up:

```bash
ls -l ~/.kiro/skills/dokploy-cloudflared-cgnat/SKILL.md
```

---

## Option B — Global agent skills directory (`~/.agents/skills`)

The same directory your `skills.sh` installs land in (archify, playwright-cli). Symlinked across
supported agents:

```bash
ln -s /Users/yash/.kiro/crew/workspace/skills/dokploy-cloudflared-cgnat ~/.agents/skills/dokploy-cloudflared-cgnat
```

---

## Option C — KiroCrew dashboard (Settings → Skills)

In the KiroCrew dashboard: **Settings → Skills → Discover / Add**, then point it at this folder or
install from your skills source. KiroCrew reads the `SKILL.md` frontmatter (`name`, `description`)
for discovery and search.

---

## Confirm the agent can find it

Once installed, in a Kiro/KiroCrew session:

```
skill_search(query="dokploy cloudflare tunnel cgnat")
```

It should return the key `dokploy-cloudflared-cgnat`. Load it with:

```
skill_search(action='read', key='dokploy-cloudflared-cgnat')
```

or just mention `$dokploy-cloudflared-cgnat` to load it explicitly.

---

## File layout

```
dokploy-cloudflared-cgnat/
├── SKILL.md      # the skill the agent loads (frontmatter + procedure)
├── README.md     # human setup guide (this project's full walkthrough)
└── INSTALL.md    # this file
```

## Updating

Because Options A/B use a **symlink** back into the workspace, editing
`/Users/yash/.kiro/crew/workspace/skills/dokploy-cloudflared-cgnat/SKILL.md` updates the installed
skill immediately. If you **copied** instead, re-copy after edits.
