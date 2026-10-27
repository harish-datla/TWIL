# Test setup and teardown

[Back to Part 4](../index.md)

Initialize predictable database state with `beforeEach`, remove test data, insert fixtures, and close connections with `after`. Use `Promise.all`, sequential `for...of`, or Mongoose `insertMany` when preparing asynchronous test data.

[Read Part 4 online](https://fullstackopen.com/en/part4)
