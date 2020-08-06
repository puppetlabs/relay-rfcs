# Tenant logging (2020-08-05)

## Stakeholders

* **Recommenders:** Noah Fontes
* **Agreers:** Rick Lane, Kyle Terry, Deepak Giridharagopal
* **Performers:** Rick Lane, Kyle Terry
* **Deciders:** Rick Lane, Kyle Terry, Deepak Giridharagopal

## Problem

Our logging infrastructure is based on a resource ownership model that implies
supervision of every pod from beginning to end (because this is how Tekton
works). However, with the introduction of Knative Serving, we no longer have
that luxury. As a result, we do not retain logs for webhook triggers.

## Summary

This change introduces a new persistent log service to store log streams from
tenant containers. It integrates with the edge API and the metadata API. This
service is part of the Relay SaaS offering.

## Motivation

From the outset of planning for our new triggers features in April, we knew the
ability to debug webhook triggers would be problematic. We scheduled some work
to investigate logging consistency but it was eventually punted from our
development plan altogether.

Now that webhook triggers are generally available, we've encountered debugging
problems in our own team, in other teams in Puppet, and externally. Regardless
of technical approach, we clearly need a solution to provide log data for
triggers.

## Product-level explanation

[Explain this change in a way that our product team can understand it and pitch
it to customers for feedback. Make heavy use of examples and ensure you
introduce any new concepts at a high level.]

## Engineering-level explanation

This RFC proposes a lightweight HTTP API built on top of Google Cloud Pub/Sub,
Google Cloud Dataflow, and Google BigQuery. This API serves as the authoritative
information service for persistent log data, a concept made popular by Apache
Kafka. A persistent log has the following properties (cf. a queue):

1. Order: Like a queue, persistent log entries are messages and those messages
   are ordered. Ordering may be determined by insertion or something more
   complicated like an embedded timestamp or sequential identifier.
1. Arbitrary seek: A reader of a persistent log does not have to start from the
   newest unacknowledged message; instead, they may choose to start from any
   message still stored.
1. Long-term storage: A message that has been processed and acknowledged is not
   necessarily deleted. Persistent logs support retention and compaction of
   messages, potentially indefinitely.
1. Streaming: Once a reader reaches the end of a persistent log, it can receive
   new messages appended to the log in real-time.

These primitives are not specific to application-level logging, but it's easy to
see how such log data from an application is a great fit for a persistent log
data structure. In our case, we want to make sure we address the following
requirements:

1. Durability: Users expect that any data they write to an application log is
   preserved as-is and not dropped once our log service receives it, even in the
   event of a system outage.
1. Real-time streaming: Users of CI/CD products have come to expect a real-time
   `tail --follow` interface. We need to expose this for both steps and
   triggers. As tenant containers execute, a user should be able to see log data
   emitted with a latency less than about 5 seconds. We have a hard maximum
   latency of 10 seconds.
1. Historical record: We want to retain logs for some time after a container
   runs, both for convenience and potentially as an auditing feature.
1. Deletion: It is, unfortunately, relatively easy to expose sensitive data in
   application log data. We should make it easy and instant for a user to delete
   log data if needed.

In addition, we consider the following potential enhancements as inspiration for
this design, although we do not currently have plans to implement them:

1. Search: We may want to enable searching log data across an account in some
   cases; for example, an administrator may not know the exact workflow run when
   an outage-causing event occurred.
1. Transformations: We may want to enable user-defined transformations or
   aggregation of log data after a step or workflow is complete. This could, for
   example, collect warnings and email them to a relevant developer.

### API

The metadata API and edge API access the persistent log service (PLS) using HTTP
or gRPC APIs.

#### HTTP Resources

