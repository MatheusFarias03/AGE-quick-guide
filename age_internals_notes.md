---
title: "Personal Notes from the Apache AGE Internals"
permalink: /
layout: default
---

#  Apache AGE Quick Guide

## Introduction
This is a quick guide I made for the Apache AGE. It sums up some things I have read from different sources to have a concise understanding on how to use AGE, how it works internally, and how to create some data with it. This guide shows how to set AGE with PostgreSQL, the internals of this extension and some Cypher queries to use with it. Documentations and further explanation of the topics covered here can be viewed in the links provided at the "References" section. 

## Overview

Apache AGE is a PostgreSQL extension that provides graph database functionality. AGE is an acronym for "A Graph Extension", and is inspired by Bitnine's fork of PostgreSQL 10, AgensGraph, which is a multi-model database. The goal of AGE is to create a single storage that can handle both relational and graph model data so that users can use standard ANSI SQL along with openCypher, the Graph query language.

## Setup
First of all, you will need to install the essential libraries according to each OS. Building AGE from source depends on the following Linux libraries:
* Ubuntu : `sudo apt-get install build-essential libreadline-dev zlib1g-dev flex bison` ;
*  Fedora : `dnf install gcc glibc bison flex readline readline-devel zlib zlib-devel` ;
* CentOS : `yum install gcc glibc glib-common readline readline-devel zlib zlib-devel flex bison` .

You will also need to install a AGE compatible version of Postgres. For now, AGE only supports Postgres 11 and the ALPHA of 12. To install Postgres from a package manager:
* Postgres 11 : `sudo apt install postgresql-server-dev-11`
* Postgres 12 : `sudo apt install postgresql-12`

Then, clone the github repository at : https://github.com/apache/agex

Now, inside the AGE directory, build and install the extension with the following command : 
```
make PG_CONFIG=/path/to/postgres/bin/pg_config install
```

## Post Installation AGE Setup

After the installation, open a connection to a running instance of your database and run the `CREATE EXTENSION` command to have AGE be installed on the server.

```sql
CREATE EXTENSION age;
```

## Per Session Instructions

For every connection of AGE you start you will need to load AGE extension.

```sql
LOAD 'age';
```
It is recommended adding ag_catalog to the sarch_path to simplify the queries. If not done so, remember to add 'ag_catalog' to your cypher query function calls.

```sql
SET search_path = ag_catalog, "$user", public;
```

## Graphs

A *graph* consist of a set of *vertices* and *edges*, where each individual node and edge possesses a map properties. A vertex is the basic object of a graph, that can exist independently of everything else in the graph. An edge creates a directed connection between two vertices.

### Vertex
An agtype vertex is its graphid, label, and it's properties:
```JSON
{
    "id" : 1125899906842777,
    "label": "city",
    "properties": {"key": "value"}
}::vertex
``` 

### Edge
An edge is its graphid, label, the graphid of its start and end vertex and its properties:
```JSON
{
    "id": 1125899906842777,
    "label": "has_city",
    "start_id": 1125899906842777,
    "end_id": 1125899906842777,
    "properties": {"key": "value"}
}::edge
```

### Path
A path is a list of alternating vertices and edges:
```
[vertex, edge, vertex... etc]::path
```

### Creating a Graph
To create a graph, use the `create_graph` function, located in the ag_catalog namespace. This function will not return any results. The graph is created if there is no error message. Tables needed to setup the graph are autmoatically created. 

Example:
```sql
SELECT * FROM ag_catalog.create_graph('graph_name');
``` 

### Deleting a Graph

Use `drop_graph` function to delete the graph. It is located in the `ag_catalog` namespace. This function will not return any results. If there is no error message the graph has been deleted. It is recommended to set the cascade option to true, otherwise everything in the graph must be manually dropped with SQL DDL commands.

Example:
```sql
SELECT * FROM ag_catalog.drop_graph('graph_name', true);
```

## How Data is Stored in Apache AGE

### Graphs 
AGE uses a Postgres namespace for every individual graph. Data for a graph is stored in the table `ag_catalog.ag_graph`. Running the query: 

```sql 
SELECT oid, * FROM ag_catalog.ag_graph; 
```
will return the following result:

```
   oid   |    name   |     namespace     |
---------+-----------+-------------------+
  16937  |  database |     database      |
(1 row)
```
*- oid is the Postgres Object Identifier, which are 4-byte integers. All the database objects in PostgreSQL are internally managed by respective OIDs.*
### Labels
A label is a table that holds all vertices/edges associated with the label. The data for a label is stored in the ag_label. By typing the following query, you can view the available labels:

```sql
SELECT oid, * FROM ag_catalog.ag_label;
```
```
    oid   |        name       | graph | id | kind |        relation          |
----------+-------------------+-------+----+------+--------------------------+
   16950  | _age_label_vertex | 16937 |  1 | v    | database.ag_label_vertex |
   16963  | _age_label_edge   | 16937 |  2 | e    | database.ag_label_edge   |
   16975  | Country           | 16937 |  3 | v    | database."Country"       |
   16987  | City              | 16937 |  4 | v    | database."City"          |
   16999  | has_city          | 16937 |  5 | e    | database.has_city        |
```
With each column of the table above, we can identify some field names:
* oid : An unique Postgres identifier per label ;
* name: The name of the label ;
* graph: Oid from ag_graph ;
* id : The id for the label, it is unique for each graph ;
* kind : Shows if it is a vertex "v" or an edge "e" ;
* relation : Schema qualified name of table name for label.

### Vertices
*Vertices* have unique identifiers and a set of properties associated with them. You can see all the available vertices of a selected label with the following query (The "City" is just an example of a label and "database" the name of the database you are working with):
```sql
SELECT * FROM database."City";
```

With this query, it will show the **id** and **properties** as columns of each vertex. The id will be an integer number and the properties will be stored as a JSON object with key : value pairs.

### Edges
Each *edge* has an unique identifier, the identifier of the vertices it is associated with and it's properties. Below is an example on how to show the edges available on the graph:

```sql
SELECT * FROM cypher_vle.edge;
```
This query will show the **id** of the edge, **start_id**, **end_id**, and **properties**. The start_id and end_id will show the id of the vertex it starts with and ends with, respectively. 

## Cypher

*Cypher* is a declarative graph query language that allows data querying in a graph. Cypher queries are assembled with patterns of nodes and relationships with any specific filter on label or properties to use the CRUD methods.

Like SQL, Cypher has a variety of keywords for specifying, filtering patterns, and returning results. The most common are: `MATCH`, `WHERE` and `RETURN`.

* `MATCH` is used for finding nodes, relationships, or combinations of nodes and relationships together ; 
* `WHERE` in cypher is used to add additional constraints to patterns and filter unwanted patterns ;
* `RETURN` in cypher formats and organizes the way that the result will be outputted.

An example of a cypher query would be the following:
```sql
MATCH (jorge:Actor {name: 'Jorge Mário da Silva'})-[:ACTED_IN]->(movie:Movie)
WHERE movie.year < $yearParameter
RETURN movie
```
This example will search for the pattern of the node with the Actor label and with the name "Jorge Mário da Silva" connected by a relationship (ACTED_IN) to another node, in this case, a node with the "Movie" label. The `WHERE` clause filters to only keep patterns where the Movie node in the `MATCH` clause has a year property that is less than the value passed in `$yearParameter`.

*- A more indepth information about cypher syntax and a guide about it can be accessed on: https://neo4j.com/developer/cypher/intro-cypher/*

### Using Cypher with AGE
Cypher queries are constructed using a function called cypher in ag_catalog, which returns a Postgres SETOF records. A `RECORD` in Postgres is a placeholder data type that allows the user to define the output columns at runtime. And `SETOF` means that multiple rows might be returned.

The syntax for using cypher with AGE is: ``` cypher(graph_name, query_string, parameters) ```.
* graph_name is the graph for the Cypher query ;
* query_string is the Cypher query to be executed ;
* parameters is an optional map of parameters used for Prepared Statements. Default is NULL.

The query utilising this syntax would be something like this:
```sql
SELECT * FROM cypher(
  'graph_name',
  $$ /*Cypher query here*/ $$
) AS (result1 agtype, result2 agtype);
```

