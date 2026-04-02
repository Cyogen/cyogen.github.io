+++
date = '2026-04-02T00:00:00-00:00'
draft = false
title = 'Tools'
+++

## HTB Season Bot

A Discord bot for Hack The Box seasonal servers. When a member earns the root flag on a seasonal machine, the bot automatically creates a dedicated channel for that machine (if one doesn't exist) and grants the solver read and post access. Everyone in the server can see the channel exists, but only root solvers can read and participate in it.

**Source code:** [github.com/Cyogen/htb-season-bot](https://github.com/Cyogen/htb-season-bot)

**Invite HTB Season Bot:** *(invite link coming soon)*

---

### Prerequisites

HTB Season Bot works by watching for badge notifications posted by the **HTB Updates** bot. HTB Updates must be in your server and posting to a channel before HTB Season Bot can do anything.

**Invite HTB Updates first:**
[https://discord.com/api/oauth2/authorize?client_id=806824180074938419&permissions=277025441856&scope=bot+applications.commands](https://discord.com/api/oauth2/authorize?client_id=806824180074938419&permissions=277025441856&scope=bot+applications.commands)

HTB Updates requires you to link your Hack The Box account. Follow the instructions on the [HTB Updates GitHub page](https://github.com/Et3rnos/HTB_Updates) to connect your members' HTB accounts to Discord so their completions are announced.

---

### Setup

Once both bots are in your server:

1. **Create a category** in your Discord server where solved-machine channels will live (e.g. `Season 10 Machines`)
2. Run `/setup` — the bot will guide you through two steps:
   - Select the channel where HTB Updates posts badge notifications
   - Select the category where machine channels should be created
3. That's it. The bot handles everything from there

---

### How It Works

When a member earns root on a seasonal machine, HTB Updates posts a badge to your configured channel. HTB Season Bot reads that badge and:

- If no channel exists for that machine yet — creates `#machinename` in your category with a congratulations message
- If the channel already exists — adds the new solver and announces them
- In both cases, only confirmed root solvers can read and post in the channel

User-flag completions are ignored — root only.
