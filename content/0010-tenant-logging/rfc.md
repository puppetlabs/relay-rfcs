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

This RFC defines a consistent internal API for handling customer-driven log
messages. In the near-term, this gives webhook trigger logs parity with step
logs in our UIs. It is also the first step toward the following features:

1. Structured logging: We anticipate extending the internal API to support
   JSON-formatted metadata, which means that we can enhance log data with
   additional information like severity or resource identifiers.
1. Custom logging: Each container will have a unique set of log streams
   associated with it, initially corresponding to standard output and error (as
   in POSIX environments). We anticipate adding support for image authors to
   create and write to their own streams and to be able to work with them in the
   UI.
1. Additional container logging: Because this system does not rely on the
   Kubernetes pod-oriented logging and annotations on Tekton objects, we can
   more easily enable new customer-facing features that don't use Tekton (like
   queries, potentially) with the same quality as our Tekton-driven workflow
   offering.

## Engineering-level explanation

This RFC proposes a lightweight gRPC API built on top of Google Cloud Pub/Sub,
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

The metadata API and edge API access the persistent log service (PLS) using gRPC
APIs. We define the following initial RPC protocol:

```protobuf
syntax = "proto3";

option go_package = "github.com/puppetlabs/relay-pls/pkg/plspb";

package plspb;

import "google/protobuf/timestamp.proto";

service Credential {
  // Issue creates a new token.
  //
  // If this request is authorized, a child token is created. The child token's
  // expiration may not exceed the expiration of the parent, and the child
  // token's contexts must be a subset of the parent's. When the parent token
  // is deleted, so are any children.
  rpc Issue(CredentialIssueRequest) returns (CredentialIssueResponse);

  // Refresh reissues a token with a new expiration. The returned credential
  // may or may not reuse the same identifier as the request.
  rpc Refresh(CredentialRefreshRequest) returns (CredentialRefreshResponse);

  // Revoke deletes a token and prevents it from being used again. Any children
  // of the token are also revoked.
  rpc Revoke(CredentialRevokeRequest) returns (CredentialRevokeResponse);
}

service Log {
  // Create sets up a new log stream with a given context and name.
  rpc Create(LogCreateRequest) returns (LogCreateResponse);

  // Delete removes access to an existing log stream. The log stream will no
  // longer be accessible to any client, although physical removal of data may
  // be delayed.
  rpc Delete(LogDeleteRequest) returns (LogDeleteResponse);

  // List enumerates the log stream the authenticated credential has access to.
  rpc List(LogListRequest) returns (stream LogListResponse);

  // MessageAppend adds a new message to the log stream. If the payload is
  // larger than 2MB, this RPC will return INVALID_ARGUMENT. If the service
  // needs to rate-limit this request, this RPC will return RESOURCE_EXHAUSTED
  // and additional information will be available in the QuotaFailure and
  // RetryInfo messages.
  rpc MessageAppend(LogMessageAppendRequest) returns (LogMessageAppendResponse);

  // MessageList retrieves part or all of the messages in a log stream.
  // Messages are returned in the order received by the service.
  rpc MessageList(LogMessageListRequest) returns (stream LogMessageListResponse);
}

message CredentialIssueRequest {
  // contexts is the list of allowed log storage contexts for this credential.
  repeated string contexts = 1;

  // expires_at indicates when this credential should expire.
  google.protobuf.Timestamp expires_at = 2;
}

message CredentialIssueResponse {
  // credential_id is the unique public identifier for this credential.
  string credential_id = 1;

  // contexts is the list of contexts actually granted to this token. It will
  // be a subset of the requested contexts.
  repeated string contexts = 2;

  // expires_at indicates when this credential actually expires. It will be on
  // or before the requested expiration.
  google.protobuf.Timestamp expires_at = 3;

  // token is the opaque authentication token for this credential to be passed
  // to other RPC calls.
  string token = 4;
}

message CredentialRefreshRequest {
  // credential_id is the public identifier for the credential to refresh. If
  // not provided, the credential authenticating this request will be refreshed.
  // The credential must be that of the authenticated token or one of its
  // children.
  string credential_id = 1;

  // expires_at is the desired new expiration for the given credential.
  google.protobuf.Timestamp expires_at = 2;
}

message CredentialRefreshResponse {
  // credential_id is the unique public identifier for the refreshed credential.
  string credential_id = 1;

  // expires_at is the new expiry for the credential. It will be on or before
  // the requested expiration.
  google.protobuf.Timestamp expires_at = 2;

  // token is the new opaque authentication token for this credential. Any
  // previously issued token for this credential are invalidated.
  string token = 3;
}

message CredentialRevokeRequest {
  // credential_id is the public identifier for the credential to revoke. If
  // not provided, the credential authenticating this request will be revoked.
  // The credential must be that of the authenticated token or one of its
  // children.
  string credential_id = 1;
}

message CredentialRevokeResponse {
  // credential_id is the unique public identifier of the revoked credential.
  string credential_id = 1;
}

message LogCreateRequest {
  // context for this log. It must be one of the contexts allowed for the
  // authenticated credential. If the credential only has access to one
  // context, this field is optional.
  string context = 1;

  // name is a human-readable identifier for the log stream in the provided
  // context like "stdout" or "info".
  string name = 2;
}

message LogCreateResponse {
  // log_id is the unique identifier for the newly created log stream.
  string log_id = 1;
}

message LogDeleteRequest {
  // log_id is the unique identifier for the log stream to delete.
  string log_id = 1;
}

message LogDeleteResponse {}

message LogListRequest {
  // contexts is an optional list of contexts to limit the response to. If not
  // specified, the response includes all contexts the authenticating
  // credential has access to.
  repeated string contexts = 1;
}

message LogListResponse {
  // log_id is the unique identifier for the log stream.
  string log_id = 1;

  // context for this log stream.
  string context = 2;

  // name is the human-readable identifier for this log stream.
  string name = 3;
}

message LogMessageAppendRequest {
  // log_id is the identifier for the log stream to append to.
  string log_id = 1;

  // media_type is the IANA media type for the payload. Initially, the only
  // supported media type is "application/octet-stream".
  string media_type = 2;

  // payload is the actual log data to append to the stream.
  bytes payload = 3;

  // timestamp is the time the message was originally received
  google.protobuf.Timestamp timestamp = 4;
}

message LogMessageAppendResponse {
  // log_id is the identifier for the log stream appended to.
  string log_id = 1;

  // log_message_id is an opaque identifier for the message, unique to this log
  // stream.
  string log_message_id = 2;
}

message LogMessageListRequest {
  // log_id is the identifier for the log stream to retrieve messages from.
  string log_id = 1;

  // follow indicates whether this request should stay open while new messages
  // are added to the stream. This method is opportunistic and the server may
  // cancel streaming at any time. The client may retry by issuing another list
  // request.
  bool follow = 2;

  // start_at is the offset to begin reading messages, inclusive.
  google.protobuf.Timestamp start_at = 3;

  // end_at is the offset to stop reading messages, exclusive.
  google.protobuf.Timestamp end_at = 4;
}

message LogMessageListResponse {
  // log_message_id is the stream-unique identifier for this message.
  string log_message_id = 1;

  // media_type is the IANA media type for the payload.
  string media_type = 2;

  // payload is the actual log data.
  bytes payload = 3;

  // timestamp is the time the message was originally received
  google.protobuf.Timestamp timestamp = 4;
}
```

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
  pointing to `/pls/data/logs/{logId}` to enable searching by context.

