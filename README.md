# [Ednsquare](https://www.ednsquare.com) API Design Guidelines

# Tutorial: Designing API

 
## Intro

Welcome! This document will walk you through designing a new GraphQL API (or a
new piece of an existing GraphQL API). API design is a challenging
task that strongly rewards iteration, experimentation, and a thorough
understanding of your business domain.
 
 
 
 ## Step Zero: Background

For the purposes of this tutorial, imagine you work at an e-commerce company.
You have an existing GraphQL API exposing information about your products, but
very little else. However, your team just finished a project implementing
"collections" in the back-end and wants to expose collections over the API as
well.

Collections are the new go-to method for grouping products; for example, you
might have a collection of all of your t-shirts. Collections can be used for
display purposes when browsing your website, and also for programmatic tasks
(e.g. you may want to make a discount only apply to products in a certain
collection).

On the back-end, your new feature has been implemented as follows:
- All collections have some simple attributes like a title, a description body
  (which may include HTML formatting), and an image.
- You have two specific kinds of collections: "manual" collections where you
  list the products you want them to include, and "automatic" collections where
  you specify some rules and let the collection populate itself.
- Since the product-to-collection relationship is many-to-many, you've got a
  join table in the middle called `CollectionMembership`.
- Collections, like products before them, can be either published (visible on
  the storefront) or not.

With this background, you're ready to start thinking about your API design.


## Step One: A Bird's-Eye View

A naive version of the schema might look something like this (leaving out all
the pre-existing types like `Product`):
```graphql
interface Collection {
  id: ID!
  memberships: [CollectionMembership!]!
  title: String!
  imageId: ID
  bodyHtml: String
}

type AutomaticCollection implements Collection {
  id: ID!
  rules: [AutomaticCollectionRule!]!
  rulesApplyDisjunctively: Bool!
  memberships: [CollectionMembership!]!
  title: String!
  imageId: ID
  bodyHtml: String
}

type ManualCollection implements Collection {
  id: ID!
  memberships: [CollectionMembership!]!
  title: String!
  imageId: ID
  bodyHtml: String
}

type AutomaticCollectionRule {
  column: String!
  relation: String!
  condition: String!
}

type CollectionMembership {
  collectionId: ID!
  productId: ID!
}
```

This is already decently complicated at a glance, even though it's only four
objects and an interface. It also clearly doesn't implement all of the features
that we would need if we're going to be using this API to build out e.g. our
mobile app's collection feature.

