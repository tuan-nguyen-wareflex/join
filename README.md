# @objectwow/join

Join objects with functionality similar to MongoDB’s $lookup

⭐️ Your star shines on us. Star us on GitHub!

# Use case

- In a microservices system where each service owns its own database, querying data requires calling multiple services to retrieve the necessary information and manually joining the data. This package simplifies the process of joining data.

- Imagine you have an array of `orders`. Each order contains `fulfillments`, and each `fulfillment` has a list of `products`. However, in the `product` data, you’re only storing the `productId` and `quantity`. The task is to enrich this data by retrieving the `full product details` for each `product`.

# Installation

```
npm i @objectwow/join
```

# Usage

```typescript
import { joinData } from "@objectwow/join";

const orders = [
  {
    id: 1,
    code: "1",
    fulfillments: [
      {
        id: 11,
        code: "11",
        products: [
          { id: 111, quantity: 1 },
          { id: 112, quantity: 4 },
        ],
      },
      {
        id: 12,
        code: "12",
        products: [{ id: 111, quantity: 8 }],
      },
    ],
  },
];

const products = [
  { id: 111, name: "Product 1", price: 10 },
  { id: 112, name: "Product 2", price: 20 },
  { id: 113, name: "Product 3", price: 30 },
];

const result = await joinData({
  local: orders,
  from: products,
  localField: "fulfillments.products.id",
  fromField: "id",
  as: "fulfillments.products",
  asMap: { name: "name", price: "price" },
});
```

LocalData (orders) will be overwritten. Order products will have the `name` and `price` fields.

```typescript
orders = [
  {
    id: 1,
    code: "1",
    fulfillments: [
      {
        id: 11,
        code: "11",
        products: [
          { id: 111, name: "Product 1", price: 10, quantity: 1 },
          { id: 112, name: "Product 2", price: 20, quantity: 4 },
        ],
      },
      {
        id: 12,
        code: "12",
        products: [{ id: 111, name: "Product 1", price: 10, quantity: 8 }],
      },
    ],
  },
];

result = {
  allSuccess: true,
  joinFailedValues: [],
};
```

