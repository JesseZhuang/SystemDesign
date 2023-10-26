# Data Models and Query Languages

Most applications are built by layering one data model on top of another. For each layer, the key question is: how is it represented in terms of the next lower layer. For example:

1. As an application developer, you model the real word in terms of objects or data structures, and APIs that manipulate those data structures.
1. You express the data structures in terms of a general-purpose data model, such as JSON or XML documents, tables in a relational database, or a graph model.
1. The engineers who built your DB software decided on a way of representing that JSON/XML/relational/graph data in terms of bytes in memory, on disl, or on a network. The representation may allow query, search, manipulation, and processing in various ways.
1. On yet lower levels, hardware engineers have figured out how to represent bytes in terms of electrical currents, pulses of light, magnetic fields, and more.


Data model has a profound effect on what the software above it can and can't do, it's important to choose one appropriate.

## 2.1 Relational Model Versus Document Model

Best known data model is probably SQL [1]: data is organized into relations (tables), where each relation is an unordered collection of tuples (rows).

The dominance of relational databases has lasted around 30 years since mid-1980s, an eternity in computing history.

The use cases appear mundane from today's perspective:

- transaction processing: sales, bank transactions, airline reservations, stock-keeping in warehouses
- batch processing: customer invoicing, payroll, reporting

Other databases at the time forced application developers to think a lot about the internal representation of the data in the database. The goal of the relational model was to hide the implementation detail behind a cleaner interface.

In the 1970s and early 1980s, the network model and the hierarchical model were the main alternatives, but the relational model came to dominate them. Object databases came and went again in the late 1980s and early 1990s. XML databases appeared in the early 2000s, but have only seen niche adoption.

As computers became more powerful and networked, relational model continue to generalize well, beyond their original scope of business data processing, to a broad variety: online publishing, discusssion, social networking, .etc.

### 2.1.1 The Birth of NoSQL

The name "NoSQL" is unfortunate, originally intended simply as a catchy Twitter hashtag for a meetup on open source, distributed, nonrelational databases in 2009 [3]. It has been retroactively reinterpreted as Not Only SQL [4].

Driving forces:

1. Greater scalability than relational databases, including very large datasets or very high write throughput
1. Widespread preference for free and open source software over commercial database products
1. Specialized query opterations not well supported by the relational model
1. Frustration with the restrictiveness of relational schemas, desire for a more dynamic and expressive data model

### 2.1.2 The Object-Relational Mismatch

The disconnect between the object-oriented model and the SQL data model is sometimes called an impedance mismatch (a term borrowed from electronics).

Object-relational mapping (ORM) frameworks like ActiveRecord and Hibernate reduce boilerplate code required for the translation layer but they can't completely hide the differences between the two models.

For a resume or a LinkedIn profile, the profile as a whole can be identified by a unique identifier, `user_id`. Fields like `first_name` and `last_name` can be columns on the `users` table. However, most people have had more than one job and varying numners of education. The one-to-many relationship can be represented:

1, In the traditional SQL (prior to SQL 1999), put positions, education, and contact information in separate tables, with a foreign key reference to the `users` table.
1. Later versions of SQL added support for structured datatypes and XML, with support for querying and indexing inside those documents. These features are supported to varying degrees by Orable, IBM DB2, MS SQL Server, and PostegreSQL. A JSON datatype is also supported by some, including IBM DB2, MySQL, and PostgreSQL.
1. A third options is to encode jobs as a JSON or XML, store as text column and let the application interpret structure and content. In this setup, you typically cannot use the database to query values inside that encoded column.

For a data structure like a resume, which is mostly a self-contained document, a JSON representation is quite appropriate. JSON has the appeal of much simper than XML. Document oriented databses like MongoDB, RethinkDB, CouchDB, and Espresso support this data model.

The JSON representation has better locality than the multi-table schema. If you want to fetch a profile in the relational example, you need to either perform multiple queries (query each table by `user_id`) or perform a messy multi-way join between the user table and its subordinate tables. In the JSON example, all the information is in one place and one query is enough.

The one-to-many relationships from the user to positions, contact info, and education history imply a tree structure, and the JSON makes that explicit.

![](./2-2.1tMTree.png)

### 2.1.3 Many-to-One and Many-to-Many Relationships

In previous example, `region_id` and `industry_id` are given as IDs, not plain-text strings.

When you use an ID, the information meaningful to human is stored only at one-place. The advantage is the ID never has to change and since it is not meaningful to human. The info it identifies can change and ID remains the same. If the information is duplicated, all the redudant copies need to be updated. That incurs write overheads, and risks inconsistences (some copies updated and some not). Removing such duplication is the key idea of normalization in relational databses (The distinctions among the normal forms are of little practical impact. As a rule of thumb, duplicating values indicate the schema is not normalized).

