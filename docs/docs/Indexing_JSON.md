---
title: Index and search JSON documents
linkTitle: Index JSON
weight: 2
description: How to index and search JSON documents
---

In addition to indexing Redis hashes, RediSearch can also index JSON documents. To index and search JSON, you need to install both the RediSearch and [RedisJSON](/docs/stack/json) modules.

## Prerequisites

Before you can index and search JSON documents, you need a database with either:

- [Redis Stack](/docs/stack), which automatically includes RediSearch and RedisJSON

- Redis v6.x or later and the following modules installed and enabled:
   - RediSearch v2.2 or later
   - RedisJSON v2.0 or later

## Create index with JSON schema

When you create an index with the `FT.CREATE` command, include the `ON JSON` keyword to index any existing and future JSON documents stored in the database.

To define the `SCHEMA`, you can provide [JSONPath](/docs/stack/json/path) expressions.
The result of each JSONPath expression is indexed and associated with a logical name called an `attribute` (previously known as a `field`).
You can use these attributes in queries.

{{% alert title="Note" color="info" %}}
Note: `attribute` is optional for `FT.CREATE`.
{{% /alert %}}

Use the following syntax to create a JSON index:

```sql
FT.CREATE {index_name} ON JSON SCHEMA {json_path} AS {attribute} {type}
```

For example, this command creates an index that indexes the name, description, and price of each JSON document that represents an inventory item:

```sql
127.0.0.1:6379> FT.CREATE itemIdx ON JSON PREFIX 1 item: SCHEMA $.name AS name TEXT $.description as description TEXT $.price AS price NUMERIC
```

