# bp4mc2-graphql
Mapping graphql to RDF and back. This document describes the requirements to which a Knowledge Graph GraphQL endpoint needs to comply. These requirements are made with respect to the following fundamentals:

- **A Knowledge Graph endpoint should be accessible by an API user without any need to "know" Linked Data**: one of the most important reasons to use GraphQL is the acceptance of the developers community (otherwise we could just use SPARQL). Linked Data is perceived as complex and hard to understand. By hiding this complexity, we lower the bar to get started.
- **A Knowledge Graph endpoint should only contain data that can be linked to a universal scheme, e.g.: Linked Data**: although we want to hide the RDF details, these details are very important to obtain a working and extensible knowledge graph. GraphQL schemas are by definition bound to a specific closed-world context: it requires an interface-specific schema. This therefore makes it difficult to combine GraphQL data that originates from different sources. Without the notion of globally, universal identifiers, linking between datasets is also a challenge. The universal, open-world assumption of Linked Data, with the availability of URI's for schema and data, is needed to link different closed-world datasets together.
- **At least RDF datasets and other GraphQL endpoints should be accessible via the Knowledge Graph endpoint**: as this Knowledge Graph endpoint is all about the union of RDF and GraphQL, the actual gain is achieved when closed world GraphQL endpoints are opened by adding a RDF context, and RDF datasets are made available for developers by adding a GraphQL frontend.