Normalizing this data requires many-to-one relationships, which don't fit nicely into the document model. In document databases, joins are not needed for one-to-many tree structures, and support for joins is often weak (supported in RethinkDB, not in MongoDB, only supported in predeclared views in CouchDB). If the database does not support joins, you have to emulate joins in applicaiton code by making multiple queries.

Moreover, initial version of an application may fit well in a join-free document model. Data has a tendency of becoming more interconnected as features are added.

### 2.1.4 Are Document Databases Repeating History?

The most popular database for business data processing in the 1970s was IBM's Information Management System (IMS), which used a fairly simple data model called he hirearchical model. It has some remarkable similarities to the JSON model used by document databases.

Like document databases, IMS worked well for one-to-many relationships, but it made many-to-many relationships difficult, and did not support joins. Developers had to decide whether to duplicate (denormalize) data or to manually resolve references from one record to another. These problems of the 1960s and '70s were much like the problems with document databases today [15].

Various solutions were proposed to solve the limitations of the hierarchical model. The two most prominent were the relational model (which became SQL) and the network model (which initially had a large following but eventually faded into obscurity). The "great debate" between these two camps lasted for much of the 1970s.

**The network model**

The network model was standarized by a committee called the Conference on Data Systems Languages (CODASYL). The CODASYL model was a generalization of the hirearchical model. In the tree structure of the hierarchical model, every record has exactly one parent; in the network model, a record could have multiple parents. This allowed many-to-one and many-to-many relationships to be modeled.

The links in the network model were not foreign keys but more like pointers (while still being stored on disk). The only way of accessing a record was to follow a path from a root record along these chains of links. This was called an access path. If a record had multiple parents, the application code had to keep track of all the various relationships. Even CODASYL committee members admitted that this was like navigating around an n-dimensional data space [17].

**The relational model**

The relational model lay out all the data in the open: a relation (table) is simply a collection of tuples (rows). You can insert a new row into any table without worring about foreign key relationships. Foreign key constraints allow to restrict modifications, but such constraints are not required by the relational model. Even with constraints, joins on foreign keys are performed at query time, whereas in CODASYL, the join was effectively done at insert time.

The "access path" are made automatically by the query optimizer, not by the application developer. If you want to query in new ways, you can declare a new index. You don't need to change your queries to take advantage of a new index. The relational model made it much easier to add new features to applications.

Query optimizers are complicated beasts and have consumed many years of research and development [18]. But you only need to build a query optimizer once.

**Comparison to document databases**

Document databases reverted to the hierarchical model in one aspect: storing nested records (one-to-many relationships) within their parent record rather than in a separate table. However, when it comes to representing many-to-one and many-to-many relationships, relational and document databases are not fundamentally different. The related item is referenced by a unique id, a foreign key in relational model and a document reference in the document model [9]. The id is resolved at read time by using a join or follow-up queries. To date, document databases have not followed the path of CODASYL.

### 2.1.5 Relational Versus Document Databases Today

The main arguments in favor of the document data model are schema flexibility, better performance due to locality, and that for some applications it is closer to the data structures used by the application.

**Which data model leads to simpler application code?**

If the data in your application has a document-like structure (a tree of one-to-many relationships, typically entire tree is loaded at once), it's probably a good idea to use a document model.

If your application does use many-to-many relationships, the document model becomes less appealing.

For highly connected data, the document model is awkward, the relational model is acceptable, and graph models are the most natual.

**Schema flexibility in the document model**

Most document databases, xml and json support in relational databases do not enforce schema or optional. Document databases are sometimes called schemaless, but that is misleading. There is an implicit schema not enforced by the database. A more accurate term is schema-on-read, in contrast with schema-on-write (traditional approach with relational databases).

Schema-on-read is similar to dynamic (runtime) type checking in programming lan‐ guages, whereas schema-on-write is similar to static (compile-time) type checking. Just as the advocates of static and dynamic type checking have big debates about their relative merits [22], enforcement of schemas in database is a contentious topic, and in general there’s no right or wrong answer.

The difference between the approaches is particularly noticeable in situations where an application wants to change the format of its data. The schema-on-read approach is advantageous if the items in the collection don’t all have the same structure for some reason (i.e., the data is heterogeneous)—for example, because:

- There are many different types of objects, and it is not practical to put each type of object in its own table.
- The structure of the data is determined by external systems over which you have no control and which may change at any time.

**Data locality for queries**

