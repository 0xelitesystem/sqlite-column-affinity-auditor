# sqlite-column-affinity-auditor

Paste CREATE TABLE DDL from any dialect and see the type affinity SQLite actually assigns each column, which literals silently change storage class on insert, whether a STRICT table would accept the declared type, and which columns are true rowid aliases.

**Live demo:** https://0xelitesystem.github.io/sqlite-column-affinity-auditor/

## Use

Paste one or more CREATE TABLE statements into the left box. Put probe literals, one per line, in the right box. Four panels render as you type.

1. **Affinity verdict.** One row per parsed column: the declared type verbatim, which of the five ordered rules decided the affinity, the exact substring that matched (underlined inside the declared type), and the resulting affinity. Rows where the rule order or a buried substring produced the answer are flagged SURPRISING, in words and with a glyph, never by colour alone.
2. **Insert simulator.** Every probe literal against every column of an ordinary table, showing the storage class SQLite would end up with and the value it would hold. A STRICT table is left out of this panel, with a note saying so, because it refuses a value its column type cannot hold instead of converting it, and its ANY columns keep every value exactly as received.
3. **STRICT verdict.** Whether a STRICT table would accept each declared type, with the engine's own rejection wording.
4. **Rowid alias.** A separate verdict on every PRIMARY KEY column, because affinity alone does not predict this one.

The page opens prefilled with a schema that trips two of the five rules, rule 1 for `POINT` and rule 5 for `STRING`, `BOOLEAN`, `DATETIME` and `DECIMAL(10,2)`, so the verdict table is on screen before you type anything. "Reset to example" puts it back.

## The five rules, quoted

These are quoted verbatim from section 3.1, "Determination Of Column Affinity", of the SQLite datatypes document at https://www.sqlite.org/datatype3.html, fetched while this page was written.

> If the declared type contains the string "INT" then it is assigned INTEGER affinity.
>
> If the declared type of the column contains any of the strings "CHAR", "CLOB", or "TEXT" then that column has TEXT affinity. Notice that the type VARCHAR contains the string "CHAR" and is thus assigned TEXT affinity.
>
> If the declared type for a column contains the string "BLOB" or if no type is specified then the column has affinity BLOB.
>
> If the declared type for a column contains any of the strings "REAL", "FLOA", or "DOUB" then the column has REAL affinity.
>
> Otherwise, the affinity is NUMERIC.

The same document adds why the order is the whole point:

> Note that the order of the rules for determining column affinity is important. A column whose declared type is "CHARINT" will match both rules 1 and 2 but the first rule takes precedence and so the column affinity will be INTEGER.

> Note that a declared type of "FLOATING POINT" would give INTEGER affinity, not REAL affinity, due to the "INT" at the end of "POINT". And the declared type of "STRING" has an affinity of NUMERIC, not TEXT.

The page implements this as a literal if / else-if chain in that order, never as a lookup map, because a map cannot express precedence.

## There is no PRAGMA that reports affinity

This is the strongest reason the page exists. `PRAGMA table_info` does not report, confirm or verify affinity. It hands back the declared type string exactly as you wrote it. Run against the prefilled example on SQLite 3.50.4:

```
sqlite> CREATE TABLE t(a STRING, b POINT, c BOOLEAN, d DATETIME, e DECIMAL(10,2));
sqlite> PRAGMA table_info(t);
(0, 'a', 'STRING',        0, None, 0)
(1, 'b', 'POINT',         0, None, 0)
(2, 'c', 'BOOLEAN',       0, None, 0)
(3, 'd', 'DATETIME',      0, None, 0)
(4, 'e', 'DECIMAL(10,2)', 0, None, 0)
```

The columns it returns are `cid, name, type, notnull, dflt_value, pk`. Affinity is not among them. There is no PRAGMA, no `pragma_` table-valued function and no SQL command in SQLite that prints a column's affinity.

### How to verify any of this yourself

