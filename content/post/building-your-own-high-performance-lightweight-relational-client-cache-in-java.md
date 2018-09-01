+++
date = "2018-09-01T18:17:34+02:00"
title = "Building your own High-Performance Lightweight Relational Client Cache in Java"
draft = true
+++

Some months ago, I was asked by a vendor to overhaul their Java client's cache. There current
implementation was a run-of-the-mill key-value data structure that cached the server's replies and keyed them by URL path.
The requirement driving the overhaul was for the client to run while disconnected from the server.
In other words, the cache was to be preloaded with data at startup. This posed a number of interesting challenges.
First of all, replicating data stores like CouchDB were ruled out given the technical constraints in the client
and server environments. Challenge number two was that the data to be cached was kept in a relational
database on the server. Even if the server's SQL could be run on the client, that wasn't enough: a number of
complicated filtering rules couldn't be codified in SQL. With the current design, pre-filtering the data and converting it to a
key-value structure on the server was out of the question. Luckily, there was a silver lining in all of this:
the server offered a JSON over HTTP API to export the database:

//snippet

Unluckily, the client was running in a stack (i.e., hardware, OS, application container, and the application itself) outside the vendor's control.
The application container was of particular concern because the client had to co-exist with the container and, potentially,
several applications that might not be good citizens. Despite containers try their best to isolate applications, anyone with enough experience,
can relate to the countless class loaders issues one can have due to some seemingly innocent application dependency.
As such, it was determined that the cache implementation should be conservative in terms of packaged dependencies (e.g., an ORM such as Hibernate was a no no),
though conversely, it could be liberal as long as the dependency was provided by the container.

Finally but not least, as the title suggests, the new cache had to mimic the high performance of the cache it was replacing.
The enterprises using the client were expected to hit the cache millions of times in a given day.

Given the above constraints, I'm going to outline a practical solution for ...

Data Store

H2 was chosen to store and serve the cached data. The tiny (1.7MB as of version 1.4.196) single JAR database has several ways
for storing data and one of them is in-memory. By leveraging H2's awesome CSV import feature, the JSON export downloaded from the
server could be transformed into CSV and imported into H2. Plenty of libraries exist for mapping JSON into CSV. Jackson was used
as it was a dependency provided by the container even though the CSV module (i.e., jackson-dataformat-csv) had to bundled with the client.
Here is an example for how one might go about writing out as CSV a map entry representing a table in the JSON export:

private File writeAsCsv(Map.Entry<String, List<Map<String, Object>>> table) throws IOException {
        CsvMapper csvMapper = new CsvMapper();
        CsvSchema.Builder schemaBuilder = CsvSchema.builder();
        for (Map<String, Object> record : table.getValue()) {
			for (String column : record.keySet()) {
				schemaBuilder.addColumn(column);
			}
        }
        CsvSchema schema = schemaBuilder.build().withLineSeparator("\r").withHeader();

        File file = File.createTempFile(table.getKey(), ".csv");
        file.deleteOnExit();
        try (Writer writer = new BufferedWriter(new OutputStreamWriter(new FileOutputStream(file), "UTF-8"))) {
            csvMapper.writer(schema).writeValue(writer, table.getValue());
        }

        return file;
    }

writeAsCsv(...) takes in a map entry representing a table where the key is
the table's name and the value is a list consisting of its records. With the help of Jackson, the table is
written out to a temporary CSV file and a reference to the file is return to the caller.

A means to talking to H2 was needed before the CSV could imported. Raw JDBC requires too much plumbing therefore, keeping in mind the
conservative dependency principle, the container's bundled Spring JDBC was used for DDL and DML operations. Before proceeding to
instantiate Spring's core JdbcTemplate class, the data source needs to be set up:

// snippet

All the H2 parameters in the JDBC URL were taken from the database docs. Of particular interest are:

*. nioMemFS: the cache could reach GBs in size so in order to reduce GC pauses, that data will be stored in the NIO portion of the JVM's memory
*  LOG=0;UNDO_LOG=0: disables the transaction and session undo log respectively. According to the doc, this reduces the import time

Since high-performance is a key requirement, the connections to the database are pooled with c3po: another dependency provided
by the container. One might be forgiven to think that connection pooling is of questionable usefulness for an embedded in-memory database,
however, tests showed that connection pooling does noticeably improve throughput. As a side note, H2's org.h2.jdbcx.JdbcConnectionPool class
performed poorly due to high lock contention.

With a datasource at hand, JdbcTemplate is initialised along with a few other classes:

//snippet

readWriteLock plays a lead role in cache invalidation. As you will see later on, for a brief moment, the database must be
made inaccessible to reads while tables are dropped and re-created with the latest fetched data. secondLevelCache is a java.util.Mao
keyed by query....


Prefetch and Invalidation

"execute" is run for DDL operations:

//snippet

The second level cache is cleared given that execute will only be invoked when the cache is created or invalidated.

"query" is run, well, for query operations:

//snippet

In the above code, on cache miss, the database will be queried. ColumnMapRowMapper maps result set rows to a list of maps.

Having now the facility to read and write to H2, let's observe the code for preloading the cache:

//snippet

Though perhaps not the best way of doing things, each column is indexed in line .. The cache might
contain hundreds of thousands of records and having indexes will help with the query time. Since the
database is read only, I don't need to worry about the performance of writes.


Cache invalidation

The KISS principle was followed for cache invalidation. Concretely, every so often, the client makes a
call to the server to check if changes were made to the data since the last download. This is where the If-Modified-Since HTTP header
is useful. If...


Querying

The cache couldn't use URL paths like the previous implementation to find the data the client wanted.
SQL had to be used given the data store is now a relational one. This lead to two interesting problems:

1. How will the cache know which query or queries to run for a given client operation?
2. From where will it get the SQL to query the database?

Problem 1 was solved by inserting a layer of business logic between the client and cache:



...




The SQL run on the server for returning data to clients was reused in the cache. The queries couldn't
be shared at compile time for technical reasons so a simple query repository was built on the server
 which made the SQL queries visible with the cache at run-time.

GET .../queryRepository?queryId=xxxx

{"query": "SELECT * FROM"}

At startup, the cache fetches all the queries it requires for reading data. The query IDs
are declared as constants in file:

//snippet

Each query ID corren


Filtering

The data returned by the cache couldn't be served "as is" to the client. The old
key-value cache had the luxury of caching already filtered data. However, the new cache,
didn't have such a luxury given it's serving data directly from the source.

client is disconnected from the server.

This business logic for processing the data
was executed on the server whe

As opposed to the SQL, the server's business logic could be shared


Once of the useful features


I'm going to describe
a solution for the afort

Query Repository
Second-level cache
H2
Bean Mapper
Refreshing
