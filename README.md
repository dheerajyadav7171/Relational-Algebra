# Relational-Algebra

A small interactive interpreter for **relational algebra** queries over CSV tables.
Type an algebra expression at the prompt and it prints the resulting table.

## Build & run

```sh
g++ -std=c++17 -O2 -o ra ra.cpp
./ra
```

At the `>> ` prompt, enter a query and press Enter. Type `exit` to quit.

The interpreter reads its data from the CSV files in this directory. The available
base tables and their columns are:

| Table        | Columns                                              |
|--------------|------------------------------------------------------|
| `STUDENT`    | ROLLNO, FIRSTNAME, LASTNAME, DEPARTMENT, PROGRAMME, SEX |
| `COURSE`     | COURSEID, DEPARTMENT, CREDITS, ADVISOR               |
| `ENROLLMENT` | COURSEID, ROLLNO                                     |
| `PROFESSOR`  | EMPID, FIRSTNAME, LASTNAME, DEPARTMENT, SEX, YEAR    |
| `DEPARTMENT` | DEPTID, DEPTNAME, HOD                                |

---

## Query syntax

Every operation has the shape:

```
LETTER[ argument ]( input_relation )
```

- **`LETTER`** — a single uppercase letter naming the operation.
- **`[ ... ]`** — the argument (columns, predicate, or new name).
- **`( ... )`** — the relation it operates on, which is itself another operation
  or, at the bottom, a base table.

Every expression must bottom out at one of the five base tables. You can also type
a base table name on its own (e.g. `STUDENT`) to dump the whole table.

### Operations

