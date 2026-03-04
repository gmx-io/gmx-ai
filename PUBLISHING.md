# Publishing Guide — gmx-trading Skill

## Pre-publish Checklist

```bash
# Validate structure
find . -type f | grep -v .git | sort

# SKILL.md < 500 lines
wc -l skills/gmx-trading/SKILL.md

# Description < 1024 chars
head -3 skills/gmx-trading/SKILL.md

# No placeholders
grep -r "TODO\|FIXME\|TBD" skills/

# Frontmatter starts with ---
head -1 skills/gmx-trading/SKILL.md

# .well-known matches skills
diff -r skills/gmx-trading .well-known/skills/gmx-trading
```

---

## Step 1 — Push to GitHub

Create the repo under `gmx-io` org:

```bash
git add -A
git commit -m "Initial release: gmx-trading skill v1.0"
gh repo create gmx-io/gmx-ai --public --source=. --push
```

After pushing, the skill is immediately installable via:

```bash
npx skills add gmx-io/gmx-ai
```

This works across Claude Code, Cursor, Codex CLI, Windsurf, Aider, and 30+ other agents.

---

## Step 2 — Claude Code Marketplace

The `.claude-plugin/marketplace.json` is already in place. Users install via:

```bash
# Add marketplace
/plugin marketplace add gmx-io/gmx-ai

# Install the skill
/plugin install gmx-trading@gmx-ai
```

The skill appears at `~/.claude/skills/gmx-trading/SKILL.md` after installation.

### Test locally first

```bash
# Copy to personal skills (note: copy contents, not the directory itself)
mkdir -p ~/.claude/skills/gmx-trading
cp -r skills/gmx-trading/* ~/.claude/skills/gmx-trading/

# Open Claude Code and test
claude
# Then type: /gmx-trading
# Verify it loads and responds correctly
```

---

## Step 3 — Web Discovery (.well-known)

The `.well-known/skills/` directory enables discovery at a predictable URL. After deploying to a web server (e.g., via GitHub Pages or Cloudflare Pages):

```
https://gmx-io.github.io/gmx-ai/.well-known/skills/index.json
```

### GitHub Pages setup

```bash
# Enable GitHub Pages on the repo (Settings → Pages → Source: main, / (root))
# Or via CLI:
gh api repos/gmx-io/gmx-ai/pages -X POST -f source.branch=main -f source.path=/
```

Agents that support the Cloudflare Agent Skills Discovery RFC will auto-discover skills from this URL.

---

## Step 4 — Submit to Anthropic's Official Skills Repo

The official repo at `github.com/anthropics/skills` accepts community contributions:

```bash
# Fork and clone
gh repo fork anthropics/skills --clone

# Add skill
cp -r skills/gmx-trading skills/community/gmx-trading

# Submit PR
cd skills
git checkout -b add-gmx-trading
git add -A
git commit -m "Add gmx-trading skill for GMX V2 perpetuals trading"
gh pr create --title "Add gmx-trading skill" \
  --body "Adds trading skill for GMX V2 — perpetuals and spot exchange on Arbitrum, Avalanche, and Botanix. Includes SDK reference, API endpoints, contract addresses, and order type documentation."
```

Once merged, the skill becomes available to all Claude Code users via:

```bash
/plugin marketplace add anthropics/skills
/plugin install gmx-trading@skills
```

---

## Step 5 — Cross-Platform Registries (Optional)

### skills.sh (Vercel directory)

Skills published on GitHub are automatically indexed by [skills.sh](https://skills.sh/) if the repo follows the standard structure. No manual submission needed.

### SkillsMP

Submit to [skillsmp.com](https://skillsmp.com) for discovery across 350,000+ skills:

```bash
# Install CLI
npm install -g agent-skills-cli

# Publish
agent-skills publish gmx-io/gmx-ai
```

### npm (optional)

```bash
# Add package.json
cat > package.json << 'EOF'
{
  "name": "@gmx-io/agent-skills",
  "version": "1.0.0",
  "description": "Agent skills for trading on GMX V2",
  "license": "MIT",
  "repository": "gmx-io/gmx-ai",
  "files": ["skills/", ".well-known/", ".claude-plugin/", "LICENSE", "README.md"]
}
EOF

npm publish --access public
```

---

## Step 6 — Announce

### Places to announce

1. **GMX Discord** — #announcements or #dev channel
2. **GMX Twitter/X** — Link to repo
3. **GMX Docs** — Add section to docs.gmx.io under SDK/Integration
4. **Anthropic Discord** — #skills channel
5. **awesome-agent-skills** — PR to [VoltAgent/awesome-agent-skills](https://github.com/VoltAgent/awesome-agent-skills)

### Example announcement

```
GMX is now available as an Agent Skill — the 2nd DeFi protocol after Uniswap.

Install in one command:
  npx skills add gmx-io/gmx-ai

Works with Claude Code, Cursor, Copilot, Gemini CLI, and 30+ other AI agents.

What it does:
• Open long/short positions with up to 100x leverage
• Swap tokens at oracle prices
• Market, limit, stop-loss, and take-profit orders
• Full SDK and API reference included

GitHub: github.com/gmx-io/gmx-ai
```

---

## Maintenance

### Updating the skill

```bash
# Edit files in skills/gmx-trading/
# Then sync to .well-known
cp -r skills/gmx-trading/* .well-known/skills/gmx-trading/

# Bump version in SKILL.md frontmatter
# Commit and push
git add -A
git commit -m "Update gmx-trading skill to v1.1"
git push
```

