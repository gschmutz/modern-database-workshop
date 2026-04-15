# Working with CockroachDB

In this workshop we will learn how to work with **CockroachDB**, a distributed **NewSQL** database that speaks the PostgreSQL wire protocol. We will use the same 3NF film data as the PostgreSQL workshop (workshop 01), and then explore the features that set CockroachDB apart: its built-in Admin UI, time-travel queries, distributed transactions, and range management.

## Table of Contents

- [What you will learn](#what-you-will-learn)
- [Prerequisites](#prerequisites)
- [Background: What is NewSQL?](#background-what-is-newsql)
- [CockroachDB vs PostgreSQL](#cockroachdb-vs-postgresql)
- [The 3NF Schema Design](#the-3nf-schema-design)
- [Connecting to CockroachDB](#connecting-to-cockroachdb)
- [Creating the Database and Schema](#creating-the-database-and-schema)
- [Inserting Data](#inserting-data)
- [Basic Queries](#basic-queries)
- [Joining Tables](#joining-tables)
- [Aggregating Data](#aggregating-data)
- [CockroachDB-Specific Features](#cockroachdb-specific-features)
- [Transactions](#transactions)
- [Performance with EXPLAIN](#performance-with-explain)
- [CockroachDB vs PostgreSQL: a side-by-side comparison](#cockroachdb-vs-postgresql-a-side-by-side-comparison)

## What you will learn

- What distinguishes NewSQL from traditional RDBMS and NoSQL databases
- How to connect to CockroachDB using the `cockroach sql` CLI, `psql`, the Admin UI, and DbGate
- How the same 3NF schema works identically in CockroachDB
- How to use CockroachDB-specific SQL commands (`SHOW RANGES`, `SHOW JOBS`, `AS OF SYSTEM TIME`)
- How CockroachDB handles distributed ACID transactions
- How to choose primary key strategies that avoid hotspots in a distributed system

## Prerequisites

- The **Data Platform** described [here](../00-environment/README.md) is running and accessible
- Familiarity with the 3NF schema introduced in the [PostgreSQL workshop](../01-working-with-postgresql/README.md) is helpful but not required

## Background: What is NewSQL?

Traditional databases fall into two camps:

| Category | Examples | Strength | Weakness |
|----------|----------|----------|---------|
| **RDBMS** | PostgreSQL, MySQL, Oracle | ACID, rich SQL, strong consistency | Scales up (bigger hardware), not out |
| **NoSQL** | MongoDB, Cassandra, Redis | Scales out horizontally | Sacrifices ACID, SQL, or both |
| **NewSQL** | CockroachDB, Google Spanner, YugabyteDB | ACID + full SQL + horizontal scale-out | More complex to operate |

**CockroachDB** is a NewSQL database designed to survive node failures without downtime. Its key properties:

- **Distributed storage** — data is split into *ranges* (like partitions) and replicated across nodes using the Raft consensus algorithm
- **Serializable isolation** — the strongest isolation level; no anomalies possible
- **PostgreSQL wire protocol** — any PostgreSQL client library or tool works without modification
- **Geo-partitioning** — data can be pinned to specific regions for latency and compliance
- **Automatic rebalancing** — when you add nodes, CockroachDB redistributes ranges automatically

In this workshop we run a single-node *insecure* cluster, which is the standard approach for development. In production you would run at least 3 nodes across failure domains with TLS enabled.

## CockroachDB vs PostgreSQL

| Feature | PostgreSQL | CockroachDB |
|---------|-----------|-------------|
| SQL dialect | Full SQL:2016 | PostgreSQL-compatible SQL |
| Wire protocol | pgwire | pgwire (identical) |
| Horizontal scale-out | No (read replicas only) | Yes (automatic sharding) |
| Distributed transactions | No | Yes (Raft-based) |
| Default isolation | Read Committed | Serializable |
| Primary key recommendation | SERIAL (auto-increment) | UUID or `gen_random_uuid()` |
| Admin UI | pgAdmin (external) | Built-in at port 8080 |
| Time-travel queries | No | `AS OF SYSTEM TIME` |
| Schema inspection | `\dt`, `\d` meta-commands | `SHOW TABLES`, `SHOW CREATE TABLE` |
| Background jobs | pg_stat_activity | `SHOW JOBS` |

> **Why avoid auto-increment in CockroachDB?** Sequential integer keys create a *write hotspot* — all inserts land on the same range (the one holding the highest key value), preventing writes from distributing across nodes. UUID keys spread inserts randomly across all ranges from day one.

## The 3NF Schema Design

The same 8-table schema from the PostgreSQL workshop applies here unchanged:

```
movie          person
─────────      ────────────────
movie_id (PK)  person_id (PK)
title          name
year           headshot
runtime        birth_date
rating
votes                  person_trademark
plot_outline           ────────────────
cover_url              person_id (PK, FK → person)
rank                   trademark (PK)

genre          language
─────────      ──────────
genre_id (PK)  language_id (PK)
name           code

movie_genre          movie_language
────────────────     ──────────────────
movie_id (PK, FK)    movie_id (PK, FK)
genre_id (PK, FK)    language_id (PK, FK)

movie_person
───────────────────────────────
movie_id  (PK, FK → movie)
person_id (PK, FK → person)
role      (PK)  -- 'actor' | 'director' | 'producer'
```

The only schema difference from PostgreSQL is in the `genre` and `language` tables: instead of `SERIAL` (auto-increment), we use `UUID DEFAULT gen_random_uuid()` to avoid write hotspots in a distributed cluster.

## Connecting to CockroachDB

### Using the cockroach sql CLI

CockroachDB ships with its own SQL shell. Open a terminal and run:

```bash
docker exec -it cockroachdb cockroach sql --insecure
```

You should see the CockroachDB prompt:

```
#
# Welcome to the CockroachDB SQL shell.
# All statements must be terminated by a semicolon.
# To exit, type: \q.
#
root@:26257/defaultdb>
```

> **What you should see:** The `root@:26257/defaultdb>` prompt confirming a successful connection as the `root` user to the default database.

Useful built-in commands:

| Command | Description |
|---------|-------------|
| `SHOW DATABASES;` | List all databases |
| `USE dbname;` | Switch to a database |
| `SHOW TABLES;` | List tables in the current database |
| `SHOW CREATE TABLE tablename;` | Show the DDL for a table |
| `SHOW COLUMNS FROM tablename;` | Show column definitions |
| `SHOW INDEXES FROM tablename;` | Show indexes |
| `SHOW RANGES FROM TABLE tablename;` | Show distributed ranges |
| `SHOW JOBS;` | Show background jobs |
| `\q` | Quit |

### Using psql

Because CockroachDB speaks the PostgreSQL wire protocol, the standard `psql` client also works:

```bash
docker exec -it cockroachdb cockroach sql --insecure --url "postgresql://root@cockroachdb:26257/defaultdb?sslmode=disable"
```

Or from outside the container (if you have `psql` installed locally):

```bash
psql "postgresql://root@dataplatform:26257/defaultdb?sslmode=disable"
```

### Using the Admin UI

Open a browser and navigate to <http://dataplatform:28170>.

![CockroachDB Admin UI](./images/cockroachdb-admin-ui-1.png)

> **What you should see:** The CockroachDB Admin UI dashboard showing cluster health, node status, SQL activity metrics, and a navigation bar on the left.

The Admin UI provides:

- **Overview** — cluster health, replication status, and capacity
- **Metrics** — real-time charts for SQL throughput, latency, and storage
- **Databases** — browse databases, tables, and their range distribution
- **Jobs** — running and completed background jobs (schema changes, imports, backups)
- **SQL Activity** — recent statements with execution plans and latency percentiles

### Using DbGate

In a browser navigate to <http://dataplatform:28120> and log in with user `dbgate` and password `abc123!`.

Click **+ Add new connection**, select `PostgreSQL` as the connection type (CockroachDB is wire-compatible), and enter:

- **Server**: `cockroachdb`
- **Port**: `26257`
- **User**: `root`
- **Password**: *(leave empty — the cluster runs insecure)*
- **Database**: `defaultdb`

Click **Test** and then **Connect**.

> **What you should see:** The DbGate web UI with the CockroachDB connection listed. Expanding it shows the `defaultdb`, `postgres`, and `system` databases. Once you create `filmdb` it will appear here too.

## Creating the Database and Schema

Connect to the CockroachDB SQL shell:

```bash
docker exec -it cockroachdb cockroach sql --insecure
```

### Create the filmdb database

```sql
CREATE DATABASE filmdb;
USE filmdb;
```

```
SET
```

> **What you should see:** `SET` — CockroachDB's confirmation that the active database has changed.

### Create the tables

The DDL is almost identical to PostgreSQL. The key difference is using `UUID DEFAULT gen_random_uuid()` instead of `SERIAL` for the genre and language surrogate keys, which distributes inserts across the cluster from the start.

```sql
-- Controlled vocabulary: genres
CREATE TABLE genre (
    genre_id  UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    name      VARCHAR(50) NOT NULL UNIQUE
);

-- Controlled vocabulary: ISO 639-1 language codes
CREATE TABLE language (
    language_id UUID       PRIMARY KEY DEFAULT gen_random_uuid(),
    code        VARCHAR(5) NOT NULL UNIQUE
);

-- Core movie entity
CREATE TABLE movie (
    movie_id     VARCHAR(10)  PRIMARY KEY,
    title        VARCHAR(255) NOT NULL,
    year         INTEGER,
    runtime      INTEGER,
    rating       DECIMAL(3,1),
    votes        INTEGER,
    plot_outline TEXT,
    cover_url    VARCHAR(500),
    rank         INTEGER
);

-- Core person entity
CREATE TABLE person (
    person_id  VARCHAR(10)  PRIMARY KEY,
    name       VARCHAR(255) NOT NULL,
    headshot   VARCHAR(500),
    birth_date DATE
);

-- Multi-valued attribute: trademarks
CREATE TABLE person_trademark (
    person_id VARCHAR(10) NOT NULL REFERENCES person(person_id) ON DELETE CASCADE,
    trademark TEXT        NOT NULL,
    PRIMARY KEY (person_id, trademark)
);

-- Junction: movie ↔ genre
CREATE TABLE movie_genre (
    movie_id UUID NOT NULL REFERENCES movie(movie_id)  ON DELETE CASCADE,
    genre_id UUID NOT NULL REFERENCES genre(genre_id)  ON DELETE CASCADE,
    PRIMARY KEY (movie_id, genre_id)
);

-- Junction: movie ↔ language
CREATE TABLE movie_language (
    movie_id    VARCHAR(10) NOT NULL REFERENCES movie(movie_id)      ON DELETE CASCADE,
    language_id UUID        NOT NULL REFERENCES language(language_id) ON DELETE CASCADE,
    PRIMARY KEY (movie_id, language_id)
);

-- Junction: movie ↔ person with role discriminator
CREATE TABLE movie_person (
    movie_id  VARCHAR(10) NOT NULL REFERENCES movie(movie_id)  ON DELETE CASCADE,
    person_id VARCHAR(10) NOT NULL REFERENCES person(person_id) ON DELETE CASCADE,
    role      VARCHAR(20) NOT NULL CHECK (role IN ('actor','director','producer')),
    PRIMARY KEY (movie_id, person_id, role)
);
```

Verify:

```sql
SHOW TABLES;
```

```
  schema_name | table_name       | type  | owner | estimated_row_count | locality
--------------+------------------+-------+-------+---------------------+-----------
  public      | genre            | table | root  |                   0 | NULL
  public      | language         | table | root  |                   0 | NULL
  public      | movie            | table | root  |                   0 | NULL
  public      | movie_genre      | table | root  |                   0 | NULL
  public      | movie_language   | table | root  |                   0 | NULL
  public      | movie_person     | table | root  |                   0 | NULL
  public      | person           | table | root  |                   0 | NULL
  public      | person_trademark | table | root  |                   0 | NULL
```

> **What you should see:** All 8 tables with `estimated_row_count: 0`. Notice the `locality` column — in a multi-region cluster this would show which region each table's data is pinned to.

Inspect a table's DDL:

```sql
SHOW CREATE TABLE movie_person;
```

```sql
  table_name  |                       create_statement
--------------+---------------------------------------------------------------
  movie_person | CREATE TABLE public.movie_person (
               |     movie_id VARCHAR(10) NOT NULL,
               |     person_id VARCHAR(10) NOT NULL,
               |     role VARCHAR(20) NOT NULL,
               |     CONSTRAINT movie_person_pkey PRIMARY KEY (movie_id ASC, person_id ASC, role ASC),
               |     CONSTRAINT movie_person_movie_id_fkey FOREIGN KEY (movie_id) REFERENCES public.movie(movie_id) ON DELETE CASCADE,
               |     CONSTRAINT movie_person_person_id_fkey FOREIGN KEY (person_id) REFERENCES public.person(person_id) ON DELETE CASCADE,
               |     CONSTRAINT check_role CHECK (role IN ('actor', 'director', 'producer'))
               | )
```

> **What you should see:** The full CREATE TABLE statement reconstructed from the catalog, including all constraints. CockroachDB names constraints automatically.

## Inserting Data

The INSERT statements are identical to the PostgreSQL workshop. Run each block in order.

### Genres and languages

```sql
INSERT INTO genre (name) VALUES
    ('Action'), ('Adventure'), ('Animation'), ('Biography'),
    ('Comedy'), ('Crime'), ('Drama'), ('Family'),
    ('Fantasy'), ('History'), ('Horror'), ('Music'),
    ('Musical'), ('Mystery'), ('Romance'), ('Sci-Fi'),
    ('Thriller'), ('War'), ('Western');

INSERT INTO language (code) VALUES ('en'), ('es'), ('fr');
```

### Movies

```sql
INSERT INTO movie (movie_id, title, year, runtime, rating, votes, plot_outline, cover_url) VALUES
('0110912', 'Pulp Fiction', 1994, 154, 8.9, 2084331,
 'Jules Winnfield and Vincent Vega are two hit men who are out to retrieve a suitcase stolen from their employer, mob boss Marsellus Wallace. Wallace has also asked Vincent to take his wife Mia out a few days later. Butch Coolidge is an aging boxer who is paid by Wallace to lose his fight. The lives of these seemingly unrelated people are woven together comprising a series of funny, bizarre and uncalled-for incidents.',
 'https://m.media-amazon.com/images/M/MV5BNGNhMDIzZTUtNTBlZi00MTRlLWFjM2ItYzViMjE3YzI5MjljXkEyXkFqcGdeQXVyNzkwMjQ5NzM@._V1_SY150_CR1,0,101,150_.jpg'),
('0133093', 'The Matrix', 1999, 136, 8.7, 1496538,
 'Thomas A. Anderson is a man living two lives. By day he is an average computer programmer and by night a hacker known as Neo. Morpheus awakens Neo to the real world, a ravaged wasteland where most of humanity have been captured by a race of machines.',
 'https://m.media-amazon.com/images/M/MV5BNzQzOTk3OTAtNDQ0Zi00ZTVkLWI0MTEtMDllZjNkYzNjNTc4L2ltYWdlXkEyXkFqcGdeQXVyNjU0OTQ0OTY@._V1_SX101_CR0,0,101,150_.jpg');

INSERT INTO movie (movie_id, title, year, rating, rank) VALUES
('0111161', 'The Shawshank Redemption',                              1994, 9.2,  1),
('0068646', 'The Godfather',                                         1972, 9.2,  2),
('0071562', 'The Godfather: Part II',                                1974, 9.0,  3),
('0468569', 'The Dark Knight',                                       2008, 9.0,  4),
('0050083', '12 Angry Men',                                          1957, 8.9,  5),
('0108052', 'Schindler''s List',                                     1993, 8.9,  6),
('0167260', 'The Lord of the Rings: The Return of the King',         2003, 8.9,  7),
('0060196', 'The Good, the Bad and the Ugly',                        1966, 8.8,  9),
('0137523', 'Fight Club',                                            1999, 8.8, 10),
('4154796', 'Avengers: Endgame',                                     2019, 8.8, 11),
('0120737', 'The Lord of the Rings: The Fellowship of the Ring',     2001, 8.8, 12),
('0109830', 'Forrest Gump',                                          1994, 8.7, 13),
('0080684', 'Star Wars: Episode V - The Empire Strikes Back',        1980, 8.7, 14),
('1375666', 'Inception',                                             2010, 8.7, 15),
('0167261', 'The Lord of the Rings: The Two Towers',                 2002, 8.7, 16),
('0073486', 'One Flew Over the Cuckoo''s Nest',                     1975, 8.7, 17),
('0099685', 'Goodfellas',                                            1990, 8.7, 18),
('0047478', 'Seven Samurai',                                         1954, 8.6, 20),
('0114369', 'Se7en',                                                 1995, 8.6, 21),
('0317248', 'City of God',                                           2002, 8.6, 22),
('0076759', 'Star Wars: Episode IV - A New Hope',                    1977, 8.6, 23),
('0102926', 'The Silence of the Lambs',                              1991, 8.6, 24),
('0038650', 'It''s a Wonderful Life',                                1946, 8.6, 25),
('0118799', 'Life Is Beautiful',                                     1997, 8.6, 26),
('0245429', 'Spirited Away',                                         2001, 8.5, 27),
('0120815', 'Saving Private Ryan',                                   1998, 8.5, 28),
('0114814', 'The Usual Suspects',                                    1995, 8.5, 29),
('0110413', 'Léon: The Professional',                                1994, 8.5, 30),
('0120689', 'The Green Mile',                                        1999, 8.5, 31),
('0816692', 'Interstellar',                                          2014, 8.5, 32),
('0054215', 'Psycho',                                                1960, 8.5, 33),
('0120586', 'American History X',                                    1998, 8.5, 34),
('0021749', 'City Lights',                                           1931, 8.5, 35),
('0034583', 'Casablanca',                                            1942, 8.5, 36),
('0064116', 'Once Upon a Time in the West',                          1968, 8.5, 37),
('0253474', 'The Pianist',                                           2002, 8.5, 38),
('0027977', 'Modern Times',                                          1936, 8.5, 39),
('1675434', 'The Intouchables',                                      2011, 8.5, 40),
('0407887', 'The Departed',                                          2006, 8.5, 41),
('0088763', 'Back to the Future',                                    1985, 8.5, 42),
('0103064', 'Terminator 2: Judgment Day',                            1991, 8.5, 43),
('2582802', 'Whiplash',                                              2014, 8.5, 44),
('0110357', 'The Lion King',                                         1994, 8.5, 45),
('0047396', 'Rear Window',                                           1954, 8.5, 46),
('0082971', 'Raiders of the Lost Ark',                               1981, 8.5, 47),
('0172495', 'Gladiator',                                             2000, 8.5, 48),
('0482571', 'The Prestige',                                          2006, 8.5, 49),
('0078788', 'Apocalypse Now',                                        1979, 8.4, 50);
```

### Persons, trademarks, genres, languages, and cast

These INSERT statements are identical to the PostgreSQL workshop. Copy the following blocks from [Working with PostgreSQL](../01-working-with-postgresql/README.md):

- **Persons** — the `INSERT INTO person` block
- **Person trademarks** — the `INSERT INTO person_trademark` block
- **Movie genres** — the `INSERT INTO movie_genre` blocks (using the name-based lookup)
- **Movie languages** — the `INSERT INTO movie_language` block
- **Movie–person relationships** — all `INSERT INTO movie_person` blocks

> **Note:** CockroachDB is fully SQL-standard here. Every INSERT statement from the PostgreSQL workshop runs without modification.

Verify the row counts:

```sql
SELECT
    'movie'             AS "table", count(*) FROM movie            UNION ALL
SELECT 'person',                              count(*) FROM person           UNION ALL
SELECT 'genre',                               count(*) FROM genre            UNION ALL
SELECT 'language',                            count(*) FROM language         UNION ALL
SELECT 'movie_genre',                         count(*) FROM movie_genre      UNION ALL
SELECT 'movie_language',                      count(*) FROM movie_language   UNION ALL
SELECT 'movie_person',                        count(*) FROM movie_person     UNION ALL
SELECT 'person_trademark',                    count(*) FROM person_trademark;
```

> **What you should see:** The same row counts as in the PostgreSQL workshop.

## Basic Queries

All standard SQL queries from the PostgreSQL workshop run unchanged. Below are the key ones as a reminder.

### Top-rated movies

```sql
SELECT movie_id, title, year, rating
FROM movie
ORDER BY rating DESC NULLS LAST, title
LIMIT 10;
```

### Filter with WHERE

```sql
-- Movies released in 1994
SELECT title, year, rating
FROM movie
WHERE year = 1994
ORDER BY rating DESC NULLS LAST;

-- Case-insensitive title search
SELECT title, year
FROM movie
WHERE title ILIKE '%the%'
ORDER BY year;
```

### Pagination with LIMIT / OFFSET

```sql
SELECT title, year, rating
FROM movie
ORDER BY rating DESC NULLS LAST
LIMIT 10 OFFSET 10;   -- page 2
```

## Joining Tables

All JOIN queries from the PostgreSQL workshop run unchanged in CockroachDB.

### Genres of a movie

```sql
SELECT m.title, g.name AS genre
FROM movie m
JOIN movie_genre mg ON mg.movie_id = m.movie_id
JOIN genre g        ON g.genre_id  = mg.genre_id
WHERE m.title = 'Pulp Fiction'
ORDER BY g.name;
```

### All actors in a movie

```sql
SELECT p.name, mp.role
FROM person p
JOIN movie_person mp ON mp.person_id = p.person_id
JOIN movie m         ON m.movie_id   = mp.movie_id
WHERE m.title = 'Pulp Fiction'
ORDER BY mp.role, p.name;
```

### All films for an actor

```sql
SELECT m.title, m.year
FROM movie m
JOIN movie_person mp ON mp.movie_id  = m.movie_id
JOIN person p        ON p.person_id  = mp.person_id
WHERE p.name = 'Keanu Reeves'
ORDER BY m.year;
```

## Aggregating Data

### Movies per genre

```sql
SELECT g.name AS genre, COUNT(mg.movie_id) AS num_movies
FROM genre g
LEFT JOIN movie_genre mg ON mg.genre_id = g.genre_id
GROUP BY g.genre_id, g.name
ORDER BY num_movies DESC;
```

### Average rating per genre

```sql
SELECT g.name AS genre,
       ROUND(AVG(m.rating), 2) AS avg_rating,
       COUNT(m.movie_id)       AS num_movies
FROM genre g
JOIN movie_genre mg ON mg.genre_id  = g.genre_id
JOIN movie m        ON m.movie_id   = mg.movie_id
WHERE m.rating IS NOT NULL
GROUP BY g.genre_id, g.name
ORDER BY avg_rating DESC;
```

### Most prolific directors

```sql
SELECT p.name, COUNT(DISTINCT mp.movie_id) AS num_films
FROM person p
JOIN movie_person mp ON mp.person_id = p.person_id
WHERE mp.role = 'director'
GROUP BY p.person_id, p.name
ORDER BY num_films DESC
LIMIT 10;
```

## CockroachDB-Specific Features

### Inspecting the Admin UI

Navigate to <http://dataplatform:28170> and explore the **Databases** section.

![CockroachDB Databases view](./images/cockroachdb-admin-ui-2.png)

Click on **filmdb** and then on the **movie** table to see its range distribution, replication factor, and live bytes.

> **What you should see:** The movie table listed with its estimated row count and the number of ranges it occupies. On a single-node cluster all data lives in one range per table.

### SHOW RANGES — understanding data distribution

CockroachDB splits data into *ranges* of roughly 512 MB each and replicates each range to multiple nodes using Raft. You can inspect the ranges for any table:

```sql
SHOW RANGES FROM TABLE movie;
```

```
  start_key | end_key | range_id | replicas | lease_holder
------------+---------+----------+----------+--------------
  NULL      | NULL    |       38 | {1}      |            1
```

> **What you should see:** A single range covering the entire `movie` table (because we only have 59 rows). In a large production table with millions of rows you would see multiple ranges, each held by different nodes.

### SHOW JOBS — background operations

CockroachDB runs schema changes, garbage collection, and imports as background jobs. List recent jobs:

```sql
SHOW JOBS;
```

```
  job_id             | job_type        | description                                     | status    | ...
---------------------+-----------------+-------------------------------------------------+-----------+----
  1006618518...      | SCHEMA CHANGE   | CREATE TABLE filmdb.public.movie (...)          | succeeded | ...
  1006618519...      | SCHEMA CHANGE   | CREATE TABLE filmdb.public.person (...)         | succeeded | ...
  ...
```

> **What you should see:** One completed `SCHEMA CHANGE` job for each `CREATE TABLE` statement. CockroachDB executes schema changes as online, non-blocking operations — tables remain fully accessible while they are being modified.

### AS OF SYSTEM TIME — time-travel queries

CockroachDB retains historical versions of data for a configurable garbage collection window (default 25 hours). You can query data *as it was at a past timestamp*:

First, note the current time:

```sql
SELECT now();
```

```
              now
-------------------------------
 2026-04-14 10:00:00.000000+00
```

Now update a row:

```sql
UPDATE movie SET rating = 9.5 WHERE movie_id = '0137523';  -- Fight Club
SELECT title, rating FROM movie WHERE movie_id = '0137523';
```

```
    title    | rating
-------------+--------
 Fight Club  |    9.5
```

Query the data as it was before the update (substitute your actual timestamp):

```sql
SELECT title, rating
FROM movie AS OF SYSTEM TIME '2026-04-14 09:59:00'
WHERE movie_id = '0137523';
```

```
    title    | rating
-------------+--------
 Fight Club  |    8.8
```

> **What you should see:** The old rating `8.8` returned for the timestamp before the update.
>
> **What just happened?** CockroachDB's MVCC (Multi-Version Concurrency Control) storage keeps past versions of every row. `AS OF SYSTEM TIME` instructs the query engine to read from the historical version at that timestamp instead of the latest one. This is useful for auditing, debugging, and consistent long-running analytics without blocking writers.

Revert the change:

```sql
UPDATE movie SET rating = 8.8 WHERE movie_id = '0137523';
```

### SHOW CLUSTER SETTING — inspecting configuration

```sql
SHOW CLUSTER SETTING kv.range_merge.queue_enabled;
SHOW CLUSTER SETTING server.time_until_store_dead;
```

Check the garbage collection window (how far back `AS OF SYSTEM TIME` can reach):

```sql
SHOW CLUSTER SETTING kv.closed_timestamp.target_duration;
```

### Checking table statistics

CockroachDB automatically collects table statistics to inform the query planner. View them with:

```sql
SHOW STATISTICS FOR TABLE movie;
```

```
  statistics_name | column_names | row_count | distinct_count | null_count | ...
------------------+--------------+-----------+----------------+------------+----
  __auto__        | {movie_id}   |        59 |             59 |          0 | ...
  __auto__        | {title}      |        59 |             59 |          0 | ...
  __auto__        | {year}       |        59 |             30 |          0 | ...
  __auto__        | {rating}     |        59 |             10 |          9 | ...
```

> **What you should see:** Per-column statistics including row count, distinct count, and null count. The query planner uses these to choose join strategies and index usage.

Force a manual statistics refresh:

```sql
ANALYZE movie;
```

## Transactions

CockroachDB provides **serializable** isolation — the strongest guarantee possible — across distributed nodes. Concurrent transactions are automatically serialized as if they ran one at a time.

### Basic transaction

```sql
BEGIN;

UPDATE movie SET votes = votes + 1 WHERE movie_id = '0133093';
UPDATE movie SET votes = votes + 1 WHERE movie_id = '0110912';

COMMIT;
```

> **What you should see:** `COMMIT` — both updates committed atomically. Either both succeed or neither does.

### Transaction rollback

```sql
BEGIN;

DELETE FROM movie WHERE movie_id = '0137523';

-- Oh no, wrong movie — roll back
ROLLBACK;

-- Verify the row is still there
SELECT title FROM movie WHERE movie_id = '0137523';
```

```
   title
-----------
 Fight Club
```

> **What you should see:** Fight Club is still in the table. The DELETE was rolled back and had no effect.

### Savepoints

CockroachDB supports standard SQL savepoints for partial rollbacks within a transaction:

```sql
BEGIN;

INSERT INTO movie (movie_id, title, year) VALUES ('TEST01', 'Test Movie A', 2025);

SAVEPOINT sp1;

INSERT INTO movie (movie_id, title, year) VALUES ('TEST02', 'Test Movie B', 2025);

-- Only roll back to the savepoint — keep TEST01
ROLLBACK TO SAVEPOINT sp1;

COMMIT;

SELECT movie_id, title FROM movie WHERE movie_id IN ('TEST01', 'TEST02');
```

```
 movie_id |    title
----------+--------------
 TEST01   | Test Movie A
```

> **What you should see:** `TEST01` was kept (it was inserted before the savepoint) and `TEST02` was rolled back. Clean up:

```sql
DELETE FROM movie WHERE movie_id = 'TEST01';
```

## Performance with EXPLAIN

### Basic execution plan

```sql
EXPLAIN
SELECT m.title, g.name AS genre
FROM movie m
JOIN movie_genre mg ON mg.movie_id = m.movie_id
JOIN genre g        ON g.genre_id  = mg.genre_id
WHERE g.name = 'Drama';
```

```
                         info
-------------------------------------------------------
  distribution: local
  vectorized: true

  • hash join
  │ equality: (genre_id) = (genre_id)
  │
  ├── • hash join
  │   │ equality: (movie_id) = (movie_id)
  │   │
  ├── • scan
  │   │   table: movie@movie_pkey
  │   │   spans: FULL SCAN
  │   │
  │   └── • scan
  │           table: movie_genre@movie_genre_pkey
  │           spans: FULL SCAN
  │
  └── • filter
      │ filter: name = 'Drama'
      └── • scan
              table: genre@genre_pkey
              spans: FULL SCAN
```

> **What you should see:** The query plan tree showing hash joins and full table scans. `distribution: local` means the plan runs on a single node (our single-node cluster).

### EXPLAIN ANALYZE — actual runtime statistics

```sql
EXPLAIN ANALYZE
SELECT m.title, g.name AS genre
FROM movie m
JOIN movie_genre mg ON mg.movie_id = m.movie_id
JOIN genre g        ON g.genre_id  = mg.genre_id
WHERE g.name = 'Action'
ORDER BY m.title;
```

> **What you should see:** The same plan tree enriched with actual row counts, execution times per operator, and memory usage — equivalent to PostgreSQL's `EXPLAIN ANALYZE`.

### Create an index and observe the difference

```sql
CREATE INDEX idx_movie_title ON movie (title);

EXPLAIN SELECT title, year FROM movie WHERE title = 'The Matrix';
```

```
              info
--------------------------------
  distribution: local
  vectorized: true

  • scan
      table: movie@idx_movie_title
      spans: [/'The Matrix' - /'The Matrix']
```

> **What you should see:** The plan now uses `idx_movie_title` (an *index scan* on a specific span) instead of a full table scan. CockroachDB refers to index scans as scanning a *span* of the key space.

## CockroachDB vs PostgreSQL: a side-by-side comparison

| Operation | PostgreSQL | CockroachDB |
|-----------|-----------|-------------|
| List tables | `\dt` | `SHOW TABLES;` |
| Describe a table | `\d tablename` | `SHOW CREATE TABLE tablename;` |
| List databases | `\l` | `SHOW DATABASES;` |
| Switch database | `\c dbname` | `USE dbname;` |
| Auto-increment PK | `SERIAL` | `UUID DEFAULT gen_random_uuid()` (preferred) |
| Time-travel query | Not available | `AS OF SYSTEM TIME 'timestamp'` |
| Schema change blocking | Locks table | Online, non-blocking |
| Background jobs | `pg_stat_activity` | `SHOW JOBS;` |
| Data distribution | Single node | `SHOW RANGES FROM TABLE t;` |
| Isolation default | Read Committed | Serializable |
| Execution plan | `EXPLAIN ANALYZE` | `EXPLAIN ANALYZE` (same syntax) |
| Table statistics | `ANALYZE tablename` | `ANALYZE tablename` (same syntax) |
| Admin dashboard | pgAdmin (external) | Built-in at port 8080 |

> **Key takeaway:** CockroachDB is a drop-in SQL replacement for PostgreSQL for most applications, while adding horizontal scalability, built-in high availability, and time-travel queries. The primary operational differences are in schema inspection commands and the recommendation to use UUID primary keys for write-heavy distributed workloads.