Note: see more samples in the [`tests`](https://github.com/objectwow/join/blob/main/tests/core.spec.ts) and [`test-by-cases`](https://github.com/objectwow/join/blob/main/test-by-cases)

```typescript
/**
 * Parameters for the `joinData` function to perform joins between local data and source data.
 */
export interface JoinDataParam {
  /**
   * Local object or array of local objects to be joined.
   */
  local: LocalParam;

  /**
   * Object(s) or an asynchronous callback function that returns the data from the source.
   */
  from: FromParam;

  /**
   * Field name in the local object(s) used for the join.
   */
  localField: string;

  /**
   * Field name in the `from` object(s) used for the join.
   */
  fromField: string;

  /**
   * Optional new field name to store the result of the join in the local object(s).
   */
  as?: string;

  /**
   * Optional mapping from the `fromField` values to new field names in the local object(s).
   */
  asMap?: AsMap;
}

export type FromParam =
  | ((localFieldValues: Primitive[], metadata: any) => any)
  | object
  | object[];

export type AsMap =
  | ((currentFrom: any, currentLocal: any, metadata: any) => any)
  | { [key: string]: string }
  | string;
```

# Join performance

The join method in `@objectwow/join` offers better performance compared to the join techniques used by `databases, Krakend, Hasura, and GraphQL`. Here’s why:

### Traditional Approach (Database, Krakend, Hasura, GraphQL):

1. Loop through the original array.
2. `For each element, make a call` to the table/service containing the related data by its ID.
3. Combine the data from both sources.
4. This results in a time complexity of `O(n x m)`, where `n` is the number of elements in the original array, and `m` is the number of elements fetched from the related table or service.

### @objectwow/join Approach:

1. Provides a `callback function` where the input is `an array of IDs`, allowing the developer to fetch related data from the database/API in a `single call`.
2. Uses JavaScript’s `new Map` to optimize the process, reducing the time complexity from O(m) to O(1), where m is the number of elements retrieved..
3. Combines the data efficiently after retrieving it in bulk through a `single call`.
4. This results in a time complexity of O(n), where `n` is the number of elements in the original array.

By fetching related data in bulk and leveraging efficient JavaScript data structures, `@objectwow/join` minimizes redundant calls and improves overall performance.

### Tradeoff

Of course, the tools/platforms mentioned above offer capabilities that `@objectwow/join` cannot provide, such as direct connection to the data source, pagination, conditional filtering, and more.

# Test

`npm run test` or `npm run test:cov`

| File         | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s                    |
| ------------ | ------- | -------- | ------- | ------- | ------------------------------------ |
| All files    | 93      | 78.68    | 95.23   | 92.8    |                                      |
| core.ts      | 90.82   | 73.17    | 100     | 90.82   | 18,42,53,106,128,141,163,206,247,251 |
| singleton.ts | 100     | 100      | 75      | 100     |                                      |
| util.ts      | 100     | 89.47    | 100     | 100     | 23-24                                |

# Customization

With an out-of-the-box design, you can create your own function using the current structure.

```typescript
import { JoinData } from "@objectwow/join";

export class YourJoin extends JoinData {
  // Use case: Currently, deep values are split by a dot ('.'), but you can use a different symbol if needed
  protected separateSymbol: string;

  // Use case: Return your custom output
  protected generateResult(
    joinFailedValues: Primitive[],
    localOverwrite: LocalParam,
    metadata?: any
  ) {}

  // Use case: Shadow clone local data without overwriting the original.
  protected standardizeLocalParam(
    local: LocalParam,
    metadata?: any
  ): Promise<LocalParam> {}

  // Use case: Automatically call internal or external services to retrieve data based on the input
  protected standardizeFromParam(
    from: FromParam,
    localFieldValues: string[],
    metadata?: any
  ): Promise<any[]> {}

  // Use case: Throw an error if the field is invalid
  protected validateFields(
    arr: { key: string; value: any }[],
    metadata?: any
  ): void {}
}
```

- Using solution 1: Override the default JoinData instance

```typescript
SingletonJoinData.setInstance(new YourJoin())

await joinData({...})
```

- Using solution 2: create new singleton instances as needed

```typescript
import { SingletonJoinData } from "@objectwow/join";

export class YourSingletonJoinData extends SingletonJoinData{}

YourSingletonJoinData.setInstance(new YourJoin())

export async function yourJoinDataFunction(
  params: JoinDataParam,
  metadata?: any
): Promise<JoinDataResult> {
  return YourSingletonJoinData.getInstance().execute(params, metadata);
}

await yourJoinDataFunction({...})
```

- Using solution 3: you can directly use your new class without a singleton instance.

```typescript
const joinCls = new YourJoin()
await joinCls.execute({...})
```

Tips:

- You can call original function (parent function) with `super.
- You can pass anything to the metadata

# Benchmark

- Source:

```typescript
// Generate localData with 100 orders and 2 fulfillments per order
const localData = Array.from({ length: 100 }, (_, i) => ({
  id: i + 1, // Order ID starting from 1
  code: `${i + 1}`, // Order code
  fulfillments: Array.from({ length: 2 }, (_, j) => ({
    id: (i + 1) * 10 + j, // Fulfillment ID for each order
    code: `${(i + 1) * 10 + j}`, // Fulfillment code
    products: Array.from({ length: 2 }, () => {
      const productId = Math.floor(Math.random() * 100) + 111; // Random product ID between 111 and 210
      return {
        id: productId,
        quantity: Math.floor(Math.random() * 10) + 1, // Random quantity between 1 and 10
      };
    }),
  })),
}));

// Generate fromData with 100 products
const fromData = Array.from({ length: 100 }, (_, i) => ({
  id: i + 111, // IDs starting from 111
  name: `Product ${i + 1}`,
  price: Math.floor(Math.random() * 100 + 1), // Random price between 1 and 100
}));
```

- Execute: `node lib/benchmark.js` (need build first)
- Report: JoinData Execution x 125,972 ops/sec ±1.27% (66 runs sampled)
- Device: Macbook Pro M1 Pro, 16 GB RAM, 12 CPU

# Contributors

<a href="https://github.com/objectwow/join/graphs/contributors"><img src="https://opencollective.com/objectwow-join/contributors.svg?width=882&button=false" /></a>

# Contact

If you have any questions, feel free to open an [`open an issue on GitHub`](https://github.com/objectwow/join/issues) or connect with me on [`Linkedin`](https://www.linkedin.com/in/vtuanjs/).

Thank you for using and supporting the project!
