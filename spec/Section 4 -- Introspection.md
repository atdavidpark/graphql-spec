# Introspection

A GraphQL service supports introspection over its schema. This schema is queried
using GraphQL itself, creating a powerful platform for tool-building.

Take an example request for a trivial app. In this case there is a User type with
three fields: id, name, and birthday.

For example, given a service with the following type definition:

```graphql example
type User {
  id: String
  name: String
  birthday: Date
}
```

A request containing the operation:

```graphql example
{
  __type(name: "User") {
    name
    fields {
      name
      type {
        name
      }
    }
  }
}
```

would produce the result:

```json example
{
  "__type": {
    "name": "User",
    "fields": [
      {
        "name": "id",
        "type": { "name": "String" }
      },
      {
        "name": "name",
        "type": { "name": "String" }
      },
      {
        "name": "birthday",
        "type": { "name": "Date" }
      }
    ]
  }
}
```

## Reserved Names

Types and fields required by the GraphQL introspection system that are used in
the same context as user-defined types and fields are prefixed with {"__"} two
underscores. This in order to avoid naming collisions with user-defined GraphQL
types. Conversely, GraphQL type system authors must not define any types,
fields, arguments, or any other type system artifact with two leading
underscores.


## Documentation

All types in the introspection system provide a `description` field of type
`String` to allow type designers to publish documentation in addition to
capabilities. A GraphQL service may return the `description` field using Markdown
syntax (as specified by [CommonMark](https://commonmark.org/)). Therefore it is
recommended that any tool that displays `description` use a CommonMark-compliant
Markdown renderer.


## Deprecation

To support the management of backwards compatibility, GraphQL fields and enum
values can indicate whether or not they are deprecated (`isDeprecated: Boolean`)
and a description of why it is deprecated (`deprecationReason: String`).

Tools built using GraphQL introspection should respect deprecation by
discouraging deprecated use through information hiding or developer-facing
warnings.


## Type Name Introspection

GraphQL supports type name introspection within any selection set in an
operation, with the single exception of selections at the root of a subscription
operation. Type name introspection is accomplished via the meta-field
`__typename: String!` on any Object, Interface, or Union. It returns the name of
the concrete Object type at that point during execution.

This is most often used when querying against Interface or Union types to
identify which actual Object type of the possible types has been returned.

As a meta-field, `__typename` is implicit and does not appear in the fields list
in any defined type.


## Schema Introspection

The schema introspection system is accessible from the meta-fields `__schema`
and `__type` which are accessible from the type of the root of a query
operation.

```graphql
__schema: __Schema!
__type(name: String!): __Type
```

Like all meta-fields, these are implicit and do not appear in the fields list in
the root type of the query operation.

The schema of the GraphQL schema introspection system:

```graphql
type __Schema {
  description: String
  types: [__Type!]!
  queryType: __Type!
  mutationType: __Type
  subscriptionType: __Type
  directives: [__Directive!]!
}

type __Type {
  kind: __TypeKind!
  name: String
  description: String

  # should be non-null for OBJECT and INTERFACE only, must be null for the others
  fields(includeDeprecated: Boolean = false): [__Field!]

  # should be non-null for OBJECT and INTERFACE only, must be null for the others
  interfaces: [__Type!]

  # should be non-null for INTERFACE and UNION only, always null for the others
  possibleTypes: [__Type!]

  # should be non-null for ENUM only, must be null for the others
  enumValues(includeDeprecated: Boolean = false): [__EnumValue!]

  # should be non-null for INPUT_OBJECT only, must be null for the others
  inputFields: [__InputValue!]

  # should be non-null for NON_NULL and LIST only, must be null for the others
  ofType: __Type

  # should be non-null for custom SCALAR only, must be null for the others
  specifiedBy: String
}

type __Field {
  name: String!
  description: String
  args: [__InputValue!]!
  type: __Type!
  isDeprecated: Boolean!
  deprecationReason: String
}

type __InputValue {
  name: String!
  description: String
  type: __Type!
  defaultValue: String
}

type __EnumValue {
  name: String!
  description: String
  isDeprecated: Boolean!
  deprecationReason: String
}

enum __TypeKind {
  SCALAR
  OBJECT
  INTERFACE
  UNION
  ENUM
  INPUT_OBJECT
  LIST
  NON_NULL
}

type __Directive {
  name: String!
  description: String
  locations: [__DirectiveLocation!]!
  args: [__InputValue!]!
  isRepeatable: Boolean!
}

enum __DirectiveLocation {
  QUERY
  MUTATION
  SUBSCRIPTION
  FIELD
  FRAGMENT_DEFINITION
  FRAGMENT_SPREAD
  INLINE_FRAGMENT
  VARIABLE_DEFINITION
  SCHEMA
  SCALAR
  OBJECT
  FIELD_DEFINITION
  ARGUMENT_DEFINITION
  INTERFACE
  UNION
  ENUM
  ENUM_VALUE
  INPUT_OBJECT
  INPUT_FIELD_DEFINITION
}
```


### The __Type Type

`__Type` is at the core of the type introspection system, it represents all
types in the system: both named types (e.g. Scalars and Object types) and
type modifiers (e.g. List and Non-Null types).

Type modifiers are used to modify the type presented in the field `ofType`.
This modified type may recursively be a modified type, representing lists,
non-nullables, and combinations thereof, ultimately modifying a named type.

### Type Kinds

There are several different kinds of type. In each kind, different fields are
actually valid. These kinds are listed in the `__TypeKind` enumeration.


#### Scalar

Represents scalar types such as Int, String, and Boolean. Scalars cannot have fields.

Also represents [Custom scalars](#sec-Scalars.Custom-Scalars) which may provide 
`specifiedBy` as a scalar specification URL.

Fields

* `kind` must return `__TypeKind.SCALAR`.
* `name` must return a String.
* `specifiedBy` may return a String (in the form of a URL) for custom scalars, 
   otherwise must be {null}.
* `description` may return a String or {null}.
* All other fields must return {null}.


#### Object

Object types represent concrete instantiations of sets of fields. The
introspection types (e.g. `__Type`, `__Field`, etc) are examples of objects.

Fields

* `kind` must return `__TypeKind.OBJECT`.
* `name` must return a String.
* `description` may return a String or {null}.
* `fields`: The set of fields that can be selected for this type.
  * Accepts the argument `includeDeprecated` which defaults to {false}. If
    {true}, deprecated fields are also returned.
* `interfaces`: The set of interfaces that an object implements.
* All other fields must return {null}.


#### Union

Unions are an abstract type where no common fields are declared. The possible
types of a union are explicitly listed out in `possibleTypes`. Types can be
made parts of unions without modification of that type.

Fields

* `kind` must return `__TypeKind.UNION`.
* `name` must return a String.
* `description` may return a String or {null}.
* `possibleTypes` returns the list of types that can be represented within this
  union. They must be object types.
* All other fields must return {null}.


#### Interface

Interfaces are an abstract type where there are common fields declared. Any type
that implements an interface must define all the fields with names and types
exactly matching. The implementations of this interface are explicitly listed
out in `possibleTypes`.

Fields

* `kind` must return `__TypeKind.INTERFACE`.
* `name` must return a String.
* `description` may return a String or {null}.
* `fields`: The set of fields required by this interface.
  * Accepts the argument `includeDeprecated` which defaults to {false}. If
    {true}, deprecated fields are also returned.
* `interfaces`: The set of interfaces that this interface implements.
* `possibleTypes` returns the list of types that implement this interface.
  They must be object types.
* All other fields must return {null}.


#### Enum

Enums are special scalars that can only have a defined set of values.

Fields

* `kind` must return `__TypeKind.ENUM`.
* `name` must return a String.
* `description` may return a String or {null}.
* `enumValues`: The list of `EnumValue`. There must be at least one and they
  must have unique names.
  * Accepts the argument `includeDeprecated` which defaults to {false}. If
    {true}, deprecated enum values are also returned.
* All other fields must return {null}.


#### Input Object

Input objects are composite types defined as a list of named input values. They
are only used as inputs to arguments and variables and cannot be a field
return type.

For example the input object `Point` could be defined as:

```graphql example
input Point {
  x: Int
  y: Int
}
```

Fields

* `kind` must return `__TypeKind.INPUT_OBJECT`.
* `name` must return a String.
* `description` may return a String or {null}.
* `inputFields`: a list of `InputValue`.
* All other fields must return {null}.


#### List

Lists represent sequences of values in GraphQL. A List type is a type modifier:
it wraps another type instance in the `ofType` field, which defines the type of
each item in the list.

The modified type in the `ofType` field may itself be a modified type, allowing
the representation of Lists of Lists, or Lists of Non-Nulls.

Fields

* `kind` must return `__TypeKind.LIST`.
* `ofType`: Any type.
* All other fields must return {null}.


#### Non-Null

GraphQL types are nullable. The value {null} is a valid response for field type.

A Non-Null type is a type modifier: it wraps another type instance in the
`ofType` field. Non-null types do not allow {null} as a response, and indicate
required inputs for arguments and input object fields.

The modified type in the `ofType` field may itself be a modified List type,
allowing the representation of Non-Null of Lists. However it must not be a
modified Non-Null type to avoid a redundant Non-Null of Non-Null.

* `kind` must return `__TypeKind.NON_NULL`.
* `ofType`: Any type except Non-Null.
* All other fields must return {null}.


### The __Field Type

The `__Field` type represents each field in an Object or Interface type.

Fields

* `name` must return a String
* `description` may return a String or {null}
* `args` returns a List of `__InputValue` representing the arguments this
  field accepts.
* `type` must return a `__Type` that represents the type of value returned by
  this field.
* `isDeprecated` returns {true} if this field should no longer be used,
  otherwise {false}.
* `deprecationReason` optionally provides a reason why this field is deprecated.


### The __InputValue Type

The `__InputValue` type represents field and directive arguments as well as the
`inputFields` of an input object.

Fields

* `name` must return a String
* `description` may return a String or {null}
* `type` must return a `__Type` that represents the type this input
  value expects.
* `defaultValue` may return a String encoding (using the GraphQL language) of the
  default value used by this input value in the condition a value is not
  provided at runtime. If this input value has no default value, returns {null}.

### The __EnumValue Type

The `__EnumValue` type represents one of possible values of an enum.

Fields

* `name` must return a String
* `description` may return a String or {null}
* `isDeprecated` returns {true} if this enum value should no longer be used,
  otherwise {false}.
* `deprecationReason` optionally provides a reason why this enum value is deprecated.

### The __Directive Type

The `__Directive` type represents a Directive that a service supports.

Fields

* `name` must return a String
* `description` may return a String or {null}
* `locations` returns a List of `__DirectiveLocation` representing the valid
  locations this directive may be placed.
* `args` returns a List of `__InputValue` representing the arguments this
  directive accepts.
* `isRepeatable` must return a Boolean that indicates if the directive may be
  used repeatedly at a single location.