#### Messages

The storage backend for log messages is Google BigQuery. The table structure for
messages is:

```sql
CREATE TABLE `prod-1-logs.messages` (
  log_id STRING NOT NULL,
  log_message_id STRING NOT NULL,
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
  msgs.log_message_id,
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
   the BigQuery result by comparing timestamps or other metadata
1. Polls BigQuery for more data if the next message on the subscription
   indicates a gap in timestamps (likely a data loading delay)
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

There are a lot of moving parts in this subsystem and a failure in any one of
them could cause a significant delay in delivering logs to end users. Luckily, I
think we do not risk data loss once the initial acknowledgment from the PLS hits
the metadata API unless Google experiences a very significant outage (enough to
affect our other infrastructure in a way that will make this problem
irrelevant).

Retrieving logs from BigQuery is slower than doing the same from GCS by about
2-3 seconds for most requests. We expect users will tolerate this since log data
is normally only used for debugging anyway. If it becomes problematic, we may
need to investigate a caching layer of some sort.

## Rationale and alternatives

### Why is this design the best in the space of possible designs?

We evaluated numerous logging systems, including fully hosted offerings and
open-source infrastructure. Ultimately, we found tradeoffs in each. Many of the
applications we could run within our cluster were either immature or require a
lot of hands-on management and specialized knowledge. SaaS logging services are
largely not designed for API access or a multi-tenant environment; some have a
long delay before messages are available for consumption.

Ultimately, putting a microservice interface like the PLS in front of a
collection of fully managed backend PaaS offerings gives us a reasonable level
of abstraction and the ability to swap out parts of the logging subsystem
without having to change the way our other systems interact with logging.

### What other designs have been considered and what is the rationale for not choosing them?

* Sumo Logic and Loggly: Expensive, not oriented toward multi-tenant
  architectures, poor API offerings, and delayed streaming.
* Google Cloud Logging: Delayed streaming, sometimes on the order of minutes!
* NATS streaming: Management complexity and difficulty of building a persistent
  log as opposed to a simple message queue.
* Liftbridge: Too new to the market to have confidence in data consistency.
* Apache Kafka: Management complexity, archaic APIs, and difficulty of managing
  storage architecture.
* Apache Pulsar: Management complexity and difficulty of managing storage
  architecture (ZooKeeper/BookKeeper).

For each of these, we additionally struggled to figure out how to safely encrypt
user log data.

### What is the impact of not doing this?

We do not provide users with sufficient logging information when they try to use
webhook triggers. Therefore, users do not use webhook triggers, which
significantly neuters a core value proposition of our product.

### What specific risks are associated with this design?

* BigQuery itself is not _really_ designed for this sort of workload. It's
  relatively expensive; it keeps a log of every query issued, which isn't great
  when we're passing encryption keys directly in the queries; and it's slow when
  compared to our current GCS-backed implementation.
* We don't have a lot of experience with Cloud Dataflow. We've done some initial
  testing of BigQuery and understand how Cloud Pub/Sub works, but we're
  basically going off of Google's examples to verify that Cloud Dataflow does
  what we want.
* Ordering messages in a streaming context remains complex. Other than using an
  off-the-shelf persistent log platform like Pulsar, there doesn't seem to be a
  good way to make this simpler.

## Success criteria

When this is complete, the Relay API will start to provide webhook trigger logs
under the relevant API endpoint. Also, we will not see any change to the
delivery of step logs, both current and historical.

## Future possibilities

The PLS is a relatively unique software offering unto itself. It may be valuable
to other organizations that need lightweight multi-tenant logging
infrastructure, and we should consider whether it makes sense to manage and
distribute as a separate standalone project.
