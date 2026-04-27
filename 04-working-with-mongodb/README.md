# Getting started with MongoDB

In this workshop we will learn how to use the MongoDB NoSQL database.

We assume that the platform described [here](../01-environment) is running and accessible. 

## Table of Contents

- [What you will learn](#what-you-will-learn)
- [Prerequisites](#prerequisites)
- [Connecting to the MongoDB environment](#connecting-to-the-mongodb-environment)
- [Working with MongoDB](#working-with-mongodb)
- [Creating Movie documents in the `movies` collection](#creating-movie-documents-in-the-movies-collection)
- [Creating Actor documents in the `persons` collection](#creating-actor-documents-in-the-persons-collection)
- [Querying Documents using a Query Selector](#querying-documents-using-a-query-selector)
- [Updating Documents](#updating-documents)
- [Performance Optimizations using Indexes](#performance-optimizations-using-indexes)
- [Text Search](#text-search)
- [Aggregating Data](#aggregating-data)
- [Removing Documents](#removing-documents)
- [Using the Python API with MongoDB](#using-the-python-api-with-mongodb)

## What you will learn

- How to connect to MongoDB using the `mongosh` command-line utility and browser-based GUIs
- How to create and manage documents and collections
- How to query documents using query selectors and projection
- How to update and delete documents
- How to optimize query performance using indexes
- How to perform full-text search
- How to aggregate data using the aggregation pipeline
- How to interact with MongoDB using the Python API

## Prerequisites

- The **Data Platform** described [here](../01-environment) is running and accessible

## Connecting to the MongoDB environment

### Using the MongoDB Command Line utility

You can find the `mongo` command line utility inside the MongoDB docker container running as part of the platform. Connect via SSH onto the Docker Host and run the following `docker exec` command

```bash
docker exec -ti mongo-1 mongosh -u "root" -p "abc123!"
```

This will connect you into the `mongo` container and run the `mongo` shell inside it. 

You should see an output similar to this one below. 

```bash
bigdata@bigdata:~$ docker exec -ti mongo-1 mongosh -u "root" -p "abc123!"
CCurrent Mongosh Log ID:	69efb17239762e8dd144ba88
Connecting to:		mongodb://<credentials>@127.0.0.1:27017/?directConnection=true&serverSelectionTimeoutMS=2000&appName=mongosh+2.8.2
Using MongoDB:		8.2.7
Using Mongosh:		2.8.2

For mongosh info see: https://www.mongodb.com/docs/mongodb-shell/
To help improve our products, anonymous usage data is collected and sent to MongoDB periodically (https://www.mongodb.com/legal/privacy-policy).
You can opt-out by running the disableTelemetry() command.

------
   The server generated these startup warnings when booting
   2026-04-26T13:21:22.520+00:00: Using the XFS filesystem is strongly recommended with the WiredTiger storage engine. See http://dochub.mongodb.org/core/prodnotes-filesystem
   2026-04-26T13:21:26.993+00:00: Soft rlimits for open file descriptors too low
   2026-04-26T13:21:26.993+00:00: For customers running the current memory allocator, we suggest changing the contents of the following sysfsFile
   2026-04-26T13:21:26.993+00:00: For customers running the current memory allocator, we suggest changing the contents of the following sysfsFile
   2026-04-26T13:21:26.993+00:00: We suggest setting the contents of sysfsFile to 0.
------

test> 
```

> **What you should see:** The `test>` prompt confirming a successful connection, along with the MongoDB server version (e.g. `Using MongoDB: 8.2.7`) and the Mongosh shell version (e.g. `Using Mongosh: 2.8.2`).

You are now at the MongoDB command prompt, ready to execute any MongoDB statements. We can also see the version of the MongoDB server as well as of the MongoDB shell.

The shell runs JavaScript. There are some global commands you can execute, like help or exit. Commands that you execute against the current database are executed against the db object, such as `db.help()` or `db.stats()`. 

Commands that you execute against a specific collection, which is what we’ll be doing a lot of, are executed against the `db.COLLECTION_NAME` object, such as `db.movies.help()` or `db.movies.countDocuments()`.

Go ahead and enter `db.help()`, you’ll get a list of commands that you can execute against the db object.

**A small side note:** Because this is a JavaScript shell, if you execute a method and omit the parentheses (), you’ll see the method body rather than executing the method. I only mention it so that the first time you do it and get a response that starts with function (...){  you won’t be surprised. For example, if you enter db.help (without the parentheses), you’ll see the internal implementation of the help method.

### Using browser-based GUI

Instead of working over the command line and therefore having to connect to the Docker Host via SSH, we can also use a browser based GUI to access MongoDB. Two browser-based utilities are available as part of the platform. 

#### Mongo Express 

The first one is [Mongo Express](https://github.com/mongo-express/mongo-express), a Web-based MongoDB admin interface written with Node.js, Express and Bootstrap3.

In a browser window, navigate to <http://dataplatform:28123/> and you should directly arrive on the home screen as shown below. 

![Alt Image Text](./images/mongo-express-home.png "Mongo Express")

> **What you should see:** The Mongo Express home screen listing all databases in the MongoDB instance (e.g. `admin`, `config`, `local`), with a navigation bar for browsing collections and documents.

#### DbGate

The next one is the [DbGate](https://dbgate.io/) we already know from the other workshops.

In a browser window navigate to <http://dataplatform:28120/> and login in as user `dbgate` and password `abc123!`.

Click on the **Add new connection** (`+`) menu under **CONNECTIONS**. 

Select `MongoDB` for the **Connection type** and enter the following values:

 * **Server**: `mongo-1` 
 * **Port**: `27017`
 * **User**: `root`
 * **Password**: `abc123!`
 * **Display name**: `MongoDB`

![Alt Image Text](./images/dbgate.png "DBGate")

Click **Test** to check that connection settings are valid and then click **Connect**. 

The **Cassandra** connection will show up below **CONNECTIONS**. 

> **What you should see:** The DbGate web UI with the MongoDB connection listed under CONNECTIONS, and the keyspaces visible when the connection is expanded.

#### Admin Mongo (not installed)

The second one is [Admin Mongo](https://github.com/adicom-systems/adminMongo), an open source admin user interface for your MongoDB.

This tool is not installed by default, but it is supported by Platys and you can enable it in the config.yml.

If installed, then in a browser window navigate to <http://dataplatform:28124/> and login with user `admin` and password `pass` and you should see the home screen as shown below. 

![Alt Image Text](./images/admin-mongo-home.png "Admin Mongo")

To connect to the MongoDB instance, add a new connection to Admin Mongo. Enter `Data Platform` into the **Connection name** field and `mongodb://mongo-1:27017` into the **Connection string** field and click **Add connection**. A message should appear saying that the connection has been added successfully.  

![Alt Image Text](./images/admin-mongo-connection.png "Admin Mongo Connection")

A Click on the **Connect** button brings you to the Admin Mongo details page for the connection. 

### Using Desktop Applications

There are also various desktop applications for MongoDB management and administration, which can be downloaded and installed on a Desktop. From there you can connect either to a local or remote Mongo instance.

#### MongoDB Compass

[MongoDB Compass](https://www.mongodb.com/products/tools/compass) is a free interactive tool for querying, optimizing, and analyzing your MongoDB data. Get key insights, drag and drop to build pipelines, and more. You can download it from [here](https://www.mongodb.com/try/download/compass).

![Alt Image Text](./images/mongodb-compass.png "MongoDB Compass")

Click on the **Add new connection** button to create a new connection. 

Enter `mongodb://dataplatform:27017` into the **URI** field.

Scroll down and expand **Advanced Connection Options**. Navigate to the **Authentication** tab and select **Username/Password** for the **Authenticaiton Method**. 

Enter `root` into **Username**, `abc123!` into the **Password** and `admin` into the **Authentication Database** field.

![Alt Image Text](./images/mongodb-compass-add-connection.png "MongoDB Compass")
 
Click **Save & Connect** to connect to the MongoDB instance.

#### Studio 3T (formerly known as Robo 3T or Robomongo)

Anoher one we are showing here is [Studio 3T](https://robomongo.org/), a desktop application embedding the MongoDB shell. It is available for Windows, Mac and Linux.

![Alt Image Text](./images/studio3T.png "Studio 3T")

Click on the **Connect** icon in the left upper corner and click on **New Connection** to create a new connection. 

Enter `dataplatform` or the IP address of your Docker Host into the **Server** field. Leave the port on 27017 and click on **Save**. With the newly created connection selected, click **Connect**. On the right side you should now see a list of available databases. 

## Working with MongoDB

We begin our journey by getting to know the basic mechanics of working with MongoDB. Obviously this is core to understanding MongoDB, but it should also help us answer higher-level questions about where MongoDB fits.

To get started, there are six simple concepts we need to understand.

 1.	MongoDB has the same concept of a database with which you are likely already familiar (or a schema for you Oracle folks). Within a MongoDB instance you can have zero or more databases, each acting as high-level containers for everything else. 
 2.	A database can have zero or more collections. A collection shares enough in common with a traditional table that you can safely think of the two as the same thing. 
 3.	Collections are made up of zero or more documents. Again, a document can safely be thought of as a row. 
 4.	A document is made up of one or more fields, which you can probably guess are a lot like columns. 
 5.	Indexes in MongoDB function mostly like their RDBMS counterparts. 
 6.	Cursors are different from the other five concepts but they are important enough, and often overlooked, that I think they are worthy of their own discussion. The important thing to understand about cursors is that when you ask MongoDB for data, it returns a pointer to the result set called a cursor, which we can do things to, such as counting or skipping ahead, before actually pulling down data. 

To recap, MongoDB is made up of databases which contain collections. A collection is made up of documents. Each document is made up of fields. Collections can be indexed, which improves lookup and sorting performance. Finally, when we get data from MongoDB we do so through a cursor whose actual execution is delayed until necessary.

Why use new terminology (collection vs. table, document vs. row and field vs. column)? Is it just to make things more complicated? The truth is that while these concepts are similar to their relational database counterparts, they are not identical. The core difference comes from the fact that relational databases define columns at the table level whereas a document-oriented database defines its fields at the document level. That is to say that each document within a collection can have its own unique set of fields. As such, a collection is a dumbed down container in comparison to a table, while a document has a lot more information than a row.

Although this is important to understand, don’t worry if things aren’t yet clear. It won’t take more than a couple of inserts to see what this truly means. Ultimately, the point is that a collection isn’t strict about what goes in it (it’s schema-less). Fields are tracked with each individual document. The benefits and drawbacks of this will be explored in a future chapter.

So let’s get started in the MongoDB Shell. 

First we’ll use the global use helper to switch databases, so go ahead and enter 

```
use filmdb
```

> **What you should see:** `switched to db filmdb`

> **What just happened?** MongoDB creates the database lazily — `filmdb` does not actually exist on disk yet and will not appear in `show dbs` until at least one document is inserted into one of its collections.

It doesn’t matter that the database doesn’t really exist yet. The first collection that we create will also create the `filmdb` database. Now that you are inside a database, you can start issuing database commands, like `db.getCollectionNames()`. 

```
db.getCollectionNames()
```

You get back an empty array, as there are not yet any collections in the `filmdb` database.

Since collections are schema-less, we don’t explicitly need to create them and we can directly start adding documents to a collection. 

## Creating Movie documents in the `movies` collection

We can simply insert a document into a new collection. To do so, use the `insert` command, supplying it with the document to insert:

Add a document for the movie "Pulp Fiction" to the `movies` collection. The command below can be used from the MongoDB shell. 

```
db.movies.insertOne (
{ 
    "id": "0110912", 
    "title": "Pulp Fiction",
    "year": 1994,
    "runtime": 154,
    "languages": ["en", "es", "fr"],
    "rating": 8.9,
    "votes": 2084331,
    "genres": ["Crime", "Drama"],
    "plotOutline": "Jules Winnfield (Samuel L. Jackson) and Vincent Vega (John Travolta) are two hit men who are out to retrieve a suitcase stolen from their employer, mob boss Marsellus Wallace (Ving Rhames). Wallace has also asked Vincent to take his wife Mia (Uma Thurman) out a few days later when Wallace himself will be out of town. Butch Coolidge (Bruce Willis) is an aging boxer who is paid by Wallace to lose his fight. The lives of these seemingly unrelated people are woven together comprising of a series of funny, bizarre and uncalled-for incidents.",
    "coverUrl": "https://m.media-amazon.com/images/M/MV5BNGNhMDIzZTUtNTBlZi00MTRlLWFjM2ItYzViMjE3YzI5MjljXkEyXkFqcGdeQXVyNzkwMjQ5NzM@._V1_SY150_CR1,0,101,150_.jpg",
    "actors": [
        { "actorID": "0000619", "name": "Tim Roth"},
        { "actorID": "0001625", "name": "Amanda Plummer"},    
        { "actorID": "0522503", "name": "Laura Lovelace"},         
        { "actorID": "0000237", "name": "John Travolta"},   
        { "actorID": "0000168", "name": "Samuel L. Jackson"},   
        { "actorID": "0482851", "name": "Phil LaMarr"},   
        { "actorID": "0001844", "name": "Frank Whaley"},  
        { "actorID": "0824882", "name": "Burr Steers"},  
        { "actorID": "0000246", "name": "Bruce Willis"}, 
        { "actorID": "0000609", "name": "Ving Rahmes"},         
        { "actorID": "0000235", "name": "Uma Thurman"},
        { "actorID": "0000233", "name": "Quentin Tarantino"}
    ],
    "directors": [
        { "directorID": "0000233", "name": "Quentin Tarantino"}
    ],
    "producers": [
        { "producerID": "0004744", "name": "Lawrence Bender"},
        { "producerID": "0000362", "name": "Danny DeVito"},
        { "producerID": "0321621", "name": "Richard N. Gladstein"},        
        { "producerID": "0787834", "name": "Michael Shamberg"},        
        { "producerID": "0792049", "name": "Stacey Sher"},  
        { "producerID": "0918424", "name": "Bob Weinstein"},  
        { "producerID": "0005544", "name": "Harvey Weinstein"}  
    ]
})
```

after executing the command, you should get back the following result, telling that 1 document has been inserted. 

```
{
  acknowledged: true,
  insertedId: ObjectId('66cb796c8137d3ecf9c76a8d')
}
```

> **What you should see:** An acknowledgement object with `acknowledged: true` and an `insertedId` containing the newly generated `ObjectId`.

> **What just happened?** MongoDB assigned an auto-generated `_id` of type `ObjectId` to the document because none was provided. Documents are stored internally as BSON (Binary JSON), which supports richer types than plain JSON — such as `ObjectId`, `Date`, and `BinData`.

In the graphical tools, most of the time you only have to provide the JSON document, without having to specify the `db.movies.insertOne()` command. 

The above line is executing insert against the **movies** collection, passing it a single parameter. Internally MongoDB uses a binary serialised JSON format called BSON. Externally, this means that we use JSON a lot, as is the case with our parameters. 

Let's also add the movie "The Matrix"

```
db.movies.insertOne (
{ 
    "id": "0133093", 
    "title": "The Matrix",
    "year": 1999,
    "runtime": 136,
    "languages": ["en"],
    "rating": 8.7,
    "votes": 1496538,
    "genres": ["Action", "Sci-Fi"],
    "plotOutline": "Thomas A. Anderson is a man living two lives. By day he is an average computer programmer and by night a hacker known as Neo. Neo has always questioned his reality, but the truth is far beyond his imagination. Neo finds himself targeted by the police when he is contacted by Morpheus, a legendary computer hacker branded a terrorist by the government. Morpheus awakens Neo to the real world, a ravaged wasteland where most of humanity have been captured by a race of machines that live off of the humans' body heat and electrochemical energy and who imprison their minds within an artificial reality known as the Matrix. As a rebel against the machines, Neo must return to the Matrix and confront the agents: super-powerful computer programs devoted to snuffing out Neo and the entire human rebellion.",
    "coverUrl": "https://m.media-amazon.com/images/M/MV5BNzQzOTk3OTAtNDQ0Zi00ZTVkLWI0MTEtMDllZjNkYzNjNTc4L2ltYWdlXkEyXkFqcGdeQXVyNjU0OTQ0OTY@._V1_SX101_CR0,0,101,150_.jpg",
    "actors": [
        { "actorID": "0000206", "name": "Keanu Reeves"},
        { "actorID": "0000401", "name": "Laurence Fishburne"},    
        { "actorID": "0005251", "name": "Carrie-Anne Moss"},         
        { "actorID": "0915989", "name": "Hugo Weaving"},   
        { "actorID": "0287825", "name": "Gloria Foster"},   
        { "actorID": "0001592", "name": "Joe Pantoliano"},   
        { "actorID": "0159059", "name": "Marcus Chong"},  
        { "actorID": "0032810", "name": "Julian Arahanga"},  
        { "actorID": "0000246", "name": "Bruce Willis"}, 
        { "actorID": "0000609", "name": "Ving Rahmes"},         
        { "actorID": "0000235", "name": "Uma Thurman"},
        { "actorID": "0000233", "name": "Quentin Tarantino"}
    ],
    "directors": [
        { "directorID": "0905154", "name": "Lana Wachowski"},
        { "directorID": "0905152", "name": "Lilly Wachowski"}
    ],
    "producers": [
        { "producerID": "0075732", "name": "Bruce Berman"},
        { "producerID": "0185621", "name": "Dan Cracchiolo"},
        { "producerID": "0400492", "name": "Carol Hughes"}  
    ]        
})
```

> **What you should see:** An acknowledgement object with `acknowledged: true` and an `insertedId` containing the newly generated `ObjectId` for "The Matrix".

> **What just happened?** As before, MongoDB assigned an auto-generated `ObjectId` and stored the document as BSON in the `movies` collection.

If we execute `db.getCollectionNames()` now, we should see the collection we have just added documents to

```
> db.getCollectionNames()
[ 'movies' ]
```

You can now use the `find` command against the **movies** collection to return a list of documents:

```
db.movies.find()
```

> **What you should see:** All documents in the `movies` collection printed as JSON objects, including the `_id`, `title`, `year`, `genres`, and all other stored fields.

In fact what we are executing is actually this statement.

```
db.movies.find({})
```

you can see that an empty document is passed as a parameter. This document will later hold the actually query in order to perform a restriction on the documents returned. An empty document just means return all and is the default. 

To display the results in a nicely formatted way, you can use `pretty()` method

```
db.movies.find().pretty()
```

Notice that, in addition to the data you specified, there’s an `_id` field. Every document must have a unique id field. 
You can either generate one yourself or let MongoDB generate a value for you which has the type `ObjectId`. Most of the time you’ll probably want to let MongoDB generate it for you. By default, the `_id` field is indexed - which can be checked using the `db.persons.getIndexes()` command

```
> db.movies.getIndexes()
[ { "v" : 2, "key" : { "_id" : 1 }, "name" : "_id_" } ]
```

What you’re seeing is the name of the index, the database and collection it was created against and the fields included in the index. 

## Creating Actor documents in the `persons` collection

Now let's also add some actors to another, new collection named `persons`. We name it this way, because the same person can have different roles in one or many movies.

Let's first add the actor "Bruce Willis"

```
db.persons.insertOne (
{ 
    "id": 0000246, 
    "name": "Bruce Willis",
    "headshot": "https://m.media-amazon.com/images/M/MV5BMjA0MjMzMTE5OF5BMl5BanBnXkFtZTcwMzQ2ODE3Mw@@._V1_UY98_CR8,0,67,98_AL_.jpg",
    "birthDate": "1955-03-19",
    "tradeMarks": ['Frequently plays a man who suffered a tragedy, had lost something or had a  crisis of confidence or conscience.',
					  'Frequently plays likeable wisecracking heroes with a moral centre',
					  'Headlines action-adventures, often playing a policeman, hitman or someone in the military',
					  'Often plays men who get caught up in situations far beyond their control',
					  'Sardonic one-liners',
					  'Shaven head',
					  'Distinctive, gravelly voice',
					  'Smirky grin.',
					  'Known for playing cynical anti-heroes with unhappy personal lives'],
    "actedInMovies": [
        { "movieId": "0110912", "title": "Pulp Fiction"},
        { "movieId": "1606378", "title": "A Good Day to Die Hard"},
        { "movieId": "0217869", "title": "Unbreakable"},
        { "movieId": "0377917", "title": "The Fifth Element"},
        { "movieId": "0112864", "title": "Die Hard: With a Vengeance"}
    ]        
})
```

> **What you should see:** An acknowledgement object with `acknowledged: true` and an `insertedId` containing the newly generated `ObjectId` for "Bruce Willis".

> **What just happened?** MongoDB assigned an auto-generated `ObjectId` and stored the document as BSON in the `persons` collection.

then add the actor "Keanu Reeves"

```
db.persons.insertOne (
{ 
    "id": 0000206, 
    "name": "Keanu Reeves",
    "headshot": "https://m.media-amazon.com/images/M/MV5BMjA0MjMzMTE5OF5BMl5BanBnXkFtZTcwMzQ2ODE3Mw@@._V1_UY98_CR8,0,67,98_AL_.jpg",
    "birthDate": "1955-03-19",
    "tradeMarks": ['Intense contemplative gaze',
		  'Deep husky voice',
		  'Known for playing stoic reserved characters'],
    "actedInMovies": [
        { "movieId": "0133093", "title": "The Matrix"},
        { "movieId": "0234215", "title": "The Matrix Reloaded"},
        { "movieId": "0111257", "title": "Speed"}
    ]        
})
```

> **What you should see:** An acknowledgement object with `acknowledged: true` and an `insertedId` containing the newly generated `ObjectId` for "Keanu Reeves".

> **What just happened?** MongoDB assigned an auto-generated `ObjectId` and stored the document as BSON in the `persons` collection.

followed by the actress "Sandra Bullock"

```
db.persons.insertOne (
{ 
    "id": 0000113, 
    "name": "Sandra Bullock",
    "headshot": "https://m.media-amazon.com/images/M/MV5BMTI5NDY5NjU3NF5BMl5BanBnXkFtZTcwMzQ0MTMyMw@@._V1_UX67_CR0,0,67,98_AL_.jpg",
    "birthDate": "1964-07-26",
    "actedInMovies": [
        { "movieId": "2737304", "title": "Bird Box"},
        { "movieId": "0120179", "title": "Speed 2: Cruise Control"},
        { "movieId": "0111257", "title": "Speed"},
        { "movieId": "0212346", "title": "Miss Congeniality"}
    ]        
})
```

> **What you should see:** An acknowledgement object with `acknowledged: true` and an `insertedId` containing the newly generated `ObjectId` for "Sandra Bullock".

> **What just happened?** MongoDB assigned an auto-generated `ObjectId` and stored the document as BSON in the `persons` collection.

and finally we also add "Quentin Tarantino"

```
db.persons.insertOne (
{ 
    "id": 0000233, 
    "name": "Quentin Tarantino",
    "headshot": "https://m.media-amazon.com/images/M/MV5BMTgyMjI3ODA3Nl5BMl5BanBnXkFtZTcwNzY2MDYxOQ@@._V1_UX67_CR0,0,67,98_AL_.jpg",
    "birthDate": "1963-03-27",
    "tradeMarks": ['Lead characters usually drive General Motors vehicles, particularly Chevrolet and Cadillac, such as Jules 1974 Nova and Vincents 1960s Malibu.',
          'Briefcases and suitcases play an important role in Pulp Fiction (1994), Reservoir Dogs (1992), Jackie Brown (1997), True Romance (1993) and Kill Bill: Vol. 2 (2004).',
          'Makes references to cult movies and television',
          'Frequently works with Harvey Keitel, Tim Roth, Michael Madsen, Uma Thurman, Michael Bowen, Samuel L. Jackson, Michael Parks and Christoph Waltz.',
          'His films usually have a shot from inside an automobile trunk',
          'He always has a Dutch element in his films: The opening tune, Little Green Bag, in Reservoir Dogs (1992) was performed by George Baker Selection and written by Jan Gerbrand Visser and Benjamino Bouwens who are all Dutch. The character Freddy Newandyke, played by Tim Roth is a direct translation to a typical Dutch last name, Nieuwendijk. The code name of Tim Roth is Mr. Orange, the royal color of Holland and the last name of the royal family. The Amsterdam conversation in Pulp Fiction (1994), Vincent Vega smokes from a Dutch tobacco shag (Drum), the mentioning of Rutger Hauer in Jackie Brown (1997), the brides name is Beatrix, the name of the Royal Dutch Queen.',
		  '[The Mexican Standoff] All his movies (including True Romance (1993), which he only wrote and did not direct) feature a scene in which three or more characters are pointing guns at each other at the same time.',
         'Often uses an unconventional storytelling device in his films, such as retrospect (Reservoir Dogs (1992)), non-linear (Pulp Fiction (1994)), or "chapter" format (Kill Bill: Vol. 1 (2003)).',
         'His films will often include one long, unbroken take where a character is  followed around somewhere.'],
    "actedInMovies": [
        { "movieId": "0378194", "title": "Kill Bill: Vol. 2"},
        { "movieId": "0110912", "title": "Speed 2: Cruise Control"},
        { "movieId": "0116367", "title": "From Dusk Till Dawn"},
        { "movieId": "0119396", "title": "Jackie Brown"}
    ]        
})
```

> **What you should see:** An acknowledgement object with `acknowledged: true` and an `insertedId` containing the newly generated `ObjectId` for "Quentin Tarantino".

> **What just happened?** MongoDB assigned an auto-generated `ObjectId` and stored the document as BSON in the `persons` collection.

So now let's also check that we have all 4 persons added to the collection

```
db.persons.find()
```

> **What you should see:** All 4 documents in the `persons` collection printed as JSON objects, each containing `_id`, `name`, `birthDate`, `actedInMovies`, and any other stored fields.

We can also use the `countDocuments()` method to return the number of documents within the collection. 

```
db.persons.find().count()
```

or the `estimatedDocumentCount()` method to get an estimated count (based on metadata)

```
db.persons.estimatedDocumentCount()
```

Which in that case (because we don't specify a query selector) is the same as 

```
db.persons.countDocuments()
```

**Note:** notice that not all documents are exactly the same. The "Sandra Bullock" document does not contain the `tradeMark` array. The collections are schema-less, there is only a JSON parsing being done and therefore the document has to be valid JSON. Let's see what happens if we are using an invalid document.

```
db.persons.insertOne (
{ 
    "id: 0000113, 
    "name": "Invalid Actor"
})
```

Notice that we have not properly closed the `id` key (missing "). We will get the following error upon insert

```
> db.persons.insertOne (
... {
...     "id: 0000113,
Uncaught:
SyntaxError: Unterminated string constant. (3:4)

  1 | db.persons.insertOne (
  2 | {
> 3 |     "id: 0000113,
    |     ^
  4 |

filmdb>     "name": "Invalid Actor"
Uncaught:
SyntaxError: Missing semicolon. (1:10)

> 1 |     "name": "Invalid Actor"
    |           ^
  2 |
```

So while documents in one collection can be completely different from other documents in that collection, they always have to valid JSON documents. 

## Querying Documents using a Query Selector

So far we have used `find` to retrieve all the documents within a collection. That's fine if you have a limited set of documents like here, but of course if you have million of documents, you need a way to query for just some of the documents. 

In addition to the six concepts we’ve explored, there’s one practical aspect of MongoDB you need to have a good grasp of before moving to more advanced topics, the **query selectors**. 

A MongoDB **query selector** is like the where clause of an SQL statement. As such, you use it when finding, counting, updating and removing documents from collections. A selector is a JSON object, the simplest of which is {} which matches all documents. If we wanted to find all movies for the Action genre, we could use `{ genres:'Action' }`.
 
Before delving too deeply into selectors, let’s set up some additional movies to play with. We will use the Top 50 movies from IMDB, without the two movies we have added already before. We won't add the "full-blown" documents, we only add limited information for each movie. We can again see that MongoDB is "schema-less", in the sense that not all documents have to hold all the information. 

We use the `insertMany` method to add multiple JSON documents at once. 

```
db.movies.insertMany([
	{"id": "0111161", "title": "The Shawshank Redemption", "genres": ["Drama"], "year": 1994, "rating": 9.2, "rank": 1},
	{"id": "0068646", "title": "The Godfather", "genres": ["Crime", "Drama"], "year": 1972, "rating": 9.2, "rank": 2},
	{"id": "0071562", "title": "The Godfather: Part II", "genres": ["Crime", "Drama"], "year": 1974, "rating": 9.0, "rank": 3},
	{"id": "0468569", "title": "The Dark Knight", "genres": ["Action", "Crime", "Drama", "Thriller"], "year": 2008, "rating": 9.0, "rank": 4},
	{"id": "0050083", "title": "12 Angry Men", "genres": ["Drama"], "year": 1957, "rating": 8.9, "rank": 5},
	{"id": "0108052", "title": "Schindler's List", "genres": ["Biography", "Drama", "History"], "year": 1993, "rating": 8.9, "rank": 6},
	{"id": "0167260", "title": "The Lord of the Rings: The Return of the King", "genres": ["Adventure", "Drama", "Fantasy"], "year": 2003, "rating": 8.9, "rank": 7},
	{"id": "0060196", "title": "The Good, the Bad and the Ugly", "genres": ["Western"], "year": 1966, "rating": 8.8, "rank": 9},
	{"id": "0137523", "title": "Fight Club", "genres": ["Drama"], "year": 1999, "rating": 8.8, "rank": 10},
	{"id": "4154796", "title": "Avengers: Endgame", "genres": ["Action", "Adventure", "Fantasy", "Sci-Fi"], "year": 2019, "rating": 8.8, "rank": 11},
	{"id": "0120737", "title": "The Lord of the Rings: The Fellowship of the Ring", "genres": ["Adventure", "Drama", "Fantasy"], "year": 2001, "rating": 8.8, "rank": 12},
	{"id": "0109830", "title": "Forrest Gump", "genres": ["Drama", "Romance"], "year": 1994, "rating": 8.7, "rank": 13},
	{"id": "0080684", "title": "Star Wars: Episode V - The Empire Strikes Back", "genres": ["Action", "Adventure", "Fantasy", "Sci-Fi"], "year": 1980, "rating": 8.7, "rank": 14},
	{"id": "1375666", "title": "Inception", "genres": ["Action", "Adventure", "Sci-Fi", "Thriller"], "year": 2010, "rating": 8.7, "rank": 15},
	{"id": "0167261", "title": "The Lord of the Rings: The Two Towers", "genres": ["Adventure", "Drama", "Fantasy"], "year": 2002, "rating": 8.7, "rank": 16},
	{"id": "0073486", "title": "One Flew Over the Cuckoo's Nest", "genres": ["Drama"], "year": 1975, "rating": 8.7, "rank": 17},
	{"id": "0099685", "title": "Goodfellas", "genres": ["Biography", "Crime", "Drama"], "year": 1990, "rating": 8.7, "rank": 18},
	{"id": "0047478", "title": "Seven Samurai", "genres": ["Adventure", "Drama"], "year": 1954, "rating": 8.6, "rank": 20},
	{"id": "0114369", "title": "Se7en", "genres": ["Crime", "Drama", "Mystery", "Thriller"], "year": 1995, "rating": 8.6, "rank": 21},
	{"id": "0317248", "title": "City of God", "genres": ["Crime", "Drama"], "year": 2002, "rating": 8.6, "rank": 22},
	{"id": "0076759", "title": "Star Wars: Episode IV - A New Hope", "genres": ["Action", "Adventure", "Fantasy", "Sci-Fi"], "year": 1977, "rating": 8.6, "rank": 23},
	{"id": "0102926", "title": "The Silence of the Lambs", "genres": ["Crime", "Drama", "Thriller"], "year": 1991, "rating": 8.6, "rank": 24},
	{"id": "0038650", "title": "It's a Wonderful Life", "genres": ["Drama", "Family", "Fantasy"], "year": 1946, "rating": 8.6, "rank": 25},
	{"id": "0118799", "title": "Life Is Beautiful", "genres": ["Comedy", "Drama", "Romance", "War"], "year": 1997, "rating": 8.6, "rank": 26},
	{"id": "0245429", "title": "Spirited Away", "genres": ["Animation", "Adventure", "Family", "Fantasy", "Mystery"], "year": 2001, "rating": 8.5, "rank": 27},
	{"id": "0120815", "title": "Saving Private Ryan", "genres": ["Drama", "War"], "year": 1998, "rating": 8.5, "rank": 28},
	{"id": "0114814", "title": "The Usual Suspects", "genres": ["Crime", "Mystery", "Thriller"], "year": 1995, "rating": 8.5, "rank": 29},
	{"id": "0110413", "title": "L\u00e9on: The Professional", "genres": ["Action", "Crime", "Drama", "Thriller"], "year": 1994, "rating": 8.5, "rank": 30},
	{"id": "0120689", "title": "The Green Mile", "genres": ["Crime", "Drama", "Fantasy", "Mystery"], "year": 1999, "rating": 8.5, "rank": 31},
	{"id": "0816692", "title": "Interstellar", "genres": ["Adventure", "Drama", "Sci-Fi"], "year": 2014, "rating": 8.5, "rank": 32},
	{"id": "0054215", "title": "Psycho", "genres": ["Horror", "Mystery", "Thriller"], "year": 1960, "rating": 8.5, "rank": 33},
	{"id": "0120586", "title": "American History X", "genres": ["Drama"], "year": 1998, "rating": 8.5, "rank": 34},
	{"id": "0021749", "title": "City Lights", "genres": ["Comedy", "Drama", "Romance"], "year": 1931, "rating": 8.5, "rank": 35},
	{"id": "0034583", "title": "Casablanca", "genres": ["Drama", "Romance", "War"], "year": 1942, "rating": 8.5, "rank": 36},
	{"id": "0064116", "title": "Once Upon a Time in the West", "genres": ["Western"], "year": 1968, "rating": 8.5, "rank": 37},
	{"id": "0253474", "title": "The Pianist", "genres": ["Biography", "Drama", "Music", "War"], "year": 2002, "rating": 8.5, "rank": 38},
	{"id": "0027977", "title": "Modern Times", "genres": ["Comedy", "Drama", "Family", "Romance"], "year": 1936, "rating": 8.5, "rank": 39},
	{"id": "1675434", "title": "The Intouchables", "genres": ["Biography", "Comedy", "Drama"], "year": 2011, "rating": 8.5, "rank": 40},
	{"id": "0407887", "title": "The Departed", "genres": ["Crime", "Drama", "Thriller"], "year": 2006, "rating": 8.5, "rank": 41},
	{"id": "0088763", "title": "Back to the Future", "genres": ["Adventure", "Comedy", "Sci-Fi"], "year": 1985, "rating": 8.5, "rank": 42},
	{"id": "0103064", "title": "Terminator 2: Judgment Day", "genres": ["Action", "Sci-Fi"], "year": 1991, "rating": 8.5, "rank": 43},
	{"id": "2582802", "title": "Whiplash", "genres": ["Drama", "Music"], "year": 2014, "rating": 8.5, "rank": 44},
	{"id": "0110357", "title": "The Lion King", "genres": ["Animation", "Adventure", "Drama", "Family", "Musical"], "year": 1994, "rating": 8.5, "rank": 45},
	{"id": "0047396", "title": "Rear Window", "genres": ["Mystery", "Thriller"], "year": 1954, "rating": 8.5, "rank": 46},
	{"id": "0082971", "title": "Raiders of the Lost Ark", "genres": ["Action", "Adventure"], "year": 1981, "rating": 8.5, "rank": 47},
	{"id": "0172495", "title": "Gladiator", "genres": ["Action", "Adventure", "Drama"], "year": 2000, "rating": 8.5, "rank": 48},
	{"id": "0482571", "title": "The Prestige", "genres": ["Drama", "Mystery", "Sci-Fi", "Thriller"], "year": 2006, "rating": 8.5, "rank": 49},
	{"id": "0078788", "title": "Apocalypse Now", "genres": ["Drama", "War"], "year": 1979, "rating": 8.4, "rank": 50}
])
```

After executing the multi insert, we can check that we have in fact 50 movies in our `movies` collection. 

```
> db.movies.find().count()
50 
```

> **What you should see:** The number `50`, confirming that all documents (the 2 full-detail movies inserted earlier plus the 48 just added) are now in the `movies` collection.

> **What just happened?** `insertMany` inserted all 48 documents in a single operation. MongoDB assigned an auto-generated `ObjectId` `_id` to each document and stored them as BSON. Because the new documents contain fewer fields than the earlier ones, this also demonstrates MongoDB's schema-less nature.

Now that we have data, we can master selectors. `{field: value}` is used to find any documents where field is equal to value. `{field1: value1, field2: value2}` is how we can combine them with **and** semantic. 
The special `$lt`, `$lte`, `$gt`, `$gte` and `$ne` are used for less than, less than or equal, greater than, greater than or equal and not equal operations. 

To get all Family movies, we can perform 

```
db.movies.find({"genres": "Family"})
```

> **What you should see:** The matching documents printed as JSON — all movies whose `genres` array contains `"Family"`.

If we want get all movies that have been published in 2010 and after, we could do:

```
db.movies.find({"genres":"Action", "year": { $gte :  2010 } })
```

> **What you should see:** The matching documents printed as JSON — all Action movies released in 2010 or later.

To find all movies which are **not** of genre **Drama**

```
db.movies.find({"genres": { $ne: "Drama"} })
```

> **What you should see:** The matching documents printed as JSON — all movies whose `genres` array does not contain `"Drama"`.

The `$exists` operator can be used for matching the presence or absence of a field

```
db.movies.find({ "plotOutline": { $exists: true} })
```

> **What you should see:** The matching documents printed as JSON — only the movies that have the `plotOutline` field (the two full-detail movies inserted first).

we can see that only two movies have the `plotOutline` property set. 

The `$in` operator can be used for matching one of several values that we pass as an array

```
db.movies.find({ "genres": { $in: ['Family', 'Mistery']} })
```

> **What you should see:** The matching documents printed as JSON — all movies in either the `Family` or `Mistery` genre.

which returns all movies in either the `Family` or `Mistery` genre.

If we want to **OR** rather than **AND** several conditions on different fields, we use the `$or` operator and assign to it an array of selectors we want or’d

To find all movies of genre *Music* **OR** which have been published *2012* or later

```
db.movies.find({ $or: [ { "genres":"Music" },  { "year": { $gte :  2012 } } ] })
```

> **What you should see:** The matching documents printed as JSON — all movies in the `Music` genre or released in 2012 or later.

To find all movies of genre *Action* **AND** which have been published *2010* or later **OR** have a rating later than *8.8*

```
db.movies.find({ "genres":"Action", $or: [ { "year": { $gte :  2010 } },  { "rating": { $gt :  8.8 } } ] })
```

> **What you should see:** The matching documents printed as JSON — all Action movies that were either released in 2010 or later, or have a rating greater than 8.8.

There’s something pretty neat going on in our last two examples. You might have already noticed, but the `genres` field is an array. MongoDB supports arrays as first class objects. This is an incredibly handy feature. Once you start using it, you wonder how you ever lived without it. What’s more interesting is how easy selecting based on an array value is: `{ genres: 'Action' }` will return any document where genres has a value of `Action`.

There are more available operators than what we’ve seen so far. These are all described in the [Query Selectors](https://docs.mongodb.com/manual/reference/operator/query/index.html) section of the MongoDB manual. What we’ve covered so far though is the basics you’ll need to get started. It’s also what you’ll end up using most of the time.

We’ve seen how these selectors can be used with the `find` command. But they can also be used with the `remove` command, the `count` command and the `update` command which we’ll spend more time with later on.

The `ObjectId` which MongoDB generated for our `_id` field can be selected like so:

```
db.movies.find( {_id: ObjectId("<the-object-id>")})
```

Make sure to replace the `<the-object-id>` by an actual value of one of the movies you have inserted before. 

## Updating Documents

In its simplest form, `updateOne()` takes two parameters: the selector (where) to use and what updates to apply to fields. Let's say that we want to change the rating of the movie `Fight Club` to `9`

```
db.movies.updateOne ( {title: 'Fight Club'} , { $set: {rating: 9} } )
```

> **What you should see:** A result object with `matchedCount: 1` and `modifiedCount: 1`, confirming that the document was found and the `rating` field was updated.

> **What just happened?** MongoDB's `$set` operator only changed the `rating` field — all other fields in the document were preserved exactly as they were. This is unlike a SQL UPDATE which replaces the specified columns but leaves the row structure fixed; with `$set` only the named fields are touched.

In addition to `$set`, we can leverage other operators to do some nifty things. All update operators work on fields - so your entire document won’t be wiped out. For example, the `$inc` operator is used to increment a field by a certain positive or negative amount. 

If we want to increase the votes for the movie "The Matrix", which is currently set to `1496538` as we can easily see using a find

```
db.movies.find( {title: 'The Matrix'}, {"votes":1})
```

*Note:* the second parameter in the find specifies that instead of the complete document we only want to return the `votes` property as we can see in the result (the _id is always returned by default and could be removed by explicitly also specifying `{ _id:0}` in the 2nd parameter). 

```
> db.movies.find( {title: 'The Matrix'}, {"votes":1})
{ "_id" : ObjectId("5ccffa52aff86ec587e35faa"), "votes" : 1496538 }
```

> **What you should see:** A single document with only the `_id` and `votes` fields returned — the projection has excluded all other fields.

we can execute the following update

```
db.movies.updateOne( {title: 'The Matrix'} , { $inc: {votes: 1} } )
```

> **What you should see:** A result object with `matchedCount: 1` and `modifiedCount: 1`, confirming that the `votes` field was incremented.

> **What just happened?** The `$inc` operator added `1` to the existing `votes` value atomically, leaving all other fields untouched.

check the new result using the same find as above a 2nd time

```
db.movies.find( {title: 'The Matrix'}, {"votes":1})
```

> **What you should see:** A single document showing only `_id` and `votes`, now with the value incremented by 1 (e.g. `1496539`).

## Performance Optimizations using Indexes

Indexes in MongoDB work a lot like indexes in a relational database: they help improve query and sorting performance. Indexes are created via `createIndex` command. So let's add an index on the title of movies documents. For an ascending index on a field, specify a value of `1`; for descending index, specify a value of `-1`.

```
db.movies.createIndex( {title: 1} );
```

> **What you should see:** A confirmation string such as `title_1`, which is the name MongoDB assigned to the new index.

> **What just happened?** MongoDB built a B-tree index on the `title` field in ascending order. Queries that filter or sort by `title` will now use this index instead of scanning every document in the collection.

if we know execute a query on the tile, the index will be used

```
db.movies.find ( {title: "The Matrix"} );
```

If we would have a lot more data in the movies collection, we might see a visual difference. But with only 50 movies, that's not the case. However we can use the `explain()` method to view the execution plan of the optimiser.

Adding the `explain` method at the end of the find statement will return the following result

```
> db.movies.find ( {title: "The Matrix"} ).explain();
{
  explainVersion: '1',
  queryPlanner: {
    namespace: 'filmdb.movies',
    indexFilterSet: false,
    parsedQuery: { title: { '$eq': 'The Matrix' } },
    queryHash: '2495AF30',
    planCacheKey: 'D2B6550E',
    maxIndexedOrSolutionsReached: false,
    maxIndexedAndSolutionsReached: false,
    maxScansToExplodeReached: false,
    winningPlan: {
      stage: 'FETCH',
      inputStage: {
        stage: 'IXSCAN',
        keyPattern: { title: 1 },
        indexName: 'title_1',
        isMultiKey: false,
        multiKeyPaths: { title: [] },
        isUnique: false,
        isSparse: false,
        isPartial: false,
        indexVersion: 2,
        direction: 'forward',
        indexBounds: { title: [ '["The Matrix", "The Matrix"]' ] }
      }
    },
    rejectedPlans: []
  },
  command: { find: 'movies', filter: { title: 'The Matrix' }, '$db': 'filmdb' },
  serverInfo: {
    host: 'mongo-1',
    port: 27017,
    version: '7.0.12',
    gitVersion: 'b6513ce0781db6818e24619e8a461eae90bc94fc'
  },
  serverParameters: {
    internalQueryFacetBufferSizeBytes: 104857600,
    internalQueryFacetMaxOutputDocSizeBytes: 104857600,
    internalLookupStageIntermediateDocumentMaxSizeBytes: 104857600,
    internalDocumentSourceGroupMaxMemoryBytes: 104857600,
    internalQueryMaxBlockingSortMemoryUsageBytes: 104857600,
    internalQueryProhibitBlockingMergeOnMongoS: 0,
    internalQueryMaxAddToSetBytes: 104857600,
    internalDocumentSourceSetWindowFieldsMaxMemoryBytes: 104857600,
    internalQueryFrameworkControl: 'trySbeRestricted'
  },
  ok: 1
}
```

> **What you should see:** Execution plan details including `winningPlan` with a stage of `IXSCAN` (index scan) referencing `title_1`, confirming the index is being used.

> **What just happened?** `explain()` shows the query execution plan chosen by MongoDB's query optimiser. An `IXSCAN` stage means the index was used, while a `COLLSCAN` would mean a full collection scan. The plan also shows how many documents were examined versus returned, helping you identify inefficient queries.

We can see that the `winningPlan` uses the `title_1` index. 

A unique index can be created by supplying a second parameter and setting `unique` to true. Let's add an index on the `id` field to ensure that it is unique. 

```
db.movies.createIndex( {id: 1}, {unique: true} );
```

> **What you should see:** A confirmation string such as `id_1`, which is the name MongoDB assigned to the new unique index.

> **What just happened?** MongoDB built a B-tree index on the `id` field with a uniqueness constraint. Any future attempt to insert a document with a duplicate `id` value will be rejected with a duplicate key error.

If we now try to add one of the movies a 2nd time we get an error:

```
> db.movies.insertOne( {"id": "0111161", "title": "The Shawshank Redemption", "genres": ["Drama"], "year": 1994, "rating": 9.2, "rank": 1} )
MongoServerError: E11000 duplicate key error collection: filmdb.movies index: id_1 dup key: { id: "0111161" }
```

We can list the index we currently have on the `movies` collection using `db.movies.getIndexes()`:

```
> db.movies.getIndexes()
[
  { v: 2, key: { _id: 1 }, name: '_id_' },
  { v: 2, key: { title: 1 }, name: 'title_1' },
  { v: 2, key: { id: 1 }, name: 'id_1', unique: true }
]
```

We can see a total of 3 indices, the two we have just added and a 3rd one on the `_id` field which has been created automatically by MongoDB.


An index can be dropped using the `dropIndex` command. 

```
db.movies.dropIndex( {title: 1} );
```

We can also create a **Compound** Index covering multiple fields, we can create a **Multikey** Index to index the content of an array field and create **Geospatial**, **Text** and **Hashed** Indexes. 

Consult the [MongoDB's documentation](https://docs.mongodb.com/manual/indexes/) to read more about Indexes.  

## Text Search 

MongoDB supports query operations that perform a text search of string content. To perform text search, MongoDB uses a **text index** and the `$text` operator.

To perform text search queries, you must have a text index on your collection. A collection can only have one text search index, but that index can cover multiple fields.

For example you can run the following in a mongo shell to allow text search over the `title` and `plotOutline` fields:

```
db.movies.createIndex ( { title: "text", plotOutline: "text" } )
```

> **What you should see:** A confirmation string such as `title_text_plotOutline_text`, which is the name MongoDB assigned to the new text index.

> **What just happened?** MongoDB built an inverted text index that tokenises, stems, and indexes all string values in the `title` and `plotOutline` fields. This enables full-text search with relevance scoring using the `$text` operator.

Now let's to a text search for the term "fight"

```
db.movies.find( { $text: { $search: "fight" } } )
```

The `$text` query operator will tokenize the search string using whitespace and most punctuation as delimiters, and perform a logical OR of all such tokens in the search string.
We should get a result with two movies, one Flight Club where the term can be found in the title and another one where the term is used in the `plotOutline`. 

```
db.movies.find( { $text: { $search: "fight" } } )
[
  {
    _id: ObjectId('66cb7a728137d3ecf9c76a9b'),
    rating: 9,
    genres: [ 'Drama' ],
    rank: 10,
    title: 'Fight Club',
    year: 1999
  },
  {
    _id: ObjectId('66cb796c8137d3ecf9c76a8d'),
    id: '0110912',
    title: 'Pulp Fiction',
    year: 1994,
    runtime: 154,
    languages: [ 'en', 'es', 'fr' ],
    rating: 8.9,
    votes: 2084331,
    genres: [ 'Crime', 'Drama' ],
    plotOutline: 'Jules Winnfield (Samuel L. Jackson) and Vincent Vega (John Travolta) are two hit men who are out to retrieve a suitcase stolen from their employer, mob boss Marsellus Wallace (Ving Rhames). Wallace has also asked Vincent to take his wife Mia (Uma Thurman) out a few days later when Wallace himself will be out of town. Butch Coolidge (Bruce Willis) is an aging boxer who is paid by Wallace to lose his fight. The lives of these seemingly unrelated people are woven together comprising of a series of funny, bizarre and uncalled-for incidents.',
    coverUrl: 'https://m.media-amazon.com/images/M/MV5BNGNhMDIzZTUtNTBlZi00MTRlLWFjM2ItYzViMjE3YzI5MjljXkEyXkFqcGdeQXVyNzkwMjQ5NzM@._V1_SY150_CR1,0,101,150_.jpg',
    actors: [
      { actorID: '0000619', name: 'Tim Roth' },
      { actorID: '0001625', name: 'Amanda Plummer' },
      { actorID: '0522503', name: 'Laura Lovelace' },
      { actorID: '0000237', name: 'John Travolta' },
      { actorID: '0000168', name: 'Samuel L. Jackson' },
      { actorID: '0482851', name: 'Phil LaMarr' },
      { actorID: '0001844', name: 'Frank Whaley' },
      { actorID: '0824882', name: 'Burr Steers' },
      { actorID: '0000246', name: 'Bruce Willis' },
      { actorID: '0000609', name: 'Ving Rahmes' },
      { actorID: '0000235', name: 'Uma Thurman' },
      { actorID: '0000233', name: 'Quentin Tarantino' }
    ],
    directors: [ { directorID: '0000233', name: 'Quentin Tarantino' } ],
    producers: [
      { producerID: '0004744', name: 'Lawrence Bender' },
      { producerID: '0000362', name: 'Danny DeVito' },
      { producerID: '0321621', name: 'Richard N. Gladstein' },
      { producerID: '0787834', name: 'Michael Shamberg' },
      { producerID: '0792049', name: 'Stacey Sher' },
      { producerID: '0918424', name: 'Bob Weinstein' },
      { producerID: '0005544', name: 'Harvey Weinstein' }
    ]
  }
]
```

> **What you should see:** Documents whose `title` or `plotOutline` fields contain the word "fight" — in this case "Fight Club" (matched in the title) and "Pulp Fiction" (matched in the plot outline where the word "fight" appears).

If we change the term to `fight terrorist` we can see that the search string will be tokenized into `fight` and `terrorist` and all the movies will be returned matching either of the two terms in the `title` or the `plotOutline` field. 

```
db.movies.find( { $text: { $search: "fight terrorist" } } )
```

> **What you should see:** Documents whose `title` or `plotOutline` fields contain the word "fight" or "terrorist" — "Fight Club", "Pulp Fiction", and "The Matrix" (which uses the word "terrorist" in its plot outline).

Therefore we will also get back a 3rd movie, the movie "The Matrix" which uses the word Terrorist in the plot outline.

## Aggregating Data

Aggregation pipeline gives you a way to transform and combine documents in your collection. You do it by passing the documents through a pipeline that’s somewhat analogous to the Unix “pipe” where you send output from one command to another to a third, etc.

The simplest aggregation you are probably already familiar with is the SQL group by expression. We already saw the simple `countDocuments()` and `count()` method, but what if we want to see how many movies we have for the different ratings?

```
db.movies.aggregate( [{$group:{_id:'$rating', total: { $sum:1 }}}]) 
```

> **What you should see:** The aggregated result documents — one document per distinct `rating` value, each with an `_id` (the rating) and a `total` (the count of movies with that rating).

> **What just happened?** MongoDB's aggregation pipeline processed all documents through a single `$group` stage that grouped them by the `rating` field and summed up the count for each group, similar to a SQL `GROUP BY rating` with `COUNT(*)`.

In the shell we have the aggregate helper which takes an array of pipeline operators. For a simple count grouped by something, we only need one such operator and it’s called `$group`. This is the exact analog of GROUP BY in SQL where we create a new document with `_id` field indicating what field we are grouping by (here it’s rating) and other fields usually getting assigned results of some aggregation, in this case we `$sum 1` for each document that matches a particular rating. You probably noticed that the `_id` field was assigned `$rating` and not only `rating` - the `$` before a field name indicates that the value of this field from incoming document will be substituted.

What are some of the other pipeline operators that we can use? 

The most common one to use before (and frequently after) `$group` is the `$match` - this is exactly like the find method and it allows us to aggregate only a matching subset of our documents, or to exclude some documents from our result.

In the following example we group by `genres` and count the number of movies for each genre. Because the `genres` field is an array, we first have to use `$unwind` to flatten the array. We also return the minimum, maximum and average rating for each group. The result is sorted by the number of movies per genre in descending order. 

```
db.movies.aggregate([
					{$match: {year:{$gt:2000}}}, 
					{$unwind: "$genres" }, 
					{$group: {_id:'$genres',
					    number :{ $sum:1 },
					    minRating:{$min:'$rating'}, 
					    maxRating:{$max:'$rating'}, 
					    avgRating:{$avg:'$rating'}
					}}, 
					{$sort:{number:-1}} ])
```

Execution should return the following result

```
[
  {
    _id: 'Drama',
    number: 11,
    minRating: 8.5,
    maxRating: 9,
    avgRating: 8.636363636363637
  },
  {
    _id: 'Adventure',
    number: 7,
    minRating: 8.5,
    maxRating: 8.9,
    avgRating: 8.7
  },
  {
    _id: 'Fantasy',
    number: 5,
    minRating: 8.5,
    maxRating: 8.9,
    avgRating: 8.74
  },
  {
    _id: 'Sci-Fi',
    number: 4,
    minRating: 8.5,
    maxRating: 8.8,
    avgRating: 8.625
  },
  {
    _id: 'Thriller',
    number: 4,
    minRating: 8.5,
    maxRating: 9,
    avgRating: 8.675
  },
  {
    _id: 'Crime',
    number: 3,
    minRating: 8.5,
    maxRating: 9,
    avgRating: 8.700000000000001
  },
  {
    _id: 'Action',
    number: 3,
    minRating: 8.7,
    maxRating: 9,
    avgRating: 8.833333333333334
  },
  {
    _id: 'Biography',
    number: 2,
    minRating: 8.5,
    maxRating: 8.5,
    avgRating: 8.5
  },
  {
    _id: 'Mystery',
    number: 2,
    minRating: 8.5,
    maxRating: 8.5,
    avgRating: 8.5
  },
  {
    _id: 'Music',
    number: 2,
    minRating: 8.5,
    maxRating: 8.5,
    avgRating: 8.5
  },
  {
    _id: 'Animation',
    number: 1,
    minRating: 8.5,
    maxRating: 8.5,
    avgRating: 8.5
  },
  {
    _id: 'Family',
    number: 1,
    minRating: 8.5,
    maxRating: 8.5,
    avgRating: 8.5
  },
  {
    _id: 'Comedy',
    number: 1,
    minRating: 8.5,
    maxRating: 8.5,
    avgRating: 8.5
  },
  {
    _id: 'War',
    number: 1,
    minRating: 8.5,
    maxRating: 8.5,
    avgRating: 8.5
  }
]
```

> **What you should see:** The aggregated result documents — one document per genre (for movies released after 2000), each showing the genre name, the number of movies, and the minimum, maximum, and average rating, sorted by the number of movies in descending order.

> **What just happened?** MongoDB's aggregation pipeline processed documents through a series of stages: `$match` filtered to movies released after 2000, `$unwind` flattened each movie's `genres` array into separate documents, `$group` grouped by genre while computing count and rating statistics, and `$sort` ordered the results by movie count descending. Each stage transforms the stream of documents passed to the next stage.

There is another powerful pipeline operator called `$project` (analogous to the projection we can specify to the find command) which allows you not just to include certain fields, but to create or calculate new fields based on values in existing fields. For example, you can use math operators to add together values of several fields before finding out the average, or you can use string operators to create a new field that’s a concatenation of some existing fields.

This just barely scratches the surface of what you can do with aggregations. Consult the [MongoDB's documentation](https://docs.mongodb.com/manual/core/aggregation-pipeline/index.html) to read more about Aggregation Pipelines.  

## Removing Documents

For removing one or more documents, just use what we have learned about the Query Selectors, but as a parameter to the `deleteOne` command instead of the `find` command. 

If we want to remove a specific document, for example the movie "Fight Club", we can perform

```
db.movies.deleteOne( { "title": "Fight Club" } )
```

The result will show how many documents have been removed:

```
> db.movies.deleteOne( { "title": "Fight Club" } )
{ acknowledged: true, deletedCount: 1 }
```

> **What you should see:** A result object with `deletedCount: 1`, confirming that exactly one document was removed.

We can see that as expected, one movie has been removed. 

We can easily also remove the rest of the additional movies we have added before with the following command, specifying to remove all documents where there is no `plotOutline` field. 

```
db.movies.deleteMany( { "plotOutline": { $exists: false} } )
```

> **What you should see:** A result object with `deletedCount` equal to the number of documents that had no `plotOutline` field — the 48 minimal movie documents inserted with `insertMany`.

## Using the Python API with MongoDB