Let's take a step back. A decently complex GraphQL API will consist of many
objects, related via multiple paths and with dozens of fields. Trying to design
something like this all at once is a recipe for confusion and mistakes. Instead,
you should start with a higher-level view first, focusing on just the types and
their relations without worrying about specific fields or mutations.
Basically think of an [Entity-Relationship model](https://en.wikipedia.org/wiki/Entity%E2%80%93relationship_model)
but with a few GraphQL-specific bits thrown in. If we shrink our naive schema
down like that, we end up with the following:

```graphql
interface Collection {
  Image
  [CollectionMembership]
}

type AutomaticCollection implements Collection {
  [AutomaticCollectionRule]
  Image
  [CollectionMembership]
}

type ManualCollection implements Collection {
  Image
  [CollectionMembership]
}

type AutomaticCollectionRule { }

type CollectionMembership {
  Collection
  Product
}
```

To get this simplified representation, I took out all scalar fields, all field
names, and all nullability information. What you're left with still looks kind
of like GraphQL but lets you focus on higher level of the types and their
relationships.

``` *Rule #1: Always start with a high-level view of the objects and their
relationships before you deal with specific fields.* ```


## Step Two: A Clean Slate

Now that we have something simple to work with, we can address the major flaws
with this design.

As previously mentioned, our implementation defines the existence of manual and
automatic collections, as well as the use of a collector join table. Our naive
API design was clearly structured around our implementation, but this was a
mistake.

The root problem with this approach is that an API operates for a different
purpose than an implementation, and frequently at a different level of
abstraction. In this case, our implementation has led us astray on a number of
different fronts.

### Representing `CollectionMembership`s

The one that may have stood out to you already, and is hopefully fairly obvious,
is the inclusion of the `CollectionMembership` type in the schema. The collection memberships table is
used to represent the many-to-many relationship between products and collections.
Now read that last sentence again: the relationship is *between products and
collections*; from a semantic, business domain perspective, collection memberships have
nothing to do with anything. They are an implementation detail.

This means that they don't belong in our API. Instead, our API should expose the
actual business domain relationship to products directly. If we take out
collection memberships, the resulting high-level design now looks like:

```graphql
interface Collection {
  Image
  [Product]
}

type AutomaticCollection implements Collection {
  [AutomaticCollectionRule]
  Image
  [Product]
}

type ManualCollection implements Collection {
  Image
  [Product]
}

type AutomaticCollectionRule { }
```

This is much better.

``` *Rule #2: Never expose implementation details in your API design.* ```

### Representing Collections

This API design still has one major flaw, though it's one that's probably much
less obvious without a really thorough understanding of the business domain. In
our existing design, we model AutomaticCollections and ManualCollections as two
different types, each implementing a common Collection interface. Intuitively
this makes a fair bit of sense: they have a lot of common fields, but are
still distinctly different in their relationships (AutomaticCollections have
rules) and some of their behaviour.

But from a business model perspective, these differences are also basically an
implementation detail. The defining behaviour of a collection is that it groups
products; the method of choosing those products is secondary. We could expand
our implementation at some point to permit some third method of choosing
products (machine learning?) or to permit mixing methods (some rules and some
manually added products) and *they would still just be collections*. You could
even argue that the fact we don't permit mixing right now is an implementation
failure. All of this to say is that the shape of our API should really look more
like this:

```graphql
type Collection {
  [CollectionRule]
  Image
  [Product]
}

type CollectionRule { }
```

That's really nice. The immediate concern you may have at this point is that
we're now pretending ManualCollections have rules, but remember that this
relationship is a list. In our new API design, a "ManualCollection" is just a
Collection whose list of rules is empty.

### Conclusion

Choosing the best API design at this level of abstraction necessarily requires
you to have a very deep understanding of the problem domain you're modeling.
It's hard in a tutorial setting to provide that depth of context for a specific
topic, but hopefully the collection design is simple enough that the reasoning
still makes sense. Even if you don't have this depth of understanding
specifically for collections, you still absolutely need it for whatever domain
you're actually modeling. It is critically important when designing your API
that you ask yourself these tough questions, and don't just blindly follow the
implementation.

On a closely related note, a good API does not model the user interface either.
The implementation and the UI can both be used for inspiration and input into
your API design, but the final driver of your decisions must always be the
business domain.

Even more importantly, existing REST API choices should not necessarily be
copied. The design principles behind REST and GraphQL can lead to very different
choices, so don't assume that what worked for your REST API is a good choice for
GraphQL.

As much as possible let go of your baggage and start from scratch.

``` *Rule #3: Design your API around the business domain, not the implementation,
user-interface, or legacy APIs.* ```


## Step Three: Adding Detail

Now that we have a clean structure to model our types, we can add back our
fields and start to work at that level of detail again.

Before we start adding detail, ask yourself if it's really needed at this
time. Just because a database column, model property, or REST attribute may
exist, doesn't mean it automatically needs to be added to the GraphQL schema.

Exposing a schema element (field, argument, type, etc) should be driven by an
actual need and use case. GraphQL schemas can easily be evolved by adding
elements, but changing or removing them are breaking changes and much more
difficult.

``` *Rule #4: It's easier to add fields than to remove them.* ```

### Starting point

Restoring our naive fields adjusted for our new structure, we get:

```graphql
type Collection {
  id: ID!
  rules: [CollectionRule!]!
  rulesApplyDisjunctively: Bool!
  products: [Product!]!
  title: String!
  imageId: ID
  bodyHtml: String
}

type CollectionRule {
  column: String!
  relation: String!
  condition: String!
}
```

Now we have a whole new host of design problems to resolve. We'll work through
the fields in order top to bottom, fixing things as we go.

### IDs and the `Node` Interface

The very first field in our Collection type is an ID field, which is fine and
normal; this ID is what we'll need to use to identify our collections throughout
the API, in particular when performing actions like modifying or deleting them.
However there is one piece missing from this part of our design: the `Node`
interface. This is a very commonly-used interface that already exists in most
schemas and looks like this:
```graphql
interface Node {
  id: ID!
}
```
It hints to the client that this object is persisted and retrievable by the
given ID, which allows the client to accurately and efficiently manage local
caches and other tricks. Most of your major identifiable business objects
(e.g. products, collections, etc) should implement `Node`.

The beginning of our design now just looks like:
```graphql
type Collection implements Node {
  id: ID!
}
```

``` *Rule #5: Major business-object types should always implement `Node`.* ```

### Rules and Subobjects

We will consider the next two fields in our Collection type together: `rules`,
and `rulesApplyDisjunctively`. The first is pretty straightforward: a list of
rules. Do note that both the list itself and the elements of the list are marked
as non-null: this is fine, as GraphQL does distinguish between `null` and `[]`
and `[null]`. For manual collections, this list can be empty, but it cannot be
null nor can it contain a null.

*Protip: List-type fields are almost always non-null lists with non-null
elements. If you want a nullable list make sure there is real semantic value in
being able to distinguish between an empty list and a null one.*

The second field is a bit weird: it is a boolean field indicating whether the
rules apply disjunctively or not. It is also non-null, but here we run into a
problem: what value should this field take for manual collections? Making it
either false or true feels misleading, but making the field nullable then makes
it a kind of weird tri-state flag which is also awkward when dealing with
automatic collections. While we're puzzling over this, there is one other thing
that is worth mentioning: these two fields are obviously and intricately related.
This is true semantically, and it's also hinted by the fact that we chose names
with a shared prefix. Is there a way to indicate this relationship in the schema
somehow?

As a matter of fact, we can solve all of these problems in one fell swoop by
deviating even further from our underlying implementation and introducing a new
GraphQL type with no direct model equivalent: `CollectionRuleSet`. This is often
warranted when you have a set of closely-related fields whose values and
behaviour are linked. By grouping the two fields into their own type at the API
level we provide a clear semantic indicator and also solve all of our problems
around nullability: for manual collections, it is the rule-set itself which is
null. The boolean field can remain non-null. This leads us to the following
design:

```graphql
type Collection implements Node {
  id: ID!
  ruleSet: CollectionRuleSet
  products: [Product!]!
  title: String!
  imageId: ID
  bodyHtml: String
}

type CollectionRuleSet {
  rules: [CollectionRule!]!
  appliesDisjunctively: Bool!
}

type CollectionRule {
  column: String!
  relation: String!
  condition: String!
}
```

*Protip: Like lists, boolean fields are almost always non-null. If you want a
nullable boolean, make sure there is real semantic value in being able to
distinguish between all three states (null/false/true) and that it doesn't
indicate a bigger design flaw.*

``` *Rule #6: Group closely-related fields together into subobjects.* ```

### Lists and Pagination

Next on the chopping block is our `products` field. This one might seem safe;
after all we already "fixed" this relation back when we removed our `CollectionMembership`
type, but in fact there's something else wrong here.

The field as currently defined returns an array of products, but collections can
easily have many tens of thousands of products, and trying to gather all of
those into a single array would be incredibly expensive and inefficient. For
situations like this, GraphQL provides lists pagination.

Whenever you implement a field or relation returning multiple objects, always
ask yourself if the field should be paginated or not. How many of this object
can there be? What quantity is considered pathological?

Paginating a field means you need to implement a pagination solution first.
This tutorial uses [Connections](https://graphql.org/learn/pagination/#complete-connection-model)
which is defined by the [Relay Connection spec](https://facebook.github.io/relay/graphql/connections.htm).

In this case, paginating the products field in our design is as simple as
changing its definition to `products: ProductConnection!`. Assuming you have
connections implemented, your types would look like this:

```graphql
type ProductConnection {
  edges: [ProductEdge!]!
  pageInfo: PageInfo!
}

type ProductEdge {
  cursor: String!
  node: Product!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
}
```


``` *Rule #7: Always check whether list fields should be paginated or not.* ```

###  Strings

Next up is the `title` field. This one is legitimately fine the way it is. It's
a simple string, and it's marked non-null because all collections must have a
title.

*Protip: As with booleans and lists, it's worth noting that GraphQL does
distinguish between empty strings (`""`) and nulls (`null`), so if you need a
nullable string make sure there is a legitimate semantic difference between
not-present (`null`) and present-but-empty (`""`). You can often think of empty
strings as meaning "applicable, but not populated", and null strings meaning
"not applicable".*

### IDs and Relations

Now we come to the `imageId` field. This field is a classic example of what
happens when you try and apply REST designs to GraphQL. In REST APIs it's
pretty common to include the IDs of other objects in your response as a way to
link together those objects, but this is a major anti-pattern in GraphQL.
Instead of providing an ID, and forcing the client to do another round-trip to
get any information on the object, we should just include the object directly
into the graph - that's what GraphQL is for after all. In REST APIs this pattern
often isn't practical, since it inflates the size of the response significantly
when the included objects are large. However, this works fine in GraphQL because
every field must be explicitly queried or the server won't return it.

As a general rule, the only ID fields in your design should be the IDs of the
object itself. Any time you have some other ID field, it should probably be an
object reference instead. Applying this to our schema so far, we get:

```graphql
type Collection implements Node {
  id: ID!
  ruleSet: CollectionRuleSet
  products: ProductConnection!
  title: String!
  image: Image
  bodyHtml: String
}

type Image {
  id: ID!
}

type CollectionRuleSet {
  rules: [CollectionRule!]!
  appliesDisjunctively: Bool!
}

type CollectionRule {
  column: String!
  relation: String!
  condition: String!
}
```

``` *Rule #8: Always use object references instead of ID fields.* ```

### Naming and Scalars

The last field in our simple `Collection` type is `bodyHtml`. To a user who is
unfamiliar with the way that collections were implemented, it's not entirely
obvious what this field is for; it's the body description of the specific
collection. The first thing we can do to make this API better is just to rename
it to `description`, which is a much clearer name.

``` *Rule #9: Choose field names based on what makes sense, not based on the
implementation or what the field is called in legacy APIs.* ```

Next, we can make it non-nullable. As we talked about with the title field, it
doesn't make sense to distinguish between the field being null and simply being
an empty string, so we don't expose that in the API. Even if your database
schema does allow records to have a null value for this column, we can hide that
at the implementation layer.

Finally, we need to consider if `String` is actually the right type for this
field. GraphQL provides a decent set of built-in scalar types (`String`, `Int`,
`Boolean`, etc) but it also lets you define your own, and this is a prime use
case for that feature. Most schemas define their own set of additional scalars
depending on their use cases. These provide additional context and semantic
value for clients. In this case, it probably makes sense to define a custom
`HTML` scalar for use here (and potentially elsewhere) when the string in
question must be valid HTML.

Whenever you're adding a scalar field, it's worth checking your existing list of
custom scalars to see if one of them would be a better fit. If you're adding a
field and you think a new custom scalar would be appropriate, it's worth talking
it over with your team to make sure you're capturing the right concept.

``` *Rule #10: Use custom scalar types when you're exposing something with specific
semantic value.* ```

### Pagination Again

That covers all of the fields in our core `Collection` type. The next object is
`CollectionRuleSet`, which is quite simple. The only question here is whether or
not the list of rules should be paginated. In this case the existing array
actually makes sense; paginating the list of rules would be overkill. Most
collections will only have a handful of rules, and there isn't a good use case
for a collection to have a large rule set. Even a dozen rules is probably an
indicator that you need to rethink that collection, or should just be manually
adding products.

### Enums

This brings us to the final type in our schema, `CollectionRule`. Each rule
consists of a column to match on (e.g. product title), a type of relation (e.g.
equality) and an actual value to use (e.g. "Boots") which is confusingly called
`condition`. That last field can be renamed, and so should `column`; column is
very database-specific terminology, and we're working in GraphQL. `field` is
probably a better choice.

As far as types go, both `field` and `relation` are probably implemented
internally as enumerations (assuming your language of choice even has
enumerations). Fortunately GraphQL has enums as well, so we can convert those
two fields to enums. Our completed schema design now looks like this:

```graphql
type Collection implements Node {
  id: ID!
  ruleSet: CollectionRuleSet
  products: ProductConnection!
  title: String!
  image: Image
  description: HTML!
}

type CollectionRuleSet {
  rules: [CollectionRule!]!
  appliesDisjunctively: Bool!
}

type CollectionRule {
  field: CollectionRuleField!
  relation: CollectionRuleRelation!
  value: String!
}

enum CollectionRuleField {
  TAG
  TITLE
  TYPE
  INVENTORY
  PRICE
  VENDOR
}

enum CollectionRuleRelation {
  CONTAINS
  ENDS_WITH
  EQUALS
  GREATER_THAN
  LESS_THAN
  NOT_CONTAINS
  NOT_EQUALS
  STARTS_WITH
}
```

``` *Rule #11: Use enums for fields which can only take a specific set of values.* ```
