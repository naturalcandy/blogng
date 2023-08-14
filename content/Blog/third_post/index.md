---
title: '"DSL Transformations": A Simple Solution to Hierarchical JSON Schemas?'
date: 2023-07-30T14:00:03-04:00
draft: false
tags: ["Technology", "Programming", "Functional Programming", "Javascript", "TypeScript"]
summary: "Explore the application of DSL transformations for preserving hierarchical rules and metadata in JSON schemas while simultaneously enhancing their usability through structural simplification." 
---

I've recently found myself working with some pretty large and hierarchical JSON Schema data. I quickly came to notice that navigating, updating, or any other form of interaction with schema data felt like getting lost in a convoluted maze—especially when the schemas, ironically, become more complex than the JSON data they're meant to model. Trust me, once you're five levels deep, it's already over for you.

In this post, I'm going to share how you can employ functional programming concepts to create a Domain-Specific Language (DSL). This approach simplifies complex schemas, transforming them into equivalent but less verbose, and far more intuitive structural representations. The end result? Making schemas easier to work with within the development landscape.

{{< lead >}}
From reducing thousands of lines of data to only a few hundred. Now that's pretty nice.
{{< /lead >}}

## What are JSON Schemas?

JSON Schemas are a vocabulary that allows you to annotate and validate JSON documents, offering a contract for the shape and structure of a JSON object. It's like a type system for JSON. The core of this "contract" is described through sets and boolean logic.

### What's their Purpose?

JSON Schemas serve to ensure data integrity and consistency, particularly when exchanging JSON data between systems. By defining a JSON Schema, you can ensure that the data you send or receive has the correct structure and context, which can prevent errors and bugs. But schemas on their own just provide a valuable blueprint for how JSON should be structured. The inclusion of JSON Schema validation libraries are the actual tools for implementing the validation logic that is defined by the JSON Schema. Some examples are, [Ajv](https://github.com/ajv-validator/ajv) and [jsonschema](https://github.com/python-jsonschema/jsonschema). 

### What Does a Simple JSON Schema Look Like?

Let's take a look at this JSON Schema, for example:

```json
{
  "type": "object",
  "properties": {
    "name": { "type": "string" },
    "age": { "type": "integer", "minimum": 0 },
    "city": { "type": "string"},
    "hobbies": { 
      "type": "array",
      "items": { 
        "anyOf": [
          { "type": "string" },
          { "type": "number" }
        ]
      } 
    }
  },
  "required": ["name", "age"]
}
```

This JSON Schema describes an object with these conditions:

* The object contains a `name` property which is a string.
* The object contains an `age` property which is an integer, its minimum value being 0.
* The object contains a `city` property which is a string.
* The object contains a `hobbies` which if present, should be an array. The elements of this array can either be a string or a number.
* The properties name and age must be present. Other properties (`city`, and `hobbies`), are optional.

Any JSON that follows these constraints will be verified by JSON Schema validation libraries.

### What Challenges Arise with JSON Schemas with Nested Data?

JSON Schemas require additional fields to ensure proper validation of JSON data. Consequently, schemas for even seemingly simple JSON objects can become complex when data becomes increasingly hierarchical.

Consider the following JSON object:

```json
{
  "company": {
    "name": "ABC Corp",
    "address": {
      "street": "123 Anywhere St",
      "city": "New York",
      "country": "USA"
    },
    "employees": [
      {
        "firstName": "John",
        "lastName": "Doe",
        "position": "Developer"
      }
    ]
  }
}
```

Pretty straightforward, right? Let's look at its schema now.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "company": {
      "type": "object",
      "properties": {
        "name": { "type": "string", "pattern": "^[A-Za-z0-9 ]+$" },
        "address": {
          "type": "object",
          "properties": {
            "street": { "type": "string", "minLength": 1 },
            "city": { "type": "string", "minLength": 1 },
            "country": { "type": "string", "enum": ["USA", "Canada", "UK"] }
          },
          "required": ["street", "city", "country"]
        },
        "employees": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "firstName": { "type": "string", "minLength": 1 },
              "lastName": { "type": "string", "minLength": 1 },
              "position": { "type": "string", "enum": ["Developer", "Designer", "Manager"] }
            },
            "required": ["firstName", "lastName", "position"]
          },
          "minItems": 1
        }
      },
      "required": ["name", "address", "employees"]
    }
  },
  "required": ["company"]
}
```

*Yikes*. Despite the object's simple structure, the schema is complex due to the need to define each nested level's type, properties, and other additional metadata supported by JSON Schema. And this is just a schema for a JSON that has at most **2** levels of depth. This complexity *drastically* escalates with deeply nested JSON structures. But for your sake, I will omit such examples.

### Why can't we achieve simplification through flattening?

Flattening is a strategy typically used for dealing with hierarchical JSON data itself, but it doesn't address the complexity of the associated JSON Schemas. Unlike regular JSON, JSON Schemas present a unique challenge when it comes to simplification, as they don't represent data but rather the **structure, relationships, and constraints** of that data. This means JSON schemas contain valuable metadata and rules for validation that might be lost or become even more complicated when the data is flattened.

Thus we must find a way to simplify JSON Schemas while also ensuring that the result of our simplification maintains the original validity rules.

## What is the Concept of DSL/DSL Transformations in JSON Schemas?

This is where the concept of a Domain-Specific Language (DSL) comes in. A DSL is a specialized programming language designed to make certain tasks easier or more intuitive for people working within a specific domain. In our case, we're focused on assisting developers, the frequent users of JSON Schemas, by designing a DSL that expresses schemas less verbosely and more intuitively, without sacrificing underlying semantics.

Now I said I dislike dealing with deeply hierarchical data, but computers love that stuff. So we need to introduce something called a DSL transformation to address this. You can think of a DSL transformation as a sort of translator that translates the information being conveyed by machine-readable datastructures (deeply nested illegible JSON Schemas that I never want to see again in this case) to our more human-readable DSL code.

Using a more formal definition, consider you have two sets. Set {{< katex >}} \\(S\\) which represents all of the possible JSON Schemas and another set {{< katex >}} \\(D\\) of our domain-specific language expressions.

A DSL can be seen as a subset of \\(D\\) pertinent to a specific application. Specifically in this application, the DSL represents one simplified version of JSON Schemas.

And a well formed DSL transformation is a function {{< katex >}} \\(f\\) that maps each element in set \\(S\\) (our JSON Schema) to set \\(D\\) (our DSL).

### Can We Apply Functional Programming Concepts in Writing DSL Transformations?

Functional programming, anchored in mathematical functions, is ideal for transforming data due to its predictable and testable nature, precisely what we need for JSON Schemas.

We can leverage [Higher Order Functions](https://en.wikipedia.org/wiki/Higher-order_function) (HOFs), that accept or return other functions, making our code more modular and maintainable. We also preserve and easily add to JSON schema rules by passing metadata as function parameters, presenting a more comprehensible view compared to JSON schema's nested hierarchy.

Moreover, we utilize [lazy evaluation](https://en.wikipedia.org/wiki/Lazy_evaluation#:~:text=In%20programming%20language%20theory%2C%20lazy,by%20the%20use%20of%20sharing), postponing computations until needed, saving resources and enhancing performance with complex data.

As JSON Schema has a recursive structure, akin to multiple function compositions, this approach suits it perfectly. And one thing to note is that the application of functional programming concepts won't inherently "flatten" the schema. Rather, they will provide a different and more intuitive way of defining and manipulating the schema. It abstracts away some of the complexities involved in constructing a JSON schema, especially those that arise from deep nesting.

### How Can DSL Transformation Simplify JSON Schema Structure?

Let's take a look at our previous JSON schema:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "company": {
      "type": "object",
      "properties": {
        "name": { "type": "string", "pattern": "^[A-Za-z0-9 ]+$" },
        "address": {
          "type": "object",
          "properties": {
            "street": { "type": "string", "minLength": 1 },
            "city": { "type": "string", "minLength": 1 },
            "country": { "type": "string", "enum": ["USA", "Canada", "UK"] }
          },
          "required": ["street", "city", "country"]
        },
        "employees": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "firstName": { "type": "string", "minLength": 1 },
              "lastName": { "type": "string", "minLength": 1 },
              "position": { "type": "string", "enum": ["Developer", "Designer", "Manager"] }
            },
            "required": ["firstName", "lastName", "position"]
          },
          "minItems": 1
        }
      },
      "required": ["name", "address", "employees"]
    }
  },
  "required": ["company"]
}
```

