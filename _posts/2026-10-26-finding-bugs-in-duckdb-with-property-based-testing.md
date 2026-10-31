---
layout: post
title: "Finding bugs in DuckDB with property-based testing"
date: 2026-10-26
---

DuckDB is a widely used database and a personal favorite. I recently found some bugs in DuckDB using property-based testing and wanted to share how the bugs were found (all these bugs were reported and promptly fixed).  

In total, I found 5 bugs, four of which were related to the `COPY ... TO` command and the remaining one was a different issue. Let’s look at these one by one.

DuckDB has a `COPY ... TO` command which can be used to save data to a file. For example, its template looks like this:
```sql
COPY (SELECT 1 AS a) TO 'out.json' (..., ...);
-- (..., ...) placeholder for the options the command accepts
```

It copies what the select query returns and writes to a json file using the options provided within the right most parenthesis. As for these options in the right most parenthesis, as one example, one could run this command:
```sql
COPY (SELECT 1 AS a) TO 'out.json' (FORMAT JSON, use_tmp_file);
```

The `FORMAT JSON` option lists the file format we want, and `use_tmp_file`, a boolean option, enables writing the data to a temporary file. Once writing finishes, that file is renamed to `out.json`.

One of the first bugs we found came from a `COPY ... TO` command containing `use_tmp_file NULL`. When the property-based test ran it, DuckDB failed with the error:
```
INTERNAL Error: Calling GetValueInternal on a value that is NULL
```

How did we write that property-based test? I have been using Hypothesis, a Python based property-testing framework, for the last couple of years and I find it very useful for not only testing systems with combinatorially many random inputs but also letting the code express all this magic in a less obscure way.

The idea is that you write invariants that must hold and then you try all kinds of inputs on your code and ensure that invariants hold. There were two main components of the code. The first component acted as a generator, and the second as a manager reporting on the success or failure of the operation being run. The generator produced a lot of `COPY ... TO` commands with different kinds of inputs plugged into the command’s options, and the manager ran these against DuckDB and reported success or failure. Invalid inputs should produce a normal error, but one caused an internal error instead.

Just to give an idea, here is the simplified code that could generate such data:

```python 
@composite
def copy_statement(draw):
    opt = draw(sampled_from(["use_tmp_file", "compression", "dateformat"]))
    val = draw(sampled_from(["NULL", "''", "42", "[1,2,3]", "TRUE"]))  # odd on purpose
    return f"COPY (SELECT 1 AS a) TO 'out.json' (FORMAT JSON, {opt} {val})"
```

And it would generate statements like:
```sql
COPY (SELECT 1 AS a) TO 'out.json' (FORMAT JSON, compression 42);
COPY (SELECT 1 AS a) TO 'out.json' (FORMAT JSON, dateformat [1,2,3]);
COPY (SELECT 1 AS a) TO 'out.json' (FORMAT JSON, use_tmp_file NULL);
```

The `composite` decorator in Hypothesis lets us write custom generators. The `sampled_from` strategy selects a value from a given list, and `draw` gives us a value from that strategy. This is how we produce many different `COPY ... TO` commands.

As you can see, the generator produced a command containing `use_tmp_file NULL`. When the manager ran it, DuckDB raised an internal error. I then manually reproduced it and reported it in DuckDB’s issue tracker ([#22866](https://github.com/duckdb/duckdb/issues/22866)). The same property testing code found three more bugs with the `COPY ... TO` command ([#22867](https://github.com/duckdb/duckdb/issues/22867), [#22868](https://github.com/duckdb/duckdb/issues/22868), [#22869](https://github.com/duckdb/duckdb/issues/22869)). These were promptly fixed by the DuckDB team.

The second class of bug we found was an interesting one. The idea of this test was simple. Generate a float and write it into an INSERT as a numeric literal, insert it into a DOUBLE column, read it back, and validate that the number we get back is the same number we started with. In one of the cases it was not. We inserted _0.41000000000000003_ and got back _0.40999999999999998_. This was not correct. 

For this bug, generator code that we used, in its simplified form, looked like this:

```python
@composite
def doubles(draw):
    return draw(lists(one_of(
        floats(allow_nan=False, allow_infinity=False),
        sampled_from([0.1, 0.41000000000000003, 0.9999999999999999]),  # hard cases
    )))
```

The code generates lists of Python floats (double-precision numbers) to feed into DuckDB: 
```
[]                                              
[0.1]                                          
[0.41000000000000003]                          
[0.41000000000000003, 0.0]                      
[0.1, 6.820642817941818e+16]                    
[0.9999999999999999, -3.782657869065369e+16]
```

The `floats` strategy basically generates finite double-precision numbers. The `sampled_from` strategy selects from our list of values, and `one_of` allows each element to come from either strategy. 

The code would then use these values, write them to the database and read them back and compare to determine success or failure. Success, if they are the same; failure, if not. 

For example, in the case of _0.41000000000000003_ it would issue SQL like this and compare and read values. 
```sql
CREATE TABLE t (x DOUBLE);
INSERT INTO t VALUES (0.41000000000000003);   
SELECT x FROM t;                                     
```

We inserted _0.41000000000000003_, but read back _0.40999999999999998_. These are two different DOUBLE values. This was not a bug in storage per se. DuckDB interpreted the numeric literal as a DECIMAL, then rounded incorrectly when converting it to DOUBLE. The same digits when used in the query as text, and converted directly to DOUBLE with CAST, would give the correctly rounded result:

```sql
SELECT
    -- this outputs the wrong value: 0.40999999999999998
    printf('%.17g', 0.41000000000000003),
    -- this outputs the correct value: 0.41000000000000003
    printf('%.17g', CAST('0.41000000000000003' AS DOUBLE));
```

This was found on DuckDB 1.5.3 and has since been fixed ([#22837](https://github.com/duckdb/duckdb/issues/22837)). 


DuckDB's testing coverage is wide. It uses fuzzing and checks invariants too. Even with all this testing, property-based tests in Hypothesis helped and found a few issues. 

I've been helping a number of teams use property-based testing lately, and it's been an effective way not only to raise coverage but also to make code more robust. With some guidance, AI can also help make progress towards this goal. If you're considering property-based testing for your team, I'd be happy to share what I've learned. 