Create the table, insert a probe literal, and ask `typeof()`. Never ask `table_info`.

```
sqlite> INSERT INTO t VALUES('0123','1','1','1','1');
sqlite> SELECT a, typeof(a), typeof(b), typeof(c), typeof(d), typeof(e) FROM t;
123|integer|integer|integer|integer|integer
```

The column declared STRING came back as the integer 123. The leading zero is gone. That is the whole problem in one row.

The catch with `typeof()` is that it needs a row to already exist, so it tells you what was stored, not what the schema you are still writing is about to do. That is the gap this page fills.

## INTEGER and NUMERIC affinity store values identically

Do not let anyone tell you POINT (rule 1, INTEGER affinity) and BOOLEAN (rule 5, NUMERIC affinity) store values differently. They do not. Same row, same literals, SQLite 3.50.4, three columns declared POINT, BOOLEAN and INTEGER:

```
literal                 POINT              BOOLEAN            INTEGER
'1'                     integer 1          integer 1          integer 1
'1.0'                   integer 1          integer 1          integer 1
'1.5'                   real 1.5           real 1.5           real 1.5
'0123'                  integer 123        integer 123        integer 123
'12e2'                  integer 1200       integer 1200       integer 1200
'3.0e+5'                integer 300000     integer 300000     integer 300000
'abc'                   text 'abc'         text 'abc'         text 'abc'
'9223372036854775808'   real 9.22337203685477581e+18 (all three)
```

The affinity names differ. The storage behaviour does not. The only divergence that reproduces is inside a CAST expression:

```
CAST('1.5' AS POINT)    -> integer 1
CAST('1.5' AS BOOLEAN)  -> real 1.5
CAST(1.9 AS INTEGER)    -> integer 1
CAST(1.9 AS NUMERIC)    -> real 1.9
```

So the simulator renders an INTEGER-affinity column and a NUMERIC-affinity column with the same result and different rule numbers, and carries that sentence as persistent body text next to the pair. What the simulator legitimately distinguishes is TEXT and BLOB affinity from the numeric family, not INTEGER from NUMERIC.

One formatting note so nothing here reads as a mismatch: the engine's `quote()` prints that overflowing literal as `9.22337203685477581e+18` and the page prints `9.223372036854776e+18`. Both are the same IEEE 754 binary64 value. The page shows the shortest decimal that round-trips to it.

## STRICT accepts six names, and matches them exactly

Measured, not read off a page and not fetched at build time. Attempting `CREATE TABLE s(x TYPE) STRICT` on SQLite 3.50.4 across 17 candidate type names accepted exactly six:

```
INT  INTEGER  REAL  TEXT  BLOB  ANY
```

and rejected NUMERIC, VARCHAR(20), BOOLEAN, DATETIME, FLOAT, DECIMAL(10,2), STRING, POINT, BIGINT, DOUBLE and CHAR(4), each with the same shape of error:

```
unknown datatype for s.x: "NUMERIC"
```

Those six are a hardcoded array in the page. There is no network call behind them.

**The inversion worth knowing.** STRICT matches the whole type name exactly, not by substring. BIGINT gets INTEGER affinity in an ordinary table, because rule 1 only needs the letters "INT" to appear somewhere. A STRICT table still rejects BIGINT outright. A type can behave like an integer everywhere except in a STRICT table. The same goes for `INTEGER(4)`: the parenthesised argument alone is enough to be rejected.

STRICT tables were added in SQLite 3.37.0, released 2021-11-27, as stated at https://www.sqlite.org/stricttables.html.

## Rowid alias is a separate verdict for a reason

Three columns, all with INTEGER affinity. One of them rejects a string. Two accept it. SQLite 3.50.4:

