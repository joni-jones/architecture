# Introduction of UUID

## Overview

According to Magento [Technical Guidelines](https://devdocs.magento.com/guides/v2.3/coding-standards/technical-guidelines.html#64-service-contracts-application-layer), service contracts should support client code generated IDs.

> 6.4.3.7. Operation UUID MAY be provided by the client during service invocation. UUID MUST allow the client to get the operation status information.

> 6.4.3.9. Service contracts SHOULD allow client side generated IDs. A service SHOULD accept an ID instead of generating it. See UUID as an example of client side generated ID.

It means, that entity may have an ID different to the primary key in database and not obviously integer and sequential. But in the current reality, most of the entities use a database primary auto increment key as ID and this approach has several drawbacks:

 - There is no possibility to specify ID value before persisting the entity
 - Primary Key as ID guarantees uniqueness only across one table for the same database
 - Business logic depends on the persistence layer
 - Persistence of multiple related entities might be a bottleneck because entities should be stored sequentially one after another
 - Entity persistence takes 2 operations: one for entity save, another to get the ID for the saved entity
 - Violates CQRS principles as create operations (Commands) should not return data

The mentioned drawbacks force us to consider other options as ID for entities.

## Solution

A few years ago, Magento architects published a [blog post](https://community.magento.com/t5/Magento-DevBlog/Entity-ID-Allocation-Schemes/ba-p/68316) to describe a problem with auto-increment keys as entity ID. In a general case, we can define two main ID allocational strategies Sequence Number and UUID. And each of these strategies has own pros and cons.

### Sequence Numbers

In general, the sequence number is represented by integer sequential numbers provided by a separate database table or even service like Twitter Snowflake or Sonyflake. PostgreSQL [provides](https://www.postgresql.org/docs/12/datatype-numeric.html#DATATYPE-SERIAL) possibility to use sequence numbers as primary auto-increment keys.

Let's consider pros and cons of sequence numbers usage.
Pros:
- Sequence numbers are represented by integers
- Unsigned integers take less database space than strings and binary
- Better for indexing as represented in the natural order

Cons:
- Additional query to database table or service to get sequence number before entity persistence
- In the case of MySQL, the increment step [depends on](https://dev.mysql.com/doc/refman/8.0/en/replication-options-master.html#sysvar_auto_increment_offset) the database server configuration, which might cause "holes" in sequences

Magento already uses sequence numbers for some business entities like order, invoice, credit memo, shipment, RMA, catalog category, product and related options, CMS page and block, catalog and cart rules.

### UUID

Another approach is to use [UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier). There multiple versions of UUID, which will be considered below in this proposal, but in general, [UUID](https://tools.ietf.org/html/rfc4122#section-3) is an identifier that is unique across both space and time, with respect to the space of all UUIDs represented by a 128-bit number. In canonical textual representation, UUID is presented as 32 hexadecimal digits separated by 4 hyphens:
```
xxxxxxxx-xxxx-Mxxx-Nxxx-xxxxxxxxxxxx
```
Where `M` is UUID version (1-4) and `N` is a code of [variant](https://tools.ietf.org/html/rfc4122#section-4.1.1).

RFC defines the following UUID versions:

- UUID 1 (date-time and MAC address) - based on a combination of a timestamp and a "node" MAC address. MySQL `uuid()` uses UUID 1
- UUID 2 (date-time and MAC address, DCE security version) - similar to UUID 1 but reserved for [DCE security version](https://pubs.opengroup.org/onlinepubs/9629399/apdxa.htm) UUIDs
- UUID 3 and 5 (namespace and name-based) - UUID 3 uses `md5` and UUID 5 uses `sha1` as hash functions to hash data for value generation. The same namespace and name will map the same UUID. A namespace identifier is itself a UUID. In most cases, UUID 3 and 5 are used to represent URLs, FQDNS, object identifiers.
- UUID 4 (random) - randomly generated value.

Let's consider the pros and cons of UUID usage.
Pros:
- Environment independent, the most of programming languages and database provide functions to generate UUID
- Unique across applications/tables/databases
- A value is known before the entity is persisted
- The value does not expose the information about the entity
- Suitable for distributed systems

Cons:
- Take up more storage space as in most cases stored as binary and cannot be represented as integers
- Inserts to the storage are random as UUID do not support natural ordering
- Database indexing less efficient and size of indexes is bigger
- Less convenient for humans, especially if stored as binary in database

Scenarios of UUID much wider than just as a primary key for the entity. UUID can be used as identifiers for:
 - logging and tracing records
 - history of executed operations
 - cross application/database data import
 - payment processing for order with the same order numbers (which are unique only across one application) from different applications

Based on the described pros and cons, the UUID's usage looks more preferable than sequence numbers.

### Benchmarks

By the nature of specification and implementation UUIDs less performant than auto-increment IDs. We need to consider multiple scenarios and provide benchmarks for them. The proposal considers the following scenarios:

- Auto increment BIGINT as primary key
- Auto increment BIGINT as primary key + index on BINARY UUID column
- BINARY UUID as primary key
- Rearranged UUID stored in BINARY as primary key

> The latest scenario uses the UUID 1 transformation to make it closer to sequential for a more optimal database insert. As 3rd part of UUID represents the UUID version it always represented by a number and we can use it as the first part of UUID. The 2nd part represents time-mid and will be on its the place. The 1st part is time-low and will be after time-mid. The last two parts are without changes. And we can remove dashes. The UUID transformation looks like this `58e0a7d7–eebc–11d8-9669-0800200c9a66` => `11d8eebc58e0a7d796690800200c9a66`. The original idea got from [this post](https://www.percona.com/blog/2014/12/19/store-uuid-optimized-way/).

Each table contains 500K records and average is calculated time for 1K queries.
