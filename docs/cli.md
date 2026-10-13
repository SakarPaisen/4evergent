# 4evergent Operator CLI

`4evergent` is a command-line operator client for the 4evergent HTTP API. It is
an operator tool only — it does NOT sign transactions, build Stellar operations,
or access Horizon directly. All transaction work is delegated to the API server.

## Install / Build

```bash
# From repository root
pnpm install
pnpm --filter @4evergent/cli build

# Run via npx (from repo root)
npx tsx apps/cli/src/cli.ts health
```

## Configuration

| Variable | Default | Description |
|---|---|---|
| `FOREGENT_API_URL` | `http://127.0.0.1:3000` | API base URL |
| `FOREGENT_API_KEY` | - | Bearer API key (production auth mode) |

```bash
export FOREGENT_API_URL=http://localhost:3000
export FOREGENT_API_KEY=your-api-key-here
```

## Commands

```
4evergent health                           Check API health
4evergent help | --help | -h               Show the command list (top-level only)
4evergent agent list                        List your agents
4evergent agent get <id>                    Show agent details
4evergent agent create <name> [--description <desc>] [--capabilities a,b] [--stellar-address G...]
                                            Create an agent (all flags optional)
4evergent agent pause <id>                  Pause agent (rejects new intents)
4evergent agent resume <id>                 Resume agent (active)
4evergent agent disable <id>                Disable agent
4evergent approval list                     List pending approvals
4evergent approval approve <id>             Approve an intent
4evergent approval reject <id>              Reject an intent
4evergent execution list                    List executions across all agents
4evergent execution get <id>                Show execution detail
4evergent execution retry <id>              Retry a failed or dead-letter execution
4evergent execution cancel <id>             Cancel a queued or executing execution
4evergent policy get <agent-id>             Show effective policy rules
4evergent policy update <agent-id> --rules '<json>'
                                            Replace policy rules (--rules required)
4evergent activity list <agent-id> [limit]  List agent activity (limit 1-100, default 50)
4evergent schedule list <agent-id> [limit]  List agent schedules (limit 1-100, default 50)
4evergent schedule get <agent-id> <sch-id>  Show schedule detail
4evergent schedule create <agent-id> <cron-expression> <intent-type> [intent-args...] [--timezone <tz>]
                                            Create a schedule (see Schedules below)
4evergent schedule update <agent-id> <sch-id> <expression>
                                            Update a schedule's cron expression
4evergent schedule pause <agent-id> <sch-id>
                                            Pause a schedule (stops runs until resumed)
4evergent schedule resume <agent-id> <sch-id>
                                            Resume a paused schedule
4evergent schedule disable <agent-id> <sch-id>
                                            Disable a schedule
4evergent schedule delete <agent-id> <sch-id>
                                            Delete a schedule
4evergent intent submit <agent-id> <type> [args...] [--idempotency-key <key>]
                                            Submit an intent (see Intent types below)
```

## Intent submission

Supported intent types (accepted by the transaction pipeline):

```
payment       <asset> <amount> <destination> <reason> [memo]
trustline     <assetCode> <issuer> <reason> [limit]
contract_call <contractId> <function> <args-json-array> <reason>
```

`account_settings` is NOT supported by the transaction pipeline and is
rejected by the CLI.

The server decides the outcome: policy `allow` executes directly,
`requires_approval` returns an approvalId (approve it later with
`4evergent approval approve`), `deny` is rejected with 403.

### Idempotency

Use `--idempotency-key <key>` to make submissions safe against network
timeouts: the same key + same intent returns the original result instead
of creating a duplicate (HTTP 200), and the same key with a different
intent is rejected with 409.

If the CLI reports a network failure, the request outcome is UNKNOWN —
the request may have reached the server. Retry manually with the SAME
`--idempotency-key`. The CLI never retries automatically and never
submits twice silently.

## Help

