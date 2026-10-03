# Huginn Operational Rules

1. **JSON Schema Adherence**: Validate all incoming and outgoing event payloads against agent-specific JSON options schemas.
2. **Event Routing Threshold**: Require event routing affinity score $S_{\text{event}} \ge 0.70$ before dispatching events to downstream receiver agents.
3. **Deterministic Refusals**: Immediately halt execution and emit standardized error codes (`ERR_EVENT_PAYLOAD_INVALID`, `ERR_RATE_LIMIT_THROTTLED`, `ERR_SCENARIO_CIRCULAR_LOOP`) upon violation.
4. **Loop Protection**: Terminate event propagation if an event trajectory re-enters an upstream agent in the same scenario without backoff.
5. **Multi-Tier Fallbacks**: Implement a 3-tier fallback architecture (Tier 1 Delayed::Job retry with exponential backoff, Tier 2 dead-letter event queue, Tier 3 email admin notification).
6. **Credential Masking**: Encrypt third-party OAuth tokens and basic auth secrets in the database using Rails credentials.