A document is usually stored as a single continuous string, encoded as JSON, XML, or a binary variant thereof (such as MongoDB’s BSON). If your application often needs to access the entire document (for example, to render it on a web page), there is a performance advantage to this storage locality. If data is split across multiple tables, like in Figure 2-1, multiple index lookups are required to retrieve it all, which may require more disk seeks and take more time.

It’s worth pointing out that the idea of grouping related data together for locality is not limited to the document model. For example, Google’s Spanner database offers the same locality properties in a relational data model, by allowing the schema to declare that a table’s rows should be interleaved (nested) within a parent table [27]. Oracle allows the same, using a feature called multi-table index cluster tables [28]. The column-family concept in the Bigtable data model (used in Cassandra and HBase) has a similar purpose of managing locality [29].

**Convergence of document and relational databases**

Most relational database systems (other than MySQL) have supported XML since the mid-2000s. Many relational databases have also added support for JSON.

On the document database side, RethinkDB supports relational-like joins in its query language, and some MongoDB drivers automatically resolve database references.

A hybrid of the relational and document models is a good route for databases to take in the future.

## 2.2 Query Languages for Data

Declarative languages often lend themselves to parallel execution. Imperative code is very hard to parallelize across mul‐ tiple cores and multiple machines, because it specifies instructions that must be per‐ formed in a particular order.

### 2.2.1 Declarative Queries on the Web

In a web browser, using declarative CSS styling is much better than manipulating styles imperatively in JavaScript. Similarly, in databases, declarative query languages like SQL turned out to be much better than imperative query APIs. IMS and CODASYL both used imperative query APIs. Applications typically used COBOL code to iterate over records in the database, one record at a time.

### 2.2.2 MapReduce Querying

MapReduce [33] is neither a declarative query language nor a fully imperative query API, but somewhere in between. A limited form of MapReduce is supported by some NoSQL datastores, including MongoDB and CouchDB, as a mechanism for performing read-only queries across many documents. MongoDB 2.2 added support for a declarative query language called the aggregation pipeline [9].

## 2.3 Graph-Like Data Models

If your application has mostly one-to-many relationships (tree-structured data) or no relationships between records, the document model is appropriate. The rela‐ tional model can handle simple cases of many-to-many relationships, but as the con‐ nections within your data become more complex, it becomes more natural to start modeling your data as a graph.

Graphs are not limited to such homogeneous data: an equally powerful use of graphs is to provide a consistent way of storing completely different types of objects in a single datastore. For example, Facebook maintains a single graph with many different types of vertices and edges: vertices represent people, locations, events, checkins, and comments made by users; edges indicate which people are friends with each other, which checkin hap‐ pened in which location, who commented on which post, who attended which event, and so on [35].

In this section we will discuss the property graph model (implemented by Neo4j, Titan, and InfiniteGraph) and the triple-store model (implemented by Datomic, AllegroGraph, and others).

### 2.3.1 Property Graphs

You can think of a graph store as consisting of two relational tables, one for vertices and one for edges. By using different labels for different kinds of relationships, you can store several different kinds of information in a single graph, while still maintaining a clean data model.

A few things that would be difficult to express in a traditional relational schema, such as different kinds of regional structures in different countries (France has départements and régions, whereas the US has counties and states), quirks of history such as a country within a country (ignoring for now the intricacies of sovereign states and nations), and varying granularity of data (Lucy’s current residence is specified as a city, whereas her place of birth is specified only at the level of a state).

A graph can easily be extended to accommodate changes in your application’s data structures. For instance, you could use it to indicate any food allergies they have.

### 2.3.2 The Cypher Query Language

Cypher is a declarative query language for property graphs, created for the Neo4j graph database [37]. (It is named after a character in the movie The Matrix and is not related to ciphers in cryptography [38].)


```
CREATE
  (NAmerica:Location {name:'North America', type:'continent'}),
  (USA:Location      {name:'United States', type:'country'  }),
  (Idaho:Location    {name:'Idaho',         type:'state'    }),
  (Lucy:Person       {name:'Lucy' }),
  (Idaho) -[:WITHIN]->  (USA)  -[:WITHIN]-> (NAmerica),
  (Lucy)  -[:BORN_IN]-> (Idaho)
```
To find people who emigrated from US to Europe:

```
MATCH
  (person) -[:BORN_IN]->  () -[:WITHIN*0..]-> (us:Location {name:'United States'}),
(person) -[:LIVES_IN]-> () -[:WITHIN*0..]-> (eu:Location {name:'Europe'}) RETURN person.name
```

