# Working with PostgreSQL

In this workshop we will learn how to work with **PostgreSQL**, a powerful open-source relational database. We will use film data as we will use in other workshops as well.

## Table of Contents

- [What you will learn](#what-you-will-learn)
- [Prerequisites](#prerequisites)
- [Background: Relational Model and 3NF](#background-relational-model-and-3nf)
- [The 3NF Schema Design](#the-3nf-schema-design)
- [Connecting to PostgreSQL](#connecting-to-postgresql)
- [Creating the Database and Schema](#creating-the-database-and-schema)
- [Inserting Data](#inserting-data)
- [Basic Queries](#basic-queries)
- [Joining Tables](#joining-tables)
- [Aggregating Data](#aggregating-data)
- [Performance Optimisations using Indexes](#performance-optimisations-using-indexes)
- [Views](#views)

## What you will learn

- How relational normalisation (3NF) eliminates redundancy and enforces integrity
- How to connect to PostgreSQL using the `psql` CLI and DbGate
- How to create tables with primary keys, foreign keys, and constraints
- How to insert data using `INSERT INTO`
- How to query data with `SELECT`, `WHERE`, `ORDER BY`, and `LIMIT`
- How to combine data from multiple tables using `JOIN`
- How to aggregate data with `GROUP BY`, `COUNT`, `AVG`, and `HAVING`
- How to improve query performance with indexes
- How to create reusable views

## Prerequisites

- The **Data Platform** described [here](../00-environment/README.md) is running and accessible

## Background: Relational Model and 3NF

The **relational model** organises data into tables of rows and columns. Each row represents one fact; each column holds one atomic value. Relationships between entities are expressed through foreign keys rather than by embedding or duplicating data.

**Normalisation** is the process of structuring a relational schema to reduce redundancy and prevent update anomalies. The standard target is **Third Normal Form (3NF)**:

1. **1NF** — No repeating groups. Each cell contains a single atomic value. Multi-valued attributes like genres and languages become separate tables linked by foreign keys.
2. **2NF** — Every non-key column depends on the *whole* primary key (eliminates partial dependencies in composite-key tables).
3. **3NF** — Every non-key column depends *only* on the primary key, not transitively on another non-key column. For example, a genre's name is stored once in `genre`, not repeated inside every row of `movie_genre`.

## The 3NF Schema Design

The film dataset maps to the following relational schema:

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

This design eliminates all redundancy: a person's name is stored once; a genre string is stored once; the relationship tables only carry foreign keys.

## Connecting to PostgreSQL

### Using the psql CLI

Open a terminal and connect to the running PostgreSQL container:

```bash
docker exec -it postgresql psql -U postgres
```

You should see the PostgreSQL prompt:

```
psql (18.3 (Debian 18.3-1.pgdg13+1))
Type "help" for help.

postgres=#
```

> **What you should see:** The `postgres=#` prompt confirming a successful connection as the `postgres` superuser.

Useful meta-commands inside `psql`:

| Command | Description |
|---------|-------------|
| `\l` | List all databases |
| `\c dbname` | Connect to a database |
| `\dt` | List tables in the current schema |
| `\d tablename` | Describe a table |
| `\timing` | Toggle query execution timing |
| `\q` | Quit |

### Using DbGate

In a browser navigate to <http://dataplatform:28120> and log in with user `dbgate` and password `abc123!`.

Click **+ Add new connection** under **CONNECTIONS**, select `PostgreSQL` as the connection type, and enter:

- **Server**: `postgresql`
- **Port**: `5432`
- **User**: `postgres`
- **Password**: `abc123!`

Click **Test** and then **Connect**.

> **What you should see:** The DbGate web UI with the PostgreSQL connection listed. Expanding it shows the `postgres` and `demodb` database and any others that have been created.

## Creating the Database and Schema

### Create the filmdb database

In `psql`, create a dedicated database for the film data:

```sql
CREATE DATABASE filmdb;
```

Connect to it:

```sql
\c filmdb
```

and you should see the following message

```
You are now connected to database "filmdb" as user "postgres".
```

### Create the tables

Paste the following DDL in order. Each `CREATE TABLE` statement includes inline comments explaining the design decision.

```sql
-- Controlled vocabulary: genres (e.g. "Drama", "Action")
CREATE TABLE genre (
    genre_id  SERIAL      PRIMARY KEY,
    name      VARCHAR(50) NOT NULL UNIQUE
);

-- Controlled vocabulary: ISO 639-1 language codes (e.g. "en", "fr")
CREATE TABLE language (
    language_id SERIAL     PRIMARY KEY,
    code        VARCHAR(5) NOT NULL UNIQUE
);

-- Core movie entity
CREATE TABLE movie (
    movie_id     VARCHAR(10)    PRIMARY KEY,
    title        VARCHAR(255)   NOT NULL,
    year         INTEGER,
    runtime      INTEGER,                    -- minutes
    rating       NUMERIC(3,1),
    votes        INTEGER,
    plot_outline TEXT,
    cover_url    VARCHAR(500),
    rank         INTEGER
);

-- Core person entity (actors, directors, producers all share this table)
CREATE TABLE person (
    person_id  VARCHAR(10)  PRIMARY KEY,
    name       VARCHAR(255) NOT NULL,
    headshot   VARCHAR(500),
    birth_date DATE
);

-- Multi-valued attribute: a person can have many trademarks
CREATE TABLE person_trademark (
    person_id VARCHAR(10) NOT NULL REFERENCES person(person_id) ON DELETE CASCADE,
    trademark TEXT        NOT NULL,
    PRIMARY KEY (person_id, trademark)
);

-- Junction: a movie belongs to one or more genres
CREATE TABLE movie_genre (
    movie_id VARCHAR(10) NOT NULL REFERENCES movie(movie_id)  ON DELETE CASCADE,
    genre_id INTEGER     NOT NULL REFERENCES genre(genre_id)  ON DELETE CASCADE,
    PRIMARY KEY (movie_id, genre_id)
);

-- Junction: a movie is available in one or more languages
CREATE TABLE movie_language (
    movie_id    VARCHAR(10) NOT NULL REFERENCES movie(movie_id)      ON DELETE CASCADE,
    language_id INTEGER     NOT NULL REFERENCES language(language_id) ON DELETE CASCADE,
    PRIMARY KEY (movie_id, language_id)
);

-- Junction: links movies to people with a role discriminator
CREATE TABLE movie_person (
    movie_id  VARCHAR(10) NOT NULL REFERENCES movie(movie_id)  ON DELETE CASCADE,
    person_id VARCHAR(10) NOT NULL REFERENCES person(person_id) ON DELETE CASCADE,
    role      VARCHAR(20) NOT NULL CHECK (role IN ('actor','director','producer')),
    PRIMARY KEY (movie_id, person_id, role)
);
```

Verify the tables were created:

```sql
\dt
```

```
                List of tables
 Schema |       Name       | Type  |  Owner
--------+------------------+-------+----------
 public | genre            | table | postgres
 public | language         | table | postgres
 public | movie            | table | postgres
 public | movie_genre      | table | postgres
 public | movie_language   | table | postgres
 public | movie_person     | table | postgres
 public | person           | table | postgres
 public | person_trademark | table | postgres
(8 rows)
```

> **What you should see:** All 8 tables listed. The naming convention makes the junction tables self-documenting.

## Inserting Data

### Genres and languages

Insert the controlled vocabulary first, so the junction tables can reference them by ID.

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

Insert the two full-detail movies, then the remaining 48 from the Top-50 list:

```sql
INSERT INTO movie (movie_id, title, year, runtime, rating, votes, plot_outline, cover_url) VALUES
('0110912', 'Pulp Fiction', 1994, 154, 8.9, 2084331,
 'Jules Winnfield and Vincent Vega are two hit men who are out to retrieve a suitcase stolen from their employer, mob boss Marsellus Wallace. Wallace has also asked Vincent to take his wife Mia out a few days later. Butch Coolidge is an aging boxer who is paid by Wallace to lose his fight. The lives of these seemingly unrelated people are woven together comprising a series of funny, bizarre and uncalled-for incidents.',
 'https://m.media-amazon.com/images/M/MV5BNGNhMDIzZTUtNTBlZi00MTRlLWFjM2ItYzViMjE3YzI5MjljXkEyXkFqcGdeQXVyNzkwMjQ5NzM@._V1_SY150_CR1,0,101,150_.jpg'),
('0133093', 'The Matrix', 1999, 136, 8.7, 1496538,
 'Thomas A. Anderson is a man living two lives. By day he is an average computer programmer and by night a hacker known as Neo. Neo finds himself targeted by the police when he is contacted by Morpheus, a legendary computer hacker branded a terrorist by the government. Morpheus awakens Neo to the real world, a ravaged wasteland where most of humanity have been captured by a race of machines.',
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

> **What you should see:** `INSERT 0 2` then `INSERT 0 48` — 50 movies total.
>
> **What just happened?** The schema enforces that every row has the same columns. The second batch omits `runtime`, `votes`, `plot_outline`, and `cover_url` — those columns simply receive `NULL`, which is explicit and queryable rather than simply absent.

### Persons

```sql
INSERT INTO person (person_id, name, headshot, birth_date) VALUES
('0000246', 'Bruce Willis',
 'https://m.media-amazon.com/images/M/MV5BMjA0MjMzMTE5OF5BMl5BanBnXkFtZTcwMzQ2ODE3Mw@@._V1_UY98_CR8,0,67,98_AL_.jpg',
 '1955-03-19'),
('0000206', 'Keanu Reeves',    NULL, NULL),
('0000113', 'Sandra Bullock',
 'https://m.media-amazon.com/images/M/MV5BMTI5NDY5NjU3NF5BMl5BanBnXkFtZTcwMzQ0MTMyMw@@._V1_UX67_CR0,0,67,98_AL_.jpg',
 '1964-07-26'),
('0000233', 'Quentin Tarantino',
 'https://m.media-amazon.com/images/M/MV5BMTgyMjI3ODA3Nl5BMl5BanBnXkFtZTcwNzY2MDYxOQ@@._V1_UX67_CR0,0,67,98_AL_.jpg',
 '1963-03-27'),
-- Additional cast for Pulp Fiction
('0000619', 'Tim Roth',            NULL, NULL),
('0001625', 'Amanda Plummer',      NULL, NULL),
('0522503', 'Laura Lovelace',      NULL, NULL),
('0000237', 'John Travolta',       NULL, NULL),
('0000168', 'Samuel L. Jackson',   NULL, NULL),
('0482851', 'Phil LaMarr',         NULL, NULL),
('0001844', 'Frank Whaley',        NULL, NULL),
('0824882', 'Burr Steers',         NULL, NULL),
('0000609', 'Ving Rhames',         NULL, NULL),
('0000235', 'Uma Thurman',         NULL, NULL),
('0004744', 'Lawrence Bender',     NULL, NULL),
('0000362', 'Danny DeVito',        NULL, NULL),
('0321621', 'Richard N. Gladstein',NULL, NULL),
('0787834', 'Michael Shamberg',    NULL, NULL),
('0792049', 'Stacey Sher',         NULL, NULL),
('0918424', 'Bob Weinstein',       NULL, NULL),
('0005544', 'Harvey Weinstein',    NULL, NULL),
-- Additional cast for The Matrix
('0000401', 'Laurence Fishburne',  NULL, NULL),
('0005251', 'Carrie-Anne Moss',    NULL, NULL),
('0915989', 'Hugo Weaving',        NULL, NULL),
('0287825', 'Gloria Foster',       NULL, NULL),
('0001592', 'Joe Pantoliano',      NULL, NULL),
('0159059', 'Marcus Chong',        NULL, NULL),
('0032810', 'Julian Arahanga',     NULL, NULL),
('0905154', 'Lana Wachowski',      NULL, NULL),
('0905152', 'Lilly Wachowski',     NULL, NULL),
('0075732', 'Bruce Berman',        NULL, NULL),
('0185621', 'Dan Cracchiolo',      NULL, NULL),
('0400492', 'Carol Hughes',        NULL, NULL);
```

### Person trademarks

```sql
INSERT INTO person_trademark (person_id, trademark) VALUES
('0000246', 'Frequently plays a man who suffered a tragedy, had a crisis of confidence or conscience'),
('0000246', 'Frequently plays likeable wisecracking heroes with a moral centre'),
('0000246', 'Headlines action-adventures, often playing a policeman, hitman or someone in the military'),
('0000246', 'Often plays men who get caught up in situations far beyond their control'),
('0000246', 'Sardonic one-liners'),
('0000246', 'Shaven head'),
('0000246', 'Distinctive, gravelly voice'),
('0000206', 'Intense contemplative gaze'),
('0000206', 'Deep husky voice'),
('0000206', 'Known for playing stoic reserved characters'),
('0000233', 'Lead characters usually drive General Motors vehicles, particularly Chevrolet and Cadillac'),
('0000233', 'Briefcases and suitcases play an important role in many of his films'),
('0000233', 'Makes references to cult movies and television'),
('0000233', 'His films usually have a shot from inside an automobile trunk'),
('0000233', 'Often uses an unconventional storytelling device in his films');
```

### Movie genres

```sql
-- Pulp Fiction: Crime, Drama
INSERT INTO movie_genre (movie_id, genre_id)
SELECT '0110912', genre_id FROM genre WHERE name IN ('Crime','Drama');

-- The Matrix: Action, Sci-Fi
INSERT INTO movie_genre (movie_id, genre_id)
SELECT '0133093', genre_id FROM genre WHERE name IN ('Action','Sci-Fi');

-- Top-50 movies
INSERT INTO movie_genre (movie_id, genre_id)
SELECT t.movie_id, g.genre_id FROM (VALUES
    ('0111161','Drama'),
    ('0068646','Crime'),  ('0068646','Drama'),
    ('0071562','Crime'),  ('0071562','Drama'),
    ('0468569','Action'), ('0468569','Crime'), ('0468569','Drama'), ('0468569','Thriller'),
    ('0050083','Drama'),
    ('0108052','Biography'),('0108052','Drama'),('0108052','History'),
    ('0167260','Adventure'),('0167260','Drama'),('0167260','Fantasy'),
    ('0060196','Western'),
    ('0137523','Drama'),
    ('4154796','Action'),  ('4154796','Adventure'),('4154796','Fantasy'),('4154796','Sci-Fi'),
    ('0120737','Adventure'),('0120737','Drama'),  ('0120737','Fantasy'),
    ('0109830','Drama'),   ('0109830','Romance'),
    ('0080684','Action'),  ('0080684','Adventure'),('0080684','Fantasy'),('0080684','Sci-Fi'),
    ('1375666','Action'),  ('1375666','Adventure'),('1375666','Sci-Fi'),('1375666','Thriller'),
    ('0167261','Adventure'),('0167261','Drama'),  ('0167261','Fantasy'),
    ('0073486','Drama'),
    ('0099685','Biography'),('0099685','Crime'),  ('0099685','Drama'),
    ('0047478','Adventure'),('0047478','Drama'),
    ('0114369','Crime'),   ('0114369','Drama'),   ('0114369','Mystery'),('0114369','Thriller'),
    ('0317248','Crime'),   ('0317248','Drama'),
    ('0076759','Action'),  ('0076759','Adventure'),('0076759','Fantasy'),('0076759','Sci-Fi'),
    ('0102926','Crime'),   ('0102926','Drama'),   ('0102926','Thriller'),
    ('0038650','Drama'),   ('0038650','Family'),  ('0038650','Fantasy'),
    ('0118799','Comedy'),  ('0118799','Drama'),   ('0118799','Romance'),('0118799','War'),
    ('0245429','Animation'),('0245429','Adventure'),('0245429','Family'),('0245429','Fantasy'),('0245429','Mystery'),
    ('0120815','Drama'),   ('0120815','War'),
    ('0114814','Crime'),   ('0114814','Mystery'), ('0114814','Thriller'),
    ('0110413','Action'),  ('0110413','Crime'),   ('0110413','Drama'),  ('0110413','Thriller'),
    ('0120689','Crime'),   ('0120689','Drama'),   ('0120689','Fantasy'),('0120689','Mystery'),
    ('0816692','Adventure'),('0816692','Drama'),  ('0816692','Sci-Fi'),
    ('0054215','Horror'),  ('0054215','Mystery'), ('0054215','Thriller'),
    ('0120586','Drama'),
    ('0021749','Comedy'),  ('0021749','Drama'),   ('0021749','Romance'),
    ('0034583','Drama'),   ('0034583','Romance'), ('0034583','War'),
    ('0064116','Western'),
    ('0253474','Biography'),('0253474','Drama'),  ('0253474','Music'),  ('0253474','War'),
    ('0027977','Comedy'),  ('0027977','Drama'),   ('0027977','Family'), ('0027977','Romance'),
    ('1675434','Biography'),('1675434','Comedy'), ('1675434','Drama'),
    ('0407887','Crime'),   ('0407887','Drama'),   ('0407887','Thriller'),
    ('0088763','Adventure'),('0088763','Comedy'), ('0088763','Sci-Fi'),
    ('0103064','Action'),  ('0103064','Sci-Fi'),
    ('2582802','Drama'),   ('2582802','Music'),
    ('0110357','Animation'),('0110357','Adventure'),('0110357','Drama'),('0110357','Family'),('0110357','Musical'),
    ('0047396','Mystery'), ('0047396','Thriller'),
    ('0082971','Action'),  ('0082971','Adventure'),
    ('0172495','Action'),  ('0172495','Adventure'),('0172495','Drama'),
    ('0482571','Drama'),   ('0482571','Mystery'),  ('0482571','Sci-Fi'),('0482571','Thriller'),
    ('0078788','Drama'),   ('0078788','War')
) AS t(movie_id, genre_name)
JOIN genre g ON g.name = t.genre_name;
```

> **What you should see:** `INSERT 0 N` for each batch. The `SELECT … FROM genre WHERE name IN (…)` pattern avoids hard-coding genre IDs — the lookup is done by the database at insert time, which is both safer and easier to read.

### Movie languages

```sql
INSERT INTO movie_language (movie_id, language_id)
SELECT '0110912', language_id FROM language WHERE code IN ('en','es','fr');

INSERT INTO movie_language (movie_id, language_id)
SELECT '0133093', language_id FROM language WHERE code = 'en';
```

### Movie–person relationships

```sql
-- Pulp Fiction cast
INSERT INTO movie_person (movie_id, person_id, role) VALUES
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
('0110912','0005544','producer');

-- The Matrix cast
INSERT INTO movie_person (movie_id, person_id, role) VALUES
('0133093','0000206','actor'), ('0133093','0000401','actor'),
('0133093','0005251','actor'), ('0133093','0915989','actor'),
('0133093','0287825','actor'), ('0133093','0001592','actor'),
('0133093','0159059','actor'), ('0133093','0032810','actor'),
('0133093','0905154','director'),('0133093','0905152','director'),
('0133093','0075732','producer'),('0133093','0185621','producer'),
('0133093','0400492','producer');

-- Bruce Willis film appearances (from persons collection)
-- Note: Pulp Fiction already inserted above; add the others as movie entries first
INSERT INTO movie (movie_id, title, year) VALUES
('1606378','A Good Day to Die Hard',  2013),
('0217869','Unbreakable',             2000),
('0377917','The Fifth Element',       1997),
('0112864','Die Hard: With a Vengeance',1995)
ON CONFLICT (movie_id) DO NOTHING;

INSERT INTO movie_person (movie_id, person_id, role) VALUES
('1606378','0000246','actor'),
('0217869','0000246','actor'),
('0377917','0000246','actor'),
('0112864','0000246','actor');

-- Keanu Reeves film appearances
INSERT INTO movie (movie_id, title, year) VALUES
('0234215','The Matrix Reloaded',2003),
('0111257','Speed',              1994)
ON CONFLICT (movie_id) DO NOTHING;

INSERT INTO movie_person (movie_id, person_id, role) VALUES
('0234215','0000206','actor'),
('0111257','0000206','actor');

-- Sandra Bullock film appearances
INSERT INTO movie (movie_id, title, year) VALUES
('2737304','Bird Box',                2018),
('0120179','Speed 2: Cruise Control', 1997),
('0212346','Miss Congeniality',        2000)
ON CONFLICT (movie_id) DO NOTHING;

INSERT INTO movie_person (movie_id, person_id, role) VALUES
('2737304','0000113','actor'),
('0120179','0000113','actor'),
('0111257','0000113','actor'),
('0212346','0000113','actor');

-- Quentin Tarantino as actor/director
INSERT INTO movie (movie_id, title, year) VALUES
('0378194','Kill Bill: Vol. 2',  2004),
('0116367','From Dusk Till Dawn',1996),
('0119396','Jackie Brown',       1997)
ON CONFLICT (movie_id) DO NOTHING;

INSERT INTO movie_person (movie_id, person_id, role) VALUES
('0378194','0000233','actor'),
('0116367','0000233','actor'),
('0119396','0000233','director');
```

### Verify the row counts

```sql
SELECT 'movie'                   AS "table", COUNT(*) FROM movie     UNION ALL
SELECT 'person',                             COUNT(*) FROM person    UNION ALL
SELECT 'genre',                              COUNT(*) FROM genre     UNION ALL
SELECT 'language',                           COUNT(*) FROM language  UNION ALL
SELECT 'movie_genre',                        COUNT(*) FROM movie_genre UNION ALL
SELECT 'movie_language',                     COUNT(*) FROM movie_language UNION ALL
SELECT 'movie_person',                       COUNT(*) FROM movie_person UNION ALL
SELECT 'person_trademark',                   COUNT(*) FROM person_trademark;
```

```
      table       | count
------------------+-------
 movie            |    62
 person           |    33
 genre            |    19
 language         |     3
 movie_genre      |   139
 movie_language   |     4
 movie_person     |    46
 person_trademark |    15
(8 rows)
```

> **What you should see:** Row counts similar to the above. Every fact is stored exactly once.

## Basic Queries

### SELECT all movies

```sql
SELECT movie_id, title, year, rating
FROM movie
ORDER BY rating DESC NULLS LAST, title
LIMIT 10;
```

```
 movie_id |                     title                     | year | rating
----------+-----------------------------------------------+------+--------
 0068646  | The Godfather                                 | 1972 |    9.2
 0111161  | The Shawshank Redemption                      | 1994 |    9.2
 0468569  | The Dark Knight                               | 2008 |    9.0
 0071562  | The Godfather: Part II                        | 1974 |    9.0
 0050083  | 12 Angry Men                                  | 1957 |    8.9
 0110912  | Pulp Fiction                                  | 1994 |    8.9
 0108052  | Schindler's List                              | 1993 |    8.9
 0167260  | The Lord of the Rings: The Return of the King | 2003 |    8.9
 4154796  | Avengers: Endgame                             | 2019 |    8.8
 0137523  | Fight Club                                    | 1999 |    8.8
(10 rows)
```

> **What you should see:** The top-rated movies in descending order. `NULLS LAST` ensures movies without a rating do not bubble to the top.

### Filter with WHERE

Find all movies released in 1994:

```sql
SELECT title, year, rating
FROM movie
WHERE year = 1994
ORDER BY rating DESC NULLS LAST;
```

Find all movies with a rating above 8.8:

```sql
SELECT title, year, rating
FROM movie
WHERE rating > 8.8
ORDER BY rating DESC;
```

Find movies whose title contains "the" (case-insensitive):

```sql
SELECT title, year
FROM movie
WHERE title ILIKE '%the%'
ORDER BY year;
```

> **What you should see:** Rows matching the filter condition. `ILIKE` is PostgreSQL's case-insensitive `LIKE`.

### Limit and offset (pagination)

```sql
-- Page 1: rows 1–10
SELECT title, year, rating
FROM movie
ORDER BY rating DESC NULLS LAST
LIMIT 10 OFFSET 0;

-- Page 2: rows 11–20
SELECT title, year, rating
FROM movie
ORDER BY rating DESC NULLS LAST
LIMIT 10 OFFSET 10;
```

## Joining Tables

### Find all genres for a movie

```sql
SELECT m.title, g.name AS genre
FROM movie m
JOIN movie_genre mg ON mg.movie_id = m.movie_id
JOIN genre g        ON g.genre_id  = mg.genre_id
WHERE m.title = 'Pulp Fiction'
ORDER BY g.name;
```

```
    title     | genre
--------------+-------
 Pulp Fiction | Crime
 Pulp Fiction | Drama
(2 rows)
```

> **What you should see:** Both genres for Pulp Fiction. Each row is the result of two JOINs travelling through the junction table to reach the genre name.

### Find all movies in a given genre

```sql
SELECT m.title, m.year, m.rating
FROM movie m
JOIN movie_genre mg ON mg.movie_id = m.movie_id
JOIN genre g        ON g.genre_id  = mg.genre_id
WHERE g.name = 'Action'
ORDER BY m.rating DESC NULLS LAST;
```

```
                     title                      | year | rating
------------------------------------------------+------+--------
 The Dark Knight                                | 2008 |    9.0
 Avengers: Endgame                              | 2019 |    8.8
 The Matrix                                     | 1999 |    8.7
 Inception                                      | 2010 |    8.7
 Star Wars: Episode V - The Empire Strikes Back | 1980 |    8.7
 Star Wars: Episode IV - A New Hope             | 1977 |    8.6
 Raiders of the Lost Ark                        | 1981 |    8.5
 Gladiator                                      | 2000 |    8.5
 Terminator 2: Judgment Day                     | 1991 |    8.5
 Léon: The Professional                         | 1994 |    8.5
(10 rows)
```

> **What you should see:** All movies tagged with the Action genre, sorted by rating.

### Find all actors in a movie

```sql
SELECT p.name, mp.role
FROM person p
JOIN movie_person mp ON mp.person_id = p.person_id
JOIN movie m         ON m.movie_id   = mp.movie_id
WHERE m.title = 'Pulp Fiction'
ORDER BY mp.role, p.name;
```

```
         name         |   role
----------------------+----------
 Amanda Plummer       | actor
 Bruce Willis         | actor
 Burr Steers          | actor
 Frank Whaley         | actor
 John Travolta        | actor
 Laura Lovelace       | actor
 Phil LaMarr          | actor
 Quentin Tarantino    | actor
 Samuel L. Jackson    | actor
 Tim Roth             | actor
 Uma Thurman          | actor
 Ving Rhames          | actor
 Quentin Tarantino    | director
 Bob Weinstein        | producer
 Danny DeVito         | producer
 Harvey Weinstein     | producer
 Lawrence Bender      | producer
 Michael Shamberg     | producer
 Richard N. Gladstein | producer
 Stacey Sher          | producer
(20 rows)
```

> **What you should see:** All people linked to Pulp Fiction across all roles.

> **What just happened?** The `role` column in `movie_person` acts as a discriminator — one JOIN covers actors, directors, and producers simultaneously, without any duplication of person or movie data.

### Find all movies an actor appeared in

```sql
SELECT m.title, m.year, mp.role
FROM movie m
JOIN movie_person mp ON mp.movie_id  = m.movie_id
JOIN person p        ON p.person_id  = mp.person_id
WHERE p.name = 'Bruce Willis'
ORDER BY m.year;
```

```
           title            | year | role
----------------------------+------+-------
 Pulp Fiction               | 1994 | actor
 Die Hard: With a Vengeance | 1995 | actor
 The Fifth Element          | 1997 | actor
 Unbreakable                | 2000 | actor
 A Good Day to Die Hard     | 2013 | actor
(5 rows)
```

### LEFT JOIN — include movies with no cast data

```sql
SELECT m.title, m.year, COUNT(mp.person_id) AS cast_size
FROM movie m
LEFT JOIN movie_person mp ON mp.movie_id = m.movie_id AND mp.role = 'actor'
GROUP BY m.movie_id, m.title, m.year
ORDER BY cast_size DESC, m.title
LIMIT 10;
```

```
           title            | year | cast_size
----------------------------+------+-----------
 Pulp Fiction               | 1994 |        12
 The Matrix                 | 1999 |         8
 Speed                      | 1994 |         2
 A Good Day to Die Hard     | 2013 |         1
 Bird Box                   | 2018 |         1
 Die Hard: With a Vengeance | 1995 |         1
 From Dusk Till Dawn        | 1996 |         1
 Kill Bill: Vol. 2          | 2004 |         1
 Miss Congeniality          | 2000 |         1
 Speed 2: Cruise Control    | 1997 |         1
(10 rows)
```

> **What you should see:** Movies with the most actors listed first, and movies with no cast data showing `0` instead of being excluded entirely — the key difference between `LEFT JOIN` and `INNER JOIN`.

## Aggregating Data

### Count movies per genre

```sql
SELECT g.name AS genre, COUNT(mg.movie_id) AS num_movies
FROM genre g
LEFT JOIN movie_genre mg ON mg.genre_id = g.genre_id
GROUP BY g.genre_id, g.name
ORDER BY num_movies DESC;
```

```
   genre   | num_movies
-----------+------------
 Drama     |         36
 Adventure |         14
 Crime     |         12
 Action    |         10
 Thriller  |         10
 Fantasy   |          9
 Sci-Fi    |          9
 Mystery   |          7
 War       |          5
 Comedy    |          5
 Romance   |          5
 Biography |          4
 Family    |          4
 Music     |          2
 Western   |          2
 Animation |          2
 Horror    |          1
 History   |          1
 Musical   |          1
(19 rows)
```

### Average rating by genre

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

> **What you should see:** Each genre with its average rating and the number of rated movies. Genres with few films may have skewed averages.

### Movies per year — using HAVING to filter groups

```sql
SELECT year, COUNT(*) AS num_movies
FROM movie
WHERE year IS NOT NULL
GROUP BY year
HAVING COUNT(*) > 1
ORDER BY year;
```

> **What you should see:** Only years that have more than one movie in our dataset. `HAVING` filters on the aggregated result, whereas `WHERE` filters individual rows before grouping.

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

## Performance Optimisations using Indexes

### Check the query plan before adding an index

Use `EXPLAIN ANALYZE` to see how PostgreSQL executes a query:

```sql
EXPLAIN ANALYZE
SELECT m.title, m.year
FROM movie m
WHERE m.title = 'The Matrix';
```

With only 59 rows PostgreSQL will use a sequential scan (`Seq Scan`). On a large table this would be slow.

### Create an index on title

```sql
CREATE INDEX idx_movie_title ON movie (title);
```

Run `EXPLAIN ANALYZE` again:

```sql
EXPLAIN ANALYZE
SELECT m.title, m.year
FROM movie m
WHERE m.title = 'The Matrix';
```

```
                                             QUERY PLAN
-----------------------------------------------------------------------------------------------------
 Seq Scan on movie m  (cost=0.00..1.77 rows=1 width=22) (actual time=0.025..0.027 rows=1.00 loops=1)
   Filter: ((title)::text = 'The Matrix'::text)
   Rows Removed by Filter: 61
   Buffers: shared hit=1
 Planning Time: 0.077 ms
 Execution Time: 0.040 ms
(6 rows)
```

> **What you should see:** With 59 rows the planner still chooses a `Seq Scan` (sequential scans are faster than index lookups at very small sizes). On a table with millions of rows, you would see `Index Scan using idx_movie_title`.

### Create an index supporting the genre filter

```sql
CREATE INDEX idx_movie_genre_genre_id ON movie_genre (genre_id);
```

This speeds up `WHERE genre_id = ?` lookups, which power all genre-based queries.

### List all indexes on the filmdb tables

```sql
SELECT indexname, tablename, indexdef
FROM pg_indexes
WHERE schemaname = 'public'
ORDER BY tablename, indexname;
```

> **What you should see:** Primary key indexes (automatically created) plus the two indexes you just added.

## Views

A **view** is a stored query that can be referenced like a table. It does not store data — it is re-evaluated on every access.

### Create a view: full movie details

```sql
CREATE VIEW movie_full AS
SELECT
    m.movie_id,
    m.title,
    m.year,
    m.runtime,
    m.rating,
    m.votes,
    m.rank,
    STRING_AGG(DISTINCT g.name,     ', ' ORDER BY g.name)     AS genres,
    STRING_AGG(DISTINCT l.code,     ', ' ORDER BY l.code)     AS languages
FROM movie m
LEFT JOIN movie_genre    mg ON mg.movie_id    = m.movie_id
LEFT JOIN genre          g  ON g.genre_id     = mg.genre_id
LEFT JOIN movie_language ml ON ml.movie_id    = m.movie_id
LEFT JOIN language       l  ON l.language_id  = ml.language_id
GROUP BY m.movie_id, m.title, m.year, m.runtime, m.rating, m.votes, m.rank;
```

Query it like a table:

```sql
SELECT title, year, genres, languages
FROM movie_full
WHERE languages IS NOT NULL
ORDER BY year;
```

```
    title     | year |     genres     | languages
--------------+------+----------------+------------
 Pulp Fiction | 1994 | Crime, Drama   | en, es, fr
 The Matrix   | 1999 | Action, Sci-Fi | en
(2 rows)
```

> **What you should see:** Movies that have language data, with genres and languages collapsed into comma-separated strings using `STRING_AGG`.

### Create a view: actor filmography

```sql
CREATE VIEW actor_filmography AS
SELECT
    p.person_id,
    p.name       AS actor_name,
    p.birth_date,
    m.movie_id,
    m.title      AS movie_title,
    m.year,
    m.rating
FROM person p
JOIN movie_person mp ON mp.person_id = p.person_id AND mp.role = 'actor'
JOIN movie m         ON m.movie_id   = mp.movie_id;
```

```sql
SELECT movie_title, year, rating
FROM actor_filmography
WHERE actor_name = 'Keanu Reeves'
ORDER BY year;
```

```
     movie_title     | year | rating
---------------------+------+--------
 Speed               | 1994 |
 The Matrix          | 1999 |    8.7
 The Matrix Reloaded | 2003 |
(3 rows)
```
