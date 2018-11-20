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