There are several possible ways of executing the query. The description given here suggests that you start by scanning all the people in the database, examine each person’s birthplace and residence, and return only those people who meet the criteria. But equivalently, you could start with the two Location vertices and work backward. If there is an index on the name property, you can probably efficiently find the two vertices representing the US and Europe. The query optimizer automatically chooses the strategy that is predicted to be the most efficient.

### 2.3.3 Graph Queries in SQL

Graph data can be represented in a relational database. In a relational database, you usually know in advance which joins you need in your query. In a graph query, you may need to traverse a variable number of edges before you find the vertex you’re looking for— that is, the number of joins is not fixed in advance.

In Cypher, ``:WITHIN*0..`` expresses that fact very concisely: it means “follow a WITHIN edge, zero or more times.” It is like the `*` operator in a regular expression. Since SQL:1999, this idea of variable-length traversal paths in a query can be expressed using something called recursive common table expressions (the WITH RECURSIVE syntax). The same query is 4 lines in Cyper and 29 lines in SQL.

### 2.3.4 Triple-Stores and SPARQL

The triple-store model is mostly equivalent to the property graph model. In a triple-store, all information is stored in the form of very simple three-part state‐ ments: (subject, predicate, object). For example, in the triple (Jim, likes, bananas), Jim is the subject, likes is the predicate (verb), and bananas is the object.

```
_:lucy     :name   "Lucy".
_:lucy     :bornIn _:idaho.
_:idaho    a       :Location.
_:idaho    :name   "Idaho".
```

#### The semantic web

The triple-store data model is completely independent of the semantic web—for example, Datomic [40] is a triple-store that does not claim to have anything to do with it. (Technically, Datomic uses 5-tuples rather than triples; the two additional fields are metadata for version‐ ing.) But the two are so closely linked in many people’s minds.

The semantic web is fundamentally a simple and reasonable idea: websites already publish information as text and pictures for humans to read, so why don’t they also publish information as machine-readable data for computers to read? The Resource Description Framework (RDF) [41] was intended as a mechanism for different websites to publish data in a consistent format, allowing data from different websites to be automatically combined into a web of data—a kind of internet-wide “database of everything.”

So far it hasn't shown any sign of being realized in practice. There is also a lot of good work that has come out of the semantic web project. Triples can be a good internal data model for appli‐ cations, even if you have no interest in publishing RDF data on the semantic web.

#### The RDF data model

The Turtle language we used in Example 2-7 is a human-readable format for RDF data. Sometimes RDF is also written in an XML format, which does the same thing much more verbosely—see Example 2-8. Turtle/N3 is preferable as it is much easier on the eyes, and tools like Apache Jena can automatically convert between differ‐ ent RDF formats if necessary.

#### The SPARQL query language

SPARQL is a query language for triple-stores using the RDF data model. (It is an acronym for SPARQL Protocol and RDF Query Language, pronounced “sparkle.”) It predates Cypher, and since Cypher’s pattern matching is borrowed from SPARQL, they look quite similar.

```
PREFIX : <urn:example:>
SELECT ?personName WHERE {
  ?person :name ?personName.
  ?person :bornIn  / :within* / :name "United States".
  ?person :livesIn / :within* / :name "Europe".
}
```

The same query as before—finding people who have moved from the US to Europe— is even more concise in SPARQL than it is in Cypher.

We discussed how CODASYL and the relational model competed to solve the problem of many-to-many relationships in IMS. At first glance, CODASYL’s network model looks similar to the graph model. Are graph databases the second coming of CODASYL in disguise?

No. They differ in several important ways:

• In CODASYL, a database had a schema that specified which record type could be nested within which other record type. In a graph database, there is no such restriction: any vertex can have an edge to any other vertex. This gives much greater flexibility for applications to adapt to changing requirements.
• In CODASYL, the only way to reach a particular record was to traverse one of the access paths to it. In a graph database, you can refer directly to any vertex by its unique ID, or you can use an index to find vertices with a particular value.
• In CODASYL, the children of a record were an ordered set, so the database had to maintain that ordering (which had consequences for the storage layout) and applications that inserted new records into the database had to worry about the positions of the new records in these sets. In a graph database, vertices and edges are not ordered (you can only sort the results when making a query).
• In CODASYL, all queries were imperative, difficult to write and easily broken by changes in the schema. In a graph database, you can write your traversal in imperative code if you want to, but most graph databases also support high-level, declarative query languages such as Cypher or SPARQL.

### 2.3.5 The Foundation: Datalog

