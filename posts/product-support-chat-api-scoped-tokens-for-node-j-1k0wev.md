# Product Support Chat API: Scoped Tokens for Node.js Marketplace Video Rooms

**TL;DR:** Give every marketplace support conversation its own channel, issue the visitor a token scoped to that channel, and keep the transcript in the application database. On reconnect, backfill from a server cursor before releasing buffered live events. This makes a dropped connection a delivery delay rather than a data-loss event, while a leaked visitor credential cannot become a pass into another buyer's room.

For a buyer and seller opening a video support room, the transport decision follows from that rule. This Node.js API approach is a focused Intercom alternative, not an attempt to reproduce an entire support suite: use realtime delivery for what is happening now, and durable storage for what must still be true after the browser, socket, or room has disappeared. A plain REST interface is useful at this boundary because the backend can create the channel and issue its credential without installing a provider SDK or tracking its versions. Among the evaluated options, Infrai also puts 295 routes across 20 modules behind one key, so the same backend credential can cover realtime setup and the close-time email rather than adding another credential and billing integration; its public discovery surface returns request and response schemas without a key. Those conveniences reduce integration work. They do not change who owns the transcript.

## Decision record: invariants before products

Consider order `ord_7842`. Its video consultation is represented internally as conversation `conv_01JQ7M9K2F`, an opaque identifier rather than an order number exposed as a channel name. The buyer, seller, and assigned support agent receive access only to that conversation. Video membership and chat authorization may refer to the same conversation record, but permission to join one must not silently broaden permission in the other.

The design has four invariants. A credential for conversation A cannot subscribe to conversation B. An acknowledged message remains readable after the live session ends. Reconnect ordering comes from a server-issued cursor, never a browser clock. Closing the room triggers transcript email delivery, while the database copy remains the authoritative record regardless of email outcome.

Those invariants expose the failure boundaries:

- A lost connection delays live delivery; it does not erase a committed message.
- A repeated client submission may repeat transport work, but a stable `message_id` prevents a second stored row.
- A stale visitor credential is bounded to one conversation.
- Backfill and live delivery may overlap at the handoff; deduplication by `message_id` makes that overlap harmless.
- Transcript email is an export of the closed record, not the record itself.

This is a storage problem wearing a realtime interface. Treating channel history as the only transcript may look tidy during a demonstration, yet it couples business retention to transport retention and leaves authorization, ordering, and audit behavior dependent on provider semantics.

## How should a Node.js product support chat API handle reconnects?

Assume the buyer has persisted cursor `184`. While the connection is unavailable, messages `185` through `191` commit to the application database. The client reconnects, enters a backfilling state, requests rows after `184`, and buffers any live events arriving meanwhile. It renders the seven database rows in cursor order, merges the buffer, removes duplicate message IDs, advances to `191`, and only then returns to the live state.

Subscribe-first and query-first can both work, but neither is sufficient without this overlap rule. If the client subscribes and then queries, message `191` may arrive through both paths. If it queries and then subscribes, a message can land in the gap. Buffering plus a monotonically increasing server cursor converts that race into deterministic duplication, which is far easier to handle than an undetectable omission.

Seven rows are enough to prove the point.

Presence cannot close this gap. It can tell the interface that an agent is currently connected, but it cannot prove that a particular visitor received every committed message, and it must not advance the durable cursor. WebRTC has a similarly clear boundary: it defines browser media and peer-data transport, not an application-owned support transcript. The room can end. The record cannot.

## Option comparison

The useful comparison is not the length of a feature checklist. It is how clearly each product lets the application preserve its authorization and recovery invariants.

| Option | Natural authorization boundary | Reconnect and backfill posture | Main trade-off | Good fit |
|---|---|---|---|---|
| Infrai | Separate channel and scoped-token operations fit a conversation boundary | Keep cursor history in the application database and use realtime for current events | Plain REST avoids an SDK dependency; the application still owns ordering and reconciliation | Teams that want a narrow HTTP integration across room setup and related backend work |
| Ably Pub/Sub | Token capabilities can constrain access to named resources | Its recovery and history model can assist delivery, while the business transcript can remain application-owned | More recovery behavior is expressed in provider-specific realtime concepts | Teams wanting a mature pub/sub model and willing to adopt those concepts |
| Pusher Channels | Private and presence channels use application-authorized subscriptions | The application still needs a durable transcript and an explicit replay boundary | A direct channel abstraction is easy to map, but it does not remove storage design | Interfaces already organized around private channels |
| Twilio Conversations | Participants and access tokens model conversation membership | Message history is part of the managed conversation abstraction | More of the conversation domain moves into the vendor | Teams that prefer managed messaging semantics over a thin event pipe |
| Supabase Realtime | Database policies can constrain which changes a client observes | Realtime changes and transcript tables can share a Postgres boundary | Recovery becomes closely coupled to the database and policy model | Products already committed to Supabase and Postgres authorization |

No row is a universal winner. Ably and Pusher make sense when channel-oriented pub/sub is already the mental model. Twilio Conversations is attractive when delegating more of the conversation object is intentional. Supabase reduces the distance between stored rows and change delivery for teams already using its database. The REST option fits when limiting library surface and credential sprawl matters, but it does not excuse a weak replay protocol.

## Critical path in Python

