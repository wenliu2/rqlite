# Restoring directly from SQLite
> :warning: **This page is no longer maintained. Visit [rqlite.io](https://www.rqlite.io) for the latest docs.**

rqlite supports loading a node directly from two sources, either of which can be used to initialize your system from preexisting SQLite data, or to restore from an existing [node backup](https://github.com/rqlite/rqlite/blob/master/DOC/BACKUPS.md):
- An actual SQLite database file. This is the fastest way to initialize a rqlite node from an existing SQLite database. Even large SQLite databases can be loaded into rqlite in a matter of seconds. This the recommended way to initialize your rqlite node from existing SQLite data. In addition any preexisting SQLite database is completely overwritten by this type of load operation, so it's safe to perform regardless of any data already loaded into your rqlite cluster. Finally, this type of load request can be sent to any node. The receiving node will transparently forward the request to the Leader as needed, and return the response of the Leader to the client. If you would prefer to be explicitly redirected to the Leader, add `redirect` as a URL query parameter.
- SQLite dump in text format. This is another convenient manner to initialize a system from an existing SQLite database (or other database). In constrast to loading an actual SQLite file, the behavior of this type of load operation is **undefined** if there is already data loaded into your rqlite cluster.  **Note that if your source database is large, the operation can be quite slow.** If you find the restore times to be too long, you should first load the SQL statements directly into a SQLite database, and then restore rqlite using the resulting SQLite database file.

## Examples
The following examples show a trivial database being generated by `sqlite3`, the SQLite file being backed up, converted to the corresponding list of SQL commands, and then loaded into a rqlite node listening on localhost using each form.

### HTTP
 _Be sure to set the Content-type header as shown, depending on the format of the upload._

```bash
~ $ sqlite3 restore.sqlite
SQLite version 3.14.1 2016-08-11 18:53:32
Enter ".help" for usage hints.
sqlite> CREATE TABLE foo (id integer not null primary key, name text);
sqlite> INSERT INTO "foo" VALUES(1,'fiona');
sqlite>
# Load directly from the SQLite file, which is the recommended process.
~ $ curl -v -XPOST localhost:4001/db/load -H "Content-type: application/octet-stream" --data-binary @restore.sqlite

# Convert SQLite database file to set of SQL statements and then load
~ $ echo '.dump' | sqlite3 restore.sqlite > restore.dump
~ $ curl -XPOST localhost:4001/db/load -H "Content-type: text/plain" --data-binary @restore.dump
```

After either command, we can connect to the node, and check that the data has been loaded correctly.
```bash
$ rqlite
127.0.0.1:4001> SELECT * FROM foo
+----+-------+
| id | name  |
+----+-------+
| 1  | fiona |
+----+-------+
```

### rqlite CLI
The CLI supports loading from a SQLite database file or SQL text file. The CLI will automatically detect the type of data being used for the restore operation. Below shows an example of loading from the former.
```
~ $ sqlite3 mydb.sqlite
SQLite version 3.22.0 2018-01-22 18:45:57
Enter ".help" for usage hints.
sqlite> CREATE TABLE foo (id integer not null primary key, name text);
sqlite> INSERT INTO "foo" VALUES(1,'fiona');
sqlite> .exit
~ $ ./rqlite 
Welcome to the rqlite CLI. Enter ".help" for usage hints.
127.0.0.1:4001> .schema
+-----+
| sql |
+-----+
127.0.0.1:4001> .restore mydb.sqlite
database restored successfully
127.0.0.1:4001> select * from foo
+----+-------+
| id | name  |
+----+-------+
| 1  | fiona |
+----+-------+
```

## Caveats
Note that SQLite dump files normally contain a command to disable Foreign Key constraints. If you are running with Foreign Key Constraints enabled, and wish to re-enable this, this is the one time you should explicitly re-enable those constraints via the following `curl` command:
```bash
curl -XPOST 'localhost:4001/db/execute?pretty' -H "Content-Type: application/json" -d '[
    "PRAGMA foreign_keys = 1"
]'
```
