---
title: "What is COLLATE in a Database and What Does It Do?"
subtitle: ""
date: 2023-09-08T10:24:29+07:00
lastmod: 2023-09-08T10:24:29+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "Explains what COLLATE is in a database and how it controls string sorting and comparison, including case and accent sensitivity."
license: ""
images: []
featuredImage: "featured-image.webp"
featuredImagePreview: "featured-image.webp"

tags: ["Database", "SQL"]
categories: ["SQL"]

lightgallery: true
---

<!--more-->

## What is COLLATE?
When creating a Database or Table, we define the CHARACTER SET and COLLATE. For example, in MariaDB, `utf8mb4_unicode_ci` consists of:

### Character set
`utf8mb4` means using a character set encoded in full 4-byte UTF-8 (older `utf8` was 3-byte UTF-8, which doesn't support full emojis and Unicode).

### Collation Algorithm
`unicode` means sorting (the value of) characters according to Unicode.

### Diacritical sensitivity
- `ai` (ancient insensitive) e.g., a == ᾰ
- `as` (ancient sensitive) e.g., a != ᾰ
- `ci` (case insensitive) e.g., A == a
- `cs` (case sensitive) e.g., A != a

## Usecase
The simplest example is probably searching for names, whether people or places, like:

```sql
SELECT
  _id,
  first_name,
  last_name
FROM
  users
WHERE
  LOWER(first_name) = 'kawin';
```

Result:

```json
[
  {
    "_id": 5,
    "first_name": "KAWIN",
    "last_name": "eiei"
  },
  {
    "_id": 4,
    "first_name": "kawin",
    "last_name": "naja"
  },
  {
    "_id": 3,
    "first_name": "Kawin",
    "last_name": "Viriyaprasopsook"
  }
]
```

We get `Kawin`, `kawin`, `KAWIN`, which seems normal. But if we try to EXPLAIN it:

```sql
EXPLAIN FORMAT=JSON
SELECT
  _id,
  first_name,
  last_name
FROM
  users
WHERE
  LOWER(first_name) = 'kawin';
```

We will find that our query reads all 1000 rows in the search.

```json
{
  "query_block": {
    "select_id": 1,
    "nested_loop": [
      {
        "table": {
          "table_name": "users",
          "access_type": "index",
          "key": "udx_full_name",
          "key_length": "518",
          "used_key_parts": ["first_name", "last_name"],
          "rows": 1000,
          "filtered": 100,
          "attached_condition": "lcase(users.first_name) = 'kawin'",
          "using_index": true
        }
      }
    ]
  }
}
```

This is called **index obscure**, meaning the database sees the `WHERE` clause as `fn(column)` and has to **iterate through the entire table to apply LOWER()**. If the table/column was already created as case-insensitive, we can just use `=`.

```sql
EXPLAIN FORMAT=JSON
SELECT
  _id,
  first_name,
  last_name
FROM
  users
WHERE
  first_name = 'kawin';
```

Which will EXPLAIN to this, reading only 3 rows instead of the entire table, and the result is the same as using LOWER().

```json
{
  "query_block": {
    "select_id": 1,
    "nested_loop": [
      {
        "table": {
          "table_name": "users",
          "access_type": "ref",
          "possible_keys": ["udx_full_name"],
          "key": "udx_full_name",
          "key_length": "259",
          "used_key_parts": ["first_name"],
          "ref": ["const"],
          "rows": 3,
          "filtered": 100,
          "attached_condition": "users.first_name = 'kawin'",
          "using_index": true
        }
      }
    ]
  }
}
```

{{< admonition warning >}}
Since uppercase and lowercase characters are treated as equal, be careful when using `UNIQUE` constraints, as the database will consider "Kawin" to be a duplicate of "kawin".
{{< /admonition >}}
