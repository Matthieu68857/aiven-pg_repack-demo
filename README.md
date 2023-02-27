# Quick walkthrough for Postgres extension pg_repack

As [an addition to this video](https://video.aiven.io/watch/Cwy8qjBqSVVrFBkcnJZfXb?), you can find below the list of steps and queries used in the demo.

## Building our bloated target 

To demonstrate pg_repack, we are gonna build a bloated table.
First create a very simple table:

```
CREATE TABLE random_strings (
    id SERIAL PRIMARY KEY, 
    line text
);
```

And populate it with random generated strings.
Simple PL/pgSQL function to generate strings:
```
CREATE OR REPLACE FUNCTION generate_random_string(
  length INTEGER,
  characters TEXT default '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz'
) RETURNS TEXT AS
$$
DECLARE
  result TEXT := '';
BEGIN
  IF length < 1 then
      RAISE EXCEPTION 'Invalid length';
  END IF;
  FOR __ IN 1..length LOOP
    result := result || substr(characters, floor(random() * length(characters))::int + 1, 1);
  end loop;
  RETURN result;
END;
$$ LANGUAGE plpgsql;
```
And a big INSERT:
```
WITH str AS (SELECT generate_random_string(1024) AS value)
INSERT INTO random_strings (line)
SELECT value
FROM generate_series(1, 500000), str;
```

Now that we have a table with a lot of rows, **we want to generate some dead rows**. To do that simply, we can just disable autovacuum on the table and execute a DELETE statement.

*(In real life, autovacuum is reponsible to clean dead tuples. But sometines, things can go wrong and autovacum is not able to: long failed transactions, abandonned replication slots, etc.)*

```
ALTER TABLE random_strings set (autovacuum_enabled = off);
```

We can now delete random rows:

```
DELETE FROM random_strings WHERE ctid IN (SELECT ctid FROM random_strings LIMIT 100000);
```

## How to see if my table is bloated?

To check the dead rows of our table, we're gonna use to pgstattuple extension.

```
CREATE EXTENSION pgstattuple;

SELECT * FROM pgstattuple('random_strings');
-[ RECORD 1 ]------+----------
table_len          | 585146368
tuple_count        | 400000
tuple_len          | 422400000
tuple_percent      | 72.19
dead_tuple_count   | 100000
dead_tuple_len     | 105600000
dead_tuple_percent | 18.05
free_space         | 53146356
free_percent       | 9.08
```

As you can see, our DELETE query has generated 100k dead tuples. That means that those rows are still present on disk and **those blocks cannot be used to store new data**.

Two solutions here:
- Execute a `VACUUM FULL` on our table -> this will require an EXCLUSIVE LOCK during the whole process, making INSERT/DELETE/UPDATE forbidden. 
- Or **we can use pg_repack extension** and **keep using the table as usual**

## pg_repack in action

For the full documentation of pg_repack, see [https://reorg.github.io/pg_repack/](https://reorg.github.io/pg_repack/)

Things to remember:
- pg_repack only works on table with a PRIMARY KEY (or a UNIQUE index)
- You need to use the exact same binary version as the one installed on your Postgres server
- You need to have enough free disk space before executing pg_repack *(the process will create a table in the background containing all rows of your table, more details in the documentation)*

***Tips to use pg_repack:** build your own docker image with the right version to use (1.4.7 for pg14 for example). Great example can be found on [this Github project](https://github.com/hartmut-co-uk/pg-repack-docker).*

Let's start! As stated in the [Aiven documentation](https://docs.aiven.io/docs/products/postgresql/howto/use-pg-repack-extension), you just have to install pg_repack extension on your Postgres server and then run pg_repack binary.

```
CREATE EXTENSION pg_repack;
```

And finally:

```
pg_repack -h $PGHOST -p $PGPORT -U $PGUSER -k -d defaultdb --table=random_strings
```

During the process, you can try to add/update some data, it works!
Once pg_repack is done, voilà, no more dead tuples:

```
select * from pgstattuple('random_strings');
-[ RECORD 1 ]------+----------
table_len          | 468115456
tuple_count        | 400004
tuple_len          | 422400156
tuple_percent      | 90.23
dead_tuple_count   | 0
dead_tuple_len     | 0
dead_tuple_percent | 0
free_space         | 42515276
free_percent       | 9.08
```