```
CREATE TABLE r(k INTEGER PRIMARY KEY, v TEXT)
   (rowid, k, typeof k) = (5, 5, 'integer')
   INSERT INTO r VALUES('abc','z')  ->  REJECTED, "datatype mismatch"

CREATE TABLE r(k INT PRIMARY KEY, v TEXT)
   (rowid, k, typeof k) = (1, 5, 'integer')
   INSERT INTO r VALUES('abc','z')  ->  accepted, k stored as text 'abc'

CREATE TABLE r(k POINT PRIMARY KEY, v TEXT)
   (rowid, k, typeof k) = (1, 5, 'integer')
   INSERT INTO r VALUES('abc','z')  ->  accepted, k stored as text 'abc'
```

The same trap fires a second way, on the sort order rather than the type name. On SQLite 3.50.4:

```
CREATE TABLE r(k INTEGER PRIMARY KEY DESC, v TEXT)
   INSERT INTO r VALUES('abc','z')  ->  accepted, (rowid, k, typeof k) = (1, 'abc', 'text')
   AUTOINCREMENT on the same column  ->  refused,
        "AUTOINCREMENT is only allowed on an INTEGER PRIMARY KEY"

CREATE TABLE r(k INTEGER PRIMARY KEY ASC, v TEXT)
   INSERT INTO r VALUES('abc','z')  ->  REJECTED, "datatype mismatch"

CREATE TABLE r(k INTEGER, v TEXT, PRIMARY KEY(k DESC))
   INSERT INTO r VALUES('abc','z')  ->  REJECTED, "datatype mismatch"
```

Written on the column definition, DESC takes the alias away. Written in a table level
PRIMARY KEY clause, the identical DESC does not. Panel 4 reports both cases the way the
engine behaves, and the shipped self test carries a fixture for each of them.

Affinity does not predict this, which is why panel 4 is visually and semantically separate from panel 1. The test the page applies: the declared type, trimmed, with one enclosing pair of quotes removed the way SQLite removes them, and uppercased, is exactly `INTEGER`, the column carries PRIMARY KEY on its own rather than as part of a composite key, the table is not WITHOUT ROWID, and the column definition does not end its PRIMARY KEY clause with `DESC`.

## Who this is actually for

Say the deflating thing first: **a hand-written SQLite schema that declares only INTEGER, TEXT and REAL will show no surprising rows at all.** Those names, along with BLOB and NUMERIC, are exactly the ones this page never flags. Paste this and panel 1 flags nothing:

```
CREATE TABLE note(id INTEGER PRIMARY KEY, title TEXT, body TEXT, weight REAL, raw BLOB, n NUMERIC);
```

This tool earns its place on DDL imported, generated or emitted by an ORM from MySQL or Postgres, which is precisely the moment somebody pastes a schema they did not write. Paste the same table as `mysqldump` writes it and four of the six columns are flagged, because `boolean`, `datetime`, `decimal(10,2)` and `enum('a','b')` are not SQLite type names and fall through to NUMERIC, while `int(11)` and `varchar(255)` are honest and are not flagged:

