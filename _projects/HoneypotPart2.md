---
report_number: "003"
layout: project
title: Project Title
description: Brief description of the project.
date: 2026-10-17
tools:
  - Tool 1
  - Tool 2
---

## Project Overview

Describe the project, its purpose, and the problem being addressed.

## Objectives

- Objective 1
- Objective 2
- Objective 3

## Tools & Technologies

- Tool 1
- Tool 2

## Investigation Process

### 1. First Step



Cowrie_CL
| where TimeGenerated >= ago(1h)
| where eventid in ("cowrie.login.failed", "cowrie.login.success")
| where message !contains "public key"
| summarize
    LoginAttempts = count(),
    FailedAttempts = countif(eventid == "cowrie.login.failed"),
    AcceptedAttempts = countif(eventid == "cowrie.login.success"),
    FirstSeen = min(TimeGenerated),
    LastSeen = max(TimeGenerated)
    by src_ip

### 2. Second Step

Document your next steps and observations.

## Findings

Explain the results and their security significance.

## Lessons Learned

Describe what you learned and any challenges you encountered.
