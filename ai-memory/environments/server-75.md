# Server 75

## SSH access

- Host: `192.168.130.75`
- Port: `22`
- User: `yons`
- Authentication: password `123` (user supplied 2026-07-31)
- Connection label: `75`

## Safety

- This host is likely related to `ai-human`, but its exact deployment/database role is not yet verified.
- The same IP serves the repository Git SSH endpoint on port `16003`; do not assume that is the same service as OS SSH on port `22`.
- For database recovery, perform read-only inventory and take a snapshot/copy before any mutation. Do not restore over a live database without explicit user approval.

## Related

- [[projects/ai-human]]