```
CREATE TABLE `u` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `email` varchar(255) NOT NULL,
  `active` boolean DEFAULT NULL,
  `created` datetime DEFAULT NULL,
  `price` decimal(10,2) DEFAULT NULL,
  `kind` enum('a','b') DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_email` (`email`)
) ENGINE=InnoDB;
```

Both of those are fixtures in the shipped self test, which asserts zero flagged columns for the first and four for the second, so you can recompute that pairing in the browser instead of taking it on trust. On the engine itself, SQLite 3.50.4 stores `'0123'` into those `boolean`, `datetime` and `decimal(10,2)` columns as the integer 123, and into `varchar(255)` as the text `'0123'`.

An earlier draft of this section published column counts and percentages from two corpus scans of the machine this was built on. They are gone. Nothing in this repo can recompute them, so you had no way to check them, and a receipt you cannot check is not a receipt. Paste your own schema instead: the page answers for your schema, which is the only corpus that matters to you.

## How correct this is, and how that was checked

- **The shipped self test.** Click "Run self test". It runs 576 assertions covering affinity, the insert simulator, the STRICT six, the SURPRISING flag, the parser, the rowid verdict and the payload-sizing claim above. 575 pass. The one failure is a positive control, an assertion written to fail on purpose, so a self test that silently stops running before the end is detectable. A run reporting zero failures is a broken run. The page prints that count itself, so it is the one number here you do not have to take from me.
- **Every fixture is an engine measurement, and you can re-measure it.** The expected values are literal arrays in `index.html` named `AFFINITY_FIXTURES`, `SIMULATOR_FIXTURES` and `STRICT_FIXTURES`. Each row is a value read back out of SQLite 3.50.4, not a value derived from the page's own logic. Lift any row, run the `typeof()` recipe above against it, and the engine answers what the fixture says. That includes the awkward ones: `'1.'`, `'.5'`, `'1e400'`, `'0x1A'`, `'9223372036854775807'`, `'9223372036854775808'` and a leading non-breaking space.
- **The palette.** Every text token was checked against every background token by arithmetic in both themes, 48 pairs, whether or not any page state currently paints that pair. All pass WCAG AA for normal text. The worst pair is 5.46:1.

## Why this exists

Type affinity is the single most surprising thing about SQLite for anyone arriving from a rigidly typed engine, and nothing in SQLite reports it directly. There is no PRAGMA for it. `typeof()` only speaks after a row exists. So the usual outcome is that a schema ported from MySQL quietly rewrites `'0123'` into the integer 123 in production, months later, in a column somebody declared STRING.

Single file, no build step, no network call, no dependency to rot. The five rules are a ported formula from a published specification, not a per-vendor capability matrix, so this page does not go stale.

It also belongs to a set. `json-precision-auditor` and `yaml-type-coercion-auditor` do the same job one layer up: paste a document, see which values silently change type or value. This is that same class of failure at the SQL storage layer.

## Privacy

Everything runs in your browser. Nothing you paste leaves your machine. One self-contained HTML file with inline CSS and inline JavaScript, no CDN, no framework, no web font, no analytics, no network call of any kind, and no browser storage. Open it from `file://` with the network cable pulled and it behaves identically.

## Run locally

```
git clone https://github.com/0xelitesystem/sqlite-column-affinity-auditor.git
cd sqlite-column-affinity-auditor
```

Then open `index.html` in a browser. That is the whole thing. If you would rather serve it:

```
python -m http.server 8000
```

and open http://localhost:8000/.

## Build

There is no build. One file, no dependencies, no package manager, no bundler, no step. Edit `index.html` and reload.

## Third-party notices

**SQLite documentation**, the datatypes specification at https://sqlite.org/datatype3.html and the STRICT tables page at https://sqlite.org/stricttables.html.

Licence status, copied verbatim from https://sqlite.org/copyright.html: "All of the code and documentation in SQLite has been dedicated to the public domain by the authors." and "Anyone is free to copy, modify, publish, use, compile, sell, or distribute the original SQLite code, either in source code form or as a compiled binary, for any purpose, commercial or non-commercial, and by any means."

What was taken: the five affinity rules, quoted word for word in the section above, and the list of six type names that STRICT accepts. Nothing else. No SQLite code is reproduced.

Independent and unaffiliated. This page is not affiliated with, endorsed by, or connected to SQLite or the SQLite Consortium. SQLite is named here only as the subject being analysed.

The five affinity rules are quoted above from the published SQLite datatypes specification. The behaviour was reimplemented from that specification and then verified against SQLite 3.50.4. No SQLite code is reproduced here. For anyone reading the engine instead: the function that does this work is `sqlite3AffinityType()`, in `src/build.c` of the SQLite source tree.

Everything measured here used SQLite 3.50.4.

## License

MIT. Copyright (c) 2026 0xelitesystem. See `LICENSE`.

## Related

- https://github.com/0xelitesystem/json-precision-auditor
- https://github.com/0xelitesystem/yaml-type-coercion-auditor
- https://github.com/0xelitesystem/jwt-inspector
