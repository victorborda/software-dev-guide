# POSTGRES DON'T DO LIST

Borrowed from the excellent: <https://wiki.postgresql.org/wiki/Don't_Do_This>

A short list of common mistakes.

Kristian Dupont provides [schemalint](https://github.com/kristiandupont/schemalint), a tool to verify the database schema against these recommendations.

## Database Encoding

### Don't use SQL_ASCII

#### Why not?
SQL_ASCII means "no conversions" for the purpose of all encoding conversion functions. That is to say, the original bytes are simply treated as being in the new encoding, subject to validity checks, without any regard for what they mean. Unless extreme care is taken, an SQL_ASCII database will usually end up storing a mixture of many different encodings with no way to recover the original characters reliably.

#### When should you?

If your input data is already in a hopeless mixture of unlabelled encodings, such as IRC channel logs or
non-MIME-compliant emails, then SQL_ASCII might be useful as a last resort—but consider using bytea first
instead, or whether you could autodetect UTF8 and assume non-UTF8 data is in some specific encoding such as WIN1252.

## Tool usage

### Don't use psql -W or --password

Don't use psql -W or psql --password

#### Why not?

Using the --password or -W flags will tell [psql](https://www.postgresql.org/docs/current/static/reference-client.html)
to prompt you for a password, before trying to connect to the server - so you'll be prompted for a password even if the
server doesn't require one.

It's never required, as if the server does require a password psql will prompt you for one, and it can be very confusing
when setting up permissions. If you're connecting with -W to a server configured to allow you access via peer
authentication you may think that it's requiring a password when it really isn't. And if the user you're logging in as
doesn't have a password set or you enter the wrong password at the prompt you'll still be logged in and think you have
the right password - but you won't be able to log in from other clients (that connect via localhost) or when logged in
as other users.

#### When should you?

Never, pretty much. It will save a round trip to the server but that's about it.

## Rules

### Don't Use Rules

Don't use [rules](https://www.postgresql.org/docs/current/static/sql-createrule.html). If you think you want to, use a
[trigger](https://www.postgresql.org/docs/current/static/plpgsql-trigger.html) instead.

#### Why not?

Rules are incredibly powerful, but they don't do what they look like they do. They look like they're some conditional
logic, but they actually rewrite a query to modify it or add additional queries to it.

That means that [all non-trivial rules are incorrect](http://blog.rhodiumtoad.org.uk/2010/06/21/the-rule-challenge/).

Depesz has [more to say](https://www.depesz.com/2010/06/15/to-rule-or-not-to-rule-that-is-the-question/) about them.

#### When should you?

Never. While the rewriter is an implementation detail of VIEWs, there is no reason to pry up this cover plate directly.

## Table Inheritance

### Don't use table inheritance

Don't use [table inheritance](https://www.postgresql.org/docs/current/tutorial-inheritance.html). If you think you want
to, use foreign keys instead.

#### Why not?

Table inheritance was a part of a fad wherein the database was closely coupled to object-oriented code. It turned out
that coupling things that closely didn't actually produce the desired results.

#### When should you?

Never, almost. Now that table partitioning is done natively, that common use case for table inheritance has been
replaced by a native feature that handles tuple routing, etc., without bespoke code.

One of the very few exceptions would be [temporal_tables](https://github.com/arkhipov/temporal_tables) extension if you
are in a pinch and want to use that for row versioning in place of a lacking SQL 2011 support. Table inheritance will
provide a small shortcut instead of using `UNION ALL` to get both historical as well as current rows.
Even then you ought to be wary of
[caveats](http://clarkdave.net/2015/02/historical-records-with-postgresql-and-temporal-tables-and-sql-2011/#using-inheritance) while working with parent table.

## SQL constructs

### Don't use NOT IN

Don't use `NOT IN`, or any combination of `NOT` and `IN` such as
`NOT (x IN (select…))`.

#### Why not?

Two reasons:

1. `NOT IN` behaves in unexpected ways if there is a null present:

`select * from foo where col not in (1,null); -- always returns 0 rows`

`select * from foo where foo.col not in (select bar.x from bar); -- returns 0 rows if any value of bar.x is null`

This happens because `col IN (1,null)` returns `TRUE` if col=1, and `NULL` otherwise
(i.e. it can never return `FALSE`). Since `NOT (TRUE)` is `FALSE`, but
`NOT (NULL)` is still `NULL`, there is no way that `NOT (col IN (1,null))` (which is
the same thing as `col NOT IN (1,null)`) can return `TRUE` under any circumstances.

2. Because of point 1 above, `NOT IN (SELECT ...)` does not optimize very well. In particular, the planner
can't transform it into an anti-join, and so it becomes either a hashed Subplan or a plain Subplan. The hashed subplan
is fast, but the planner only allows that plan for small result sets; the plain subplan is '''horrifically''' slow (in
fact O(N²)). This means that the performance can look good in small-scale tests but then slow down by 5 or more orders
of magnitude once a size threshold is crossed; you '''do not''' want this to happen.

''Alternative solution:'' In most cases, the NULL behavior of `NOT IN (SELECT …)` is not intentionally
desired, and the query can be rewritten using `'''NOT EXISTS''' (SELECT …)`:

`select * from foo where '''not exists''' (select from bar where foo.col = bar.x);`

#### When should you?

`NOT IN (<i>list,of,values,...</i>)` is mostly safe ''unless'' you might have a null in the list (via a
parameter or otherwise). So it's sometimes natural and even advisable to use it when excluding specific constant values
from a query result.

### Don't use upper case table or column names

Don't use NamesLikeThis, use names_like_this.

#### Why not?

PostgreSQL folds all names - of tables, columns, functions and everything else - to lower case unless they're "double
quoted".

So <tt>create table Foo()</tt> will create a table called <tt>foo</tt>, while <tt>create table "Bar"()</tt> will create
a table called <tt>Bar</tt>.

These select commands will work: <tt>select *from Foo</tt>, <tt>select* from foo</tt>, <tt>select * from "Bar"</tt>.

These will fail with "no such table": <tt>select *from "Foo"</tt>, <tt>select* from Bar</tt>, <tt>select * from
    bar</tt>.

This means that if you use uppercase characters in your table or column names you have to either ''always'' double quote
them or ''never'' double quote them. That's annoying enough by hand, but when you start using other tools to access the
database, some of which always quote all names and some don't, it gets very confusing.

Stick to using a-z, 0-9 and underscore for names and you never have to worry about quoting them.

#### When should you?

If it's important that "pretty" names are displaying in report output then you might want to use them. But you can also
use column aliases to use lower case names in a table and still get pretty names in the output of a query: <tt>select
    character_name as "Character Name" from foo</tt>.

### Don't use BETWEEN (especially with timestamps)

#### Why not?

`BETWEEN` uses a closed-interval comparison: the values of both ends of the specified range are included in
the result.

This is a particular problem with queries of the form

`SELECT * FROM blah WHERE timestampcol BETWEEN '2018-06-01' AND '2018-06-08'`

This will include results where the timestamp is ''exactly'' 2018-06-08 00:00:00.000000, but not timestamps later in
that same day. So the query might seem to work, but as soon as you get an entry exactly on midnight, you'll end up
double-counting it.

Instead, do:

`SELECT * FROM blah WHERE timestampcol >= '2018-06-01' AND timestampcol < '2018-06-08'`

#### When should you?

`BETWEEN` is safe for discrete quantities like integers or dates, as long as you remember that both ends of
the range are included in the result. But it's a bad habit to get into.

## Date/Time storage

### Don't use timestamp (without time zone)

Don't use the <tt>timestamp</tt> type to store timestamps, use <tt>timestamptz</tt> (also known as <tt>timestamp with
    time zone</tt>) instead.

#### Why not?

<tt>timestamptz</tt> records a single moment in time. Despite what the name says it doesn't store a timestamp, just a
point in time described as the number of microseconds since January 1st, 2000 in UTC. You can insert values in any
timezone and it'll store the point in time that value describes. By default it will display times in your current
timezone, but you can use <tt>at time zone</tt> to display it in other time zones.

Because it stores a point in time it will do the right thing with arithmetic involving timestamps entered in different
timezones - including between timestamps from the same location on different sides of a daylight savings time change.

<tt>timestamp</tt> (also known as <tt>timestamp without time zone</tt>) doesn't do any of that, it just stores a date
and time you give it. You can think of it being a picture of a calendar and a clock rather than a point in time. Without
additional information - the timezone - you don't know what time it records. Because of that, arithmetic between
timestamps from different locations or between timestamps from summer and winter may give the wrong answer.

So if what you want to store is a point in time, rather than a picture of a clock, use timestamptz.

[More about timestamptz](https://community.spiceworks.com/topic/2454825-zone-of-misunderstanding).

#### When should you?

If you're dealing with timestamps in an abstract way, or just saving and retrieving them from an app, where you aren't
going to be doing arithmetic with them then timestamp might be suitable.

### Don't use timestamp (without time zone) to store UTC times

Storing UTC values in a `timestamp without time zone` column is, unfortunately, a practice commonly inherited
from other databases that lack usable timezone support.

Use `timestamp with time zone` instead.

#### Why not?

Because there is no way for the database to know that UTC is the intended timezone for the column values.

This complicates many otherwise useful time calculations. For example, "last midnight in the timezone given by
u.timezone" becomes this:

`date_trunc('day', now() AT TIME ZONE u.timezone) AT TIME ZONE u.timezone AT TIME ZONE 'UTC'`

And "the midnight prior to `x.datecol` in u.timezone" becomes this:

`date_trunc('day', x.datecol AT TIME ZONE 'UTC' AT TIME ZONE u.timezone)
   AT TIME ZONE u.timezone AT TIME ZONE 'UTC'`

#### When should you?

If compatibility with non-timezone-supporting databases trumps all other considerations.

### Don't use timetz

Don't use the <tt>timetz</tt> type. You probably want <tt>timestamptz</tt> instead.

#### Why not?

Even the manual tells you it's only implemented for SQL compliance.

>The type time with time zone is defined by the SQL standard, but the definition exhibits properties which
    lead to questionable usefulness. In most cases, a combination of date, time, timestamp without time zone, and
    timestamp with time zone should provide a complete range of date/time functionality required by any application.

#### When should you?

Never.

### Don't use CURRENT_TIME

Don't use the `CURRENT_TIME` function. Use whichever of these is appropriate:

* `CURRENT_TIMESTAMP` or `now()` if you want a `timestamp with time zone`,
* `LOCALTIMESTAMP` if you want a `timestamp without time zone`,
* `CURRENT_DATE` if you want a `date`,
* `LOCALTIME` if you want a `time`

#### Why not?

It returns a value of type `timetz`, for which see the previous entry.

#### When should you?

Never.

### Don't use timestamp(0) or timestamptz(0)

Don't use a precision specification, especially not 0, for timestamp columns or casts to timestamp.

Use `date_trunc('second', blah)` instead.

#### Why not?

Because it rounds off the fractional part rather than truncating it as everyone would expect. This can cause unexpected
issues; consider that when you store `now()` into such a column, you might be storing a value half a second
in the future.

#### When should you?

Never.

## Text storage

### Don't use char(n)

Don't use the type <tt>char(n)</tt>. You probably want <tt>text</tt>.

#### Why not?

Any string you insert into a `char(n)` field will be padded with spaces to the declared width. That's
probably not what you actually want.

The manual says:

<blockquote>Values of type character are physically padded with spaces to the specified width n, and are stored and
    displayed that way. However, trailing spaces are treated as semantically insignificant and disregarded when
    comparing two values of type character. In collations where whitespace is significant, this behavior can produce
    unexpected results; for example `SELECT 'a '::CHAR(2) collate "C" < E'a\n'::CHAR(2)` returns true, even
    though C locale would consider a space to be greater than a newline. Trailing spaces are removed when converting a
    character value to one of the other string types. Note that trailing spaces are semantically significant in
    character varying and text values, and when using pattern matching, that is LIKE and regular expressions.
</blockquote>

That should scare you off it.

The space-padding does waste space, but doesn't make operations on it any faster; in fact the reverse, thanks to the
need to strip spaces in many contexts.

It's important to note that from a storage point of view `char(n)` '''is not a fixed-width type'''. The
actual number of bytes varies since characters may take more than one byte, and the stored values are therefore treated
as variable-length anyway (even though the space padding is included in the storage).

#### When should you?

When you're porting very, very old software that uses fixed width fields. Or when you read the snippet from the manual
above and think "yes, that makes perfect sense and is a good match for my requirements" rather than gibbering and
running away.

### Don't use char(n) even for fixed-length identifiers

Sometimes people respond to "don't use `char(n)`" with "but my values must always be exactly N characters
long" (e.g. country codes, hashes, or identifiers from some other system). '''It is still a bad idea to use
`char(n)` even in these cases.'''

Use `text`, or a domain over text, with `CHECK(length(VALUE)=3)` or
`<nowiki>CHECK(VALUE ~ '^[[:alpha:]]{3}$')</nowiki>` or similar.

#### Why not?

Because `char(n)` doesn't reject values that are too short, it just silently pads them with spaces. So
there's no actual benefit over using `text` with a constraint that checks for the exact length. As a bonus,
such a check can also verify that the value is in the correct format.

Remember, '''there is no performance benefit whatsoever to using `char(n)` over `varchar(n)`.'''
In fact the reverse is true. One particular problem that comes up is that if you try and compare a `char(n)`
field against a parameter where the driver has explicitly specified a type of `text` or `varchar`,
you may be unexpectedly unable to use an index for the comparison. This can be hard to debug since it doesn't show up on
manual queries.

#### When should you?

Never.

### Don't use varchar(n) by default

Don't use the type <tt>varchar(n)</tt> by default. Consider <tt>varchar</tt> (without the length limit) or <tt>text</tt>
instead.

#### Why not?

<tt>varchar(n)</tt> is a variable width text field that will throw an error if you try and insert a string longer than n
characters (not bytes) into it.

<tt>varchar</tt> (without the <tt>(n)</tt>) or <tt>text</tt> are similar, but without the length limit. If you insert
the same string into the three field types they will take up exactly the same amount of space, and you won't be able to
measure any difference in performance.

If what you really need is a text field with an length limit then varchar(n) is great, but if you pick an arbitrary
length and choose varchar(20) for a surname field you're risking production errors in the future when Hubert Blaine
Wolfe­schlegel­stein­hausen­berger­dorff signs up for your service.

Some databases don't have a type that can hold arbitrary long text, or if they do it's not as convenient or efficient or
well-supported as varchar(n). Users from those databases will often use something like <tt>varchar(255)</tt> when what
they really want is <tt>text</tt>.

If you need to constrain the value in a field you probably need something more specific than a maximum length - maybe a
minimum length too, or a limited set of characters - and a
[check constraint](https://www.postgresql.org/docs/current/static/ddl-constraints.html#DDL-CONSTRAINTS-CHECK-CONSTRAINTS)
can do all of those things as well as a maximum string length.

#### When should you?

When you want to, really. If what you want is a text field that will throw an error if you insert too long a string into
it, and you don't want to use an explicit check constraint then varchar(n) is a perfectly good type. Just don't use it
automatically without thinking about it.

Also, the varchar type is in the SQL standard, unlike the text type, so it might be the best choice for writing
super-portable applications.

## Other data types

### Don't use money

The <tt>money</tt> data type isn't actually very good for storing monetary values. Numeric, or (rarely) integer may be
better.

#### Why not?

[lots of reasons.](https://www.postgresql.org/message-id/flat/20130328092819.237c0106@imp#20130328092819.237c0106@imp)

It's a fixed-point type, implemented as a machine int, so arithmetic with it is fast. But it doesn't handle fractions of
a cent (or equivalents in other currencies), it's rounding behaviour is probably not what you want.

It doesn't store a currency with the value, rather assuming that all money columns contain the currency specified by the
database's [lc_monetary](https://www.postgresql.org/docs/current/static/runtime-config-client.html#GUC-LC-MONETARY)
locale setting. If you change the lc_monetary setting for any reason, all money columns will contain the wrong value.
That means that if you insert '$10.00' while lc_monetary is set to 'en_US.UTF-8' the value you retrieve may be '10,00
Lei' or '¥1,000' if lc_monetary is changed.

Storing a value as a numeric, possibly with the currency being used in an adjacent column, might be better.

#### When should you?

If you're only working in a single currency, aren't dealing with fractional cents and are only doing addition and
subtraction then money might be the right thing.

### Don't use serial

For new applications, identity columns should be used instead.

#### Why not?

The serial types have some [weird behaviors](https://www.2ndquadrant.com/en/blog/postgresql-10-identity-columns/) that
make schema, dependency, and permission management unnecessarily cumbersome.

#### When should you?

* If you need support to PostgreSQL older than version 10.
* In certain combinations with table inheritance (but see there)
* More generally, if you somehow use the same sequence for multiple tables, although in those cases an explicit
declaration might be preferable over the serial types.

## Authentication

### Don't use <tt>trust</tt> authentication over TCP/IP (<tt>host</tt>, <tt>hostssl</tt>)

Don't use <tt>trust</tt> authentication over any TCP/IP method (e.g. host, hostssl) in any production environment.

Especially '''do not''' set a line like this in your <tt>pg_hba.conf</tt> file:

`host    all   all   0.0.0.0/0   trust`

which allows anyone on the Internet to authenticate as any PostgreSQL user in your cluster, including the PostgreSQL
superuser.

There is a [list of authentication methods](https://www.postgresql.org/docs/current/client-authentication.html) you can
choose that are better for establishing a remote connection to PostgreSQL. It is fairly easy to set up a
[password](https://www.postgresql.org/docs/current/auth-password.html) based authentication method, the recommendation
being <tt>scram-sha-256</tt> that is available in PostgreSQL 10 and above.

#### Why not?

The [manual](https://www.postgresql.org/docs/current/auth-trust.html) says:

>trust authentication is only suitable for TCP/IP connections if you trust every user on every
    machine that is allowed to connect to the server by the <tt>pg_hba.conf</tt> lines that specify <tt>trust</tt>. It
    is seldom reasonable to use trust for any TCP/IP connections other than those from localhost (127.0.0.1).

With <tt>trust</tt> authentication, any user can claim to be any other user and PostgreSQL will trust that assertion.
This means that someone can claim to be the <tt>postgres</tt> superuser account and PostgreSQL will accept that claim
and allow them to log in.

To take this a step further, it is also not a good idea to allow <tt>trust</tt> authentication to be used on
<tt>local</tt> UNIX socket connections in a production environment, as anyone with access to the instance running
PostgreSQL could log in as any user.

#### When should you?

The short answer is '''never'''.

The longer answer is there are a few scenarios where <tt>trust</tt> authentication may be appropriate:

* Running tests against a PostgreSQL server as part of a CI/CD job that is on a trusted network
* Working on your local development machine, but only allowing TCP/IP connections over localhost but you should see if any of the alternative methods work better for you. For example, on UNIX-based systems, you can connect to your local development environment using <tt>peer</tt> authentication.