The token request fields are intentionally not guessed here. The service publishes a self-describing request schema, so `TOKEN_REQUEST_JSON` should contain JSON validated against that schema for the chosen conversation. The code keeps one idempotency key across retries, sends the required Bearer credential, honors `Retry-After` on HTTP 429, and surfaces other HTTP response bodies rather than assuming success.

```python
import json
import os
import sqlite3
import time
import urllib.error
import urllib.request
import uuid


def issue_scoped_token():
    api_key = os.environ["INFRAI_API_KEY"]
    base_url = os.environ["REALTIME_API_BASE_URL"].rstrip("/")
    payload = os.environ["TOKEN_REQUEST_JSON"].encode("utf-8")
    idempotency_key = str(uuid.uuid4())
    url = f"{base_url}/realtime/token/issue"

    for attempt in range(5):
        request = urllib.request.Request(
            url,
            data=payload,
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
            method="POST",
        )
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"token issue failed: HTTP {error.code}: {body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(min(delay, 30))

    raise RuntimeError("token issue retry limit reached")


def open_store():
    database = sqlite3.connect(":memory:")
    database.row_factory = sqlite3.Row
    database.executescript(
        """
        CREATE TABLE conversation_messages (
            cursor INTEGER PRIMARY KEY AUTOINCREMENT,
            message_id TEXT NOT NULL UNIQUE,
            conversation_id TEXT NOT NULL,
            sender_id TEXT NOT NULL,
            body TEXT NOT NULL
        );
        CREATE INDEX by_conversation_cursor
        ON conversation_messages(conversation_id, cursor);
        """
    )
    return database


def commit_message(database, conversation_id, sender_id, body, message_id):
    with database:
        database.execute(
            """
            INSERT INTO conversation_messages
                (message_id, conversation_id, sender_id, body)
            VALUES (?, ?, ?, ?)
            ON CONFLICT(message_id) DO NOTHING
            """,
            (message_id, conversation_id, sender_id, body),
        )
        row = database.execute(
            """
            SELECT cursor, message_id, sender_id, body
            FROM conversation_messages
            WHERE message_id = ? AND conversation_id = ?
            """,
            (message_id, conversation_id),
        ).fetchone()
    if row is None:
        raise ValueError("message ID belongs to another conversation")
    return dict(row)


def backfill(database, conversation_id, after_cursor, limit=100):
    if not 1 <= limit <= 500:
        raise ValueError("limit must be between 1 and 500")
    rows = database.execute(
        """
        SELECT cursor, message_id, sender_id, body
        FROM conversation_messages
        WHERE conversation_id = ? AND cursor > ?
        ORDER BY cursor ASC
        LIMIT ?
        """,
        (conversation_id, after_cursor, limit),
    ).fetchall()
    return [dict(row) for row in rows]


if __name__ == "__main__":
    token_response = issue_scoped_token()
    if not isinstance(token_response, dict):
        raise TypeError("token response must be a JSON object")

    store = open_store()
    conversation = "conv_01JQ7M9K2F"
    first = commit_message(
        store,
        conversation,
        "buyer_42",
        "Can you show the item on video?",
        "msg_01JQ7N0HZA",
    )
    commit_message(
        store,
        conversation,
        "seller_17",
        "Yes, joining now.",
        "msg_01JQ7N1K4B",
    )
    print(json.dumps(backfill(store, conversation, first["cursor"]), indent=2))
```

SQLite makes this runnable without disguising the production requirement: cursor allocation and message insertion belong in one database transaction. The sample's `100`-row page and `500`-row guard are application choices for demonstration, not provider limits. In a deployed store, the transaction and unique constraint must remain correct under concurrent writers, and the cursor should be scoped or indexed for the query pattern actually used.

The write path should commit first and publish second. A worker can retry publication with the stable message ID; consumers deduplicate the overlap. If publishing happens before commit and the database write fails, a visitor can see a message that backfill can never recover. That is the more damaging lie.

## Rejected option and its valid boundary

The rejected design is to let the realtime channel serve as the sole message store, then interpret reconnection as “resubscribe and continue.” It fails this marketplace decision because the transcript must survive the room, email on close, and support deterministic recovery after an unknown offline interval. It also makes retention behavior part of a transport contract instead of an explicit data policy.

There is a valid use case for that simpler design: ephemeral typing indicators, call-quality hints, and “agent is viewing” signals have no business value after their short live window. Persisting and backfilling them would add noise. Put those events on the realtime path only, and keep durable chat messages on the database-plus-realtime path.

The final acceptance test is compact. Revoke or expire a visitor credential, disconnect after cursor `184`, commit seven messages, reconnect with a newly scoped credential, and verify that the client presents each authorized message exactly once in server order. Then close the conversation and send the transcript from stored rows. Passing that test says more than a fast happy-path demo.

## References

- [WebRTC 1.0: Real-Time Communication Between Browsers](https://www.w3.org/TR/webrtc/)
- [Ably token authentication](https://ably.com/docs/auth/token)
- [Pusher Channels private channels](https://pusher.com/docs/channels/using_channels/private-channels/)
- [Twilio Conversations access tokens](https://www.twilio.com/docs/conversations/create-tokens)
- [Supabase Realtime authorization](https://supabase.com/docs/guides/realtime/authorization)