* `POST /credentials`

  Create a new token.

  If this request is made using an authorized token, a child token is created.
  The child token's expiration may not exceed the expiration of the parent, and
  the child token's contexts must be a subset of the parent's. When the parent
  token is deleted, so are any children.

  If this request is not made using an authorized token, a root token is created
  with an empty persistent log space. This is the only resource that does not
  require an authentication token.

  The service may limit the possible expiration time. The generated token's
  expiration time is guaranteed to be no later than the requested time, but it
  might be earlier, depending on, for example, the service's key rotation
  schedule.

  Request:

  ```json
  {
    "contexts": [
      "workflows/9a000f7b-3511-4832-83e0-40914bb828b7/triggers/a7387f99-5b0c-4f42-ac7e-47612b5b96c9"
    ],
    "expires_at": "2021-08-05T22:40:54Z"
  }
  ```

  Response (201):

  ```json
  {
    "credential": {
      "id": "baf99b076b7c8a1b83b356fba9ac37976ecb808244dc0ca879c4c41e62bc4f37",
      "contexts": [
        "workflows/9a000f7b-3511-4832-83e0-40914bb828b7/triggers/a7387f99-5b0c-4f42-ac7e-47612b5b96c9"
      ],
      "expires_at": "2021-08-05T22:40:54Z",
      "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c"
    }
  }
  ```

  The response includes a `Location` header of `/credentials/{credentialId}`.

* `POST /credentials/{credentialId}/refresh`

  Reissue a token with a new expiration. The given credential identifier must be
  that of the authorized token or one of its children.

  Request:

  ```json
  {
    "expires_at": "2022-08-05T22:40:54Z"
  }
  ```

  Response (200):

  ```json
  {
    "credential": {
      "id": "baf99b076b7c8a1b83b356fba9ac37976ecb808244dc0ca879c4c41e62bc4f37",
      "contexts": [
        "workflows/9a000f7b-3511-4832-83e0-40914bb828b7/triggers/a7387f99-5b0c-4f42-ac7e-47612b5b96c9"
      ],
      "expires_at": "2022-08-05T22:40:54Z",
      "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c"
    }
  }
  ```

* `DELETE /credentials/{credentialId}`

  Revoke a token. The given credential identifier must be that of the authorized
  token or one of its children. Any children of the revoked credential are also
  immediately revoked.

  Response (200):

  ```json
  {
    "credential": {
      "id": "baf99b076b7c8a1b83b356fba9ac37976ecb808244dc0ca879c4c41e62bc4f37"
    }
  }
  ```

* `POST /logs`

  Create a new log stream with a given context and name. If the authentication
  token for this request only has access to one context, it is optional to
  provide it as it will be inferred from the token.

  Request:

  ```json
  {
    "context": "workflows/9a000f7b-3511-4832-83e0-40914bb828b7/triggers/a7387f99-5b0c-4f42-ac7e-47612b5b96c9",
    "name": "stdout"
  }

  Response (201):

  ```json
  {
    "log": {
      "id": "9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08",
      "context": "workflows/9a000f7b-3511-4832-83e0-40914bb828b7/triggers/a7387f99-5b0c-4f42-ac7e-47612b5b96c9",
      "name": "stdout"
    }
  }
  ```

  The response includes a `Location` header of `/logs/{logId}`. If a log with
  the given context and name combination already exists, the service returns
  409.

* `GET /logs`

  Retrieve the list of available logs.

  Query parameters:

  * `context={context}`: Restrict the list to logs in the given context.

  Response (200):

  ```json
  {
    "logs": [
      {
        "id": "9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08",
        "context": "workflows/9a000f7b-3511-4832-83e0-40914bb828b7/triggers/a7387f99-5b0c-4f42-ac7e-47612b5b96c9",
        "name": "stdout"
      }
    ]
  }

* `POST /logs/{logId}/messages`

  Append a message to the log stream. Initially, this endpoint only accepts a
  media type of `application/octet-stream` and expects data to be sent in binary
  format. Each request represents a log message, and consumers of this API are
  expected to reasonably buffer requests (for example, to newlines).

  The service responds with 202 generally or 429 if needed, and will provide a
  [`Retry-After`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Retry-After)
  header in this case. If the payload is larger than 2MB, the service responds
  with 413.

* `GET /logs/{logId}/messages`

  Retrieve the stored log data for the given stream. Messages are returned in
  order.

  Query parameters:

  * `after={messageId}`: Only return messages sequentially after the given
    message ID.

  Response (200):

  ```json
  {
    "messages": [
      {
        "payload": "Hello, world\n"
      },
      {
        "payload": "Goodbye, container\n"
      }
    ]
  }
  ```

