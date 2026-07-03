+++
title = "HTB Season Bot: Spoiler-Free Machine Discussion for Discord"
date = "2026-05-15T00:00:00-05:00"
tags = ["python", "discord", "hackthebox", "automation", "bot", "scripting"]
description = "A Discord bot that monitors the HTB-Updates bot, extracts box names from root flag submissions, and auto-creates solver-only channels so rooted machines can be discussed freely without spoiling anyone still working through them."
draft = false
+++

HTB seasonal Discord servers all have the same problem. Everyone's at a different point on the same machine, and literally anything posted about a box can spoil it for someone still stuck on it. The usual fixes, spoiler tags, someone manually making channels, just trusting people not to talk, all fall apart once the server gets big enough.

So I wrote a bot that handles it automatically instead of relying on people remembering to be careful.

## How It Works

HTB runs its own bot, HTB-Updates, that posts to a channel every time someone submits a root flag. That post has the machine name and who solved it.

My bot just watches that channel. When a root flag notification comes through, it:

1. Parses the post for the machine name
2. Checks if a channel for that machine already exists
3. Creates one with locked-down permissions if it doesn't
4. Restricts it to people who've actually solved the box

Anyone who hasn't rooted it yet can't even see the channel exists, let alone read it or use it to cheat their way through. Once you do root it, you get access the moment your flag posts through HTB-Updates, no manual step needed.

## What It's Replacing

Before this, seasonal servers basically had three bad options: one big channel where spoilers are unavoidable, an admin manually pinning and creating channels for every box every season, or spoiler tags that still leak into notification previews anyway.

Now it's just zero-maintenance. A channel appears the second the first person solves a box, and who gets in is based on actual solve data instead of the honor system.

## Requirements

- Discord bot token with channel management permissions
- HTB-Updates bot present and posting to a monitored channel in the same server
- Python 3.10+
- `discord.py`
