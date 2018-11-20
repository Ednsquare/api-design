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

*Rule #1: Always start with a high-level view of the objects and their
relationships before you deal with specific fields.*