`4evergent help`, `4evergent --help`, and `4evergent -h` print the command
list and exit `0`. Help is only recognized as the top-level command —
subcommands do not take `--help` (e.g. `4evergent agent --help` is rejected
as an unknown subcommand). Running `4evergent` with no arguments prints a
usage summary and exits `1`.

## Agent creation

`agent create <name>` takes the display name as the first positional
argument. All flags are optional and may appear in any order after it:

- `--description <desc>` — agent description (default: empty)
- `--capabilities a,b` — comma-separated capability list (default: none)
- `--stellar-address G...` — Stellar account address (default: unset)

## Policy update

`policy update <agent-id> --rules '<json>'` replaces the agent's policy
rules. `--rules` is required and must be valid JSON; the API validates the
object and fills in defaults for omitted fields. Rule fields: `maxTxAmount`,
`dailySpendingLimit`, `allowedAssets`, `allowedDestinations`,
`allowedContractIds`, `txTypeRestrictions`, `approvalThreshold`,
`requireHumanApprovalForAmountAbove`. Invalid rules are rejected with 400.

## Schedules

A schedule runs one intent on a 5-field cron expression — minute (0-59),
hour (0-23), day-of-month (1-31), month (1-12), day-of-week (0-6) —
validated by the API.

- `schedule create <agent-id> <cron-expression> <intent-type> [intent-args...] [--timezone <tz>]`
  takes the intent in the same argument order as `intent submit`. The API
  currently accepts only `payment` intents for schedules — `trustline` and
  `contract_call` schedule intents are rejected. `--timezone` defaults to
  `UTC`; the API accepts `UTC`, `America/New_York`, `Europe/London`, and
  `Asia/Tokyo`.
- `schedule update` replaces the cron expression only; the intent and the
  timezone are unchanged.
- `schedule pause` stops runs until `schedule resume` (paused and disabled
  schedules are not executed).
- `schedule delete` removes the schedule permanently.

## Examples

```bash
4evergent health
4evergent help
4evergent agent list
4evergent agent create "Payout Bot" --description "weekly payouts" --capabilities payments,reporting --stellar-address GDAAAA...
4evergent agent pause agent-123
4evergent approval list
4evergent approval approve ap-123
4evergent execution list
4evergent execution get exec-123
4evergent execution retry exec-123
4evergent execution cancel exec-123
4evergent policy get agent-123
4evergent policy update agent-123 --rules '{"maxTxAmount":{"XLM":"500"}}'
4evergent activity list agent-123
4evergent activity list agent-123 10
4evergent schedule list agent-123
4evergent schedule get agent-123 sch-123
4evergent schedule create agent-123 "0 9 * * 1" payment XLM 25 GDAAAA... "weekly payout"
4evergent schedule create agent-123 "0 9 * * 1" payment XLM 25 GDAAAA... "weekly payout" --timezone Asia/Tokyo
4evergent schedule update agent-123 sch-123 "0 10 * * 1"
4evergent schedule pause agent-123 sch-123
4evergent schedule resume agent-123 sch-123
4evergent schedule disable agent-123 sch-123
4evergent schedule delete agent-123 sch-123
4evergent intent submit agent-123 payment XLM 25 GDAAAA... "weekly payout"
4evergent intent submit agent-123 payment XLM 25 GDAAAA... "weekly payout" --idempotency-key weekly-42
4evergent intent submit agent-123 trustline USDC GDAAAA... "add USDC trustline" 1000
4evergent intent submit agent-123 contract_call CABCD... transfer '["arg1","arg2"]' "transfer funds"
```

## Output

Default output is human-readable. Every command exits `0` on success,
non-zero on failure (network error, HTTP 4xx/5xx). The `--json` flag is
not yet supported — planned for a future CLI phase.

## Security

- **Never stores credentials.** API key is read from the environment on each run.
- **Never prints secrets.** API keys, XDR blobs, and private keys never appear in output.
- **No direct Stellar access.** The CLI only makes HTTP calls to the configured API.
- **No mainnet support.** The API itself rejects non-testnet Horizon URLs.
- The API key is sent as `Authorization: Bearer <key>` on every request when set.
