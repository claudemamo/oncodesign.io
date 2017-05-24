+++
date = "2017-04-22T09:33:52+02:00"
title = "Paging One-to-Many Results in SQL"
tags = ["Pagination", "PostgreSQL", "SQL", "DENSE_RANK"]
+++

Pagination in databases, the division of query results into manageable subsets, is often
implemented in terms of SQL constructs like _LIMIT_ or _FETCH FIRST_, and _OFFSET_. This
works well when the result is compromised of one-to-one relationships. On the other hand,
hypothesize what happens if the relationship is a one-to-many as in the following DDL [1]:

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=schema.sql"></script>

An author can publish _n_ books and this relationship is expressed as a foreign key constraint declared
on Book's _authorId_ column. A _SELECT * FROM_ statement joining the two tables by the author's ID will produce a row for
each book the author has written. This means that if you have 3 authors and each author has written 3 books,
the query will return 9 rows:

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=result-1.txt"></script>

The first attempt at formulating a query that fetches the first page of the result, ordered by author
name and were a page can have at most 2 authors, might go like this:

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=limit-offset.sql"></script>

The above _SELECT_ statement, with a _LIMIT_ of 3 and an _OFFSET_ of 0, will only return books
authored by Antonopoulos as opposed to books authored by Chomsky in addition to Antonopoulos
because authors are duplicated in the result set:

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=result-2.txt"></script>

Paging a one-to-many relationship isn't a trivial problem to solve in pure SQL.
In fact, it might be tempting to page the result from within the application to achieve
the desired outcome but such a solution would arguably suffer for large result sets given
that all rows in the result need to be brought over from the database to the application.

Most popular vendor DBMSs offer window functions for performing calculations over ranges of rows. One such
function is *DENSE_RANK*. This function ranks each row against the rest of the rows
to produce a sequence of numbers starting from 1. Rows which are equal in rank have
the same sequence number. *DENSE_RANK* is the key to solving the one-to-many pagination
problem as it helps us simulate logical counts and offsets based on the author's primary key column:

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=dense-rank-offset.sql"></script>

This _SELECT_ statement uses *DENSE_RANK* combined with _ORDER BY_ to produce rows such that
row _x_ and _y_ have the same rank if and only if row _x_'s author ID and name are equal
to row _y_'s author's ID and name:

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=result-3.txt"></script>

With the *offset_* rank column, we can scroll forwards or backwards taking into account
that duplicate authors in the result have the same rank. Say we want to get records
 starting from offset 1. Keeping in mind that _rank = offset + 1_, this requirement would be simply expressed as:

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=dense-rank-offset-1.sql"></script>

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=result-4.txt"></script>

To add a logical count, another rank, representing the count, is re-computed over the sub-select's result:

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=dense-rank-offset-1-count.sql"></script>

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=result-5.txt"></script>

In the beginning of this post, we wanted to fetch the first page of the result
with a limit of 2 authors. Here's how to achieve this with DENSE_RANK:

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=dense-rank-offset-0-count-2.sql"></script>

<script src="https://gist.github.com/claudemamo/0ba4ad21df38dacee9d64258c0166da4.js?file=result-6.txt"></script>

<div style="text-align: justify; line-height: 1.3;">
  <span style="font-family: Times, Times New Roman, serif; font-size: small;">
    <span class="num">1: All SQL examples target PostgreSQL.</span>
  </span>
  <br/>
  <span style="font-family: Times, Times New Roman, serif; font-size: small;">
    2: The described solution won't solve the one-to-many paging problem should the result be ordered by a column in the Book table. I'll leave it as an exercise for the reader to figure out why that's the case.
  </span>
</div>
