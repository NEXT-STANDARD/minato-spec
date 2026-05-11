# AGENT_LOG examples

`AGENT_LOG` is a post-hoc audit notification emitted after an agent takes an autonomous action
in `auto` or `full_auto` trust mode.

Receivers should:

- verify the envelope signature before trusting the log;
- deduplicate by `(from, log_id)`;
- treat the log as informational, not as an approval request.
