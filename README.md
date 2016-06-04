# Stepan's PostgreSQL Patches

## numeric in() operator-patch

This patch improves just the specific usecase when you have an in() operator
with a very long number of integers. This scenario came up in application
code I was involved in where such long in() list were created. This is in
most cases not the approach you want to take, but it happens. Improving on
the database side is attractive, since application code can remain as is.

Eg you have a query like this:

select * from sometable where id in (100,200,300,302,78,3,......) 

and the number of items in the in() list is more than 50 you can end up in
a scenario where during result-set scaning each record's id has to be compared
to each item in the in()-list sequentially. That costs occurs for every row
in your query unless discarded before.

In PostgreSQL there is workaround like using values(), temp-tables, real-tables
etc. All these are propably much better than this approach here. However in my
case this approach does give me a huge speedup (eg 3 times) and such querries
are easily build. Patch was developed against PostgreSQL 9.4

## Better error reporting when there is a varchar2 overflow

Imagine you have a table like this

create table insanetable (
	c1 varchar(32),
	c2 varchar(32),
	c3 varchar(32),
	... (many more the above with the same length)

If you insert something that is too large on a single column you end up with
errors that look like 
  
  ERROR:  value too long for type character varying(32)

This is not helping me much. The patch will turn this too

  ERROR:  value too long for type character varying(32) (hint: column-name is c2)

if the column that was overflown was c2.


I fired up gdb and saw that the error message is generated during the 
preprocessing of the query where some kind of the 
constant-folding/constant-elimination happens on the parse-tree. 
I went ahead and added a try/catch at some point upwards in the 
call-stack where at least i have the contact of the T_TargetEntry. 
That has a field resname which gives me exactly the information i need... 
The column which was overflown. With that info i can fix the application code 
much more easily. Relation name was out of reach for me, there is a void* 
passed transparently to the constant-mutator but that is not checkable at 
the point. That context contains the original top-level statement node however.

The patch has some issues, though it works for me in production just fine...
See 

https://www.postgresql.org/message-id/trinity-53b335a3-11e3-43df-b820-e28935108e6b-1438771166514@3capp-gmx-bs49

and reply.



