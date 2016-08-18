# Kinvey Backend SDK (beta)

This is the SDK for Backend code execution for kinvey-hosted Data Link Connectors and Logic tasks. The module provides a framework for building kinvey-backed Data Link Connectors easily.



This module provides an easy way to connect to a Kinvey Business Logic (BL) instance running on docker. It is up to the user to download, configure and start the docker image itself before running code using this module. Two [utility](#Utilities) functions are provided to automate setting up the docker image.

## Installation

To install this project, add it to your `package.json` file and install it via `npm`
```
npm install kinvey-backend-sdk
```

To use this module, require it in your project and
```
const sdk = require('kinvey-backend-sdk');
```

You then must initialize the sdk to retrieve a reference to the backend service:

```
const service = sdk.service((err, service) => {
  // code goes here
};
```


When running locally, you can specify a host and port to listen on by passing an options object with an optional host and port.  If no host/port is specified, localhost:10001 will be used:

```
const service = sdk.service({ host: 'somehost', port: 7777 }, (err, service) => {
  // code goes here
};
```

To run your code locally, execute `node .` in the root of your project.  Routes conform to the Kinvey Data Link specification.  

## DataLink framework

The DataLink framework can be accessed via the sdk's `dataLink` property.

```
const dataLink = sdk.dataLink;
```

### Registering ServiceObjects

Once you initialize the DataLink framework, you define your entites by defining ServiceObjects, and then wire up data access event handlers to those ServiceObjects.  To register a ServiceObject, use the `serviceObject` method of the `dataLink` framework:

```
// To register the 'widgets' ServiceObject:
const widgets = sdk.dataLink.serviceObject('widgets');
```

### Data Events

Each ServiceObject exposes data events that are invoked by Kinvey collections.  The data event takes a single handler function to be executed.


| event     | description |
| --------- | ----------- |
| onInsert  | executed on inserts (or POST to REST API) |
| onUpdate  | executed on updates (or PUT to the REST API) |
| onDeleteById | executed when a single entity is to be deleted |
| onDeleteByQuery | executed when a query is included as part of a DELETE |
| onDeleteAll | executed when the DELETE command is invoked |
| onGetById | get a single entity by Id |
| onGetByQuery | retrieve results based on a query |
| onGetAll | get all entities in a given ServiceObject |
| onGetCount | get the count of the entities in a ServiceObject |
| onGetCountByQuery | get the count of the entities in a query result |

For example, to get all entities in `widgets`:

```
widgets.onGetAll(callbackFunction);
```

### Data Handler Functions

The data events take a handler funciton, which takes two arguments:  `request` and `complete`.  `request` represents the request made to Kinvey, and `complete` is a completion handler for completing the data request.

#### request object

The request object contains the following properties

| property | description |
| --------- | ----------- |
| method    | the HTTP method (GET, POST, PUT, DELETE) |
| headers   | the HTTP request headers |
| entityId  | the entityId, if specified |
| serviceObjectName | the name of the serviceObject |
| body | the HTTP body |
| query | the query object |

#### Completion Handler

The completion handlers object follows a builder pattern for creating the datalink's response.  The pattern for the completion handler is `complete(<entity>).<status>.<done|next>`

For example, a sample completion handler is:

```
complete(myEntity).ok.next()
```

##### complete

The `complete` handler takes either an entity, an array of entities, or an error description.  The result of the `complete` handler is an object of status functions.

```
// Sets the response to include an entity.
complete({"foo", "bar"});

// Sets the response to include an array of entities
complete([{"foo":"bar"}, {"abc":"123}]);

// Sets the response to an error string, to be used with error status codes
complete("Record 123 was not found");
```

##### status functions

Status functions set the valid status codes for a Data Link Connector.  The status function also sets the body to a Kinvey-formatted error, and uses the value passed into the `complete` function as the debug property, if it is present.

The available status functions are:

| Function | Status Code | Description |
| --------- | ----------- |------------|
| ok | 200 | Used for a normal success response |
| created | 201 | Used for creating new records |
| accepted | 202 | Used when the request has been submitted for processing, but will be processed asynchronously |
| notFound | 404 | Used when the entity or entities could not be found |
| badRequest | 400 | Used when the request is invalid |
| unauthorized | 401 | Used when the request has not been authorized for the given user context |
| forbidden | 403 | Used when the specific request is forbidden for some reason |
| notAllowed | 405 | Used when the specific request or request method is not allowed |
| notImplemented | 501 | Used when the specific handler has no implementation |
| runtimeError | 550 | Used for custom runtime errors |

For example:

```
// Return that the record has been created
complete(myRecord).created();

// Entity wasn't found
complete("The given entity wasn't found").notFound();
```

##### End processing

Once the status is set, you can end the processing of the dataLink request with either `done` or `next`.  Most requests should normally end with `next`, which will continue the Kinvey request pipeline with Business Logic.  `done` will return the response that was set in the DataLink, and end request processing without executing any further business logic.

```
// This will continue the request chain
complete(myEntity).ok().next();

// This will end the request chain with no further processing
complete(myEntity).ok().done();
```

### Example

The following is an example

```
const sdk = require('kinvey-backend-sdk');
const service = sdk.service(function(err, service) {
  const dataLink = service.dataLink;   // gets the datalink object from the service

  function getRecordById(request, complete) {
    let entityId = request.entityId;
    let entity = null;

    // Do some logic to get the entity id from the remote data store
    // Assume that data is retrieved and stored in "entity" variable

    // After entity is retrieved, check to see if it exists
    if (typeof entity === 'undefined' || entity === null || entity === {}) {
      return complete("The entity could not be found").notFound().next();
    } else  {
      // return the entity
      return complete(entity).ok().next();
    }
  }

  // set the serviceObject
  const widgets = dataLink.serviceObject('widgets');

  // wire up the event that we want to process
  widgets.onGetById(getRecordById);

  // wire up the events that we are not implementing
  widgets.onGetByQuery(notImplementedHandler);
  widgets.onGetAll(notImplementedHandler);
  widgets.onGetCount(notImplementedHandler);
  widgets.onInsert(notImplementedHandler);
  widgets.onUpdate(notImplementedHandler);
  widgets.onDeleteAll(notImplementedHandler);
  widgets.onDeleteByQuery(notImplementedHandler);
  widgets.onDeleteById(notImplementedHandler);
};
```

## Business Logic framework

The DataLink framework can be accessed via the sdk's `businessLogic` property.

```
const dataLink = sdk.businessLogic;
```

### Registering Business Logic Handlers

In order to register a business logic handler, you define that handler and give it a name by using the `register` method of the `businessLogic` framework:

```
// To register the '' handler:
const widgets = sdk.businessLogic.register('someEventHanlerName', eventHandlerCallbackFunction);
```

In the console, when you define business logic, you will be presented your list of event handlers to tie to a collection hook or custom endpoint.

### Business Logic Handler Functions

Like the DataLink handlers, business logic events take a handler funciton, which takes two arguments:  `request` and `complete`.  `request` represents the request made to Kinvey, and `complete` is a completion handler for completing the business logic.

#### request object

The request object contains the following properties:

| property | description |
| --------- | ----------- |
| method    | the HTTP method (GET, POST, PUT, DELETE) |
| headers   | the HTTP request headers |
| entityId  | the entityId included in the request, if specified |
| collectionName | the name of the collection |
| body | the HTTP body |
| query | the query object |

#### Completion Handler

The completion handlers object follows a builder pattern for creating the business logic's response.  The pattern for the completion handler is `complete(<entity>).<status>.<done|next>`

For example, a sample completion handler is:

```
complete(myEntity).ok.next()
```

The entity is optional, as it will not always be returned.  

You can also alter the request object by making changes to the query, body, or headers objects.  If the request body is modified, it will be persisted back for subsequent steps in the request pipeline.   

##### complete

The `complete` handler takes either an entity, an array of entities, or an error description.  The result of the `complete` handler is an object of status functions.

```

// No entity as part of the response
complete();

// Sets the response to include an entity.
complete({"foo", "bar"});

// Sets the response to include an array of entities
complete([{"foo":"bar"}, {"abc":"123}]);

// Sets the response to an error string, to be used with error status codes
complete("Record 123 was not found");
```

##### status functions

Status functions set the valid status codes for the request.  The status function also sets the body to a Kinvey-formatted error, and uses the value passed into the `complete` function as the debug property, if it is present.

The available status functions are:

| Function | Status Code | Description |
| --------- | ----------- |------------|
| ok | 200 | Used for a normal success response |
| created | 201 | Used for creating new records |
| accepted | 202 | Used when the request has been submitted for processing, but will be processed asynchronously |
| notFound | 404 | Used when the entity or entities could not be found |
| badRequest | 400 | Used when the request is invalid |
| unauthorized | 401 | Used when the request has not been authorized for the given user context |
| forbidden | 403 | Used when the specific request is forbidden for some reason |
| notAllowed | 405 | Used when the specific request or request method is not allowed |
| notImplemented | 501 | Used when the specific handler has no implementation |
| runtimeError | 550 | Used for custom runtime errors |

For example:

```
// Return that the record has been created
complete(myRecord).created();

// Entity wasn't found
complete("The given entity wasn't found").notFound();
```

##### End processing

Once the status is set, you can end the processing of the business logic request with either `done` or `next`.  Most requests should normally end with `next`, which will continue the Kinvey request pipeline with Business Logic.  `done` will return the response that was set in the handler, and end request processing without executing any further part of the kinvey request pipeline.

```
// This will continue the request chain
complete(myEntity).ok().next();

// This will end the request chain with no further processing
complete(myEntity).ok().done();
```

### Example

The following is an example

```
const sdk = require('kinvey-backend-sdk');
const request = require('request'); // assumes that the request module was added to package.json
const service = sdk.service(function(err, service) {
  
  const bl = service.businessLogic;   // gets the businessLogic object from the service

  function getRedLineSchedule(req, complete) {
    requset.get('http://developer.mbta.com/Data/Red.json', (err, response, body) => {
      // if error, return an error
      if (err) {
        return complete("Could not complete request").runtimeError().done();
      }
      
      //otherwise, return the results
      return complete(body).ok().done();
    });
    
   }

  // set the handler
  bl.register('getRedLineData', getRedLineSchedule);
};
```

You can include both dataLink and business logic handlers in the same flex service

## Executing a long-running script

Because the services are persisted, you can execute long running tasks that run in the background.  However, all requests still need to be completed in less than 60 seconds.  

To accomplish this, you can execute a function asynchronously using one of the `Timer` functions from node.js: `setImmediate`, `setTimeout`, or `setInterval`.  
  
For example:  

```
const sdk = require('kinvey-backend-sdk');
const request = require('request'); // assumes that the request module was added to package.json
const service = sdk.service(function(err, service) {
  
  const bl = service.businessLogic;   // gets the businessLogic object from the service

    function calcSomeData() {
        // do something
    }

  function calcAndPostData() {
    auth = {
      user: '<MY_APP_KEY>',
      pass: '<MY_APP_SECRET>'
     };
     
    options = {
        url: 'https://baas.kinvey.com/appdata/<MY_APP_KEY?/someCollection',
        auth: auth,
        json: {
            someData: calcSomeData(),
            date: new Date().toString()
        }
     }
        
    request.get(options, (err, response, body) => {
      // if error, return an error
      if (err) {
        return console.log('Error: ' + err);
      }
      
      //otherwise, just return
      return; 
    });
    
  }
   
  function initiateCalcAndPost(req, complete) {
    // Since the calc and post data function may take a long time, execute it asynchronously
    setImmediate(calcAndPostData);  
    
    // Immediately complete the business logic script.  The response will be returned to the caller, and calcAndPostData will execute in the background.  
    complete().accepted().done();
    
  // set the handler to point to the initiateCalcAndPost header
  bl.register('getRedLineData', initiateCalcAndPost);
};
```

In the above example, the handler receives a request and executes the `initiateCalcAndPost` function.  The function schedules an immediate asynchronous execution of `calcAndPostData`, and then executes the complete handler, returning control to the client and sending the response (in this case, `accepted`, because the request is accepted for processing).  The service stays running in the background, and the long-running `calcAndPostData` function is executed asynchronously.  
