# AGENT_REVOKE examples

`AGENT_REVOKE` terminates some or all locally stored trust relationship state for a remote agent.

Scopes:

- `trust`: remove local Trust Settings for the sender's `agent_id` / npub.
- `agent_card`: remove the sender's cached remote Agent Card and require a future handshake.
- `all`: remove both Trust Settings and the cached remote Agent Card.