Note that the "`AS (result1 agtype, result2 agtype)`" defines the `RECORD` of the query.

## CRUD
CREATE, READ, UPDATE, DELETE (CRUD) are the four basic operations of persistent storage. In this section we can see some examples of the CRUD operations with Cypher on AGE. More detailed explanations are available on the links at the end of each operation.

### Create
The **CREATE** clause is used to create graph vertices and edges. A create clause that is not followed by another clause is called a terminal clause. When a cypher query ends with a terminal clause, no results will be returned from the cypher function call. However, the cypher function call still requires a column list definition. When cypher ends with a terminal node, define a single value in the column list definition: no data will be returned in this variable.

A query for creating a new vertex with labels can be done as:

```sql
SELECT * 
FROM cypher(
  'graph_name', 
  $$ CREATE (:Person {name: 'Dennis Ritchie', title: 'Programmer'})$$
) as (n agtype);
```
To create an edge between two vertices, we first get the two vertices. Once the nodes are loaded, we simply create an edge between them.

```sql
SELECT * FROM cypher(
  'graph_name',
  $$ MATCH (a:Person), (b:Person)
  WHERE a.name = 'Node A' AND b.name = 'Node B'
  CREATE (a)-[e:RELTYPE]->(b)
  RETURN e $$
) as (e agtype);
```
*- More about the CREATE clause with AGE on: https://age.apache.org/age-manual/master/clauses/create.html*

### Read
The **MATCH** clause allows you to specify the patterns Cypher will search for in the database. This is the primary way of getting data into the current set of bindings. 

By just specifying a pattern with a single vertex and no labels, all vertices in the graph will be returned.

```sql
SELECT * FROM cypher(
  'graph_name', 
  $$ MATCH (v)
  RETURN v $$
) as (v agtype);
```

To return related vertices within a table, the -[]- symbol is used. It means "related to", without regarding to the type or relation to the edge.

```sql
SELECT * FROM cypher(
  'graph_name', 
  $$ MATCH (director {name: 'John Carpenter'})-[]-(movie) 
  RETURN movie.title $$
) as (title agtype);
```

*- More about the MATCH clause with AGE on: https://age.apache.org/age-manual/master/clauses/match.html*

### Update
The **SET** clause is used to update labels on nodes and properties on vertices and edges. To remove a property, normally it is used **REMOVE**, but sometimes the **SET** command also works.

An example of the SET clause with cypher on AGE:
```sql
SELECT * 
FROM cypher(
  'graph_name', 
  $$ MATCH (v {name: 'Andres'})
  SET v.surname = 'Taylor'
  RETURN v $$
) as (v agtype);
```

An example of the REMOVE clause with cypher on AGE:
```sql
SELECT * 
FROM cypher('graph_name', $$
    MATCH (andres {name: 'Andres'})
    REMOVE andres.age
    RETURN andres
$$) as (andres agtype);
```
*- More about the SET clause at: https://age.apache.org/age-manual/master/clauses/set.html*

*- More about the REMOVE clause at: https://age.apache.org/age-manual/master/clauses/remove.html*

### Delete
The DELETE clause is used to delete graph elements—nodes, relationships or paths. You cannot delete a node without also deleting edges that start or end on said vertex. Either explicitly delete the vertices,or use DETACH DELETE.

Deleting single vertex :
```sql
SELECT * 
FROM cypher('graph_name', $$
	MATCH (v:Useless)
	DELETE v
$$) as (v agtype);
```

Deleting all vertices (use the DETACH option to first delete a vertice’s edges then delete the vertex itself) : 

```sql
SELECT * 
FROM cypher('graph_name', $$
	MATCH (v:Useless)
	DETACH DELETE v
$$) as (v agtype);
```

*-More about the DELETE clause at: https://age.apache.org/age-manual/master/clauses/delete.html*

## References
* Internals of PostgreSQL - https://www.interdb.jp/pg/index.html
* Apache AGE Documentation - https://age.apache.org/age-manual/master
* Neo4j Intro to Cypher - https://neo4j.com/developer/cypher/intro-cypher/
* Cypher on Wikipedia - https://en.wikipedia.org/wiki/Cypher_(query_language)