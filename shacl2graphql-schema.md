# Mapping shacl shapes to graphql schema and back

## Object types and fields

### Minimal representation

The most basic components of a GraphQL schema are object types, which just represent a kind of object you can fetch from your service, and what fields it has. In the GraphQL schema language, we might represent it like this:

```
type Character {
  name: String!
  appearsIn: [Episode!]!
}
```

The *minimal* corresponding shape looks like:

```
ex:Character a sh:NodeShape;
  sh:property [
    graphql:name "name"
    sh:minCount 1;
    sh:maxCount 1;
    sh:datatype xsd:string;
  ];
  sh:property [
    graphql:name "appearsIn";
    sh:node ex:Episode;
  ];
.
```

The [graphql vocabulary](http://datashapes.org/graphql) is used to annotate a SHACL shapesgraph with directions how to represent a GraphQL scheme in RDF.

- `Character` is a GraphQL Object Type, meaning it's a type with some fields. Most of the types in your schema will be object types. Every Object Type is represented by a SHACL NodeShape.
- `name` and `appearsIn` are fields on the `Character` type. That means that `name` and `appearsIn` are the only fields that can appear in any part of a GraphQL query that operates on the `Character` type. Every field is represented by a SHACL Propertyshape, that is linked to the SHACL NodeShape via the `sh:property` predicate.
- `String` is on of the build-in scalar types - these types are types that resolve to a single scalar object, and can't have sub-selections in the query. The string type is represented by the `xsd:string` RDF datatype.
- `String!` means that the field is non-nullable, meaning that the GraphQL service promises to always give you a value when you query this field. This constraint is represented by the `sh:minCount 1` RDF statement.
- `[Episode]` represents an array of `Episode` objects. Since it is also non-nullable, you can always expect an array (with zero or more items) when you query the `appearsIn` field. And since `Episode!` is also non-nullable, you can always expect every item of the array to be an `Episode` object. In RDF this is represented not by declaring something as an array, but instead declare a maxCount. So the `name` field is declared as having a `sh:maxCount 1`, and the `appearsIn` field is not limited to any maxCount, thus representing an array. To represent that the items should be episodes, the statement `sh:node ex:Episode` is used.

### Explicitly defined propertyshapes, naming

The representation in the previous section uses blank nodes to declare propertyshapes. It is also possible to used URI's for the propertyshapes.

```
ex:Character a sh:NodeShape;
  sh:property ex:name;
  sh:property ex:appearsIn;
.
ex:name a sh:PropertyShape;
  sh:minCount 1;
  sh:maxCount 1;
  sh:datatype xsd:string;
.
ex:appearsIn a sh:PropertyShape;
  sh:node ex:Episode;
.
```

In this case, it is not necessary to specify the name of the field, as -by default- the name will be inferred from the local name of the URI. Mark that you can reuse field specifications in this way, but every field must have the same specifications! You could also combine both approaches, even use a `graphql:name` statement for the nodeshape. For example, the following shapes represent the same GraphQL type scheme.

```
ex:MyCharacter a sh:NodeShape;
  graphql:name "Character";
  sh:property ex:name;
  sh:property ex:Character_appearsIn;
.
ex:name a sh:PropertyShape;
  sh:minCount 1;
  sh:maxCount 1;
  sh:datatype xsd:string;
.
ex:Character_appearsIn a sh:PropertyShape;
  graphql:name "appearsIn";
  sh:node ex:Episode;
.
```

### Property linkage

The shape representation only specifies the GraphQL scheme itself. It doesn't link the representation to an meaningful vocabulary. It has just the same, closed world properties as the original GraphQL scheme. To achieve the benefit of a semantic scheme, we need to link the shape representation to a corresponding RDF vocabulary.

For properties, this can be easily achieved by adding a `sh:path` statement to the shapes declaration. Let's say that the `name` field corresponds to a `rdfs:label` predicate, and the `appearsIn` field corresponds to an `film:appearsIn` predicate, from some own-defined `film` vocabulary.

```
ex:Character a sh:NodeShape;
  sh:property [
    graphql:name "name";
    sh:path rdfs:label;
    sh:minCount 1;
    sh:maxCount 1;
    sh:datatype xsd:string;
  ];
  sh:property [
    sh:path film:appearsIn;
    sh:node ex:Episode;
  ];
.
```

### Getting the name of a GraphQL field

As you might have noticed, in this case, it is not needed to specify a `graphql:name` declaration for the appearsIn field. When a `graphql:name` is not specified, the local name of the `sh:path` object is used. The `graphql:name` declaration is still needed for the name field, as the local name of the `sh:path` object is different (`label` instead of `name`).

To get the name of a GraphQL field the following rules are used:
- When a `graphql:name` declaration is available, use that one, or else...
- When a `sh:path` declaration is available, use the localname of the object, or else...
- When the propertyshape is identified by an URI, use the localname of the URI, or else...
- Raise an error: no suitable name for the field was found.

### Class linkage

Class linkage is a bit harder to achieve. The reason is that a class has meaning outside of the GraphQL scheme. Different specifications for the same class might occur, even within the same GraphQL schema. For this reason, three separate constructions are available to link a class to a specific Object type:

1. Punning of shape and class;
2. Class targetting;
3. Class typing.

#### 1. Punning of a shape and class

The easiest way of class linking is by way of punning. This means that we declare that some shape is a `sh:NodeShape` as well as an `owl:Class`.

```
ex:Character a sh:NodeShape, owl:Class;
  sh:property ex:name;
  sh:property ex:appearsIn;
.
```

The downside of this construction, is that the URI of the class is the same as the URI for the shape, so we can't reuse existing class vocabularies. This construction can only be used ones for every class, other shapes that target the same class should use the class typing construction.

#### 2. Class targeting

By using class targeting, we can distinguish the namespace for the shape and the namespace for the class vocabulary. This enables the reuse of existing class vocabularies.

```
ex:Character a sh:NodeShape;
  sh:targetClass film:Character;
  sh:property ex:name;
  sh:property ex:appearsIn;
.
```

This example reused the `film:Character` class from some film vocabulary as the target for the character shape. As with the former construction, this construction can also only be used ones for every class, other shapes that target the same class should use the class typing construction.

#### 3. Class typing

Distinguishing the namespace for the shape and class vocabulary can also be achieved by using class typing. In such a case, we specify that any resource that conforms to this particular shape, should also be typed as a specific class:

```
ex:Character a sh:NodeShape;
  sh:property [
    sh:path rdf:type;
    sh:hasValue film:Character;
    sh:minCount 1;
  ];
  sh:property ex:name;
  sh:property ex:appearsIn;
.
```

To distinguish this construction from a regular field, the propertyshape should always be blank node and not have a `graphql:name` statement.

### Getting the name of a GraphQL object type

To get the name of a GraphQL object type the following rules are used:
- When a `graphql:name` declaration is available, use that one, or else...
- When a `sh:targetClass` declaration is available, use the localname of the object, or else...
- Use the localname of the URI, or else...
- Raise an error: no suitable name for the field was found.

## Scalar types
A GraphQL object type has a name and fields, but at some point those fields have to resolve to some concrete data. That's where the scalar types come in: they represent the leaves of the query.

In the following GraphQL result, the `name` and `appearsIn` fields will resolve to scalar types:

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

GraphQL comes with a set of default scalar types out of the box:

- `Int`: A signed 32-bit integer. It is represented with the `xsd:int` datatype;
- `Float`: A signed double-precision floating-point value. It is represented with the `xsd:float` datatype;
- `String`: A UTF-8 character sequence. It is represented with the `xsd:string` datatype;
- `Boolean`: `true` or `false`. It is represented by the `xsd:boolean` datatype;
- `ID`: The ID scalar type represents a unique identifier, often used to refetch an object or as the key for a cache. The ID type is serialized in the same way as a String; however, defining it as an ID signifies that it is not intended to be human‐readable. It is represented by the `xsd:string` datatype.

### Custom scalar types

In most GraphQL service implementations, there is a way to specify custom scalar types. Some RDF datatypes (notably xsd:date and xsd:dateTime) are not available oht of the box, so we need a way of representing these custom scalar types as "regular" RDF datatypes. For example, let's have a custom data scalar type.

```
scalar Data
```

To represent this custom scalar type with the regular RDF xsd:date datatype, we specifiy the following:

```
ex:Data a graphql:ScalarType;
  sh:datatype xsd:date;
.
```

As with Object Types, the name of the scalar type is infered from the localname, or from an explicitly specified graphql:name statement.

### Enumeration types

Also called Enums, enumeration types are a special kind of scalar that is restricted to a particular set of allowed values. An enum definition might look like:

```
enum Episode {
  NEWHOPE
  EMPIRE
  JEDI
}
```

The corresponding shape representation of an enum type looks like:

```
ex:Episode a sh:NodeShape;
  sh:in "NEWHOPE", "EMPIRE", "JEDI"
.
```

## URI's construction and ID's
The ID scalar type represents a unique identifier, often used to refetch an object or as the key for a cache. The ID type is serialized in the same way as a String; however, it is not intended to be human‐readable. While it is often numeric, it should always serialize as a String.

URI's refer to resources as ID's might refer to objects. However, ID's do not always comply to the URI format, and it is not necessary to use ID's as keys in a GraphQL schema. The construction or deconstruction of a URI might be needed.

Let's for example look at the following GraphQL result:

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

Some comments can be made with regard to this response:

- The hero has a name, but not an ID;
- The hero appears in three episodes, but we only have a string value that refers to the episode, but not an ID.

So we ask ourselves:
- Which R2-D2 are we talking about?? (maybe [this](https://wikipedia.org/wiki/R2-D2) one, or [this](https://www.lego.com/biassets/bi/6003560.pdf) one, or...).
- Which EMPIRE are we talking about?? The film [The empire strikes back](https://wikipedia.org/wiki/The_Empire_Strikes_Back), or the Atari 2600 [The empire strikes back](https://wikipedia.org/wiki/Star_Wars:_The_Empire_Strikes_Back_(1982_video_game)) game, or the [British Empire](https://wikipedia.org/wiki/British_Empire), or...).

By using the `graphql:uriTemplate` declaration, we can add context to the names and codes from the GraphQL result and deliver real identifiers:

```
ex:Character a sh:NodeShape;
  graphql:uriTemplate "http://example.org/id/hero/{$name}";
  sh:property [
    graphql:name "name"
    sh:minCount 1;
    sh:maxCount 1;
    sh:datatype xsd:string;
  ];
  sh:property [
    graphql:name "appearsIn";
    sh:node ex:Episode;
    graphql:uriTemplate "http://example.org/id/episode/{$appearsIn}";
  ];
.
ex:Episode a sh:NodeShape;
  sh:in "NEWHOPE", "EMPIRE", "JEDI"
  graphql:uriTemplate "http://example.org/id/episode/{$this}";
.
```

For completeness, the `ex:Episode` enum scalar type is also added. In this particular GraphQL scheme, the ex:Episode is not defined as an Object Type, but as a enum scalar type.

For every nodeshape template, an `@id` field is added. For every propertyshape template, a scalar type is transformed to a URI, in case of an object type, an `@id` field is added.

```
{
  "data": {
    "hero": {
      "@id": "http://example.org/id/hero/R2-D2"
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

But what if the Episode type was defined as an Object type, for example:

```
type Episode {
  code: ID!
  name: String
}
```

An the corresponding shapes declaration:

```
ex:Episode a sh:NodeShape;
  graphql:uriTemplate "http://example.org/id/episode/{$code}";
  sh:property [
    graphql:name "code";
    sh:minCount 1;
    sh:maxCount 1;
    sh:datatype xsd:string;
    graphql:isIDField true;
  ];
  sh:property [
    graphql:name "name";
    sh:maxCount 1;
    sh:datatype xsd:string;
  ];
.
```

The corresponding data might look like:

```
{
  "data": {
    "hero": {
      "@id": "http://example.org/id/hero/R2-D2"
      "name": "R2-D2",
      "appearsIn": [
        {
          "@id": "http://example.org/id/episode/NEWHOPE",
          "code": "NEWHOPE",
          "name": "Star Wars Episode IV: A New Hope"
        },
        {
          "@id": "http://example.org/id/episode/EMPIRE",
          "code": "EMPIRE",
          "name": "Star Wars Episode V: The Empire Strikes Back"
        },
        {
          "@id": "http://example.org/id/episode/JEDI",
          "code": "JEDI",
          "name": "Star Wars Episode VI: Return Of The Jedi"
        }
      ]
    }
  }
}
```

A field with an `ID` scalar type corresponds to a regular property shape, with the addition of a `graphql:isIDField` statement. The field is actually not treated differently in the RDF representation, because the actual URI is generated from the `graphql:uriTemplate` field.

## URI's deconstruction and ID's

When we deal with RDF data that is delivered using a graphQL scheme, we might want to pass through the URI value itself. Because it doesn't correspond with an actual property, the URI value wouldn't be represented by a GraphQL field.

By adding directions to the shacl shapes, we can direct if, and how, the URI value is passed through:

- When nothing is added, the `@id` field corresponds to the URI value, but is only added whenever a JSON-LD response is requested;
- When a propertyshape is added with `graphql:name "@id"`, this field corresponds to the URI value and is always added;
- When a propertyshape is added with `graphql:uriTemplate` and the template contains the name of the field and a `sh:path` directive is missing, the value is inferred from the URI (This makes it possible to create a custom named identifier field).

Examples:

```
# @id field is only added in case of JSON-LD
ex:Character a sh:NodeShape;
.
```

```
# @id field is always added
ex:Character a sh:NodeShape;
  sh:property [
    graphql:name "@id";
  ]
.
```

```
# an code field is added, with the local name of the URI
ex:Episode a sh:NodeShape;
  sh:property [
    graphql:name "code";
    graphql:uriTemplate "http://example.org/id/episode/{$code}"
  ];
.
```

In RDF, links to other resources are always URI's. In GraphQL results, plain strings might be more appropriate (as might be the case with enum types). To deconstruct a URI, the graphql:uriTemplate is used (but in the inverse direction):

```
# the appearsIn URI value is deconstructed into a string value
ex:Character a sh:NodeShape;
  sh:property [
    sh:path film:appearsIn;
    sh:node ex:Episode;
    graphql:uriTemplate "http://example.org/id/episode/{$appearsIn}";
  ];
.
```

## Linking a schema to a shapes graph
