---
id: plugin-queryComplexity
title: Query Complexity Plugin
sidebar_label: Query Complexity
---

A single GraphQL query can potentially generate a huge workload for a server, like thousands of database operations which can be used to cause DDoS attacks. In order to limit and keep track of what each GraphQL operation can do, the query complexity plugin allows defining field-level complexity values that works with the [graphql-query-complexity](https://github.com/slicknode/graphql-query-complexity) library.

To install, add the `queryComplexityPlugin` to the `makeSchema.plugins` array, along with any other plugins you'd like to include:

```ts
import { makeSchema, queryComplexityPlugin } from "@nexus/schema";

const schema = makeSchema({
  // ... types, etc,
  plugins: [
    // ... other plugins
    queryComplexityPlugin(),
  ],
});
```

The plugin will install a `complexity` property on the output field config:

```ts
export const User = objectType({
  name: "User",
  definition(t) {
    t.id("id", {
      complexity: 2,
    });
  },
});
```

And of course, integrate `graphql-query-complexity` with your GraphQL server. You can setup with `express-graphql` as in [the library's example](https://github.com/slicknode/graphql-query-complexity#usage-with-express-graphql).

### Complexity Value

There are two ways to define the complexity:

- A number
- A complexity estimator

The easiest way to specify the complexity is to just provide a number, as in the example code above. The `complexity` property can be omitted if its value is `1`, provided that you have a simple estimator of 1 when configuring `graphql-query-complexity` like below:

```ts
const complexity = getComplexity({
  // ... other configurations
  estimators: [
    // All undefined complexity values will fallback to 1
    simpleEstimator({ defaultComplexity: 1 }),
  ],
});
```

Another way is with the complexity estimator, which is a function that returns a number, but also provides arguments to compute the final value. The query complexity plugin augments `graphql-query-complexity`'s default complexity estimator by providing its corresponding nexus types to ensure type-safety. No additional arguments are introduced so the function declaration is still syntactically equal.

Augmented complexity estimator function signature:

```ts
type QueryComplexityEstimatorArgs<
  TypeName extends string,
  FieldName extends string
> = {
  // The root type the field belongs too
  type: RootValue<TypeName>;

  // The GraphQLField that is being evaluated
  field: GraphQLField<
    RootValue<TypeName>,
    GetGen<"context">,
    ArgsValue<TypeName, FieldName>
  >;

  // The input arguments of the field
  args: ArgsValue<TypeName, FieldName>;

  // The complexity of all child selections for that field
  childComplexity: number;
};

type QueryComplexityEstimator = (
  options: QueryComplexityEstimatorArgs
) => number | void;
```

And you can use it like so:

```ts
export const users = queryField("users", {
  type: "User",
  list: true,
  args: {
    count: intArg({ nullable: false }),
  },
  // This will calculate the complexity based on the count and child complexity.
  // This is useful to prevent clients from querying mass amount of data.
  complexity: ({ args, childComplexity }) => args.count * childComplexity,
  resolve: () => [{ id: "1" }],
});
```

For more info about how query complexity is computed, please visit [graphql-query-complexity](https://github.com/slicknode/graphql-query-complexity).
