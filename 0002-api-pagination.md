# API pagination, limiting, sorting, and searching (2019-09-01)

## Stakeholders

* **Recommenders:** Manny Batule, Noah Fontes
* **Agreers:** Noah Muldavin, Sebastian Prokuski
* **Performers:** [PN-259](https://tickets.puppetlabs.com/browse/PN-259)
* **Inputers:** Brad Heller, Rick Lane, Kyle Terry, Geoff Woodburn
* **Deciders:** Rick Lane, Noah Muldavin, Kyle Terry

## Problem

Currently, our HTTP API list-style responses contain full, unordered result sets
for all endpoints, with the exception of the source control integrations'
ability to list repositories and branches.

From a product perspective, this is unfortunate because it causes the
application to perform more slowly as the number of records returned increases,
and it can cause confusion due to the lack of ordering of results.

From an engineering perspective, this has security implications (creating a
multitude of resources could effectively cause out-of-memory errors when
retrieving them, i.e. a DoS attack). It also makes it more difficult to perform
operations external to the database, like querying integration repositories and
branches, that may be subject to throughput restrictions, etc., in their APIs.

## Summary

This RFC defines the user-facing syntax the API uses to handle the following
request options:

* Pagination
* Limiting
* Sorting
* Searching

It defines the syntax the API uses when responding for pagination.

It provides a general guideline for implementing these features in the API, as
well as scope of work for completing the changes.

## Motivation

This RFC addresses all of the issues outlined in the problem statement,
preventing unnecessary memory utilization, and aligns our API to other APIs on
the internet. Customers should find the pagination support enables them to work
faster as they expand their use of the product.

## Product-level explanation

This RFC does not expose any end-user-facing functionality, but it does enable
convenience features, such as pagination through lists of repositories and
repository branches belonging to an SCM integration. We do not anticipate any
customer engagement required to assess the utility of this feature. Over time,
without this feature, we may experience significant customer-facing performance
degradation.

This change should bring our API closer to what customers expect to find in an
API, should we ever publish it publicly.

## Engineering-level explanation

### Definition

#### Requests

Consider the following list response for an arbitrary type `widget` located at
`/api/widgets`:

```json
{
    "widgets": [
        {
            "id": "2f767f4a-eade-4330-a631-1a0af7380271",
            "name": "Mailbox"
        },
        {
            "id": "a0241fa9-4436-432f-a80b-06fdca845216",
            "name": "Flying Disc of Frobozz"
        },
        ...
    ]
}
```

We propose amending the request to support the following additional query string
parameters:

* `limit=`*`N`*: The response will contain at most *N* entities, but potentially
  arbitrarily less.
* `page=`*`P`*: The response will begin enumerating results starting from page
  *P*. Acceptable values for *P* are given by prior list responses and cannot be
  constructed by API clients.
* `sort=`*`[-]S`*: The response will sort by field *S*. If *S* is preceded by
  the literal character 0x2D (`-`), the sort order is descending. Otherwise, the
  sort order is ascending. This field may be specified multiple times. For
  example, `sort=user.id&sort=-created_at`.
* `search=`*`Q`*: Perform a full-text search using the query *Q*.

These parameters are hereby standardized and reserved in the API. No API request
shall repurpose these parameters with different semantics, even for endpoints
that do not represent lists of entities.

#### Responses

We propose amending the response to support an additional field, `page`, at the
top level of the response:

```json
{
    "widgets": [...],
    "page": {
        "prev": "5/b1n0YlYWbsyOxU0X75",
        "next": "Mg9O7NvEZKiNWtfC+HCl"
    }
}
```

The values of fields within `page` are opaque; they cannot be decoded by an API
client into any meaningful information. The meaning of the cursor is given by
the field name, and correspond to one of the [IANA-registered link
relations](https://www.iana.org/assignments/link-relations/link-relations.xhtml),
with the semantics given therein. For clarity:

* `prev`: A page of data before the current page, such that the last entity
  returned in a corresponding response is the entity exactly before the first
  entity in the current response.
* `next`: A page of data after the current page, such that the first entity
  returned in a corresponding response is the entity exactly after the last
  entity in the current response.

If, at the time a response is rendered, no entities fulfill the conditions of
one or more of the page fields, those fields may be omitted entirely. However,
the `page` structure must never be omitted from a list response.

Pagination is susceptible to the request's sort order. Changing or omitting the
sort order in subsequent requests may reorder the results only in the exact way
that causes the semantics of the page field to be fulfilled.

#### Defaults

Each API entity type shall define a default sort order, if none is specified in
the request, as appropriate. For example, for widgets, a sensibile default sort
may be by the `name` field in ascending order.

Each API entity type shall define default and maximum limits. A maximum limit is
required to ensure a hard upper bound for memory utilization in the API.
Requests for limits above the maximum shall return the maximum without error.

#### Entity-specific requirements

All list endpoints must support the `limit` and `page` query parameters, and
must provide `page` in the response.

All list endpoints must support the `sort` query parameter; however, it may be
limited to the default sort order (ascending or descending).

Endpoints are not required to provide any filtering mechanisms, including via
the `search` parameter. However, if a full-text search is permitted, it must be
called `search`.

### Implementation

For now, we do not consider full-text search in any API response. It is already
enabled (correctly) as mentioned above for SCM integrations in the relevant
endpoints, and is not needed elsewhere currently.

That leaves pagination and sorting, both of which need to be pushed down into
the query layer in the API. We propose ad-hoc modification to the query
functions to accommodate these requirements.

For the managers, we propose a standardized `Options` struct format to provide
pagination, sorting, and filtering for a given entity type. Using workflows as
an example:

```go
// model/page.go

type PageTerminus int

const (
    PageTerminusStart PageTerminus = iota
    PageTerminusEnd
)

type Page struct {
    ID       string
    Terminus PageTerminus
}

type Pages struct {
    Next *Page
    Prev *Page
}

type Pagination struct {
    Limit int
    Page  Page
}

// model/workflow.go

type WorkflowSortField string

const (
    WorkflowSortFieldName WorkflowSortField = "name"
)

type WorkflowSort struct {
    Field      WorkflowSortField
    Descending bool
}

type WorkflowFilters struct {
    IntegrationID []string
}

type AllWorkflowOptions struct {
    Pagination Pagination
    Sort       []WorkflowSort
    Filters    WorkflowFilters
}

type WorkflowManager interface {
    AllByAccountID(ctx context.Context, accountID string, opts AllWorkflowOptions) ([]*Workflow, Pages, error)
    // ...
}
```

By providing consistent pagination types, we can reduce the effort to implement
the `limit` and `page` request parameters and the `page` response field to
simple helper methods. Sorting remains a one-off, but since we don't need to
provide any sorting other than the default for now, we can actually skip most of
the associated complexity.

We need to address the interaction of RBAC with limiting result sets. The RBAC
push-down behavior may need to manipulate result sets and perform multiple
requests to the underlying database. I leave this as an open question.

### Tasks

| Component | Size | Description |
|-----------|------|-------------|
| API | L | Update API database queries to have default sort order and support pagination implementation |
| API | M | Update API managers to conform to the standard implementation |
| API | S | Update OpenAPI specification |
| API | M | Add support for `page` and `limit` to list requests, and add `page` to each API list response |
| API | M | Add support for `sort` to list requests |
| GUI | L | Add automatic pagination support to list responses |
| CLI | S | Add automatic pagination support to list responses |
| GUI | L | Design pagination components |
| API | S | Enforce default limits for paged responses |
| GUI | M | Add pagination components to lists where appropriate |

## Drawbacks

As with all system-wide additions to the API (e.g., RBAC, rigor around OpenAPI,
etc.), this introduces a cognitive burden on implementers of list responses. It
also increases client complexity, as every client will now need to figure in the
pagination data to determine if their result set is complete enough for their
needs.

This design does not provide a total number of records in the paginated
response, nor does it allow skipping pages, so it is impossible to build a UI
that shows literal pages, which is common on the Web, e.g.:

`[ 1 | ... | 3 | 4 | ` ` `**`5`**` ` ` | 6 | 7 | ... | 20 ]`

## Rationale and alternatives

### Why is this design the best in the space of possible designs?

This design ensures the integrity of list responses, i.e., that no records will
be omitted from the response as a client iterates through pages, with a tradeoff
of slightly greater complexity for a client (or human implementer), as
cursor-based pagination is more abstract than offset-based pagination. We
deliberately prioritize data integrity in this proposal, as it carries greater
weight with the type of entities we return. (For an e-commerce product catalog,
we would likely consider offset-based pagination, for example.)

### What other designs have been considered and what is the rationale for not choosing them?

The other typical design for pagination is to provide an integer offset instead
of a set of cursors. This is easier from a client perspective, and rendering
arbitrary offsets is possible. However, it exposes a serious risk of duplicating
or omitting relevant data: if a record that appears earlier in the sort order is
inserted between paginated requests, a subsequent response will duplicate the
last record of data in the current page. Similarly, removing a record will cause
the subsequent response to omit a record entirely.

### What is the impact of not doing this?

In the immediate term, it primarily affects the ability to paginate through the
results from listing SCM integrations' repositories and branches. Over time, we
may experience DoS conditions as we exhaust memory trying to load many thousands
or tens of thousands of workflow runs, etc.

### What specific risks are associated with this design?

Although the API response itself is backward-compatible, the addition of default
limits for responses is not. Therefore, this change will require an upgrade to
API clients.

As with entity responses, the scope of this work affects all list responses, and
has the potential to conflict with new feature work.

## Success criteria

All list responses support pagination and sorting. We successfully eliminate one
substantial unbounded memory condition in the API. Clients, including the GUI
and CLI, integrate paged requests and responses completely and without
substantial difficulty.

## Open questions

* The current design of RBAC makes it difficult to push down entity instance
  authorization filters to the database layer. We should find a long-term
  solution to this, but in the medium term, we can simply continue to request
  data until we fill the requested limit.

## Future possibilities

We may expand the full-text capability of the `search` query parameter to
support more complex filtering à la ElasticSearch or Solr's query syntax.
