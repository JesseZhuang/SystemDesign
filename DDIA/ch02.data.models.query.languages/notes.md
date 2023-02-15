# Data Models and Query Languages

Most applications are built by layering one data model on top of another. For each layer, the key question is: how is it represented in terms of the next lower layer. For example:

1. As an application developer, you model the real word in terms of objects or data structures, and APIs that manipulate those data structures.
1. You express the data structures in terms of a general-purpose data model, such as JSON or XML documents, tables in a relational database, or a graph model.
1. The engineers who built your DB software decided on a way of representing that JSON/XML/relational/graph data in terms of bytes in memory, on disl, or on a network. The representation may allow query, search, manipulation, and processing in various ways.
1. On yet lower levels, hardware engineers have figured out how to represent bytes in terms of electrical currents, pulses of light, magnetic fields, and more.

## 2.1 Relational Model Versus Document Model

### 2.1.1 The Birth of NoSQL
### 2.1.2 The Object-Relational Mismatch
### 2.1.3 Many-to-One and Many-to-Many Relationships
### 2.1.4 Are Document Databases Repeating History?
### 2.1.5 Relational Versus Document Databases Today

## 2.2 Query Languages for Data

### 2.2.1 Declarative Queries on the Web
### 2.2.2 MapReduce Querying

## 2.3 Graph-Like Data Models

### 2.3.1 Property Graphs
### 2.3.2 The Cypher Query Language
### 2.3.3 Graph Queries in SQL
### 2.3.4 Triple-Stores and SPARQL
### 2.3.5 The Foundation: Datalog

## 2.4 Summary


