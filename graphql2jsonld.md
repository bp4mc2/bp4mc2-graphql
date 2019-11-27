# GraphQL result to JSON-LD

By adding `@context`, any JSON GraphQL can become a semantically rich result.

Let's have a look at the following GraphQL query:

```
query FavoriteHero {
  hero {
    name
    appearsIn
  }
}
```

Every GraphQL result starts with a root object, which represented by the "data" object in the corresponding result. This root object can have multiple fields, in this case the "hero" field.

```
{
  "data": {
    "hero": {
      "name": "R2-D2",
      "appearsIn": [
        "NEWHOPE",
        "EMPIRE",
        "JEDI"
      ]
    }
  }
}
```

The root object is actually the result of the FavoriteHero query, so we can specify the query by the following shape:

```
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#>.
@prefix film: <http://example.org/def/film#>.
ex:FavoriteHero a sh:NodeShape;
  sh:property [
    graphql:name "hero";
    sh:path film:hero;
    sh:node ex:Character;
  .
.
ex:Character a sh:NodeShape;
  sh:property [
    graphql:name "name";
    sh:path rdfs:label;
    sh:datatype xsd:string;
  ];
  sh:property [
    graphql:name "appearsIn";
    sh:path film:appearsIn;
    sh:node ex:Episode;
  ];
.
```

From this shape, we can create the `@context`:

```
"@context": {
  "rdfs": "http://www.w3.org/2000/01/rdf-schema#",
  "film": "http://example.org/def/film#",
  "hero": "film:hero",
  "name": "rdfs:label",
  "appearsIn": "film:appearsIn",
  "data": "@graph"
}
```

- The first two context fields define prefixes. This is not really necessary, but gives a nicer to read context.
- The `hero`, `name` and `appearsIn` fields are taken directly from the corresponding property shapes: the name from `graphql:name` and the URI of the corresponding property from `sh:path`.
- The last context field is needed to state that the "data" field contains RDF data.

When we add this context to the original JSON GraphQL response (as part of the header, to maintain the original http body), we get the following triples:

```
_:b0 <http://example.org/def/film#hero> _:b1 .
_:b1 <http://www.w3.org/2000/01/rdf-schema#label> "R2-D2" .
_:b1 <http://example.org/def/film#appearsIn> "NEWHOPE" .
_:b1 <http://example.org/def/film#appearsIn> "EMPIRE" .
_:b1 <http://example.org/def/film#appearsIn> "JEDI" .
```

So we get RDF data. But not very useful RDF data, because no URI's are available for the resources themselves. Only blank nodes are visible, and we might also want to make resources for the films that our hero appears in.

This can be achieved by adding some more directives to the SHACL shapes:

```
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#>.
@prefix film: <http://example.org/def/film#>.
ex:FavoriteHero a sh:NodeShape;
  sh:property [
    graphql:name "hero";
    sh:path film:hero;
    sh:node ex:Character;
  .
.
ex:Character a sh:NodeShape;
  graphql:uriTemplate "http://example.org/id/character/{$name}";
  sh:property [
    graphql:name "name";
    sh:path rdfs:label;
    sh:datatype xsd:string;
  ];
  sh:property [
    graphql:name "appearsIn";
    sh:path film:appearsIn;
    sh:node ex:Episode;
    graphql:uriTemplate "http://example.org/id/episode/{$appearsIn}";
  ];
.
```

With two `graphql:uriTemplate` statements, we can direct the converter to mint URI's. No URI template is needed for the `ex:FavoriteHero` shape. As this shape is the root of the response, it seems logical that the URI for this resource is the URL of the request itself. When this is not wanted, we might add another `graphql:uriTemplate`.

The new directives changes the `@context` part at one place: the "appearsIn" doesn't have literal values any more, but URI's. The directive adds some extra statements to the GraphQL result, whenever JSON-LD is requested.

Remember that this response is a serialization of a GraphQL request, but doesn't fully comply to the GraphQL specification as it includes a `@context` element in the root of the response and the `@id` field doesn't comply the the GraphQL specification for fieldnames: `/[_A-Za-z][_0-9A-Za-z]*/`). This could be fixed to use another name for the `@id` field (for example: `_self`) and include this field in the context as well. The context itself should be part of the header to fully comply to the GraphQL specification.

```
{
  "@context": {
    "rdfs": "http://www.w3.org/2000/01/rdf-schema#",
    "film": "http://example.org/def/film#",
    "hero": "film:hero",
    "name": "rdfs:label",
    "appearsIn": {"@id": "film:appearsIn", "@type": "@id"},
    "data": "@graph"
  },
  "data": {
    "@id": "",
    "hero": {
      "@id": "http://example.org/id/character/R2-D2",
      "name": "R2-D2",
      "appearsIn": [
        "http://example.org/id/episode/NEWHOPE",
        "http://example.org/id/episode/EMPIRE",
        "http://example.org/id/episode/JEDI"
      ]
    }
  }
}
```

Resulting into the following triples:

```
<https://example.org/graphql?query=hero> <http://example.org/def/film#hero> <http://example.org/id/character/R2-D2> .
<http://example.org/id/character/R2-D2> <http://www.w3.org/2000/01/rdf-schema#label> "R2-D2" .
<http://example.org/id/character/R2-D2> <http://example.org/def/film#appearsIn> <http://example.org/id/episode/NEWHOPE> .
<http://example.org/id/character/R2-D2> <http://example.org/def/film#appearsIn> <http://example.org/id/episode/EMPIRE> .
<http://example.org/id/character/R2-D2> <http://example.org/def/film#appearsIn> <http://example.org/id/episode/JEDI> .
```
