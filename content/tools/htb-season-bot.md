+++
title = "HTB Season Bot"
date = "2026-04-02T00:00:00-05:00"
tags = ["discord", "hackthebox", "python", "automation"]
description = "A Discord bot that automatically creates spoiler-locked channels for HackTheBox machines as they get solved."
draft = false
+++

A Discord bot I built for HTB seasonal servers, mostly to solve one annoying problem: how do people actually talk about a machine without spoiling it for everyone still working on it?

When someone submits a root flag, HTB's own bot posts a badge in a monitored channel. Mine reads that post, pulls out the box name, and spins up a dedicated channel locked to confirmed solvers only. Anyone who hasn't rooted it yet doesn't even see the channel exists, can't get spoiled, can't peek at the discussion to shortcut their way through.

End result is a server where people can actually talk openly about rooted machines without putting anyone else at risk.

Only real requirement is that HTB-Updates has to be present and posting in the same server.