See [Index limitations](#index-limitations) for more details about JSON index `SCHEMA` restrictions.

## Add JSON documents

After you create an index, RediSearch automatically indexes any existing, modified, or newly created JSON documents stored in the database.

You can use any RedisJSON write command, such as `JSON.SET` and `JSON.ARRAPPEND`, to create or modify JSON documents.

The following examples use these JSON documents to represent individual inventory items.

Item 1 JSON document:

```json
{
  "name": "Noise-cancelling Bluetooth headphones",
  "description": "Wireless Bluetooth headphones with noise-cancelling technology",
  "connection": {
    "wireless": true,
    "type": "Bluetooth"
  },
  "price": 99.98,
  "stock": 25,
  "colors": [
    "black",
    "silver"
  ]
}
```

Item 2 JSON document:

```json
{
  "name": "Wireless earbuds",
  "description": "Wireless Bluetooth in-ear headphones",
  "connection": {
    "wireless": true,
    "type": "Bluetooth"
  },
  "price": 64.99,
  "stock": 17,
  "colors": [
    "black",
    "white"
  ]
}
```

Use `JSON.SET` to store these documents in the database:

```sql
127.0.0.1:6379> JSON.SET item:1 $ '{"name":"Noise-cancelling Bluetooth headphones","description":"Wireless Bluetooth headphones with noise-cancelling technology","connection":{"wireless":true,"type":"Bluetooth"},"price":99.98,"stock":25,"colors":["black","silver"]}'
"OK"
127.0.0.1:6379> JSON.SET item:2 $ '{"name":"Wireless earbuds","description":"Wireless Bluetooth in-ear headphones","connection":{"wireless":true,"type":"Bluetooth"},"price":64.99,"stock":17,"colors":["black","white"]}'
"OK"
```

Because indexing is synchronous, the document will be available on the index as soon as the `JSON.SET` command returns.
Any subsequent queries that match the indexed content will return the document.

## Search the index

To search the index for JSON documents, use the `FT.SEARCH` command.
You can search any attribute defined in the `SCHEMA`.

For example, use this query to search for items with the word "earbuds" in the name:

```sql
127.0.0.1:6379> FT.SEARCH itemIdx '@name:(earbuds)'
1) "1"
2) "item:2"
3) 1) "$"
   2) "{\"name\":\"Wireless earbuds\",\"description\":\"Wireless Bluetooth in-ear headphones\",\"connection\":{\"wireless\":true,\"connection\":\"Bluetooth\"},\"price\":64.99,\"stock\":17,\"colors\":[\"black\",\"white\"]}"
```

This query searches for all items that include "bluetooth" and "headphones" in the description:

```sql
127.0.0.1:6379> FT.SEARCH itemIdx '@description:(bluetooth headphones)'
1) "2"
2) "item:1"
3) 1) "$"
   2) "{\"name\":\"Noise-cancelling Bluetooth headphones\",\"description\":\"Wireless Bluetooth headphones with noise-cancelling technology\",\"connection\":{\"wireless\":true,\"type\":\"Bluetooth\"},\"price\":99.98,\"stock\":25,\"colors\":[\"black\",\"silver\"]}"
4) "item:2"
5) 1) "$"
   2) "{\"name\":\"Wireless earbuds\",\"description\":\"Wireless Bluetooth in-ear headphones\",\"connection\":{\"wireless\":true,\"connection\":\"Bluetooth\"},\"price\":64.99,\"stock\":17,\"colors\":[\"black\",\"white\"]}"
```

Now search for Bluetooth headphones with a price less than 70:

```sql
127.0.0.1:6379> FT.SEARCH itemIdx '@description:(bluetooth headphones) @price:[0 70]'
1) "1"
2) "item:2"
3) 1) "$"
   2) "{\"name\":\"Wireless earbuds\",\"description\":\"Wireless Bluetooth in-ear headphones\",\"connection\":{\"wireless\":true,\"connection\":\"Bluetooth\"},\"price\":64.99,\"stock\":17,\"colors\":[\"black\",\"white\"]}"
```

For more information about search queries, see [Search query syntax](/docs/stack/search/reference/query_syntax).

{{% alert title="Note" color="info" %}}
`FT.SEARCH` queries require `attribute` modifiers. Don't use JSONPath expressions in queries because the query parser doesn't fully support them.
{{% /alert %}}

## Index JSON arrays as TAG

If you want to index string or boolean values as TAG within a JSON array, use the [JSONPath](/docs/stack/json/path) wildcard operator.

To index an item's list of available `colors`, specify the JSONPath `$.colors.*` in the `SCHEMA` definition during index creation:

```sql
127.0.0.1:6379> FT.CREATE itemIdx2 ON JSON PREFIX 1 item: SCHEMA $.colors.* AS colors TAG $.name AS name TEXT $.description as description TEXT
```

Now you can search for silver headphones:

```sql
127.0.0.1:6379> FT.SEARCH itemIdx2 "@colors:{silver} (@name:(headphones)|@description:(headphones))"
1) "1"
2) "item:1"
3) 1) "$"
   2) "{\"name\":\"Noise-cancelling Bluetooth headphones\",\"description\":\"Wireless Bluetooth headphones with noise-cancelling technology\",\"connection\":{\"wireless\":true,\"type\":\"Bluetooth\"},\"price\":99.98,\"stock\":25,\"colors\":[\"black\",\"silver\"]}"
```

## Index JSON arrays as TEXT
Starting with RediSearch 2.6.0, full text search can be done on array of strings or on a JSONPath leading to multiple strings.

If you want to index multiple string values as TEXT, either use a JSONPath leading to a single array of strings, or a JSONPath leading to multiple string values, using JSONPath operators such as wildcard, filter, union, array slice, and/or recursive descent.

To index an item's list of available `colors`, specify the JSONPath `$.colors` in the `SCHEMA` definition during index creation:

```sql
127.0.0.1:6379> FT.CREATE itemIdx3 ON JSON PREFIX 1 item: SCHEMA $.colors AS colors TEXT $.name AS name TEXT $.description as description TEXT
```

```sql
JSON.SET item:3 $ '{"name":"True Wireless earbuds","description":"True Wireless Bluetooth in-ear headphones","connection":{"wireless":true,"type":"Bluetooth"},"price":74.99,"stock":20,"colors":["red","light blue"]}'
"OK"
```

Now you can do full text search for light colored headphones:

```sql
127.0.0.1:6379> FT.SEARCH itemIdx3 '@colors:(white|light) (@name|description:(headphones))' RETURN 1 $.colors
1) (integer) 2
2) "item:2"
3) 1) "$.colors"
   2) "[\"black\",\"white\"]"
4) "item:3"
5) 1) "$.colors"
   2) "[\"red\",\"light blue\"]"
```

### Limitations
- When a JSONPath may lead to multiple values and not only to a single array, e.g., when a JSONPath contains wildcards, etc., specifying `SLOP` or `INORDER` in `FT.SEARCH` will return an error, since the order of the values matching the JSONPath is not well defined, leading to potentially inconsistent results.

   For example, using a JSONPath such as `$..b[*]` on a JSON value such as
   ```json
   {
      "a": [
         {"b": ["first first", "first second"]},
         {"c":
            {"b": ["second first", "second second"]}},
         {"b": ["third first", "third second"]}
      ]
   }
   ```
   may match values in various ordering, depending on the specific implementation of the JSONPath library being used. 
   
   Since `SLOP` and `INORDER` consider relative ordering among the indexed values, and results may change in future releases, therefore an error will be returned.

- When JSONPath leads to multiple values:
  - String values are indexed
  - `null` values are skipped
  - Any other value type is causing an indexing failure
  
- `SORTBY` is only sorting by the first value
- No `HIGHLIGHT` support
- `RETURN` of a Schema attribute, whose JSONPath leads to multiple values, returns only the first value (as a JSON String)
- If a JSONPath is specified by the `RETURN`, instead of a Schema attribute, all values are returned (as a JSON String)

### Handling phrases in different array slots:
When indexing, a predefined delta is used to increase positional offsets between array slots for multi text values. This delta controls the level of separation between phrases in different array slots (related to the `SLOP` parameter of `FT.SEARCH`).
This predefined value is set by `RediSearch` configuration parameter `MULTI_TEXT_SLOP` (at module load-time). The default value is 100.



## Index JSON objects

You cannot index JSON objects. If the JSONPath expression returns an object, it will be ignored.

To index the contents of a JSON object, you need to index the individual elements within the object in separate attributes.

For example, to index the `connection` JSON object, define the `$.connection.wireless` and `$.connection.type` fields as separate attributes when you create the index:

```sql
127.0.0.1:6379> FT.CREATE itemIdx3 ON JSON SCHEMA $.connection.wireless AS wireless TAG $.connection.type AS connectionType TEXT
"OK"
```

After you create the new index, you can search for items with the wireless `TAG` set to `true`:

```sql
127.0.0.1:6379> FT.SEARCH itemIdx3 '@wireless:{true}'
1) "2"
2) "item:2"
3) 1) "$"
   2) "{\"name\":\"Wireless earbuds\",\"description\":\"Wireless Bluetooth in-ear headphones\",\"connection\":{\"wireless\":true,\"connection\":\"Bluetooth\"},\"price\":64.99,\"stock\":17,\"colors\":[\"black\",\"white\"]}"
4) "item:1"
5) 1) "$"
   2) "{\"name\":\"Noise-cancelling Bluetooth headphones\",\"description\":\"Wireless Bluetooth headphones with noise-cancelling technology\",\"connection\":{\"wireless\":true,\"type\":\"Bluetooth\"},\"price\":99.98,\"stock\":25,\"colors\":[\"black\",\"silver\"]}"
```

You can also search for items with a Bluetooth connection type:

```sql
127.0.0.1:6379> FT.SEARCH itemIdx3 '@connectionType:(bluetooth)'
1) "2"
2) "item:1"
3) 1) "$"
   2) "{\"name\":\"Noise-cancelling Bluetooth headphones\",\"description\":\"Wireless Bluetooth headphones with noise-cancelling technology\",\"connection\":{\"wireless\":true,\"type\":\"Bluetooth\"},\"price\":99.98,\"stock\":25,\"colors\":[\"black\",\"silver\"]}"
4) "item:2"
5) 1) "$"
   2) "{\"name\":\"Wireless earbuds\",\"description\":\"Wireless Bluetooth in-ear headphones\",\"connection\":{\"wireless\":true,\"type\":\"Bluetooth\"},\"price\":64.99,\"stock\":17,\"colors\":[\"black\",\"white\"]}"
```

## Field projection

`FT.SEARCH` returns the entire JSON document by default. If you want to limit the returned search results to specific attributes, you can use field projection.

### Return specific attributes

When you run a search query, you can use the `RETURN` keyword to specify which attributes you want to include in the search results. You also need to specify the number of fields to return.

For example, this query only returns the `name` and `price` of each set of headphones:

```sql
127.0.0.1:6379> FT.SEARCH itemIdx '@description:(headphones)' RETURN 2 name price
1) "2"
2) "item:1"
3) 1) "name"
   2) "Noise-cancelling Bluetooth headphones"
   3) "price"
   4) "99.98"
4) "item:2"
5) 1) "name"
   2) "Wireless earbuds"
   3) "price"
   4) "64.99"
```

### Project with JSONPath

You can use [JSONPath](/docs/stack/json/path) expressions in a `RETURN` statement to extract any part of the JSON document, even fields that were not defined in the index `SCHEMA`.

For example, the following query uses the JSONPath expression `$.stock` to return each item's stock in addition to the `name` and `price` attributes.

```sql
127.0.0.1:6379> FT.SEARCH itemIdx '@description:(headphones)' RETURN 3 name price $.stock
1) "2"
2) "item:1"
3) 1) "name"
   2) "Noise-cancelling Bluetooth headphones"
   3) "price"
   4) "99.98"
   5) "$.stock"
   6) "25"
4) "item:2"
5) 1) "name"
   2) "Wireless earbuds"
   3) "price"
   4) "64.99"
   5) "$.stock"
   6) "17"
```

Note that the returned property name is the JSONPath expression itself: `"$.stock"`.

You can use the `AS` option to specify an alias for the returned property:

```sql
127.0.0.1:6379> FT.SEARCH itemIdx '@description:(headphones)' RETURN 5 name price $.stock AS stock
1) "2"
2) "item:1"
3) 1) "name"
   2) "Noise-cancelling Bluetooth headphones"
   3) "price"
   4) "99.98"
   5) "stock"
   6) "25"
4) "item:2"
5) 1) "name"
   2) "Wireless earbuds"
   3) "price"
   4) "64.99"
   5) "stock"
   6) "17"
```

This query returns the field as the alias `"stock"` instead of the JSONPath expression `"$.stock"`.

### Highlight search terms

You can [highlight](/docs/stack/search/reference/highlight) relevant search terms in any indexed `TEXT` attribute.

For `FT.SEARCH`, you have to explicitly set which attributes you want highlighted after the `RETURN` and `HIGHLIGHT` parameters.

Use the optional `TAGS` keyword to specify the strings that will surround (or highlight) the matching search terms.

For example, highlight the word "bluetooth" with bold HTML tags in item names and descriptions:

```sql
127.0.0.1:6379> FT.SEARCH itemIdx '(@name:(bluetooth))|(@description:(bluetooth))' RETURN 3 name description price HIGHLIGHT FIELDS 2 name description TAGS '<b>' '</b>'
1) "2"
2) "item:1"
3) 1) "name"
   2) "Noise-cancelling <b>Bluetooth</b> headphones"
   3) "description"
   4) "Wireless <b>Bluetooth</b> headphones with noise-cancelling technology"
   5) "price"
   6) "99.98"
4) "item:2"
5) 1) "name"
   2) "Wireless earbuds"
   3) "description"
   4) "Wireless <b>Bluetooth</b> in-ear headphones"
   5) "price"
   6) "64.99"
```

## Aggregate with JSONPath

You can use [aggregation](/docs/stack/search/reference/aggregations) to generate statistics or build facet queries.

The `LOAD` option accepts [JSONPath](/docs/stack/json/path) expressions. You can use any value in the pipeline, even if the value is not indexed.

This example uses aggregation to calculate a 10% price discount for each item and sorts the items from least expensive to most expensive:

```sql
127.0.0.1:6379> FT.AGGREGATE itemIdx '*' LOAD 4 name $.price AS originalPrice APPLY '@originalPrice - (@originalPrice * 0.10)' AS salePrice SORTBY 2 @salePrice ASC
1) "2"
2) 1) "name"
   2) "Wireless earbuds"
   3) "originalPrice"
   4) "64.99"
   5) "salePrice"
   6) "58.491"
3) 1) "name"
   2) "Noise-cancelling Bluetooth headphones"
   3) "originalPrice"
   4) "99.98"
   5) "salePrice"
   6) "89.982"
```

{{% alert title="Note" color="info" %}}
`FT.AGGREGATE` queries require `attribute` modifiers. Don't use JSONPath expressions in queries, except with the `LOAD` option, because the query parser doesn't fully support them.
{{% /alert %}}

## Index limitations

### Schema mapping

During index creation, you need to map the JSON elements to `SCHEMA` fields as follows:

- Strings as `TEXT`, `TAG`, or `GEO`.
- Numbers as `NUMERIC`.
- Booleans as `TAG`.
- JSON array of strings or booleans in a `TAG` field. Other types (`NUMERIC`, `GEO`, `NULL`) are not supported.
- You cannot index JSON objects. Index the individual elements as separate attributes instead.
- `NULL` values are ignored.

### TAG not sortable

If you create an index for JSON documents, you cannot sort on a `TAG` field. If you try to set a `TAG` to `SORTABLE`, `FT.CREATE` will return an error:

```sql
127.0.0.1:6379> FT.CREATE itemIdx4 ON JSON PREFIX 1 item: SCHEMA $.colors.* AS colors TAG SORTABLE
"On JSON, cannot set tag field to sortable - colors"
```

With hashes, you can use `SORTABLE` (as a side effect) to improve the performance of `FT.AGGREGATE` on `TAG` fields.
This is possible because the value in the hash is a string, such as "black,white,silver".

However with JSON, you can index an array of strings.
Because there is no valid single textual representation of those values,
RediSearch doesn't know how to sort the result.
