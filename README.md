# rusted_cypher

Rust crate for accessing the cypher endpoint of a neo4j server

This crate allows you to send cypher queries to the REST endpoint of a neo4j database. You can
execute queries inside a transaction or simply send queries that commit immediately.

## Examples

Code in examples are assumed to be wrapped in:

```rust
extern crate rusted_cypher;

use std::collections::BTreeMap;
use rusted_cypher::GraphClient;
use rusted_cypher::cypher::Statement;

fn main() {
  // Connect to the database
  let graph = GraphClient::connect(
      "http://neo4j:neo4j@localhost:7474/db/data").unwrap();

  // Example code here!
}
```

### Performing Queries

```rust
let mut query = graph.cypher().query();

// Statement implements From<&str>
query.add_statement(
    "CREATE (n:LANG { name: 'Rust', level: 'low', safe: true })");

let statement = Statement::new(
    "CREATE (n:LANG { name: 'C++', level: 'low', safe: {safeness} })")
    .with_param("safeness", false)
    .unwrap();

query.add_statement(statement);

query.send().unwrap();

graph.cypher().exec(
    "CREATE (n:LANG { name: 'Python', level: 'high', safe: true })")
    .unwrap();

let result = graph.cypher().exec(
    "MATCH (n:LANG) RETURN n.name, n.level, n.safe")
    .unwrap();

assert_eq!(result.data.len(), 3);

for row in result.rows() {
    let name: String = row.get("n.name").unwrap();
    let level: String = row.get("n.level").unwrap();
    let safeness: bool = row.get("n.safe").unwrap();
    println!("name: {}, level: {}, safe: {}", name, level, safeness);
}

graph.cypher().exec("MATCH (n:LANG) DELETE n").unwrap();
```

### With Transactions

```rust
let transaction = graph.cypher().transaction()
    .with_statement("CREATE (n:IN_TRANSACTION { name: 'Rust', level: 'low', safe: true })");

let (mut transaction, results) = transaction.begin().unwrap();

// Use `exec` to execute a single statement
transaction.exec("CREATE (n:IN_TRANSACTION { name: 'Python', level: 'high', safe: true })")
    .unwrap();

// use `add_statement` (or `with_statement`) and `send` to executes multiple statements
let stmt = Statement::new("MATCH (n:IN_TRANSACTION) WHERE (n.safe = {safeness}) RETURN n")
    .with_param("safeness", true)
    .unwrap();

transaction.add_statement(stmt);
let results = transaction.send().unwrap();

assert_eq!(results[0].data.len(), 2);

transaction.rollback();
```

### Statements with Macro

There is a macro to help building statements

```rust
let statement = cypher_stmt!(
    "CREATE (n:WITH_MACRO { name: {name}, level: {level}, safe: {safe} })", {
        "name" => "Rust",
        "level" => "low",
        "safe" => true
    }
).unwrap();
graph.cypher().exec(statement).unwrap();

let statement = cypher_stmt!("MATCH (n:WITH_MACRO) WHERE n.name = {name} RETURN n", {
    "name" => "Rust"
}).unwrap();

let results = graph.cypher().exec(statement).unwrap();
assert_eq!(results.data.len(), 1);

let statement = cypher_stmt!("MATCH (n:WITH_MACRO) DELETE n").unwrap();
graph.cypher().exec(statement).unwrap();
```

## License

Licensed under either of

 * Apache License, Version 2.0, ([LICENSE-APACHE](LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0)
 * MIT license ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally
submitted for inclusion in the work by you, as defined in the Apache-2.0
license, shall be dual licensed as above, without any additional terms or
conditions.
