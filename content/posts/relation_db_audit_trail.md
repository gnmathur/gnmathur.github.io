
+++
authors = ["Gaurav Mathur"]
title = "Audit trails in relational databases like Postgres"
date = "2023-11-11"
description = "How to implement audit trails in a relational database like Postgres"
tags = [
    "relational",
    "database",
    "postgres",
    "cdc"
]
categories = [
    "postgres",
    "database"
]
series = ["Postgres"]
+++

# Introduction
An `audit trail` is a way capture changes being made to a database table. This articles
describes a common way to implement audit trails in a relational database like Postgres 
using `audit tables`.  Audit tables record changes made to a table, including INSERT, UPDATE,
and DELETE operations. This allows you to track who made what changes and when, which is
crucial for understanding the history of the data. These trails can help in achieving
security compliance, in error resolution that require tracing the history of data, and
can service as historical records for generating reports. There's also use cases where 
audit table records can be replayed (chronologically, based on audit timestamp fields),
to reconstruct the state of a table from the beginging of time.

# A table and its audit table

Consider the following table that holds users information in a system.

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY, -- Table primary key; An autoincrementing number
    first_name VARCHAR(255), -- User first name string
    last_name VARCHAR(255), -- User last name string
    address TEXT, -- Free-form text field for address 
    last_logged_in TIMESTAMP WITH TIME ZONE, -- The time the User last logged into a system
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP, -- The timestamp this record was created
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP -- The timestamp this record was last updated
)
```

To keep track of all the changes made to the `users` table, we'll create an __audit__ table.
This table will have two versions of the fields that are in the table its auditing - one
to keep track of the _original_ value of the record, and the second to keep track of the
_updated_ values. The schema for that could be like this - 

```sql
CREATE TABLE audit_users (
    audit_id SERIAL PRIMARY KEY,
    operation_type CHAR(1), -- 'C' for Create, 'U' for Update, 'D' for Delete
    user_id INT, -- id value in the users table
    old_first_name VARCHAR(255),
    new_first_name VARCHAR(255),
    old_last_name VARCHAR(255),
    new_last_name VARCHAR(255),
    old_address TEXT,
    new_address TEXT,
    old_last_logged_in TIMESTAMP WITH TIME ZONE,
    new_last_logged_in TIMESTAMP WITH TIME ZONE,
    old_created_at TIMESTAMP WITH TIME ZONE,
    new_created_at TIMESTAMP WITH TIME ZONE,
    old_updated_at TIMESTAMP WITH TIME ZONE,
    new_updated_at TIMESTAMP WITH TIME ZONE,
    changed_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP -- timestamp this audit record was created
);
```

With these two schemas in place, we can now update both, whenever there's a CREATE, UPDATE
or DELETE operation on the `users` table. So now,

* Each row in `audit_users` corresponds to an operation performed on the `users` table
* For INSERT operations ('C'), all the `new_` fields will be populated with the inserted values. The 
`old_` fields will have NULL values
* For UPDATE operations ('U'), the `new_` fields will contain the updated values, the `old_` fields will
have the values before the update transaction was committed
* For DELETE operations ('D'), you might only have the `user_id` available. The `new_` fields could be
NULL or retain the last known values before deletion, depending on how you wish to track deletions.

This approach with an audit table captures _both_, the _old_ and _new_ values after a
transactions updating the `users` table completes. This approach for an audit table is
suitable when you have a use case where its important to get instant access to the old and
new values for a row associated with a particular update. The caveat with this approach is
that its expensive from a storage point-of-view. If storage is a concern and/or, you don't
need the old values associated with an update, a following reduced schema can be used -

```sql
CREATE TABLE audit_users (
    audit_id SERIAL PRIMARY KEY,
    operation_type CHAR(1),
    user_id INT,
    new_first_name VARCHAR(255),
    new_last_name VARCHAR(255),
    new_address TEXT,
    new_last_logged_in TIMESTAMP WITH TIME ZONE,
    new_created_at TIMESTAMP WITH TIME ZONE,
    new_updated_at TIMESTAMP WITH TIME ZONE,
    changed_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

Now this simplified audit table will only capture the state of the row _after_ the update
transaction is complete.

# Adding to the Audit table

There are at least two possible ways we can write to the audit table when there's a CREATE,
UPDATE, or DELETE operation on the `users` table. One way is to write to audit table within
the same transaction that's updating the `users` table. This approach might seem logical
however, it has the significant downsides - 
1. The authors of the transactions that update the `users` table needs to be aware that they
need to consistently update the `users` audit table too within the transaction
2. If multiple actors (services) update the `users` table, all the transactions code that
updates the table then needs to be updated
3. If there is a migration to the `users` table, again there might be a need to update all the
transactions updating the `users` table. 

All of these make this method error-prone, as developer intervention is required in all
places where the transactions are being handled.

There's an alternate approach that can take the burden of keeping the audit table updated away
from the developer. That's the use of [_triggers_](https://www.postgresql.org/docs/current/triggers.html). For Postgres, a trigger function can be defined in the following way on a table - 

```sql
CREATE OR REPLACE FUNCTION audit_user_changes() RETURNS TRIGGER AS $$
BEGIN
    IF (TG_OP = 'INSERT') THEN
        INSERT INTO audit_users(operation_type, user_id, new_first_name, new_last_name, new_address, new_last_logged_in, new_created_at, new_updated_at, changed_at)
        VALUES ('C', NEW.id, NEW.first_name, NEW.last_name, NEW.address, NEW.last_logged_in, NEW.created_at, NEW.updated_at, CURRENT_TIMESTAMP);
    ELSIF (TG_OP = 'UPDATE') THEN
        INSERT INTO audit_users(operation_type, user_id, new_first_name, new_last_name, new_address, new_last_logged_in, new_created_at, new_updated_at, changed_at)
        VALUES ('U', NEW.id, NEW.first_name, NEW.last_name, NEW.address, NEW.last_logged_in, NEW.created_at, NEW.updated_at, CURRENT_TIMESTAMP);
    ELSIF (TG_OP = 'DELETE') THEN
        INSERT INTO audit_users(operation_type, user_id, changed_at)
        VALUES ('D', OLD.id, CURRENT_TIMESTAMP);
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

>For our example schemas presented above, this trigger function updates the `audit_users`
table with the simplified schema. Modifying it to work with the extended schema, that
captures both the old and new tables values should be trivial.

This trigger can then be activated on `CREATE` `UPDATE` `DELETE` operations on the table like this - 

```sql
CREATE TRIGGER trigger_user_changes
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION audit_user_changes();
```

This approach has the advantage of taking the hassle of keeping the audit table update away
from the developer. In fact, _the author of a transaction updating the `users` table need not
be aware of the audit table at all_!

# Audit table audit timestamps

One of the things to notice in the `audit_users` table is the `changed_at` field. The purpose of this
field is to record the timestamp of when the audit record was created. Notice that the default value
of this field in the database is `CURRENT_TIMESTAMP`. The trigger function is also defined to set
this value at `CURRENT_TIMESTAMP`. This property of the `created_at` fields having the
`CURRENT_TIMESTAMP` value has a subtle but important implication, that's best illustrated by this
example. Assume the following two transactions that are lined up in time as shown -

## Transaction timeline

Assume two transactions that are executed in time like this - 

| time | transaction 1| transaction 2|
|------|----------|--------------|
| t0   |          |begin;
| t1   | `INSERT INTO Users (first_name, last_name, address, last_logged_in, created_at, updated_at) VALUES ('Sam', 'Smith', '123 Main St, Oaktown, USA', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);` | |
| t2   |          | `UPDATE Users set address = '789 Another St, Newtown,USA' WHERE id = 4;` |
| t3   |          | commit;

Assume that the insert operation at `t1` inserted a row with the identifier 4. If you look at
the audit table for `users` you will notice something peculiar. Here's an example of when
this operation was run. Take a look at the `created_at` timestamp. The timestamp value for
the Update operation, which is chronologically _after_ the create operation, is _lower_ 
(`16:14:01.677`) than that of the Create operation (`16:14:22.834`)! 

```
6	C	4	Sam	Smith	123 Main St, Oaktown, USA	2023-11-25 16:14:22.834 -0800	2023-11-25 16:14:22.834 -0800	2023-11-25 16:14:22.834 -0800	2023-11-25 16:14:22.834 -0800
7	U	4	Sam	Smith	789 Another St, Newtown, USA	2023-11-25 16:14:22.834 -0800	2023-11-25 16:14:22.834 -0800	2023-11-25 16:14:22.834 -0800	2023-11-25 16:14:01.677 -0800
```

The reason why this happens is because the CREATE_TIMESTAMP value is captured at the 
_beginning of a transaction_. Since in the above timeline, the second transaction was 
started earlier than the first one, which actually did the insert, the update audit entry
for the same row (with id 4), had a lower timestamp value. The implication of this observation
is that with this approach we would not be able to _replay_ transactions back on a table
purely based on a the `created_at`. One important thing to mention is that this issue only happens if the transaction isolation level for the update transaction is 
`READ COMMITTED` or lower. It will not happen if its either `REPEATABLE READ` or
`SERIALIZABLE`.

# Final Notes

We have talked about keeping audit records in a database like Postgres. Similar technique
can be employed with other relational database like MySQL, SQL Server etc. The notion of
keeping an audit trail is not limited to relational database though. Some databases offer
solution-specific audit solutions. For example, MongoDB offer an auditing facility described
[here](https://www.mongodb.com/docs/manual/core/auditing/), Cassandra offers [Audit Logging](https://cassandra.apache.org/doc/4.0/cassandra/operating/audit_logging.html) etc.