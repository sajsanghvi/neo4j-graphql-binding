# neo4j-graphql-binding

<code>neo4j-graphql-binding</code> provides a quick way to embed a [Neo4j Graph Database](https://neo4j.com/product/) GraphQL API (using the [neo4j-graphql](https://github.com/neo4j-graphql/neo4j-graphql) plugin) into your local GraphQL server.

<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:0 orderedList:0 -->

- [Overview](#overview)
	- [neo4jGraphQLBinding](#neo4jgraphqlbinding)
	- [neo4jIDL](#neo4jidl)
	- [neo4jExecute](#neo4jexecute)
	- [buildNeo4jTypeDefs and @model directives](#buildneo4jtypedefs-and-model-directives)
- [Installation and usage](#installation-and-usage)
	- [Request Examples](#request-examples)
- [Auto-Generated Query Types and Mutations](#auto-generated-query-types-and-mutations)
	- [Request Examples](#request-examples)
- [License](#license)

<!-- /TOC -->

### Overview

##### neo4jGraphQLBinding
In your server setup, you use <code>neo4jGraphQLBinding</code> to create a [GraphQL binding](https://www.npmjs.com/package/graphql-binding) to your Neo4j server and set the binding into your request context. The binding can then be accessed in your local resolvers to send requests to your remote Neo4j GraphQL server for any <code>query</code> or <code>mutation</code> in your <code>typeDefs</code> with a [@cypher directive](https://neo4j.com/developer/graphql/#_neo4j_graphql_extension). Queries use the read only [graphql.query](https://github.com/neo4j-graphql/neo4j-graphql/tree/3.3#procedures) procedure and mutations use the read/write [graphql.execute](https://github.com/neo4j-graphql/neo4j-graphql/tree/3.3#procedures) procedure.

##### neo4jIDL
In order to update your Neo4j GraphQL schema, you can use the neo4jIDL helper function. Internally, this sends a request to Neo4j to call the [graphql.idl](https://github.com/neo4j-graphql/neo4j-graphql/tree/3.3#uploading-a-graphql-schema) procedure using the typeDefs you provide.

##### neo4jExecute
You can use <code>neo4jExecute</code> as a helper for using the binding. If you delegate the processing of a local resolver entirely to your Neo4j GraphQL server, then you only use the binding once in that resolver and have to repeat its name. <code>neo4jExecute</code> automates this away for you by obtaining request information from the local resolver's [info](https://blog.graph.cool/graphql-server-basics-demystifying-the-info-argument-in-graphql-resolvers-6f26249f613a) argument.

##### buildNeo4jTypeDefs and @model directives
In order to use the query and mutation types [generated by neo4j-graphql](https://github.com/neo4j-graphql/neo4j-graphql/tree/3.3#auto-generated-query-types), you can use <code>buildNeo4jTypeDefs</code> to add the <i>same generated types</i> into your <code>typeDefs</code>. This currently generates query types for each type with a [@model](#auto-generated-query-types-and-mutations) directive. The generated queries have support for <code>first</code> and <code>offset</code> (for pagination), and <code>orderBy</code> in asc and desc order for each field on a type. For now, only a creation mutation is generated (e.g. createPerson).

### Installation and usage

	npm install --save neo4j-graphql-binding

In your local GraphQL server setup, <code>neo4jGraphQLBinding</code> is used with your schema's <code>typeDefs</code> and your [Neo4j driver](https://www.npmjs.com/package/neo4j-driver) to create a GraphQL binding to your Neo4j Graphql server. The binding is then set into the server's request context at the path <code>.neo4j</code>:
```js
import { GraphQLServer } from 'graphql-yoga';
import { makeExecutableSchema } from 'graphql-tools';
import { v1 as neo4j } from 'neo4j-driver';

import { neo4jGraphQLBinding, neo4jIDL } from 'neo4j-graphql-binding';
import { typeDefs, resolvers } from './schema.js';

const driver = neo4j.driver("bolt://localhost", neo4j.auth.basic("user", "password"));

neo4jIDL(driver, typeDefs);

const localSchema = makeExecutableSchema({
  typeDefs: typeDefs,
  resolvers: resolvers
});

const neo4jGraphqlAPI = neo4jGraphQLBinding({
  typeDefs: typeDefs,
  driver: driver
});

const server = new GraphQLServer({
  schema: localSchema,
  context: {
    neo4j: neo4jGraphqlAPI
  }
});

const options = {
  port: 4000,
  endpoint: '/graphql',
  playground: '/playground',
};

server.start(options, ({ port }) => {
  console.log(`Server started, listening on port ${port} for incoming requests.`)
});

```
In your schema, the binding is accessed to send a request to your Neo4j Graphql server to process any <code>query</code> or <code>mutation</code> in your <code>typeDefs</code> that has a <code>@cypher</code> directive.
Note that the @cypher directive on the createPerson mutation formats its return data into a JSON that matches the custom payload type createPersonPayload. This is possible with some Cypher features released in Neo4j 3.1 (see: https://neo4j.com/blog/cypher-graphql-neo4j-3-1-preview/).

<code>schema.js</code>
```js
export const typeDefs = `
  type Person {
    name: String
    friends: [Person] @relation(
      name: "friend",
      direction: OUT
    )
  }

  type Query {
    readPersonAndFriends(name: String!): [Person]
      @cypher(statement: "MATCH (p:Person {name: $name}) RETURN p")
  }

  input createPersonInput {
    name: String!
  }

  type createPersonPayload {
    name: String
  }

  type Mutation {
    createPerson(person: createPersonInput!): createPersonPayload
      @cypher(statement: "CREATE (p:Person) SET p += $person RETURN p{ .name } AS createPersonPayload")
  }

  schema {
    query: Query
    mutation: Mutation
  }
`;

export const resolvers = {
  Query: {
    readPersonAndFriends: (obj, params, ctx, info) => {
      return ctx.neo4j.query.readPersonAndFriends(params, ctx, info);
    }
  },
  Mutation: {
    createPerson: (obj, params, ctx, info) => {
      return ctx.neo4j.mutation.createPerson(params, ctx, info);
    }
  }
};
```

If you use the binding to call a remote resolver of the same name as the local resolver it's called in, you can use <code>neo4jExecute</code> to avoid repeating the resolver name:

```js
import { neo4jExecute } from 'neo4j-graphql-binding';

export const resolvers = {
  Query: {
    readPersonAndFriends: (obj, params, ctx, info) => {
      return neo4jExecute(params, ctx, info);
    }
  },
  Mutation: {
    createPerson: (obj, params, ctx, info) => {
      return neo4jExecute(params, ctx, info);
    }
  }
}
```

Handling return data using async / await:
```js
Query: {
  readPersonAndFriends: async (obj, params, ctx, info) => {
    const data = await neo4jExecute(params, ctx, info);
    // post-process data, send subscriptions, etc.
    return data;
  }
}
```

##### Request Examples
<code>readPersonAndFriends.graphql</code>
```js
query readPersonAndFriends($name: String!) {
  readPersonAndFriends(name: $name) {
    name
    friends {
      name
    }
  }
}
```
```json
{
  "data":{
    "readPersonAndFriends": [
      {
        "name": "Michael",
        "friends": [
          {
            "name": "Marie"
          }
        ]
      }
    ]
  }
}
```
<code>createPerson.graphql</code>
```js
mutation createPerson($person: createPersonInput!) {
  createPerson(person: $person) {
    name
  }
}
```
```json
{
  "data":{
    "createPerson": {
      "name": "Michael"
    }
  }
}
```

### Auto-Generated Query Types and Mutations
First, add the @model type directive to any type for which you want query and mutation types to be generated.
```js
type Person @model {
  name: String
  friends: [Person] @relation(
    name: "friend",
    direction: OUT
  )
}
```
Next, use <code>buildNeo4jTypeDefs</code> in your server setup to generate those queries and mutations into your typeDefs and use the result in both your binding and your schema.
```js
const neo4jTypeDefs = buildNeo4jTypeDefs({ typeDefs: typeDefs });

const neo4jGraphqlAPI = neo4jGraphQLBinding({
  typeDefs: neo4jTypeDefs,
  driver: driver
});

const localSchema = makeExecutableSchema({
  typeDefs: neo4jTypeDefs,
  resolvers: resolvers
});
```
If you already have a Person query or a createPerson mutation, <code>buildNeo4jTypeDefs</code> <b>will not overwrite</b> them. In this case, the following query type would be added to your typeDefs:
```js
Person(name, names, orderBy, _id, _ids, first, offset): [Person]
```
as well as the mutation (which returns update statistics):
```js
createPerson(name): String
```
Now the binding will be able to generate requests for the auto-generated query and mutation types.
```js
import { neo4jExecute } from 'neo4j-graphql-binding';

export const resolvers = {
  Query: {
    Person: (obj, params, ctx, info) => {
      return neo4jExecute(params, ctx, info);
    }
  },
  Mutation: {
    createPerson: (obj, params, ctx, info) => {
      return neo4jExecute(params, ctx, info);
    }
  }
}
```
##### Request Examples
To query for someone's first two friends, in ascending order:
<code>readPersonAndFriends.graphql</code>
```js
query readPersonAndFriends($name: String!) {
  Person(name: $name) {
    name
    friends(first: 2, orderBy: name_asc) {
      name
    }
  }
}
```
```json
{
  "data":{
    "Person": [
      {
        "name": "Finn",
        "friends": [
          {
            "name": "BMO"
          },
          {
            "name": "Jake"
          }
        ]
      }
    ]
  }
}
```
<code>createPerson.graphql</code>
```js
mutation createPerson($name: String) {
  createPerson(name: $name)
}
```
```json
{
  "data": {
    "createPerson": "Nodes created: 1\nProperties set: 1\nLabels added: 1\n"
  }
}
```
### License
The code is available under the [MIT](LICENSE) license.
