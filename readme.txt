Query:

Select            - S[predicate](relation)
Project           - P[column list](relation)  columns in column list are seperated by comma
Rename            - R[new relation name(new column names)](old relation name)
Union             - U[relation_1](relation_2)
Intersection      - I[relation_1](relation_2)
Set Difference    - D[relation_1](relation_2)  relation_1 - relation_2
Cartesian Product - C[relation_1](relation_2)
Exit              - exit

Note:

-> In predicate, constants (either string or integer) must be enclosed by ' '

-> Queries are case insensitive (both table names and column names)

-> Use * in project query to project all the columns of a table

-> Table and its columns can be renamed independently. But to rename a column, all the columns must be
   renamed. i.e., no.of new column names = total no.of columns in the table, or 0