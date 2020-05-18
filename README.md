# Inventory and order API

A Flask API with an SQL database used for maintaining inventory and placing orders.

## Contents

1. [Set up](#set-up)
1. [API](#api)
1. [Development tools and features](#development-tools-and-features)
1. [Program design](#program-design)
1. [Troubleshooting](#troubleshooting)
1. [Pair programming review](#pair-programming-review)

## Set up

You will need `docker-compose` to run the app.

- Install dependencies: `make install`
- Start server: `make start`
- Or run in background: `make daemon`
- Stop the containers: `make stop`
- Unit tests :`make test`
- Unit tests with coverage: `make test.coverage`
- Lint: `make test.lint`
- Refresh/clear database tables: `make database.refresh`

The API is running locally on port `3000`. You can verify the server and database were set up correctly by navigating to the [menu](http://localhost:3000/v1/menu) endpoint or by running sample commands in the [API](docs/API.md).

## API

This [document](docs/API.md) contains the API resources, JSON specs and sample curl commands.

## Development tools and features

1. Continuous Integration: `Github actions` to run unit tests and linting on every push to remote. The action will fail if a linting rule is violated or if there is < 90% unit test coverage.
1. Database with versioning: `Postgresql` database with `Alembic` being for setting up initial tables and handling database versioning.
1. Docker: `docker-compose` to manage development, test and production environments for the Flask and SQL containers
1. Linting: [`flake8`](setup.cfg)
1. Logging: container logs can be veiwed by running `make server.logs`

## Program design

Inside the `src` folder, there are four main subfolders: `routes`, `resources`, `commands` and `models`.

- `routes` defines the HTTP endpoints and assigns a `resource` class to handle the needs of the request. Instead of directly adding all the business logic here we use`flask_restful` to assign classes to endpoints. We also use Flask Blueprints to allow us to define and later combine all the different routes together in the main `server.py`. It also lets us provide an api version prefix which is useful for supporting backwards compatability with our integrations if breaking changes are introduced.

- `resources` handles parsing the requests and will take care of the business logic by invoking other functions. It will then return a JSON response to the client. These classes essentially serve as interfaces for client's HTTP requests, having them as seperated classes allow us to easilly add more functionality, like authentication, later on.

- `models` serve as an ORM used for abstracting interactions with our PSQL database. The models all inherent from an abstract base class (see below) to perform generic database tasks like creating, querying or deleting entities. The models follow an [Active record pattern](https://en.wikipedia.org/wiki/Active_record_pattern) where the class attributes are columns of our database tables.

- `commands` are the classes that translate and combine the generic database actions to perform higher-level operations and follows the [command pattern](https://en.wikipedia.org/wiki/Command_pattern). We found it was useful to not merge the functionality of this class with the models as it allowed us to use the commands in our unit tests when we wanted to initialize objects in our test db. Instead of repeating chunks of ORM code throughout the app, we could have these classes store commands and invoke them with simpler calls to `get(), create()`, `update()`, `delete()`, etc. This pattern can also be used to manage several requests from an invoker into a queue and can make the app more resillient to receiving many requests to our database models and could perform retries on fails caused by those CRUD requests.

- `database` contains all the code for managing our alembic database. It's a generic set up for database versioning, you can find more info [here](https://alembic.sqlalchemy.org/en/latest/tutorial.html).

- We used an [abstract base class](src/models/abc.py) as a parent class to inherit the common actions needed to create, read, update or delete items in our PSQL database. The `Order` and `Item` models both inherit from this class.

- We use integration tests against an identical test server and test database to check that HTTP requests made to our API return the correct response and make the correct changes to our database. Instead of unit testing every function and its logic, this approach worked well with the purpose of our API and results in fewer tests with more coverage that gives us greater confidence in our code's health.

- Our database contains two main tables, `orders` and `items`. Their rows match the objects given [here](docs/API.md). The `orders` refer to `items` using foreign keys for `name` and `size` to get specific pricing and category values to limit redundancy between tables and make updating pricing information possible.

## Troubleshooting

- For database issues, try to drop and recreate tables with `make database.refresh`
- After stopping with `make stop`, to delete all stopped docker containers, you can run `docker rm $(docker ps -a -q)`
