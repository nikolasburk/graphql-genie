- [GraphQLGenie API](#graphqlgenie-api)
	- [**constructor**](#constructor)
	- [**use**](#use)
	- [**getSchema**](#getschema)
	- [**printSchema**](#printschema)
	- [**getUserTypes**](#getusertypes)
	- [**getFragmentTypes**](#getfragmenttypes)
	- [**getRawData**](#getrawdata)
	- [**importRawData**](#importrawdata)
	- [**getDataResolver**](#getdataresolver)
		- [Hooks](#hooks)
	- [**getSchemaBuilder**](#getschemabuilder)
- [GraphQLSchemaBuilder API](#graphqlschemabuilder-api)
	- [**printSchemaWithDirectives**](#printschemawithdirectives)
	- [**addTypeDefsToSchema**](#addtypedefstoschema)
	- [**setResolvers**](#setresolvers)
	- [**setIResolvers**](#setiresolvers)
	- [**isUserType**](#isusertype)

### GraphQLGenie API

---

#### **constructor**

```typescript
constructor(options: GraphQLGenieOptions)
```

Create a new GraphQLGenie. GraphQLGenieOptions look like:

```typescript
// Most likely pass in typeDefs, schemaBuilder is for advanced use cases 
interface GraphQLGenieOptions {
    schemaBuilder?: GraphQLSchemaBuilder;
    typeDefs?: string;
    generatorOptions?: GenerateConfig;
    fortuneOptions?: FortuneOptions;
}
// All default to true
interface GenerateConfig {
	  generateGetOne?: boolean; // GraphQL API will have a singular queries using unique fields
    generateGetAll?: boolean; // GraphQL API will have a Query to get all of a type, with filters
    generateCreate?: boolean; // GraphQL API will have a Mutation to create new data of each type
    generateUpdate?: boolean; // GraphQL API will have a Mutation to update data of each type
    generateDelete?: boolean; // GraphQL API will have a Mutation to delete data of each type
    generateUpsert?: boolean; // GraphQL API will have a Mutation to upsert data of each type
    generateConnections?: boolean; // GraphQL API will have a Query to get all of a type, with filters, that returns a Connection rather than simple array
		generateMigration?: boolean //a Query exportData and a Mutation importData will be created
}
```

---

####  **use**

```ts
use(plugin: GeniePlugin): Promise<Void>
```

Pass in a plugin that alters the schema, see the [subscriptions plugin](https://github.com/genie-team/graphql-genie/tree/master/plugins/subscriptions) for an example

> See info about the GeniePlugin interface in [GraphQLGenieInterfaces.ts](https://github.com/genie-team/graphql-genie/blob/master/src/GraphQLGenieInterfaces.ts)

#### **getSchema**

```typescript
getSchema(): GraphQLSchema
```

Get the schema

---

#### **printSchema**

```ts
printSchema(): string
```

Return a string of the full schema with directives

---

#### **getUserTypes**

```typescript
getUserTypes(): Promise<string[]>
```

Returns an array of all the user defined object types in the schema

---

#### **getFragmentTypes**

```typescript
getFragmentTypes(): Promise<object>
```

When using Apollo or another tool you may need to get information on the fragment types, genie provides a helper for this

```typescript
import { IntrospectionFragmentMatcher, IntrospectionResultData } from 'apollo-cache-inmemory';
const introspectionQueryResultData = <IntrospectionResultData>await genie.getFragmentTypes();
const fragmentMatcher = new IntrospectionFragmentMatcher({
    introspectionQueryResultData
});
```

---

#### **getRawData**

```
getRawData(types?: string[], context?): Promise<any[]>
```
Returns data in the database, this will look a little different than what is returned by graphql. Every object will have a __typename field and relations will just be ids or an array of ids rather than objects. Also if you use interfaces and unions there may be null fields you weren't expecting on that type.

If `generateMigrations` is configured to true a `exportData` query will be created in your API that calls this function.

**types** - Optional. List of the GraphQL Object Types you want data for. If null or blank all data will be returned

**context** - Optional. Context object that will be sent to input/output hooks, may be needed if using the authentication plugin

 It will look something like:

```json
[
   {
      "id":"eFRKZHZNM2xPaURGU3ljOlVzZXI=",
      "__typename":"User",
      "importID":null,
      "email":null,
      "name":"Update5",
      "submissions":[
         "VkhoRzNWNWJkNHlUVDBuOkNvbW1lbnQ="
      ],
      "address":"UldLemkxNnVld1N0Q1pOOkFkZHJlc3M=",
      "liked":[]
   },
   {
      "id":"UldLemkxNnVld1N0Q1pOOkFkZHJlc3M=",
      "__typename":"Address",
      "importID":null,
      "city":"EC",
      "user":"eFRKZHZNM2xPaURGU3ljOlVzZXI="
   },
   {
      "id":"VkhoRzNWNWJkNHlUVDBuOkNvbW1lbnQ=",
      "__typename":"Comment",
      "importID":null,
      "title":null,
      "text":"update5",
      "author":"eFRKZHZNM2xPaURGU3ljOlVzZXI=",
      "approved":true
   }
]
```

---

#### **importRawData**

```
importRawData(data: any[], merge = false, defaultTypename?: string, context?): Promise
```

Import data into the store.  Note any relationship fields must also either exist already or also be part of the data provided.

If `generateMigrations` is configured to true a `importData` mutation will be created in your API that calls this function.

**data** 

an array of objects to import. It can be either in the format of raw data (as exported from `getRawData` ) or in the format returned from a graphql query. Note that if it is in the format of the graphql query and __typename fields are not added the defaultTypename must be provided


**merge** - Default = false

If false every object will create a new object, the id won't be preserved from the current data but relationships will still be built as they were in the provided data.

If true data will be merged based on ID, with new entries only being created if the given id does not exist already. Provided id will be used for creating data as well.

Note when merging list fields by default the array in the provided data will replace the existing data array. If you don't want to do that instead of providing an array you can provide an object with fields for push and pull or set. 

```json
 {
  "id":"VkhoRzNWNWJkNHlUVDBuOkNvbW1lbnQ=",
  "submissions": {
    "push": "VkhoRzNWNWJkNHlUVDBuOkNvbW1lbnQ=",
    "pull": "UldLemkxNnVld1N0Q1pOOkFkZHJlc3M="
   }
 }
```

**defaultTypename** - Optional.

Must be provided if every object in data does not have a `__typename` property

**context** - Optional. 

Context object that will be sent to input/output hooks, may be needed if using the authentication plugin


---

#### **getDataResolver**

```typescript
getDataResolver(): DataResolver
```

DataResolver handles all the operations with your actual data. Such as CRUD and hooks. It has the following functions 

```typescript
export interface DataResolver {
	create(graphQLTypeName: string, records, meta?): Promise<any>;
	find(graphQLTypeName: string, ids?: string[], options?, meta?): Promise<any>;
	update(graphQLTypeName: string, updates: object, meta?, options?: object): Promise<any>;
	delete(graphQLTypeName: string, ids?: string[], meta?): Promise<any>;	
	addOutputHook(graphQLTypeName: string, hook: DataResolverOutputHook);
	addInputHook(graphQLTypeName: string, hook: DataResolverInputHook);
	getValueByUnique(returnTypeName: string, args, meta): Promise<Object>;
	canAdd(graphQLTypeName: string, records: Object, meta): Promise<boolean>;
	getConnection(allEdges: any[], before: string, after: string, first: number, last: number): Connection;
	getFeatures(): Features;
	applyOptions(graphQLTypeName: string, records, options, meta?);
	getStore(): any;
	beginTransaction(): Promise<void>;
	endTransaction(): Promise<void>;
	computeId(graphType: string, id?: string): string;
	getTypeFromId(inputId: string): string;
	getOriginalIdFromObjectId(inputId: string): string;
	getLink(graphQLTypeName: string, field: string): string;
}
```

##### Hooks

Most likely use of this is to add hooks into the CRUD operations against your database. The DataResolver has 2 functions to add hooks. For more info on the context, record and update objects see the [fortune documentation](http://fortune.js.org/#input-and-output-hooks).

Note that when using fortune hooks the resolvers context and info arguments can be had at context.request.meta.context and context.request.meta.info


```typescript
 interface DataResolverInputHook {
    (context?, record?, update?): any;
}
 interface DataResolverOutputHook {
    (context?, record?): any;
}
```

> See info about the DataResolver interface in [GraphQLGenieInterfaces.ts](https://github.com/genie-team/graphql-genie/blob/master/src/GraphQLGenieInterfaces.ts)

#### **getSchemaBuilder**

```ts
getSchemaBuilder(): GraphQLSchemaBuilder
```

> GraphQLSchemaBuilder has some additional helpers to add types and resolvers to a graphql schema
> See the [GraphQLSchemaBuilder API documentation](#graphqlschemabuilder-api)

---

### GraphQLSchemaBuilder API

---

#### **printSchemaWithDirectives**

```typescript
printSchemaWithDirectives()
```

Returns a string of the full schema with directives

---

#### **addTypeDefsToSchema**

```typescript
addTypeDefsToSchema($typeDefs = ''): GraphQLSchema
```

Completely rebuilds the schema with the new typeDefs. You need to use this if we want any of the custom directives to work on your new typeDefs. Otherwise you can use the schema stitching tools from 

---

#### **setResolvers**

```typescript
setResolvers(typeName: string, fieldResolvers: Map<string, GraphQLFieldResolver<any, any>>)
```

Set resolvers on the schema for the given typename and a map of the fild name to the resolver

---

#### **setIResolvers**

```ts
setIResolvers(iResolvers: IResolvers): GraphQLSchema
```

Set resolvers of type [IResolvers from graphql-tools](https://www.apollographql.com/docs/graphql-tools/resolvers.html#Resolver-map)

---

#### **isUserType**

```ts
isUserType(type: GraphQLType): boolean
```

returns true if the type isn't generated by GraphQLGenie

---
