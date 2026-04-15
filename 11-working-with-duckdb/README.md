# Working with DuckDB

In this workshop we will learn how to work with **DuckDB**, an in-process analytical database engine. We will use the same film dataset as the PostgreSQL workshop, but model it as a **star schema** to showcase DuckDB's OLAP capabilities — window functions, PIVOT, direct file querying, and vectorised execution.

> **No Docker required.** DuckDB runs embedded inside Python. The entire workshop runs inside the **Jupyter** environment already part of the platform.

## Table of Contents

- [What you will learn](#what-you-will-learn)
- [Prerequisites](#prerequisites)
- [Background: OLAP, Columnar Storage, and the Star Schema](#background-olap-columnar-storage-and-the-star-schema)
- [The Star Schema Design](#the-star-schema-design)
- [3NF vs Star Schema: key differences](#3nf-vs-star-schema-key-differences)
- [Setting Up DuckDB](#setting-up-duckdb)
- [Creating the Schema](#creating-the-schema)
- [Loading Data](#loading-data)
- [Exploring the Schema](#exploring-the-schema)
- [Analytical Queries](#analytical-queries)
- [Window Functions](#window-functions)
- [Common Table Expressions (CTEs)](#common-table-expressions-ctes)
- [PIVOT and UNPIVOT](#pivot-and-unpivot)
- [Querying Files Directly](#querying-files-directly)
- [Python and Pandas Integration](#python-and-pandas-integration)
- [Understanding Vectorised Execution](#understanding-vectorised-execution)

## What you will learn

- How OLAP workloads differ from OLTP and why columnar storage matters
- How a star schema differs from a 3NF relational schema
- How to create and query a persistent DuckDB database from Jupyter
- How to populate a time dimension using `generate_series`
- How to write analytical SQL: `GROUP BY`, `ROLLUP`, window functions, CTEs, and `PIVOT`
- How to query CSV and Parquet files directly without loading them
- How to pass results between DuckDB and Pandas DataFrames
- How to read a DuckDB query plan to understand vectorised execution

## Prerequisites

- The **Data Platform** described [here](../00-environment/README.md) is running and accessible
- Familiarity with the star schema concept is helpful but not required

## Background: OLAP, Columnar Storage, and the Star Schema

### OLTP vs OLAP

| Property | OLTP (e.g. PostgreSQL) | OLAP (e.g. DuckDB) |
|----------|----------------------|-------------------|
| Workload | Many short reads/writes | Few large aggregation queries |
| Storage | Row-oriented | Column-oriented |
| Optimised for | Single-row lookup | Scanning millions of rows fast |
| Schema style | 3NF (normalised) | Star / snowflake (denormalised) |
| Concurrency | High (thousands of users) | Low (analysts, batch jobs) |
| Typical use | Operational applications | Reporting, data science, BI |

### Columnar storage

In a **row store** (PostgreSQL), all columns of a row are stored together on disk. Reading just one column requires scanning the entire row. In a **column store** (DuckDB), all values of one column are stored contiguously. An aggregation like `AVG(rating)` reads only the `rating` column — skipping all others entirely.

```
Row store layout:           Column store layout:
────────────────────        ──────────────────────
| Shawshank | 1994 | 9.2 |  rating: | 9.2 | 9.2 | 9.0 | 9.0 | 8.9 | …
| Godfather | 1972 | 9.2 |  year:   |1994 |1972 |1974 |2008 |1957 | …
| Dark Knt  | 2008 | 9.0 |  title:  | … string heap …
```

Column stores also compress far better, because consecutive values in one column tend to be similar.

### Star schema

A **star schema** organises an analytical database into:

- A central **fact table** — one row per measurable event (here: one row per movie), containing numeric measures and foreign keys to dimensions
- Surrounding **dimension tables** — descriptive context for the facts (year, genre, person, role)
- **Bridge tables** — handle many-to-many relationships between facts and dimensions

The schema looks like a star when drawn: the fact table in the centre, dimension tables radiating out. Queries join the fact to one or more dimensions, then aggregate the measures.

## The Star Schema Design

```
                    dim_genre
                        |
               bridge_movie_genre
                        |
dim_year ──────── fact_movie ──────── (title, plot_outline, cover_url
  (year,                |              are degenerate dimensions —
  decade,        bridge_movie_person   stored in the fact row)
  era)               /       \
              dim_person    dim_role
              (name,        (actor /
              birth_year,   director /
              birth_decade) producer)
```

### Tables

| Table | Type | Grain / Purpose |
|-------|------|-----------------|
| `fact_movie` | Fact | One row per movie; measures: rating, votes, runtime, rank |
| `dim_year` | Dimension | One row per year 1920–2030; adds decade and era |
| `dim_genre` | Dimension | One row per genre name |
| `dim_person` | Dimension | One row per person (actor, director, or producer) |
| `dim_role` | Dimension | Three rows: actor, director, producer |
| `bridge_movie_genre` | Bridge | Resolves movie ↔ genre many-to-many |
| `bridge_movie_person` | Bridge | Resolves movie ↔ person ↔ role many-to-many |

## 3NF vs Star Schema: key differences

| Aspect | PostgreSQL 3NF | DuckDB Star Schema |
|--------|---------------|-------------------|
| Year stored as | Integer column on `movie` | FK into `dim_year` with `decade` and `era` attributes |
| Role stored as | `CHECK` constraint in `movie_person` | Separate `dim_role` dimension — queryable and joinable |
| Genre IDs | Auto-increment SERIAL | Auto-increment SERIAL (same, different purpose) |
| Query style | Precise point lookups + joins | Wide aggregations + dimension traversal |
| NULL handling | Explicit column constraint | Measures default to NULL; dimensions always populated |
| Redundancy | None (fully normalised) | Controlled denormalisation in fact row (title, plot_outline) |

## Setting Up DuckDB

Open a browser and navigate to <http://dataplatform:28888>. Create a new Python 3 notebook. Work through the cells below in order.

### Cell 1 — Install the library

```python
import sys
!{sys.executable} -m pip install duckdb
```

> **What you should see:** `Successfully installed duckdb-…`

### Cell 2 — Create a persistent connection

```python
import duckdb

# Persist the database to the shared data-transfer volume so it
# survives Jupyter restarts. Use ':memory:' for a temporary session.
conn = duckdb.connect('/data-transfer/filmdb.duckdb')
print(duckdb.__version__)
```

> **What you should see:** The installed DuckDB version (e.g. `1.x.x`).
>
> **What just happened?** DuckDB created a single-file database at `/data-transfer/filmdb.duckdb`. There is no server process — the engine runs entirely inside the Python interpreter.

## Creating the Schema

### Cell 3 — Dimension tables

```python
conn.executemany("DROP TABLE IF EXISTS {t}".format(t=t), [
    ["bridge_movie_person"], ["bridge_movie_genre"],
    ["fact_movie"], ["dim_year"], ["dim_genre"], ["dim_person"], ["dim_role"],
])

conn.execute("""
CREATE TABLE dim_year (
    year_key  INTEGER PRIMARY KEY,
    year      INTEGER NOT NULL,
    decade    INTEGER NOT NULL,
    era       VARCHAR NOT NULL   -- 'Classic' | 'Modern' | 'Contemporary'
)
""")

conn.execute("""
CREATE TABLE dim_genre (
    genre_key  INTEGER PRIMARY KEY,
    name       VARCHAR NOT NULL UNIQUE
)
""")

conn.execute("""
CREATE TABLE dim_person (
    person_key   INTEGER PRIMARY KEY,
    person_id    VARCHAR NOT NULL UNIQUE,
    name         VARCHAR NOT NULL,
    birth_date   DATE,
    birth_year   INTEGER,
    birth_decade INTEGER
)
""")

conn.execute("""
CREATE TABLE dim_role (
    role_key  INTEGER PRIMARY KEY,
    name      VARCHAR NOT NULL UNIQUE
)
""")
```

> **What you should see:** No error — all four dimension tables created.

### Cell 4 — Fact and bridge tables

```python
conn.execute("""
CREATE TABLE fact_movie (
    movie_id     VARCHAR  PRIMARY KEY,   -- natural/degenerate dimension key
    title        VARCHAR  NOT NULL,
    year_key     INTEGER  REFERENCES dim_year(year_key),
    plot_outline TEXT,
    cover_url    VARCHAR,
    rating       DECIMAL(3,1),           -- MEASURE
    votes        INTEGER,                -- MEASURE
    runtime      INTEGER,                -- MEASURE (minutes)
    rank         INTEGER                 -- MEASURE
)
""")

conn.execute("""
CREATE TABLE bridge_movie_genre (
    movie_id  VARCHAR REFERENCES fact_movie(movie_id),
    genre_key INTEGER REFERENCES dim_genre(genre_key),
    PRIMARY KEY (movie_id, genre_key)
)
""")

conn.execute("""
CREATE TABLE bridge_movie_person (
    movie_id   VARCHAR REFERENCES fact_movie(movie_id),
    person_key INTEGER REFERENCES dim_person(person_key),
    role_key   INTEGER REFERENCES dim_role(role_key),
    PRIMARY KEY (movie_id, person_key, role_key)
)
""")

print("Schema created.")
```

> **What you should see:** `Schema created.`

## Loading Data

### Cell 5 — Populate dim_year using generate_series

```python
conn.execute("""
INSERT INTO dim_year (year_key, year, decade, era)
SELECT
    y                                    AS year_key,
    y                                    AS year,
    (y // 10) * 10                       AS decade,
    CASE
        WHEN y < 1960 THEN 'Classic'
        WHEN y < 2000 THEN 'Modern'
        ELSE 'Contemporary'
    END                                  AS era
FROM generate_series(1920, 2030) AS t(y)
""")
print(conn.execute("SELECT COUNT(*) FROM dim_year").fetchone()[0], "years loaded")
```

```
111 years loaded
```

> **What you should see:** `111 years loaded`.
>
> **What just happened?** `generate_series(1920, 2030)` produced integers 1920 to 2030 inclusive — no loop, no manual INSERT. DuckDB evaluated the expression in a single vectorised pass and inserted all 111 rows at once.

### Cell 6 — Populate dim_genre and dim_role

```python
conn.execute("""
INSERT INTO dim_genre (genre_key, name) VALUES
    (1,'Action'),    (2,'Adventure'),  (3,'Animation'), (4,'Biography'),
    (5,'Comedy'),    (6,'Crime'),      (7,'Drama'),     (8,'Family'),
    (9,'Fantasy'),  (10,'History'),   (11,'Horror'),   (12,'Music'),
   (13,'Musical'),  (14,'Mystery'),   (15,'Romance'),  (16,'Sci-Fi'),
   (17,'Thriller'), (18,'War'),       (19,'Western')
""")

conn.execute("""
INSERT INTO dim_role (role_key, name) VALUES
    (1, 'actor'), (2, 'director'), (3, 'producer')
""")

print("Genres:", conn.execute("SELECT COUNT(*) FROM dim_genre").fetchone()[0])
print("Roles: ", conn.execute("SELECT COUNT(*) FROM dim_role").fetchone()[0])
```

### Cell 7 — Populate fact_movie

```python
conn.execute("""
INSERT INTO fact_movie (movie_id, title, year_key, rating, votes, runtime, plot_outline, cover_url) VALUES
('0110912','Pulp Fiction',            1994, 8.9, 2084331, 154,
 'Jules Winnfield and Vincent Vega are two hit men out to retrieve a suitcase stolen from their employer. Butch Coolidge is an aging boxer paid to lose his fight. The lives of these seemingly unrelated people are woven together in a series of funny, bizarre incidents.',
 'https://m.media-amazon.com/images/M/MV5BNGNhMDIzZTUtNTBlZi00MTRlLWFjM2ItYzViMjE3YzI5MjljXkEyXkFqcGdeQXVyNzkwMjQ5NzM@._V1_SY150_CR1,0,101,150_.jpg'),
('0133093','The Matrix',              1999, 8.7, 1496538, 136,
 'Thomas Anderson lives two lives: by day a computer programmer, by night a hacker called Neo. Morpheus awakens Neo to the real world — a ravaged wasteland where machines have enslaved humanity inside a simulated reality.',
 'https://m.media-amazon.com/images/M/MV5BNzQzOTk3OTAtNDQ0Zi00ZTVkLWI0MTEtMDllZjNkYzNjNTc4L2ltYWdlXkEyXkFqcGdeQXVyNjU0OTQ0OTY@._V1_SX101_CR0,0,101,150_.jpg')
""")

conn.execute("""
INSERT INTO fact_movie (movie_id, title, year_key, rating, rank) VALUES
('0111161','The Shawshank Redemption',                          1994, 9.2,  1),
('0068646','The Godfather',                                     1972, 9.2,  2),
('0071562','The Godfather: Part II',                            1974, 9.0,  3),
('0468569','The Dark Knight',                                   2008, 9.0,  4),
('0050083','12 Angry Men',                                      1957, 8.9,  5),
('0108052','Schindler''s List',                                 1993, 8.9,  6),
('0167260','The Lord of the Rings: The Return of the King',     2003, 8.9,  7),
('0060196','The Good, the Bad and the Ugly',                    1966, 8.8,  9),
('0137523','Fight Club',                                        1999, 8.8, 10),
('4154796','Avengers: Endgame',                                 2019, 8.8, 11),
('0120737','The Lord of the Rings: The Fellowship of the Ring', 2001, 8.8, 12),
('0109830','Forrest Gump',                                      1994, 8.7, 13),
('0080684','Star Wars: Episode V - The Empire Strikes Back',    1980, 8.7, 14),
('1375666','Inception',                                         2010, 8.7, 15),
('0167261','The Lord of the Rings: The Two Towers',             2002, 8.7, 16),
('0073486','One Flew Over the Cuckoo''s Nest',                  1975, 8.7, 17),
('0099685','Goodfellas',                                        1990, 8.7, 18),
('0047478','Seven Samurai',                                     1954, 8.6, 20),
('0114369','Se7en',                                             1995, 8.6, 21),
('0317248','City of God',                                       2002, 8.6, 22),
('0076759','Star Wars: Episode IV - A New Hope',                1977, 8.6, 23),
('0102926','The Silence of the Lambs',                          1991, 8.6, 24),
('0038650','It''s a Wonderful Life',                            1946, 8.6, 25),
('0118799','Life Is Beautiful',                                 1997, 8.6, 26),
('0245429','Spirited Away',                                     2001, 8.5, 27),
('0120815','Saving Private Ryan',                               1998, 8.5, 28),
('0114814','The Usual Suspects',                                1995, 8.5, 29),
('0110413','Léon: The Professional',                            1994, 8.5, 30),
('0120689','The Green Mile',                                    1999, 8.5, 31),
('0816692','Interstellar',                                      2014, 8.5, 32),
('0054215','Psycho',                                            1960, 8.5, 33),
('0120586','American History X',                                1998, 8.5, 34),
('0021749','City Lights',                                       1931, 8.5, 35),
('0034583','Casablanca',                                        1942, 8.5, 36),
('0064116','Once Upon a Time in the West',                      1968, 8.5, 37),
('0253474','The Pianist',                                       2002, 8.5, 38),
('0027977','Modern Times',                                      1936, 8.5, 39),
('1675434','The Intouchables',                                  2011, 8.5, 40),
('0407887','The Departed',                                      2006, 8.5, 41),
('0088763','Back to the Future',                                1985, 8.5, 42),
('0103064','Terminator 2: Judgment Day',                        1991, 8.5, 43),
('2582802','Whiplash',                                          2014, 8.5, 44),
('0110357','The Lion King',                                     1994, 8.5, 45),
('0047396','Rear Window',                                       1954, 8.5, 46),
('0082971','Raiders of the Lost Ark',                           1981, 8.5, 47),
('0172495','Gladiator',                                         2000, 8.5, 48),
('0482571','The Prestige',                                      2006, 8.5, 49),
('0078788','Apocalypse Now',                                    1979, 8.4, 50),
('1606378','A Good Day to Die Hard',                            2013, NULL, NULL),
('0217869','Unbreakable',                                       2000, NULL, NULL),
('0377917','The Fifth Element',                                 1997, NULL, NULL),
('0112864','Die Hard: With a Vengeance',                        1995, NULL, NULL),
('0234215','The Matrix Reloaded',                               2003, NULL, NULL),
('0111257','Speed',                                             1994, NULL, NULL),
('2737304','Bird Box',                                          2018, NULL, NULL),
('0120179','Speed 2: Cruise Control',                           1997, NULL, NULL),
('0212346','Miss Congeniality',                                 2000, NULL, NULL),
('0378194','Kill Bill: Vol. 2',                                 2004, NULL, NULL),
('0116367','From Dusk Till Dawn',                               1996, NULL, NULL),
('0119396','Jackie Brown',                                      1997, NULL, NULL)
""")

print("Movies:", conn.execute("SELECT COUNT(*) FROM fact_movie").fetchone()[0])
```

### Cell 8 — Populate dim_person

```python
conn.execute("""
INSERT INTO dim_person (person_key, person_id, name, birth_date, birth_year, birth_decade) VALUES
  (1,'0000246','Bruce Willis',       '1955-03-19', 1955, 1950),
  (2,'0000206','Keanu Reeves',        NULL,         NULL, NULL),
  (3,'0000113','Sandra Bullock',     '1964-07-26', 1964, 1960),
  (4,'0000233','Quentin Tarantino',  '1963-03-27', 1963, 1960),
  (5,'0000619','Tim Roth',            NULL,         NULL, NULL),
  (6,'0001625','Amanda Plummer',      NULL,         NULL, NULL),
  (7,'0522503','Laura Lovelace',      NULL,         NULL, NULL),
  (8,'0000237','John Travolta',       NULL,         NULL, NULL),
  (9,'0000168','Samuel L. Jackson',   NULL,         NULL, NULL),
 (10,'0482851','Phil LaMarr',         NULL,         NULL, NULL),
 (11,'0001844','Frank Whaley',        NULL,         NULL, NULL),
 (12,'0824882','Burr Steers',         NULL,         NULL, NULL),
 (13,'0000609','Ving Rhames',         NULL,         NULL, NULL),
 (14,'0000235','Uma Thurman',         NULL,         NULL, NULL),
 (15,'0004744','Lawrence Bender',     NULL,         NULL, NULL),
 (16,'0000362','Danny DeVito',        NULL,         NULL, NULL),
 (17,'0321621','Richard N. Gladstein',NULL,         NULL, NULL),
 (18,'0787834','Michael Shamberg',    NULL,         NULL, NULL),
 (19,'0792049','Stacey Sher',         NULL,         NULL, NULL),
 (20,'0918424','Bob Weinstein',       NULL,         NULL, NULL),
 (21,'0005544','Harvey Weinstein',    NULL,         NULL, NULL),
 (22,'0000401','Laurence Fishburne',  NULL,         NULL, NULL),
 (23,'0005251','Carrie-Anne Moss',    NULL,         NULL, NULL),
 (24,'0915989','Hugo Weaving',        NULL,         NULL, NULL),
 (25,'0287825','Gloria Foster',       NULL,         NULL, NULL),
 (26,'0001592','Joe Pantoliano',      NULL,         NULL, NULL),
 (27,'0159059','Marcus Chong',        NULL,         NULL, NULL),
 (28,'0032810','Julian Arahanga',     NULL,         NULL, NULL),
 (29,'0905154','Lana Wachowski',      NULL,         NULL, NULL),
 (30,'0905152','Lilly Wachowski',     NULL,         NULL, NULL),
 (31,'0075732','Bruce Berman',        NULL,         NULL, NULL),
 (32,'0185621','Dan Cracchiolo',      NULL,         NULL, NULL),
 (33,'0400492','Carol Hughes',        NULL,         NULL, NULL)
""")
print("Persons:", conn.execute("SELECT COUNT(*) FROM dim_person").fetchone()[0])
```

### Cell 9 — Populate bridge_movie_genre

```python
conn.execute("""
INSERT INTO bridge_movie_genre (movie_id, genre_key)
SELECT t.movie_id, g.genre_key
FROM (VALUES
    ('0110912','Crime'),    ('0110912','Drama'),
    ('0133093','Action'),   ('0133093','Sci-Fi'),
    ('0111161','Drama'),
    ('0068646','Crime'),    ('0068646','Drama'),
    ('0071562','Crime'),    ('0071562','Drama'),
    ('0468569','Action'),   ('0468569','Crime'),   ('0468569','Drama'),   ('0468569','Thriller'),
    ('0050083','Drama'),
    ('0108052','Biography'),('0108052','Drama'),   ('0108052','History'),
    ('0167260','Adventure'),('0167260','Drama'),   ('0167260','Fantasy'),
    ('0060196','Western'),
    ('0137523','Drama'),
    ('4154796','Action'),   ('4154796','Adventure'),('4154796','Fantasy'),('4154796','Sci-Fi'),
    ('0120737','Adventure'),('0120737','Drama'),   ('0120737','Fantasy'),
    ('0109830','Drama'),    ('0109830','Romance'),
    ('0080684','Action'),   ('0080684','Adventure'),('0080684','Fantasy'),('0080684','Sci-Fi'),
    ('1375666','Action'),   ('1375666','Adventure'),('1375666','Sci-Fi'), ('1375666','Thriller'),
    ('0167261','Adventure'),('0167261','Drama'),   ('0167261','Fantasy'),
    ('0073486','Drama'),
    ('0099685','Biography'),('0099685','Crime'),   ('0099685','Drama'),
    ('0047478','Adventure'),('0047478','Drama'),
    ('0114369','Crime'),    ('0114369','Drama'),   ('0114369','Mystery'), ('0114369','Thriller'),
    ('0317248','Crime'),    ('0317248','Drama'),
    ('0076759','Action'),   ('0076759','Adventure'),('0076759','Fantasy'),('0076759','Sci-Fi'),
    ('0102926','Crime'),    ('0102926','Drama'),   ('0102926','Thriller'),
    ('0038650','Drama'),    ('0038650','Family'),  ('0038650','Fantasy'),
    ('0118799','Comedy'),   ('0118799','Drama'),   ('0118799','Romance'), ('0118799','War'),
    ('0245429','Animation'),('0245429','Adventure'),('0245429','Family'), ('0245429','Fantasy'),('0245429','Mystery'),
    ('0120815','Drama'),    ('0120815','War'),
    ('0114814','Crime'),    ('0114814','Mystery'), ('0114814','Thriller'),
    ('0110413','Action'),   ('0110413','Crime'),   ('0110413','Drama'),   ('0110413','Thriller'),
    ('0120689','Crime'),    ('0120689','Drama'),   ('0120689','Fantasy'), ('0120689','Mystery'),
    ('0816692','Adventure'),('0816692','Drama'),   ('0816692','Sci-Fi'),
    ('0054215','Horror'),   ('0054215','Mystery'), ('0054215','Thriller'),
    ('0120586','Drama'),
    ('0021749','Comedy'),   ('0021749','Drama'),   ('0021749','Romance'),
    ('0034583','Drama'),    ('0034583','Romance'), ('0034583','War'),
    ('0064116','Western'),
    ('0253474','Biography'),('0253474','Drama'),   ('0253474','Music'),   ('0253474','War'),
    ('0027977','Comedy'),   ('0027977','Drama'),   ('0027977','Family'),  ('0027977','Romance'),
    ('1675434','Biography'),('1675434','Comedy'),  ('1675434','Drama'),
    ('0407887','Crime'),    ('0407887','Drama'),   ('0407887','Thriller'),
    ('0088763','Adventure'),('0088763','Comedy'),  ('0088763','Sci-Fi'),
    ('0103064','Action'),   ('0103064','Sci-Fi'),
    ('2582802','Drama'),    ('2582802','Music'),
    ('0110357','Animation'),('0110357','Adventure'),('0110357','Drama'),  ('0110357','Family'),('0110357','Musical'),
    ('0047396','Mystery'),  ('0047396','Thriller'),
    ('0082971','Action'),   ('0082971','Adventure'),
    ('0172495','Action'),   ('0172495','Adventure'),('0172495','Drama'),
    ('0482571','Drama'),    ('0482571','Mystery'),  ('0482571','Sci-Fi'), ('0482571','Thriller'),
    ('0078788','Drama'),    ('0078788','War')
) AS t(movie_id, genre_name)
JOIN dim_genre g ON g.name = t.genre_name
""")
print("Movie-genre links:", conn.execute("SELECT COUNT(*) FROM bridge_movie_genre").fetchone()[0])
```

### Cell 10 — Populate bridge_movie_person

```python
conn.execute("""
INSERT INTO bridge_movie_person (movie_id, person_key, role_key)
SELECT t.movie_id, p.person_key, r.role_key
FROM (VALUES
    ('0110912','0000619','actor'), ('0110912','0001625','actor'),
    ('0110912','0522503','actor'), ('0110912','0000237','actor'),
    ('0110912','0000168','actor'), ('0110912','0482851','actor'),
    ('0110912','0001844','actor'), ('0110912','0824882','actor'),
    ('0110912','0000246','actor'), ('0110912','0000609','actor'),
    ('0110912','0000235','actor'), ('0110912','0000233','actor'),
    ('0110912','0000233','director'),
    ('0110912','0004744','producer'),('0110912','0000362','producer'),
    ('0110912','0321621','producer'),('0110912','0787834','producer'),
    ('0110912','0792049','producer'),('0110912','0918424','producer'),
    ('0110912','0005544','producer'),
    ('0133093','0000206','actor'), ('0133093','0000401','actor'),
    ('0133093','0005251','actor'), ('0133093','0915989','actor'),
    ('0133093','0287825','actor'), ('0133093','0001592','actor'),
    ('0133093','0159059','actor'), ('0133093','0032810','actor'),
    ('0133093','0905154','director'),('0133093','0905152','director'),
    ('0133093','0075732','producer'),('0133093','0185621','producer'),
    ('0133093','0400492','producer'),
    ('1606378','0000246','actor'), ('0217869','0000246','actor'),
    ('0377917','0000246','actor'), ('0112864','0000246','actor'),
    ('0234215','0000206','actor'), ('0111257','0000206','actor'),
    ('2737304','0000113','actor'), ('0120179','0000113','actor'),
    ('0111257','0000113','actor'), ('0212346','0000113','actor'),
    ('0378194','0000233','actor'), ('0116367','0000233','actor'),
    ('0119396','0000233','director')
) AS t(movie_id, person_id, role_name)
JOIN dim_person p ON p.person_id  = t.person_id
JOIN dim_role   r ON r.name       = t.role_name
""")
print("Movie-person links:", conn.execute("SELECT COUNT(*) FROM bridge_movie_person").fetchone()[0])
```

### Cell 11 — Verify row counts

```python
counts = conn.execute("""
SELECT 'fact_movie'          AS "table", COUNT(*) AS rows FROM fact_movie           UNION ALL
SELECT 'dim_year',                       COUNT(*) FROM dim_year                     UNION ALL
SELECT 'dim_genre',                      COUNT(*) FROM dim_genre                    UNION ALL
SELECT 'dim_person',                     COUNT(*) FROM dim_person                   UNION ALL
SELECT 'dim_role',                       COUNT(*) FROM dim_role                     UNION ALL
SELECT 'bridge_movie_genre',             COUNT(*) FROM bridge_movie_genre           UNION ALL
SELECT 'bridge_movie_person',            COUNT(*) FROM bridge_movie_person
""").fetchdf()
print(counts.to_string(index=False))
```

```
         table  rows
     fact_movie    62
       dim_year   111
      dim_genre    19
     dim_person    33
       dim_role     3
bridge_movie_genre   139
bridge_movie_person   46
```

## Exploring the Schema

### Cell 12 — SHOW TABLES and DESCRIBE

```python
# List all tables
print(conn.execute("SHOW TABLES").fetchdf().to_string(index=False))
```

```python
# Inspect the fact table structure
print(conn.execute("DESCRIBE fact_movie").fetchdf().to_string(index=False))
```

> **What you should see:** Column names, types, nullability, and key information — DuckDB's equivalent of PostgreSQL's `\d tablename`.

### Cell 13 — SUMMARIZE: instant column statistics

```python
conn.execute("SUMMARIZE fact_movie").fetchdf()
```

```
  column_name  column_type   min    max   approx_unique   avg    std   q25   q50   q75  count  null_percentage
  movie_id     VARCHAR       …      …          62         …      …     …     …     …    62     0.0
  title        VARCHAR       …      …          62         …      …     …     …     …    62     0.0
  year_key     INTEGER       1931   2019       45         …      …     …     …     …    62     0.0
  rating       DECIMAL(3,1)  8.4    9.2        10         8.58   0.12  8.5   8.5   8.7  50     19.4
  votes        INTEGER       …      …          2          …      …     …     …     …    2      96.8
  runtime      INTEGER       136    154        2          145    …     …     …     …    2      96.8
  rank         INTEGER       1      50         50         …      …     …     …     …    50     19.4
```

> **What you should see:** One row per column with min, max, approximate distinct count, mean, standard deviation, and quartiles — computed in a single pass. This is one of DuckDB's most useful exploratory commands; there is no equivalent in standard PostgreSQL.

## Analytical Queries

### Cell 14 — Top movies by rating

```python
conn.execute("""
SELECT title, year_key AS year, rating, rank
FROM fact_movie
WHERE rating IS NOT NULL
ORDER BY rating DESC, title
LIMIT 10
""").fetchdf()
```

### Cell 15 — Movies per era

```python
conn.execute("""
SELECT y.era,
       COUNT(f.movie_id)       AS num_movies,
       ROUND(AVG(f.rating), 2) AS avg_rating,
       MIN(f.rating)           AS min_rating,
       MAX(f.rating)           AS max_rating
FROM fact_movie f
JOIN dim_year y ON y.year_key = f.year_key
WHERE f.rating IS NOT NULL
GROUP BY y.era
ORDER BY y.era
""").fetchdf()
```

```
           era  num_movies  avg_rating  min_rating  max_rating
       Classic          10        8.62         8.4         9.2
        Modern          27        8.59         8.4         9.0
 Contemporary          13        8.59         8.5         9.0
```

> **What you should see:** The three eras defined in `dim_year` with aggregate statistics. This query joins from the fact table to the time dimension — a classic star schema query pattern.

### Cell 16 — Movies per genre (sorted)

```python
conn.execute("""
SELECT g.name AS genre,
       COUNT(DISTINCT f.movie_id)    AS num_movies,
       ROUND(AVG(f.rating), 2)       AS avg_rating
FROM fact_movie f
JOIN bridge_movie_genre b ON b.movie_id  = f.movie_id
JOIN dim_genre          g ON g.genre_key = b.genre_key
WHERE f.rating IS NOT NULL
GROUP BY g.genre_key, g.name
ORDER BY num_movies DESC
""").fetchdf()
```

### Cell 17 — ROLLUP: genre totals and grand total

`ROLLUP` extends `GROUP BY` to automatically add subtotals and a grand total row — essential for summary reports.

```python
conn.execute("""
SELECT
    COALESCE(g.name, '*** ALL GENRES ***') AS genre,
    COUNT(DISTINCT f.movie_id)             AS num_movies,
    ROUND(AVG(f.rating), 2)                AS avg_rating
FROM fact_movie f
JOIN bridge_movie_genre b ON b.movie_id  = f.movie_id
JOIN dim_genre          g ON g.genre_key = b.genre_key
WHERE f.rating IS NOT NULL
GROUP BY ROLLUP(g.name)
ORDER BY num_movies DESC NULLS LAST
""").fetchdf()
```

> **What you should see:** One row per genre plus a final `*** ALL GENRES ***` row with the grand total count and overall average rating.
>
> **What just happened?** `GROUP BY ROLLUP(g.name)` generates two grouping sets: one with `g.name` and one with `NULL` (the grand total). `COALESCE` replaces that NULL with a readable label.

### Cell 18 — Cross-dimensional: era × genre

```python
conn.execute("""
SELECT y.era,
       g.name                        AS genre,
       COUNT(DISTINCT f.movie_id)    AS num_movies,
       ROUND(AVG(f.rating), 2)       AS avg_rating
FROM fact_movie f
JOIN dim_year           y ON y.year_key  = f.year_key
JOIN bridge_movie_genre b ON b.movie_id  = f.movie_id
JOIN dim_genre          g ON g.genre_key = b.genre_key
WHERE f.rating IS NOT NULL
GROUP BY y.era, g.name
ORDER BY y.era, num_movies DESC
""").fetchdf()
```

> **What you should see:** Each era–genre combination with its movie count and average rating — a two-dimensional slice through the data using two dimension tables simultaneously.

## Window Functions

Window functions compute values *across a set of rows related to the current row* without collapsing them into groups. They are one of the most powerful tools in analytical SQL.

### Cell 19 — Rank movies within each decade

```python
conn.execute("""
SELECT
    f.title,
    y.decade,
    f.rating,
    RANK()       OVER (PARTITION BY y.decade ORDER BY f.rating DESC) AS rank_in_decade,
    ROUND(AVG(f.rating) OVER (PARTITION BY y.decade), 2)             AS avg_decade_rating,
    ROUND(f.rating - AVG(f.rating) OVER (PARTITION BY y.decade), 2) AS delta_from_avg
FROM fact_movie f
JOIN dim_year y ON y.year_key = f.year_key
WHERE f.rating IS NOT NULL
ORDER BY y.decade, rank_in_decade
""").fetchdf()
```

> **What you should see:** Each movie with its rank within its decade, the decade's average rating, and how far the movie deviates from that average. The `PARTITION BY y.decade` clause creates a separate window per decade.

### Cell 20 — NTILE: split movies into quartiles by rating

```python
conn.execute("""
SELECT
    f.title,
    f.rating,
    NTILE(4) OVER (ORDER BY f.rating DESC) AS quartile
FROM fact_movie f
WHERE f.rating IS NOT NULL
ORDER BY f.rating DESC
""").fetchdf()
```

> **What you should see:** Movies divided into four equal-size groups (Q1 = highest rated). `NTILE(4)` assigns each row a bucket number from 1 to 4.

### Cell 21 — LAG and LEAD: compare consecutive ranked movies

```python
conn.execute("""
SELECT
    rank,
    title,
    rating,
    LAG(rating)  OVER (ORDER BY rank) AS prev_rating,
    LEAD(rating) OVER (ORDER BY rank) AS next_rating,
    ROUND(rating - LAG(rating) OVER (ORDER BY rank), 1) AS diff_from_prev
FROM fact_movie
WHERE rank IS NOT NULL
ORDER BY rank
LIMIT 15
""").fetchdf()
```

> **What you should see:** Each ranked movie alongside the rating of the movie ranked just above and just below it, and the delta from the previous rank.
>
> **What just happened?** `LAG(rating)` looks back one row in the window (ordered by rank), `LEAD(rating)` looks forward one row — all without a self-join.

### Cell 22 — Cumulative sum of votes

```python
conn.execute("""
SELECT
    title,
    year_key AS year,
    votes,
    SUM(votes) OVER (ORDER BY year_key ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
        AS cumulative_votes
FROM fact_movie
WHERE votes IS NOT NULL
ORDER BY year_key
""").fetchdf()
```

> **What you should see:** Running total of votes as you move forward in time through the ranked movies.

## Common Table Expressions (CTEs)

CTEs (introduced with `WITH`) name intermediate result sets, making complex queries readable by breaking them into named steps.

### Cell 23 — Most-genre movies via CTE

```python
conn.execute("""
WITH genre_counts AS (
    SELECT movie_id, COUNT(*) AS num_genres
    FROM bridge_movie_genre
    GROUP BY movie_id
),
ranked AS (
    SELECT
        f.title,
        f.year_key AS year,
        gc.num_genres,
        RANK() OVER (ORDER BY gc.num_genres DESC) AS genre_rank
    FROM fact_movie f
    JOIN genre_counts gc ON gc.movie_id = f.movie_id
)
SELECT title, year, num_genres, genre_rank
FROM ranked
WHERE genre_rank <= 10
ORDER BY genre_rank, title
""").fetchdf()
```

> **What you should see:** The movies with the most genre tags, ranked. The query reads as a pipeline: count genres → rank by count → filter top 10.

### Cell 24 — Directors with most films via CTE

```python
conn.execute("""
WITH director_films AS (
    SELECT
        p.name                          AS director,
        COUNT(DISTINCT b.movie_id)      AS num_films,
        ROUND(AVG(f.rating), 2)         AS avg_rating
    FROM bridge_movie_person b
    JOIN dim_person p  ON p.person_key = b.person_key
    JOIN dim_role   r  ON r.role_key   = b.role_key AND r.name = 'director'
    JOIN fact_movie f  ON f.movie_id   = b.movie_id
    GROUP BY p.person_key, p.name
)
SELECT *
FROM director_films
ORDER BY num_films DESC, avg_rating DESC
""").fetchdf()
```

## PIVOT and UNPIVOT

### Cell 25 — PIVOT: movies per genre × era

DuckDB has native `PIVOT` syntax that rotates rows into columns without manual `CASE WHEN` expressions.

```python
conn.execute("""
PIVOT (
    SELECT g.name AS genre, y.era, COUNT(DISTINCT f.movie_id) AS cnt
    FROM fact_movie f
    JOIN dim_year           y ON y.year_key  = f.year_key
    JOIN bridge_movie_genre b ON b.movie_id  = f.movie_id
    JOIN dim_genre          g ON g.genre_key = b.genre_key
    GROUP BY g.name, y.era
)
ON era
USING SUM(cnt)
GROUP BY genre
ORDER BY genre
""").fetchdf()
```

```
         genre  Classic  Contemporary  Modern
        Action        0             7       7
     Adventure        0             8       6
     Animation        0             1       1
     Biography        2             1       3
        Comedy        2             2       3
         Crime        6             4       7
         Drama       13             9      16
    ...
```

> **What you should see:** A matrix with genres as rows and eras as columns. Each cell contains the number of movies in that genre–era combination. `NULL` means no movies for that combination.
>
> **What just happened?** `PIVOT … ON era USING SUM(cnt)` rotated the distinct values of `era` into column headers automatically — no hardcoded column list needed.

### Cell 26 — UNPIVOT: reverse the pivot

```python
conn.execute("""
UNPIVOT (
    PIVOT (
        SELECT g.name AS genre, y.era, COUNT(DISTINCT f.movie_id) AS cnt
        FROM fact_movie f
        JOIN dim_year           y ON y.year_key  = f.year_key
        JOIN bridge_movie_genre b ON b.movie_id  = f.movie_id
        JOIN dim_genre          g ON g.genre_key = b.genre_key
        GROUP BY g.name, y.era
    )
    ON era USING SUM(cnt) GROUP BY genre
)
ON (Classic, Contemporary, Modern)
INTO NAME era VALUE num_movies
ORDER BY genre, era
""").fetchdf()
```

> **What you should see:** The matrix folded back into rows — one row per genre–era combination with a `num_movies` column. `UNPIVOT` is useful for feeding pivoted reports back into row-based processing.

## Querying Files Directly

One of DuckDB's most distinctive features is the ability to query files without loading them. DuckDB can scan CSV, Parquet, JSON, and other formats directly from the file system.

### Cell 27 — Export to CSV and query back

```python
# Export the top-50 movies with their genres to a CSV file
conn.execute("""
COPY (
    SELECT
        f.movie_id,
        f.title,
        f.year_key            AS year,
        f.rating,
        f.rank,
        STRING_AGG(g.name, ', ' ORDER BY g.name) AS genres
    FROM fact_movie f
    JOIN bridge_movie_genre b ON b.movie_id  = f.movie_id
    JOIN dim_genre          g ON g.genre_key = b.genre_key
    WHERE f.rank IS NOT NULL
    GROUP BY f.movie_id, f.title, f.year_key, f.rating, f.rank
    ORDER BY f.rank
) TO '/data-transfer/top50_movies.csv' (HEADER, DELIMITER ',')
""")
print("Exported.")
```

```python
# Query the CSV file directly — no table load needed
conn.execute("""
SELECT title, year, rating, genres
FROM '/data-transfer/top50_movies.csv'
WHERE genres LIKE '%Drama%'
ORDER BY rating DESC
LIMIT 5
""").fetchdf()
```

> **What you should see:** Results queried directly from the CSV file, filtered and ordered, with no `CREATE TABLE` or `COPY FROM` step.
>
> **What just happened?** DuckDB inferred the schema from the CSV header row, opened the file as a virtual table, and applied the `WHERE` and `ORDER BY` in a single columnar scan — exactly like querying a real table.

### Cell 28 — Export to Parquet and query back

```python
# Export to Parquet (columnar binary format — much faster for analytics)
conn.execute("""
COPY (SELECT * FROM fact_movie WHERE rating IS NOT NULL)
TO '/data-transfer/movies.parquet' (FORMAT PARQUET)
""")
print("Parquet written.")
```

```python
# Query the Parquet file
conn.execute("""
SELECT
    title,
    rating,
    NTILE(3) OVER (ORDER BY rating DESC) AS tier
FROM '/data-transfer/movies.parquet'
ORDER BY rating DESC
LIMIT 10
""").fetchdf()
```

> **What you should see:** Window function results computed directly from the Parquet file — no table required.
>
> **What just happened?** Parquet stores data in column groups with statistics per row group. DuckDB pushed down the `ORDER BY` into the Parquet scan, reading only the `title` and `rating` columns — skipping all others entirely. On large datasets this is dramatically faster than CSV.

## Python and Pandas Integration

DuckDB integrates tightly with Pandas: query results can be returned as DataFrames, and DataFrames can be queried as if they were tables.

### Cell 29 — Query result as DataFrame

```python
import pandas as pd

df_genres = conn.execute("""
SELECT g.name AS genre,
       COUNT(DISTINCT f.movie_id) AS num_movies,
       ROUND(AVG(f.rating), 2)    AS avg_rating
FROM fact_movie f
JOIN bridge_movie_genre b ON b.movie_id  = f.movie_id
JOIN dim_genre          g ON g.genre_key = b.genre_key
WHERE f.rating IS NOT NULL
GROUP BY g.genre_key, g.name
ORDER BY num_movies DESC
""").fetchdf()

print(df_genres.head(10).to_string(index=False))
```

> **What you should see:** A Pandas DataFrame with genre statistics.

### Cell 30 — Register a DataFrame as a DuckDB table

```python
# Create a DataFrame with extra data not in the database
extra = pd.DataFrame({
    'movie_id': ['0133093', '0110912'],
    'streaming_platform': ['Netflix', 'Amazon Prime'],
    'available_4k': [True, False]
})

# Register it as a virtual table
conn.register('streaming_info', extra)

# Join it with fact_movie in SQL
conn.execute("""
SELECT f.title, f.rating, s.streaming_platform, s.available_4k
FROM fact_movie f
JOIN streaming_info s ON s.movie_id = f.movie_id
""").fetchdf()
```

```
          title  rating streaming_platform  available_4k
     The Matrix     8.7            Netflix          True
   Pulp Fiction     8.9       Amazon Prime         False
```

> **What you should see:** A join between a real DuckDB table and an in-memory Pandas DataFrame — treated identically.
>
> **What just happened?** `conn.register()` exposed the DataFrame to DuckDB's query engine as a virtual table without copying any data. This lets you combine database data with live DataFrame transformations in a single SQL query.

## Understanding Vectorised Execution

### Cell 31 — EXPLAIN: reading the query plan

```python
plan = conn.execute("""
EXPLAIN
SELECT g.name AS genre, ROUND(AVG(f.rating), 2) AS avg_rating
FROM fact_movie f
JOIN bridge_movie_genre b ON b.movie_id  = f.movie_id
JOIN dim_genre          g ON g.genre_key = b.genre_key
WHERE f.rating IS NOT NULL
GROUP BY g.genre_key, g.name
ORDER BY avg_rating DESC
""").fetchdf()

print(plan['explain_value'].iloc[0])
```

```
┌───────────────────────────┐
│         PROJECTION        │
│   ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│          genre            │
│        avg_rating         │
└─────────────┬─────────────┘
              │
┌─────────────┴─────────────┐
│       ORDER BY            │
│   ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│     avg_rating DESC       │
└─────────────┬─────────────┘
              │
┌─────────────┴─────────────┐
│     HASH_GROUP_BY         │
│   ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │ 
│  genre_key, name, avg(…)  │
└─────────────┬─────────────┘
              │
┌─────────────┴─────────────┐
│        HASH_JOIN          │
│    bridge ⋈ dim_genre     │
└─────────────┬─────────────┘
              │
┌─────────────┴─────────────┐
│        HASH_JOIN          │
│    fact_movie ⋈ bridge    │
└─────────────┬─────────────┘
              │
┌─────────────┴─────────────────┐
│  SEQ_SCAN fact_movie          │
│  Filter: rating IS NOT NULL.  │
│  Projections: movie_id, rating│
└───────────────────────────────┘
```

> **What you should see:** A tree of physical operators. Key things to note:
> - `SEQ_SCAN` with a projection list — DuckDB only reads the columns it needs (`movie_id`, `rating`), not the whole row
> - `HASH_JOIN` — DuckDB chose hash joins for both bridge joins, appropriate for moderate-sized tables
> - `HASH_GROUP_BY` — the aggregation is hash-based, not sort-based
> - The entire plan runs in a **vectorised** fashion: each operator processes a batch of ~2048 rows at a time rather than one row at a time, dramatically reducing per-row overhead

### Cell 32 — EXPLAIN ANALYZE: actual runtime metrics

```python
plan = conn.execute("""
EXPLAIN ANALYZE
SELECT g.name AS genre, ROUND(AVG(f.rating), 2) AS avg_rating
FROM fact_movie f
JOIN bridge_movie_genre b ON b.movie_id  = f.movie_id
JOIN dim_genre          g ON g.genre_key = b.genre_key
WHERE f.rating IS NOT NULL
GROUP BY g.genre_key, g.name
ORDER BY avg_rating DESC
""").fetchdf()

print(plan['explain_value'].iloc[0])
```

> **What you should see:** The same plan tree enriched with actual row counts processed per operator, elapsed time, and memory usage — identical semantics to PostgreSQL's `EXPLAIN ANALYZE`.
