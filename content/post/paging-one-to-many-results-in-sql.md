+++
date = "2017-04-22T09:33:52+02:00"
title = "Paging One-to-Many Results in SQL"
tags = ["pagination", "PostgreSQL", "SQL", "DENSE_RANK", "one-to-many"]
+++

Pagination in databases, the division of query results into manageable subsets, is often
implemented in terms of SQL constructs like _LIMIT_ or _FETCH FIRST_, and _OFFSET_. This
works well when the result is compromised of one-to-one relationships. On the other hand,
hypothesise what happens if the relationship is a one-to-many as in the following DDL [1]:

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=schema.sql"></script>

An author can write _n_ books and this relationship is expressed as a foreign key constraint declared
on Book's _authorId_ column. A _SELECT * FROM_ statement joining the two tables by the author's ID will produce a row for
each book the author has written. This means that if you have 3 authors and each one has authored 3 books,
the query will return 9 rows:

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=result-1.txt"></script>

An attempt at formulating a query that sorts the result by author name and fetches
the result's first page, were the page can have at most 2 authors, might go like this:

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=limit-offset.sql"></script>

Perhaps to one's surprise, the above _SELECT_ statement, with a _LIMIT_ of 2 and an _OFFSET_ of 0, will only return books
authored by Antonopoulos:

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=result-2.txt"></script>

The books authored by Chomsky weren't included in the result because it's the many-side
of the relationship (i.e., books) which is paginated as opposed to the one-side (i.e., authors).
Paging the one-side of a one-to-many relationship isn't a trivial problem to solve in pure SQL
without re-writing part of the query as a sub-select:

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=sub-select.sql"></script>

In practice, I find it inconvenient to formulate a query in this way in order to accommodate
pagination. It doesn't easily lend itself to runtime string manipulation so the
developer is forced to think about pagination every time he writes a query.

Most popular vendor DBMSs offer window functions for performing calculations over ranges of rows. One such
function is *DENSE_RANK*. This function ranks each row against the rest of the rows
to produce a sequence of numbers starting from 1. Rows equal in rank have the
same sequence number. *DENSE_RANK* is an alternative solution for the one-to-many
pagination problem as it helps us simulate logical counts and offsets based on the author's primary key column:

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=dense-rank-offset.sql"></script>

This _SELECT_ statement uses *DENSE_RANK* combined with _ORDER BY_ to produce rows such that
row _x_ and _y_ have the same rank if and only if row _x_'s author ID and name are equal
to row _y_'s author's ID and name:

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=result-3.txt"></script>

With the *offset_* rank column, we can scroll forwards or backwards taking into account
that duplicate authors in the result have the same rank. Say we want to get records
 starting from offset 1. Keeping in mind _rank = offset + 1_, this requirement would be simply expressed as:

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=dense-rank-offset-1.sql"></script>

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=result-4.txt"></script>

To add a logical count, another rank, representing the count, is re-computed over the sub-select's result:

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=dense-rank-offset-1-count.sql"></script>

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=result-5.txt"></script>

In the beginning of this post, we wanted to fetch the first page of the result
with a limit of 2 authors. Here's how to achieve this with *DENSE_RANK*:

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=dense-rank-offset-0-count-2.sql"></script>

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=result-6.txt"></script>

Note how *DENSE_RANK* allows us to more or less **wrap around** the main query rather than changing it.
We can even take this one step further:

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=dense-rank-offset-0-count-2-wrapped.sql"></script>

By moving the computation of the offset rank outside of the main query, we have
the possibility to easily write string manipulation code in the application that
dynamically, and transparently, applies pagination to queries.

<div style="text-align: justify; line-height: 1.3;">
  <span style="font-family: Times, Times New Roman, serif; font-size: small;">
    <span class="num">1: All SQL examples target PostgreSQL.</span>
  </span>
  <br/>
  <span style="font-family: Times, Times New Roman, serif; font-size: small;">
    2: The described solution won't solve the one-to-many paging problem should the result be ordered by a column in the Book table. I'll leave it as an exercise for the reader to figure out why that's the case.
  </span>
</div>
