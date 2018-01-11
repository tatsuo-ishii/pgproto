# What is Pgproto?

Pgproto is a tool to test [PostgreSQL](http://www.postgresql.org) or
any other servers that understand the frontend/backend
protocol. Pgproto reads a text file describing message data to be sent
to PostgreSQL. Also Pgproto can read responses from PostgreSQL if
there's a request command in the file. See README.format for more
details.

All the data exchanged between Pgproto and PostgreSQL are printed to
stderr in the trace format similar to the PostgreSQL JDBC driver.

The motivation behind Pgproto is, to test
[Pgpool-II](http://pgpoo.net) which is proxy server between clients
and PostgreSQL by sending arbitrary combinations of message, which is
very hard to implement using existing PostgreSQL clients API,
especially when dealing with extended protocol.

# Example 

In the example below, the first character in the file (i.e. 'Q')
indicates the message kind specified in the PostgreSQL
frontend/backend protocol. Exceptions are 'Y' and 'y'. 'Y' reads
messages from backend until 'Ready for query' is received. 'y' reads
messages from backend while messages are coming from backend then
stops if messages are not available for 1 second. 'Y' is used for
waiting for reply of 'Q' (simple query) or after 'S' (sync) in
extended queries. 'y' can be used for receiving messages after 'H'
(flush).

Here is an example input file:

~~~~
#
# Test data example
#
'Q'	"SELECT * FROM aaa"
'Y'
'P'	"S1"	"BEGIN"	0
'B'	""	"S1"	0	0	0
'E'	""	0
'C'	'S'	"S1"
'P'	"foo"	"SELECT 1"	0
'B'	"myportal"	"foo"	0	0	0
'E'	"myportal"	0
'P'	"S2"	"COMMIT"	0
'B'	""	"S2"	0	0	0
'E'	""	0
'C'	'S'	"S2"
'S'
'Y'
'X'
~~~~

Here is an example trace data:

~~~~
FE=> Query(query="SELECT * FROM aaa")
<= BE ErrorResponse(S ERROR V ERROR C 42P01 M relation "aaa" does not exist P 15 F parse_relation.c L 1159 R parserOpenTable )
<= BE ReadyForQuery(I)
FE=> Parse(stmt="S1", query="BEGIN")
FE=> Bind(stmt="S1", portal="")
FE=> Execute(portal="")
FE=> Close(stmt="S1")
FE=> Parse(stmt="foo", query="SELECT 1")
FE=> Bind(stmt="foo", portal="myportal")
FE=> Execute(portal="myportal")
FE=> Parse(stmt="S2", query="COMMIT")
FE=> Bind(stmt="S2", portal="")
FE=> Execute(portal="")
FE=> Close(stmt="S2")
FE=> Sync
<= BE ParseComplete
<= BE BindComplete
<= BE CommandComplete(BEGIN)
<= BE CloseComplete
<= BE ParseComplete
<= BE BindComplete
<= BE DataRow
<= BE CommandComplete(SELECT 1)
<= BE ParseComplete
<= BE BindComplete
<= BE CommandComplete(COMMIT)
<= BE CloseComplete
<= BE ReadyForQuery(I)
FE=> Terminate
~~~~

# Installation

C compiler, PostgreSQL client library (libq) is required (the version
does not matter).

~~~~
./configure
make
make install
~~~~

# Usage

~~~~
Usage: pgproto
-h, --hostname=HOSTNAME (default: UNIX domain socket)
-p, --port=PORT (default: 5432)
-u, --user USERNAME (default: OS user)
-d, --database DATABASENAME (default: same as user)
-f, --proto-data-file FILENAME (default: pgproto.data)
-r, --read-nap NAPTIME (in micro seconds. default: 0)
-D, --debug
-?, --help
-v, --version
~~~~

* Setting "read-nap" for greater than 0 will let pgproto sleep between each data reading from socket. This is useful to simulate slow clients.

# Restrictions

* You need to fully understand the frontend/backend protocol of PostgreSQL.

* Pgproto only supports V3 protocol.

* Many message types are not supported yet (patches are welcome!)

* Pgproto does not support SSL connections.

# History

* 2018/1/11: V1.2 release.
  * Add support for 'read-nap' option.
* 2017/10/10: V1.1 release.
  * Add support for 'y' message.
* 2017/08/02: v1.0 release.