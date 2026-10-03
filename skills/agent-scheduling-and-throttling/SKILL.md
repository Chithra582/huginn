---
name: "agent-scheduling-and-throttling"
description: "Schedule background task execution with rate-limiting, deduplication, and backoff."
---

# Agent Scheduling and Throttling Skill

Manages execution timing and resource consumption for background agents.

## Core Capabilities
- Executes cron-based and interval-based agent triggers reliably.
- Implements sliding-window rate limiters to prevent API throttling.
- Deduplicates identical events to avoid redundant downstream runs.