In practice, Datalog is used in a few data systems: for example, it is the query lan‐ guage of Datomic [40], and Cascalog [47] is a Datalog implementation for querying large datasets in Hadoop. Datomic and Cascalog use a Clojure S-expression syntax for Datalog. Datalog define rules that tell the database about new predicates.The Datalog approach requires a different kind of thinking to the other query lan‐ guages discussed in this chapter, but it’s a very powerful approach, because rules can be combined and reused in different queries. It’s less convenient for simple one-off queries, but it can cope better if your data is complex.

## 2.4 Summary

Historically, data started out being represented as one big tree (the hierarchical model), but that wasn’t good for representing many-to-many relationships, so the relational model was invented to solve that problem. More recently, developers found that some applications don’t fit well in the relational model either. New nonrelational “NoSQL” datastores have diverged in two main directions:

1. Document databases target use cases where data comes in self-contained docu‐ ments and relationships between one document and another are rare.
2. Graph databases go in the opposite direction, targeting use cases where anything is potentially related to everything.

One thing that document and graph databases have in common is that they typically don’t enforce a schema for the data they store, which can make it easier to adapt applications to changing requirements. However, your application most likely still assumes that data has a certain structure; it’s just a question of whether the schema is explicit (enforced on write) or implicit (handled on read).

Although we have covered a lot of ground, there are still many data models left unmentioned. To give just a few brief examples:
1. Researchers working with genome data often need to perform sequence-similarity searches, which means taking one very long string (representing a DNA molecule) and matching it against a large database of strings that are similar, but not identical. None of the databases described here can handle this kind of usage, which is why researchers have written specialized genome database software like GenBank.
1. Particle physicists have been doing Big Data–style large-scale data analysis for decades, and projects like the Large Hadron Collider (LHC) now work with hundreds of petabytes! At such a scale custom solutions are required to stop the hardware cost from spiraling out of control.
1. Full-text search is arguably a kind of data model that is frequently used alongside databases. Information retrieval is a large specialist subject that we won’t cover in great detail in this book, but we’ll touch on search indexes in Chapter 3 and Part III.


<!-- references -->

[1]: Edgar F. Codd: “A Relational Model of Data for Large Shared Data Banks,” Com‐ munications of the ACM, volume 13, number 6, pages 377–387, June 1970. doi: 10.1145/362384.362685

[3]: Pramod J. Sadalage and Martin Fowler: NoSQL Distilled. Addison-Wesley, August 2012. ISBN: 978-0-321-82662-6

[4]: Eric Evans: “NoSQL: What’s in a Name?,” blog.sym-link.com, October 30, 2009.

[9]: “The MongoDB 2.4 Manual,” MongoDB, Inc., 2013.

[15]: Sarah Mei: “Why You Should Never Use MongoDB,” sarahmei.com, November 11, 2013.

[17]: Charles W. Bachman: “The Programmer as Navigator,” Communications of the ACM, volume 16, number 11, pages 653–658, November 1973. doi: 10.1145/355611.362534

[18]: Jsoseph M. Hellerstein, Michael Stonebraker, and James Hamilton: “Architecture of a Database System,” Foundations and Trends in Databases, volume 1, number 2, pages 141–259, November 2007. doi:10.1561/1900000002

[22]: Martin Odersky: “The Trouble with Types,” at Strange Loop, September 2013.

[27]: James C. Corbett, Jeffrey Dean, Michael Epstein, et al.: “Spanner: Google’s Globally-Distributed Database,” at 10th USENIX Symposium on Operating System Design and Implementation (OSDI), October 2012.

[28]: Donald K. Burleson: “Reduce I/O with Oracle Cluster Tables,” dba-oracle.com.

[29]: Fay Chang, Jeffrey Dean, Sanjay Ghemawat, et al.: “Bigtable: A Distributed Stor‐ age System for Structured Data,” at 7th USENIX Symposium on Operating System Design and Implementation (OSDI), November 2006.

[33]: Jeffrey Dean and Sanjay Ghemawat: “MapReduce: Simplified Data Processing on Large Clusters,” at 6th USENIX Symposium on Operating System Design and Imple‐ mentation (OSDI), December 2004.

[35]: Nathan Bronson, Zach Amsden, George Cabrera, et al.: “TAO: Facebook’s Dis‐ tributed Data Store for the Social Graph,” at USENIX Annual Technical Conference (USENIX ATC), June 2013.

[37]: “The Neo4j Manual v2.0.0,” Neo Technology, 2013.

[38]: Emil Eifrem: Twitter correspondence, January 3, 2014.

[40]: “Datomic Development Resources,” Metadata Partners, LLC, 2013.

[47]: Nathan Marz: “Cascalog,” cascalog.org.
