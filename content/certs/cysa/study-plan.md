+++
title = "CySA+ Study Plan"
date = "2026-07-05T00:00:00-05:00"
tags = ["cysa", "certification", "study", "comptia"]
description = "Fourteen day sprint plan for the CompTIA CySA+ (CS0-003) exam. Video blueprint first, then a hands-on tool pass, then a daily practice exam grind up to test day."
draft = false
+++

## The Fourteen Day Sprint

CySA+ leans more hands-on than Security+, expect log analysis, threat intel, and incident response scenarios instead of pure definitions. Same 3 to 4 hour daily budget, same rule about not skipping days.

---

## Phase 1: Domain Blueprint (Days 1-5)

Get the shape of the exam in your head before you touch a lab or a practice question.

### Days 1-5

- **Task:** Watch Professor Messer's full CySA+ CS0-003 course on YouTube, start to finish, at 1.25x to 1.5x speed
- **Focus:** Every domain in order, no skipping ahead to the fun parts
- **Goal:** This exam rewards pattern recognition on logs and alerts more than memorized definitions. If you're already running Wazuh or Splunk day to day, the security operations domain will feel familiar fast, slow down instead on threat intelligence and vulnerability management, that's where the exam gets specific about frameworks and scoring systems.

---

## Phase 2: Hands-On Reinforcement (Days 6-9)

CySA+ has performance-based questions that hand you a log excerpt or a scan output and ask what's actually going on. Reading about it isn't the same as doing it.

### Day 6: Diagnostic

- Take one full-length practice exam cold
- Score each domain separately
- This is a baseline, don't read anything into a bad score yet

### Days 7-9: Weak Domain Repair

- Re-watch Messer's sections for your two or three lowest domains
- Spend at least one session actually reading a CVSS score end to end and a sample SIEM alert or two, don't just memorize what the acronyms stand for
- Skip anything you already scored well on

---

## Phase 3: Practice Exam Grind (Days 10-13)

One exam in the morning, full autopsy in the afternoon. This is the phase that moves the score.

### Daily Routine, Days 10-13

- **Morning:** One new practice exam, timed, no notes
- **Afternoon:** Review every wrong answer, including the log-reading and PBQ-style questions. For those, don't just check which answer was right, walk back through the log or output yourself until you can see why

---

## Day 14: Test Day

- Light review only: CVSS scoring, common framework names (MITRE ATT&CK, NIST CSF, Cyber Kill Chain), anything you keep mixing up
- No new material
- Take the exam

---

## Resources

- **Video:** Professor Messer's CySA+ CS0-003 course on YouTube. Free, current to the exam objectives, enough on its own for Phase 1.
- **Video, paid alternative:** Jason Dion's Udemy course for a more structured format with built-in quizzes.
- **Practice exams:** Dion Training or Professor Messer's practice tests, closest match to the real exam's phrasing and PBQ style.
- **Exam objectives:** Pull the official CompTIA CS0-003 objectives PDF and use it as a checklist, same as any other CompTIA exam.

---

## Test Day Tactics

- **PBQs still go last.** Same rule as Security+, flag the hands-on simulations, clear every multiple choice question first, then come back with whatever time is left.
- **Know your frameworks cold.** MITRE ATT&CK tactics and techniques, the NIST Cybersecurity Framework functions, the Cyber Kill Chain stages. These get referenced constantly and the exam expects you to know which framework a given term belongs to.
- **Don't book the exam early.** Wait for 80-85% or better on fresh, first-attempt practice exams before scheduling.

---

## Score Tracking

| Domain | Score | Status |
|---|---|---|
| 1.0 - Security Operations | | |
| 2.0 - Vulnerability Management | | |
| 3.0 - Incident Response and Management | | |
| 4.0 - Reporting and Communication | | |

- **Above 80%:** Safe, stop drilling it.
- **Below 70%:** Back to Messer's section on that domain before the next practice exam.