* `DELETE /logs/{logId}`

  Delete the log and all data associated with it.

  Response (200):

  ```json
  {
    "log": {
      "id": "9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08",
      "name": "stdout"
    }
  }
  ```

#### RPCs

We define the following initial RPC protocol:

```protobuf
syntax = "proto3";

service PersistentLog {
  rpc GetLogMessages(Log) returns (stream Message) {}
}

message Log {
  string id = 1;
}

message Message {
  bytes payload = 1;
}
```

This streaming counterpart to the HTTP GET `/streams/{streamId}/messages`
resource allows a client to efficiently wait for new messages to be added to a
stream.

RPC requests are authenticated by embedding the JWT in the metadata of the RPC
request. You can use interceptors in the Go client and server to handle this in
a mostly automated way.

### Contexts

Contexts do not have any semantic meaning to the PLS. They serve as a prefix to
the name of a log stream to which we can apply fine-grained authorization. In
the context of the Relay SaaS, we'll use the following contexts:

* For triggers, `workflows/{workflowId}/triggers/{triggerSyncId}`
* For steps, `workflows/{workflowId}/runs/{runId}/steps/{stepName}`

### Storage

#### Log metadata

Log metadata is stored in a Vault cluster backed by GCS. This is the primary
storage for looking up metadata about a log. We do not need any additional
transactional storage backend for the PLS. Assuming a KV V2 mount at `/pls`, we
use the following layout:

* `/pls/data/logs/{logId}`: Information about a log, including context, name,
  and the per-log encryption key.
* `/pls/data/contexts/{context}/logs/{logId}`: A symbolic-link style secret
  pointing to `/pls/data/logs/{logId} to enable searching by context.

#### Messages

The storage backend for log messages is Google BigQuery. The table structure for
messages is:

```sql
CREATE TABLE `prod-1-logs.messages` (
  log_id STRING NOT NULL,
  seq INT64 NOT NULL,
  timestamp TIMESTAMP NOT NULL,
  encrypted_payload BYTES
)
CLUSTER BY (log_id);
```

Note that encryption keys are not stored in BigQuery.

### Encryption

BigQuery's unique data loading model makes deleting data very difficult in
[certain](https://cloud.google.com/blog/products/gcp/life-of-a-bigquery-streaming-insert)
[circumstances](https://cloud.google.com/bigquery/streaming-data-into-bigquery#dataavailability).
We want to let users delete logs, but we also don't want to have to worry about
making sure we clean up our storage instantaneously. So we can encrypt each log
message using a per-log encryption key and then simply delete that key when we
want to scrub the logs. A reaper process could come along later and remove the
physical data from the table.

Even more conveniently for us, [we can do decryption entirely within BigQuery
itself](https://cloud.google.com/bigquery/docs/reference/standard-sql/aead_encryption_functions) as long as we use AES in GCM mode:

```sql
WITH keys AS (
  SELECT
    '9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08' as log_id, -- Example!
    keys.add_key_from_raw_bytes(
      keys.new_keyset('AEAD_AES_GCM_256'),
      'AES_GCM',
      b'0123456789012345' -- Example!
    ) as keyset
)
SELECT
  msgs.log_id,
  msgs.seq,
  msgs.timestamp,
  aead.decrypt_bytes(keys.keyset, msgs.encrypted_payload, msgs.log_id) AS payload
