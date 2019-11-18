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

The representation in the previous section uses blank nodes to declare propertyshapes. It is also possible to used IRI's for the propertyshapes.

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

In this case, it is not necessary to specify the name of the field, as -by default- the name will be inferred from the local name of the IRI. Mark that you can reuse field specifications in this way, but every field must have the same specifications! You could also combine both approaches, even use a `graphql:name` statement for the nodeshape. For example, the following shapes represent the same GraphQL type scheme.

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
- When the propertyshape is identified by an IRI, use the localname of the IRI, or else...
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

This construction can only be used ones for every class, other shapes that target the same class should use the class typing construction.

#### 2. Class targeting

By using class targeting, you can distinguish the namespace for the shape and the namespace for the class vocabulary. This enables the reuse of existing class vocabularies.

```
ex:Character a sh:NodeShape;
  sh:targetClass film:Character;
  sh:property ex:name;
  sh:property ex:appearsIn;
.
```

This example reused the `film:Character` class from some film vocabulary as the target for the character shape. As the former construction, this construction can also only be used ones for every class, other shapes that target the same class should use the class typing construction.

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
- Use the localname of the IRI, or else...
- Raise an error: no suitable name for the field was found.