| Op | Form | Meaning |
|----|------|---------|
| Project           | `P[col, col, ...](rel)` | Keep the listed columns. `P[*](rel)` keeps all. |
| Select            | `S[predicate](rel)`     | Keep rows matching the predicate. |
| Rename            | `R[newname](rel)` or `R[newname(c1, c2, ...)](rel)` | Rename the table and/or all columns. |
| Union             | `U[A](B)` | A ∪ B |
| Intersection      | `I[A](B)` | A ∩ B |
| Set difference    | `D[A](B)` | A − B |
| Cartesian product | `C[A](B)` | A × B (A's columns first) |
| Exit              | `exit`    | Quit the interpreter. |

### Examples (all verified against the bundled data)

Project:
```
P[rollno, firstname, department](STUDENT)
```

Select — numeric, string, AND (`^`), OR (`|`):
```
S[credits > '11'](COURSE)
S[department = 'CSE'](PROFESSOR)
S[department = 'CSE' ^ sex = 'M'](PROFESSOR)
S[department = 'CSE' | department = 'CIV'](PROFESSOR)
```

Rename the table, or the table and all columns:
```
R[TEACHERS](P[empid, department](PROFESSOR))
R[DEPT(id, name, head)](DEPARTMENT)
```

Union / Intersection / Set difference (operands must be union-compatible):
```
U[S[department = 'CSE'](PROFESSOR)](S[department = 'PHY'](PROFESSOR))
I[P[department](STUDENT)](P[department](PROFESSOR))
D[P[department](PROFESSOR)](P[department](STUDENT))
```

Cartesian product (rename the two inputs so they have different names):
```
P[deptname, empid](
  S[deptid = department](
    C[R[d](DEPARTMENT)](R[p](P[empid, department](PROFESSOR)))))
```

### Select predicates

A predicate is a mini expression language. Operators, from tightest-binding to
loosest:

```
* / %        arithmetic (multiply, divide, modulo)
+ -          arithmetic (add, subtract)
> >= < <=    comparison
= !=         comparison
^            logical AND
|            logical OR
```

Operands are **column names** or **quoted constants**. Both sides of a comparison
may be columns (`c1.credits < c2.credits`) or arithmetic (`credits + '1' > '13'`).
When both sides of `>`/`<`/etc. are numeric, they are compared as numbers;
otherwise as strings.

---

## Rules that keep queries working

1. **Input is upper-cased.** `sex = 'F'` and `department = 'CSE'` work because that
   data is stored uppercase, but `firstname = 'CARL'` will never match the stored
   value `Carl`. Only already-uppercase data (dept codes, IDs, `M`/`F`) can be matched.
2. **Constants must be single-quoted** — strings *and* numbers: `credits > '11'`,
   not `credits > 11` (an unquoted `11` is read as a column name).
3. **A column must exist at that point in the pipeline.** After `P[...]` drops
   columns or `R[...]` renames them, later operations must use the surviving names.
4. **Cartesian needs distinct table names**, and shared column names get prefixed
   with the table name. After `C[R[p](PROFESSOR)](R[c](COURSE))`, the common
   `DEPARTMENT` column is referred to as `p.department` and `c.department`.
5. **Union / Intersection / Set difference need matching column lists** (same names,
   same order). Usually you `P[...]` both sides to the same columns first, and
   `R[(...)]` to align names.
6. **Renaming columns is all-or-nothing** — supply a new name for every column, or none.
7. **Keep `[]` and `()` balanced and in their roles** — `[]` wraps the argument,
   `()` wraps the input relation. Unbalanced brackets are the main way to actually
   crash the parser rather than get a clean error.

A cleanly-broken rule (bad column, incompatible tables, wrong rename count, unknown
table) prints an `ERROR : ...` message and returns to the prompt — it does not crash.

---

## How the code works

Everything revolves around one structure:

```cpp
class relation {
    string name;                 // display name
    set<vector<string>> records; // rows — a set, so duplicates vanish & rows stay sorted
    vector<string> att_list;     // column names, in order
    map<string, int> att_map;    // column name -> its index in each row
};
```

A `relation` is an in-memory table. Storing rows in a `set` gives relational
set-semantics for free: automatic deduplication and sorted output.

The program is a REPL (`main`): read a line, upper-case it, evaluate, print, repeat
until `exit`. Evaluating one query flows through three layers:

1. **`process(query)`** — strips the `[...]` arguments to expose the skeleton, digs
   down to the innermost `(...)` to find the base table name, validates it against the
   five allowed tables, constructs a `Table` (which loads the CSV), and hands off to
   `Table::parse`. It is also the entry point for the right-hand operand of every
   binary operation, which it evaluates recursively.

2. **`Table::parse(query)`** — a nested query like `R[..](P[..](S[..](...)))` is a
   chain of operators wrapping the base table. `parse` peels operators off the outside
   one at a time, pushing each `(letter, arg)` onto a stack, until what remains is the
   table name. It then pops the stack (reversing the order) so operators apply
   inside-out, dispatching each letter to its function:

   ```
   case 'P': out = project(arg, out);
   case 'S': out = select(arg, out);
   case 'C': out = cartesian(process(arg), out);   // binary -> recurse on arg
   ...
   ```

   `out` starts as the base table and is transformed step by step.

3. **The operator functions** — `project`, `select`, `rename` (unary) and `Union`,
   `set_diff`, `intersect`, `cartesian` (binary). Each returns a new `relation`.
   `intersect` is composed from `set_diff` and `Union`.

Cross-cutting details:

- **`success` (global flag):** any operator hitting an error prints `ERROR : ...`
  and sets `success = false`; every function early-returns while it is false, so a
  single error unwinds the whole pipeline and `main` skips printing.
- **Helpers:** `split` (comma parsing), `strip`, `remove_brackets`, `to_int` (which
  also serves as the "is this numeric?" test), and the overloaded `operator<<` that
  draws the ASCII result table.

### Select in detail

`select` keeps rows matching a predicate, and works in two phases.

**Phase 1 — `postfix()`** converts the predicate to postfix (Reverse Polish
Notation) using the shunting-yard algorithm and the `precedence` map, merging
two-character operators (`>`+`=` → `>=`). For example
`department = 'CSE' ^ sex = 'M'` becomes `department 'CSE' = sex 'M' = ^`.

**Phase 2 — evaluation** walks the postfix with a stack. The key idea is that
values are handled *by column, for the whole table at once*, not row-by-row. Each
stack entry is a `pair<relation, vector<string>>`: the relation whose rows are being
filtered, plus a column of values (one per row). A column name expands to that
column's values; a constant expands to the same value repeated for every row.

Operators branch by precedence tier:

- **Arithmetic** (`+ - * / %`): combine two value-columns element-wise (numeric).
- **Comparisons** (`= != > >= < <=`): keep the rows where the comparison holds,
  producing a *filtered relation*. Numeric columns compare as numbers, else as strings.
- **Boolean combiners** (`^`, `|`): pop two filtered relations and combine their
  row-sets — `^` is set intersection, `|` is set union.

When the postfix is consumed, the single relation left on the stack is the result.

## Files

- `ra.cpp` — the interpreter.
- `readme.txt` — the in-app quick syntax reference.
- `queries.txt` — five worked example queries with descriptions.
- `*.csv` — the data tables.