This requirements document is **not** about a single GraphQL endpoint that might not contain RDF data at all, but is only about GraphQL endpoints that are able to comply to a knowledge graph perspective. We are talking about a GraphQL+ endpoint: it complies to the [GraphQL specification](https://graphql.github.io/graphql-spec/), but - when you ask for more - you can get more contextual information and linked data.

For example:
- a http request to the endpoint with accept header `application/json` would return the [usual JSON GraphQL response](https://graphql.github.io/graphql-spec/June2018/#sec-Response), serialized as JSON;
- a http request to the endpoint with accept header `application/ld+json` would return a contextualised GraphQL response, serialized as [JSON-LD](https://www.w3.org/TR/json-ld/);
- a http request to the endpoint with accept header `text/turtle` would return a contextualised GraphQL response, serialized as [Turtle](https://www.w3.org/TR/turtle/).

## Resources

In [Bridges between GraphQL and RDF](https://www.w3.org/Data/events/data-ws-2019/assets/position/Ruben%20Taelman.pdf) four different solutions are investigated to query RDF datasources using GraphQL:

- [HyperGraphQL](https://www.hypergraphql.org)
- [Topquadrant GraphQL](https://www.topquadrant.com/technology/graphql/)
- [StarDog](https://www.stardog.com/docs/#_graphql_queries)
- [Comunica](https://github.com/comunica/comunica)

The first three solutions are commercial, the fourth solution is an academic solution that is described in more detail in the paper [GraphQL-LD: Linked Data Querying with GraphQL](https://comunica.github.io/Article-ISWC2018-Demo-GraphQlLD/).

## Requirements for the solution:

The solution can be distinguished into three different capabilities:
1. GraphQL request capability;
2. GraphQL querying capability;
3. GraphQL combining capability.

The *GraphQL request* capability delivers a way of sending a GraphQL request that results in a GraphQL response that can be serialized into different formats, including plain JSON and also semantically rich Linked Data formats as JSON-LD, RDF/XML and Turtle.

The *GraphQL querying* capability delivers a way of querying a backend based on the GraphQL request. The backend might be any kind of backend (SPARQL, GraphQL, relational, API).

The *GraphQL combining* capability delivers a way of combining results from different GraphQL endpoints into one resultset. This means that the corresponding GraphQL schema should in some way be constructed from the schema's from the individual GraphQL endpoints.

From these capabilities, the following requirements can be formulated. Any GraphQL endpoint that can be considered a Knowledge Graph endpoint, should at least implement requirements R1 to R5. Any Knowledge Graph endpoint that can considered a complete replacement of a SPARQL endpoint with federation features (e.g.: an endpoint that is used to combine results from different, federated datasets), should also implement requirements R6 to R11.

1. It should be possible to send a regular GraphQL query (without any extensions) over http: CAP-1;
2. It should be possible to add a JSON-LD context to the GraphQL result: CAP-1;
3. It should be possible to use content negotiation to obtain specific RDF serialisations (at least RDF/XML, Turtle, JSON-LD and annotated HTML): CAP-1;
4. It should be possible to construct and deconstruct URI's in de resulting response: CAP-1;
5. It should be possible to construct and deconstruct URI's in the query: CAP-2;
6. It should be possible to map the GraphQL query to a SPARQL query: CAP-2;
7. It should be possible to extend the GraphQL query with specific back-end extensions: CAP-2;
8. It should be possible to map an RDF result to a GraphQL result, conforming the GraphQL schema: CAP-2;
9. It should be possible to map the GraphQL query to another GraphQL query with a different schema: CAP-2, CAP-3;
10. It should be possible to combine results from different GraphQL schema's into one combined GraphQL schema: CAP-3;
11. It should be possible to extend the GraphQL schema at run-time: CAP-3.

### 1. Send a regular GraphQL query

It is important that a regular Joe with only knowledge about GraphQL should be able to use the GraphQL interface over http. This make the starting-point for first-time use as low as possible. From this first step, extensions could be added for people that need more expressibility than the regular Joe.

### 2. Add JSON-LD context to the GraphQL result

JSON-LD is a very clever concept to enrich a plain JSON response with a semantic context. Only a `@context` element is needed, without changing the remaining part of the response. By adding a JSON-LD context, any GraphQL result magically transforms in a semantic rich Linked Data response. This context should not be hand-made. It should be possible to automatically generate the context from the configuration.

A regular JSON response can be contextualised without changing the response body itself. This can be done by adding a response header to the reponse, for example:

```
HTTP/1.1 200 OK
...
Content-Type: application/json
Link: <http://graphql.org/context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"

{
  data: {
    "project": {
      "tagline": "A query language for APIs"
    }
  }
}
```

As the GraphQL specification doesn't disallow the use of this Link header, a response that complies to the GraphQL specification might include such a Link header. For a knowledge graph GraphQL endpoint, this header MUST be included when an `application/json` response is requested.

When an `application/ld+json` response is requested, the context should be part of the response body. Mark that this extends the current implementation of GraphQL, that only described a `data` and `errors` root element. It doesn't disallow other root elements, so this extension is still complies to the [GraphQL specification for responses](https://graphql.github.io/graphql-spec/June2018/#sec-Response):

```
HTTP/1.1 200 OK
...
Content-Type: application/ld+json

{
  "@context": "http://graphql.org/context.jsonld",
  "data": {
    "project": {
      "tagline": "A query language for APIs"
    }
  }
}
```

It is recommended to add the context as a reference to a seperate context document and not include the context document into the response itself.

### 3. Content-negotiation

A very important concept of the http protocol is the content-negotation feature. With content-negotation, clients can state which data format they accept, and the server response with the corresponding format. With Linked Data, format is separated from meaning, so different formats may express the same meaning. At least the following formats should be available:

- Plain JSON: as stated by requirement R1, a regular GraphQL query should be possible, so it should also be possible to deliver a regular, plain JSON result;
- JSON-LD: as stated by requirement R2, it should be possible to add a JSON-LD context to the plain JSON result, which actually makes it a JSON-LD result;
- RDF/XML: With the availability of a JSON-LD result, any other serialization is available (these transformations are already available). Some clients are more familiar with an XML result, and without any real costs at the server side, this should be available.
- Turtle: As with the RDF/XML, a Turtle result is available without any real costs at the server side. As turtle is the most human-readable serialization of Linked Data, this serialization should be available.
- Annotated HTML. Some might argue that HTML should not be in the list of result formats, but HTML is actually the most used serialization format for Linked Data, as it is used by all major search engines to crawl the web. Annotated HTML is a specific kind of HTML that encapsulated Linked Data in a HTML web page. It also gives human-friendly access to the GraphQL result. Ideally, a clear separation of content and format is used (html5, css, etc), because we really only want to have another serialization format and not heavy-load user interface features to the GraphQL capabilities.

### 4. (De)construct URI's in the response

Apart from context, another major difference between plain JSON and semantically rich JSON-LD, is the use of universally usable URI's to identify resources instead of context-specific identifiers like numbers or codes.

In most situations, these context specific identifiers are actually functional properties of the resource itself. For example: the registration number of a car might also be used as the identifier ("primary key" in traditional systems) for the car itself. In such a case, the requirement should be to add another property that can be used as the URI for this car.

When the original source is a RDF dataset, the inverse situation can also occur: a URI is available, but the actual primary key is missing from the dataset. In such a case, the requirement should be to add another function property that can be used as the primary key.

The use of the [HATEOAS concept](https://en.wikipedia.org/wiki/HATEOAS) dictates yet another kind of identifier, the URL at which location more information is available. These URL's are in most cases links to other (information) resources. These links resemble the same links between resources in an RDF data model, with the difference that in an RDF model the links are between the things themselves (for example between a car and its owner), and the HATEOS links are between the location of information about a car, and the location of information about its owner. From a more traditional perspective, these links resemble foreign keys.

### 5. (De)construct URI's in the query

As the query itself could also include references to specific resources, the construction and deconstruction requirements not only holds for the response, but also to the query itself.

### 6. Map GraphQL to SPARQL query

As the original dataset might be an RDF dataset, available via a SPARQL endpoint, it should be possible to map such a GraphQL query to its corresponding SPARQL counterpart. As is [already proven](https://comunica.github.io/Article-ISWC2018-Demo-GraphQlLD/), any GraphQL query can be transformed to a SPARQl query. All that is needed, is a mapping between the GraphQL schema and the corresponding RDF context. This means that this requirement can probably be realised with the same feature as the one that is needed for requirement R2 (and in some cases requirements R3 and R4).

Because GraphQL is not as expressive as SPARQL, as [GraphQL queries represent trees](http://olafhartig.de/files/HartigPerez_WWW2018_Preprint.pdf), and not full graphs, it should be possible to extend the GraphQL request to obtain the same expressibility as a SPARQL query, as is specified in more detail in requirement R7.

### 7. Extend GraphQL

As the expressibility of GraphQL lacks certain features that are needed for some data analyses or querying capabilities, it should be possible to extend GraphQL.

The downside of these extensions is that they are specific to the solution that can use these extensions. Any extension should be fully documented and be specified as generic as possible. Standards should be used whenever possible. For example: a "sparql" extension should comply to the SPARQL standard and should not be specific to a certain solution. It might be that a specific solution only implements part of the extension, but the extension declaration itself should be as complete and generic as possible.

To specify extensions, the available extension mechanism(s) of GraphQL should be used.

### 8. Map RDF to GraphQL schema

This requirement is in fact the inverse of requirement R2. With a proper context, it should be possible to obtain an inverse mapping from the semantically rich RDF dataset to a specific GraphQL schema. When some statement in de RDF dataset is not mapped, it should also not be available in the GraphQL resultset.

### 9. Map GraphQl schema to GraphQL schema

A GraphQL schema only has meaning within the context of that specific GraphQL schema. When two GraphQL schemas need to be combined, we might ran into problems when two schemas use the same terms. This requirement states that there should be some solution to fix this. For example, by using prefixes or namespaces to distinguish terms from different GraphQL schemas.

### 10. Combine results from different GraphQL schemas

One of the unique features of Linked Data is the possibility to combine data from different sources. The use of universal identifiers in Linked Data vocabularies is the key capability to obtain this feature. Unfortunatelly, a GraphQL schema doesn't have this capability by itself. This requirement states that there should be some solution to fix this. For example, by using the solution for requirement R9 and use the combination of schemas as the combined schema for the combined resultset.

For example, lets example the following two results from two seperate GraphQL endpoints:

```
{
  "account": {
    "number": "+31 555 123 45";
  }
}
```

```
{
  "account": {
    "number": "INGB0001234567";
  }
}
```

A human might recognise the first account number as a telephone number, and the second account number as a bank account number, but to a machine, there are just the same: two string values, identified by the key account.number. The context of the individual GraphQL endpoint is needed to know which number is which. Fortunatelly, using http, both requests are serviced from different endpoints, so the endpoint itself can be used as the identification of context.

If we want to combine these two results, we end up with one endpoint, so we need another mechanism to distinguish the two account numbers. For example, we might use one of the two solutions depicted below (not a requirement, just as an illustration what the solution might look like, other solutions might be possible as well):

**Solution A: prefixing**
```
{
  "tel_account": {
    "tel_number": "+31 555 123 45";
  }
  "bnk_account": {
    "bnk_number": "INGB0001234567";
  }
}
```

**Solution B: grouping**
```
{
  "tel": {
    "account": {
      "number": "+31 555 123 45";
    }
  }
  "bnk": {
    "account": {
      "number": "INGB0001234567";
    }
  }
}
```

### 11. Run-time extension

In most situations, the GraphQL schema is fixed: it will only change when a new version of the GraphQL endpoint is made available. But when we want to combine different datasets, it is very much possible that we include new datasets at run-time, which implies that the combined schema is will also change.

## Implementation for the requirements

Non of the currently available solutions comply to these requirements, but a clever mix of the individual features of these solutions might deliver all of the required functionality.

This solution as the following features:

1. A "regular" API implementation is used to handle the request/response functionality of a regular GraphQL endpoint, to implement requirement R1.
2. Using the SHACL vocabulary, a shapes graph is created for every GraphQL endpoint that maps the RDF vocabulary to the corresponding GraphQL schema. In this way, it is even possible to generate the GraphQL schema from the shapes graph. This shapes graph can be used for the implementation of requirements R2 (context, graphQL to RDF), R6 (graphql to sparql) and R8 (RDF to graphql).
3. Available open source RDF serialization software components are used to transform the JSON-LD resultset to specific RDF serializations, to implement requirement R3. For HTML an extendable open source component is needed, to separate content from presentation.
4. Specific GraphQL directives are added to specify how an `@id` field is constructed for the JSON-LD URI and/or an extra functional property is constructed from the URI, to implement requirements R4 and R5.
5. A `@sparql` directive is added to enable the construction of a SPARQl query from the GraphQL directive. The `@sparql` directive is a complex directive. It has properties as `limit` and `offset` that correspond with the specific parts of a SPARQl query. It also has a `query` property. Without this property, the SPARQL query is generated from the schema. With this property, it is possible to construct the whole query, using all the expressibility of a SPARQL query. The terms follow the GraphQL schema itself, so there is no need for knowledge about the underlying semantics.
6. Specific GraphQL directives can be added, as specified by requirement R7. All these directives are also specified using a shapes graph and corresponding vocabulary. This mitigates the risk that the directives are only usable within the context of this specific GraphQL endpoint. By using a mapping to a RDF vocabulary, a universal directives schema is created. This vocabulary can be added to the shapes graph, so the full GraphQL schema, including directives, can be automatically generated from the shapes graph.
7. As part of the shapes graph is a statement that states the namespace and prefix for any GraphQL schema. This statement is not part of the GraphQL schema itself, because it is needed for the combined GraphQL schema. Any term in the combined query can be contextualised using this prefix, as implementation for requirement R9 and R10.
8. As the shapes graph can be stored in memory or into a SPARQL endpoint (or even a GraphQL endpoint, using the shapes graph for the GraphQL schema itself), the GraphQL schema can be changed at run-time.