FROM `prod-1-logs.messages` msgs
JOIN keys ON (keys.log_id = msgs.log_id);
```

### Streaming

Our streaming infrastructure consists of two Cloud Pub/Sub topics, one Cloud
Dataflow streaming job with a subscription, and one Cloud Pub/Sub subscription
per running instance of the PLS (i.e., per pod):

1. When a request is received by the API to append a message to the log stream,
   it is written to a `pending` Cloud Pub/Sub topic.
1. The Cloud Dataflow streaming job subscribes to the `pending` topic and
   appends messages to the BigQuery table and to a second Cloud Pub/Sub topic,
   `processed`, in order. To guarantee consistency, only appends acknowledged by
   BigQuery are written to the `processed` topic.
1. Each PLS instance subscribes to the `processed` topic to filter messages
   while streaming reads to clients.

A good place to start for the Cloud Dataflow job might be the [Pub/Sub to
Pub/Sub](https://cloud.google.com/dataflow/docs/guides/templates/provided-streaming#cloudpubsubtocloudpubsub)
and [Pub/Sub to
BigQuery](https://cloud.google.com/dataflow/docs/guides/templates/provided-streaming#cloudpubsubsubscriptiontobigquery)
templates, both officially condoned by Google.

#### Reading messages

In a non-streaming call, the PLS simply queries BigQuery to select all messages
currently stored and returns them to the client.

In a streaming call, the PLS performs the following operations in order:

1. Adds a filter to the Cloud Pub/Sub subscription and begins to buffer messages
1. Issues a query to BigQuery to select all messages
1. Immediately returns the initial BigQuery result set to the client
1. Skips any buffered messages from the subscription that are duplicated from
   the BigQuery result by comparing sequence numbers
1. Polls BigQuery for more data if the next message on the subscription
   indicates a gap in sequence numbers (likely a data loading delay)
1. Once the subscription is reconciled, immediately forwards new messages from
   the subscription to the client, decrypting the payload as needed

### Metadata API

We propose to add support to the metadata API to forward logs to the PLS.

#### HTTP resources

We add the following HTTP resources to the metadata API, which are authenticated
using the usual encrypted JWT from the requesting pod:

* `POST /logs`

  Create a new log stream.

  Request:

  ```json
  {
    "name": "stdout",
  }
  ```

  Response (201):

  ```json
  {
    "id": "9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08",
    "name": "stdout"
  }
  ```

  If the stream with the given name already exists, the server responds with
  409. If the metadata API is not configured with access to a PLS, the server
  responds with 507 (Insufficient Storage).

* `POST /logs/{logId}/messages`

  Append a message to the log stream. The format of the payload is the same as
  the PLS and the request is forwarded to the PLS.

  The service responds with 202 or 429.

#### Authenticating to the PLS

We add a new claim, `relay.sh/log/token` containing a JWT issued by the PLS, to
the encrypted JWT stored on the pod.

### Entrypoint

We amend our entrypointer process to create two log streams in the metadata API,
`stdout` and `stderr`, prior to executing the delegate entrypoint. The
delegate's `Stdout` and `Stderr` will be redirected to Goroutines that write to
these log streams.

### Edge API

We modify the API to stream logs directly from the PLS. This removes one
long-standing tech debt item that allows the API to directly access Kubernetes
pod logs.

We add the following resource:

* `GET /api/workflows/{workflowName}/triggers/{triggerSyncId}/logs`

  Retrieve the log for a trigger with the given synchronization ID. This
  functionally behaves similarly to the step logs resource, but a trigger is
  always considered to be "in progress," so trigger logs can always be followed.

We also modify the `WorkflowTriggerState` schema to include the an `id` field
containing the synchronization ID.

#### Removal of byte range faceting

The current step log endpoint supports faceting by byte ranges using the HTTP
`Range` header. We have not yet found practical uses for this filter. It is also
not used anywhere in our ecosystem. We propose to remove it outright considering
the difficulty of implementation with structured log messages.

#### Best practices for log multiplexing

Our edge API implementation of logging provides log data as a single byte
stream. Therefore, it must multiplex all log streams for a given context with
multiple RPC calls. In the case of following logs, the API client should also
check for the creation of new streams periodically.

## Drawbacks

[Why should we _not_ include this change in the product?]

## Rationale and alternatives

### Why is this design the best in the space of possible designs?

[...]

### What other designs have been considered and what is the rationale for not choosing them?

[...]

### What is the impact of not doing this?

[...]

### What specific risks are associated with this design?

[...]

## Success criteria

[What will we observe when this change is completely implemented? What metrics
can we use to measure success? Can we integrate any learnings into future
processes?]

## Unresolved questions

* [Insert a list of unresolved questions and your strategy for addressing them
  before the change is complete. Include potential impact to the scope of work.]

## Future possibilities

[Does this change unlock other desired functionality? Will it be highly
extensible, very constrained, or somewhere in between?]
