---
layout: posts
title:  "Sql injection  with prepared statements"
excerpt: We will explore how to be safe against sql injection attacks.
date:   2021-11-07 03:00:47 +0530
layout: single
tags: [rails, postgresql]
---
We will explore how to be safe against sql :syringe: attacks. Using bound parameters and prepared statements.

Most of the time when interacting with the database from rails application. We use `ActiveRecord` and most of the time we can just use the ORM to generate the sql query.

But sometimes we might need to venture out of the usual path and write our own queries. Sometimes we need to write complex reporting queries.

For example, let us say we need to find the distance between two coordinates or words. In these cases, there is no ready-made help available from `active-record`. We will need to write custom sql queries.

We can use trigram search for that. And Postgres has [pg_trgm](https://www.postgresql.org/docs/current/pgtrgm.html) module which provides us with the necessary tools.

Let us move on to a concrete example.

## Seed Database

To setup our test environment, we will insert an entire dictionary to the database as seed data.

We will start with setting up a Postgres instance and also check the IP address which will be helpful later.

![start_pg](/assets/images/start_pg_2021-11-07-02:17:03.png)

Next, we can fire the psql shell and create `words` tables.

![create_table](/assets/images/create_table_2021-11-07-02:19:04.png)

We will start a ruby container and install a dictionary.

![install_dict](/assets/images/install_dict_2021-11-07-02:11:28.png)

Time to write some code to seed the database with the dictionary.

![seed_data](/assets/images/seed_data_2021-11-07-02:25:58.png)

We have successfully seed the database with `102774` records.

## SQL :syringe:

Now that we have the database setup, let us define the task at hand.

**We would like to grab the 5 closest matches for an user input word.**

We can use `pg_trgm` for this. The `<->` operator Returns the 'distance' between the arguments.

And to run the query we can use `ActiveRecord::Base.connection.execute`

![raw_sql_1](/assets/images/raw_sql_1_2021-11-07-02:51:36.png)

But that is prone to sql injection

![raw_sql_2](/assets/images/raw_sql_2_2021-11-07-02:52:01.png)

There are roughly two well-known safe-guards present to solve this.
1. Quoting
2. Bind Parameters

### Quoting

We can quote the column value to help prevent sql injection.

ActiveRecord provides us with [ActiveRecord::Base.connection.quote](https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/Quoting.html#method-i-quote).

![quoting](/assets/images/quoting_2021-11-07-02:58:40.png)

Important to remember
> Connection quoting in Rails uses a db connection from the existing pool, a repeated call and a heavy load might affect performance.

### Bind Parameters And Prepared Statements

In most DBMS we can use a prepared statement, which pre-compiles the SQL query, separating it from data.

It is a server-side object that can be used to optimize performance, reducing/eliminating SQL injection attacks.

Postgresql support PREPARED statement, you can read more about it [here](https://www.postgresql.org/docs/9.3/sql-prepare.html).

Let us fire up the psql shell and use PREPARED statements.

![bind_query](/assets/images/bind_query_2021-11-07-03:25:20.png)

We can check the pg log.

![pg_log_1](/assets/images/pg_log_1_2021-11-07-03:22:08.png)

Back to rails. How do we achieve this using ActiveRecord?

Well ActiveRecord has [find_by_sql](https://api.rubyonrails.org/v6.1.4/classes/ActiveRecord/Querying.html#method-i-find_by_sql) method, which helps us with this.

![find_by_sql](/assets/images/find_by_sql_2021-11-07-03:11:15.png)

Let us check the pg log.

![pg_log_2](/assets/images/pg_log_2_2021-11-07-03:33:47.png)

Important to note prepared statement's lifecycle is per-connection basis.

> Prepared statements only last for the duration of the current database session. When the session ends, the prepared statement is forgotten.

![pg_log_3](/assets/images/pg_log_3_2021-11-07-03:38:51.png)


Alright, that is all I had to share on sql injections.

Until next week! :heart:
