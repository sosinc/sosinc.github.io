---
title : "Lossless soft-deletes with Hasura"
date  : 2020-09-03T10:00:01+05:30
description: "How to implement soft delete with Hasura while keep using database features and maximizing developer productivity"
feature_image: "/images/blog/soft-delete-hasura.jpg"
image: "/images/blog/quick-sand-of-redux-1280.png"
author: Jatinder Khabra
---

Soft deletes provide the ability to delete records from a database, without
actually losing data. It's like putting data in a recycle bin, so that it is
practically deleted, but can still be restored if needed.

Most important benefit soft-deletes provide is perhaps, the restore capability.
They are also important for keeping a healthy data warehouse. We don't want
references in our data warehouse to suddenly start pointing to non-existent
data.

### Traditional soft delete

Traditionally, we simply add a bit column, e.g `is_deleted` to mark a row as
deleted. Instead of deleting rows, we flip this bit column. It is a rather
popular method, especially in smaller applications. Even [Hasura itself
recommends](https://hasura.io/docs/1.0/graphql/core/guides/data-modelling/soft-deletes.html#setting-up-soft-deletes-for-data)
this method to handle soft deletes.

It is however a double-edged sword. Our biggest beef with this method is that we
lose a lot of database's power. **Soft-deleted records cause constraint
violations**, which usually means not using any constraints at database level at
all. Designing database indexes also become very hard.

### Alternatives

Problems with the traditional soft-delete are serious enough. Two common
approaches used to circumvent them are:

1. Using an event sourcing database, which keeps an immutable record of all events
2. Exporting deleted records to an outside storage, and re-importing for restore

They both have their pros and cons as well. Event sourced systems tend to grow
very complex, and need a lot of care for growing. Exporting and importing itself
becomes a pain to maintain for developers.

### Goals

When writing [Sos App](https://github.com/sosinc/sos-app), we had a clear
picture of what we wanted to achieve from our soft-delete implementation:

1. Full utilization of database's abilities

   Fully functional database constraints is a must-have for clean boundaries.
   Easy and efficient indexes are also critical.

2. Developer productivity

   Soft delete shouldn't become a burden for developers. SoS Inc is a small
   development shop. We try to optimize for developer productivity as much as
   we can.

### Our Approach

![Soft Deletes with Hasura](/images/blog/delete-flow.jpg)

We went for a sort of a hybrid of *event sourcing* and *exporting deleted
records*. This approach we laid out was:

1. Actually delete rows from database, so that deleted data don't interfere with
   our everyday use
2. Export deleted rows to a dedicated deleted_entities table, in form of
   "Deleted" events

Trick is to use Hasura's trigger mechanism for making the whole process really
cheap to develop. Once the hook is set, zero developer intervention is needed
moving forward. Developers can keep using the database without needing to pay
any special care to soft deletes.

The whole process goes like:
1. Client deletes a record
2. Hasura actually deletes the record from database, but raises an event to be
   processed by a sidecar application
3. Sidecar prepares a "delete event" with additional information
    - Who deleted the row?
    - Which table has the row been deleted from?
    - The actual data that was deleted
4. Sidecar inserts the "delete event" to `deleted_entities`

### Wins

- Allows full utilization of database capabilities, e.g constraint violations
- No future intervention from developers needed
- Queries do not need to be manually adjusted to account for soft deletion logic
- Keep track who deleted the data and when
- Does not become a problem as application grows
