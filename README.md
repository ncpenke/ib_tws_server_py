# Introduction

This package exposes the Interactive Brokers TWS Python API as service. Two options are provided:

1. An asyncio API is provided which can be directly used to implement a service
2. A GraphQL endpoint that internally uses the asyncio API is provided, that allows running the TWS API as a GraphQL server.

This work is distinct from other projects in that it doesn't attempt to re-implement the TWS API. Instead, the TWS API is used as is, and the service is implemented through code generation and supporting classes.

Testing has been performed with TWS API 9.85.1.

# Examples

- [Example of using the asyncio API](./examples/asyncio_client_checks.py)

# TWS API Setup and Notes Notes

The following section describes the Interactive Brokers TWS API setup and notes that describe the TWS API, which were useful in the implementation of this package.

## TWS API Setup

Interactive Brokers does not allow redistribution of the TWS API so it needs to be setup by accepting the license agreement via the following steps:

1. [Download the TWS API](https://interactivebrokers.github.io/#) 
2. After unzipping the downloaded file, follow the steps in the `source/pythonclient/README.md`. At the time of the writing the API can be setup by running the following commands in the `source/pythonclient` directory of the unzipped folder:

    1. Install the wheel module via `pip3 install wheel`
    2. python3 setup.py bdist_wheel
    3. python3 -m pip install --user --upgrade dist/ibapi-*-py3-none-any.whl

The following are notes that characterize the TWS API to help with the design of this project.

## TWS API Threading Model

The TWS API uses the following threading model:

1. Messages from TWS are read from the socket via the `EReader` class in a dedicated thread and pushed into a queue. 
2. Messages to TWS can be sent from any thread via the `EClient` class, but is a blocking operation.
3. The main event loop for the `EClient` class, is responsible for decoding messages sent by TWS by taking them out of the queue used by `EReader`. The dequeue operation is blocking, so the `EClient` event loop is typically run in a dedicated thread.

## TWS API Patterns

The TWS API uses the following request/response patterns:

1. Queries that have a single item response. These have a single method to make a request, an optional method to cancel the request and a single callback for the response. 

    Example: `EClient.reqCurrentTime` and `EWrapper.currentTime` 

2. Queries that have a response consisting of a list of items. These have a single method to make a request, an optional method to cancel the request, and one or more callbacks for each item. Additionally, there's a method or flag in the callbacks to signal the end of the list.
    
    Example: `EClient.reqPositions`, `EClient.cancelPositions`, `EWrapper.position`, `EWrapper.positionEnd`

3. Subscriptions. These have a single method to start the subscription, a method to stop the subscription, and one or more callbacks for status updates.

    Example" `EClient.reqTickByTickData`, `EClient.tickByTickAllLast`, `EWrapper.tickByTickBidAsk` `EClient.cancelTickByTickData`

4. A variant of this pattern are requests that take a requestId parameter. This allows the same type of request to be issued with different parameters

    Example: `EClient.reqPositionsMulti`, `EClient.cancelPositionsMulti`, `EWrapper.positionMulti`, `EWrapper.positionMultiEnd`

5. Fire and forget requests. These have a single request method. 

    Example: `EClient.setServerLogLevel`.

6. A one-off pattern that wraps both a subscription and query in a single request call controlled by a flag.

    Example `EClient.reqHistoricalTicks`, `EWrapper.historicalTicks`, `EWrapper.historicalTicksBidAsk`, `EWrapper.historicalTicksLast`

# Code Generation

The code generation is implemented as part of the `codegen` module and can be run via [ib_tws_server/codegen/main.py](./ib_tws_server/codegen/main.py). 

The code generator assumes the TWS API is available as part of the python module search path. The TWS API version that the generator uses can be changed by modifying the module path via overriding the `PYTHONPATH` environment variable, using Python virtual environments, etc.

The code generator uses definitions captured in [ib_tws_server/api_definition.py](./ib_tws_server/api_definition.py). These definitions describe the TWS API in terms of patterns described in the [TWS API Patterns](#tws-api-patterns) section.

## Generated Files

The generated files are in the `ib_tws_server/gen` directory.

The following files are generated:
- `gen/client_responses.py`:
    - Contains classes used for responses from the TWS API. The following types/classes are generated:
    - Top-Level Unions:
        - For requests that have more than one callback, and have complex responses that return more than one parameter, a Union type is generated to encapsulate all the different return types for a request
    - Callback Classes:
        - A top-level class is generated for every request that has one or more callbacks that return more than one value.
        - For callbacks for queries the response class has the name `{RequestName}Response`
        - Additional classes are generated that encapsulate the parameters for each of the callbacks when the callbacks return one or more parameters
- `gen/asyncio_client.py`: 
    - Contains the AsyncioClient class which subclasses the `ibapi.client.EClient` to provide an asyncio API around the TWS API
    - All request methods are asynchronous and declared using `async`
    - Subscriptions return a `SubscriptionGenerator` instance is an `AsyncGenerator`
    - Request ids of the original TWS API are implicitly managed. 
    - Currently only subscriptions can be cancelled. Even though TWS API allows cancelling queries with multiple responses this is not exposed as part of the API. 
- Other improvements
    - Errors from TWS are propagated via exceptions
    - To avoid blocking the asyncio running loop, an `IBWriter` class to send messages to TWS in a separate thread.
- `gen/asyncio_wrapper.py`: 
    - Subclasses `ibapi.client.EWrapper` and used internally by the `AsyncioClient` class
- `gen/schema.graphql`: The GraphQL schema 
- `gen/graphql_resolver.py`: GraphQL resolvers

# Useful References

- [TWS API](https://interactivebrokers.github.io/tws-api/index.html)
- [ib_insync library](https://github.com/erdewit/ib_insync/tree/master/ib_insync)
- Posts by Juri Sarbach to deploy TWS as a microservice:
    - [Post 1 ](https://medium.com/@juri.sarbach/building-my-own-cloud-based-robo-advisor-5588ec1b74d3)
    - [Post 2 Serverless](https://levelup.gitconnected.com/run-gateway-run-algorithmic-trading-the-serverless-way-71634dc1a37)
- [Guide to Interactive Brokers API Code](https://github.com/corbinbalzan/IBAPICode/blob/master/ExecOrders_Part2/ibProgram1.py)
- [Build a GraphQL API with Subscriptions using Python, Asyncio and Ariadne](https://www.twilio.com/blog/graphql-api-subscriptions-python-asyncio-ariadne)
- [Introduction to Generators](https://realpython.com/introduction-to-python-generators/)
- [GraphQL Configuration](https://graphql-config.com/introduction/)
- [Resolving Union Types in Ariadne](https://ariadnegraphql.org/docs/unions)
