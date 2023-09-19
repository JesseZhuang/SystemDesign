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
### 2.2.2 MapReduce Querying

## 2.3 Graph-Like Data Models

### 2.3.1 Property Graphs
### 2.3.2 The Cypher Query Language
### 2.3.3 Graph Queries in SQL
### 2.3.4 Triple-Stores and SPARQL
### 2.3.5 The Foundation: Datalog

## 2.4 Summary

<!-- references -->

[1]:

[3]:

[4]:

[9]:

[15]:

[17]:

[18]:

[22]:

[27]:

[28]:

[29]:
