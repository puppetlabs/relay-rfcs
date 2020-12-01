# Relay Identity Management (2020-11-20)

## Stakeholders

* **Recommenders:** Kyle Terry
* **Agreers:** Geoff Woodburn, Noah Fontes, Nathan Ward
* **Performers:** Kyle Terry, Nathan Ward
* **Inputers:** Rick Lane, Melissa Cherry
* **Deciders:** Noah Fontes, Kyle Terry, Nathan Ward

## Problem

Relay's SaaS platform currently hand rolls identity management, including user
accounts, auth and rbac. There is a lot more work to do, including implementing
single sign-on and social login using OAuth.

## Summary

This RFC defines an integration with a 3rd party identity management platform
that handles user management, RBAC and authentication support across user/pass,
social via OAuth and single sign-on.

## Motivation

We want to utilize trusted and proven identity management tools to do the hard
work for us. This allows us to standardize these processes across the API and
reduce the large code footprint that's normally required to homebrew identity
management.

In addition, we can take advantage of the data we get from OAuth flows to
further integrate relay with services like GitHub without having to setup the
OAuth connection ourselves. Doing this manually is time consuming.

## Product-level explanation

This RFC provides a plan for implementing a 3rd party identity management system
and a migration path for moving current user account data and roles over to it.

This work includes moving our user records into the IDP data store and
re-engineering some parts of our client and server side code to authenticate and
authorize the users of our system.

Doing this will allow us to easily integrate with Puppet's own SSO provider and allows
our users to use their pre-existing social logins (like GitHub and Gitlab) to
login to Relay.

## Engineering-level explanation

Identity management will be provided by IDP. 

### Deliverables

* Register an account with IDP and configure the minimum components.
* Re-engineer client side code to work with IDP
* Re-engineer API side code to work with IDP
* Build a migration tool to move users, roles and permissions to IDP database.
* Attempt the deploy on staging and test.
* Security audit/test for leaks.
* Deploy to production.

### Operational impact

* At a minimum, this work completely overhauls how users register with and login
  to Relay.
* In addition, some major changes to RBAC will need to be made in the API code.
* An audit of the security ramifications of forcing a user to leave the API
  should be done.

### Client-side integration

TBD

### Server-side integration

#### Authorization (authn) interface

The authn interface will be an implementation of our existing `UserManager` and
`AccountManager` interfaces. Both of these interface behaviors will change and
become unions of new `Reference` types for both users and accounts.

`User` and `Account` will eventually be deleted. 

Two new tables and management interfaces will be introduced to reference the
remote objects in the IDP. These new data models will reside in Relay's Postgres
database.

```go
type UserReferenceManager interface {
    Create
    Delete
    All
    GetByID
    GetByExternalID
}

// UserReference is a data model for a record inside the user_refernces table.
type UserReference struct {
    // ID is the internal database ID
    ID string
    // ExternalID is the id provided by the IDP
    ExternalID string
    CreatedAt time.Time
    UpdatedAt *time.Time
}

type TenantReferenceManager interface {
    Create
    Delete
    All
    GetByID
    GetByExternalID
}

// TenantReference is a data model for a record inside the tenant_refernces table.
type TenantReference struct {
    // ID is the internal database ID
    ID string
    // ExternalID is the id provided by the IDP
    ExternalID string
    CreatedAt time.Time
    UpdatedAt *time.Time
}
```

UserManager changes include removing methods and repurposing implementations to
call IDP APIs instead of loading records from the internal database.

* REMOVE `UserManager.Create`
* REMOVE `UserManager.Update`
* REMOVE `UserManager.Delete`
* REMOVE `UserManager.UpdateProfile`
* REMOVE `UserManager.AuthenticateWithPassword`
* REMOVE `UserManager.UpdatePassword`
* REMOVE `UserManager.ForgotPassword`
* REMOVE `UserManager.ComparePassword`

#### Authorization (authz) interface

The authz interface will utilize the scopes sent from the IDP for a user and
apply the same as the current model Relay uses. A couple refactors are required
for this to work:

`Role` and `Permission` become data models that aren't tracked by our database.
We would load permissions into the current Granter interface the same, but the
`Permission` objects would be preemptively loaded and returned by a method on
`User`.

`PermissionManager` will be removed and `RoleManager` will be refactored to make
changes to a user's via the IDP API.

#### Data migration

**NOTE** do NOT remove users, accounts, permissions and roles tables just yet.

I don't think it would be worthwhile to migrate either staging or production
without downtime, so I propose we announce and schedule downtime for the
deployment. This will allow us to verify the user accounts, tenants and RBAC
have been migrated successfully and the relevant reference tables have been
synchronized.

Below outlines the multi-deploy phased rollout of the migration.

**Pre-deploy**: Using tools and interfaces provided by IDP, create a Connection
for username/password authentication and setup the base-RBAC structure that most
tenants will use.

**Deploy 1**: We create the `*_references` tables with a schema migration and
run a data migration to copy all the relevant IDs into those tables, leaving
`external_id` empty. This deploy contains code to copy the ID for any new
Account and User records created before the next deploy into those tables.

**Deploy 2**: We start the scheduled downtime and deploy a version of the API
that contains a migration to do the following:

* Generate a tenant in IDP for each Relay account.
* Generate a JSON dump of Relay users (users.json) matching IDP's bulk import
  schema.
    - include email, password_hash, profile, preferences and current
      roles/permissions.
* Request import job using IDP API -- sends users.json.
* Check import job status until complete.
* Cleanup users.json.
* Use IDP API to retrieve bulk user and tenant records to synchronize
  `*_reference` tables with IDP `external_id` for the entity.
* Once complete, end scheduled downtime.

## Rationale and alternatives

### Why is this design the best in the space of possible designs?

This offloads a lot of complexity to systems that are purpose built for handling
the identities of our customers and their users. In addition to identity
management, we can integration with external services using OAuth to create
connections for things like GitHub and other hosted code repositories and allow
our customers to easily use their own SSOs.

It allows us to reduce the code footprint and indirection, which should help us
spot bugs easier and write cleaner and safer code, overall.

### What is the impact of not doing this?

We will need to maintain our current authn and authz system in addition to
implementing support for OAuth for social logins, 2FA, and SSO. All of which are
time consuming and complicated.

## Drawbacks

* There are potential security implications that can make a thorough audit hard to
  reconcile. It's also a potential blocker for customer who have a very strict
  compliance policy.

* The time to migrate the data and re-engineer the UI and API might be a lengthy
  process. I can see this easily being 2 sprints worth of work for 2 engineers.

## Success criteria

* Users will be able to register and log into Relay using a regular
  email/password or by clicking one of the social login buttons we want to
  support (GitHub, Google).

## Unresolved questions

* Level of effort for the UI?

## Future possibilities

* Automating SSO enrollment for enterprise customers
* OAuth connections usable by workflows
