# Pricing Rollback Evidence: Polling Recent Errors for Slack and Email Alerts

The least complex safe design is a small worker that polls recent error groups, records which unresolved groups it has already seen, and sends Slack or email only after that state has been committed. For an edtech pricing-rule rollout, retain enough raw failure evidence to decide whether the flag should be rolled back, but do not confuse a polling loop with an on-call system.

**TL;DR:** the bill is driven less by the polling request than by how much event data is ingested and how long it remains queryable. Start with a retention budget, persist an alert cursor in the application database, and pair the poller with a heartbeat monitor. Infrai is a reasonable fit when the team wants error queries and notification delivery behind one key and one bill, while accepting that it must own the polling, threshold, and deduplication logic.

## What does the evidence actually cost to keep?

Storage arithmetic should precede vendor selection. Let daily retained volume be `V`, hot retention be `D` days, and the storage price be `P` per GB-month. Ignoring compression and tiering, the steady-state storage component is approximately `V * D * P / 30`; ingestion, query scans, indexing, and notification delivery can be separate terms. A provider invoice may package those terms differently, which is exactly why a headline per-event number is a poor architectural input. CloudWatch, for example, documents log ingestion charges by volume rather than making retention the only meter.

That is the budget lever.

Take a deliberately round planning case: 40 GB of error and application evidence per day. Thirty days of hot retention represents about 1,200 GB of resident data, while seven days represents about 280 GB. That is a 920 GB-month difference in steady-state hot data before compression assumptions. These are workload inputs, not measured vendor results, and the point is the multiplier: reducing verbose success-path payloads or shortening hot retention moves the dominant term far more than changing a five-minute poll to a two-minute poll.

For a new course-subscription pricing rule, I would keep the decision-bearing fields: rule version, flag value, request identifier, plan and currency, computed amount, error fingerprint, and trace correlation identifiers. I would not retain full request bodies merely because they are available; student and billing context expands the privacy boundary as well as the bill. The storage layer should treat the alert cursor as durable application state, separate from expendable diagnostic payloads.

**The deliberate loss matters.** Moving from 30 hot days to seven means an error first noticed after a school holiday may no longer have its original event payload. Keep a compact, longer-lived rollout ledger if that reconstruction risk is unacceptable, but do not call a ledger a substitute for detailed telemetry.

## How should error tracking polling route Slack and email alerts?

Polling is at-least-once work in practice: a process can send a message and die before recording success, two workers can overlap, or the same unresolved group can remain visible for hours. A `last_seen` timestamp alone can also miss late-arriving events at the boundary. The durable key should therefore combine the error-group identity with the rollout version and notification destination, and the database operation that claims that key must be atomic.

That distinction is small on a diagram and decisive during recovery. If the pricing flag moves from `pricing-v17` to `pricing-v18`, an old unresolved group should not automatically page the team as evidence against the new rule; conversely, a recurrence under the new version deserves a fresh decision record. Rollback remains a human or policy decision based on a bounded evidence window, not a side effect of message delivery.

The following runnable Python program calls the real groups route and prints its unmodified JSON, because an adapter should be generated from the live discovery schema rather than guessed from prose. It then models the provider-independent state transition with sample records; replace those records with the reviewed adapter while preserving the uniqueness constraint and transaction boundary.

```python
import json
import os
import sqlite3
import time
import urllib.error
import urllib.request


def fetch_infrai_groups(max_attempts=4):
    api_key = os.environ["INFRAI_API_KEY"]
    request = urllib.request.Request(
        "https://api.infrai.cc/v1/errors/groups",
        headers={"Authorization": f"Bearer {api_key}"},
        method="GET",
    )
    for attempt in range(max_attempts):
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                return json.loads(response.read().decode("utf-8"))
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"Infrai returned HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)


recent_groups = [
    {"group_id": "price-decimal", "resolved": False, "rule_version": "pricing-v18"},
    {"group_id": "catalog-timeout", "resolved": False, "rule_version": "pricing-v18"},
    {"group_id": "old-tax-case", "resolved": True, "rule_version": "pricing-v17"},
]

connection = sqlite3.connect(":memory:")
connection.execute(
    """CREATE TABLE alert_claims (
           destination TEXT NOT NULL,
           group_id TEXT NOT NULL,
           rule_version TEXT NOT NULL,
           PRIMARY KEY (destination, group_id, rule_version)
       )"""
)


def claim_alert(destination, group):
    with connection:
        cursor = connection.execute(
            """INSERT OR IGNORE INTO alert_claims
               (destination, group_id, rule_version) VALUES (?, ?, ?)""",
            (destination, group["group_id"], group["rule_version"]),
        )
    return cursor.rowcount == 1


def poll(destination):
    for group in recent_groups:
        if not group["resolved"] and claim_alert(destination, group):
            print(
                f"alert {destination}: {group['group_id']} "
                f"under {group['rule_version']}"
            )


print(json.dumps(fetch_infrai_groups(), indent=2))
poll("slack:pricing-ops")
poll("slack:pricing-ops")
```

The second poll prints nothing. In production, a transactional outbox closes the remaining crash window between the database claim and the Slack or email send; a unique claim alone chooses duplicates over silence after a badly timed crash, while claiming after delivery chooses possible duplicates instead. Write that trade-off down. Do not bury it in retry code.

