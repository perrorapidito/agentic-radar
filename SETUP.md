# Setup

> 🌐 **English** · [Español](SETUP.es.md)

Built for [Claude Code](https://claude.com/claude-code). About 30 minutes the first time. After that, the radar runs itself every Monday.

If you can edit a spreadsheet, you can set this up. You don't need to know Python or any framework — you talk to Claude in the chat and Claude edits the files for you.

## 1. Install Claude Code

Download from [claude.com/claude-code](https://claude.com/claude-code). Sign in. Normal Mac or Windows app.

## 2. Add the three skills

A "slash command" in Claude Code is a pre-written instruction you trigger by typing `/something` in chat. This repo ships three.

1. Open Claude Code.
2. Download the three files in [`commands/`](commands/): `scan-targets.md`, `evaluate-target.md`, `radar.md`.
3. Drop them into `~/.claude/commands/` (Claude Code creates that folder on first launch).
4. Restart Claude Code. Type `/` in the chat. You should see `/scan-targets`, `/evaluate-target`, `/radar` in the list.

## 3. Build your target list

Make a markdown table of the companies you want the radar to check each week. The format lives in [`examples/targets-inventory.example.md`](examples/targets-inventory.example.md). Each row needs only the company name and its careers page URL. The radar figures out which job-board platform each one uses on the first run.

Save the file somewhere private. Don't commit it to your public fork — that's why `.gitignore` excludes `inventory/` by default.

## 4. Tune the filter to your role

Open the chat in Claude Code and tell Claude what kind of role you want. Examples that work:

> *"In `commands/scan-targets.md`, also accept Principal PM in the title filter."*

> *"Add Berlin and Amsterdam as valid locations."*

> *"Reject anything that mentions on-site mandatory."*

You don't need to open the file. Claude edits it for you.

## 5. Run it

In chat, type `/radar`.

First run is slower because the system has to discover each company's job-board platform via WebSearch. Subsequent runs reuse what it learned. When it finishes, you see the ranked table.

## 6. Schedule it weekly

In chat, type `/schedule`.

Give it:
- **Name**: `radar-weekly`
- **Cron**: `0 8 * * 1` (Monday 8:00, your timezone)
- **Command**: `/radar`

You're done. Every Monday morning the output is waiting.

## When you get stuck

Ask Claude in the chat. Examples that work:

> *"The scan returned 0 hits for Company X. What's wrong?"*

> *"Re-run the radar only for Lumora, DataForge, Northwind."*

> *"Add a new company to my inventory: Atomic AI, careers at https://atomic.ai/jobs"*

The skills are designed so Claude can fix, extend, and operate them through normal conversation.
