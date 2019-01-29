# Javascript dependency injection and microservices

## Singletons

Many headaches in Javascript development result from the singleton method of loading modules.

Consider the case of MongoDB client singleton:

```javascript
// mongodb.js
import { MongoClient } from 'mongodb'
export default new MongoClient(process.env.MONGO_URI)
```

This file exports an instance of `MongoClient` that has yet to connect, via the `.connect()` method.

```javascript
// app.js
import db from './db'
db.connect()
  .then(client => client.db())
  .then(db => {
    const app = express()
    // ...
    app.listen(3000)
  })
```

This is an extremely common pattern, but can be difficult to work with for several reasons:

1. Library-specific code is introduced into the implementation layer

Because Javascript is asynchronous, it is impossible to export an instance of `Db` from `mongodb.js`, and so it becomes the implementation's responsibility to do that. As applications grow, this quickly becomes unwieldy as more services must be configured before the application can start.

2. Modules must be able to self-configure

By hiding `process.env.MONGO_URI` within `mongodb.js`, the coder is not only required to know how to use the module, but also how it self-configures. This makes debugging code and writing tests (and mocks) much more difficult, and much development time is generally wasted.

3. It becomes expensive and time-consuming to change module code

Because we have written code specifically around connecting to a Mongo database, it will be a lot of work to change this in the future. In order to change the behavior of our database client, or replace MongoDB with another database, application code, configuration and mocks will all have to be rewritten, not to mention the client itself. As a result, developers end up wasting a lot of time coding around libraries instead of focusing on the actual business logic behind the application.

4. Upstream modules can no export the proper objects

With an asynchronous database connection, you can see that there is no way to export `app` from `app.js`.

While there are ways to work around these issues using different testing and mocking frameworks, dependency injection can solve them all simultaneously.

## Dependency injection

1. Factories

An elegant alternative to the singleton pattern is the Factory pattern. A factory module exports a function that, when called, returns the dependency. By contrast, singletons export the dependency itself.

```javascript
// db.js
import { MongoClient } from 'mongodb'
export default async () => MongoClient.connect(process.env.MONGO_URI).then(client => client.db())
```

Note that our factory is an asynchronous function. Now the only thing `app.js` needs to do is to call it and `await`/`.then()` the return value to receive a database handle. With the MongoDB-specific code removed, the application setup is reduced to:

```javascript
// app.js
import dbFactory from './db'
dbFactory().then(db => {
  const app = express()
  // routes...
  app.listen(3000)
})
```

This allows us to contain the database logic to `db.js` so that we can change it without needing to adjust our application logic.

2. Inversion of control (IoC)

Inversion of control is a concept closely related to dependency injection. IoC moves module configuration from the bottom of the dependency tree to the top. What this means for a factory is simply that it can receive its configuration, rather than try to build it.

```javascript
// db.js
import { MongoClient } from 'mongodb'
export default async (uri) => MongoClient.connect(uri).then(client => client.db())
```

```javascript
// app.js
import dbFactory from './db'

const { MONGO_URI } = process.env

dbFactory(MONGO_URI).then(db => {
  const app = express()
  // routes...
  app.listen(3000)
})
```

As a result, we will be able to see the entire application configuration in one place, and move it into it's own module if desired (i.e. `config.js`). This is good so far, but we are still doing some database configuration inside our application, which is not ideal. To solve this, we can rewrite `app.js` to be a factory as well, and remove any non-application logic from it. Instead of doing the database setup, the app could just receive the database connection as its own dependency.

```javascript
// app.js
export default (db) => {
  const app = express()
  // routes...
  app.listen(3000)
}
```

Then we'll create a new app entrypoint/index to handle dependency configuration and start the app.

```javascript
// index.js
import dbFactory from './db'
import appFactory from './app'

dbFactory(process.env.MONGO_URI).then(appFactory)
```

3. Service factories

In a more complex application, we may want to further componentize service configuration into a factory of its own:

```javascript
// services.js
import loggerFactory from './logger'
import dbFactory from './db'
import userServiceFactory from './user-service'
import paymentServiceFactory from './payment-service'

export default async (config) => {
  const services = {}
  services.logger = loggerFactory(config.logger)
  services.db = await dbFactory(config.db)

  // paymentServices needs a logger and a db
  services.paymentService = paymentServiceFactory(config.payment)(services)

  // userService needs a logger, a db, and a paymentService
  services.userService = userServiceFactory(services)

  return services
}
```

Then in the application:

```javascript
// index.js
import config from './config'
import servicesFactory from './services'
import appFactory from './app'

// connect to all services and then provide them to appFactory
servicesFactory(config).then(appFactory)
```

In this way, the application code is never even reached if even one of our service connection fails, making debugging much easier.

4. TODO: Higher-order factories

## Service-oriented architecture

One of the most important skills to develop as a coder is the ability to separate library-specific logic from business logic.

Consider the case of a `userService`, which in this example is simply an object with methods on it that allow us to create, update, and otherwise operation on a collection of users backed by MongoDB. It's tempting to create a user class or factory to handle all of this, but when writing enterprise-level code, we will want to introduce a few levels of abstraction to help separate different code responsibilities such as logging, validation, or database logic.

1. The library layer

The first (and arguably most important) abstraction is the library layer, which serves to normalize a library's API into one of our own. By relying on our own abstraction, we can easily handle changes in the underlying library, or replace the library with another one without needing to modify our application.

A good example is a `util.js` used to defined commonly used helper methods. If we wanted to use ramda's `pick` in my application, but underscore's `is` method, it would be a mistake to import those modules directly all over the codebase. Instead, we can create a library abstraction to use instead, where we have control over which methods to use from which libary.

```javascript
import { pick } from 'ramda'
import { is } from 'underscore'
const toUpper = str => str.toUpperCase()

export { pick, is, toUpper }
```

In this way, we can easily change from one library to another, or replace these methods with custom implementations.

2. The repository layer

When working with databases, the repository layer serves to translate the database client's API into methods that we can rely on.

```javascript
import { ObjectID } from 'bson'

const userRepositoryFactory = coll => {
  const insert = async (obj) => coll.insertOne(obj).then(result => result.ops[0])
  const findById = async (_id) => coll.findOne({ _id: new ObjectID(_id) })
  return { insert, findById }
}
```

This will allow us to easily change the underlying implementation of `userRepository` while maintaining a consistent interface. It's important to note that by design, no validation, logging, or other logic is included. We have created a single-responsibility module that serves only to translate data coming in and out of the database. By injecting our `coll` dependency, we have also removed the need to overload Javascript's module loader in order to mock it.

Because the repository layer is so simple, we will very rarely find ourselves debugging any logic specific to MongoDB or how it accepts or returns data once the respository is written.

3. The service layer

The service layer sits one level above the repository and can be responsible for concerns such as validation or logging as it reads and writes to the repository. It can also serve to translate objects into domain objects (class instances). A `userService` implementation 

```javascript
import User from './user-type'

// multiple dependencies are passed in an object
const userServiceFactory = ({ logger, userRepository }) => {
  const create = (params) => {
    // validation
    if (!isPopulatedString(params.name)) {
      throw new ValidationError('You must provide a name')
    }

    // logging
    logger.info(`creating account for ${params.name}`)

    // custom business logic
    params.score = calculateScore(params)

    // respository
    const userObj = await userRepository.insert(user)

    // domain object translation
    const user = new User(userObj)

    return user
  }
  return { create }
}
```

Because our service is a factory, we will even be able to separate these responsiblities further if desired, using higher-order factories or other methods.

Using Jest as an example, we can easily unit test the service logic with very little effort.

```javascript
// create some mocks
const logger = {
  info: jest.fn(),
  debug: jest.fn(),
}
const userRepository = {
  insert: jest.fn(),
  findById: jest.fn(),
}

// start the service using the mocks as dependencies
const userService = userServiceFactory({ logger, userRepository })

describe('userService', () => {
  describe('.create()', () => {
    it('should require user.name', () =>
      expect(userService.create({})).rejects.toThrow()
    )

    it('should calculate and set user.score', () => {
      // a valid user
      const params = { name: 'michael' }
      // mock the repository response to return the user
      userRepository.insert = jest.fn().mockResolvedValue(params)
      
      // call the service and verify that the repository was provided the user, and user.score was set
      const user = userService.create(params)
      expect(userRepository.insert).toHaveBeenCalledWith({
        name: 'michael',
        score: 100,
      })
      
      // verify the return value also has the score
      expect(user.score).toEqual(100)
      
    })
  })
})
```

A rather mediocre definition of a unit test would be something like, "a script that tests a specific piece of code," but better definition would be, "a script that tests a specific piece of logic." By injecting our userRepository as a dependency, it becomes trivial to write simple unit tests that do not need external connections (an integration test). With this architecture, we can write true unit tests that are only checking the bits of logic that the service is responsible for, and not have to worry about what is going on downstream.
