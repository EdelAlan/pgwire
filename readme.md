# PGWIRE

PostgreSQL client library for Node.js

[![npm](https://img.shields.io/npm/v/pgwire.svg)](https://www.npmjs.com/package/pgwire) [![travis](https://travis-ci.com/kagis/pgwire.svg?branch=master)](https://travis-ci.com/kagis/pgwire)

Under development

## Features

- Memory efficient data streaming
- Logical replication using pgoutput protocol
- Multi-statement queries
- Efficient bytea transfering
- Copy from stdin and to stdout
- SCRAM-SHA-256 authentication method support
- [Pure js without dependencies](package.json#L33)

# Connecting to PostgreSQL server

```js
import pgwire from 'pgwire';
const client = await pgwire.connect(
  'postgres://USER:PASSWORD@HOST:PORT/DATABASE?application_name=my-app'
);
```

https://www.postgresql.org/docs/11/libpq-connect.html#id-1.7.3.8.3.6

Good practice is to get connection URI from environment variable:

```js
// myapp.js
const client = await pgwire.connect(process.env.POSTGRES);
```

Set `POSTGRES` environment variable when run node process:

```bash
$ POSTGRES=postgres://USER:PASSWORD@HOST:PORT/DATABASE node myapp.js
```

`pgwire.connect()` function also accepts parameters as object:

```js
const client = await pgwire.connect({
  user: 'postgres',
  password: 'postgres',
  host: '127.0.0.1',
  port: 5432,
  database: 'postgres',
});
```

Its possible to pass multiple connection URIs or objects to `pgwire.connect()` function. In this case actual connection parameters will be computed by merging all parameters in first-win priority. Following technique can be used to set default connection parameters:

```js
const client = await pgwire.connect(process.env.POSTGRES, {
  application_name: 'my-awesome-app',
});
```

Don't forget to `.end()` connection when you don't need it anymore:

```js
const client = await pgwire.connect(process.env.POSTGRES);
try {
  // use client
} finally {
  client.end();
}
```

# Querying

```js
const { rows } = await client.query({
  statement: `
    SELECT i, 'Mississippi'
    FROM generate_series(1, 3) i
  `,
});

console.assert(JSON.stringify(rows) == JSON.stringify([
  [1, 'Mississippi'],
  [2, 'Mississippi'],
  [3, 'Mississippi'],
]));
```

Function call and other single-value query result can be accessed by `scalar` property. It equals to the first cell of the first row. If query returns no rows then `scalar` equals to `undefined`.

```js
const { scalar } = await client.query({
  statement: `SELECT current_user`,
});

console.assert(scalar == 'postgres');
```

# Parametrized query

```js
const { rows } = await client.query({
  statement: `
    SELECT $1
    FROM generate_series(1, $2)
  `,
  params: [{
    type: 'text',
    value: 'hello',
  }, {
    type: 'int4',
    value: 3,
  }],
});

console.assert(JSON.stringify(rows) == JSON.stringify([
  ['hello'],
  ['hello'],
  ['hello'],
]));
```

# Multi-statement queries

Its possible to execute multiple statements within a single query.

```js
const {
  results: [
    { rows: categories },
    { rows: products },
  ],
} = await client.query({
  statement: `
    SELECT id, name
    FROM category
  `,
}, {
  statement: `
    SELECT id, category_id, name
    FROM product
  `,
});
```

Benefit of multi-statement queries is that statements are wrapped into transaction implicitly. Implicit transaction does rollback automatically when error occures or does commit when all statements successfully executed. Multi-statement queries and implicit transactions are described here https://www.postgresql.org/docs/11/protocol-flow.html#PROTOCOL-FLOW-MULTI-STATEMENT


Response top level `rows` and `scalar` properties take their values from last statement result:

```js
const { scalar, rows } = await client.query({
  statement: `SELECT 'a'`,
}, {
  statement: `SELECT 'b', 1`,
});

console.assert(scalar == 'b');
console.assert(
  JSON.stringify(rows) ==
  JSON.stringify([['b', 1]])
);
```

# Reading from stream

`.query()` response can be consumed in two ways. First way is to `await` response like a promise (or call `response.then` explicitly). In this case all rows will be loaded in memory.

Other way to consume response is to use it as Readable stream.

```js
// omit `await` keyword to get underlying response stream
const response = client.query({
  statement: `SELECT i FROM generate_series(1, 2000) i`,
});
let sum = 0;
for await (const chunk of response) {
  if (chunk.boundary) {
    // This chunks are useful to determine
    // multi-statement results boundaries.
    // But we don't need it now so
    continue;
  }
  const [i] = chunk;
  sum += i;
}
// prints sum of natural numbers from 1 to 2000
console.log(sum);
```

# Copy from stdin

```js
await client.query({
  statement: `COPY foo FROM STDIN`,
  stdin: fs.createReadableStream('/tmp/file'),
});
```

# Copy to stdout

`COPY TO STDOUT` statement acts like `SELECT` but fills `rows` with `Buffer` instances instead of row tuples.

```js
const { rows } = await client.query({
  statement: `
    COPY (
      SELECT i, 'Mississippi'
      FROM generate_series(1, 3) i
    ) TO STDOUT
  `,
});

const tsv = Buffer.concat(rows).toString();
console.assert(tsv == (
  '1\tMississippi\n' +
  '2\tMississippi\n' +
  '3\tMississippi\n'
));
```

Dump table data to TSV file:

```js
import { pipeline } from 'stream';
import { promisify } from 'util';
const ppipeline = promisify(pipeline);

const copyUpstream = client.query({
  statement: `COPY upstream_table TO STDOUT`,
});
await ppipeline(
  copyUpstream,
  fs.createWritableStream('/tmp/file.tsv'),
);
```

Copy table from one database to another:

```js
const copyUpstream = clientA.query({
  statement: `COPY upstream_table TO STDOUT`,
});
await clientB.query({
  statement: `COPY downstream_table FROM STDIN`,
  stdin: copyUpstream,
});
```

Response stream emits boundary chunks at start and end of stream, but this chunks are ignored by piped writable stream since boundary chunks inherited from empty `Buffer`.

# Listen and Notify

https://www.postgresql.org/docs/11/sql-notify.html

```js
client.on('notification', ({ channel, payload }) => {
  console.log(channel, payload);
});
await client.query(`LISTEN some_channel`);
```

# Extended query protocol

Postgres protocol has two subprotocols for querying - simple and extended. Simple protocol allows to send multi-statement query as single script where statements are delimited by semicolon. **pgwire** uses simple protocol when `.query()` is called with a string in first argument:

```js
await client.query(`
  CREATE TABLE foo (a int, b text);
  INSERT INTO foo VALUES (1, 'hello');
  COPY foo FROM STDIN;
  SELECT * FROM foo;
`, {
  // optional stdin for COPY FROM STDIN statements
  stdin: fs.createReadableStream('/tmp/file1.tsv'),
  // stdin also accepts array of streams for multiple
  // COPY FROM STDIN statements
  stdin: [
    fs.createReadableStream(...),
    fs.createReadableStream(...),
    ...
  ],
});
```

Extended query protocol allows to pass parameters for each statement so it splits statements into separate chunks. Its possible to use extended query protocol by passing one or more statement objects to `.query()` function:

```js
await client.query({
  statement: `CREATE TABLE foo (a int, b text)`,
}, {
  statement: `INSERT INTO foo VALUES ($1, $2)`,
  params: [{
    type: 'int4',
    value: 1,
  }, {
    type: 'text',
    value: 'hello',
  }],
}, {
  statement: 'COPY foo FROM STDIN',
  stdin: fs.createReadableStream('/tmp/file1.tsv'),
}, {
  statement: 'SELECT * FROM foo',
});
```

Low level extended protocol messages:

```js
await client.query({
  // creates prepared statement
  op: 'parse',
  // sql statement body
  statement: `SELECT $1, $2`,
  // optional statement name
  // if not specified then unnamed statement will be created
  statementName: 'example-statement',
  // optional parameters types hints
  // if not specified then parameter types will be inferred
  paramTypes: [
    // can use builtin postgres type names
    'int4',
    // also can use any type oid directly
    25,
  ],
}, {
  // creates portal (cursor) from prepared statement
  op: 'bind',
  // optional statement name
  // if not specified then last unnamed prepared statement
  // will be used
  statementName: 'example-statement',
  // optional portal name
  // if not specified then unnamed portal will be created
  portal: 'example-portal',
  // optional parameters list
  params: [{
    // optional builtin parameter type name
    // if not specified then value should be
    // string or Buffer representing raw text or binary
    // encoded value
    type: 'int4',
    // $1 parameter value
    value: 1,
  }, {
    // also can use builtin type oid
    type: 25,
    // $2 parameter value
    value: 'hello',
  }],
}, {
  // executes portal
  op: 'execute',
  // optional portal name
  // if not specified then last unnamed portal will be executed
  portal: 'example-portal',
  // optional row limit value
  // if not specified then all rows will be returned
  // if limit is exceeded then result will have `suspeded: true` flag
  limit: 10,
  // optional source stream for `COPY target FROM STDIN` queries
  stdin: fs.createReadStream(...),
});
```

# Connection pool

```js
const client = pgwire.pool(
  'postgres://USER:PASSWORD@HOST:PORT/DATABASE?poolMaxConnections=4&idleTimeout=900000'
);
```

When `poolMaxConnections` is not set then a new connection is created for each query. This option makes possible to switch to external connection pool like pgBouncer.

# Logical replication

Logical replication is native PostgreSQL mechanism which allows your app to subscribe on data modification events such as insert, update, delete and truncate. This mechanism can be useful in different ways - replicas synchronization, cache invalidation or history tracking. https://www.postgresql.org/docs/11/logical-replication.html

Lets prepare database for logical replication

```sql
CREATE TABLE foo(a INT NOT NULL PRIMARY KEY, b TEXT);
```

Replication slot is PostgreSQL entity which behaves like a message queue of replication events. Lets create replication slot for our app.

```sql
SELECT pg_create_logical_replication_slot(
  'my-app-slot',
  'test_decoding' -- logical decoder plugin
);
```

Generate some modification events:

```sql
INSERT INTO foo VALUES (1, 'hello'), (2, 'world');
UPDATE foo SET b = 'all' WHERE a = 1;
DELETE FROM foo WHERE a = 2;
TRUNCATE foo;
```

Now we are ready to consume replication events.

```js
import pgwire from 'pgwire';
const client = await pgwire.connect(process.env.POSTGRES, {
  replication: 'database',
});

const replicationStream = await client.logicalReplication({
  slot: 'my-app-slot',
  startLsn: '0/0',
});

for await (const { lsn, data } of replicationStream) {
  console.log(data.toString());
  replicationStream.ack(lsn);
}
```

# "pgoutput" logical replication decoder

Modification events go through plugable logical decoder at first before events emitted to client. Purpose of logical decoder is to serialize in-memory events structures into consumable message stream. PostgreSQL has two built-in logical decoders:

- `test_decoding` which emits human readable messages for debugging and testing,

- and production usable `pgoutput` logical decoder which adapts modification events for replicas synchronization. **pgwire** implements `pgoutput` messages parser

```sql
CREATE PUBLICATION "my-app-pub" FOR TABLE foo
```

```sql
SELECT pg_create_logical_replication_slot(
  'my-app-slot',
  'pgoutput' -- logical decoder plugin
)
```

```js
import pgwire from 'pgwire';
const client = await pgwire.connect(process.env.POSTGRES, {
  replication: 'database',
});

const replicationEvents = await client.logicalReplication({
  slot: 'my-app-slot',
  startLsn: '0/0',
  options: {
    'proto_version': 1,
    'publication_names': 'my-app-pub',
  },
});

const pgoEvents = replicationEvents.pgoutput();

for await (const logicalEvent of pgoEvents) {
  switch (logicalEvent.tag) {
    case 'begin':
    case 'relation':
    case 'type':
    case 'origin':
    case 'insert':
    case 'update':
    case 'delete':
    case 'truncate':
    case 'commit':
  }
}
```
