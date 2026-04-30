# Working with Neo4J

In this workshop we will learn how to use the Neo4J NoSQL database. We will use movie and actor data as we use in other workshops, so the domain is familiar — but the data model and query patterns are fundamentally different from a relational database.

We assume that the platform described [here](../01-environment/README.md) is running and accessible.

## Table of Contents

- [What you will learn](#what-you-will-learn)
- [Prerequisites](#prerequisites)
- [Background: Graph Databases and the Property Graph Model](#background-graph-databases-and-the-property-graph-model)
- [The Movie Graph Data Model](#the-movie-graph-data-model)
- [Connecting to the Cypher Shell (optional)](#connecting-to-the-cypher-shell-optional)
- [Connecting with Neo4J Browser](#connecting-with-neo4j-browser)
- [Building the Movie Graph Step by Step](#building-the-movie-graph-step-by-step)
- [Indexes and Constraints](#indexes-and-constraints)
- [Viewing the Database](#viewing-the-database)
- [Querying the Graph](#querying-the-graph)
- [Working with Neo4J from Python](#working-with-neo4j-from-python)
- [Using MCP with Neo4J](#using-mcp-with-neo4j)

## What you will learn

- How a property graph differs from a relational model and when to prefer one over the other
- How to model entities and relationships as nodes, relationships, and properties
- How to build a graph database step by step using `CREATE` and `MERGE`
- How to connect to Neo4J using the Cypher Shell and the Neo4J Browser
- How to write Cypher queries to find, traverse, and analyse graph data
- How to aggregate, find shortest paths, and build recommendation patterns in Cypher
- How to update and delete nodes and relationships

## Prerequisites

- The **Data Platform** described [here](../01-environment/README.md) is running and accessible

## Background: Graph Databases and the Property Graph Model

### Why a graph database?

Relational databases model data as rows in tables, connected through foreign-key joins. This works well when relationships are few and well-defined. When data is *highly connected* — social networks, fraud detection, recommendation engines, knowledge graphs — every question requires many expensive joins, and the schema becomes rigid.

A **graph database** stores data as nodes and relationships directly. There are no join tables, no foreign keys, and no multi-hop join cost. Traversing ten relationships in a graph takes the same time regardless of whether the database has one thousand nodes or one billion, because the traversal follows direct pointers rather than scanning tables.

### The Property Graph Model

Neo4J uses the **property graph model**, which has four building blocks:

| Concept | Description | Example |
|---------|-------------|---------|
| **Node** | An entity in the domain | a Movie, a Person |
| **Label** | A type tag on a node (a node can have multiple labels) | `:Movie`, `:Person` |
| **Relationship** | A directed, named connection between two nodes | `(Person)-[:ACTED_IN]->(Movie)` |
| **Property** | A key-value pair stored on a node or relationship | `{name: 'Tom Hanks'}`, `{roles: ['Neo']}` |

Unlike a relational table, every node and relationship can carry its own set of properties — there is no fixed schema, no `NULL` column padding, and no need to normalise multi-valued attributes into junction tables.

### Graph vs Relational for movie data

Consider the question *"which actors have worked with Tom Hanks?"*

- **Relational:** two joins — `movie_person → movie → movie_person` — and a self-join on the person table
- **Graph:** one pattern — `(tom)-[:ACTED_IN]->(m)<-[:ACTED_IN]-(coActor)` — traversed in a single step

As the number of hops grows (co-actors of co-actors, shortest path between two people) the graph approach stays concise and fast while the relational approach multiplies joins.

## The Movie Graph Data Model

The graph we will build models movies, the people who made them, and some reviewers. Here is the full data model:

**Node labels:**

| Label | Properties |
|-------|------------|
| `Movie` | `title` (string), `released` (integer), `tagline` (string) |
| `Person` | `name` (string), `born` (integer) |

**Relationship types:**

| Type | Direction | Properties |
|------|-----------|------------|
| `ACTED_IN` | Person → Movie | `roles` (list of strings) |
| `DIRECTED` | Person → Movie | — |
| `PRODUCED` | Person → Movie | — |
| `WROTE` | Person → Movie | — |
| `REVIEWED` | Person → Movie | `rating` (integer), `summary` (string) |
| `FOLLOWS` | Person → Person | — |

Notice that relationships carry their own properties. `ACTED_IN` stores the character names the actor played — data that belongs on the relationship, not on either endpoint.

## Connecting to the Cypher Shell (optional)

To use the `cypher-shell`, in a terminal window execute:

```bash
docker exec -ti neo4j-1 ./bin/cypher-shell -u neo4j -p abc123abc123
```

You should see the Neo4J command prompt:

```
Connected to Neo4j using Bolt protocol version 6.0 at neo4j://localhost:7687 as user neo4j.
Type :help for a list of available commands or :exit to exit the shell.
Note that Cypher queries must end with a semicolon.
neo4j@neo4j>
```

> **What you should see:** the `neo4j@neo4j>` prompt confirming a successful connection over the Bolt protocol.

Type `:help` to see the available shell commands:

```
Available commands:
  :begin       Open a transaction
  :commit      Commit the currently open transaction
  :disconnect  Disconnects from database
  :exit        Exit the logger
  :help        Show this help message
  :history     Statement history
  :param       Set the value of a query parameter
  :rollback    Rollback the currently open transaction
  :source      Executes Cypher statements from a file
  :use         Set the active database
```

Enter `:exit` to leave the CLI. For the workshop we will use Neo4J Browser, which provides a visual canvas for exploring the graph.

## Connecting with Neo4J Browser

In a browser window, navigate to <http://dataplatform:7474>. You should land on the Neo4j Browser login screen.

![Alt Image Text](./images/neo4j-login.png "Neo4J Browser")

> **What you should see:** the login form with a Connect URL field, a Username field, and a Password field.

Enter `bolt://dataplatform:7687` into the **Connect URL**, `neo4j` into the **Username** and `abc123abc123` into the **Password** field and click **Connect**.

![Alt Image Text](./images/neo4j-browser-home.png "Neo4J Browser")

> **What you should see:** the Neo4J Browser home page with a command bar at the top and an empty canvas below.

Type Cypher queries into the command bar at the top and press **Ctrl+Enter** (or click the play button) to run them.

## Building the Movie Graph Step by Step

Rather than loading the graph all at once, we will build it in clusters. Each step introduces a new concept and adds a connected subset of the data.

### Cleaning up

If you need to start fresh at any point, run:

```cypher
MATCH (n) DETACH DELETE n
```

> **What just happened?** `MATCH (n)` matches every node in the database. `DETACH DELETE` removes each node together with all its relationships. Running this on an empty database is safe — it simply returns zero affected rows.

---

### Step 1 — Your First Node

Let's create a single `Movie` node to understand the basic syntax:

```cypher
CREATE (m:Movie {title: 'The Matrix', released: 1999, tagline: 'Welcome to the Real World'})
RETURN m
```

> **What you should see:** a single green node labelled `The Matrix` on the canvas.

> **Syntax breakdown:**
> - `CREATE` creates a new node
> - `(m:Movie {...})` — `m` is a query-local variable, `Movie` is the node label, `{...}` are the properties
> - Property values can be strings, integers, floats, booleans, or lists
> - `RETURN m` renders the created node immediately

Now verify it persisted with a `MATCH` query:

```cypher
MATCH (m:Movie {title: 'The Matrix'}) RETURN m
```

> **What you should see:** the same node as before — `MATCH` found it by its `title` property.

> **What just happened?** `MATCH` searches the graph for nodes that fit the pattern. The inline `{title: 'The Matrix'}` is a property filter — only nodes whose `title` property equals that string are returned.

---

### Step 2 — Creating a Person and a Relationship

Create a `Person` node:

```cypher
CREATE (p:Person {name: 'Keanu Reeves', born: 1964})
RETURN p
```

> **What you should see:** a single blue node labelled `Keanu Reeves`.

Now connect the person to the movie. First `MATCH` both existing nodes to bring them into scope, then `CREATE` the relationship between them:

```cypher
MATCH (keanu:Person {name: 'Keanu Reeves'})
MATCH (matrix:Movie {title: 'The Matrix'})
CREATE (keanu)-[:ACTED_IN {roles: ['Neo']}]->(matrix)
```

Verify the relationship was created:

```cypher
MATCH (p:Person {name: 'Keanu Reeves'})-[r:ACTED_IN]->(m:Movie)
RETURN p.name, r.roles, m.title
```

```
 p.name        | r.roles  | m.title
---------------|----------|------------
 Keanu Reeves  | ["Neo"]  | The Matrix
(1 row)
```

> **What you should see:** one row with the actor name, the character list, and the movie title.

> **Syntax breakdown:**
> - `(a)-[:ACTED_IN {roles: ['Neo']}]->(b)` creates a directed relationship from `a` to `b`
> - Relationship types are written in UPPERCASE by convention
> - The `roles` property is a list — Cypher list literals use `[...]`
> - Direction matters: `(Person)-[:ACTED_IN]->(Movie)`, not the other way around

---

### Step 3 — CREATE vs MERGE

Before building the full graph, it is important to understand the difference between `CREATE` and `MERGE`.

`CREATE` always creates a new node — it does not check whether a matching node already exists. Running it twice creates a duplicate.

`MERGE` works as "find or create": if a matching node exists it is returned; if not, it is created. This makes it safe to use in scripts that may be run more than once.

```cypher
// This would create a second Keanu Reeves node (avoid this):
CREATE (p:Person {name: 'Keanu Reeves', born: 1964})

// This safely finds the existing one (prefer this):
MERGE (p:Person {name: 'Keanu Reeves'})
RETURN p
```

| | `CREATE` | `MERGE` |
|-|----------|---------|
| Node already exists | Creates a duplicate | Matches the existing node |
| Node is new | Creates the node | Creates the node |
| Use when... | You are certain the node is new | You are not sure if it already exists |

> **Best practice:** always `MERGE` on a uniquely identifying property (such as `name` or `title`) to prevent partial matches from creating unexpected duplicates.

---

### Step 4 — Building the Full Matrix Cluster

Delete what we created so far and rebuild the Matrix cluster in a single statement. Comma-separated patterns inside one `CREATE` is the idiomatic way to create a connected subgraph at once:

```cypher
MATCH (n) DETACH DELETE n
```

```cypher
CREATE
  (theMatrix:Movie    {title:'The Matrix', released:1999, tagline:'Welcome to the Real World'}),
  (keanu:Person       {name:'Keanu Reeves', born:1964}),
  (carrie:Person      {name:'Carrie-Anne Moss', born:1967}),
  (laurence:Person    {name:'Laurence Fishburne', born:1961}),
  (hugo:Person        {name:'Hugo Weaving', born:1960}),
  (lilly:Person       {name:'Lilly Wachowski', born:1967}),
  (lana:Person        {name:'Lana Wachowski', born:1965}),
  (joelS:Person       {name:'Joel Silver', born:1952}),
  (keanu)-[:ACTED_IN    {roles:['Neo']}]->(theMatrix),
  (carrie)-[:ACTED_IN   {roles:['Trinity']}]->(theMatrix),
  (laurence)-[:ACTED_IN {roles:['Morpheus']}]->(theMatrix),
  (hugo)-[:ACTED_IN     {roles:['Agent Smith']}]->(theMatrix),
  (lilly)-[:DIRECTED]->(theMatrix),
  (lana)-[:DIRECTED]->(theMatrix),
  (joelS)-[:PRODUCED]->(theMatrix)
```

Verify the cluster:

```cypher
MATCH (p:Person)-[r]->(m:Movie {title:'The Matrix'})
RETURN p.name, type(r) AS relationship, r.roles
ORDER BY relationship, p.name
```

```
 p.name              | relationship | r.roles
---------------------|--------------|-------------------
 Carrie-Anne Moss    | ACTED_IN     | ["Trinity"]
 Hugo Weaving        | ACTED_IN     | ["Agent Smith"]
 Keanu Reeves        | ACTED_IN     | ["Neo"]
 Laurence Fishburne  | ACTED_IN     | ["Morpheus"]
 Lana Wachowski      | DIRECTED     | null
 Lilly Wachowski     | DIRECTED     | null
 Joel Silver         | PRODUCED     | null
(7 rows)
```

> **What you should see:** seven rows — four actors with their roles, two directors, and one producer.

> **What just happened?** `type(r)` returns the relationship type as a string. Using `[r]` without a type filter matches any relationship type between the person and the movie. Variables declared in `CREATE` are local to that statement — they cannot be reused in a separate query, which is why we use `MATCH` in the next step to retrieve them.

---

### Step 5 — The Matrix Sequels

The sequels share the same cast as the original. `MATCH` the existing actors first, then `CREATE` the new movies and their relationships:

```cypher
MATCH (keanu:Person    {name:'Keanu Reeves'}),
      (carrie:Person   {name:'Carrie-Anne Moss'}),
      (laurence:Person {name:'Laurence Fishburne'}),
      (hugo:Person     {name:'Hugo Weaving'}),
      (lilly:Person    {name:'Lilly Wachowski'}),
      (lana:Person     {name:'Lana Wachowski'})
CREATE
  (reloaded:Movie    {title:'The Matrix Reloaded', released:2003, tagline:'Free your mind'}),
  (revolutions:Movie {title:'The Matrix Revolutions', released:2003, tagline:'Everything that has a beginning has an end'}),
  (keanu)-[:ACTED_IN    {roles:['Neo']}]->(reloaded),
  (carrie)-[:ACTED_IN   {roles:['Trinity']}]->(reloaded),
  (laurence)-[:ACTED_IN {roles:['Morpheus']}]->(reloaded),
  (hugo)-[:ACTED_IN     {roles:['Agent Smith']}]->(reloaded),
  (lilly)-[:DIRECTED]->(reloaded),
  (lana)-[:DIRECTED]->(reloaded),
  (keanu)-[:ACTED_IN    {roles:['Neo']}]->(revolutions),
  (carrie)-[:ACTED_IN   {roles:['Trinity']}]->(revolutions),
  (laurence)-[:ACTED_IN {roles:['Morpheus']}]->(revolutions),
  (hugo)-[:ACTED_IN     {roles:['Agent Smith']}]->(revolutions),
  (lilly)-[:DIRECTED]->(revolutions),
  (lana)-[:DIRECTED]->(revolutions)
```

Verify that the Wachowskis now appear as directors of all three Matrix films:

```cypher
MATCH (p:Person)-[:DIRECTED]->(m:Movie)
WHERE m.title STARTS WITH 'The Matrix'
RETURN p.name, m.title ORDER BY m.title
```

```
 p.name          | m.title
-----------------|-----------------------------
 Lana Wachowski  | The Matrix
 Lilly Wachowski | The Matrix
 Lana Wachowski  | The Matrix Reloaded
 Lilly Wachowski | The Matrix Reloaded
 Lana Wachowski  | The Matrix Revolutions
 Lilly Wachowski | The Matrix Revolutions
(6 rows)
```

> **What you should see:** six rows — both Wachowskis linked to each of the three movies.

> **What just happened?** `MATCH` returned the existing person nodes as variables. `CREATE` then used those variables to attach new movie nodes and relationships directly — no risk of duplicating the people, because we started from the existing nodes rather than recreating them.

---

### Step 6 — Tom Hanks Movies

This cluster introduces Tom Hanks and several of his films. All nodes are new, so we use `CREATE` throughout:

```cypher
CREATE
  (castAway:Movie     {title:'Cast Away', released:2000, tagline:'At the edge of the world, his journey begins'}),
  (theGreenMile:Movie {title:'The Green Mile', released:1999, tagline:'Miracles do happen'}),
  (aLeague:Movie      {title:'A League of Their Own', released:1992, tagline:'Once in a lifetime you get a chance to do something different.'}),
  (tomH:Person        {name:'Tom Hanks', born:1956}),
  (helenH:Person      {name:'Helen Hunt', born:1963}),
  (robertZ:Person     {name:'Robert Zemeckis', born:1951}),
  (frankD:Person      {name:'Frank Darabont', born:1959}),
  (geenaD:Person      {name:'Geena Davis', born:1956}),
  (loriP:Person       {name:'Lori Petty', born:1963}),
  (pennyM:Person      {name:'Penny Marshall', born:1943}),
  (tomH)-[:ACTED_IN   {roles:['Chuck Noland']}]->(castAway),
  (helenH)-[:ACTED_IN {roles:['Kelly Frears']}]->(castAway),
  (robertZ)-[:DIRECTED]->(castAway),
  (tomH)-[:ACTED_IN   {roles:['Paul Edgecomb']}]->(theGreenMile),
  (frankD)-[:DIRECTED]->(theGreenMile),
  (tomH)-[:ACTED_IN   {roles:['Jimmy Dugan']}]->(aLeague),
  (geenaD)-[:ACTED_IN {roles:['Dottie Hinson']}]->(aLeague),
  (loriP)-[:ACTED_IN  {roles:['Kit Keller']}]->(aLeague),
  (pennyM)-[:DIRECTED]->(aLeague)
```

Check how many movies Tom Hanks has been added to so far:

```cypher
MATCH (tom:Person {name:'Tom Hanks'})-[:ACTED_IN]->(m:Movie)
RETURN m.title, m.released ORDER BY m.released
```

```
 m.title               | m.released
-----------------------|-----------
 A League of Their Own | 1992
 The Green Mile        | 1999
 Cast Away             | 2000
(3 rows)
```

> **What you should see:** three movies in release-year order. More will be added in later steps.

---

### Step 7 — Romantic Comedies (Meg Ryan cluster)

These movies establish the connection between Tom Hanks and Meg Ryan, which is needed for the shortest-path queries in the query section. `MATCH` Tom Hanks from the previous step, then create all new nodes:

```cypher
MATCH (tomH:Person {name:'Tom Hanks'})
CREATE
  (youGotMail:Movie    {title:"You've Got Mail", released:1998, tagline:'At odds in life... in love on-line.'}),
  (sleepless:Movie     {title:'Sleepless in Seattle', released:1993, tagline:'What if someone you never met was the only someone for you?'}),
  (whenHarry:Movie     {title:'When Harry Met Sally', released:1989, tagline:'Can two friends sleep together and still love each other in the morning?'}),
  (megR:Person         {name:'Meg Ryan', born:1961}),
  (billyC:Person       {name:'Billy Crystal', born:1948}),
  (carrieF:Person      {name:'Carrie Fisher', born:1956}),
  (brunoK:Person       {name:'Bruno Kirby', born:1949}),
  (gregK:Person        {name:'Greg Kinnear', born:1963}),
  (parkerP:Person      {name:'Parker Posey', born:1968}),
  (billP:Person        {name:'Bill Pullman', born:1953}),
  (nora:Person         {name:'Nora Ephron', born:1941}),
  (robR:Person         {name:'Rob Reiner', born:1947}),
  (tomH)-[:ACTED_IN    {roles:['Joe Fox']}]->(youGotMail),
  (megR)-[:ACTED_IN    {roles:['Kathleen Kelly']}]->(youGotMail),
  (gregK)-[:ACTED_IN   {roles:['Frank Navasky']}]->(youGotMail),
  (parkerP)-[:ACTED_IN {roles:['Patricia Eden']}]->(youGotMail),
  (nora)-[:DIRECTED]->(youGotMail),
  (nora)-[:WROTE]->(youGotMail),
  (tomH)-[:ACTED_IN    {roles:['Sam Baldwin']}]->(sleepless),
  (megR)-[:ACTED_IN    {roles:['Annie Reed']}]->(sleepless),
  (billP)-[:ACTED_IN   {roles:['Walter']}]->(sleepless),
  (nora)-[:DIRECTED]->(sleepless),
  (nora)-[:WROTE]->(sleepless),
  (megR)-[:ACTED_IN    {roles:['Sally Albright']}]->(whenHarry),
  (billyC)-[:ACTED_IN  {roles:['Harry Burns']}]->(whenHarry),
  (carrieF)-[:ACTED_IN {roles:['Marie']}]->(whenHarry),
  (brunoK)-[:ACTED_IN  {roles:['Jess']}]->(whenHarry),
  (robR)-[:DIRECTED]->(whenHarry)
```

Notice that Nora Ephron has both a `DIRECTED` and a `WROTE` relationship to *You've Got Mail* and *Sleepless in Seattle*. In a relational model this would require two rows in a junction table; here it is simply two relationship types between the same pair of nodes.

Verify that Nora Ephron appears in both roles:

```cypher
MATCH (nora:Person {name:'Nora Ephron'})-[r]->(m:Movie)
RETURN m.title, type(r) AS contribution ORDER BY m.title, contribution
```

```
 m.title              | contribution
----------------------|-------------
 Sleepless in Seattle | DIRECTED
 Sleepless in Seattle | WROTE
 You've Got Mail      | DIRECTED
 You've Got Mail      | WROTE
(4 rows)
```

> **What you should see:** Nora Ephron appearing twice for each film — once as director and once as writer.

---

### Step 8 — Tom Cruise Movies (Kevin Bacon enters)

This cluster is important for the "Six Degrees of Kevin Bacon" path query. **Kevin Bacon** appears in *A Few Good Men* alongside Tom Cruise. In Step 9 we will connect Kevin Bacon to Tom Hanks through *Apollo 13*, which creates the complete path to Meg Ryan.

`MATCH` Rob Reiner, who was created in Step 7 (he also directed *A Few Good Men*):

```cypher
MATCH (robR:Person {name:'Rob Reiner'})
CREATE
  (aFewGoodMen:Movie    {title:'A Few Good Men', released:1992, tagline:"In the heart of the nation's capital, one man will stop at nothing to keep his honor, and one will stop at nothing to find the truth."}),
  (topGun:Movie         {title:'Top Gun', released:1986, tagline:'I feel the need, the need for speed.'}),
  (jerryMaguire:Movie   {title:'Jerry Maguire', released:1996, tagline:'The rest of his life begins now.'}),
  (tomC:Person          {name:'Tom Cruise', born:1962}),
  (jackN:Person         {name:'Jack Nicholson', born:1937}),
  (demiM:Person         {name:'Demi Moore', born:1962}),
  (kevinB:Person        {name:'Kevin Bacon', born:1958}),
  (kieferS:Person       {name:'Kiefer Sutherland', born:1966}),
  (cubaG:Person         {name:'Cuba Gooding Jr.', born:1968}),
  (aaronS:Person        {name:'Aaron Sorkin', born:1961}),
  (kellyMcG:Person      {name:'Kelly McGillis', born:1957}),
  (valK:Person          {name:'Val Kilmer', born:1959}),
  (anthonyE:Person      {name:'Anthony Edwards', born:1962}),
  (tomSk:Person         {name:'Tom Skerritt', born:1933}),
  (tonyS:Person         {name:'Tony Scott', born:1944}),
  (reneeZ:Person        {name:'Renee Zellweger', born:1969}),
  (cameronC:Person      {name:'Cameron Crowe', born:1957}),
  (tomC)-[:ACTED_IN     {roles:['Lt. Daniel Kaffee']}]->(aFewGoodMen),
  (jackN)-[:ACTED_IN    {roles:['Col. Nathan R. Jessep']}]->(aFewGoodMen),
  (demiM)-[:ACTED_IN    {roles:['Lt. Cmdr. JoAnne Galloway']}]->(aFewGoodMen),
  (kevinB)-[:ACTED_IN   {roles:['Capt. Jack Ross']}]->(aFewGoodMen),
  (kieferS)-[:ACTED_IN  {roles:['Lt. Jonathan Kendrick']}]->(aFewGoodMen),
  (cubaG)-[:ACTED_IN    {roles:['Cpl. Carl Hammaker']}]->(aFewGoodMen),
  (robR)-[:DIRECTED]->(aFewGoodMen),
  (aaronS)-[:WROTE]->(aFewGoodMen),
  (tomC)-[:ACTED_IN     {roles:['Maverick']}]->(topGun),
  (kellyMcG)-[:ACTED_IN {roles:['Charlie']}]->(topGun),
  (valK)-[:ACTED_IN     {roles:['Iceman']}]->(topGun),
  (anthonyE)-[:ACTED_IN {roles:['Goose']}]->(topGun),
  (tomSk)-[:ACTED_IN    {roles:['Viper']}]->(topGun),
  (tonyS)-[:DIRECTED]->(topGun),
  (tomC)-[:ACTED_IN     {roles:['Jerry Maguire']}]->(jerryMaguire),
  (cubaG)-[:ACTED_IN    {roles:['Rod Tidwell']}]->(jerryMaguire),
  (reneeZ)-[:ACTED_IN   {roles:['Dorothy Boyd']}]->(jerryMaguire),
  (cameronC)-[:DIRECTED]->(jerryMaguire),
  (cameronC)-[:WROTE]->(jerryMaguire)
```

Notice that Cuba Gooding Jr. appears in both *A Few Good Men* and *Jerry Maguire*. Because both movies are in the same `CREATE` statement, his variable `cubaG` is reused directly — no `MATCH` needed.

---

### Step 9 — Apollo 13 (The Kevin Bacon Bridge)

*Apollo 13* is the key link: it puts **Tom Hanks** and **Kevin Bacon** in the same movie. Together with Step 7 (Hanks ↔ Meg Ryan) and Step 8 (Bacon ↔ Tom Cruise), this creates a chain of connections that the `shortestPath` query will traverse later.

`MATCH` both Tom Hanks and Kevin Bacon from previous steps:

```cypher
MATCH (tomH:Person   {name:'Tom Hanks'}),
      (kevinB:Person {name:'Kevin Bacon'})
CREATE
  (apollo13:Movie      {title:'Apollo 13', released:1995, tagline:'Houston, we have a problem.'}),
  (ronH:Person         {name:'Ron Howard', born:1954}),
  (billPax:Person      {name:'Bill Paxton', born:1955}),
  (garyS:Person        {name:'Gary Sinise', born:1955}),
  (edH:Person          {name:'Ed Harris', born:1950}),
  (tomH)-[:ACTED_IN    {roles:['Jim Lovell']}]->(apollo13),
  (kevinB)-[:ACTED_IN  {roles:['Jack Swigert']}]->(apollo13),
  (billPax)-[:ACTED_IN {roles:['Fred Haise']}]->(apollo13),
  (garyS)-[:ACTED_IN   {roles:['Ken Mattingly']}]->(apollo13),
  (edH)-[:ACTED_IN     {roles:['Gene Kranz']}]->(apollo13),
  (ronH)-[:DIRECTED]->(apollo13)
```

Verify the Kevin Bacon bridge by checking which movies he and Tom Hanks share:

```cypher
MATCH (tomH:Person {name:'Tom Hanks'})-[:ACTED_IN]->(m:Movie)<-[:ACTED_IN]-(kevinB:Person {name:'Kevin Bacon'})
RETURN m.title
```

```
 m.title
----------
 Apollo 13
(1 row)
```

> **What you should see:** *Apollo 13* — the single movie both actors appeared in. This one-hop connection is enough to link Kevin Bacon to Meg Ryan via Tom Hanks.

---

### Step 10 — Cloud Atlas and The Da Vinci Code

Both films reuse people already in the graph. `MATCH` them before creating the new movies:

```cypher
MATCH (tomH:Person   {name:'Tom Hanks'}),
      (hugo:Person   {name:'Hugo Weaving'}),
      (lilly:Person  {name:'Lilly Wachowski'}),
      (lana:Person   {name:'Lana Wachowski'}),
      (ronH:Person   {name:'Ron Howard'})
CREATE
  (cloudAtlas:Movie   {title:'Cloud Atlas', released:2012, tagline:'Everything is connected'}),
  (daVinci:Movie      {title:'The Da Vinci Code', released:2006, tagline:'Seek the Truth'}),
  (tomTyk:Person      {name:'Tom Tykwer', born:1965}),
  (halleB:Person      {name:'Halle Berry', born:1966}),
  (jimB:Person        {name:'Jim Broadbent', born:1949}),
  (ianMcK:Person      {name:'Ian McKellen', born:1939}),
  (audrey:Person      {name:'Audrey Tautou', born:1976}),
  (paulB:Person       {name:'Paul Bettany', born:1971}),
  (tomH)-[:ACTED_IN   {roles:['Zachry', 'Dr. Henry Goose', 'Isaac Sachs', 'Dermot Hoggins']}]->(cloudAtlas),
  (hugo)-[:ACTED_IN   {roles:['Bill Smoke', 'Haskell Moore', 'Tadeusz Kesselring']}]->(cloudAtlas),
  (halleB)-[:ACTED_IN {roles:['Luisa Rey', 'Jocasta Ayrs', 'Meronym']}]->(cloudAtlas),
  (jimB)-[:ACTED_IN   {roles:['Vyvyan Ayrs', 'Captain Molyneux', 'Timothy Cavendish']}]->(cloudAtlas),
  (tomTyk)-[:DIRECTED]->(cloudAtlas),
  (lana)-[:DIRECTED]->(cloudAtlas),
  (lilly)-[:DIRECTED]->(cloudAtlas),
  (tomTyk)-[:WROTE]->(cloudAtlas),
  (lana)-[:WROTE]->(cloudAtlas),
  (lilly)-[:WROTE]->(cloudAtlas),
  (tomH)-[:ACTED_IN   {roles:['Dr. Robert Langdon']}]->(daVinci),
  (ianMcK)-[:ACTED_IN {roles:['Sir Leigh Teabing']}]->(daVinci),
  (audrey)-[:ACTED_IN {roles:['Sophie Neveu']}]->(daVinci),
  (paulB)-[:ACTED_IN  {roles:['Silas']}]->(daVinci),
  (ronH)-[:DIRECTED]->(daVinci)
```

Notice that Tom Hanks plays four different characters in *Cloud Atlas* — the `roles` property is a list that can hold as many values as needed.

---

### Step 11 — As Good as It Gets, The Devil's Advocate, V for Vendetta

This cluster reuses Jack Nicholson, Helen Hunt, Greg Kinnear, Keanu Reeves, and Hugo Weaving from earlier steps:

```cypher
MATCH (jackN:Person  {name:'Jack Nicholson'}),
      (helenH:Person {name:'Helen Hunt'}),
      (gregK:Person  {name:'Greg Kinnear'}),
      (keanu:Person  {name:'Keanu Reeves'}),
      (hugo:Person   {name:'Hugo Weaving'}),
      (lilly:Person  {name:'Lilly Wachowski'})
CREATE
  (asGoodAsItGets:Movie  {title:'As Good as It Gets', released:1997, tagline:'A comedy from the heart that goes for the throat.'}),
  (devilsAdvocate:Movie  {title:"The Devil's Advocate", released:1997, tagline:'Evil has its winning ways.'}),
  (vForVendetta:Movie    {title:'V for Vendetta', released:2006, tagline:'People should not be afraid of their governments. Governments should be afraid of their people.'}),
  (jamesLB:Person        {name:'James L. Brooks', born:1940}),
  (alP:Person            {name:'Al Pacino', born:1940}),
  (charlizeT:Person      {name:'Charlize Theron', born:1975}),
  (taylorH:Person        {name:'Taylor Hackford', born:1944}),
  (natalieP:Person       {name:'Natalie Portman', born:1981}),
  (jamesMcT:Person       {name:'James McTeigue', born:1971}),
  (jackN)-[:ACTED_IN     {roles:['Melvin Udall']}]->(asGoodAsItGets),
  (helenH)-[:ACTED_IN    {roles:['Carol Connelly']}]->(asGoodAsItGets),
  (gregK)-[:ACTED_IN     {roles:['Simon Bishop']}]->(asGoodAsItGets),
  (jamesLB)-[:DIRECTED]->(asGoodAsItGets),
  (keanu)-[:ACTED_IN     {roles:['Kevin Lomax']}]->(devilsAdvocate),
  (alP)-[:ACTED_IN       {roles:['John Milton']}]->(devilsAdvocate),
  (charlizeT)-[:ACTED_IN {roles:['Mary Ann Lomax']}]->(devilsAdvocate),
  (taylorH)-[:DIRECTED]->(devilsAdvocate),
  (hugo)-[:ACTED_IN      {roles:['V']}]->(vForVendetta),
  (natalieP)-[:ACTED_IN  {roles:['Evey Hammond']}]->(vForVendetta),
  (jamesMcT)-[:DIRECTED]->(vForVendetta),
  (lilly)-[:PRODUCED]->(vForVendetta),
  (lilly)-[:WROTE]->(vForVendetta)
```

---

### Step 12 — Pulp Fiction

This final cluster shows that Quentin Tarantino both acted in and directed *Pulp Fiction* — a good example of how a single person can carry more than one relationship type to the same movie. All people here are new to the graph, so we use a single `CREATE`:

```cypher
CREATE
  (pulpFiction:Movie      {title:'Pulp Fiction', released:1994, tagline:"You won't know the facts until you've seen the fiction."}),
  (johnT:Person           {name:'John Travolta', born:1954}),
  (samuelJ:Person         {name:'Samuel L. Jackson', born:1948}),
  (bruceW:Person          {name:'Bruce Willis', born:1955}),
  (umaT:Person            {name:'Uma Thurman', born:1970}),
  (vingR:Person           {name:'Ving Rhames', born:1959}),
  (harveyK:Person         {name:'Harvey Keitel', born:1939}),
  (timR:Person            {name:'Tim Roth', born:1961}),
  (quentinT:Person        {name:'Quentin Tarantino', born:1963}),
  (johnT)-[:ACTED_IN      {roles:['Vincent Vega']}]->(pulpFiction),
  (samuelJ)-[:ACTED_IN    {roles:['Jules Winnfield']}]->(pulpFiction),
  (bruceW)-[:ACTED_IN     {roles:['Butch Coolidge']}]->(pulpFiction),
  (umaT)-[:ACTED_IN       {roles:['Mia Wallace']}]->(pulpFiction),
  (vingR)-[:ACTED_IN      {roles:['Marsellus Wallace']}]->(pulpFiction),
  (harveyK)-[:ACTED_IN    {roles:['Winston Wolfe']}]->(pulpFiction),
  (timR)-[:ACTED_IN       {roles:['Pumpkin']}]->(pulpFiction),
  (quentinT)-[:ACTED_IN   {roles:['Jimmie']}]->(pulpFiction),
  (quentinT)-[:DIRECTED]->(pulpFiction)
```

Verify that Quentin Tarantino appears in both roles for the same movie:

```cypher
MATCH (p:Person {name:'Quentin Tarantino'})-[r]->(m:Movie {title:'Pulp Fiction'})
RETURN type(r) AS contribution, r.roles
```

```
 contribution | r.roles
--------------|-------------
 ACTED_IN     | ["Jimmie"]
 DIRECTED     | null
(2 rows)
```

> **What you should see:** two rows for the same person and movie — one `ACTED_IN` relationship carrying his character name, and one `DIRECTED` relationship with no properties.

> **What just happened?** In a relational schema, a person who both acted and directed would need two rows in the junction table distinguished by a role discriminator column. In the graph model, they are simply two different relationship types between the same pair of nodes — no discriminator column, no ambiguity, and each relationship can carry its own independent properties.

---

### Step 13 — REVIEWED and FOLLOWS Relationships

The `REVIEWED` relationship adds two properties — `rating` (0–100) and `summary` — to model a social review graph. The `FOLLOWS` relationship models connections between reviewers. These are new people who have not appeared before, so we use `CREATE` for them and `MATCH` for the movies they reviewed:

```cypher
CREATE
  (jessicaT:Person   {name:'Jessica Thompson'}),
  (jamesT:Person     {name:'James Thompson'}),
  (angelaS:Person    {name:'Angela Scope'})
WITH jessicaT, jamesT, angelaS
MATCH (cloudAtlas:Movie {title:'Cloud Atlas'}),
      (daVinci:Movie    {title:'The Da Vinci Code'}),
      (matrix:Movie     {title:'The Matrix'}),
      (topGun:Movie     {title:'Top Gun'})
CREATE
  (jessicaT)-[:REVIEWED {rating:95, summary:'An amazing journey across time and lives.'}]->(cloudAtlas),
  (jessicaT)-[:REVIEWED {rating:65, summary:'Masterful! The visuals were stunning.'}]->(daVinci),
  (jamesT)-[:REVIEWED   {rating:55, summary:'A solid film but the book was better.'}]->(daVinci),
  (angelaS)-[:REVIEWED  {rating:45, summary:'Not my cup of tea.'}]->(daVinci),
  (jessicaT)-[:REVIEWED {rating:85, summary:'Keanu was born for this role.'}]->(matrix),
  (angelaS)-[:REVIEWED  {rating:70, summary:'Great action but thin on plot.'}]->(topGun),
  (jessicaT)-[:FOLLOWS]->(angelaS),
  (jessicaT)-[:FOLLOWS]->(jamesT),
  (angelaS)-[:FOLLOWS]->(jamesT)
```

> **What just happened?** `WITH jessicaT, jamesT, angelaS` passes the three newly created nodes through to the next clause, which then uses `MATCH` to find the movies. Without `WITH`, Cypher would not know which nodes to carry forward from `CREATE` into `MATCH`.

---

### Verify the Complete Graph

Check the total count of nodes by label:

```cypher
MATCH (n) RETURN labels(n)[0] AS label, COUNT(n) AS count ORDER BY label
```

```
 label  | count
--------|------
 Movie  |    19
 Person |    64
(2 rows)
```

Check the total count of relationships by type:

```cypher
MATCH ()-[r]->() RETURN type(r) AS relationship, COUNT(r) AS count ORDER BY relationship
```

```
 relationship | count
--------------|------
 ACTED_IN     |    72
 DIRECTED     |    24
 FOLLOWS      |     3
 PRODUCED     |     2
 REVIEWED     |     6
 WROTE        |     8
(6 rows)
```

> **What you should see:** 19 movies, roughly 64 people, and relationships across all six types. The exact numbers may vary if you ran any steps more than once.

---

## Indexes and Constraints

Neo4J can create indexes to speed up node lookups by property. Create explicit uniqueness constraints to prevent duplicate movies or people from being inserted:

```cypher
CREATE CONSTRAINT movie_title_unique IF NOT EXISTS
FOR (m:Movie) REQUIRE m.title IS UNIQUE
```

```cypher
CREATE CONSTRAINT person_name_unique IF NOT EXISTS
FOR (p:Person) REQUIRE p.name IS UNIQUE
```

View all indexes:

```cypher
SHOW INDEXES
```

![Alt Image Text](./images/neo4j-show-indexes.png "Neo4J Browser")

> **What you should see:** at least the two constraints you just created, shown as `RANGE` index type with `UNIQUENESS` constraint type. Any indexes created automatically during the build steps also appear here.

> **What just happened?** A uniqueness constraint in Neo4J implicitly creates a backing index on that property. Every subsequent `MATCH (m:Movie {title: '...'})` will use the index instead of scanning all nodes, and any attempt to `CREATE` a duplicate will fail with a constraint violation error.

## Viewing the Database

Click on the database icon in the top-left corner of the Neo4J Browser.

![Alt Image Text](./images/neo4j-database-view.png "Neo4J Browser")

> **What you should see:** a sidebar listing Node Labels (`Movie`, `Person`), Relationship Types (`ACTED_IN`, `DIRECTED`, `PRODUCED`, `REVIEWED`, `FOLLOWS`, `WROTE`), and Property Keys — each with a count. Clicking any label runs a quick `MATCH` and displays a sample of nodes on the canvas.

---

## Querying the Graph

### Basic Node Lookup

Find a specific person by name:

```cypher
MATCH (p:Person {name: 'Tom Hanks'}) RETURN p
```

> **What you should see:** a single Person node labelled "Tom Hanks" on the canvas.

Find a movie by title:

```cypher
MATCH (m:Movie {title: 'The Matrix'}) RETURN m
```

Double-click the node to expand all its relationships and see the connected people.

![Alt Image Text](./images/neo4j-show-related-nodes.png "Neo4J Browser")

> **What you should see:** all Person nodes connected to The Matrix node via labelled edges.

> **What just happened?** Double-clicking a node in the browser triggers a `MATCH (n)-[r]-() WHERE id(n) = <id> RETURN r` query automatically, fetching all neighbours and rendering them on the canvas.

List all movies ordered by release year:

```cypher
MATCH (m:Movie) RETURN m.title, m.released ORDER BY m.released
```

```
 m.title                     | m.released
-----------------------------|------------
 Top Gun                     | 1986
 When Harry Met Sally        | 1989
 A League of Their Own       | 1992
 A Few Good Men              | 1992
 Sleepless in Seattle        | 1993
 Pulp Fiction                | 1994
 The Green Mile              | 1999
 The Matrix                  | 1999
 ...
(17 rows)
```

---

### Filtering with WHERE

`WHERE` allows conditions that are more expressive than inline property maps.

Find all movies released after 2000:

```cypher
MATCH (m:Movie)
WHERE m.released > 2000
RETURN m.title, m.released ORDER BY m.released
```

```
 m.title                | m.released
------------------------|------------
 The Matrix Reloaded    | 2003
 The Matrix Revolutions | 2003
 The Da Vinci Code      | 2006
 V for Vendetta         | 2006
 Cloud Atlas            | 2012
(5 rows)
```

Find movies released between 1990 and 2000 (inclusive):

```cypher
MATCH (m:Movie)
WHERE m.released >= 1990 AND m.released <= 2000
RETURN m.title, m.released ORDER BY m.released
```

Find all people born in the 1960s:

```cypher
MATCH (p:Person)
WHERE p.born >= 1960 AND p.born < 1970
RETURN p.name, p.born ORDER BY p.born
```

```
 p.name              | p.born
---------------------|-------
 Hugo Weaving        | 1960
 Laurence Fishburne  | 1961
 Meg Ryan            | 1961
 Aaron Sorkin        | 1961
 Tim Roth            | 1961
 Tom Cruise          | 1962
 Demi Moore          | 1962
 Anthony Edwards     | 1962
 Helen Hunt          | 1963
 ...
(13 rows)
```

Find movies whose title contains a specific word:

```cypher
MATCH (m:Movie)
WHERE m.title CONTAINS 'Matrix'
RETURN m.title, m.released
```

```
 m.title                 | m.released
-------------------------|------------
 The Matrix              | 1999
 The Matrix Reloaded     | 2003
 The Matrix Revolutions  | 2003
(3 rows)
```

Find people whose name starts with "Tom":

```cypher
MATCH (p:Person)
WHERE p.name STARTS WITH 'Tom'
RETURN p.name ORDER BY p.name
```

```
 p.name
--------------
 Tom Cruise
 Tom Hanks
 Tom Skerritt
 Tom Tykwer
(4 rows)
```

> **What just happened?** `CONTAINS`, `STARTS WITH`, and `ENDS WITH` are Cypher string predicates that work like SQL `LIKE` — but they are case-sensitive. For case-insensitive matching, use `toLower(p.name) CONTAINS 'tom'`.

---

### Relationship Traversal

Find all movies Tom Hanks acted in:

```cypher
MATCH (tom:Person {name:'Tom Hanks'})-[:ACTED_IN]->(m:Movie)
RETURN m.title, m.released ORDER BY m.released
```

```
 m.title                | m.released
------------------------|------------
 A League of Their Own  | 1992
 Sleepless in Seattle   | 1993
 Apollo 13              | 1995
 You've Got Mail        | 1998
 The Green Mile         | 1999
 Cast Away              | 2000
 The Da Vinci Code      | 2006
 Cloud Atlas            | 2012
(8 rows)
```

Find all actors in *The Matrix* and their roles:

```cypher
MATCH (p:Person)-[r:ACTED_IN]->(m:Movie {title:'The Matrix'})
RETURN p.name, r.roles
```

```
 p.name              | r.roles
---------------------|-------------------
 Keanu Reeves        | ["Neo"]
 Carrie-Anne Moss    | ["Trinity"]
 Laurence Fishburne  | ["Morpheus"]
 Hugo Weaving        | ["Agent Smith"]
(4 rows)
```

> **What you should see:** one row per actor. The `r.roles` column shows the list of character names stored on the relationship.

Find who directed each movie:

```cypher
MATCH (p:Person)-[:DIRECTED]->(m:Movie)
RETURN p.name AS director, m.title AS movie ORDER BY p.name
```

Find who wrote each movie:

```cypher
MATCH (p:Person)-[:WROTE]->(m:Movie)
RETURN p.name AS writer, m.title AS movie ORDER BY p.name
```

Find all types of contribution a person has made (any relationship type):

```cypher
MATCH (p:Person {name:'Tom Hanks'})-[r]->(x)
RETURN type(r) AS relationship, x.title AS title ORDER BY relationship
```

```
 relationship | title
--------------|------------------------
 ACTED_IN     | A League of Their Own
 ACTED_IN     | Sleepless in Seattle
 ACTED_IN     | You've Got Mail
 ACTED_IN     | Cast Away
 ACTED_IN     | Apollo 13
 ACTED_IN     | The Green Mile
 ACTED_IN     | Cloud Atlas
 ACTED_IN     | The Da Vinci Code
(8 rows)
```

---

### Pattern Matching Across Multiple Hops

Find all of Tom Hanks' co-actors (people who appeared in any movie with him):

```cypher
MATCH (tom:Person {name:'Tom Hanks'})-[:ACTED_IN]->(m:Movie)<-[:ACTED_IN]-(coActor:Person)
RETURN DISTINCT coActor.name ORDER BY coActor.name
```

```
 coActor.name
--------------
 Bill Paxton
 Bill Pullman
 Billy Crystal
 Ed Harris
 Gary Sinise
 Geena Davis
 Greg Kinnear
 Halle Berry
 ...
(18 rows)
```

> **What just happened?** The pattern `(tom)-[:ACTED_IN]->(m)<-[:ACTED_IN]-(coActor)` traverses two hops: from Tom Hanks out to movies he acted in, then back in to all other actors in those same movies. `DISTINCT` removes duplicates in case an actor appeared in more than one film with Tom Hanks.

Find Tom Hanks' co-actors and the movies they share:

```cypher
MATCH (tom:Person {name:'Tom Hanks'})-[:ACTED_IN]->(m:Movie)<-[:ACTED_IN]-(coActor:Person)
RETURN coActor.name, COLLECT(m.title) AS sharedMovies ORDER BY coActor.name
```

```
 coActor.name    | sharedMovies
-----------------|--------------------------
 Bill Paxton     | ["Apollo 13"]
 Kevin Bacon     | ["Apollo 13"]
 Meg Ryan        | ["Sleepless in Seattle", "You've Got Mail"]
 ...
```

> **What just happened?** `COLLECT(m.title)` groups all matching movie titles into a list per co-actor, rather than returning one row per movie. This is the graph equivalent of `STRING_AGG` in SQL.

---

### Aggregation

Count how many movies each actor appeared in (top 10):

```cypher
MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
RETURN p.name, COUNT(m) AS movieCount
ORDER BY movieCount DESC LIMIT 10
```

```
 p.name              | movieCount
---------------------|------------
 Tom Hanks           | 8
 Hugo Weaving        | 5
 Keanu Reeves        | 4
 Tom Cruise          | 3
 Carrie-Anne Moss    | 3
 Laurence Fishburne  | 3
 ...
(10 rows)
```

Count how many actors appear in each movie:

```cypher
MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
RETURN m.title, COUNT(p) AS actorCount
ORDER BY actorCount DESC
```

Count movies per release decade:

```cypher
MATCH (m:Movie)
RETURN (m.released / 10) * 10 AS decade, COUNT(m) AS movieCount
ORDER BY decade
```

```
 decade | movieCount
--------|------------
 1980   |          2
 1990   |          11
 2000   |          5
 2010   |          1
 ...
```

Find people who both acted and directed:

```cypher
MATCH (p:Person)-[:ACTED_IN]->()
MATCH (p)-[:DIRECTED]->()
RETURN DISTINCT p.name
```

```
 p.name             |
--------------------|
 Quentin Tarantino  |
```

---

### OPTIONAL MATCH

`OPTIONAL MATCH` is the graph equivalent of a LEFT JOIN. It returns `null` for the optional part when no match is found, rather than omitting the row entirely.

List all people and any movies they directed (null if they never directed):

```cypher
MATCH (p:Person)
OPTIONAL MATCH (p)-[:DIRECTED]->(m:Movie)
RETURN p.name, m.title AS directed
ORDER BY p.name
```

```
 p.name              | directed
---------------------|-------------------
 Aaron Sorkin        | null
 Al Pacino           | null
 Angela Scope        | null
 Anthony Edwards     | null
 ...
 Cameron Crowe       | Jerry Maguire
 Carrie-Anne Moss    | null
 Frank Darabont      | The Green Mile
 ...
```

> **What you should see:** all people listed, with `null` in the `directed` column for those who have no `DIRECTED` relationship. Without `OPTIONAL MATCH`, people who have never directed would be excluded from the results entirely.

List all movies and their directors (null if no director was recorded):

```cypher
MATCH (m:Movie)
OPTIONAL MATCH (p:Person)-[:DIRECTED]->(m)
RETURN m.title, p.name AS director
ORDER BY m.title
```

---

### Chaining Queries with WITH

`WITH` pipes the output of one part of the query into the next — similar to a CTE in SQL. It is commonly used to filter on aggregated values.

Find only actors who have appeared in three or more movies:

```cypher
MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
WITH p, COUNT(m) AS movieCount
WHERE movieCount >= 3
RETURN p.name, movieCount
ORDER BY movieCount DESC
```

```
 p.name              | movieCount
---------------------|------------
 Tom Hanks           | 8
 Hugo Weaving        | 5
 Keanu Reeves        | 4
 Tom Cruise          | 3
 Laurence Fishburne  | 3
 Carrie-Anne Moss    | 3
 Meg Ryan.           | 3
 Tom Cruise          | 3
(8 rows)
```

> **What just happened?** The `WHERE movieCount >= 3` clause can only be written after `WITH p, COUNT(m) AS movieCount`, because `COUNT` is an aggregation and does not exist until that point. This is the same reason SQL uses `HAVING` rather than `WHERE` for post-aggregation filters.

Find the average release year of movies each person acted in (actors with 2+ films only):

```cypher
MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
WITH p, AVG(m.released) AS avgYear, COUNT(m) AS movieCount
WHERE movieCount >= 2
RETURN p.name, ROUND(avgYear) AS avgReleaseYear, movieCount
ORDER BY avgYear
```

Find the top co-actor suggestions for Tom Hanks — people connected through his co-actors who are not already his direct co-actors:

```cypher
MATCH (tom:Person {name:'Tom Hanks'})-[:ACTED_IN]->(m1)<-[:ACTED_IN]-(coActor)
WITH tom, COLLECT(DISTINCT coActor) AS coActors
UNWIND coActors AS coActor
MATCH (coActor)-[:ACTED_IN]->(m2)<-[:ACTED_IN]-(candidate)
WHERE NOT candidate IN coActors AND candidate <> tom
RETURN DISTINCT candidate.name AS suggestion
ORDER BY suggestion LIMIT 10
```

---

### Path Queries

The shortest path between Kevin Bacon and Meg Ryan across any relationship:

```cypher
MATCH p=shortestPath(
  (bacon:Person {name:'Kevin Bacon'})-[*]-(meg:Person {name:'Meg Ryan'})
)
RETURN p
```

![Alt Image Text](./images/neo4j-shortest-path.png "Neo4J Browser")

> **What you should see:** a chain of nodes and relationships showing the shortest sequence of connections between Kevin Bacon and Meg Ryan.

> **What just happened?** `shortestPath` runs a breadth-first search through the entire graph and returns the minimum-hop path. The `[*]` pattern matches any relationship of any type in either direction. The path passes through *Apollo 13* (Bacon ↔ Hanks) and then *Sleepless in Seattle* or *You've Got Mail* (Hanks ↔ Ryan).

Find all shortest paths (there may be more than one):

```cypher
MATCH p=allShortestPaths(
  (bacon:Person {name:'Kevin Bacon'})-[*]-(meg:Person {name:'Meg Ryan'})
)
RETURN p
```

![Alt Image Text](./images/neo4j-all-shortest-path.png "Neo4J Browser")

Inspect the nodes along a path explicitly:

```cypher
MATCH p=shortestPath(
  (bacon:Person {name:'Kevin Bacon'})-[*]-(meg:Person {name:'Meg Ryan'})
)
UNWIND nodes(p) AS n
RETURN n.name, n.title
```

```
 n.name       | n.title
--------------|---------------------
 Kevin Bacon  | null
 null         | Apollo 13
 Tom Hanks    | null
 null         | You've Got Mail
 Meg Ryan     | null
(5 rows)
```

> **What just happened?** `nodes(p)` returns the ordered list of nodes along the path. `UNWIND` expands the list into individual rows. Because Movie nodes have `title` but no `name`, and Person nodes have `name` but no `title`, exactly one column is non-null per row.

Find all people within 2 hops of Tom Hanks via any relationship:

```cypher
MATCH (tom:Person {name:'Tom Hanks'})-[*1..2]-(p:Person)
WHERE tom <> p
RETURN DISTINCT p.name ORDER BY p.name
```

> **What just happened?** `[*1..2]` is a variable-length pattern — it matches paths of between 1 and 2 relationship hops. Increasing this number quickly expands to most of the graph, which is why path queries typically use `shortestPath` or a bounded hop count.

---

### Working with REVIEWED and FOLLOWS

List all reviews with rating and summary:

```cypher
MATCH (p:Person)-[r:REVIEWED]->(m:Movie)
RETURN p.name, m.title, r.rating, r.summary
ORDER BY r.rating DESC
```

```
 p.name             | m.title       | r.rating | r.summary
--------------------|---------------|----------|----------------------------------------------
 Jessica Thompson   | Cloud Atlas   | 95       | An amazing journey across time and lives.
 Jessica Thompson   | The Matrix    | 85       | Keanu was born for this role.
 Jessica Thompson   | The Da Vinci  | 65       | Masterful! The visuals were stunning.
 Angela Scope       | Top Gun       | 70       | Great action but thin on plot.
 James Thompson     | The Da Vinci  | 55       | A solid film but the book was better.
 Angela Scope       | The Da Vinci  | 45       | Not my cup of tea.
(6 rows)
```

Find the average rating per reviewed movie:

```cypher
MATCH (p:Person)-[r:REVIEWED]->(m:Movie)
WITH m, AVG(r.rating) AS avgRating, COUNT(r) AS reviewCount
RETURN m.title, ROUND(avgRating) AS avgRating, reviewCount
ORDER BY avgRating DESC
```

```
 m.title         | avgRating | reviewCount
-----------------|-----------|-------------
 Cloud Atlas     | 95        | 1
 The Matrix      | 85        | 1
 Top Gun         | 70        | 1
 The Da Vinci Code | 55      | 3
(4 rows)
```

Find who each reviewer follows, and whether they reviewed any movies in common:

```cypher
MATCH (follower:Person)-[:FOLLOWS]->(followed:Person)
OPTIONAL MATCH (follower)-[:REVIEWED]->(m:Movie)<-[:REVIEWED]-(followed)
RETURN follower.name, followed.name, COLLECT(m.title) AS sharedReviews
```

```
 follower.name      | followed.name    | sharedReviews
--------------------|------------------|------------------------
 Jessica Thompson   | Angela Scope     | ["The Da Vinci Code"]
 Jessica Thompson   | James Thompson   | ["The Da Vinci Code"]
 Angela Scope       | James Thompson   | ["The Da Vinci Code"]
(3 rows)
```

> **What you should see:** the reviewer pairs and the movies they both reviewed. *The Da Vinci Code* was reviewed by all three reviewers, so it appears in every pair's shared list.

---

### Recommendations

Suggest movies that Tom Hanks' co-actors appeared in, that Tom Hanks himself has not:

```cypher
MATCH (tom:Person {name:'Tom Hanks'})-[:ACTED_IN]->(m1)<-[:ACTED_IN]-(coActor)-[:ACTED_IN]->(rec:Movie)
WHERE NOT (tom)-[:ACTED_IN]->(rec)
RETURN DISTINCT rec.title AS recommendation, coActor.name AS via
ORDER BY recommendation
```

Recommend movies based on shared actors with a given film (collaborative filtering):

```cypher
MATCH (liked:Movie {title:'The Matrix'})<-[:ACTED_IN]-(p:Person)-[:ACTED_IN]->(rec:Movie)
WHERE liked <> rec
RETURN rec.title AS recommendation, COUNT(p) AS sharedActors
ORDER BY sharedActors DESC
```

```
 recommendation          | sharedActors
-------------------------|-------------
 The Matrix Reloaded     | 4
 The Matrix Revolutions  | 4
 Cloud Atlas             | 1
 The Devil's Advocate    | 1
 V for Vendetta          | 1
(5 rows)
```

> **What just happened?** The query counts how many actors from *The Matrix* also appear in each other movie. The two sequels share the most actors, so they rank highest — exactly the kind of signal a recommendation engine would use.

Find reviewers with similar taste — pairs who reviewed the same movie with ratings within 15 points:

```cypher
MATCH (a:Person)-[r1:REVIEWED]->(m:Movie)<-[r2:REVIEWED]-(b:Person)
WHERE a <> b AND ABS(r1.rating - r2.rating) <= 15
RETURN a.name, b.name, m.title, r1.rating AS ratingA, r2.rating AS ratingB
ORDER BY m.title
```

---

### Updating Data

Change the tagline of an existing movie using `SET`:

```cypher
MATCH (m:Movie {title:'The Matrix'})
SET m.tagline = 'Welcome to the Real World.'
RETURN m.title, m.tagline
```

```
 m.title    | m.tagline
------------|----------------------------
 The Matrix | Welcome to the Real World.
(1 row)
```

Add a new property that did not exist before:

```cypher
MATCH (m:Movie {title:'The Matrix'})
SET m.budget = 63000000
RETURN m
```

Add a second label to a node:

```cypher
MATCH (m:Movie {title:'The Matrix'})
SET m:SciFi
RETURN labels(m)
```

```
 labels(m)
---------------
 ["Movie", "SciFi"]
(1 row)
```

Remove a property from a node:

```cypher
MATCH (m:Movie {title:'The Matrix'})
REMOVE m.budget
RETURN m
```

Remove the extra label:

```cypher
MATCH (m:Movie {title:'The Matrix'})
REMOVE m:SciFi
RETURN labels(m)
```

Update a property on a relationship:

```cypher
MATCH (:Person {name:'Jessica Thompson'})-[r:REVIEWED]->(:Movie {title:'Cloud Atlas'})
SET r.rating = 98
RETURN r.rating, r.summary
```

---

### Deleting Data

Delete a specific relationship (both nodes remain):

```cypher
MATCH (:Person {name:'Angela Scope'})-[r:REVIEWED]->(:Movie {title:'Top Gun'})
DELETE r
```

> **What you should see:** `Deleted 1 relationships` — the relationship is removed but Angela Scope and Top Gun still exist as nodes.

Delete a node that has no relationships:

```cypher
MATCH (p:Person {name:'Angela Scope'})
DELETE p
```

> **Note:** this will fail if the node still has any relationships. Use `DETACH DELETE` to remove the node and all its relationships in one step:

```cypher
MATCH (p:Person {name:'Angela Scope'})
DETACH DELETE p
```

> **What just happened?** `DETACH DELETE` first removes all relationships attached to the node, then deletes the node itself. It is the safe default for deleting any node. Plain `DELETE` is a safeguard — it prevents accidental deletion of a node that is still connected to others.

---

## Working with Neo4J from Python

The official Neo4J Python driver (`neo4j`) lets you run Cypher queries directly from Python code. In this section we will connect from the **Jupyter** environment and demonstrate the most important operations using the movie graph we built above.

Open a browser and navigate to <http://dataplatform:28888> and log in with token `abc123!`.

Create a new Python 3 notebook by clicking on the **Python 3 (ipykernel)** widget, then work through the cells below in order.

### Cell 1 — Install the driver

```python
import sys
!{sys.executable} -m pip install neo4j
```

> **What you should see:** pip output ending with `Successfully installed neo4j-...`.

### Cell 2 — Connect to Neo4J

```python
from neo4j import GraphDatabase

URI  = "bolt://neo4j-1:7687"
AUTH = ("neo4j", "abc123abc123")

driver = GraphDatabase.driver(URI, auth=AUTH)
driver.verify_connectivity()
print("Connected to Neo4J")
```

> **What you should see:** `Connected to Neo4J` — the driver verified it can reach the database over the Bolt protocol.

`verify_connectivity()` raises an exception immediately if the host is unreachable or the credentials are wrong, so problems surface here rather than during the first query.

### Cell 3 — Run a read query

`execute_query` is the simplest way to run a query and get results back as a list of records. Parameters are passed as keyword arguments and referenced in Cypher with `$name` syntax, which prevents injection issues.

```python
records, summary, keys = driver.execute_query(
    """
    MATCH (p:Person {name: $name})-[:ACTED_IN]->(m:Movie)
    RETURN m.title AS title, m.released AS released
    ORDER BY m.released
    """,
    name="Tom Hanks",
    database_="neo4j"
)

for record in records:
    print(record["title"], record["released"])
```

```
A League of Their Own 1992
Sleepless in Seattle 1993
Apollo 13 1995
You've Got Mail 1998
The Green Mile 1999
Cast Away 2000
The Da Vinci Code 2006
Cloud Atlas 2012
```

> **What you should see:** the eight movies Tom Hanks acted in, in release-year order.

### Cell 4 — Parameterized write query

Use `execute_query` for writes as well. `MERGE` ensures that running the cell more than once does not create duplicate nodes or relationships:

```python
driver.execute_query(
    """
    MERGE (p:Person {name: $name})
    ON CREATE SET p.born = $born
    MERGE (m:Movie  {title: $title})
    MERGE (p)-[:REVIEWED {rating: $rating, summary: $summary}]->(m)
    """,
    name="Guido Schmutz",
    born=1968,
    title="Pulp Fiction",
    rating=99,
    summary="An absolute masterpiece.",
    database_="neo4j"
)
print("Review added")
```

> **What you should see:** `Review added` with no errors.

> **What just happened?** Each `$parameter` in the Cypher string is substituted safely by the driver — the values are never concatenated into the query string. This is the graph equivalent of a prepared statement in SQL.

Verify the new review was written:

```python
records, _, _ = driver.execute_query(
    """
    MATCH (p:Person {name: $name})-[r:REVIEWED]->(m:Movie)
    RETURN m.title AS movie, r.rating AS rating, r.summary AS summary
    """,
    name="Guido Schmutz",
    database_="neo4j"
)

for record in records:
    print(record["movie"], record["rating"], record["summary"])
# Pulp Fiction 99 An absolute masterpiece.
```

### Cell 5 — Traversal query: co-actors

This query traverses two hops — from a person to their movies, then back out to everyone else in those movies:

```python
records, _, _ = driver.execute_query(
    """
    MATCH (p:Person {name: $name})-[:ACTED_IN]->(m:Movie)<-[:ACTED_IN]-(coActor:Person)
    RETURN DISTINCT coActor.name AS coActor, COLLECT(m.title) AS sharedMovies
    ORDER BY coActor
    """,
    name="Tom Hanks",
    database_="neo4j"
)

for record in records:
    print(f"{record['coActor']:25s}  {record['sharedMovies']}")
```

```
Audrey Tautou              ['The Da Vinci Code']
Bill Paxton                ['Apollo 13']
Bill Pullman               ['Sleepless in Seattle']
Ed Harris                  ['Apollo 13']
Gary Sinise                ['Apollo 13']
Geena Davis                ['A League of Their Own']
Greg Kinnear               ["You've Got Mail"]
Halle Berry                ['Cloud Atlas']
Helen Hunt                 ['Cast Away']
Hugo Weaving               ['Cloud Atlas']
Ian McKellen               ['The Da Vinci Code']
Jim Broadbent              ['Cloud Atlas']
Kevin Bacon                ['Apollo 13']
Lori Petty                 ['A League of Their Own']
Meg Ryan                   ['Sleepless in Seattle', "You've Got Mail"]
Parker Posey               ["You've Got Mail"]
Paul Bettany               ['The Da Vinci Code']
```

> **What you should see:** Tom Hanks' co-actors with the list of films they share. `COLLECT` returns a Python list in the driver result.

### Cell 6 — Close the driver

Always close the driver when you are finished to release the underlying connection pool:

```python
driver.close()
print("Driver closed")
```

---

## Using MCP with Neo4J

**Model Context Protocol (MCP)** is an open standard that lets AI assistants connect to external tools and data sources through a common interface. Instead of writing Cypher queries manually, you can give an AI assistant access to your Neo4J graph via an MCP server and interact with it in natural language.

The data platform includes two components for this:

| Component | URL | Purpose |
|-----------|-----|---------|
| **Neo4J MCP Server** | `http://dataplatform:28271/mcp` | Exposes the Neo4J graph as MCP tools (Streamable HTTP transport) |
| **MCP Inspector** | <http://dataplatform:6274> | Web UI for exploring and testing any MCP server |

The Neo4J MCP server uses the `mcp/neo4j` image, connected to `bolt://neo4j-1:7687` on the `neo4j` database with `NEO4J_READ_ONLY: false`, meaning it supports both read and write operations.

### Connecting an AI assistant to the MCP server

Any MCP-compatible AI client (Claude Desktop, Cursor, VS Code with an MCP extension) can connect to the Neo4J MCP server.

To connect from **Claude Desktop**:

1. Navigate to **Settings**.
2. Click on **Developer** in the menu on the left side.
3. Click on **Edit Config**, open `claude_desktop_config.json` in an editor, and add the following:

```json
{
  "mcpServers": {
    "neo4j": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote",
        "http://63.178.5.123:28271/mcp",
        "--allow-http",
        "--debug",
        "--header",
        "Authorization: Basic bmVvNGo6YWJjMTIzYWJjMTIz"
      ]
    }
  }
}
```

Replace the IP address with the one of the server where the data platform is running.

4. Save the file and restart **Claude Desktop**.

Once connected, the assistant can answer questions like:

- *"Which actors have appeared in the most movies in the graph?"*
- *"Find the shortest path between Kevin Bacon and Meg Ryan."*
- *"Which movies did Tom Hanks and Meg Ryan both appear in?"*
- *"Add a review for Pulp Fiction with a rating of 95."*
- *"What relationship types does the graph contain?"*

The assistant translates these into Cypher queries that run against your Neo4J database and returns the results in plain language.

> **What just happened?** The MCP server acts as a bridge between the AI assistant and Neo4J. The assistant sees three typed tools — schema inspection, read queries, and write queries — rather than a raw Bolt connection. This means the assistant knows exactly what operations are available, can compose valid Cypher from a natural-language question, and the `NEO4J_READ_ONLY` flag gives you a single switch to make the connection read-only if needed.

We can also use MCP to add new data to the graph

- *"get information about the movie "Die Hard 1" and add it to the graph"*
- *"also get it for Die Hard 2 and Die Hard 3"*
- *"Retrieve all movies from the graph, determine the genre, add the genre as a Genre node and connect the movie to it"*

After the last request the graph should also include **Genre** nodes

![Alt Image Text](./images/neo4j-after-mcp-add.png "Neo4J Browser")