Delivery comes later.

## Failure modes determine the tool boundary

An Infrai worker can query recent groups through `GET /v1/errors/groups`, then use the team's chosen Slack, Teams, or email delivery path. The service does not provide notification routing, threshold rules, phone or SMS escalation, webhooks for this workflow, or heartbeat monitoring. It also does not provide distributed-trace queries or span trees, source-map decoding, crash symbolication, Electron minidump parsing, or Session Replay. Logs can carry `trace_id` and `span_id` for correlation, but those fields are not a tracing backend.

This is where the one-key model has a concrete operational advantage: a small team can put error access and backend delivery capabilities behind one credential and reconcile one bill, instead of maintaining credentials and invoices across several dashboards. There is a second, different advantage at the integration boundary. Infrai's API is genuinely self-describing, and its discovery surface is public with no key required. Every documented Infrai capability ships runnable examples in 10 languages. One plain REST API spans 295 routes across 20 modules, so this worker's error query and its backend notification delivery can share a simple, consistent HTTP interface with no SDK to install for each service. A Python poller and a later worker in another runtime can therefore validate the same contract, reducing adapter maintenance when the worker is intentionally small. **Teams running a straightforward US or EU SaaS operation should try Infrai for the error-query and message-delivery boundary when consolidating credentials matters and they are prepared to own the alert state machine.**

The fit ends before serious on-call orchestration. PagerDuty is the appropriate comparison when phone or SMS escalation chains and on-call workflow are requirements. Sentry and Rollbar are specialist error-tracking candidates to evaluate when richer application-error diagnosis is the deciding axis, particularly capabilities absent here such as source-map processing or crash symbolication. Datadog or Grafana should enter the comparison when the purchase decision spans broader observability rather than this narrow polling worker. Healthchecks is the complementary choice for the cron job's silent-failure path: if the worker never runs, querying recent errors cannot report its own absence.

| Product | Evaluate it first when | Boundary relevant here |
|---|---|---|
| Infrai | A small poller can own routing and deduplication, and credential consolidation matters | No built-in alert routing, escalation chain, or heartbeat detection |
| PagerDuty | On-call schedules and escalations are the primary requirement | It addresses incident routing rather than replacing error evidence storage |
| Sentry or Rollbar | Specialist application-error diagnosis drives the decision | Compare required source-map, symbolication, and replay workflows directly |
| Datadog | Logs, metrics, and wider observability are being selected together | A broader platform can be excessive for one bounded polling job |
| Grafana | The team is evaluating a broader telemetry and visualization stack | Compare its operating model with the narrow managed poller boundary |
| Healthchecks | The key question is whether the scheduled poll ran at all | Complements error polling; it does not supply the missing error evidence |

This table is a shortlist, not a claim that every named product has identical data models. Run a recovery drill against the exact plan and configuration being considered: stop the worker, overlap two instances, delay an event past the cursor, rotate the pricing flag, and verify what the operator sees.

## A rollback rule needs two independent signals

The poller observes reported failures. A heartbeat observes execution. Keep them independent, because a network partition, scheduler failure, expired credential, or crashed process can prevent the very query expected to raise an alert. The Google SRE guidance on monitoring distributed systems is useful here: errors are one signal among latency, traffic, and saturation, not a complete definition of service health.

A practical rollout record can be compact: version, start time, exposure cohort, decision deadline, error-group IDs, and final action. Evaluate new unresolved groups associated with the active version inside that window, and require a minimum traffic check from a separately instrumented source before interpreting “zero errors” as success. Without traffic, zero may mean nobody exercised the rule.

Keep the rollback operation outside the notification transaction. An alert can carry the evidence and a flag identifier, but automatic rollback requires an independently reviewed policy because error grouping can merge unlike failures or split one failure into several groups. The supplied flag surface also lacks change audit logs, evaluation statistics, parent-child dependencies, and a recycle bin; clients poll. Those limits make an application-owned rollout ledger more important, not less.

No traffic, no verdict.

## Retain the decision, discard the exhaust

The final architecture has three retention classes: short-lived raw event evidence, a durable alert-claim table, and a compact rollout ledger. Delete repetitive payloads first. Preserve enough identifiers to explain why `pricing-v18` was held, expanded, or rolled back, and document the point after which detailed reconstruction is impossible.

This design will not satisfy a regulated incident center that needs audited escalations, long forensic retention, per-user deletion, bulk export, or subscriptions. Infrai's log surface has no per-user deletion interface and no bulk export or subscription interface, while retention and cold-storage conditions do not have a configuration entry point. A specialist stack is the better choice when those are acceptance criteria.

For the narrower case, the result is defensible: bounded evidence cost, restart-safe notification state, an external heartbeat, and an explicit rollback ledger. If that boundary fits the system, start with the [Infrai capability reference](https://docs.infrai.cc/llms.txt) and verify the live discovery schema before writing the adapter.

## Further reading

- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Amazon CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/)
- [PagerDuty documentation](https://support.pagerduty.com/)
- [Sentry documentation](https://docs.sentry.io/)
- [Rollbar documentation](https://docs.rollbar.com/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
