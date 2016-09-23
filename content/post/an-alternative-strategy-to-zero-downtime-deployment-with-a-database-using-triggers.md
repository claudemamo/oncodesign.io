+++
date = "2016-09-23T14:29:41+02:00"
title = "an alternative strategy to zero downtime deployment with a database using triggers"
draft = true
+++

This summer I was initiated into the arcane art of zero downtime database deployment.
A vendor, for better or worse, mandated that application releases are deployed
on their platform without sacrificing the service's availability. Achieving no downtime
for a stateless application is more or less trivial. Take down the application server,
upgrade the application, and restart the server; repeat for each replica. Bringing
in a database into the picture complicates the process because changes to the database schema
must not break the older application which is live until the upgrade is over (i.e.,
assuming the upgrade isn't rolled back). That's not all. A subtle requirement one
must be keep in mind is that data must be correct relative to version of the application
reading/writing it. For example, if app V1 writes value X to column A and now app V2
starts writing value Y to column A, we expect app V1's user to get X when reading
from column A while app V2's user to get Y when reading from the same column.

I was on unchartered terrain here and, after researching different strategies, I
came upon [Marcin's zero downtime deployment for databases](https://spring.io/blog/2016/05/31/zero-downtime-deployment-with-a-database)
article. Marcin explains how to achieve zero downtime for a database schema upgrade by enumerating through
a sequence of n application deployments where n - 1 intermediate deployments migrate
the database, using [Flyway](https://flywaydb.org/), in a backwards compatible way. In the final deployment,
assuming previous deployments went well, the database schema is changed to its final
state such that (1) any data slipped in from the older version of the application
between the first and last deployment is migrated and  (2) the schema is no longer
backwards compatible.

What struck me about the above approach is the relatively long deployment procedure
ops have to follow: minimally, ops are required to make 3 deployments. Moreover,
for each deployment, a different version of the application needs to be released.
This means promoting the application's version no. to released, tagging the released
application source code, building the application, and so on. Leaving aside logistics,
this strategy necessitates that you relax NOT NULL constraints in the intermediate
deployments since app V1 doesn't know about the new columns.

Given the mentioned drawbacks, I decided to alter the approach. Instead of going
through a series of deployments, I opted for using database insert/update triggers.
All popular databases have support for triggers and our database, PostgreSQL, wasn't
the exception. Consider the following table taken from the article's example, adapted
to PostGreSQL and where the _surname_ column will be added:

<script src="https://gist.github.com/claudemamo/e4b3af389f7a5ba031f7813716c0c3de.js?file=V1__init.sql"></script>

The SQL for adding the _surname_ column is like the article's BUT with one critical
difference:

<script src="https://gist.github.com/claudemamo/e4b3af389f7a5ba031f7813716c0c3de.js?file=V2__Add_surname(1).sql"></script>

Can you spot the difference? I've made _surname_ NOT NULL guaranteeing a value to
be always be present. The subsequent ON INSERT OR UPDATE trigger will copy from
_last_name_ to _surname_ preventing the NOT NULL constraint from firing when app
V1 is inserting a record into PERSON:

<script src="https://gist.github.com/claudemamo/e4b3af389f7a5ba031f7813716c0c3de.js?file=V2__Add_surname(2).sql"></script>

With the trigger in place, I only have to deploy the application once for every
each replica because the heavy lifting is performed inside of the system-wide trigger
as opposed to particular versions of the application. In the next release, the trigger
and the deprecated column will be dropped from the database.

Note that if the database we're running on wasn't PostGreSQL, it might be possible
for data to be inserted or updated without firing the trigger. Indeed, the NOT NULL
constraint may be fired from app V1 if an attempt to insert the data is performed
between the time line 1 finishes execution and line 2 starts execution. The reason
is that many databases don't support transactional DDL. However, this [isn't the case
for PostGreSQL](https://wiki.postgresql.org/wiki/Transactional_DDL_in_PostgreSQL:_A_Competitive_Analysis)
and so the DML and DDL will run in single transaction.
