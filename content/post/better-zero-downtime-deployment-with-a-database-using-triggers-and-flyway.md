+++
date = "2016-09-23T14:29:41+02:00"
title = "Better 0-Downtime Deployment with a Database using Triggers (& Flyway)"
tags = ["PostGreSQL", "zero downtime deployment", "triggers", "database", "Flyway"]
aliases = [
  "/2016/09/23/better-ø-downtime-deployment-with-a-database-using-triggers--flyway/"
]
+++

This summer I was initiated into the arcane art of zero downtime deployment.
A vendor, for better or worse, mandated that application releases are deployed
on their platform without sacrificing the service's availability. Achieving no downtime
for a stateless application is more or less straightforward. Take down the application server,
upgrade the application, and restart the server; repeat per replica. Bringing
a database into the picture complicates the process because alterations to the database schema or
data must not break the older application which is live until the upgrade is over (i.e.,
assuming the upgrade isn't rolled back). If matters weren't complicated enough, a
subtle requirement for correctness one has to keep in mind is that data must be
consistent relative to the version of the application reading/writing it. For example,
if app V1 writes _x_ to column A and now app V2 starts writing _x′_ to column B
superseding column A, we expect app V1's user to see _x_ while app V2's user to
see _x′_ instead of _x_ while both versions of the application are running side by side given
that, logically, column A and column B represent the same field.

I was on unchartered terrain here and, after researching various strategies, I
came across [Marcin's zero downtime deployment for databases](https://spring.io/blog/2016/05/31/zero-downtime-deployment-with-a-database)
article. Marcin explains how to achieve zero downtime for a database schema upgrade by enumerating through
a sequence of _n_ application deployments where _n - 1_ intermediate deployments migrate
the database with [Flyway](https://flywaydb.org/) in a backwards compatible way. In the final deployment,
assuming the previous deployments went well, the database is changed to its final
state such that (1) any un-migrated data slipped in from the previous deployment (i.e., _n - 1_)
is migrated   and (2) the schema is no longer backwards compatible.

What struck me about the above approach is the relatively long deployment procedure
ops have to follow. Minimally, ops are required to make 3 deployments multiplied by the number of replicas.
Moreover, for each deployment, a different version of the application needs to be released.
This means bumping the application version no., tagging the released application source code,
building it, and so on. Leaving aside logistics, this strategy necessitates
that you temporarily relax _NOT NULL_ constraints  in the intermediate deployments
since app V1 doesn't know about the new columns giving way to the possibility of
tainted data (naturally, one can implement a less safe form of null check constraint
in the application).

Given the mentioned drawbacks, I decided to radically alter the approach. Instead of going
through a long sequence of deployments, I shifted the strategy to using database insert/update triggers.
All the popular databases have support for triggers and the vendor's database, PostgreSQL, wasn't
an exception. Consider the following table taken from the article's example, and adapted
to PostGreSQL, where the _surname_ column will be added:

<script src="https://gist.github.com/claudemamo/e4b3af389f7a5ba031f7813716c0c3de.js?file=V1__init.sql"></script>

The migration script adding the _surname_ column to the _PERSON_ table is like
the article's but with one crucial difference:

<script src="https://gist.github.com/claudemamo/e4b3af389f7a5ba031f7813716c0c3de.js?file=V2__Add_surname(1).sql"></script>

Can you spot it? I've made _surname_ _NOT NULL_ guaranteeing a value to
be always be present. The subsequent _ON INSERT OR UPDATE_ trigger will copy from
_last\_name_ to _surname_ preventing the _NOT NULL_ constraint from firing when app
V1 is inserting a record into _PERSON_:

<script src="https://gist.github.com/claudemamo/e4b3af389f7a5ba031f7813716c0c3de.js?file=V2__Add_surname(2).sql"></script>

With the trigger in place, I only have to deploy the application once for every
replica because the heavy lifting is performed inside the system-wide trigger
as opposed to the application. In the next release, the trigger
and the obsolete column will be dropped from the database.

Note that if the database we were on wasn't PostGreSQL, it might be possible
for data to be inserted or updated without firing the trigger. Indeed, the _NOT NULL_
constraint imposed on the _surname_ column may be fired from app V1 if an attempt to insert the data is performed
between the time _surname_ is added and the trigger is created. The reason
is that many databases don't support transactional DDL. However, this [isn't the case
for PostGreSQL](https://wiki.postgresql.org/wiki/Transactional_DDL_in_PostgreSQL:_A_Competitive_Analysis)
and therefore the DML and DDL will run in single transaction.