Now using a functional programming appraoch we can simplify the JSON Schema into a more digestible representation:

```javascript
const companySchema = objectType({
  company: objectType({
    name: stringType({ pattern: '^[A-Za-z0-9 ]+$' }),
    address: objectType({
      street: stringType({ minLength: 1 }),
      city: stringType({ minLength: 1 }),
      country: stringType({ enum: ['USA', 'Canada', 'UK'] })
    }, ['street', 'city', 'country']),
    employees: arrayType(objectType({
      firstName: stringType({ minLength: 1 }),
      lastName: stringType({ minLength: 1 }),
      position: stringType({ enum: ['Developer', 'Designer', 'Manager'] })
    }, ['firstName', 'lastName', 'position']), { minItems: 1 })
  }, ['name', 'address', 'employees'])
}, ['company']);
```

A lot more easy on the eyes, right? The improved readability becomes more pronounced as we deal with larger and more complex schemas.

Now, let's take a look at the function definitions that help us create this transformed schema:

```javascript
const objectType = (properties, required=[]) => ({
  type: 'object',
  properties: Object.fromEntries(
    Object.entries(properties).map(([key, value]) => [key, value()]),
  ),
  required
});

const arrayType = items => () => ({ type: 'array', items: items() });

const stringType = (metadata = {}) => () => ({ type: 'string', ...metadata });
```

Each of these functions plays a crucial role:

* `objectType`: A HOF that forms an object in the schema using properties and required fields as arguments, mapping directly to nested structures in the original JSON Schema.
* `arrayType`: A HOF that defines an array type, taking a function as an argument to specify the item type, echoing the array definitions in the original schema.
* `stringType`: A HOF representing  the string type, accepting metadata as an argument to enhance schema context, analogous to basic type definitions in the original schema.

{{< alert "check" >}}
And there you have it! Our own DSL representation of JSON Schemas.
{{< /alert >}}

While being very limited, this example showcases a functional programming approach for representing schemas in a stucturally streamlined way, while presserving the underlying data. And to access the original schema? Simply invoke `companySchema()` using `()` as our placeholder argument. This smart move lets us harness the power of lazy evaluation.

While this example is limited, it showcases how the declarative, modular nature of this approach empowers us to simplify and clarify schema definitions, thereby enhancing understandability, flexibility, and testability. The functions, acting as building blocks, make it straightforward to build complex schemas as they mirror the structure of the schema itself, from primitive to recursive types.