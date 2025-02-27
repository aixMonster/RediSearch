Returns the execution plan for a complex query.

In the returned response, a `+` on a term is an indication of stemming.

#### Parameters

- **index**: The index name. The index must be first created with FT.CREATE
- **query**: The query string, as if sent to FT.SEARCH
- **DIALECT {dialect_version}**. Choose the dialect version to execute the query under. If not specified, the query will execute under the default dialect version set during module initial loading or via `FT.CONFIG SET` command.

{{% alert title="raw mode" color="info" %}}
You should use `redis-cli --raw` to properly read line-breaks in the returned response.
{{% /alert %}}

@return

String Response. A string representing the execution plan (see above example).

@examples

```sh
$ redis-cli --raw

127.0.0.1:6379> FT.EXPLAIN rd "(foo bar)|(hello world) @date:[100 200]|@date:[500 +inf]"
INTERSECT {
  UNION {
    INTERSECT {
      foo
      bar
    }
    INTERSECT {
      hello
      world
    }
  }
  UNION {
    NUMERIC {100.000000 <= x <= 200.000000}
    NUMERIC {500.000000 <= x <= inf}
  }
}
```