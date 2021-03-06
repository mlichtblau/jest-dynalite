# jest-dynalite

[![Pipeline status](https://github.com/freshollie/jest-dynalite/workflows/Pipeline/badge.svg)](https://github.com/freshollie/jest-dynalite/actions)
[![Npm version](https://img.shields.io/npm/v/jest-dynalite)](https://www.npmjs.com/package/jest-dynalite)
[![code style: prettier](https://img.shields.io/badge/code_style-prettier-ff69b4.svg?style=flat-square)](https://github.com/prettier/prettier)

> Enchaned unit testing, with a mock DynamoDB instance

`jest-dynalite` is a fork of [@shelf/jest-dynamodb](https://github.com/shelfio/jest-dynamodb) that allows unit tests to execute real
queries against a local DynamoDB instance. It was created in an attempt to address some of the most important missing
features of `@shelf/jest-dynamodb`, such as the fact that it uses a single shared database, which makes it hard to keep tests independent while also potentially causing race conditions because of jest's parallel execution of tests (see [this issue](https://github.com/shelfio/jest-dynamodb/issues/55) for more information).

## Why should I use this?

Using this `jest-dynalite` makes writing queries with DynamoDB very easy, your tests can really
check if your data is manipulated in the way you expect it to be. This means that queries and mutations
can be developed without ever having to deploy or run your application, and significantly speeds up
writing code which interacts with DynamoDB.

This in turn makes your tests much more robust, because a change to a data structure or
db query in your application will be reflected by failing tests, instead of using mocks to check
if calls were made correctly.

This library could almost be seen as an integration test, but the lines are very blurred these days and
I'd definitely place this within the unit testing boundary because it can easily integrate with unit tests.

## Features

- Optionally clear tables between tests
- Isolated tables between test runners
- Ability to specify config directory
- No `java` requirement

## **BREAKING CHANGES**

From `v2.0.0` `jest-dynalite` now uses a JavaScript file for table configuration. This change makes it possible to set the dynalite config programatically (enabling things such as reading the parameters from a cloudformation template) while also improving compatibility with jest-dynamodb. Thanks to [@corollari](https://github.com/corollari) for this change.

From `v3.0.0` you can now use the preset in a monorepo. The `jest-dynalite-config.js` will be picked up from your jest `<rootDir>`, which should be the same directory as your jest config.

## Installation

```
yarn add jest-dynalite -D
```

## Timeouts

Because jest has a default timeout of 5000ms per test, `jest-dynalite` can sometimes cause failures due to the timeout
being exceeded. This can happen when there are many tests or lots of tables to create between tests.
If this happens, try increasing your test timeouts `jest.setTimeout(10000)`. Another option is to selectively
run the database only for suites which use it. Please see [advanced config](###Monorepo/Advanced-setup).

## Config

In your jest project root (next to your `jest.config.js`), create a `jest-dynalite-config.js` with the tables schemas,
and an optional `basePort` to run dynalite on:

```js
module.exports = {
  tables: [
    {
      TableName: "table",
      KeySchema: [{ AttributeName: "id", KeyType: "HASH" }],
      AttributeDefinitions: [{ AttributeName: "id", AttributeType: "S" }],
      ProvisionedThroughput: {
        ReadCapacityUnits: 1,
        WriteCapacityUnits: 1
      }
    }
  ],
  basePort: 8000
};
```

## Update your sourcecode

```javascript
const client = new DocumentClient({
  ...yourConfig,
  ...(process.env.MOCK_DYNAMODB_ENDPOINT && {
    endpoint: process.env.MOCK_DYNAMODB_ENDPOINT,
    sslEnabled: false,
    region: "local"
  })
});
```

`process.env.MOCK_DYNAMODB_ENDPOINT` is unqiue to each test runner.

## Jest config

### Simple usage (preset)

jest.config.js

```javascript
module.exports = {
  ...
  preset: "jest-dynalite"
}
```

The simple preset config will use the config and clear tables
between tests.

**Important**: Only use this option if you don't have a custom `testEnvironment` set in your `jest.config.js` file.

### Advanced setup

If you are using your own `testEnvironment` in your Jest configuration, then you must setup
`jest-dynalite` manually. You should also use this manual configuration if you don't want a DynamoDB mock to run
for all your tests (faster).

setupBeforeEnv.js

```javascript
import { setup } from "jest-dynalite";

// You must give it a config directory
setup(__dirname);
```

In every test suite where you are using DynamoDB, apply `import "jest-dynalite/withDb"` to the top of
that test suite to run the db for all the tests in the suite.

If you want the tables to exist for all your suites, create a
`setupAfterEnv.js` file with the content:

```javascript
import "jest-dynalite/withDb";
```

You then must add the setup files to your jest config

jest.config.js

```javascript
module.exports = {
  ...
  setupFiles: ["./setupBeforeEnv.js"],
  setupFilesAfterEnv: ["./setupAfterEnv.js"]
}
```

If you want to be even more granular, you can start
the db yourself at any point.

```javascript
import { startDb, stopDb, createTables, deleteTables } from "jest-dynalite";

beforeAll(startDb);

// Create tables but don't delete them after tests
beforeAll(createTables);

// or
beforeEach(createTables);
afterEach(deleteTables);

afterAll(stopDb);
```

### Other options

jest.config.js

```javascript
module.exports = {
  ...
  testEnvironment: "jest-dynalite/dist/environment",

  setupFilesAfterEnv: [
    "jest-dynalite/dist/setupTables",
    // Optional (but recommended)
    "jest-dynalite/dist/clearAfterEach"
  ]
}
```

This setup should be used if you want to override the default config of `clearAfterEach`, but still want to use the most simple configuration.

## License

`MIT`
