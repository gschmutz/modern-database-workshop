# Working with GraphDB

In this workshop we will learn how to use the Ontotext GraphDB NoSQL database.

We assume that the platform described [here](../01-environment/README.md) is running and accessible. 

In this workshop you learn how to load RDF data into GraphDB and then use the SPARQL query language to query the data.

## What you will learn

- How to create a repository in GraphDB Workbench
- How to load RDF data from a URL into GraphDB
- How to explore and visualise the graph in the GraphDB Workbench
- How to write SPARQL queries to retrieve and filter RDF graph data

## Prerequisites

- The **Data Platform** described [here](../01-environment/README.md) is running and accessible

## Loading RDF data

We will use the Movies data taken from a tutorial provided by GraphDB. The data is available in turtle syntax, a common data format for storing RDF data in the GitHub project under this link <https://raw.githubusercontent.com/gschmutz/nosql-workshop/master/07-working-with-graphdb/data/movies.ttl>. If you click on the link you will see the data as shown below 

![](./images/graphdb-movies-data.png)

We will use the GraphDB workbench to load the data. In a browser, navigate to <http://dataplatform:17200> to open the GraphDB workbench.

![](./images/graphdb-workbench-1.png)

Click on **Create new repository** and select **GraphDB Repository**.

![](./images/graphdb-workbench-2.png)

Enter `Movies` into the **Repository** field 

![](./images/graphdb-workbench-3.png)

and click **Create**.

In the menu bar to the left, click on **Import**.

![](./images/graphdb-import-1.png)

Click on **Movies** and select the 2nd option **Get RDF data from a URL**

![](./images/graphdb-import-2.png)

On the pop-up window enter [`https://raw.githubusercontent.com/gschmutz/nosql-workshop/master/07-working-with-graphdb/data/movies.ttl`](https://raw.githubusercontent.com/gschmutz/nosql-workshop/master/07-working-with-graphdb/data/movies.ttl) into the field

![](./images/graphdb-import-3.png)

leave **Start import automatically** selected and click on **Import**. Leave the defaults on the other pop-up window and click **Import** again. 

After a few seconds the input should end with a successful message as shown in the following screenshot. 

![](./images/graphdb-import-4.png)

Now the database is ready to be used.

## Exploring the graph

Click on **Explore** and **Class hierarchy**

![](./images/graphdb-explore-1.png)

We can see the schema of the movies graph with the `schema:Movie` base class and the two subclasses `imbd:BlackAndWhiteMovie` and `imdb:ColorMovie`.

Click on the larger, inner circle, representing the color movies

![](./images/graphdb-explore-2.png)

We can see that there are 4'690 instances of color movies in the graph, with a selection of a a few movie titles displayed to the right. 

You can either directly click on one of the titles shown or use the search to find a certain movie. Let's type in `matrix` 

![](./images/graphdb-explore-3.png)

to find the movie **TheMatrix**. Click on it and you will see the triples listed belonging to this movie. 

![](./images/graphdb-explore-4.png)

Click on **Visual Graph** to see the graph visually

![](./images/graphdb-explore-5.png)

The beauty of the Visual Graph is that you can double click on one of the nodes to expand the graph. Let's try that on the Actor **KeanuReeves** and the graph should expand like shown below

![](./images/graphdb-explore-6.png)

We can see all the other movies Keanu Reeves also acted in. 

Navigating in the graph like that has some potential, but first we need to find a starting node in our graph. For that an RDF / Triple store offers the SPARQL query language.

## Querying the graph using SPARQL

Click on SPARQL in the navigation menu to the left and we will get to the SPARQL view which integrates the [YASGUI query editor](http://about.yasgui.org/).

![](./images/graphdb-sparql-1.png)

The most basic SPARQL select statement is pre-filled in the query window.

```sparql
select * where {
    ?s ?p ?o .
} limit 100
```

Click **Run** to run the query. It's selecting all the triples in the graph but limiting the result to 100 results. 

![](./images/graphdb-sparql-2.png)

If we want to only show triples with a certain subject, we can adapt the query like that

```sparql
select * where {
    <http://academy.ontotext.com/imdb/title/PiratesoftheCaribbeanAtWorldsEnd> ?p ?o .
}
```

The query selects RDF statements whose subject is the movie Pirates of the Caribbean At World's End (identified by the IRI `http://academy.ontotext.com/imdb/title/TheBourneUltimatum`). 

![](./images/graphdb-sparql-3.png)

We can shorten the IRIs with setting a prefix like shown here

```sparql
PREFIX imdb: <http://academy.ontotext.com/imdb/>

select * where {
    imdb:title\/PiratesoftheCaribbeanAtWorldsEnd ?p ?o .
}
```

This is more useful if the same prefix is used multiple times.

Note that we need to escape the / in the shortened IRI.

The variables ?p and ?o correspond to the predicate and object of the RDF statements. We can see that the director (via the predicate `schema:director`) is identified by the IRI `imdb:person/GoreVerbinski` (scroll down if necessary).


The next query selects all color movies by class (`a` is a short-hand notation for `rdf:type`) and then performs two joins to fetch the movie's name (via the `schema:name` predicate), and the movie's number of comments (via the `schema:commentCount` predicate). Finally, the result must be ordered by the number of comments in descending order.

```sparql
PREFIX imdb: <http://academy.ontotext.com/imdb/>
PREFIX schema: <http://schema.org/>

SELECT * { 
    ?movie a imdb:ColorMovie ;
           schema:name ?movieName ;
           schema:commentCount ?commentCount .
} ORDER BY DESC(?commentCount)
```

The table shows the results from executing the query.

![](./images/graphdb-sparql-4.png)

The variables `?movie`, `?movieName` and `?commentCount` contain each movie's IRI, name and number of comments respectively. We can see that the movie with the most comments, The Dark Knight Rises, comes on top.


The next query selects RDF statements that have the same subject (`?movie`) and the same object (`?person`). 
For any given movie and person, there must be RDF statements that link the movie and the person with both the `schema:director` and the `imdb:leadActor` predicate.

```sparql
PREFIX schema: <http://schema.org/>
PREFIX imdb: <http://academy.ontotext.com/imdb/>

SELECT * { 
	?movie schema:director ?person ;
           imdb:leadActor ?person .
} ORDER BY ?person
```

The table shows the results from executing the query.

![](./images/graphdb-sparql-5.png)

Just like the previous query, in the next query we select movies and people that are both the leading actor and the director. In this query, we also use `GROUP BY ?person` to group the results by person and `COUNT(?movie)` to count how many movies per person satisfy the criteria. The count is returned in the `?numMovies` variable.

```sparql
PREFIX schema: <http://schema.org/>
PREFIX imdb: <http://academy.ontotext.com/imdb/>

SELECT ?person (COUNT(?movie) as ?numMovies) { 
	?movie schema:director ?person ;
           imdb:leadActor ?person .
} GROUP BY ?person ORDER BY DESC(?numMovies)
```

The table shows the results from executing the query.

![](./images/graphdb-sparql-6.png)

Since we also used `ORDER BY DESC(?numMovies)` to order the results by movie count in descending order, we can easily see that both Clint Eastwood and Woody Allen made 10 movies where they were the leading actor and the director.
